# Docker Container Monitor (lite-k8s)

## 프로젝트 개요
- **기간**: 2026.02 ~
- **인원**: 1인
- **역할**: 기획 / 설계 / 개발 / 테스트 전담
- **GitHub**: https://github.com/silicao3o/lite-k8s
- **한줄 소개**: Docker 컨테이너를 K8s 없이 K8s처럼 — 실시간 모니터링, 자가치유, AI 사후 분석, 자동 롤링 업데이트를 제공하는 Spring Boot 기반 경량 오케스트레이션 플랫폼

---

## 개발 계기

사내 서버 아키텍처가 전면 도커 컨테이너 기반으로 전환되면서, GCP VM 위에서 여러 서비스를 컨테이너로 서빙하는 방식으로 운영 환경이 바뀌었습니다. 그러나 쿠버네티스 없이 순수 Docker로만 관리하다 보니, 컨테이너 상태를 확인하려면 매번 직접 CLI 명령어를 입력해야 했습니다.

테스트 환경임에도 불구하고 컨테이너가 비정상 종료됐을 때 대응이 늦어지고 원인 파악이 불명확한 상황이 반복되었습니다. 이를 해결하기 위해 두 가지 방향을 검토했습니다.

- **쿠버네티스 도입**: 강력하지만 러닝커브와 인프라 비용이 현재 팀 규모와 맞지 않음
- **경량 모니터링 도구 직접 개발**: 우리 환경에 맞는 최소한의 기능만 구현, 즉시 적용 가능

학습 비용과 인프라 부담을 고려해 경량화 버전을 직접 만들기로 결정한 것이 이 프로젝트의 시작입니다.

---

## 기술스택 (Tech Stack)

### Backend
| 기술 | 버전 | 사용 목적 |
|------|------|----------|
| Java | 17 | 프로그래밍 언어 |
| Spring Boot | 3.2.3 | 애플리케이션 프레임워크 |
| Spring WebSocket | 3.2.3 | 실시간 컨테이너 상태 스트리밍 |
| Spring Security | 3.2.3 | 인증/인가 처리 |
| Spring Mail | 3.2.3 | 이메일 알림 전송 |
| docker-java | 3.3.x | Docker Engine API 연동 |
| JJWT | 0.11.5 | JWT 토큰 인증 |

### Frontend
| 기술 | 버전 | 사용 목적 |
|------|------|----------|
| Thymeleaf | 3.2.3 | 서버사이드 렌더링 |
| Chart.js | - | 메트릭 히스토리 시각화 |
| WebSocket (JS) | - | 실시간 대시보드 갱신 |

### 인프라 / DevOps
| 기술 | 버전 | 사용 목적 |
|------|------|----------|
| Docker | - | 컨테이너 런타임 |
| Docker Compose | - | 테스트 스택 구성 |
| GHCR | - | 컨테이너 이미지 레지스트리 |

### AI 연동
| 프로바이더 | 모델 | 용도 |
|------------|------|------|
| Anthropic | claude-haiku-4-5 | 장애 원인 분석 / 재발 방지 제안 |
| Google Gemini | gemini-2.0-flash | 동일 (대체 프로바이더) |
| OpenAI | gpt-4o-mini | 동일 (대체 프로바이더) |

---

## 아키텍처 개요

```
┌─────────────────────────────────────────────┐
│              Docker Monitor                  │
│                                             │
│  DockerEventListener  ──→  SelfHealingService│
│         │                       │           │
│         ↓                       ↓           │
│  LogAnalysisService      PlaybookExecutor   │
│  (AI 사후 분석)           (자동화 조치)      │
│         │                       │           │
│         ↓                       ↓           │
│  IncidentReport         SafetyGate          │
│  Repository             (Human-in-Loop)     │
│                                             │
│  ImageUpdatePoller  ──→  RollingUpdateService│
│  (GHCR 폴링)             (자동 업데이트)    │
│                                             │
│  HealthCheckScheduler ──→ 재시작 트리거     │
│  MetricsScheduler     ──→ WebSocket 브로드캐스트│
└─────────────────────────────────────────────┘
```

---

## 기능 개발 (Feature Development)

### Phase 1 — 기본 모니터링 & 알림
- **실시간 컨테이너 감시**: Docker Events API 스트리밍으로 `die`, `oom` 이벤트 즉시 감지
- **리소스 메트릭 수집**: CPU%, 메모리%, 네트워크 I/O 주기적 수집 및 WebSocket 브로드캐스트 (15초 주기)
- **이메일 알림**: HTML 템플릿 기반 상세 알림 (종료 코드, 마지막 로그, OOM 여부 포함)
- **알림 중복 방지**: 타임 윈도우 기반 deduplication (기본 30초)
- **로그 수집 & 검색**: 실시간 로그 스트리밍, 키워드/레벨/시간 범위 필터

### Phase 2 — 규칙 기반 자가치유 (Self-Healing)
- **K8s 스타일 자동 재시작**: YAML 규칙 또는 컨테이너 라벨 기반으로 충돌 시 자동 재시작
- **재시작 제한 & 리셋**: 최대 재시작 횟수, 리셋 윈도우, 지연 시간 설정
- **Playbook 시스템**: YAML로 정의된 이상 감지 시 실행 액션 시퀀스 (container-restart, oom-recovery 등)
- **Safety Gate**: 조치의 위험도(LOW/MEDIUM/HIGH/CRITICAL) 자동 산정, 고위험 조치 자동 차단
- **Human-in-the-Loop**: CRITICAL 위험도 조치 수동 승인 UI (`/approvals`), 5분 타임아웃
- **Audit Log**: 모든 조치의 의도·실행·결과 Append-Only 기록, 180일 보존

### Phase 3 — AI 사후 분석
- **멀티 AI 프로바이더**: Anthropic / Gemini / OpenAI 런타임 전환 (`/ai-settings`)
- **자동 분석 트리거**: 컨테이너 종료 이벤트 발생 시 로그·메트릭·컨텍스트를 AI에 전달
- **Incident Report**: Root Cause 분석 결과 및 재발 방지 제안 리포트 생성 (`/incidents`)
- **패턴 감지**: 반복 충돌 패턴 학습 및 설정 최적화 제안 (`/suggestions`)

### Phase 4 — 대시보드 고도화
- **멀티 컨테이너 비교 차트**: Chart.js 기반 CPU/메모리 추이 시각화
- **메트릭 집계 API**: 시간대별 평균/최대 집계, CSV 내보내기
- **자가치유 통계**: MTTR, 성공률, 컨테이너별 재시작 히스토리

### Phase 5 — K8s-like 오케스트레이션
- **GHCR 이미지 자동 업데이트**: 60초 주기로 이미지 digest 폴링 → 변경 감지 시 Rolling Update 자동 실행
- **Health Check Probe**: HTTP / TCP / Exec 프로브 지원, 연속 실패 시 자동 재시작 트리거
- **Desired State Reconciler**: 컨테이너 삭제 시 원래 환경변수·포트 그대로 자동 재생성
- **배포 전략**: Rolling Update / Blue-Green / Canary / Recreate 전략 REST API 제공
- **다중 노드 스케줄링**: 복수의 Docker 호스트에 컨테이너 배치, LEAST_USED 전략

---

## TDD 개발 방법론

Kent Beck의 TDD(Red → Green → Refactor) 사이클과 Tidy First 원칙을 준수하여 개발:

- **테스트 수**: 430개 (전 기능 단위 테스트)
- **커밋 규칙**: 구조적 변경(rename, extract)과 행동적 변경(기능 추가)을 별도 커밋으로 분리
- **커버리지 대상**: 서비스 레이어, 도메인 로직, 스케줄러, AI 클라이언트 파싱 전 영역

```
테스트 예시
SelfHealingServiceTest      — 재시작 횟수 제한, 리셋 윈도우, 라벨 우선순위
SafetyGateTest              — 위험도 산정, 서비스 중요도 가중치
GhcrClientTest              — OAuth2 토큰 교환, digest URL 빌드
HealthCheckSchedulerTest    — 프로브 타입별 실패 감지 및 재시작 트리거
ImageUpdatePollerTest       — digest 변경 감지 및 Rolling Update 연동
```

---

## 주요 기술적 도전

### 1. GHCR 인증 — OAuth2 토큰 교환 구현
**문제**: GitHub Container Registry는 PAT를 Bearer 토큰으로 직접 사용 불가 (HTTP 403)

**해결**: Docker Registry v2 프로토콜의 토큰 교환 플로우 구현
1. `https://ghcr.io/token?scope=repository:<owner>/<image>:pull` 엔드포인트에 Basic 인증으로 토큰 요청
2. 응답 받은 Bearer 토큰으로 manifest API 호출

```java
// GhcrClient.java
String exchangeToken(String imageRef) {
    String scope = "repository:" + repoPath + ":pull";
    String tokenUrl = "https://ghcr.io/token?service=ghcr.io&scope=" + scope;
    String basic = Base64.getEncoder().encodeToString((owner + ":" + pat).getBytes());
    // ... token exchange request
}
```

### 2. Docker Compose 네트워크에서 컨테이너 IP 처리
**문제**: Docker Compose 환경에서 `getNetworkSettings().getIpAddress()` 가 빈 문자열 반환 → `StringIndexOutOfBoundsException`

**해결**: IP가 빈 문자열일 경우 컨테이너 이름을 DNS 호스트로 폴백 (Docker 내부 DNS 활용)

### 3. 자가치유 Restart Count 동시성 처리
**문제**: 빠른 연속 충돌 시 재시작 카운트 경쟁 조건 발생 가능

**해결**: `ConcurrentHashMap` + `AtomicInteger` 기반 상태 관리로 thread-safe 카운팅

### 4. AI 응답 파싱 — 다중 프로바이더 통합
**문제**: Anthropic / Gemini / OpenAI 응답 형식이 모두 다름

**해결**: `AiResponseParser`로 JSON/마크다운 혼합 응답 통합 파싱, `ClaudeResponse` DTO로 정규화

---

## 테스트 결과 (2026-03-18)

| Phase | 항목 | 결과 |
|-------|------|------|
| Phase 1 | 대시보드, 메트릭, 이메일 알림 | ✅ PASS |
| Phase 2 | 자가치유, Playbook, Safety Gate | ✅ PASS |
| Phase 3 | AI 사후 분석 | ⚠️ API 크레딧 이슈로 미완 |
| Phase 5.1 | GHCR 자동 폴링 → Rolling Update | ✅ PASS |
| Phase 5.2 | Health Check Probe → 재시작 | ✅ PASS |
| Phase 5.3 | Desired State 재생성 | ⏭️ 미실행 |
| Phase 5.4 | 다중 노드 스케줄링 | ⏭️ 환경 미구성 |

```
[Phase 5.1 실제 로그]
새 이미지 감지: exercise-auth (sha256:4f83c3f... → sha256:758c0f5a...)
Rolling Update 완료: 성공=1, 실패=0

새 이미지 감지: schedule-diary (sha256:85d9a2f... → sha256:a869e0eb...)
Rolling Update 완료: 성공=1, 실패=0
```
