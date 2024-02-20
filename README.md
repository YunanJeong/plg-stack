# PLG Stack

Promtail, Loki, Grafana Stack
클러스터 로그 모니터링용

## 설치

[loki-stack 사용](https://artifacthub.io/packages/helm/grafana/loki-stack/2.9.12)

```sh
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# (오프라인 환경 등)아카이브 파일 필요시
helm pull grafana/loki-stack --version 2.9.12

helm install my-loki-stack grafana/loki-stack --version 2.9.12
```

## 설정

```sh
# 릴리즈 이름은 서비스 분류명을 붙여주자.
helm install eks-loki grafana/loki-stack --version 2.9.12 -f my_value.yaml
helm upgrade eks-loki grafana/loki-stack --version 2.9.12 -f my_value.yaml
```

## 간단한 대시보드

https://grafana.com/grafana/dashboards/13639-logs-app/
https://grafana.com/grafana/dashboards/15141-kubernetes-service-logs/

## retention 설정 (로그삭제정책)

- default는 무제한 저장
- 로키 설정파일 loki.yaml에서 변경가능하며, 차트 value에선 `loki.config` 아래 내용이 그대로 내부 loki.yaml로 사용됨


### Compactor와 Table Manager