---
name: ocp-platform
description: OCP 플랫폼 컴포넌트 배포 스킬. GitLab Operator 설치/구성, GitLab CI Runner, ArgoCD GitOps 배포, Quay Container Registry 구성, vLLM 빌드+배포 CI/CD 파이프라인 생성. "GitLab 설치", "ArgoCD", "Quay 레지스트리", "CI/CD 파이프라인", "GitLab CI yaml", "GitOps" 관련 요청 시 반드시 이 스킬을 사용할 것.
---

# OCP Platform Skill

OCP OperatorHub 기반 플랫폼 컴포넌트 배포. GitLab → ArgoCD → Quay 순서로 구성한다.

## 환경 전제

```
클러스터 API: https://api.ocp.cpf.com:6443
Ingress: *.apps.ocp.cpf.com
Worker: 10.1.10.179, 10.1.10.180
```

## 구성 원칙

- OCP OperatorHub 우선 — Helm 차트 대신 Operator 방식
- 외부 노출은 OpenShift Route 사용 (Ingress 리소스 금지)
- ArgoCD: 선언적 GitOps — kubectl apply 직접 실행 금지
- Phase 2 전환: base_url만 변경되도록 API 엔드포인트 추상화

## GitLab (OperatorHub)

```yaml
# 1. GitLab Operator 설치
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: gitlab-operator-kubernetes
  namespace: gitlab-system
spec:
  channel: stable
  name: gitlab-operator-kubernetes
  source: certified-operators
  sourceNamespace: openshift-marketplace
```

GitLab 인스턴스 구성:
- 소스 코드 + 프롬프트 템플릿 버전 관리
- 모델 학습 데이터셋 LFS 저장
- Private Runner (GPU 작업 분리)
- GitLab AI Duo 활성화 검토

## ArgoCD (OpenShift GitOps Operator)

```bash
oc apply -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml -n argocd
```

Application 매니페스트 구조:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ai-platform
  namespace: argocd
spec:
  source:
    repoURL: https://gitlab.apps.ocp.cpf.com/ai-platform/manifests.git
    targetRevision: main
    path: manifests/
  destination:
    server: https://kubernetes.default.svc
    namespace: ai-platform
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## Quay (Container Registry)

- OCP OperatorHub → Red Hat Quay Operator
- Pull Secret 연동 (Bastion 미러 레지스트리 → Quay 내부 레지스트리)
- vLLM 이미지 저장 경로: `registry.ocp.cpf.com/ai-platform/vllm`

## CI/CD 파이프라인 (GitLab CI)

vLLM 이미지 빌드 + OCP 배포 파이프라인:

```yaml
stages:
  - build
  - push
  - deploy

variables:
  REGISTRY: registry.ocp.cpf.com
  IMAGE: $REGISTRY/ai-platform/vllm:$CI_COMMIT_SHA

build:
  stage: build
  script:
    - docker build -t $IMAGE .

push:
  stage: push
  script:
    - docker push $IMAGE

deploy:
  stage: deploy
  script:
    - argocd app sync ai-platform --revision $CI_COMMIT_SHA
  environment:
    name: production
```

## Phase 2 전환 포인트

```python
# Phase 1 (외부 API)
client = OpenAI(api_key="sk-...", base_url="https://api.openai.com/v1")

# Phase 2 (내부 vLLM — URL만 변경)
client = OpenAI(api_key="internal", base_url="https://ai-gateway.ocp.cpf.com/v1")
```

> CI/CD 파이프라인에 `AI_API_BASE_URL` 환경변수로 추상화하여 설정만으로 전환 가능하게 구성한다.

> 상세 GitLab 구성: `references/gitlab-config.md`
> ArgoCD 패턴: `references/argocd-patterns.md`
