# PLG Stack

Promtail, Loki, Grafana Stack
로그 모니터링용

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

<https://grafana.com/grafana/dashboards/13639-logs-app/>
<https://grafana.com/grafana/dashboards/15141-kubernetes-service-logs/>

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
  - 추가: TableManager는 Deprecated

### 최소 설정으로 빠르게 retention 적용방법

```yaml
loki:
  config:
    compactor:
      retention_enabled: true   # false면, 무제한 저장
    limits_config:
      retention_period: 744h  # default: 744h(31일), 최소 24h
```

## 수집주기 Default 값

- promtail이 로그수집하는 주기(scrape_interval): 10s
- promtail이 Loki로 보내는 주기: 1s (단, batch buffer에 영향 받음)
- 위 두 항목 모두 promtail에서 설정함.
- 상위차트, 하위차트 helm value파일에서 모두 제어 가능하며, 실행중인 컨테이너에선 `/etc/promtail/promtail.yaml`에서 확인가능
- 별도로 설정하지 않을 시, promtail.yaml에서도 미기입되어있으므로, 위 default 값을 참고하면 됨

## Grafana에서 Loki 패널을 여러 개 사용시, too many outstanding requests 오류 해결

```yaml
loki:
  config:
    querier:
      max_concurrent: 2048
    query_scheduler:
      max_outstanding_requests_per_tenant: 2048
```

## PVC, PV 관련 설정

- `loki-stack` 차트의 default values.yaml에는 없으나, 각 서브차트에서 지원

### Grafana

```yaml
grafana:
  persistence:
    type: statefulset  # helm uninstall 해도 남아있게 하려면 statefulset 필수. default 설정 아님.
    enabled: true
```

### Loki

- Loki는 PV프로비저닝의 두 유형에 따라 values.yaml 설정방법을 기술한다.
- 동적 프로비저닝
  - storageClass(SC)를 사용하여 PVC의 요구 사항을 정의
  - PVC를 만들 때 SC를 지정하면, K8s는 해당 SC에 정의된 Provisioner(kube-system)가 자동으로 PV를 생성하고 할당
- 정적 프로비저닝
  - 관리자가 미리 PV를 생성하고 필요한 속성을 설정
  - PVC(Persistent Volume Claim)를 만들 때 사용 가능한 PV 목록에서 원하는 PV를 선택하여 할당

#### Loki PV 동적 프로비저닝 방법

```yaml
# 동적 프로비저닝 (storageClass, provisioner기반으로 PVC,PV를 helm install시 자동생성)
loki:
  persistence:
    enabled: true
    size: 100Gi
    storageClassName: gp2  # default stroageClass 사용시 이 섹션을 삭제
```

- K8s의 PVC 매니페스트에서 원래 동작은 다음과 같다.
  - `storageClassName=""`(빈따옴표 할당): storageClass 미선택으로 정적 프로비저닝
  - `storageClassName 미기입`: default stroageClass 선택되어 동적 프로비저닝
- `loki-stack 차트는` PVC템플릿에서 이를 구분하지 못해서 `둘 다 default storageClass 선택으로 취급`되어 동적 프로비저닝이 강제된다. 따라서, 정적 프로비저닝 하려면 PVC와 PV를 미리 수동생성해야 한다.

#### Loki PV 정적 프로비저닝 방법

- `PV의 실제 로컬저장경로 변경시 정적 프로비저닝`을 쓸 수 있다.
  - 동적 프로비저닝에서 로컬저장경로를 변경하려면, storageClass와 provisioner 둘 다 수정해야 하는데, 특히 provisioner를 수정하고 재설치하기가 번거롭다.
  - provisioner는 클러스터 내 다른 App.들에게 영향을 미칠 수 있기 때문에 조심해야 한다.
- loki-stack 차트엔 PVC템플릿이 있지만, 정적 프로비저닝에 사용할 수 없다. 앞서 언급한 이유로 동적 프로비저닝이 강제되기 때문이다.
- 따라서, PVC와 PV 둘 다 미리 생성해주도록 한다.

```sh
# PVC와 PV 사전 생성
kubectl apply -f loki-pvc.yaml
```

```yaml
# 정적 프로비저닝 (provisioner를 사용하지 않고 기존 PVC, PV를 사용)
loki:
  persistence:
    enabled: true
    existingClaim: loki-pv-claim
```
