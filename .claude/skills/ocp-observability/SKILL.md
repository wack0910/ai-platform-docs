---
name: ocp-observability
description: OCP 및 H100 AI 플랫폼 관측성 스택 구성 스킬. Prometheus/Grafana GPU 대시보드, EFK 스택(AI 추론 로그), HashiCorp Vault 시크릿 관리, Istio 서비스 메시, Jaeger 트레이싱, DCGM GPU 모니터링. "Prometheus 설정", "Grafana 대시보드", "EFK", "Vault 설치", "Istio 서비스 메시", "GPU 모니터링", "로그 집계", "카나리 배포" 관련 요청 시 반드시 이 스킬을 사용할 것.
---

# OCP Observability Skill

OCP 기본 모니터링 스택을 기반으로 AI 플랫폼 전용 관측성을 구성한다.

## 구성 순서

1. Prometheus/Grafana (OCP 기본 활성화 + GPU 메트릭 추가)
2. EFK Stack (AI 추론 로그 집계)
3. Vault (시크릿 중앙화)
4. Istio (서비스 메시 + 트래픽 관리)
5. Jaeger (AI 요청 트레이싱)

## Prometheus + Grafana (GPU 메트릭)

OCP 기본 모니터링 활성화:
```bash
oc apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    enableUserWorkload: true
EOF
```

DCGM ServiceMonitor 추가:
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: dcgm-exporter
  namespace: ai-platform
spec:
  selector:
    matchLabels:
      app: dcgm-exporter
  endpoints:
    - port: metrics
      interval: 15s
```

GPU 알림 규칙:
```yaml
# GPU 메모리 90% 초과 시 알림
- alert: GPUMemoryHigh
  expr: DCGM_FI_DEV_FB_USED / DCGM_FI_DEV_FB_TOTAL > 0.9
  for: 5m
  annotations:
    summary: "H100 메모리 사용량 90% 초과"

# GPU 온도 80°C 초과 시 알림
- alert: GPUTemperatureHigh
  expr: DCGM_FI_DEV_GPU_TEMP > 80
  for: 2m
```

## EFK Stack (AI 추론 로그)

```yaml
# Elasticsearch + Fluentd + Kibana
# OpenShift Logging Operator 사용
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: cluster-logging
  namespace: openshift-logging
spec:
  channel: stable
  name: cluster-logging
  source: redhat-operators
```

vLLM/llm-d 로그 파싱 규칙 (Fluentd):
```ruby
<filter ai-platform.**>
  @type parser
  key_name log
  <parse>
    @type json
    time_key timestamp
    time_format %Y-%m-%dT%H:%M:%S
  </parse>
</filter>
```

## HashiCorp Vault

```bash
# Helm으로 설치
helm install vault hashicorp/vault \
  --namespace vault \
  --set "server.ha.enabled=false" \
  --set "server.route.enabled=true" \
  --set "server.route.host=vault.apps.ocp.cpf.com"

# OCP Secrets Engine 활성화
vault auth enable kubernetes
vault secrets enable -path=ai-platform kv-v2
```

마이그레이션 대상 시크릿:
- AI API 키 (Anthropic, OpenAI) → `ai-platform/external-apis`
- TLS 인증서 → `ai-platform/tls`
- Pull Secret → `ai-platform/registry`
- HuggingFace 토큰 → `ai-platform/huggingface`

## Istio (OpenShift Service Mesh)

```bash
# OpenShift Service Mesh Operator 설치
oc apply -f servicemesh-subscription.yaml
```

AI 트래픽 카나리 배포:
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: vllm-canary
spec:
  hosts:
    - vllm-service
  http:
    - route:
        - destination:
            host: vllm-service
            subset: stable
          weight: 90
        - destination:
            host: vllm-service
            subset: canary
          weight: 10
```

## Jaeger (AI 요청 트레이싱)

```bash
# OpenShift Distributed Tracing Platform Operator
oc apply -f jaeger-subscription.yaml
```

vLLM 요청 트레이싱: 각 `/v1/chat/completions` 요청에 trace-id를 자동 부여하여 응답 지연 분석.

> 상세 Grafana 대시보드 설정: `references/grafana-dashboards.md`
> Vault 운영 가이드: `references/vault-operations.md`
