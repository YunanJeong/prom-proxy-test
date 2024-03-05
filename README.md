# prom-proxy-test

Prometheus Proxy beyond firewall

```sh
# -d: 백그라운드 실행. 미설정시 default는 콘솔실행
# --network host: Agent는 외부로 Request하므로, 네트워크 범위 혼동이 없도록 호스트모드로 실행해준다.

# Proxy 실행 (Prometheus와 Agent 사이)
docker run --rm -d -p 50051:50051 -p 8080:8080 \
        --env ADMIN_ENABLED=false \
        --env METRICS_ENABLED=true \
        pambrose/prometheus-proxy:1.21.0
        # 에이전트의 request 수신: 50051 (grpc)
        # Prometheus의 request 수신: 8080 (http)
        # 관리자 포트(비활성화): 8082, 8092

# Agent 실행(Proxy와 Exporter사이) (로컬 설정파일 예)
docker run --rm -d -p --network host \
    --mount type=bind,source="$(pwd)"/localtest.conf,target=/app/prom-agent.conf \
    --env AGENT_CONFIG=prom-agent.conf \
    pambrose/prometheus-agent:1.21.0
    # 관리자 포트(비활성화): 8083, 8093

# Agent 실행(Proxy와 Exporter사이) (URL 설정 예)
docker run --rm -d --network host \
        --env AGENT_CONFIG='https://raw.githubusercontent.com/pambrose/prometheus-proxy/master/examples/simple.conf' \
        pambrose/prometheus-agent:1.21.0
        # 관리자 포트(비활성화): 8083, 8093
```

## node-exporter만 실행

```sh
helm install my-exporter prometheus-community/kube-prometheus-stack --version 55.8.3 -f only_exporter.yaml

# WSL 등에서 마운트 문제 발생시 옵션 추가
--set "prometheus-node-exporter.hostRootFsMount.enabled=false"
```

## Prometheus만 실행

```sh
helm install my-prom prometheus-community/kube-prometheus-stack --version 55.8.3 -f only_prom.yaml
```

## exporter의 외부 노출 방법

- 기본적으로 `hostPort`가 활성화되어 있음
- HostPort를 설정은, Host(Node)의 특정 Port에 Container의 Port를 바인딩한다는 의미
- Service, Ingress와 별개 방식이며, 중간단계없이 Host(Node)의 Port를 통해 직접적으로 Container를 외부노출
  - `도커 네트워크 호스트 모드 or Native 앱 설치와 비슷한 효과`
- NodePort 서비스와 달리, 클러스터 내 다른 Node에서 포트가 공유되지 않음
- 일반적인 쿠버네티스 앱 배포에 적절치 않은 단점들이 많으나, exporter처럼 DaemonSet으로 배포되는 앱들에게 적절한 방식
