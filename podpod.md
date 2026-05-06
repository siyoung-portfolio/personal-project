  # PodPod

  K-POP 덕질메이트 매칭 서비스

  ## 개요

  K-POP 팬들이 콘서트, 팬미팅 등 오프라인 이벤트에 함께 갈 동행(덕메)을 찾고, 성향에 맞는
  파티(모임)를 추천받을 수 있는 서비스입니다.

  - **기간**: 2024.12 ~ 2025.07
  - **역할**: Backend Developer

  ## 기술 스택

  - Python, FastAPI
  - PostgreSQL, SQLAlchemy 2.0 (async)
  - Alembic (DB 마이그레이션)
  - Pydantic v2
  - Fly.io, GitHub Actions

  ## 아키텍처

  - **Hexagonal Architecture** (Port & Adapter)
    - Domain / Application / Adapter 계층 분리
    - UseCase 인터페이스 기반 의존성 역전
  - **비동기 우선 설계**: async/await 기반 전체 I/O 처리
  - **@Transactional 데코레이터**: 선언적 트랜잭션 관리

  ## 담당 기능

  ### 1. 유저-파티 매칭 시스템

  파티(모임) CRUD 전체를 설계하고 구현했습니다.

  - 파티 생성 / 수정 / 삭제 / 단건·목록 조회
  - 카테고리 필터링 (메인/서브 카테고리)
  - 날짜 필터링, 지역 필터링
  - 조회수 기반 통계 (`user_party_view_stat`)
  - 파티 신청(Party Application) 연동

  ### 2. 주소 처리 시스템

  파티 생성 시 입력된 주소를 파싱하여 구조화된 형태로 저장합니다.

  - 주소 문자열 → `main_location` / `sub_location` / `latitude` / `longitude` 분리 저장
  - 서울 행정구역 매핑 (`SEOUL_ADMINISTRATIVE_NAME_MAPPING`)
  - 지역 기반 파티 필터링 및 추천에 활용

  ### 3. 스크래핑 데이터 가공 및 관리

  외부에서 수집된 아티스트/스케줄 데이터를 서비스에서 활용할 수 있도록 가공합니다.

  - 아티스트 데이터 정제 및 DB 적재
  - 스케줄 데이터와 파티 연동
  - 선호 아티스트(`prefer_artist`) 테이블 설계 및 CRUD

  ### 4. 추천 알고리즘 (가중치 기반)

  사용자 행동 데이터를 기반으로 다단계 폴백 추천 시스템을 구현했습니다.

  **성향 기반 추천** (`get_tendency_based_recommendations`)
  - 1순위: 동일 성향 유저가 최근 7일간 신청한 모임
  - 2순위: 동일 성향 유저가 최근 7일간 조회한 카테고리의 모임

  **유사 파티 추천** (`get_similar_party_recommendations`)
  - 1순위: 참여한 모임과 동일 카테고리
  - 2순위: 참여한 모임과 동일 지역
  - 3순위: 마감 임박 모임 (3일 이내)

  **덕메 유형 매칭 추천** (`get_tendency_match_recommendations`)
  - 최근 확정된 파티 5건에서 함께 매칭된 유저들의 덕메 유형 집계
  - 가장 빈번한 유형 → 1순위, 두 번째 → 2순위

  **아티스트 기반 추천** (`get_recommended_parties`)
  - 선택 아티스트의 최근 7일 조회수 TOP 카테고리
  - 선택 아티스트의 최근 7일 개설 TOP 카테고리

  ### 5. 유저 성향 테스트 알고리즘

  온보딩 시 유저의 덕질 성향(`podpod_tendency`)을 판별하여 프로필에 저장합니다.

  - `MateTendency` 유형 분류 체계 설계
  - 성향 결과를 추천 알고리즘 입력값으로 활용
  - PodPodUser 엔티티에 성향 필드 추가 및 온보딩 플로우 연동

  ## 프로젝트 구조 (담당 모듈)

  app/
  ├── party/              # 파티(모임) 도메인
  ├── party_application/  # 파티 신청                                                                
  ├── recommend/          # 추천 알고리즘
  ├── prefer_artist/      # 선호 아티스트                                                            
  ├── podpod/             # PodPodUser, 성향 테스트                                                
  ├── user_party_view_stat/ # 조회 통계                                                              
  └── schedule/           # 아티스트 스케줄   
