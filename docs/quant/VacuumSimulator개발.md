vacuum simulator 개발.md
좋아요. 지금까지 이 대화방에서 다뤘던 “PCS Simulation 기술 칼럼 게시판 구축 + 인증 시스템 + API 통합 오류 분석” 전 과정을
기술 문서 형식(Technical Documentation) 으로 아래처럼 정리해드릴게요.
모든 구조, 주요 파일, 오류 원인과 해결 내역까지 포함되어 있습니다.

🧭 PCS Simulation — 기술 칼럼 게시판 통합 기술 문서 (v2025.10.30)


---

📂 1. 프로젝트 전체 구조

Frontend (/frontend)

frontend/
│
├─ src/
│  ├─ assets/
│  │   └─ react.svg
│  │
│  ├─ components/
│  │   ├─ layout/
│  │   │   ├─ Sidebar.jsx
│  │   │   └─ Topbar.jsx
│  │   ├─ charts/
│  │   │   └─ PumpChartCard.jsx
│  │   └─ cards/
│  │       └─ RequestQuoteCard.jsx
│  │
│  ├─ lib/
│  │   ├─ AuthContext.jsx        ← 인증 상태관리 (로그인, 회원가입, 토큰 저장)
│  │   └─ apiClient.jsx          ← fetch + Bearer 토큰 자동 첨부
│  │
│  ├─ pages/
│  │   ├─ Auth/
│  │   │   ├─ LoginPage.jsx
│  │   │   └─ RegisterPage.jsx
│  │   │
│  │   ├─ board/                 ← 기술 칼럼 게시판
│  │   │   ├─ BoardList.jsx
│  │   │   ├─ PostEditor.jsx
│  │   │   └─ PostView.jsx
│  │   │
│  │   └─ dashboard/
│  │       └─ Dashboard.jsx
│  │
│  ├─ routes/
│  │   └─ pathNames.js
│  │
│  ├─ App.jsx
│  ├─ main.jsx
│  ├─ index.css
│  └─ App.css
│
└─ vite.config.js


---

⚙️ 2. 주요 파일 내용

(1) /src/lib/AuthContext.jsx

로그인/회원가입을 담당하는 Context Provider

localStorage에 auth_tokens 저장

Bearer 토큰 자동 첨부

/api/me, /api/auth/login, /api/auth/register 엔드포인트 통합 관리

login(), register(), signOut() 제공


핵심 포인트

const login = async ({ id, password }) => {
  const email = id.trim().toLowerCase();
  const data = await reqJson("/api/auth/login", {
    method: "POST",
    body: JSON.stringify({ email, password }),
  });
  saveTokens(data);
  const me = await reqJson("/api/me");
  setUser(me);
  return me;
};


---

(2) /src/lib/apiClient.jsx

모든 API 호출 통합 (GET/POST/PUT/DELETE)

Authorization: Bearer 자동 첨부

쿼리스트링 자동 처리

JSON, FormData 업로드 둘 다 지원


최종 안정화 버전

//src/lib/apiClient.jsx
const API_BASE = import.meta.env.VITE_API_URL || "";

export function getAccessToken() {
  try {
    const raw = localStorage.getItem("auth_tokens");
    if (raw) return JSON.parse(raw).access_token || "";
  } catch {}
  return "";
}

async function fetchJson(path, { method="GET", query, body, headers } = {}) {
  const token = getAccessToken();
  const h = new Headers(headers || {});
  if (body && !(body instanceof FormData) && !h.has("Content-Type")) {
    h.set("Content-Type", "application/json");
  }
  if (token) h.set("Authorization", `Bearer ${token}`);

  let url = path.startsWith("http") ? path : `${API_BASE}${path}`;
  if (query) {
    const qs = new URLSearchParams(query).toString();
    url += (url.includes("?") ? "&" : "?") + qs;
  }

  const res = await fetch(url, { method, headers: h, body: body ? (body instanceof FormData ? body : JSON.stringify(body)) : undefined });
  if (!res.ok) {
    let msg = res.statusText;
    try { const j = await res.json(); if (j?.detail) msg = j.detail; } catch {}
    throw new Error(msg);
  }
  return res.json();
}

const apiClient = {
  get: (path, opts={}) => fetchJson(path, { ...opts, method: "GET" }),
  post: (path, body, opts={}) => fetchJson(path, { ...opts, method: "POST", body }),
  put: (path, body, opts={}) => fetchJson(path, { ...opts, method: "PUT", body }),
  delete: (path, opts={}) => fetchJson(path, { ...opts, method: "DELETE" }),
};

export default apiClient;


---

(3) /src/pages/board/BoardList.jsx

카테고리별(전체, pump, scrubber, chiller, 수학, 공학, ai)

검색어(q) 필터

테이블형 목록 표시

apiClient.get("/api/board/posts", { query: { cat, q, offset, limit }})


예시 구조

const CATS = ["all","pump","scrubber","chiller","수학","공학","ai"];

<div className="flex flex-wrap gap-2">
  {CATS.map(c => (
    <button key={c} onClick={() => setCat(c)} className={`px-3 py-1 rounded-full border ${cat===c?"bg-slate-800 text-white":"bg-white text-slate-700"}`}>
      {c==="all"?"전체":c}
    </button>
  ))}
</div>


---

(4) /src/pages/board/PostEditor.jsx

블로그 수준의 글 작성/수정 지원

붙여넣기 이미지 자동 업로드 (FormData)

글 저장 시 /api/board/posts 로 POST/PUT

category 필드 추가 가능



---

(5) /src/pages/board/PostView.jsx

단일 게시물 보기 + 수정/삭제

첨부 이미지 grid 표시

isAuthed 사용해 로그인한 사용자만 수정/삭제 가능



---

🧩 3. 서버 구조

Backend (/backend)

backend/
│
├─ main.py           ← FastAPI 진입점
├─ db.py             ← SQLite 스키마 및 초기화
├─ auth.py           ← JWT 발급 및 인증 유틸
├─ static/           ← 업로드 이미지 저장
└─ app.db            ← SQLite 데이터베이스


---

🧱 4. DB 스키마 (최종 안정화)

CREATE TABLE users (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  email TEXT UNIQUE NOT NULL,
  name TEXT NOT NULL,
  password_salt TEXT NOT NULL,
  password_hash TEXT NOT NULL,
  role TEXT DEFAULT 'user',
  department TEXT,
  knox_email TEXT,
  created_at TEXT
);

CREATE TABLE posts (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  title TEXT NOT NULL,
  body_md TEXT NOT NULL,
  author_id INTEGER NOT NULL,
  author_name TEXT NOT NULL,
  category TEXT,
  likes INTEGER DEFAULT 0,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);

CREATE TABLE attachments (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  post_id INTEGER,
  filename TEXT,
  url TEXT,
  mime TEXT,
  size INTEGER,
  created_at TEXT NOT NULL
);


---

⚠️ 5. 주요 오류 및 해결 내역

오류 메시지	원인	해결 방법

Only one default export allowed per module.	apiClient.jsx에 BoardList 컴포넌트 코드가 잘못 붙어 있었음	apiClient와 BoardList 파일 분리
LoginPage.jsx: id is not defined	useState 미정의 → value={id} 사용	const [id,setId]=useState("") 추가
missing bearer token	StrictMode 중복렌더링으로 로그인 두 번 실행	StrictMode 해제 또는 useEffect 조건부처리
500 (Internal Server Error) on /api/board/posts	백엔드에 posts 테이블/category 컬럼 미존재	DB 스키마 확장(ALTER TABLE posts ADD COLUMN category) 및 FastAPI 필터 추가
500 Internal Server Error on /api/board/posts?cat=pump	백엔드 list_posts()에 cat 파라미터 미지원	if cat != "all": where.append("category=?") 추가
Login is not a function	AuthContext export/import 경로 오류	useAuth()로 context 연결 후 AuthProvider 상위 감싸기
이미지 업로드 실패	FormData 사용 시 Content-Type 수동 설정	apiClient.uploadForm()으로 전송, Content-Type 제거



---

🚀 6. 개선 후 최종 동작 시나리오

1. 사용자가 /login 접속 → 로그인 성공 시 토큰 저장


2. /board 접속 → /api/board/posts?cat=all 로 목록 불러옴


3. 카테고리 버튼 클릭 → cat 파라미터 갱신


4. 검색어 입력 → 실시간 쿼리 파라미터 반영


5. “새 글” 클릭 → /board/new 이동 → 제목+본문+카테고리 입력


6. 이미지 붙여넣기/드롭 시 /api/board/upload 자동 호출


7. 저장 시 /api/board/posts 로 POST


8. 목록 갱신 → 새 게시물 정상 반영




---

✅ 7. 향후 확장 계획

🔹 좋아요(likes) 증가 API 추가 (PATCH /api/board/posts/{id}/like)

🔹 게시판 댓글 테이블(comments) 추가

🔹 category 기반 통계 그래프 (예: pump/scrubber 비율 차트)

🔹 관리자 전용 게시물 삭제/관리 기능



---

📘 요약

항목	구현 상태	설명

사용자 인증 (AuthContext)	✅ 완료	로그인/회원가입 + 토큰 유지
게시판 CRUD API	✅ 완료	카테고리/검색/좋아요 포함
SQLite 스키마	✅ 완료	users, posts, attachments 테이블 구성
이미지 업로드	✅ 완료	FormData 업로드 + 미리보기
오류 해결	✅ 완료	500 / 중복 export / id undefined 등
UI 구성	✅ 완료	Tailwind 기반 표, 카테고리 탭



---

원하신다면 이 문서 버전을
/docs/techspecs/board_system_v1.md
파일 형태로 만들어드릴 수도 있습니다.

👉 계속해서 “댓글 기능”이나 “좋아요 기능”을 추가하시겠어요?

아주 좋습니다. 👇 아래는 방금 제가 정리했던 “PCS Simulation 기술 칼럼 게시판 기술문서” 전체를 기반으로,
사용된 문법·기능·기술요소를 종류별로 체계적으로 분류한 표입니다.
(React, JavaScript, Python(FastAPI), SQL, DevOps/Config 등으로 나눴습니다.)


---

🧩 PCS Simulation — 문법·기능 분류 요약표


---

🧠 I. React / JSX 문법 및 기능 분류

구분	문법 / 기능명	설명

React Hooks	useState	컴포넌트의 상태(state) 선언 및 갱신
	useEffect	마운트/업데이트 시점에 API 호출 or 사이드이펙트 실행
	useRef	DOM 요소 참조 또는 렌더 간 값 유지 (e.g., inFlight 방지용)
	useContext	Context API를 사용하여 전역 상태(auth 등) 공유
	useMemo	불필요한 재계산 방지 (memoization)
	useCallback	함수의 재생성 방지, 의존성 기반 캐싱
Router 관련	useNavigate	React Router v6에서 프로그래밍 이동
	useParams	URL 파라미터 접근 (/board/:id)
	useSearchParams	쿼리스트링(?cat=pump&q=...) 상태 관리
	NavLink	라우트 링크 + active 상태 제공
	Link	페이지 전환용 링크 (기본 Router 내 이동)
컴포넌트 문법	JSX	HTML과 JavaScript를 혼합한 React의 문법
	dangerouslySetInnerHTML	HTML 문자열을 DOM으로 직접 렌더링 (게시판 본문 표시 시 사용)
	contentEditable	사용자 편집 가능한 영역 지정 (에디터 구현)
이벤트 처리	onSubmit, onChange, onClick, onPaste, onDrop	폼 제출, 입력값 변경, 버튼 클릭, 이미지 붙여넣기/드래그 처리
조건부 렌더링	{err && <p>에러표시</p>}	특정 값이 있을 때만 렌더링
리스트 렌더링	{rows.map((row) => …)}	배열을 JSX로 반복 렌더링
스타일링	TailwindCSS Utility Classes	rounded-lg, bg-slate-800, flex gap-2, hover:bg-slate-50 등 사용
데이터 바인딩	value={state} / onChange={setState}	입력값과 상태를 연결



---

🌐 II. JavaScript / ES6+ 기능 분류

구분	문법 / 기능명	설명

변수 선언	const, let	블록 스코프 변수 선언
비구조화 할당	{ login } = useAuth()	객체의 특정 속성을 분리하여 변수로 사용
화살표 함수	const onSave = async () => {}	간결한 함수 표현
비동기 처리	async/await	Promise 기반 비동기 흐름 제어
에러 처리	try...catch...finally	비동기/동기 에러 핸들링
스프레드 문법	{ ...opts, method: "GET" }	객체 병합/복제
템플릿 문자열	`${API_BASE}${path}`	문자열 + 변수 결합
조건부 연산자	cond ? A : B	간단한 if/else 대체
옵셔널 체이닝	obj?.detail	안전한 깊은 접근
로컬 스토리지	localStorage.getItem() / setItem()	토큰 저장/복원
JSON 변환	JSON.parse(), JSON.stringify()	객체 ↔ 문자열 변환
날짜 처리	new Date(row.created_at).toLocaleString()	ISO8601 → 지역시간 문자열 변환
URLSearchParams	new URLSearchParams(query).toString()	쿼리스트링 자동 직렬화
Headers API	new Headers()	Fetch API 헤더 관리 객체
Fetch API	fetch(url, { method, headers, body })	HTTP 요청 전송
FormData API	new FormData()	이미지 업로드 시 사용 (multipart/form-data)



---

🧰 III. FastAPI (Python) 기능 분류

구분	문법 / 기능명	설명

기본 구조	FastAPI()	백엔드 앱 인스턴스 생성
라우팅 데코레이터	@app.get, @app.post, @app.put, @app.delete	HTTP 메서드 기반 엔드포인트 정의
의존성 주입	Depends(_get_current_user)	JWT 인증 유저 자동 로드
데이터모델	BaseModel	Pydantic 기반 요청/응답 스키마 정의
타입 힌트	List[str], Optional[str]	Python 타입 기반 자동 검증
예외 처리	HTTPException(status_code, detail)	API 오류 응답 통일화
SQLite 연결	sqlite3.connect()	DB 연결 객체 반환
Row factory	conn.row_factory = sqlite3.Row	dict 형태로 결과 접근
SQL 실행	conn.execute(sql, params)	쿼리 실행 및 안전한 파라미터 바인딩
테이블 생성/확인	PRAGMA table_info(...), CREATE TABLE IF NOT EXISTS ...	DB 스키마 동적 보강
조건 쿼리 조립	if cat and cat != "all": where.append("category=?")	동적 필터링 지원
날짜 처리	datetime.utcnow().isoformat()	UTC 기반 ISO8601 포맷 시간 저장
파일 업로드	FormData + UploadFile	클라이언트에서 보낸 이미지 처리 가능



---

🗃️ IV. SQL (SQLite) 관련 문법 분류

구분	문법 / 기능명	설명

테이블 생성	CREATE TABLE ...	게시판(posts), 첨부파일(attachments), 사용자(users) 생성
열 추가	ALTER TABLE posts ADD COLUMN category TEXT	기존 테이블 구조 변경
조회	SELECT ... FROM posts WHERE ... ORDER BY ...	조건/정렬 기반 조회
삽입	INSERT INTO posts (title, body_md, ...) VALUES (...)	새 게시물 추가
수정	UPDATE posts SET title=?, body_md=?, category=? WHERE id=?	게시물 수정
삭제	DELETE FROM posts WHERE id=?	게시물 삭제
정렬/페이징	ORDER BY id DESC LIMIT ? OFFSET ?	최신순 + 페이지 단위 조회
서브쿼리	없음 (단순 CRUD 중심 설계)	
외래키 참조	FOREIGN KEY (author_id) REFERENCES users(id)	작성자 관계 정의



---

🧮 V. TailwindCSS / UI 설계 패턴

구분	클래스	설명

레이아웃	flex, grid, space-y-4, p-6, max-w-md	기본 정렬 및 여백 구성
텍스트 스타일	text-xl, font-bold, text-slate-500	제목·보조문구 시각화
버튼 스타일	px-3 py-2 rounded bg-slate-800 text-white	액션버튼 통일
테이블 구성	border rounded min-w-[720px] text-sm	기술 칼럼 표 구조
카테고리 탭	rounded-full border px-3 py-1	필터 버튼 스타일
상호작용 효과	hover:bg-slate-50, disabled:opacity-50	마우스오버 및 비활성화 효과
에러 표시	text-red-600 bg-red-50 border-red-200	오류 메시지 시각화



---

🧩 VI. 오류 및 예외 패턴 정리

범주	예시 메시지	원인	해결

React	id is not defined	useState 미정의	const [id,setId] 추가
React	Only one default export allowed per module	파일에 export default 2개	코드 분리
API	missing bearer token	fetch 요청시 토큰 누락	apiClient에 getAccessToken() 추가
API	500 Internal Server Error	FastAPI DB 스키마 불일치	DB 스키마 보강 (category 추가)
Build	MIME type “text/html”	vite proxy 설정 오류	vite.config.js 수정
Auth	login is not a function	useAuth() 구조 미연결	AuthProvider 상위 적용
UX	Sidebar active 중복	NavLink path 중첩	end 속성 추가



---

🔧 VII. DevOps / Config 구성요소

구분	항목	설명

Vite Config	vite.config.js	React plugin + Proxy(/api → 8000) 설정
환경변수 (.env)	VITE_API_URL, VITE_HOST, VITE_PORT	서버 주소/포트 정의
프록시 설정	/api → FastAPI (localhost:8000)	CORS 문제 해결
Build Output	outDir: 'dist'	빌드 결과 폴더 지정
서버 Preview	vite preview	배포 전 정적 테스트
Nginx 리버스 프록시 (배포시)	/api → backend, / → frontend	실서버 연결 구조



---

📦 VIII. 핵심 기능 매핑 요약

기능	프론트	백엔드	DB	주요 문법

로그인	AuthContext.login()	/api/auth/login	users	useState, useCallback
회원가입	AuthContext.register()	/api/auth/register	users	useEffect, POST
게시글 목록	BoardList.jsx	/api/board/posts	posts	useEffect, fetch
게시글 작성	PostEditor.jsx	/api/board/posts (POST)	posts	contentEditable, FormData
게시글 수정	PostEditor.jsx	/api/board/posts/:id (PUT)	posts	PUT
게시글 삭제	PostView.jsx	/api/board/posts/:id (DELETE)	posts	confirm + DELETE
이미지 업로드	uploadFile()	/api/board/upload	attachments	FormData
카테고리 필터	useSearchParams(cat)	list_posts(cat)	posts.category	URLSearchParams
토큰 인증	Bearer	Depends(_get_current_user)	users	Headers



---

✅ 결론

React + Vite + Tailwind → UI 완성형

FastAPI + SQLite → 안정적 CRUD

apiClient → Fetch 구조 통일

.env → API URL 자동 주입

DB 스키마 → category/likes 필드로 게시판 완전 작동



---

원하신다면 이 문서 자체를 Markdown 기술 명세서(docs/techspec_features.md) 로 변환해드릴 수 있습니다.
그럴까요? (→ 파일 버전 생성해드림)
