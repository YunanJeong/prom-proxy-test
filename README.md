# prom-proxy-test

Prometheus Proxy beyond firewall

- Prometheus는 exporter로 Pull Request하는 방향으로 통신한다.
- 이는 모니터링 대상 서버가 사설망 등 보안환경에 있다면 사용하기 힘든 방식이다.
- 통신방향을 바꾸기 위해 [prometheus-proxy](https://github.com/pambrose/prometheus-proxy?tab=readme-ov-file)를 활용하는 방법을 정리한다.

## 구조 및 통신 방향

```json
Prometheus => (:8080)Proxy(:50051) <= ProxyAgent => (:9100)Exporter
```

- 모니터링 대상 서버가 여러 개인 경우에도 Proxy 1개, Agent 1개 구성가능
  - 메트릭 조회시 하나의 서버(:8080)로 취급되지만 Job이름과 metric_path를 통해 구분 가능

## Proxy 실행 (Prometheus와 Agent 사이)

```sh
# -d: 백그라운드 실행. 미설정시 default는 콘솔실행
# --network host: Agent는 외부로 Request하므로, 네트워크 범위 혼동이 없도록 호스트모드로 실행해준다.

# Proxy 실행 (Prometheus와 Agent 사이)
docker run --restart=unless-stopped -d \
        -p 50051:50051 \
        -p 8080:8080 \
        --env ADMIN_ENABLED=false \
        --env METRICS_ENABLED=true \
        pambrose/prometheus-proxy:1.21.0
        # 에이전트의 request 수신: 50051 (grpc)
        # Prometheus의 request 수신: 8080 (http)
        # 관리자 포트(비활성화): 8082, 8092
        # 모니터링 대상 수 만큼 포트를 열어 8080으로 연결해주면 좋음 (필수X)
          # 예시) -p 9100:8080 -p 9101:8080 -p 9102:8080 ...
          # Prometheus에서 포트번호로 모니터링 대상을 명확히 구분할 수 있게되어 보편적인 대시보드 호환성이 좋아짐
```

## Agent 실행 (Proxy와 Exporter 사이)

```sh
# Agent 실행(Proxy와 Exporter사이) (로컬 설정파일 예)
docker run --restart=unless-stopped -d
    --network host \
    --mount type=bind,source="$(pwd)"/agent.conf,target=/app/prom-agent.conf \
    --env AGENT_CONFIG=prom-agent.conf \
    pambrose/prometheus-agent:1.21.0
    # 관리자 포트(비활성화): 8083, 8093
    # 온라인 환경에선 AGENT_CONFIG에 URL 가능
```

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

- Prometheus는 Proxy 1개를 node-exporter 1개 (노드 1개)로 취급한다.
- 환경이 허락된다면, 실제 node-exporter 수 만큼 proxy와 agent를 따로 구축하는 것이 grafana에서 보기에 좋다. (job이 아닌 instance로 명확히 구분되므로)
- 이렇게 되면 proxy는 헬름차트화하여 여러 개 실행하기 편하게 만들고, ingress 작업까지 들어가주면 좋을 듯한데,, 손이 너무 많이 간다.
