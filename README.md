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

## retention 정책 (Log Deletion)

- 미설정시 무제한 저장
- Loki의 설정파일(loki.yaml)에서 관리됨
- 헬름 value에선 `loki.config` 섹션값이 내부 loki.yaml에 반영됨
- Loki에서 retention 기능은 Compactor 또는 TableManager에 의해 수행됨
- 여러 출처 검토 결과, `Retention 기능만 필요하다면 Compactor로 하는 것이 적절`
  - Compactor의 주목적: 오래된 로그 처리정책 관리
  - Table Manager의 주목적: 전반적인 로그 저장정책 관리
  - Compactor로 Retention 적용시, TableManager는 필요없음 (공식)
  - [Loki Retention 공식 문서](https://grafana.com/docs/loki/latest/operations/storage/retention/)
  - [블로그 중 Loki Retention 설정 팁](https://nyyang.tistory.com/167)

### 최소 설정으로 빠르게 retention 적용방법

```yaml
loki:
  config:
    compactor:
      retention_enabled: true   # false면, 무제한 저장
    limits_config:
      retention_period: 744h  # default: 744h(31일), 최소 24h
```
