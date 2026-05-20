<!-- VS Code에서 보기: Ctrl+Shift+P → "Markdown Preview: Open Preview to the Side" 권장 -->
<!-- 다이어그램이 흐릿할 경우: settings.json에서 "[markdown]": { "editor.fontFamily": "Courier New, monospace" } 추가 -->

# CAN 프로젝트 설계 단계 산출물

**Document ID:** CAN_Design_Stage_v1.0_260520  
**Project Name:** ICT CampusNav (교내 자원 내비게이션 시스템)  
**Stage:** Design Stage  
**작성일:** 2026-05-20  
**방법론:** AI소프트웨어 개발방법론 (Samsung SDS Innovator, BDD, ADR)

> **📌 참고:** 아래 다이어그램들이 정렬되지 않으면 VS Code settings.json에 다음을 추가하세요:
> ```json
> "[markdown]": {
>     "editor.fontFamily": "Courier New, 'Courier New', monospace",
>     "editor.fontSize": 12
> }
> ```

---

## 📋 목차

1. [시스템 아키텍처 설계서](#1-시스템-아키텍처-설계서-architecture-design)
2. [기능 분해도](#2-기능-분해도-function-map)
3. [데이터 설계서](#3-데이터-설계서-data-design)
4. [UI/UX 설계서](#4-uiux-설계서-interface-layout)
5. [프로세스 설계서](#5-프로세스-설계서-process-logic)
6. [인터페이스 정의서](#6-인터페이스-정의서-integration-spec)
7. [아키텍처 결정 기록](#7-아키텍처-결정-기록-decision-records)

---

## 1. 시스템 아키텍처 설계서 (Architecture Design)

### 1.1 시스템 구성도

```
┌─────────────────────────────────────────────────────────────────┐
│                         사용자 (6가지 역할)                        │
│    학부생 | 조교 | 교수 | 관리자 | 게스트 | 외부인                │
└──────────────────────┬──────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│                    Web Browser (클라이언트)                        │
│  - HTML5 / CSS3 (Bootstrap 5.3.3) / JavaScript                   │
│  - 반응형 디자인 (Desktop / Tablet / Mobile)                      │
└──────────────────┬───────────────────────────────────────────────┘
                   │ HTTP(S) Request/Response
                   ▼
┌──────────────────────────────────────────────────────────────────┐
│                  Apache Tomcat 9.x Container                      │
├──────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │              Servlet Layer (인증, 권한)                      │ │
│  ├─────────────────────────────────────────────────────────────┤ │
│  │  • LoginServlet     (/login)    - 로그인 처리               │ │
│  │  • LogoutServlet    (/logout)   - 로그아웃 처리             │ │
│  │  • GuestServlet     (/guest)    - 게스트 접근               │ │
│  │  • VisitorServlet   (/visitor)  - 외부인 접근               │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                           ▲                                       │
│                           │ Forward/Redirect                      │
│                           ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │               JSP Layer (비즈니스 로직, 표현)                 │ │
│  ├─────────────────────────────────────────────────────────────┤ │
│  │  • campuslogin.jsp        - 로그인 폼                       │ │
│  │  • search.jsp             - 자산 검색                       │ │
│  │  • detail.jsp             - 자산 상세 정보                  │ │
│  │  • reserve.jsp            - 자산 예약                       │ │
│  │  • transfer.jsp           - 자산 이전 기록                  │ │
│  │  • professor.jsp          - 교수 정보 관리                  │ │
│  │  • asset_manage.jsp       - 자산 관리 (관리자)              │ │
│  │  • floorNav.jsp           - 층별 내비게이션                │ │
│  │  • main_[role].jsp        - 역할별 메인 페이지             │ │
│  │  • [save/delete/get]*.jsp - Route 및 API 엔드포인트        │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                           ▲                                       │
│                           │ JDBC                                  │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │               Utility Layer (공통 기능)                       │ │
│  ├─────────────────────────────────────────────────────────────┤ │
│  │  • DBUtil.java     - DB 연결, 리소스 정리 (AutoCloseable)  │ │
│  │  • Session Manager - 사용자 세션 (loginUser, loginRole)    │ │
│  │  • UTF-8 Filter    - 문자 인코딩 필터                      │ │
│  └─────────────────────────────────────────────────────────────┘ │
└──────────────────────────────┬─────────────────────────────────────┘
                               │ TCP/IP (포트 3306)
                               ▼
┌──────────────────────────────────────────────────────────────────┐
│                    MySQL 8.x Database                             │
│                     (campusnav database)                          │
├──────────────────────────────────────────────────────────────────┤
│  • assets                 - 자산 정보 (8,401건)                   │
│  • users                  - 사용자 정보                           │
│  • asset_transfer         - 자산 이전 기록                       │
│  • asset_disposal         - 자산 폐기 기록                       │
│  • reservations           - 자산 예약 정보                       │
│  • professors             - 교수 정보                            │
│  • prof_subjects          - 교수 담당 과목                       │
│  • prof_skills            - 교수 스킬                            │
└──────────────────────────────────────────────────────────────────┘
```

### 1.2 요청 흐름도 (Request Flow)

#### 인증 플로우 (Authentication Flow)

```
사용자 입력
  │
  ▼
campuslogin.jsp (로그인 폼)
  │ (userId, userPw 제출)
  ▼
POST /login (LoginServlet)
  │
  ├─ DB에서 users 테이블 조회 (PreparedStatement)
  │  - SELECT * FROM users WHERE user_id=? AND user_pw=? AND use_yn='Y'
  │
  ├─ 성공 시:
  │  ├─ HttpSession 생성
  │  ├─ session.setAttribute("loginUser", userId)
  │  ├─ session.setAttribute("loginName", userName)
  │  ├─ session.setAttribute("loginRole", role)
  │  ├─ Cookie 저장 (saveId 옵션)
  │  └─ Redirect to /CAN/main_[role].jsp
  │
  └─ 실패 시:
     └─ Forward to campuslogin.jsp (에러 메시지 표시)
```

#### 데이터 조회 플로우 (Search Flow)

```
main_[role].jsp (메인 페이지)
  │
  ▼
사용자 검색 입력 (keyword, type, status, page)
  │
  ▼
GET /search.jsp
  │
  ├─ Session 검증 (loginUser != null)
  │  └─ null이면 campuslogin.jsp로 리다이렉트
  │
  ├─ 동적 WHERE 절 구성 (필터 적용)
  │  - LIKE (keyword)
  │  - asset_class (type)
  │  - asset_status (status)
  │
  ├─ 1단계: COUNT(*) 쿼리로 전체 건수 조회
  │  - SELECT COUNT(*) FROM assets WHERE [동적 WHERE]
  │
  ├─ 2단계: LIMIT/OFFSET로 페이지 단위 조회
  │  - SELECT * FROM assets WHERE [동적 WHERE] LIMIT 20 OFFSET (page-1)*20
  │
  └─ 결과 렌더링 (search.jsp HTML 테이블)
```

### 1.3 컴포넌트 상호작용

| Component | 책임 | 의존성 |
|-----------|------|--------|
| **Servlet** | HTTP 요청 처리, 인증, 세션 관리 | Tomcat, DBUtil, HttpSession |
| **JSP** | HTML 렌더링, JDBC 쿼리 실행 | Servlet, DBUtil, MySQL |
| **DBUtil** | DB 연결 풀, 리소스 정리 | MySQL JDBC Driver |
| **Session** | 사용자 상태 유지 | Tomcat Session Manager |
| **UTF-8 Filter** | 문자 인코딩 통일 | CharacterEncodingFilter |

---

## 2. 기능 분해도 (Function Map)

### 2.1 전체 기능 계층도

```
CAN 시스템
│
├─ [1] 인증 및 접근 제어 (Authentication & Access Control)
│  ├─ [1.1] 로그인
│  │  ├─ [1.1.1] 사용자 ID/PW 입력
│  │  ├─ [1.1.2] DB 조회
│  │  ├─ [1.1.3] 세션 생성
│  │  └─ [1.1.4] 역할별 페이지 리다이렉트
│  ├─ [1.2] 로그아웃
│  │  ├─ [1.2.1] 세션 무효화
│  │  └─ [1.2.2] 로그인 페이지로 리다이렉트
│  ├─ [1.3] 사용자 등록
│  │  ├─ [1.3.1] 회원가입 폼 제출
│  │  ├─ [1.3.2] 검증 (중복 확인 등)
│  │  └─ [1.3.3] users 테이블 INSERT
│  ├─ [1.4] 세션 관리
│  │  ├─ [1.4.1] 세션 생성/유지 (30분 타임아웃)
│  │  └─ [1.4.2] 역할 기반 권한 검증
│  └─ [1.5] 게스트/외부인 접근
│     ├─ [1.5.1] 게스트 접근 (비회원 검색)
│     └─ [1.5.2] 외부인 접근 (제한된 검색)
│
├─ [2] 자산 관리 (Asset Management)
│  ├─ [2.1] 자산 검색
│  │  ├─ [2.1.1] 키워드 검색 (item_name, asset_no, location, dept, model)
│  │  ├─ [2.1.2] 카테고리 필터링 (asset_class)
│  │  ├─ [2.1.3] 상태 필터링 (asset_status)
│  │  ├─ [2.1.4] 페이지네이션 (20개/페이지, OFFSET 기반)
│  │  └─ [2.1.5] 검색 결과 표시 (테이블 + 상세 링크)
│  ├─ [2.2] 자산 상세 조회
│  │  ├─ [2.2.1] 자산 기본 정보 표시
│  │  ├─ [2.2.2] 이전 기록 타임라인 표시
│  │  ├─ [2.2.3] 예약 현황 표시
│  │  └─ [2.2.4] 위치 맵 표시 (준비 중)
│  ├─ [2.3] 자산 관리 (관리자)
│  │  ├─ [2.3.1] 자산 정보 수정
│  │  ├─ [2.3.2] 자산 추가
│  │  ├─ [2.3.3] 자산 폐기
│  │  └─ [2.3.4] 자산 내역 조회
│  └─ [2.4] 자산 분류 및 상태 관리
│     ├─ [2.4.1] 자산분류 (PC, 프로젝터, 책상 등)
│     ├─ [2.4.2] 상태 관리 (사용중, 점검, 폐기)
│     └─ [2.4.3] 위치 관리 (건물, 층, 호실)
│
├─ [3] 예약 관리 (Reservation Management)
│  ├─ [3.1] 자산 예약
│  │  ├─ [3.1.1] 예약 폼 제출
│  │  ├─ [3.1.2] 실시간 중복 확인 (AJAX)
│  │  ├─ [3.1.3] 예약 저장
│  │  └─ [3.1.4] 예약 확인서 표시
│  ├─ [3.2] 예약 현황 조회
│  │  ├─ [3.2.1] 사용자의 예약 목록
│  │  ├─ [3.2.2] 예약 상태 표시
│  │  └─ [3.2.3] 예약 취소
│  └─ [3.3] 예약 관리 (조교/관리자)
│     ├─ [3.3.1] 예약 승인/거절
│     ├─ [3.3.2] 예약 일정 조회
│     └─ [3.3.3] 예약 통계
│
├─ [4] 자산 이전 (Asset Transfer)
│  ├─ [4.1] 이전 신청
│  │  ├─ [4.1.1] 이전 폼 제출
│  │  ├─ [4.1.2] 이전 사유 입력
│  │  └─ [4.1.3] asset_transfer 테이블 INSERT
│  ├─ [4.2] 이전 기록 조회
│  │  ├─ [4.2.1] 이전 타임라인 표시
│  │  ├─ [4.2.2] Before/After 위치 비교
│  │  └─ [4.2.3] 이전 담당자 정보
│  └─ [4.3] 이전 승인 (관리자)
│     ├─ [4.3.1] 대기 이전 목록
│     ├─ [4.3.2] 승인/거절
│     └─ [4.3.3] 이전 완료 처리
│
├─ [5] 교수 관리 (Professor Management)
│  ├─ [5.1] 교수 정보 관리
│  │  ├─ [5.1.1] 교수 카드 표시
│  │  ├─ [5.1.2] 교수 정보 수정
│  │  └─ [5.1.3] 교수 삭제
│  ├─ [5.2] 과목 관리
│  │  ├─ [5.2.1] 담당 과목 추가
│  │  ├─ [5.2.2] 담당 과목 수정
│  │  └─ [5.2.3] 담당 과목 삭제
│  └─ [5.3] 스킬 관리
│     ├─ [5.3.1] 스킬 태그 추가
│     ├─ [5.3.2] 스킬 태그 수정
│     └─ [5.3.3] 스킬 태그 삭제
│
├─ [6] 내비게이션 (Navigation)
│  ├─ [6.1] 층별 내비게이션
│  │  ├─ [6.1.1] 층 선택
│  │  ├─ [6.1.2] 자산 위치 마킹
│  │  └─ [6.1.3] 경로 저장/불러오기
│  └─ [6.2] 위치 기반 검색
│     ├─ [6.2.1] 건물 선택
│     ├─ [6.2.2] 층 선택
│     └─ [6.2.3] 주변 자산 표시
│
└─ [7] 대시보드 및 리포팅 (Reporting)
   ├─ [7.1] 사용자 대시보드
   │  ├─ [7.1.1] 최근 예약 현황
   │  ├─ [7.1.2] 자주 사용 자산
   │  └─ [7.1.3] 예약 알림
   ├─ [7.2] 관리자 대시보드
   │  ├─ [7.2.1] 자산 현황 통계
   │  ├─ [7.2.2] 예약 현황 통계
   │  ├─ [7.2.3] 사용 현황 분석
   │  └─ [7.2.4] 시스템 헬스 체크
   └─ [7.3] 리포트 생성
      ├─ [7.3.1] 월별 예약 현황
      ├─ [7.3.2] 자산 사용률 분석
      └─ [7.3.3] 부서별 자산 현황
```

### 2.2 기능별 REQ ID 매핑

| REQ ID | 기능명 | 분해도 ID | 우선순위 | 담당 JSP |
|--------|--------|-----------|---------|---------|
| F-1 | 로그인 | 1.1 | Critical | LoginServlet → main_*.jsp |
| F-2 | 자산 검색 | 2.1 | High | search.jsp |
| F-3 | 자산 상세 | 2.2 | High | detail.jsp |
| F-4 | 자산 예약 | 3.1 | High | reserve.jsp |
| F-5 | 자산 이전 | 4.1 | High | transfer.jsp |
| F-6 | 교수 관리 | 5 | Medium | professor.jsp |
| F-7 | 내비게이션 | 6 | Medium | floorNav.jsp |
| F-8 | 자산 관리 | 2.3 | Medium | asset_manage.jsp |
| F-9 | 대시보드 | 7 | Low | main_*.jsp |

---

## 3. 데이터 설계서 (Data Design)

### 3.1 논리 데이터 모델 (Logical Data Model)

#### ER 다이어그램

```
┌─────────────────────┐
│      users          │
├─────────────────────┤
│ user_id (PK)        │◄─────┐
│ user_name           │      │
│ user_pw             │      │
│ user_email          │      │
│ user_phone          │      │
│ role                │      │
│ use_yn              │      │
│ reg_date            │      │
│ mod_date            │      │
└─────────────────────┘      │
         ▲                    │
         │                    │
    1:N  │              1:N   │
         │                    │
┌─────────────────────┐  ┌────────────────────┐
│   assets            │  │  reservations      │
├─────────────────────┤  ├────────────────────┤
│ asset_no (PK)       │  │ reservation_id(PK) │
│ asset_class         │  │ asset_no (FK)      │
│ item_name           │  │ user_id (FK)       │──┐
│ model               │  │ reserve_date       │  │
│ serial_no           │  │ start_time         │  │
│ purchase_date       │  │ end_time           │  │
│ purchase_price      │  │ purpose            │  │
│ location            │  │ status             │  │
│ detail_location     │  │ reg_date           │  │
│ manage_dept         │  │ mod_date           │  │
│ manager_name        │  │ mod_user           │  │
│ asset_status        │  └────────────────────┘  │
│ reg_date            │                    1:N   │
│ mod_date            │  ┌─────────────────────┐ │
└─────────────────────┘  │  professors         │ │
         ▲                │─────────────────────┤ │
         │                │ prof_id (PK) ◄─────┼─┘
         │ 1:N            │ user_id (FK)│◄─────┐
         │                │ dept_name   │      │
┌─────────────────────┐  │ office_no   │      │
│ asset_transfer      │  │ phone       │      │
├─────────────────────┤  │ email       │      │
│ transfer_id (PK)    │  │ reg_date    │      │
│ asset_no (FK)       │  │ mod_date    │      │
│ transfer_date       │  └─────────────────────┘
│ before_dept         │          │
│ before_location     │          │ 1:N
│ after_dept          │  ┌───────┴──────────────┐
│ after_location      │  │                      │
│ remark              │  ▼                      ▼
│ mod_user            │ ┌──────────────┐  ┌──────────────┐
│ reg_date            │ │prof_subjects │  │ prof_skills  │
│ mod_date            │ ├──────────────┤  ├──────────────┤
└─────────────────────┘ │ subject_id   │  │ skill_id     │
         ▲              │ prof_id (FK) │  │ prof_id (FK) │
         │              │ subject_name │  │ skill_name   │
    1:N  │              │ credit       │  │ proficiency  │
         │              │ semester     │  │ reg_date     │
┌─────────────────────┐ │ reg_date     │  │ mod_date     │
│asset_disposal       │ │ mod_date     │  └──────────────┘
├─────────────────────┤ └──────────────┘
│ disposal_id (PK)    │
│ asset_no (FK)       │
│ disposal_date       │
│ reason              │
│ disposal_method     │
│ reg_user            │
│ reg_date            │
│ mod_date            │
└─────────────────────┘
```

### 3.2 물리 데이터 모델 (Physical Data Model)

#### 테이블 명세

| 테이블명 | 용도 | 레코드 수 | 주요 칼럼 | 인덱스 |
|----------|------|---------|---------|--------|
| **users** | 사용자 정보 | ~100 | user_id(PK), user_pw, role | PK, role |
| **assets** | 자산 정보 | 8,401 | asset_no(PK), item_name, location, status | PK, asset_class, status |
| **asset_transfer** | 자산 이전 기록 | ~1000 | transfer_id(PK), asset_no(FK), transfer_date | PK, asset_no, date |
| **asset_disposal** | 자산 폐기 기록 | ~500 | disposal_id(PK), asset_no(FK), disposal_date | PK, asset_no |
| **reservations** | 자산 예약 | ~5000 | reservation_id(PK), asset_no(FK), user_id(FK) | PK, asset_no, user_id, date |
| **professors** | 교수 정보 | ~200 | prof_id(PK), user_id(FK), dept_name | PK, user_id |
| **prof_subjects** | 교수 과목 | ~400 | subject_id(PK), prof_id(FK), subject_name | PK, prof_id |
| **prof_skills** | 교수 스킬 | ~600 | skill_id(PK), prof_id(FK), skill_name | PK, prof_id |

#### 데이터 정의 (DDL 요약)

```sql
-- users: 사용자 기본 정보
CREATE TABLE users (
  user_id VARCHAR(50) PRIMARY KEY,
  user_name VARCHAR(100) NOT NULL,
  user_pw VARCHAR(255) NOT NULL,
  user_email VARCHAR(100),
  user_phone VARCHAR(20),
  role ENUM('student','assistant','professor','admin','guest','visitor'),
  use_yn CHAR(1) DEFAULT 'Y',
  reg_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  mod_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  INDEX idx_role (role),
  INDEX idx_use_yn (use_yn)
);

-- assets: 자산 정보 (8,401건)
CREATE TABLE assets (
  asset_no VARCHAR(50) PRIMARY KEY,
  asset_class VARCHAR(50) NOT NULL,
  item_name VARCHAR(200) NOT NULL,
  model VARCHAR(100),
  serial_no VARCHAR(100),
  purchase_date DATE,
  purchase_price DECIMAL(10,2),
  location VARCHAR(100),
  detail_location VARCHAR(200),
  manage_dept VARCHAR(100),
  manager_name VARCHAR(100),
  manager_phone VARCHAR(20),
  asset_status VARCHAR(50),
  reg_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  mod_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  INDEX idx_asset_class (asset_class),
  INDEX idx_status (asset_status),
  FULLTEXT idx_search (item_name, serial_no)
);

-- reservations: 자산 예약
CREATE TABLE reservations (
  reservation_id INT AUTO_INCREMENT PRIMARY KEY,
  asset_no VARCHAR(50) NOT NULL,
  user_id VARCHAR(50) NOT NULL,
  reserve_date DATE NOT NULL,
  start_time TIME,
  end_time TIME,
  purpose VARCHAR(500),
  status VARCHAR(50),
  reg_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  mod_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  FOREIGN KEY (asset_no) REFERENCES assets(asset_no),
  FOREIGN KEY (user_id) REFERENCES users(user_id),
  INDEX idx_asset_date (asset_no, reserve_date),
  UNIQUE KEY unique_reserve (asset_no, reserve_date, start_time, end_time)
);

-- asset_transfer: 자산 이전 기록
CREATE TABLE asset_transfer (
  transfer_id INT AUTO_INCREMENT PRIMARY KEY,
  asset_no VARCHAR(50) NOT NULL,
  transfer_date DATE NOT NULL,
  before_dept VARCHAR(100),
  before_location VARCHAR(200),
  after_dept VARCHAR(100),
  after_location VARCHAR(200),
  remark VARCHAR(500),
  mod_user VARCHAR(50),
  reg_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  mod_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  FOREIGN KEY (asset_no) REFERENCES assets(asset_no),
  INDEX idx_asset_date (asset_no, transfer_date)
);

-- professors: 교수 정보
CREATE TABLE professors (
  prof_id INT AUTO_INCREMENT PRIMARY KEY,
  user_id VARCHAR(50) NOT NULL,
  dept_name VARCHAR(100),
  office_no VARCHAR(50),
  phone VARCHAR(20),
  email VARCHAR(100),
  reg_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  mod_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES users(user_id),
  UNIQUE KEY unique_user (user_id)
);

-- prof_subjects: 교수 담당 과목
CREATE TABLE prof_subjects (
  subject_id INT AUTO_INCREMENT PRIMARY KEY,
  prof_id INT NOT NULL,
  subject_name VARCHAR(100),
  credit INT,
  semester VARCHAR(20),
  reg_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  mod_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  FOREIGN KEY (prof_id) REFERENCES professors(prof_id),
  INDEX idx_prof (prof_id)
);

-- prof_skills: 교수 스킬
CREATE TABLE prof_skills (
  skill_id INT AUTO_INCREMENT PRIMARY KEY,
  prof_id INT NOT NULL,
  skill_name VARCHAR(100),
  proficiency VARCHAR(50),
  reg_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  mod_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  FOREIGN KEY (prof_id) REFERENCES professors(prof_id),
  INDEX idx_prof (prof_id)
);

-- asset_disposal: 자산 폐기
CREATE TABLE asset_disposal (
  disposal_id INT AUTO_INCREMENT PRIMARY KEY,
  asset_no VARCHAR(50) NOT NULL,
  disposal_date DATE NOT NULL,
  reason VARCHAR(500),
  disposal_method VARCHAR(100),
  reg_user VARCHAR(50),
  reg_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  mod_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  FOREIGN KEY (asset_no) REFERENCES assets(asset_no),
  INDEX idx_asset_date (asset_no, disposal_date)
);
```

### 3.3 데이터 생명주기

| 엔티티 | 생성 조건 | 수정 조건 | 삭제 조건 | 보관 기간 |
|--------|---------|---------|---------|---------|
| users | 회원가입 | 개인정보 변경 | use_yn='N' (논리삭제) | 무기한 |
| assets | 자산 구입 | 정보 수정, 상태 변경 | asset_disposal 생성 | 무기한 |
| reservations | 예약 신청 | 예약 승인/거절 | status='취소' (논리삭제) | 1년 |
| asset_transfer | 자산 이전 신청 | 이전 승인 | 삭제 불가 (감사 추적) | 무기한 |
| professors | 교수 정보 등록 | 교수 정보 변경 | use_yn='N' | 무기한 |

---

## 4. UI/UX 설계서 (Interface Layout)

### 4.1 시스템 사용자 인터페이스

#### 색상 체계 (Color Scheme)

```
Primary:   #1a56db (파란색)      → 주요 버튼, 링크, 하이라이트
Secondary: #0d9488 (녹색)        → 성공, 활성 상태
Alert:     #dc2626 (빨간색)      → 경고, 오류, 위험 작업
Neutral:   #111827 (검정)        → 텍스트
Background:#f7f8fa (라이트 그레이)  → 페이지 배경
```

#### 타이포그래피

```
Font Family:  DM Sans, Noto Sans KR (한글 폰트로 보정)
Headlines:    800 weight (28px H1, 22px H2, 18px H3)
Body:         400-500 weight (15px 기본)
Monospace:    DM Mono (데이터, 코드)
Line Height:  1.65 (가독성)
```

#### 레이아웃 가이드

```
┌─────────────────────────────────────────────────────┐
│  Logo + Nav (Sticky Top)                      User │
│  ┌────────────────────────────────────────────────┐│
│  │ Hero Section (검색 결과, 통계, 액션)             ││
│  └────────────────────────────────────────────────┘│
│  ┌────────────────────────────────────────────────┐│
│  │ Stat Row (4개 통계: 자산, 예약, 이전, 대기)    ││
│  └────────────────────────────────────────────────┘│
│  ┌────────────────────────────────────────────────┐│
│  │ Content Area (테이블, 폼, 상세)                  ││
│  │ Max Width: 1380px                              ││
│  │ Padding: 28px                                  ││
│  └────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────┘
```

### 4.2 주요 페이지 와이어프레임

#### 1. 로그인 페이지 (campuslogin.jsp)

```
┌────────────────────────────────────┐
│      ICT CAN 로그인                │
├────────────────────────────────────┤
│                                    │
│  [아이디]      [       ]           │
│                                    │
│  [비밀번호]    [       ]           │
│                                    │
│  [  ID 저장  ]  [로그인 버튼]      │
│                                    │
│  비회원 | 회원가입                  │
│                                    │
│  ─────────────────────────────────  │
│  [학부생] [조교] [교수] [관리자]     │
│  (빠른 선택 버튼 - 데모용)          │
│                                    │
└────────────────────────────────────┘
```

#### 2. 자산 검색 페이지 (search.jsp)

```
┌─────────────────────────────────────────────────┐
│ Logo                         [사용자] [로그아웃] │
├─────────────────────────────────────────────────┤
│ ┌───────────────────────────────────────────┐   │
│ │ 🔍 자산 검색 | 8,401개 자산 관리          │   │
│ │ [검색]  [카테고리▼] [상태▼]  [검색 버튼]  │   │
│ └───────────────────────────────────────────┘   │
│ ┌───────────────────────────────────────────┐   │
│ │ 검색 결과: 123개                          │   │
│ ├───────────────────────────────────────────┤   │
│ │ No | 자산분류 | 품명 | 위치 | 상태 | 상세  │   │
│ ├───────────────────────────────────────────┤   │
│ │ 1  | PC      | 옥스포드 PC-01 | 308호 | 사용중 │
│ │ 2  | 모니터  | DELL 27"       | 308호 | 사용중 │
│ │ ... (20개 표시)                           │   │
│ └───────────────────────────────────────────┘   │
│ [◀ 1 2 3 4 5 ▶] (페이지네이션)                 │
└─────────────────────────────────────────────────┘
```

#### 3. 자산 상세 페이지 (detail.jsp)

```
┌────────────────────────────────────────────┐
│ Logo                      [사용자] [로그아웃]│
├────────────────────────────────────────────┤
│ ┌──────────────────────────────────────┐   │
│ │ 옥스포드 PC-01 | [사용중]             │   │
│ │ 자산번호: 2024-005-01                │   │
│ └──────────────────────────────────────┘   │
│                                            │
│ ┌──────────────────────────────────────┐   │
│ │ 기본 정보                             │   │
│ ├──────────────────────────────────────┤   │
│ │ 분류: PC                             │   │
│ │ 모델: DELL Inspiron 5000             │   │
│ │ 위치: 308호 (정보통신관)              │   │
│ │ 관리부서: 정보통신과                  │   │
│ │ 구입가격: 1,200,000원                │   │
│ │ 구입일: 2023-05-15                  │   │
│ └──────────────────────────────────────┘   │
│                                            │
│ ┌──────────────────────────────────────┐   │
│ │ 이전 기록 (타임라인)                 │   │
│ ├──────────────────────────────────────┤   │
│ │ 2025-03-01: 308호 → 310호           │   │
│ │ 2024-08-15: 구입 (초기 배치)        │   │
│ └──────────────────────────────────────┘   │
│                                            │
│ ┌──────────────────────────────────────┐   │
│ │ 예약 현황                            │   │
│ ├──────────────────────────────────────┤   │
│ │ 2026-05-25 14:00~16:00: 김학생 예약 │   │
│ │ 2026-05-26 10:00~12:00: 이학생 예약 │   │
│ └──────────────────────────────────────┘   │
│                                            │
│ [🗺️ 지도]  [예약하기]  [다시검색]       │
└────────────────────────────────────────────┘
```

#### 4. 자산 예약 페이지 (reserve.jsp)

```
┌────────────────────────────────────────┐
│ 자산 예약                             │
├────────────────────────────────────────┤
│ 자산명: 옥스포드 PC-01                │
│ 위치: 308호                           │
│ 현재 상태: 사용중                      │
│                                        │
│ 예약 정보                              │
│ 예약일: [2026-05-25▼]                 │
│ 시작시간: [14:00▼]                    │
│ 종료시간: [16:00▼]                    │
│ 사용목적: [               ]           │
│ 예약자: 김학생                         │
│                                        │
│ ✓ 중복 확인됨 (예약 가능)              │
│                                        │
│ [예약 신청]  [취소]                   │
└────────────────────────────────────────┘
```

### 4.3 반응형 디자인 (Responsive Design)

```
Desktop (1024px+)          Tablet (768px)        Mobile (375px)
┌──────────────────┐      ┌──────────────┐     ┌──────────┐
│ Logo    Nav Items│      │ Logo    Menu│     │ Logo ≡   │
├──────────────────┤      ├──────────────┤     ├──────────┤
│                  │      │              │     │          │
│ 4-Column Grid    │      │ 2-Column Grid│     │ 1-Column │
│  [C1] [C2]       │      │  [C1] [C2]  │     │ [Content]│
│  [C3] [C4]       │      │  [C3] [C4]  │     │          │
│                  │      │  [C5] [C6]  │     │ [Content]│
│                  │      │              │     │          │
└──────────────────┘      └──────────────┘     └──────────┘
```

---

## 5. 프로세스 설계서 (Process Logic)

### 5.1 핵심 비즈니스 프로세스

#### [P-1] 사용자 로그인 프로세스

```
시작
  │
  ▼
사용자 ID/PW 입력 (campuslogin.jsp)
  │
  ▼
POST /login (LoginServlet.doPost)
  │
  ├─ 입력값 검증 (공백 제거, NULL 확인)
  │   └─ fail: 캡처 메시지 표시
  │
  ├─ DB 조회: users 테이블
  │   SQL: SELECT user_name, role FROM users 
  │         WHERE user_id=? AND user_pw=? AND use_yn='Y'
  │   └─ fail: "아이디, 비밀번호가 틀렸습니다" 표시
  │
  ├─ 기존 세션 무효화
  │
  ├─ 새 HttpSession 생성
  │   session.setAttribute("loginUser", userId)
  │   session.setAttribute("loginName", userName)
  │   session.setAttribute("loginRole", role)
  │   session.setMaxInactiveInterval(30 * 60)
  │
  ├─ saveId 쿠키 설정 (30일)
  │
  ├─ 역할별 페이지로 리다이렉트
  │   role = "student"   → /CAN/main_student.jsp
  │   role = "assistant" → /CAN/main_assistant.jsp
  │   role = "professor" → /CAN/main_professor.jsp
  │   role = "admin"     → /CAN/main_admin.jsp
  │   default            → /CAN/main_guest.jsp
  │
  ▼
완료 (세션 유지 30분)
```

#### [P-2] 자산 검색 프로세스

```
시작
  │
  ▼
search.jsp 로드
  │
  ├─ Session 검증
  │   loginUser == null → campuslogin.jsp 리다이렉트
  │
  ├─ 요청 파라미터 수집
  │   - keyword (검색어)
  │   - type (카테고리)
  │   - status (상태)
  │   - page (페이지 번호, 기본값: 1)
  │
  ├─ 동적 WHERE 절 구성
  │   StringBuilder w = new StringBuilder("WHERE 1=1 ")
  │   
  │   if (keyword != null && !keyword.isEmpty()) {
  │       w.append("AND (item_name LIKE ? OR asset_no LIKE ? OR ...")
  │       → %keyword% 5개 파라미터 추가
  │   }
  │   if (type != null && !type.isEmpty()) {
  │       w.append("AND asset_class = ?")
  │   }
  │   if (status != null && !status.isEmpty()) {
  │       w.append("AND asset_status LIKE ?")
  │   }
  │
  ├─ [1단계] 전체 건수 조회
  │   SELECT COUNT(*) FROM assets [WHERE절]
  │   → totalCount, totalPage 계산
  │
  ├─ [2단계] 페이지 단위 조회
  │   SELECT * FROM assets [WHERE절]
  │   ORDER BY reg_date DESC
  │   LIMIT 20 OFFSET (page-1)*20
  │
  ├─ ResultSet을 List<Map> 변환
  │   (item_name, model, location, status 등 표시)
  │
  ├─ HTML 렌더링
  │   - 검색 폼 렌더링
  │   - 통계 표시 (검색 결과 건수)
  │   - 테이블 렌더링 (20행 + 상세 링크)
  │   - 페이지네이션 (이전/다음, 페이지 번호)
  │
  ▼
완료 (search.jsp 화면 표시)
```

#### [P-3] 자산 상세 조회 프로세스

```
시작
  │
  ▼
detail.jsp?id=asset_no
  │
  ├─ Session 검증
  │
  ├─ asset_no 파라미터 검증
  │   null or empty → search.jsp 리다이렉트
  │
  ├─ [1단계] 자산 기본 정보 조회
  │   SELECT * FROM assets WHERE asset_no = ?
  │   → 모든 칼럼을 Map에 저장
  │
  ├─ [2단계] 이전 기록 조회
  │   SELECT transfer_date, before_dept, before_location, 
  │          after_dept, after_location, remark
  │   FROM asset_transfer 
  │   WHERE asset_no = ?
  │   ORDER BY transfer_date DESC
  │   → transfers List 생성
  │
  ├─ [3단계] 예약 현황 조회
  │   SELECT r.reserve_date, r.start_time, r.end_time, 
  │          r.purpose, u.user_name
  │   FROM reservations r 
  │   JOIN users u ON r.user_id = u.user_id
  │   WHERE r.asset_no = ? 
  │   AND r.reserve_date >= CURDATE() 
  │   AND r.status = '예약완료'
  │   ORDER BY r.reserve_date, r.start_time
  │   → reserves List 생성
  │
  ├─ HTML 렌더링
  │   - 자산명, 상태 배지 표시
  │   - 기본 정보 (분류, 모델, 위치, 부서, 가격, 구입일)
  │   - 이전 기록 타임라인
  │   - 예약 현황 리스트
  │   - 지도 플레이스홀더 (준비 중)
  │
  ├─ 액션 버튼
  │   - [예약하기] → reserve.jsp?id=asset_no
  │   - [다시검색] → search.jsp
  │   - [지도보기] (준비 중)
  │
  ▼
완료 (detail.jsp 화면 표시)
```

#### [P-4] 자산 예약 프로세스

```
시작
  │
  ▼
reserve.jsp?id=asset_no
  │
  ├─ Session 검증
  │
  ├─ asset_no 파라미터 검증
  │
  ├─ [GET] 초기 로드
  │   - 자산 기본 정보 조회 (이름, 위치, 상태)
  │   - 예약 폼 렌더링
  │
  ├─ [POST] 예약 신청
  │   사용자 입력: reserve_date, start_time, end_time, purpose
  │
  │   ├─ 입력값 검증
  │   │   - reserve_date >= CURDATE()
  │   │   - start_time < end_time
  │   │   - purpose != empty
  │   │
  │   ├─ [AJAX] 실시간 중복 확인 (beforesubmit)
  │   │   SELECT COUNT(*) FROM reservations
  │   │   WHERE asset_no = ? 
  │   │   AND reserve_date = ?
  │   │   AND status = '예약완료'
  │   │   AND ((start_time <= ? AND end_time > ?) 
  │   │        OR (start_time < ? AND end_time >= ?))
  │   │   
  │   │   if (count > 0) {
  │   │       showError("이 시간에 이미 예약되어 있습니다")
  │   │       return false  // 폼 제출 중단
  │   │   }
  │   │
  │   ├─ INSERT INTO reservations
  │   │   (asset_no, user_id, reserve_date, start_time, 
  │   │    end_time, purpose, status, reg_date)
  │   │   VALUES (?, ?, ?, ?, ?, ?, '예약완료', NOW())
  │   │
  │   ├─ 예약 확인 메시지 표시
  │   │
  │   └─ 메인 페이지로 리다이렉트
  │
  ▼
완료 (예약 저장 및 확인)
```

### 5.2 데이터 검증 규칙

| 입력 필드 | 검증 규칙 | 오류 메시지 |
|---------|---------|-----------|
| user_id | 2~20자, 영문+숫자 | "아이디는 2~20자의 영문과 숫자만 가능합니다" |
| user_pw | 6자 이상, 영문+숫자+특수문자 | "비밀번호는 6자 이상이고 영문, 숫자, 특수문자를 포함해야 합니다" |
| keyword | 50자 이내 | "검색어는 50자 이내입니다" |
| reserve_date | >= TODAY | "예약일은 오늘 이후만 가능합니다" |
| start_time | HH:MM 형식 | "시간 형식이 잘못되었습니다" |
| end_time | > start_time | "종료 시간이 시작 시간보다 빠를 수 없습니다" |
| purpose | 1~500자 | "사용 목적은 1~500자입니다" |

---

## 6. 인터페이스 정의서 (Integration Specification)

### 6.1 HTTP API 정의

#### [API-1] 로그인

```
POST /login
Content-Type: application/x-www-form-urlencoded

Request:
  userId=user123&userPw=pass123&saveId=on

Response (Success 302):
  Location: /CAN/main_student.jsp
  Set-Cookie: savedId=user123; Path=/; Max-Age=2592000

Response (Fail 200):
  Forward to: /CAN/campuslogin.jsp
  Request Attribute: errorMsg, prevId
```

#### [API-2] 로그아웃

```
GET /logout

Response (302):
  Location: /CAN/campuslogin.jsp
  Session: INVALIDATE
```

#### [API-3] 자산 검색 (GET - JSP)

```
GET /search.jsp?keyword=PC&type=&status=&page=1

Response Headers:
  Content-Type: text/html; charset=UTF-8

Response Body:
  HTML 페이지 (search.jsp)
  - 검색 필터 폼
  - 검색 결과 테이블 (20행)
  - 페이지네이션 컨트롤
```

#### [API-4] 자산 상세 조회

```
GET /detail.jsp?id=2024-005-01

Response Headers:
  Content-Type: text/html; charset=UTF-8

Response Body:
  HTML 페이지 (detail.jsp)
  - 자산 기본 정보
  - 이전 기록 타임라인
  - 예약 현황
  - 지도 플레이스홀더
```

#### [API-5] 예약 중복 확인 (AJAX)

```
POST /checkReservation
Content-Type: application/x-www-form-urlencoded
X-Requested-With: XMLHttpRequest

Request:
  assetNo=2024-005-01&reserveDate=2026-05-25
  &startTime=14:00&endTime=16:00

Response (JSON):
  {
    "available": true,
    "message": "예약 가능"
  }
  
  OR
  
  {
    "available": false,
    "message": "이 시간에 이미 예약되어 있습니다",
    "conflictReservations": [
      {"userName": "김학생", "startTime": "14:30", "endTime": "15:30"}
    ]
  }
```

#### [API-6] 예약 저장

```
POST /reserve.jsp
Content-Type: application/x-www-form-urlencoded

Request:
  assetNo=2024-005-01&reserveDate=2026-05-25
  &startTime=14:00&endTime=16:00&purpose=프로젝트

Response (302):
  Location: /CAN/main_student.jsp?msg=예약완료
  
Response (200 Error):
  Forward to: /CAN/reserve.jsp?id=2024-005-01
  Request Attribute: errorMsg
```

#### [API-7] 자산 이전 저장 (saveFloorRoute.jsp)

```
POST /saveFloorRoute.jsp
Content-Type: application/x-www-form-urlencoded

Request:
  assetNo=2024-005-01&beforeLoc=308호&afterLoc=310호
  &remark=층 간 이전

Response (JSON):
  {
    "success": true,
    "transferId": 12345,
    "message": "이전 기록이 저장되었습니다"
  }
```

### 6.2 데이터 전달 명세 (Data Format)

#### 검색 결과 데이터 구조

```java
List<Map<String,String>> searchResult = new ArrayList<>();
Map<String,String> row = new LinkedHashMap<>();
row.put("no", "2024-005-01");           // 자산번호
row.put("cls", "PC");                   // 분류
row.put("name", "옥스포드 PC-01");      // 자산명
row.put("model", "DELL Inspiron 5000"); // 모델
row.put("loc", "정보통신관");            // 위치
row.put("dept", "정보통신과");           // 부서
row.put("mgr", "이관리자");              // 담당자
row.put("st", "사용중");                // 상태
searchResult.add(row);
```

#### 자산 상세 데이터 구조

```java
Map<String,String> asset = new LinkedHashMap<>();
asset.put("asset_no", "2024-005-01");
asset.put("asset_class", "PC");
asset.put("item_name", "옥스포드 PC-01");
asset.put("model", "DELL Inspiron 5000");
asset.put("serial_no", "ABC123456789");
asset.put("purchase_date", "2023-05-15");
asset.put("purchase_price", "1200000");
asset.put("location", "정보통신관");
asset.put("detail_location", "308호");
asset.put("manage_dept", "정보통신과");
asset.put("manager_name", "이관리자");
asset.put("asset_status", "사용중");

// 이전 기록
List<Map<String,String>> transfers = new ArrayList<>();
Map<String,String> transfer = new LinkedHashMap<>();
transfer.put("date", "2025-03-01");
transfer.put("fdept", "정보통신과");
transfer.put("floc", "308호");
transfer.put("tdept", "정보통신과");
transfer.put("tloc", "310호");
transfer.put("rmk", "층 간 이전");
transfers.add(transfer);

// 예약 현황
List<Map<String,String>> reserves = new ArrayList<>();
Map<String,String> reserve = new LinkedHashMap<>();
reserve.put("date", "2026-05-25");
reserve.put("start", "14:00");
reserve.put("end", "16:00");
reserve.put("purpose", "프로젝트 진행");
reserve.put("user", "김학생");
reserves.add(reserve);
```

### 6.3 세션 및 쿠키 정의

| 항목 | 이름 | 타입 | 범위 | 유효기간 |
|------|------|------|------|---------|
| **Session** | loginUser | String | HttpSession | 30분 (Tomcat) |
| **Session** | loginName | String | HttpSession | 30분 |
| **Session** | loginRole | String | HttpSession | 30분 |
| **Cookie** | savedId | String | 브라우저 | 30일 |

---

## 7. 아키텍처 결정 기록 (Decision Records)

### ADR-1: Servlet + JSP 아키텍처 선택

**상태:** Accepted  
**날짜:** 2024-04-14

**맥락:**
교내 자산 관리 시스템으로, 중규모 프로젝트이며, 빠른 프로토타이핑과 운영 안정성이 중요함.

**결정:**
Apache Tomcat + Java Servlet + JSP를 기술 스택으로 선택

**대안 검토:**

| 대안 | 장점 | 단점 | 평가 |
|------|------|------|------|
| **Servlet+JSP** | 가볍고, 구현 빠름, 호스팅 용이 | 확장성 제한 | ✅ **선택** |
| Spring Boot | 프레임워크 지원, 확장성 높음 | 학습곡선 가파름, 오버헤드 | 차선 |
| React + REST API | 현대적 아키텍처, 반응성 좋음 | 개발 시간 증가, 배포 복잡 | 차선 |
| Node.js | 빠른 개발, JavaScript 통일 | 성능 이슈, 팀 경험 부족 | 탈락 |

**결과:**
- ✅ 2024년 4월 구현 완료
- ✅ 8,401개 자산 데이터 적재 성공
- ✅ 6개월 안에 운영 시작

---

### ADR-2: 데이터베이스로 MySQL 선택

**상태:** Accepted  
**날짜:** 2024-04-14

**결정:**
MySQL 8.x (utf8mb4 인코딩)

**근거:**
- 오픈소스, 무료 라이센스
- 한글 데이터 지원 (utf8mb4)
- ACID 트랜잭션 지원 (InnoDB)
- JDBC 드라이버 안정적

**제약사항:**
- 복제/분산 처리는 미포함
- 백업 전략은 별도 운영 절차 필요

---

### ADR-3: PreparedStatement 강제 사용

**상태:** Accepted  
**날짜:** 2024-04-15

**결정:**
모든 JDBC 쿼리는 PreparedStatement 사용 (SQL Injection 방어)

**이유:**
- OWASP Top 10: SQL Injection 방지
- 파라미터 바인딩으로 보안 향상

**현황:**
- ✅ search.jsp: PreparedStatement 구현 완료
- ✅ detail.jsp: PreparedStatement 구현 완료
- ✅ LoginServlet: PreparedStatement 구현 완료
- 📌 기타 JSP: 감사 필요

---

### ADR-4: 역할 기반 접근 제어 (Role-Based Access Control)

**상태:** Accepted  
**날짜:** 2024-04-15

**결정:**
6가지 역할 (student, assistant, professor, admin, guest, visitor)로 기능 제어

**구현:**
```java
String role = (String) session.getAttribute("loginRole");
if ("admin".equals(role)) {
    // 관리자만 접근 가능한 기능
}
```

**역할별 권한:**

| 역할 | 검색 | 예약 | 이전 신청 | 관리 |
|------|------|------|----------|------|
| student | ✅ | ✅ | ❌ | ❌ |
| assistant | ✅ | ✅ | ✅ | ✅ |
| professor | ✅ | ✅ | ❌ | ❌ |
| admin | ✅ | ✅ | ✅ | ✅ |
| guest | ✅ | ❌ | ❌ | ❌ |
| visitor | ✅ | ❌ | ❌ | ❌ |

---

### ADR-5: 페이지네이션 OFFSET 기반 방식

**상태:** Accepted  
**날짜:** 2024-04-15

**결정:**
LIMIT/OFFSET 방식으로 페이지네이션 구현 (20행/페이지)

**SQL 패턴:**
```sql
SELECT * FROM assets 
WHERE [filters]
ORDER BY reg_date DESC
LIMIT 20 OFFSET (page-1)*20
```

**한계:**
- 깊은 페이지 접근 시 성능 저하
- 8,401건 데이터 범위에서는 문제 없음

**개선 고려 (향후):**
- 커서 기반 페이지네이션 (keyset pagination)
- Elasticsearch 도입

---

### ADR-6: 세션 타임아웃 30분

**상태:** Accepted  
**날짜:** 2024-04-15

**결정:**
web.xml에서 session-timeout을 30분으로 설정

**근거:**
- 학사 행정 시스템의 보안 표준 (30~60분)
- 사용자 작업 시간 충분
- 장시간 미사용 시 자동 로그아웃 (보안)

```xml
<session-config>
    <session-timeout>30</session-timeout>
</session-config>
```

---

### ADR-7: UTF-8 강제 인코딩

**상태:** Accepted  
**날짜:** 2024-04-15

**결정:**
SetCharacterEncodingFilter로 모든 요청/응답을 UTF-8로 통일

**웹.xml:**
```xml
<filter>
    <filter-name>encodingFilter</filter-name>
    <filter-class>org.apache.catalina.filters.SetCharacterEncodingFilter</filter-class>
    <init-param>
        <param-name>encoding</param-name>
        <param-value>UTF-8</param-value>
    </init-param>
</filter>
```

**효과:**
- 한글 데이터 손상 방지
- 여러 문자 집합 통일

---

### ADR-8: 자산 삭제는 논리 삭제 (Soft Delete)

**상태:** Proposed

**검토 중인 결정:**
자산 폐기 시 DELETE가 아닌 asset_disposal 테이블 기록 + asset_status 변경

**근거:**
- 감사 추적 (audit trail) 유지
- 데이터 복구 가능성 보장
- 통계 및 분석 데이터 유지

**구현:**
```sql
-- 물리 삭제 대신 → 논리 삭제
UPDATE assets SET asset_status='폐기' WHERE asset_no=?;
INSERT INTO asset_disposal (asset_no, disposal_date, ...) VALUES (...);
```

---

### ADR-9: 비밀번호 암호화 전략 (Future)

**상태:** Proposed

**현황:**
현재 비밀번호가 평문으로 저장되어 있음 (보안 위험)

**권고:**
```
DB에 저장: BCrypt 또는 PBKDF2로 해싱된 비밀번호
검증 로직: passwordEncoder.matches(입력비밀번호, 저장된해시)
```

**예상 작업:**
- LoginServlet 수정
- 기존 사용자 비밀번호 마이그레이션
- 약 2-3일

---

### ADR-10: API 응답 형식 (Future)

**상태:** Proposed

**향후 API 추가 시:**
- JSON 형식 통일
- HTTP 상태 코드 준수 (200, 404, 500, etc.)
- Error Response 표준화

```json
{
  "success": true,
  "data": {...},
  "message": "요청 성공"
}

// Error
{
  "success": false,
  "error": {
    "code": "INVALID_PARAM",
    "message": "입력 파라미터가 잘못되었습니다"
  }
}
```

---

## 8. 설계 승인

| 역할 | 이름 | 서명 | 날짜 |
|------|------|------|------|
| 시스템 아키텍트 | _____________ | _____________ | ___/___/___ |
| 기술 리더 | _____________ | _____________ | ___/___/___ |
| 프로젝트 관리자 | _____________ | _____________ | ___/___/___ |

---

**문서 버전:** v1.0  
**최종 수정일:** 2026-05-20  
**파일명:** CAN_Design_Stage_20260520.md  
**Naming Convention:** {Project_Name}_{Stage}_{YYMMDD}.md
