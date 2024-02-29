# prom-proxy-test

Prometheus Proxy beyond firewall

```sh

# 아래 명령어 대로 실행시 콘솔실행됨.
  # 도커 인터랙티브(콘솔)모드는 아닌데, 이미지 내부 entrypoint와 자바앱이 그렇게 설정된듯
  # 라이브 실행시 -d 옵션(백그라운드 모드)을 명시적으로 활성화해주도록 하자

# 프록시 실행 (Prometheus-proxy와 모니터링 대상 사이)
docker run --rm -p 8082:8082 -p 8092:8092 -p 50051:50051 -p 8080:8080 \
        --env ADMIN_ENABLED=true \
        --env METRICS_ENABLED=true \
        pambrose/prometheus-proxy:1.21.0

# 에이전트 실행(Prometheus와 Prometheus-proxy Agent 사이)
docker run --rm -p 8083:8083 -p 8093:8093 \
        --env AGENT_CONFIG='https://raw.githubusercontent.com/pambrose/prometheus-proxy/master/examples/simple.conf' \
        pambrose/prometheus-agent:1.21.0

# 에이전트 실행(설정을 URL 대신 로컬파일로 하기)
docker run --rm -p 8083:8083 -p 8093:8093 \
    --mount type=bind,source="$(pwd)"/prom-agent.conf,target=/app/prom-agent.conf \
    --env AGENT_CONFIG=prom-agent.conf \
    pambrose/prometheus-agent:1.21.0
```

## gRPC 설정

prometheus-proxy는 agent에서 proxy로 request하는 방향으로 gRPC를 이용해 통신(port 50051)한다.

gRPC에 대한 추가설정이 필요

TLS를 쓸꺼면 그에 필요한 인증 설정을하고, 쓰지 않더라도 관련 설정이 필요할 듯 하다.

## node-exporter만 실행

helm install my-exporter prometheus-community/kube-prometheus-stack --version 55.8.3 -f only_exporter.yaml


helm upgrade my-exporter prometheus-community/prometheus-node-exporter --version 4.30.3  --set "hostRootFsMount.enabled=false"  \
--set "service.type=NodePort"

```yaml
hostRootFsMount:
  enabled: false   # default: true
```

## Prometheus만 실행

helm install my-prom prometheus-community/kube-prometheus-stack --version 55.8.3 -f only_prom.yaml

## exporter의 외부 노출 방법

- 기본적으로 `hostPort`가 활성화되어 있음
- HostPort를 설정은, Host(Node)의 특정 Port에 Container의 Port를 바인딩한다는 의미
- Service, Ingress와 별개 방식이며, 중간단계없이 Host(Node)의 Port를 통해 직접적으로 Container를 외부노출
  - `도커 네트워크 호스트 모드 or Native 앱 설치와 비슷한 효과`
- NodePort 서비스와 달리, 클러스터 내 다른 Node에서 포트가 공유되지 않음
- 일반적인 쿠버네티스 앱 배포에 적절치 않은 단점들이 많으나, exporter처럼 DaemonSet으로 배포되는 앱들에게 적절한 방식
