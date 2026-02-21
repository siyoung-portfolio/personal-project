# 운동 인증 앱 (Exercise Auth)

## 프로젝트 개요
- **기간**: -
- **인원**: -
- **역할**: -
- **한줄 소개**: Google Vision AI를 활용하여 운동 인증 사진을 자동 분석하고, Google Sheets와 연동하여 인증 기록을 관리하는 PWA 기반 웹 애플리케이션

---

## 기술스택 (Tech Stack)

### Backend
| 기술 | 버전 | 사용 목적 |
|------|------|----------|
| Java | 17 | 프로그래밍 언어 |
| Spring Boot | 3.2.0 | 백엔드 프레임워크 |
| Spring Security | 3.2.0 | 인증/인가 처리 |
| Spring Data JPA | 3.2.0 | ORM 및 데이터 접근 계층 |
| JJWT | 0.11.5 | JWT 토큰 생성 및 검증 |

### Frontend
| 기술 | 버전 | 사용 목적 |
|------|------|----------|
| Thymeleaf | 3.2.0 | 서버사이드 렌더링 템플릿 엔진 |
| PWA | - | 앱 형태로 설치 가능한 웹앱 구현 |
| Service Worker | - | 오프라인 캐싱 및 네트워크 요청 관리 |

### Database
| 기술 | 버전 | 사용 목적 |
|------|------|----------|
| PostgreSQL | Latest | 사용자 및 운동 기록 데이터 저장 (Neon 클라우드) |

### External APIs & Services
| 기술 | 버전 | 사용 목적 |
|------|------|----------|
| Google Cloud Vision AI | 3.20.0 | 이미지 분석 (라벨, 객체, OCR, 로고 인식) |
| Google Sheets API | v4 | 운동 인증 기록 시트 연동 |
| Uploadcare | 3.5.2 | CDN 기반 이미지 저장소 |

### 기타 도구
| 기술 | 버전 | 사용 목적 |
|------|------|----------|
| metadata-extractor | 2.18.0 | 이미지 EXIF 메타데이터 추출 |
| Springdoc OpenAPI | 2.3.0 | Swagger UI API 문서 자동 생성 |
| Lombok | Latest | 보일러플레이트 코드 제거 |

---

## 기능 개발 (Feature Development)

### 1. Google Vision AI 기반 운동 사진 자동 검증
- **설명**: 사용자가 업로드한 사진이 운동 관련 인증 사진인지 AI가 자동으로 판별
- **구현 내용**:
  - Google Vision API의 4가지 기능 통합 활용 (LABEL_DETECTION, OBJECT_LOCALIZATION, TEXT_DETECTION, LOGO_DETECTION)
  - 신뢰도 점수 알고리즘 설계 (0-100점)
    - 라벨 분석: gym, fitness, exercise, dumbbell 등 키워드 매칭
    - 객체 분석: 덤벨, 바벨, 트레드밀 등 운동 기구 감지
    - 텍스트 분석: 피트니스 앱 키워드 (걸음, 칼로리, 심박수, Samsung Health 등) 인식
    - 로고 분석: 피트니스 브랜드 로고 감지
  - 타임스탬프 워터마크 감지 시 자동 통과 처리
- **사용 기술**: Google Cloud Vision API, Spring Boot

### 2. EXIF/OCR 기반 다중 날짜 검증 시스템
- **설명**: 사진의 촬영 날짜를 다양한 방법으로 추출하여 당일/전일 사진인지 검증
- **구현 내용**:
  - EXIF 메타데이터에서 촬영 일시 추출 (최우선)
  - 파일명 패턴 인식 (Screenshot_20240129_173005 형식 등)
  - OCR을 통한 스크린샷 내 날짜/시간 텍스트 인식
  - "오늘", "어제" 키워드 감지
  - 날짜 유효성 검증 (기본 1일 범위, 타임스탬프 사진은 3일 범위)
- **사용 기술**: metadata-extractor, Google Vision OCR

### 3. Google Sheets 연동 자동 인증 기록 시스템
- **설명**: 운동 인증 성공 시 Google Sheets에 자동으로 도장과 날짜를 기록
- **구현 내용**:
  - 닉네임 기반 행 검색 및 첫 번째 빈 칸 자동 탐지
  - 오버레이 이미지(도장) 복사 및 날짜 기입 배치 처리
  - 주 3회 인증 제한 체크 (시트 기반 Single Source of Truth)
  - 주차/회차 자동 계산 (프로그램 시작일 기준)
  - 병합된 셀 및 "패스"/"벌금" 문자 처리
- **사용 기술**: Google Sheets API

### 4. JWT 기반 사용자 인증 시스템
- **설명**: Stateless 방식의 토큰 기반 인증으로 보안성 확보
- **구현 내용**:
  - 회원가입 시 Google Sheets 닉네임 존재 여부 검증
  - BCrypt 암호화를 통한 비밀번호 보안 저장
  - JWT 토큰 발급 (username, nickname, role 클레임 포함)
  - JwtAuthenticationFilter를 통한 요청별 토큰 검증
  - 24시간 토큰 유효기간 설정
- **사용 기술**: Spring Security, JJWT

### 5. PWA (Progressive Web App) 구현
- **설명**: 웹앱을 모바일 앱처럼 설치하고 사용할 수 있도록 PWA 적용
- **구현 내용**:
  - manifest.json 설정 (앱 이름, 아이콘, 테마 색상, standalone 모드)
  - Service Worker 구현 (Network First 캐시 전략)
  - 오프라인 시 캐시된 데이터 제공
  - 적응형 아이콘 지원 (maskable)
- **사용 기술**: PWA, Service Worker, Web App Manifest

---

## 트러블 슈팅 (Troubleshooting)

### Issue 1: Google Sheets 도장 이미지가 복사되지 않는 문제

**문제 상황 (Problem)**
- 운동 인증 성공 시 Google Sheets에 도장(이미지)을 자동으로 찍어주는 기능 구현 중
- Google Sheets API를 통해 셀 데이터를 읽어와 도장을 복사하려 했으나 빈 값이 반환됨
- API로 셀 값을 가져와도 이미지 데이터가 없어 도장 복사가 불가능

**원인 분석 (Cause)**
- Google Sheets에서 이미지를 셀에 "복사-붙여넣기"로 삽입하면 셀 내부 데이터가 아닌 **오버레이(Overlay)** 형태로 존재
- 셀의 value, formattedValue, userEnteredValue 모두에 이미지 정보가 포함되지 않음
- 이를 처음 발견한 계기: 구글시트에서 도장을 복사한 뒤 디스코드나 다른 곳에 붙여넣기했을 때 **빈 값**으로 붙여넣기 되는 것을 확인

**해결 방법 (Solution)**
- Google Sheets API의 CopyPaste 요청을 활용하여 오버레이 이미지를 복사
- 셀 데이터가 아닌 오버레이를 대상으로 복사/붙여넣기 방식으로 처리
- BatchUpdate의 CopyPasteRequest를 사용하여 원본 도장 위치에서 대상 셀 위치로 복사

**결과 (Result)**
- 도장 이미지가 정상적으로 복사되어 인증 완료 시 시트에 표시됨
- 실제 사용자가 수동으로 복사/붙여넣기 하는 것과 동일한 동작 구현

**배운 점 (Lessons Learned)**
- Google Sheets의 이미지 삽입 방식에는 "셀 내 이미지"와 "오버레이 이미지" 두 가지가 있음
- API 문서만 보지 않고 실제 동작을 디버깅하는 것의 중요성 (다른 앱에 붙여넣기 테스트)
- 문제 상황을 단순화하여 재현해보는 것이 빠른 원인 파악에 도움됨

---

### Issue 2: EXIF 날짜 추출 시 타임존 오차 발생

**문제 상황 (Problem)**
- 운동 인증 사진의 촬영 날짜를 EXIF에서 추출하여 기록하는 기능 구현 중
- 특정 시간대(저녁~밤)에 촬영한 사진의 날짜가 하루 뒤로 잘못 기록됨
- 예: 오후 9시에 촬영한 사진이 다음날 새벽 6시로 기록

**원인 분석 (Cause)**
- metadata-extractor 라이브러리의 `getDate()` 메서드 사용 시 문제 발생
- 해당 메서드는 내부적으로 서버의 타임존(UTC)을 적용하여 Date 객체 반환
- EXIF의 날짜 문자열 "2024:01:29 21:00:00"을 UTC로 해석 후 한국 시간(+9시간)을 더함
- 결과적으로 9시간이 추가되어 다음날 날짜로 변환됨

**해결 방법 (Solution)**
- `getDate()` 메서드 대신 `getString()`으로 원본 문자열을 직접 가져옴
- 타임존 변환 없이 문자열을 직접 파싱하여 LocalDateTime으로 변환
```java
// Before: 타임존 오차 발생
Date date = exifDir.getDate(ExifSubIFDDirectory.TAG_DATETIME_ORIGINAL);

// After: 문자열 직접 파싱으로 해결
String dateString = exifDir.getString(ExifSubIFDDirectory.TAG_DATETIME_ORIGINAL);
LocalDateTime result = LocalDateTime.parse(dateString,
    DateTimeFormatter.ofPattern("yyyy:MM:dd HH:mm:ss"));
```

**결과 (Result)**
- 촬영 시각이 정확하게 추출되어 올바른 날짜로 기록됨
- 저녁 시간대 인증도 당일 날짜로 정상 처리

**배운 점 (Lessons Learned)**
- 날짜/시간 처리 시 타임존을 항상 명시적으로 고려해야 함
- 라이브러리의 편의 메서드가 내부적으로 어떤 변환을 수행하는지 확인 필요
- 서버 환경(UTC)과 사용자 환경(KST)의 차이를 인지하고 설계해야 함
- EXIF 날짜는 타임존 정보가 없는 로컬 시간이므로 변환 없이 그대로 사용하는 것이 적절

---

## 성과 및 회고

### 성과
- Google Vision AI를 활용한 이미지 분석 시스템 구축 경험
- Google Sheets API 연동을 통한 외부 서비스 통합 경험
- PWA 구현으로 웹과 앱의 경계를 넘는 사용자 경험 제공
- JWT 기반 Stateless 인증 시스템 설계 및 구현

### 개선하고 싶은 점
- Vision AI 신뢰도 알고리즘 고도화 (더 정밀한 운동 판별)
- 이미지 처리 성능 최적화 (비동기 처리 적용)
- 테스트 코드 커버리지 향상
