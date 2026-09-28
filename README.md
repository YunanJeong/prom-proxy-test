# prom-proxy-test

Prometheus Proxy beyond firewall

- Prometheus는 exporter로 Pull Request하는 방향으로 통신한다.
- 이는 모니터링 대상 서버가 사설망 등 보안환경에 있다면 사용하기 힘든 방식이다.
- 통신방향을 바꾸기 위해 [prometheus-proxy](https://github.com/pambrose/prometheus-proxy?tab=readme-ov-file)를 활용하는 방법을 정리한다.

## 구조 및 통신 방향

```json
Prometheus => (:8080)Proxy(:50051) <= ProxyAgent => (:9100)Exporter
```

- 8080은 흔한 포트라 다른 서비스와 겹치기 쉬우므로 도커 배포시 다른 호스트 포트로 매핑 권장 (본 프로젝트는 9101)
- 모니터링 대상 서버가 여러 개인 경우에도 Proxy 1개, Agent 1개 구성가능
  - 대상은 metrics_path로 구분하고, Prometheus scrape 설정에서 대상마다 `labels.instance: <실제 IP:포트>` 를 지정해야 함
  - 지정하지 않으면 모든 대상의 instance가 proxy 주소로 같아져 Grafana에서 노드 구분이 안 됨

## Proxy 실행 (Prometheus와 Agent 사이)

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
```

## Agent 실행 (Proxy와 Exporter 사이)

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
```

### 대상 노드가 많을 때 연결이 죽었다 살았다 하는 경우 (동시 스크랩 수 조정)

- Agent는 기본값(`maxConcurrentClients = 1`)으로는 exporter를 한 번에 하나씩 스크랩한다.
- 대상 노드가 많으면 스크랩이 밀려 Prometheus scrape timeout에 걸리고, target이 죽었다 살았다 한다.
  - Prometheus 쪽 인터벌·타임아웃을 늘리면 완화되지만 원인 해결은 아니다.
- 대상 노드 수에 맞춰 `agent.conf`에서 값을 올린다.

```hocon
agent {
  http {
    maxConcurrentClients = 17    # 동시 스크랩 수 (기본 1). 대상 노드 수 이상으로 설정
  }
}
```

- 설정 가능한 전체 옵션과 기본값: [config/config.conf](https://github.com/pambrose/prometheus-proxy/blob/master/config/config.conf)
  - master 기준이므로, 사용 중인 이미지 버전의 태그로 바꿔서 확인

## Prometheus 단독 설치 (사설망 바깥)

```sh
helm install my-prom prometheus-community/kube-prometheus-stack --version 69.2.4 -f only_prom.yaml
```

## node-exporter 설치 (사설망 내 모니터링 대상 서버)

### Helm

```sh
# 사전에 사설 저장소에 이미지 수동업로드 필요.
# 이 차트의 이미지 출처는 docker.io가 아니라 quay.io라서 프록시 다운로드 불가
helm install my-exporter ./reference/kube-prometheus-stack-69.2.4.tgz -f only_exporter.yaml -n devnet

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
- Helm Chart 활용시, 디폴트로 `hostPort`가 활성화되어 있어서, Container 포트를 그대로 외부접근할 때도 사용가능
  - NodePort Service와 달리, 클러스터 내 다른 Node에 포트가 공유되지 않음
  - hostPort는 일반적인 쿠버네티스 앱 배포에 적절치 않으나, exporter와 같은 DaemonSet기반 앱에는 적절

## memo

- Prometheus는 기본적으로 접속한 주소(proxy 주소)를 instance 라벨로 붙이므로, 그대로 두면 모든 대상이 노드 1개처럼 보인다.
- scrape 설정에서 대상마다 `labels.instance`를 실제 exporter 주소로 지정하면 proxy·agent 1개로도 grafana에서 노드별로 구분된다.
- prometheus-proxy 4.2.0부터 공식 Helm 차트가 제공된다. (`charts/prometheus-proxy`, `charts/prometheus-agent`)
