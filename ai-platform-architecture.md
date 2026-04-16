# AI 플랫폼 아키텍처 설계 문서

> **목적**: 이 문서는 Claude AI가 본 프로젝트의 컨텍스트를 유지하며 일관된 방식으로 지원할 수 있도록 작성되었습니다.
> 새 대화를 시작할 때 이 파일을 첨부하거나 내용을 붙여넣어 사용하세요.

---

## 1. 프로젝트 개요

| 항목 | 내용 |
|------|------|
| 목표 | 사내 AI 플랫폼 구축 (개발환경 → OCP → H100 GPU 서버) |
| 단계 | Phase 1: 외부 AI 활용 구축 → Phase 2: 내부 AI 모델로 전환 |
| 현재 상태 | 아이디어/설계 단계, OCP 서버 미구축 |
| 개발 방식 | AI-Assisted (Claude Console + 외부 AI API 최대 활용) |

---

## 2. 전체 아키텍처 (3계층)

```
┌──────────────────────────────────────────────────────┐
│              개발 환경 (Dev Environment)               │
│  Desktop/VM OS · Claude Console · 외부 AI API · VPN  │
└──────────────────────┬───────────────────────────────┘
                       │ VPN / TLS
┌──────────────────────▼───────────────────────────────┐
│          OCP (OpenShift Container Platform)           │
│                                                       │
│  [API Gateway]  ←── 모든 트래픽 진입점                │
│       │                                               │
│  ┌────▼──────┐  ┌──────────┐  ┌──────────────────┐   │
│  │  GitLab   │  │  CI/CD   │  │Container Registry│   │
│  └───────────┘  └──────────┘  └──────────────────┘   │
│                                                       │
│  Prometheus/Grafana · Vault · Service Mesh · EFK      │
└──────────────────────┬───────────────────────────────┘
                       │ InfiniBand / 10GbE
┌──────────────────────▼───────────────────────────────┐
│              H100 × 1 AI 플랫폼                       │
│                                                       │
│  llm-d · vLLM/TGI · Model Store · GPU 모니터링        │
│  Model Registry · OpenAI 호환 API                    │
└──────────────────────────────────────────────────────┘
```

---

## 3. 계층별 상세 설계

### 3-1. 개발 환경

```yaml
OS: Desktop 또는 VM (Windows/Linux)
도구:
  - Claude Console (Claude Code CLI): AI 기반 코딩 어시스턴트
  - 외부 AI API: Anthropic API, OpenAI API (Phase 1 한정)
  - VPN Client: 내부 OCP 네트워크 접근
  - IaC: Terraform (OCP 리소스), Ansible (서버 구성)
  - IDE: VSCode + Continue.dev (로컬 AI 어시스턴트)
```

**AI 활용 방식 (Phase 1)**:
- Claude Console로 IaC 코드 자동 생성
- 외부 AI API로 설계 검토 및 코드 리뷰
- VPN 통해 OCP API에 직접 접근

---

### 3-2. OCP (OpenShift Container Platform)

#### API Gateway
```yaml
권장 솔루션: Red Hat 3scale API Management 또는 Kong Gateway
역할:
  - 개발 환경 ↔ OCP 내부 통신 단일 진입점
  - 인증/인가 (JWT, OAuth2)
  - Rate Limiting (H100 GPU 보호)
  - 요청 라우팅 (GitLab, CI/CD, AI API)
  - TLS 종료
설정 우선순위: UPI 설치 시 가장 먼저 구성
```

#### GitLab (Self-hosted)
```yaml
설치 방식: GitLab Operator (OCP OperatorHub)
구성:
  - 소스 코드 관리
  - AI 관련 프롬프트 템플릿 버전 관리
  - 모델 학습 데이터셋 LFS 저장
  - GitLab CI Runner (OCP 내부 실행)
보완 사항:
  - GitLab AI 기능(Duo) 활성화 검토
  - Private runner로 GPU 작업 분리
```

#### CI/CD Pipeline
```yaml
도구 스택:
  - GitLab CI: 빌드/테스트 파이프라인
  - ArgoCD: GitOps 기반 OCP 배포
  - Tekton: 컨테이너 네이티브 파이프라인 (선택)
파이프라인 단계:
  1. 코드 커밋 (개발 환경 → GitLab)
  2. 자동 빌드 + 컨테이너 이미지 생성
  3. Container Registry 푸시 (Quay.io 또는 내부 레지스트리)
  4. ArgoCD 자동 배포 (H100 서버 포함)
  5. 배포 후 모델 상태 확인
AI 자동화:
  - Claude로 Dockerfile, CI yaml 자동 생성
  - PR 리뷰 자동화 (GitLab AI Duo 또는 외부 API)
```

#### 관측성 스택
```yaml
메트릭: Prometheus + Grafana (OCP 기본 포함)
로그: EFK Stack (Elasticsearch + Fluentd + Kibana)
트레이싱: Jaeger (AI 요청 추적)
시크릿: HashiCorp Vault (모델 API 키, 인증 정보)
네트워크: Istio Service Mesh (AI 트래픽 관리, 카나리 배포)
```

---

### 3-3. H100 AI 플랫폼

#### 설치 방식 권장: UPI (User Provisioned Infrastructure)
```yaml
이유:
  - H100 GPU 서버를 직접 제어 필요
  - 커스텀 네트워크/스토리지 구성
  - GPU Operator 설정 자유도
  - AI가 Terraform/Ansible로 자동화하기 용이
절차:
  1. RHCOS 수동 설치
  2. OCP Cluster 조인
  3. GPU Operator 설치 (NVIDIA)
  4. AI 플랫폼 컴포넌트 배포
```

#### AI 추론 스택
```yaml
llm-d:
  역할: 분산 LLM 추론 오케스트레이터
  특징: KV Cache 공유, 다중 모델 라우팅
  H100 단일 노드 최적화 설정 필요

vLLM / TGI (Text Generation Inference):
  역할: 고성능 LLM 추론 서버
  권장: vLLM (PagedAttention으로 메모리 효율화)
  API: OpenAI 호환 엔드포인트 자동 제공

권장 모델 (H100 80GB 단일 기준):
  - Llama 3.1 70B (INT4 양자화 시 ~35GB)
  - Mistral 7B / Mixtral 8x7B
  - Qwen2.5 72B (INT4)
  - CodeLlama 34B (코딩 특화)
  ※ 개발 단계: 7B~13B 경량 모델 권장
```

#### 모델 관리
```yaml
Model Registry:
  - MLflow 또는 Hugging Face Hub (로컬 미러)
  - 모델 버전 관리, A/B 테스트
GPU 모니터링:
  - DCGM Exporter → Prometheus
  - GPU 사용률, 메모리, 온도, 전력
OpenAI 호환 API:
  - /v1/chat/completions 엔드포인트
  - 개발 환경에서 동일 코드로 외부→내부 전환 가능
```

---

## 4. Phase 전환 계획

### Phase 1: 외부 AI 활용 구축 단계
```
개발 환경 → 외부 AI API (Anthropic/OpenAI)
           → OCP 구성 자동화 (IaC)
           → GitLab / CI/CD 구축
           → H100 환경 구축
```

### Phase 2: 내부 AI 전환 단계
```
외부 AI API 차단
내부 vLLM/llm-d 엔드포인트로 전환
코드 변경 최소화 (OpenAI 호환 API 덕분에 URL만 변경)
```

**전환 시 변경 사항**:
```python
# Phase 1
client = OpenAI(api_key="...", base_url="https://api.openai.com")

# Phase 2 (URL만 변경)
client = OpenAI(api_key="internal", base_url="https://ai-gateway.ocp.internal/v1")
```

---

## 5. 보완된 아키텍처 요소

기존 구상에서 추가된 컴포넌트:

| 추가 요소 | 이유 |
|-----------|------|
| Vault (시크릿 관리) | AI API 키, 인증서 안전 관리 |
| Service Mesh (Istio) | AI 트래픽 카나리 배포, 관측성 |
| ArgoCD (GitOps) | H100 배포 자동화, 선언적 관리 |
| Model Registry | 모델 버전 관리, 롤백 지원 |
| GPU 모니터링 (DCGM) | H100 상태 조기 감지 |
| EFK Stack | AI 추론 로그 중앙 집계 |
| Container Registry | 내부 이미지 보안, 속도 개선 |

---

## 6. Claude와의 작업 방식

### 이 문서를 새 대화에서 사용하는 방법
```
1. 이 MD 파일을 Claude 대화에 첨부하거나 내용을 복사
2. 다음 프롬프트로 시작:
   "위 아키텍처 문서를 참고해서 [작업 내용]을 도와줘"
```

### 주요 작업별 Claude 활용 예시

**OCP UPI 설치 자동화**:
```
"OCP UPI 설치를 위한 Terraform 코드를 작성해줘.
H100 GPU 노드 1대, Master 3대, Infra 2대 구성이야."
```

**GitLab CI 파이프라인 생성**:
```
"vLLM Docker 이미지를 빌드하고 OCP에 배포하는
GitLab CI yaml 파일을 작성해줘."
```

**llm-d 설정**:
```
"H100 단일 노드에서 llm-d로 Llama3 70B를 INT4로
서빙하는 설정 파일을 만들어줘."
```

**Phase 2 전환 체크리스트**:
```
"외부 AI를 차단하고 내부 vLLM으로 전환할 때
확인해야 할 체크리스트를 만들어줘."
```

---

## 7. 구축 순서 (권장)

```
Week 1-2: OCP 환경 준비
  □ VPN 설정 및 내부 네트워크 접근 확인
  □ OCP UPI 설치 (Terraform + Ansible)
  □ API Gateway 구성 (3scale 또는 Kong)

Week 3-4: 개발 인프라
  □ GitLab Operator 설치 및 구성
  □ GitLab Runner 설정 (OCP 내부)
  □ Container Registry 구성 (Quay)
  □ ArgoCD 설치 및 GitOps 저장소 구성

Week 5-6: 관측성 스택
  □ Prometheus + Grafana (OCP 기본 활성화)
  □ EFK Stack 배포
  □ Vault 설치 및 시크릿 마이그레이션
  □ Istio Service Mesh 설치

Week 7-8: H100 AI 플랫폼
  □ NVIDIA GPU Operator 설치
  □ DCGM 모니터링 활성화
  □ vLLM 배포 (경량 모델로 시작)
  □ llm-d 설치 및 라우팅 구성
  □ OpenAI 호환 API 엔드포인트 노출
  □ Model Registry (MLflow) 구성

Week 9+: Phase 2 전환
  □ 외부 AI API 트래픽 점진적 전환
  □ 내부 AI 성능 검증
  □ 외부 API 차단 및 완전 전환
```

---

## 8. 참고 자료

- [OCP UPI 설치 공식 문서](https://docs.openshift.com/container-platform/latest/installing/installing_bare_metal/installing-bare-metal.html)
- [NVIDIA GPU Operator on OCP](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/openshift/contents.html)
- [llm-d GitHub](https://github.com/llm-d/llm-d)
- [vLLM 공식 문서](https://docs.vllm.ai)
- [GitLab Operator on OCP](https://docs.gitlab.com/operator/)
- [ArgoCD on OpenShift](https://argo-cd.readthedocs.io/en/stable/operator-manual/installation/)

---

*최종 업데이트: 설계 초안 v1.0*
*다음 단계: OCP UPI 설치 Terraform 코드 작성*