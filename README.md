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

TLS를 쓸꺼면 그에 필요한 인증 설정을하고, 안써도 그에 따른 설정이 필요할 듯 하다.
