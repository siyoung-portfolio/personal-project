# Kite (Docker Container Orchestrator)

  Kubernetes 스타일 Docker 컨테이너 오케스트레이션 및 모니터링 시스템

  ## 개발 배경

  다큐브에서 온프레미스 서버의 애플리케이션을 GCP 환경으로 이전하면서 여러 VM에서                    
  Docker 컨테이너로 서비스를 운영하게 되었습니다. 이후 특정 컨테이너에 장애가 발생하면
  해당 VM에 직접 접속하거나 브라우저로 확인하는 수밖에 없었고,                                       
  주말이나 업무 시간 외에 빠른 대응이 어려운 상황이었습니다.                                         
                                                                                                     
  K8s 도입을 검토했지만, VM 2대 + 온프레미스 2대에 총 컨테이너 10개 미만인 규모에는                  
  오버스펙이었고, 소규모 개발팀의 높은 학습곡선도 부담이었습니다.                                    
                                                                                                     
  이에 K8s의 핵심 기능(Self-Healing, Health Check, Desired State, 배포 전략)만                       
  경량화하여 직접 구현하기로 결정했습니다.                  

  ## 개요

  Docker 환경에서 Kubernetes의 핵심 기능(Self-Healing, Desired State, Health Check, 배포 전략)을
  경량화하여 구현한 컨테이너 오케스트레이션 플랫폼입니다. SSH 터널 기반 멀티 노드 관리, AI 장애 분석,
   Playbook 자동화까지 포함합니다.

  - **기간**: 2026.03 ~ 현재
  - **역할**: 기획 / Backend Developer (1인 개발)
  - **GitHub**: https://github.com/silicao3o/kite

  ## 기술 스택

  - Java 17, Spring Boot 3.2.3, Maven
  - PostgreSQL, JPA/Hibernate
  - Docker Java API (docker-java 3.3.4)
  - WebSocket, Thymeleaf, Chart.js
  - JSch (SSH 터널링)
  - LangQv (AI 멀티 프로바이더 SDK)
  - JWT (jjwt 0.12.3)
  - GitHub Actions, Dockerfile

  ## 아키텍처

  - **23개 패키지, 227개 Java 파일** 규모의 레이어드 아키텍처
  - UI/API → Service → Orchestration → Infrastructure → Domain 계층 분리
  - **Event-Driven**: Docker Events API 스트림 기반 실시간 이벤트 처리
  - **Strategy Pattern**: 배포 전략, 배치 스케줄링 등에 전략 패턴 적용

  ## 주요 기능

  ### 1. 실시간 컨테이너 모니터링

  Docker Events API를 스트리밍하여 컨테이너 라이프사이클 이벤트를 실시간 감지합니다.

  - 컨테이너 이벤트 감지 (die, kill, oom, start, stop)
  - 이벤트 중복 제거 (60초 윈도우)
  - 리소스 메트릭 수집 (CPU%, Memory%, Network I/O) - 15초 주기
  - Exit Code 분석 (0/1/126/127/137/139/143 패턴 분류)
  - OOM Killer 감지
  - 메트릭 이력 저장 및 시간별/일별 집계, CSV 내보내기

  ### 2. Self-Healing (자동 복구)

  K8s의 자동 복구 메커니즘을 Docker 환경에 구현했습니다.

  - 와일드카드 패턴 매칭 기반 자동 재시작 규칙
  - 지수 백오프 재시작 (10s → 20s → 40s → 300s)
  - Crash Loop 감지 (5분 내 3회 이상 재시작 시 자동 중지)
  - 의도적 종료 분류 (docker stop/kill vs 비정상 종료 구분)
  - 컨테이너 라벨 기반 규칙 오버라이드
  - DB 기반 동적 규칙 관리 (런타임 CRUD, 재시작 없이 적용)

  ### 3. 멀티 노드 오케스트레이션

  SSH 터널링으로 Docker TCP를 노출하지 않고 원격 노드를 관리합니다.

  - **SSH Direct**: JSch Unix 소켓 포워딩으로 원격 노드 직접 연결
  - **SSH Proxy (2-hop)**: NAT/방화벽 뒤 VM에 Control Plane 경유 접근
  - 노드 레지스트리 (JPA + 런타임 캐시 하이브리드)
  - 30초 주기 Heartbeat 모니터링
  - 노드 장애 시 컨테이너 자동 마이그레이션
  - 배치 스케줄링 전략 (LeastUsed / RoundRobin)

  ### 4. 배포 전략

  4가지 배포 전략을 Strategy Pattern으로 구현했습니다.

  - **Rolling Update**: maxUnavailable 배치 단위 순차 교체 (무중단)
  - **Blue-Green**: 새 환경 검증 후 즉시 전환, 즉시 롤백 가능
  - **Canary**: 일부 컨테이너만 업데이트하여 점진적 배포
  - **Recreate**: 전체 중지 후 재시작

  ### 5. Health Check Probes

  K8s의 Probe 시스템을 구현했습니다.

  - **HTTP Probe**: GET 요청으로 응답 코드 검증 (200-399)
  - **TCP Probe**: 소켓 연결 확인
  - **EXEC Probe**: 컨테이너 내부 셸 명령 실행
  - Liveness Probe 실패 시 자동 재시작
  - Readiness Probe 실패 시 로그 기록
  - Initial Delay, Failure Threshold 설정 지원

  ### 6. Desired State Reconciliation

  선언적 상태 관리로 컨테이너 수를 자동 유지합니다.

  - 목표 Replica 수 선언 → 시스템이 자동 유지
  - 30초 주기 상태 점검 (Undercount → 생성, Overcount → 제거)
  - CrashLoop Backoff로 무한 재생성 방지
  - DB 기반 런타임 관리 (재시작 없이 서비스 스펙 변경)

  ### 7. AI 장애 분석

  LangQv SDK를 통해 3개 AI 프로바이더를 런타임에 전환 가능합니다.

  - Anthropic Claude / OpenAI GPT / Google Gemini 연동
  - 컨테이너 메트릭 + 로그 → AI 프롬프트 구성 → 장애 원인 분석
  - 런타임 프로바이더/API Key 변경 (재시작 불필요)
  - Incident Report 자동 생성 및 AI 사후 분석

  ### 8. Playbook 자동화 및 Safety Gate

  YAML 기반 자동화 시나리오와 위험도 기반 실행 제어를 구현했습니다.

  - YAML Playbook 정의 (트리거 조건 → 액션 시퀀스)
  - Safety Gate: 액션 위험도 × 서비스 중요도 × 시간대 가중치로 위험 평가
  - LOW → 자동 실행, MEDIUM/HIGH → 승인 대기, CRITICAL → 수동 승인 필수
  - 승인 대기열 (5분 만료, 대시보드 UI 제공)
  - 감사 로그 180일 보존

  ### 9. GHCR 이미지 자동 업데이트

  GitHub Container Registry의 이미지 변경을 감지하여 자동 배포합니다.

  - Registry v2 API Digest 폴링으로 변경 감지
  - Rolling Update 방식 무중단 업데이트
  - ImageMatchPolicy: Compose에 선언된 이미지와 다른 사이드카 보호
  - 업데이트 이력 추적 (DETECTED / SUCCESS / FAILED)

  ### 10. WebSocket 실시간 대시보드

  - 15초 주기 컨테이너 상태 실시간 브로드캐스트
  - 멀티 노드 필터링 (All / Local / Remote)
  - 컨테이너 상세 페이지 (메트릭 차트, 로그, Self-Healing 이력)
  - Start / Stop / Restart / Update / Delete 컨트롤
  - 로그 검색 (키워드, 레벨, 시간 범위, 스레드, traceId 필터)

  ### 11. 이메일 알림

  - 구독 기반 라우팅 (수신자별 컨테이너/노드 패턴 매칭)
  - CPU 80% / Memory 90% 임계치 알림
  - 비동기 발송, 수신자별 중복 제거
  - HTML 템플릿 (컨테이너 정보, 메트릭, 로그, Exit Code 분석 포함)

  ## 테스트

  - **768개 테스트 케이스** (단위 + 통합)
  - 주요 테스트 영역: Self-Healing (13), Playbook (24), Desired State (37), 멀티 노드 (30), Image
  Update (48), Service Definition (42)

  ## 프로젝트 구조

  src/main/java/com/lite_k8s/
  ├── ai/           # AI 멀티 프로바이더 (LangQv, Claude/GPT/Gemini)
  ├── audit/        # 감사 로그, 보존 정책                                                           
  ├── compose/      # Compose 파싱, 서비스 배포                                                      
  ├── config/       # Spring/Docker/WebSocket/Security 설정                                          
  ├── controller/   # REST API, 대시보드 뷰                                                          
  ├── deploy/       # 4가지 배포 전략                                                                
  ├── desired/      # Desired State, Reconciler, CrashLoop Backoff                                   
  ├── envprofile/   # 환경 프로파일, 암호화                                                          
  ├── health/       # Health Check Probes (HTTP/TCP/EXEC)                                            
  ├── incident/     # 인시던트 리포트, 패턴 감지                                                     
  ├── listener/     # Docker Event 리스너                                                            
  ├── log/          # 로그 저장, 검색, 정리                                                          
  ├── metrics/      # 메트릭 수집, 집계, CSV 내보내기                                                
  ├── model/        # 도메인 엔티티                                                                  
  ├── node/         # 멀티 노드, SSH 터널, Heartbeat, 마이그레이션                                   
  ├── playbook/     # Playbook 파서, Safety Gate, 위험 평가                                          
  ├── security/     # JWT 인증                                                                       
  ├── service/      # 핵심 서비스 (Docker, Self-Healing, Email 등)                                   
  ├── update/       # GHCR 이미지 자동 업데이트                                                      
  ├── util/         # 패턴 매칭, 유틸리티                                                            
  └── websocket/    # 실시간 상태 브로드캐스트, 로그 스트리밍            
