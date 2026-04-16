---
name: ocp-ai
description: H100 AI 추론 스택 배포 스킬. NVIDIA GPU Operator 설치, DCGM 모니터링, vLLM 배포(PagedAttention/INT4), llm-d 분산 추론 오케스트레이터, MLflow Model Registry, OpenAI 호환 API 엔드포인트 노출, Phase 2 마이그레이션 체크리스트 생성. "GPU Operator", "vLLM 배포", "llm-d 설정", "모델 서빙", "AI 추론", "H100 설정", "Phase 2 전환" 관련 요청 시 반드시 이 스킬을 사용할 것.
---

# OCP AI Stack Skill

H100 단일 노드(10.1.10.181, 5월 추가)에 AI 추론 스택을 배포한다.

## 환경 정보

```
GPU Worker: 10.1.10.181 (H100 80GB × 1)
네트워크: InfiniBand / 10GbE
OCP 네임스페이스: ai-platform
AI API: https://ai-gateway.ocp.cpf.com/v1
```

## GPU Operator 설치 (OperatorHub)

```bash
# NVIDIA GPU Operator via OCP OperatorHub
oc apply -f gpu-operator-subscription.yaml

# Node Feature Discovery 먼저 설치 필요
oc apply -f nfd-subscription.yaml
```

GPU Worker 노드 Taint (AI 워크로드 전용):
```bash
oc adm taint nodes worker3.ocp.cpf.com nvidia.com/gpu=present:NoSchedule
```

DCGM Exporter 활성화:
```yaml
apiVersion: nvidia.com/v1
kind: ClusterPolicy
metadata:
  name: gpu-cluster-policy
spec:
  dcgmExporter:
    enabled: true
```

## vLLM 배포

H100 단일 노드 최적화 설정:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm-server
  namespace: ai-platform
spec:
  replicas: 1
  template:
    spec:
      nodeSelector:
        nvidia.com/gpu.present: "true"
      tolerations:
        - key: nvidia.com/gpu
          operator: Exists
      containers:
        - name: vllm
          image: registry.ocp.cpf.com/ai-platform/vllm:latest
          args:
            - --model=meta-llama/Llama-3.1-70B-Instruct
            - --quantization=awq          # INT4 양자화 (~35GB)
            - --max-model-len=8192
            - --tensor-parallel-size=1    # 단일 H100
          resources:
            limits:
              nvidia.com/gpu: "1"
          env:
            - name: HUGGING_FACE_HUB_TOKEN
              valueFrom:
                secretKeyRef:
                  name: hf-token
                  key: token
```

### 권장 모델 (H100 80GB 단일)

| 단계 | 모델 | 크기 | 비고 |
|------|------|------|------|
| 개발 | Mistral 7B | ~4GB | 빠른 응답 |
| 개발 | CodeLlama 13B | ~7GB | 코딩 특화 |
| 운영 | Llama 3.1 70B INT4 | ~35GB | 범용 |
| 운영 | Qwen2.5 72B INT4 | ~36GB | 다국어 |

## llm-d 설정

```yaml
# H100 단일 노드 최적화
apiVersion: inference.llm-d.ai/v1alpha1
kind: InferenceModel
metadata:
  name: llama3-70b
spec:
  modelName: meta-llama/Llama-3.1-70B-Instruct
  routing:
    strategy: least-loaded
  caching:
    kvCacheSharing: true
    maxCacheSize: "20Gi"
```

## OpenAI 호환 API 노출

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: ai-gateway
  namespace: ai-platform
spec:
  host: ai-gateway.ocp.cpf.com
  to:
    kind: Service
    name: vllm-service
  port:
    targetPort: 8000
  tls:
    termination: edge
```

테스트:
```bash
curl https://ai-gateway.ocp.cpf.com/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "meta-llama/Llama-3.1-70B-Instruct", "messages": [{"role": "user", "content": "Hello"}]}'
```

## MLflow Model Registry

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mlflow
  namespace: ai-platform
spec:
  template:
    spec:
      containers:
        - name: mlflow
          image: ghcr.io/mlflow/mlflow:latest
          args:
            - mlflow
            - server
            - --host=0.0.0.0
            - --backend-store-uri=postgresql://mlflow:password@postgres/mlflow
            - --default-artifact-root=s3://mlflow-artifacts
```

## Phase 2 마이그레이션 체크리스트

생성 파일: `_workspace/ai-stack/phase2-migration-checklist.md`

> 상세 vLLM 튜닝: `references/vllm-optimization.md`
> llm-d 고급 설정: `references/llm-d-config.md`
