# AI Platform Docs

## 하네스: OCP AI 플랫폼 배포

**목표:** OCP UPI 설치 + H100 AI 추론 스택 구축을 에이전트 팀으로 자동화

**트리거:** OCP 설치, 플랫폼 컴포넌트 배포, AI 스택 구성, 관측성 구성, Phase 2 전환 등 플랫폼 관련 작업 요청 시 `ocp-orchestrator` 스킬을 사용하라. 단순 질문은 직접 응답 가능.

**변경 이력:**
| 날짜 | 변경 내용 | 대상 | 사유 |
|------|----------|------|------|
| 2026-04-16 | 초기 하네스 구성 | 전체 | ai-platform-architecture.md 기반 신규 구축 |
| 2026-04-16 | 이중 네트워크 분리 반영 | ocp-infra 스킬, ocp-deploy 스킬, architecture.md, nodes-config.env | 외부 IP 절약 — Bastion만 10.x, 나머지 노드 192.168.0.x |
