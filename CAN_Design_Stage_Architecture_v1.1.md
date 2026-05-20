# CAN 프로젝트 - 아키텍처 설계서 (개선판)

**Document ID:** CAN_Design_Architecture_v1.1_260520  
**Project Name:** ICT CampusNav (교내 자원 내비게이션 시스템)  
**작성일:** 2026-05-20

---

## 1️⃣ 시스템 구성도 (System Architecture)

### 1.1 계층 구조 (Layered Architecture)

```
┌─────────────────────────────────────────────────┐
│  🎯 사용자 (6가지 역할)                          │
│  학부생 | 조교 | 교수 | 관리자 | 게스트 | 외부인 │
└────────────────────┬────────────────────────────┘
                     │
┌─────────────────────▼────────────────────────────┐
│  🌐 Web Browser (클라이언트)                     │
│  HTML5 / CSS3 / JavaScript                      │
│  Bootstrap 5.3.3 / 반응형 디자인                 │
└────────────────────┬────────────────────────────┘
                     │
        HTTP(S) Request / Response
                     │
┌─────────────────────▼────────────────────────────┐
│  📦 Apache Tomcat 9.x Container                 │
├─────────────────────────────────────────────────┤
│                                                 │
│  ┌─────────────────────────────────────────┐  │
│  │ 🔐 Servlet Layer (인증, 권한)           │  │
│  ├─────────────────────────────────────────┤  │
│  │  • LoginServlet (/login)                │  │
│  │  • LogoutServlet (/logout)              │  │
│  │  • GuestServlet (/guest)                │  │
│  │  • VisitorServlet (/visitor)            │  │
│  └─────────────────────────────────────────┘  │
│                      │                         │
│                      │ Forward/Redirect        │
│                      ▼                         │
│  ┌─────────────────────────────────────────┐  │
│  │ 🎨 JSP Layer (비즈니스 로직, 표현)      │  │
│  ├─────────────────────────────────────────┤  │
│  │  • campuslogin.jsp  - 로그인            │  │
│  │  • search.jsp       - 자산 검색         │  │
│  │  • detail.jsp       - 상세 정보         │  │
│  │  • reserve.jsp      - 예약 관리         │  │
│  │  • transfer.jsp     - 자산 이전         │  │
│  │  • professor.jsp    - 교수 관리         │  │
│  │  • asset_manage.jsp - 자산 관리(관리자) │  │
│  │  • floorNav.jsp     - 층별 네비게이션   │  │
│  │  • main_[role].jsp  - 역할별 메인      │  │
│  └─────────────────────────────────────────┘  │
│                      │                         │
│                      │ JDBC                    │
│                      ▼                         │
│  ┌─────────────────────────────────────────┐  │
│  │ 🛠️ Utility Layer (공통 기능)             │  │
│  ├─────────────────────────────────────────┤  │
│  │  • DBUtil.java - DB 연결 & 리소스 정리  │  │
│  │  • Session Manager - 사용자 세션 관리   │  │
│  │  • UTF-8 Filter - 문자 인코딩           │  │
│  └─────────────────────────────────────────┘  │
│                                                 │
└────────────────────┬────────────────────────────┘
                     │
           TCP/IP (포트 3306)
                     │
┌─────────────────────▼────────────────────────────┐
│  🗄️ MySQL 8.x Database (campusnav)             │
├─────────────────────────────────────────────────┤
│  • assets             - 자산 정보 (8,401건)    │
│  • users              - 사용자 정보             │
│  • asset_transfer     - 자산 이전 기록         │
│  • asset_disposal     - 자산 폐기 기록         │
│  • reservations       - 자산 예약 정보         │
│  • professors         - 교수 정보              │
│  • prof_subjects      - 교수 담당 과목         │
│  • prof_skills        - 교수 스킬              │
└─────────────────────────────────────────────────┘
```

---

### 1.2 Servlet 목록

| Servlet | 경로 | 역할 | 요청 |
|---------|------|------|------|
| **LoginServlet** | `/CAN/login` | 로그인 처리, 세션 생성 | POST |
| **LogoutServlet** | `/CAN/logout` | 세션 무효화, 로그아웃 | GET |
| **GuestServlet** | `/CAN/guest` | 게스트 접근 설정 | GET |
| **VisitorServlet** | `/CAN/visitor` | 외부인 접근 설정 | GET |

---

### 1.3 JSP 페이지 목록

| JSP 파일 | 설명 | 접근 권한 |
|---------|------|---------|
| `campuslogin.jsp` | 로그인/회원가입 | 전체 |
| `search.jsp` | 자산 검색 | 로그인 사용자 |
| `detail.jsp` | 자산 상세 정보 | 로그인 사용자 |
| `reserve.jsp` | 자산 예약 | 학부생, 조교, 교수 |
| `transfer.jsp` | 자산 이전 기록 | 조교, 관리자 |
| `professor.jsp` | 교수 정보 관리 | 관리자 |
| `asset_manage.jsp` | 자산 관리 | 관리자 |
| `floorNav.jsp` | 층별 내비게이션 | 로그인 사용자 |
| `main_student.jsp` | 학부생 메인 | 학부생 |
| `main_assistant.jsp` | 조교 메인 | 조교 |
| `main_professor.jsp` | 교수 메인 | 교수 |
| `main_admin.jsp` | 관리자 메인 | 관리자 |
| `main_guest.jsp` | 게스트 메인 | 게스트 |
| `main_visitor.jsp` | 방문자 메인 | 외부인 |

---

## 2️⃣ 요청 흐름도 (Request Flow)

### 2.1 인증 플로우 (Authentication Flow)

**단계별 처리:**

1. **사용자 입력**
   - `campuslogin.jsp`에서 ID/PW 입력

2. **서블릿 처리**
   - `POST /CAN/login` → `LoginServlet`

3. **데이터베이스 조회**
   ```sql
   SELECT user_name, role FROM users 
   WHERE user_id=? AND user_pw=? AND use_yn='Y'
   ```

4. **성공 시 처리**
   - ✅ HttpSession 생성
   - ✅ `loginUser`, `loginName`, `loginRole` 속성 설정
   - ✅ 쿠키 저장 (선택)
   - ✅ 역할별 메인 페이지로 리다이렉트

5. **실패 시 처리**
   - ❌ 에러 메시지 표시
   - ❌ 로그인 폼으로 Forward

---

### 2.2 데이터 조회 플로우 (Search Flow)

**단계별 처리:**

1. **페이지 로드**
   - `search.jsp` 접근

2. **세션 검증**
   ```java
   if(session.getAttribute("loginUser") == null) {
       response.sendRedirect("/CAN/campuslogin.jsp");
   }
   ```

3. **동적 WHERE 절 구성**
   - 키워드 검색: `item_name LIKE ?`
   - 카테고리 필터: `asset_class = ?`
   - 상태 필터: `asset_status LIKE ?`

4. **전체 건수 조회**
   ```sql
   SELECT COUNT(*) FROM assets WHERE [동적 필터]
   ```

5. **페이지 단위 조회**
   ```sql
   SELECT * FROM assets 
   WHERE [동적 필터]
   ORDER BY reg_date DESC
   LIMIT 20 OFFSET (page-1)*20
   ```

6. **결과 렌더링**
   - 검색 결과 테이블 표시
   - 페이지네이션 컨트롤 표시

---

## 3️⃣ 컴포넌트 역할 (Component Responsibilities)

| 컴포넌트 | 책임 | 의존성 |
|---------|------|--------|
| **Servlet** | HTTP 요청 처리, 인증, 세션 관리 | Tomcat, DBUtil |
| **JSP** | HTML 렌더링, JDBC 쿼리 실행 | Servlet, DBUtil, MySQL |
| **DBUtil** | DB 연결 풀, 리소스 정리 | MySQL JDBC Driver |
| **Session** | 사용자 상태 유지 | Tomcat Session Manager |
| **Filter** | 요청/응답 처리, 문자 인코딩 | Apache Tomcat |

---

## 4️⃣ 데이터 흐름 (Data Flow)

### 자산 검색 데이터 흐름

```
사용자 입력 (keyword, type, status)
         │
         ▼
search.jsp
         │
    (파라미터 파싱)
         │
         ▼
PreparedStatement 생성
         │
    (동적 WHERE 절)
         │
         ▼
MySQL 쿼리 실행
         │
    (2단계: COUNT → SELECT)
         │
         ▼
ResultSet → List<Map>
         │
    (데이터 변환)
         │
         ▼
HTML 테이블 렌더링
         │
    (결과 표시)
         │
         ▼
브라우저에 전송
```

---

## 5️⃣ 보안 아키텍처 (Security Architecture)

| 계층 | 보안 조치 |
|------|---------|
| **요청 필터링** | UTF-8 인코딩 필터, 문자 검증 |
| **인증** | LoginServlet DB 검증, Session 기반 |
| **인가** | 역할 기반 접근 제어 (RBAC) |
| **데이터베이스** | PreparedStatement (SQL Injection 방지) |
| **세션** | 30분 타임아웃, HttpOnly 쿠키 |

---

## 6️⃣ 사용자 역할 및 권한 (RBAC)

| 역할 | 코드 | 검색 | 예약 | 이전신청 | 자산관리 |
|------|------|------|------|---------|---------|
| 학부생 | `student` | ✅ | ✅ | ❌ | ❌ |
| 조교 | `assistant` | ✅ | ✅ | ✅ | ✅ |
| 교수 | `professor` | ✅ | ✅ | ❌ | ❌ |
| 관리자 | `admin` | ✅ | ✅ | ✅ | ✅ |
| 게스트 | `guest` | ✅ | ❌ | ❌ | ❌ |
| 외부인 | `visitor` | ✅ | ❌ | ❌ | ❌ |

---

## 7️⃣ 배포 환경 (Deployment Environment)

**서버 정보:**
- **Web Server:** Apache Tomcat 9.0
- **Database:** MySQL 8.0+
- **Java:** JDK 8+
- **Encoding:** UTF-8

**경로 설정:**
```
Tomcat ROOT: C:/Program Files/Apache Software Foundation/Tomcat 9.0/webapps/ROOT
CAN App: /CAN
Web URL: http://localhost:8080/CAN
```

---

## ✅ 마크다운 렌더링 팁

더 나은 보기 경험을 위해 **다음 중 하나를 선택하세요:**

### 📖 Option 1: VS Code 마크다운 미리보기 (추천)
- **`Ctrl + Shift + V`** 누르기
- 또는 우상단 **미리보기 버튼** 클릭

### 🔧 Option 2: VS Code 폰트 설정
`settings.json`에 추가:
```json
"[markdown]": {
    "editor.fontFamily": "Courier New, monospace",
    "editor.fontSize": 11
}
```

### 🌐 Option 3: GitHub 웹 에디터
- GitHub 저장소 웹사이트에서 파일을 열면 완벽한 렌더링 제공

---

**문서 버전:** v1.1  
**작성자:** AI 소프트웨어학과  
**최종 수정:** 2026-05-20
