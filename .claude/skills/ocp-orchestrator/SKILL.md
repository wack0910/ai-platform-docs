---
name: ocp-orchestrator
description: OCP AI 플랫폼 배포 오케스트레이터. infra-engineer/platform-engineer/ai-engineer/observability-engineer 에이전트 팀을 조율하여 전체 배포 워크플로우를 실행. "OCP 배포 시작", "플랫폼 구축", "전체 설치", "하네스 실행", "AI 플랫폼 배포", "배포 재실행", "배포 업데이트", "특정 단계만 다시", "이전 결과 기반으로 수정", "Week N 작업" 관련 요청 시 반드시 이 스킬을 사용할 것.
---

# OCP Platform Orchestrator

infra-engineer → platform-engineer (병렬) observability-engineer → ai-engineer 순서로 에이전트 팀을 조율한다.

**실행 모드:** 하이브리드
- Phase 1 (인프라): 에이전트 팀 — 순차 의존성
- Phase 2~3 (플랫폼+관측성): 에이전트 팀 — 병렬 팬아웃
- Phase 4 (AI 스택): 에이전트 팀 — GPU Worker 준비 후 순차

## Phase 0: 컨텍스트 확인

실행 전 기존 산출물 존재 여부를 확인한다:

```
_workspace/ 존재 + 부분 수정 요청 → 부분 재실행 (해당 에이전트만)
_workspace/ 존재 + 새 입력 제공  → _workspace_prev/ 로 이동 후 새 실행
_workspace/ 미존재               → 초기 실행
```

사용자 요청에서 다음을 파악한다:
- 전체 배포 vs 특정 Week/Phase만
- 특정 컴포넌트 재생성 요청
- GPU Worker 준비 여부 (10.1.10.181)

## Phase 1: 인프라 (Week 1~2)

**실행 모드:** 에이전트 팀 (순차)

infra-engineer에게 작업 지시:
1. Bastion TLS 인증서 + SSH 키
2. DNS(bind) 설정 — `ocp-infra` 스킬 사용
3. HAProxy 설정 — `ocp-infra` 스킬 사용
4. Pull Secret 미러링
5. install-config.yaml + Ignition 생성
6. 노드별 설치 명령어 생성 (Bootstrap → Master 1-3 → Worker 1-2)

완료 조건:
- `_workspace/infra/` 에 모든 설정 파일 저장
- 노드별 설치 명령어 (`node-install-cmds.sh`) 생성
- infra-engineer가 완료 SendMessage 발송

## Phase 2: 플랫폼 + 관측성 (Week 3~6)

**실행 모드:** 에이전트 팀 (병렬 팬아웃)

platform-engineer와 observability-engineer를 동시에 시작:

```
platform-engineer:
  - GitLab Operator + Runner  ← ocp-platform 스킬
  - ArgoCD GitOps 구성        ← ocp-platform 스킬
  - Quay Container Registry   ← ocp-platform 스킬
  - CI/CD 파이프라인 (vLLM)   ← ocp-platform 스킬

observability-engineer:
  - Prometheus/Grafana         ← ocp-observability 스킬
  - EFK Stack                  ← ocp-observability 스킬
  - Vault 설치 + 시크릿 마이그레이션 ← ocp-observability 스킬
  - Istio Service Mesh         ← ocp-observability 스킬
```

두 에이전트 모두 완료 SendMessage 수신 후 Phase 3으로 진행.

## Phase 3: AI 스택 (Week 7~8, GPU Worker 준비 시)

**실행 모드:** 에이전트 팀 (순차)

> GPU Worker(10.1.10.181)가 OCP에 조인되지 않은 경우: 설정 파일만 생성하고 적용 대기.

ai-engineer에게 작업 지시:
1. NVIDIA GPU Operator + DCGM — `ocp-ai` 스킬
2. vLLM 배포 (개발: Mistral 7B → 운영: Llama 3.1 70B INT4)
3. llm-d 설치 및 라우팅 구성
4. OpenAI 호환 API Route 노출
5. MLflow Model Registry
6. Phase 2 마이그레이션 체크리스트 생성

## Phase 4: 통합 검증

각 에이전트로부터 결과 수집 후:

1. DNS 해석 검증: `dig api.ocp.cpf.com @10.1.10.174`
2. OCP API 접근: `oc get nodes`
3. GitLab/ArgoCD Route 접근 확인
4. vLLM API 테스트: `curl https://ai-gateway.ocp.cpf.com/v1/models`
5. GPU 메트릭 수집 확인: Grafana DCGM 대시보드

## 산출물 구조

```
_workspace/
├── infra/
│   ├── dns/forward.zone, reverse.zone, named.conf
│   ├── haproxy/haproxy.cfg
│   └── ignition/install-config.yaml, node-install-cmds.sh
├── platform/
│   ├── gitlab/
│   ├── argocd/
│   ├── quay/
│   └── cicd/gitlab-ci.yml
├── observability/
│   ├── prometheus/
│   ├── grafana/
│   ├── efk/
│   ├── vault/
│   └── istio/
└── ai-stack/
    ├── gpu-operator/
    ├── vllm/
    ├── llm-d/
    ├── mlflow/
    └── phase2-migration-checklist.md
```

## 에러 핸들링

- 에이전트 실패 시: 1회 재시도 후 실패 컴포넌트 목록과 함께 사용자에게 보고
- GPU Worker 미준비: AI 스택 설정 파일만 생성, 적용 시점 명시
- 병렬 에이전트 중 하나 실패: 다른 에이전트 계속 진행, 실패 항목 최종 보고서에 명시

## 테스트 시나리오

**정상 흐름:**
```
"Week 1~2 인프라 작업해줘"
→ Phase 0: _workspace 없음 → 초기 실행
→ Phase 1: infra-engineer 시작
→ _workspace/infra/ 산출물 생성
→ 노드별 설치 명령어 제공
```

**에러 흐름:**
```
"vLLM 배포 설정만 다시 만들어줘"
→ Phase 0: _workspace 존재 → 부분 재실행
→ ai-engineer만 호출
→ _workspace/ai-stack/vllm/ 갱신
```
