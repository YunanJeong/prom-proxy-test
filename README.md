# prom-proxy-test

Prometheus Proxy beyond firewall

- Prometheus는 exporter로 Pull Request하는 방향으로 통신한다.
- 이는 모니터링 대상 서버가 사설망 등 보안환경에 있다면 사용하기 힘든 방식이다.
- 통신방향을 바꾸기 위해 [prometheus-proxy](https://github.com/pambrose/prometheus-proxy?tab=readme-ov-file)를 활용하는 방법을 정리한다.

## 디렉토리 구성

```
prom-proxy-test/
├── docker-compose.proxy.yaml   # Proxy 실행 (사설망 바깥)
├── docker-compose.agent.yaml   # Agent 실행 (사설망 안). agent_localtest.conf를 마운트
├── agent_localtest.conf        # Agent 설정 (로컬 연결 테스트용. exporter·federation 대상 예시)
├── only_prom.yaml              # 중앙 Prometheus values (예시·연결 테스트용)
├── only_exporter.yaml          # 사설망 서버의 node-exporter values
└── reference/
    ├── kube-prometheus-stack-86.2.2.tgz   # 사설망 설치용 차트
    └── promproxyConfigSample/             # prometheus-proxy 설정 샘플 (proxy·agent)
```

## 구조 및 통신 방향

```json
중앙 Prometheus => (:8080)Proxy(:50051) <= ProxyAgent => (:9100)Exporter
                                                      => (:9090)사설망 Prometheus
```

Agent가 사설망 안의 대상을 scrape해 그 결과를 Proxy로 전달하고, 사설망 바깥의 Prometheus(이하 `중앙 Prometheus`)가 Proxy를 scrape한다. Agent의 대상은 exporter일 수도 있고, 사설망 안에 둔 Prometheus(이하 `사설망 Prometheus`)일 수도 있다.

- 8080은 흔한 포트라 다른 서비스와 겹치기 쉬우므로 도커 배포시 다른 호스트 포트로 매핑 권장 (본 프로젝트는 9101)

### 대상이 exporter인 경우

- 모니터링 대상 서버가 여러 개인 경우에도 Proxy 1개, Agent 1개 구성가능
  - 대상은 metrics_path로 구분하고, Prometheus scrape 설정에서 대상마다 `labels.instance: <실제 IP:포트>` 를 지정해야 함
  - 지정하지 않으면 모든 대상의 instance가 proxy 주소로 같아져 Grafana에서 노드 구분이 안 됨

### 대상이 사설망 Prometheus인 경우 (federation)

federation은 한 Prometheus가 다른 Prometheus의 `/federate` 엔드포인트를 scrape해서, 그 Prometheus가 저장한 시계열을 가져오는 기능이다. 사설망의 쿠버네티스 클러스터마다 Prometheus를 하나씩 두고 이 기능을 쓰면, 중앙 Prometheus는 서버를 하나하나 scrape하는 대신 클러스터 단위로 메트릭을 가져올 수 있다. 사설망 Prometheus가 클러스터 안에서 ServiceMonitor로 수집한 메트릭(JMX 등)도 중앙 Prometheus까지 함께 전달된다.

중앙 Prometheus가 사설망 Prometheus를 직접 scrape할 수는 없으므로, 아래 순서로 Agent와 Proxy를 거친다.

1. Agent가 사설망 Prometheus의 `/federate`를 scrape하고, 그 응답을 Proxy로 전달
2. 중앙 Prometheus가 Proxy를 scrape

exporter 방식과 달라지는 점:

- 노드 구분: 중앙 Prometheus의 scrape 설정에서 `labels.instance`를 지정하지 않음. 대신 `honor_labels: true`로 사설망 Prometheus가 이미 붙인 instance, job 라벨을 그대로 유지함
- 클러스터 구분: 사설망 Prometheus에 `externalLabels.cluster`를 설정해 둠. 사설망 Prometheus가 `/federate` 응답에 이 라벨을 붙여 내보냄

## Proxy 실행 (중앙 Prometheus와 Agent 사이)

```sh
# -d: 백그라운드 실행. 미설정시 default는 콘솔실행

# Proxy 실행 (Prometheus와 Agent 사이)
docker run --restart=unless-stopped -d \
        -p 50051:50051 \
        -p 9101:8080 \
        --env ADMIN_ENABLED=false \
        --env METRICS_ENABLED=true \
        pambrose/prometheus-proxy:4.2.0
        # 에이전트의 request 수신: 50051 (grpc)
        # Prometheus의 request 수신: 컨테이너 8080 (http)
        # 호스트 8080은 다른 서비스와 겹치기 쉬워서 9101로 매핑 (특히 kube-prometheus-stack의 reloader-web포트와 겹침)
        # 관리자 포트(비활성화): 8082, 8092

# docker compose로 실행
docker compose -f docker-compose.proxy.yaml up -d
```

## Agent 실행 (Proxy와 수집 대상 사이)

```sh
# Agent 실행(Proxy와 Exporter사이) (로컬 설정파일 예)
# --network host: Agent는 외부로 Request하므로, 네트워크 범위 혼동이 없도록 호스트모드로 실행해준다.
docker run --restart=unless-stopped -d \
    --network host \
    --mount type=bind,source="$(pwd)"/agent.conf,target=/app/prom-agent.conf \
    --env AGENT_CONFIG=prom-agent.conf \
    pambrose/prometheus-agent:4.2.0
    # 관리자 포트(비활성화): 8083, 8093
    # 온라인 환경에선 AGENT_CONFIG에 URL 가능

# docker compose로 실행 (agent_localtest.conf 사용)
docker compose -f docker-compose.agent.yaml up -d
```

### 대상 노드가 많을 때 연결이 죽었다 살았다 하는 경우 (동시 스크랩 수 조정)

- Agent는 기본값(`maxConcurrentClients = 1`)으로는 exporter를 한 번에 하나씩 스크랩한다.
- 대상 노드가 많으면 스크랩이 밀려 Prometheus scrape timeout에 걸리고, target이 죽었다 살았다 한다.
  - Prometheus 쪽 인터벌·타임아웃을 늘리면 완화되지만 원인 해결은 아니다.
- 대상 노드 수에 맞춰 `agent.conf`에서 값을 올린다.
  - federation 구성이면 대상이 클러스터마다 하나라서, 서버 수가 많아도 이 값을 크게 올릴 필요가 없다.

```hocon
agent {
  http {
    maxConcurrentClients = 17    # 동시 스크랩 수 (기본 1). 대상 노드 수 이상으로 설정
  }
}
```

- 설정 가능한 전체 옵션과 기본값: [config/config.conf](https://github.com/pambrose/prometheus-proxy/blob/master/config/config.conf)
  - master 기준이므로, 사용 중인 이미지 버전의 태그로 바꿔서 확인

## 중앙 Prometheus 설치 (사설망 바깥)

```sh
helm install my-prom prometheus-community/kube-prometheus-stack --version 86.2.2 -f only_prom.yaml
```

## node-exporter 설치 (사설망 내 모니터링 대상 서버)

### Helm

```sh
# 사전에 사설 저장소에 이미지 수동업로드 필요.
# 이 차트의 이미지 출처는 docker.io가 아니라 quay.io라서 프록시 다운로드 불가
helm install my-exporter ./reference/kube-prometheus-stack-86.2.2.tgz -f only_exporter.yaml -n devnet

# WSL 등에서 테스트시 마운트 문제 발생하면 옵션 추가
--set "prometheus-node-exporter.hostRootFsMount.enabled=false"
```

### Docker

```yml
# docker-compose.yml
version: '2'
services:
  node-exporter:
    container_name: my-exporter
    image: docker.wai/rndadmin/quay.io/prometheus/node-exporter:v1.7.0
    network_mode: host
    pid: host
    restart: unless-stopped
    command:
      - '--path.rootfs=/host'
    volumes:
      - '/:/host:ro,rslave'

# 도커 컴포즈로 실행
docker compose up -d
```

### exporter의 외부 노출 방법

- `node-exporter`는 `Native 앱 설치`와 비슷한 효과가 필요
- Docker 활용시, 네트워크 모드 `host`로 활성화
- Helm Chart 활용시, 디폴트로 `hostNetwork: true`가 활성화되어 있어서, Container 포트가 노드의 같은 포트로 그대로 열리므로 외부에서 접근할 때도 사용가능
  - NodePort Service와 달리, 클러스터 내 다른 Node에 포트가 공유되지 않음
  - hostNetwork는 일반적인 쿠버네티스 앱 배포에 적절치 않으나, exporter와 같은 DaemonSet기반 앱에는 적절

#### 사설망 Prometheus가 scrape하는 경우 (federation 구성)

federation 구성에서는 Agent 대신 사설망 Prometheus가 같은 클러스터의 node-exporter들을 scrape하므로, 노드 간 9100 통신이 허용되어 있어야 한다. 이 포트가 막혀 있다고 node-exporter를 `hostNetwork: false`로 띄우면 사설망 Prometheus가 node-exporter 파드의 IP로 scrape할 수는 있지만, node-exporter가 노드가 아니라 자기 파드를 기준으로 값을 내보낸다.

| 항목 | `hostNetwork: false`일 때 | relabel로 보정 |
|---|---|---|
| CPU, 메모리, 디스크 | 정상 (hostPath로 호스트 값을 읽음) | 불필요 |
| instance, nodename | 파드 IP, 파드 이름 (파드가 재시작할 때마다 바뀜) | 가능 (노드 이름으로 교체) |
| 네트워크 인터페이스, conntrack, 소켓 통계 | 파드 네트워크 네임스페이스의 값 | 불가 |

relabel은 라벨만 바꿀 수 있고 값은 바꾸지 못하므로, 노드 간 9100을 허용하고 `hostNetwork: true`를 유지해야 한다.

## memo

- Prometheus는 기본적으로 접속한 주소(proxy 주소)를 instance 라벨로 붙이므로, 그대로 두면 모든 대상이 노드 1개처럼 보인다.
- scrape 설정에서 대상마다 `labels.instance`를 실제 exporter 주소로 지정하면 proxy·agent 1개로도 grafana에서 노드별로 구분된다.
- 대상이 사설망 Prometheus(federation)이면 반대로 라벨을 지정하지 않고, `honor_labels: true`로 사설망 Prometheus가 붙인 라벨을 유지한다.
- prometheus-proxy 4.2.0부터 공식 Helm 차트가 제공된다. (`charts/prometheus-proxy`, `charts/prometheus-agent`)
