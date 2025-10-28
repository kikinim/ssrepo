## Tailwind 설정

카드처럼 안 보이면 거의 100% Tailwind가 안 붙어있기 때문입니다. 
아래 순서대로 점검/수정하면 카드 스타일로 보일 것입니다.

### 1. Tailwind 설치 여부 확인 & 설치

프로젝트 루트에서:

```bash
npm i -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

### 2. tailwind.config.js 설정

루트의 tailwind.config.js 파일을 이렇게 바꿔주세요 (content 경로가 중요합니다).

```javascript
// tailwind.config.js
/** @type {import('tailwindcss').Config} */
export default {
  content: ["./index.html", "./src/**/*.{js,jsx,ts,tsx}"],
  theme: { extend: {} },
  plugins: [],
};
```

### 3. postcss.config.js 확인

자동 생성됐을 텐데 혹시 다르면 이렇게:

```javascript
// postcss.config.js
export default {
  plugins: {
    tailwindcss: {},
    autoprefixer: {},
  },
};
```

### 4. 글로벌 CSS에 Tailwind 지시어 들어있는지 확인

src/styles/index.css 안에 이 3줄이 꼭 있어야 합니다.

```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

그리고 src/main.jsx에서 이 파일을 임포트하고 있는지 확인:

```javascript
import "./styles/index.css";
```

### 5. 개발 서버 재시작

설정 바꾸면 Vite 재시작이 안전합니다:

```bash
npm run dev
```

### 6. 바로 확인용(선택): 인라인 스타일 폴백 한 줄

혹시 Tailwind가 여전히 안 먹든지 빠르게 판단하려면 PumpChartCard에 인라인 스타일을 잠깐 추가해봅니다. 카드 테두리가 바로 보이면 컴포넌트는 정상이고 Tailwind만 문제인 거라 확정됩니다.

// src/components/charts/PumpChartCard.jsx
```javascript
export default function PumpChartCard({ title = "Pump Chart", caption }) {
  return (
    <div
      className="rounded-xl border border-white/10 bg-neutral-900 p-4"
      style={{ border: "1px solid rgba(255,255,255,0.15)", borderRadius: 12 }} // ← 임시 폴백
    >
      <div className="flex items-center justify-between mb-3">
        <h3 className="text-base font-semibold">{title}</h3>
        <span className="text-xs text-white/60">placeholder</span>
      </div>
      <div className="h-56 rounded-lg bg-gradient-to-br from-neutral-800 to-neutral-900 grid place-items-center">
        <div className="text-white/50 text-sm">Chart Placeholder</div>
      </div>
      {caption ? <p className="mt-3 text-xs text-white/60">{caption}</p> : null}
    </div>
  );
}
```

이 인라인 테두리가 보이는데 Tailwind 배경/글씨 색이 안 먹으면 Tailwind 설정 문제고, 인라인 테두리도 안 보이면 렌더 트리(라우팅/레이아웃) 문제입니다.

## 통신 문제

프런트에서 “failed to fetch”가 뜨는 건 보통 네트워크(CORS/프록시/URL/서버 미기동) 문제입니다.

### 0. Uvicorn 실행 커맨드 정확히

```bash
uvicorn backend.main:app --host 0.0.0.0 --port 8000
```

(콜론 없이 --host 0.0.0.0 / --port 8000)

### 1. 프런트 요청 URL을 프록시로 바꾸기 (권장)

Vite 개발 서버에서:

vite.config.js에 프록시 추가

```javascript
export default defineConfig({
  plugins: [react()],
  server: {
    proxy: {
      '/api': {
        target: 'http://localhost:8000',
        changeOrigin: true,
        rewrite: (p) => p.replace(/^\/api/, ''),
      },
    },
  },
})
```

프런트 fetch는 무조건 상대 경로(/api/calculate)로:

```javascript
await fetch('/api/calculate', { ... })
```

### 2. 백엔드: 파일 경로/타입/헬스체크 보강 (안전 패치)

### 3. CORS는 이미 OK (하지만 프록시 쓰면 더 깔끔)

### 4. 프런트에서 꼭 넣을 “안전 가드”

서버 500이라도 “failed to fetch”는 아닙니다. 네트워크 에러가 아니면 HTTP 에러로 떨어지게 해두면 디버깅이 빨라집니다.

```javascript
const res = await fetch('/api/calculate', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify(payload),
})

if (!res.ok) {
  const text = await res.text()
  throw new Error(`HTTP ${res.status} ${res.statusText} :: ${text}`)
}

const data = await res.json()
if (!Array.isArray(data)) throw new Error('Invalid response (expected array of {x,y})')

setResult(data)
```

## 가상머신으로 복사, 빌드

프론트엔드(React/Vite) → 브라우저가 먹는 건 빌드 결과물(dist) 뿐입니다. dist만 서버에 두면 됩니다.

백엔드(FastAPI) → “빌드”란 게 따로 없어서, 소스 코드 자체(예: backend/main.py, data/ 등)** 를 서버에 둬야 합니다.

## 배포

### rsync로 전체 프로젝트 동기화

0) 서버에 rsync 설치

```bash
# (Ubuntu VM)
sudo apt update
sudo apt install -y rsync
```

1) 서버 디렉터리 준비

당신이 말한 배포 경로는 /home/doyul/pcs 이니까 거기에 넣자.

```bash
# (Ubuntu VM)
mkdir -p /home/doyul/pcs
# 배포 계정이 'doyul'이라면 소유자 지정
sudo chown -R doyul:doyul /home/doyul/pcs
```

2) 로컬→서버 동기화 (전체 소스)

> 중요: node_modules, dist, .git 같은 덩치 큰/불필요 폴더는 제외하고 올리는 게 일반적이야.
(프론트는 보통 로컬/CI에서 빌드해서 dist/만 서버로 올리거나, 서버에서 npm ci && npm run build를 돌리면 됨)

```bash
# 로컬(개발 PC)에서 실행 — 현재 디렉터리가 프로젝트 루트라고 가정
# 예: ~/Desktop/pcssimulationnew  (backend / frontend 둘 다 포함)
rsync -avz --delete \
  --exclude ".git" \
  --exclude "node_modules" \
  --exclude "frontend/node_modules" \
  --exclude "frontend/dist" \
  --exclude "backend/__pycache__" \
  ./  doyul@<서버IP or 도메인>:/home/doyul/pcs/
```

-a : 권한/타임스탬프 보존(archive)

-v : verbose

-z : 전송 중 압축

--delete : 서버 쪽에서 로컬에 없는 파일은 삭제(서버를 로컬과 “동기화”)

./ : 로컬 프로젝트 루트 (마지막 슬래시 중요)

doyul@host:/home/doyul/pcs/ : 서버 타깃 폴더(마지막 슬래시 중요)

먼저 미리보기(드라이런)로 확인 권장

```bash
rsync -avzn --delete [같은 옵션들...] ./  doyul@<서버>:/home/doyul/pcs/
# -n : 실제로는 복사 안 하고 “무엇을 할지”만 출력
```

3) 프론트 빌드 산출물만 따로 올리고 싶다면

(프론트는 로컬에서 빌드 → dist만 올리기)

```bash
# 로컬
cd frontend
npm ci
npm run build    # dist/ 생성

# dist만 서버로
rsync -avz --delete ./dist/  doyul@<서버>:/home/doyul/pcs/frontend/dist/
```

4) 반대로 서버→로컬 내려받기(백업)

```bash
rsync -avz --delete \
  doyul@<서버>:/home/doyul/pcs/  ./backup-pcs/
```

5) Windows에서 rsync가 없다면

WSL(Windows Subsystem for Linux) 터미널에서 위 같은 rsync 명령 그대로 사용 가능 (추천)

또는 Git Bash + rsync 설치본(cwRsync/MSYS2) 사용

급하면 scp -r도 가능(동기화가 아니라 “복사”라서 삭제 동기화는 안 됨):

```bash
scp -r ./  doyul@<서버>:/home/doyul/pcs/
```

### “var 폴더”는 뭐야?

/var 는 리눅스 표준 디렉터리로 가변 데이터(logs, caches, spool, 웹 정적 루트 등)를 넣는 곳입니다.

흔히 Nginx/Apache의 웹루트가 /var/www/html이지만, 반드시 /var 를 쓸 필요는 없습니다.

지금처럼 개인 계정(home/doyul/pcs) 아래에 배포해도 전혀 문제 없습니다.
(다만 Nginx로 정적 파일을 서비스할 땐 /var/www/pcs 같은 경로가 관습적으로 편하긴 해. 필요하면 심볼릭 링크로도 연결 가능)

예: /var/www/pcs로 정적만 서비스하고 싶다면

```bash
sudo mkdir -p /var/www/pcs
sudo rsync -avz --delete /home/doyul/pcs/frontend/dist/ /var/www/pcs/
# 또는 링크
sudo ln -s /home/doyul/pcs/frontend/dist /var/www/pcs
```

하지만 당장은 /home/doyul/pcs 로 충분합니다. 시스템 서비스(systemd)나 리버스 프록시(Nginx)가 그 경로를 바라보면 끝.

## 로그인 기능 구현

좋아, 바로 붙일 수 있게 프론트(React) + 백엔드(FastAPI) 최소 구현 세트입니다.

### 프론트엔드

1) 토큰 & 유저 상태: src/lib/auth.js

```javascript
// src/lib/auth.js
// JWT를 localStorage에 저장하고 fetch에 자동으로 Authorization 달아주는 헬퍼

const TOKEN_KEY = "auth.jwt";

export function getToken() {
  return localStorage.getItem(TOKEN_KEY) || null;
}
export function setToken(t) {
  if (t) localStorage.setItem(TOKEN_KEY, t);
}
export function clearToken() {
  localStorage.removeItem(TOKEN_KEY);
}

// 공용 fetch 래퍼
export async function apiFetch(path, { method="GET", headers={}, body, auth=true } = {}) {
  const h = { "Content-Type": "application/json", ...headers };
  if (auth) {
    const token = getToken();
    if (token) h["Authorization"] = `Bearer ${token}`;
  }
  const res = await fetch(path, { method, headers: h, body: body ? JSON.stringify(body) : undefined, credentials: "include" });
  const isJson = (res.headers.get("content-type") || "").includes("application/json");
  const data = isJson ? await res.json().catch(() => ({})) : await res.text();
  if (!res.ok) {
    const msg = (isJson && (data?.detail || data?.message)) || (typeof data === "string" ? data : "Request failed");
    throw new Error(msg);
  }
  return data;
}
```

2) Auth Context: src/components/AuthProvider.jsx

```javascript
// src/components/AuthProvider.jsx
import React, { createContext, useContext, useEffect, useState } from "react";
import { apiFetch, getToken, setToken, clearToken } from "../lib/auth";

const AuthCtx = createContext(null);
export const useAuth = () => useContext(AuthCtx);

export default function AuthProvider({ children }) {
  const [user, setUser] = useState(null);
  const [hydrated, setHydrated] = useState(false);

  // 앱 시작 시 토큰이 있으면 /api/auth/me 로 유저 복구
  useEffect(() => {
    (async () => {
      try {
        if (getToken()) {
          const me = await apiFetch("/api/auth/me");
          setUser(me);
        }
      } catch (_e) {
        clearToken();
      } finally {
        setHydrated(true);
      }
    })();
  }, []);

  async function login({ email, password }) {
    try {
      const data = await apiFetch("/api/auth/login", {
        method: "POST",
        auth: false,
        body: { email, password },
      });
      setToken(data.token);
      const me = await apiFetch("/api/auth/me");
      setUser(me);
      return { ok: true };
    } catch (e) {
      // 세분화된 메시지
      const msg = String(e.message || "");
      if (/not found/i.test(msg)) return { ok: false, message: "존재하지 않는 계정입니다." };
      if (/password/i.test(msg) || /invalid credentials/i.test(msg)) return { ok: false, message: "비밀번호가 올바르지 않습니다." };
      return { ok: false, message: msg || "로그인 실패" };
    }
  }

  async function register({ name, email, password }) {
    try {
      await apiFetch("/api/auth/register", {
        method: "POST",
        auth: false,
        body: { name, email, password },
      });
      // 가입 후 자동 로그인
      const res = await login({ email, password });
      if (!res.ok) return { ok: false, message: "가입은 되었지만 자동 로그인에 실패했습니다." };
      return { ok: true };
    } catch (e) {
      const msg = String(e.message || "");
      if (/exists/i.test(msg) || /already/i.test(msg)) return { ok: false, message: "이미 등록된 이메일입니다." };
      if (/weak/i.test(msg)) return { ok: false, message: "비밀번호가 너무 약합니다." };
      return { ok: false, message: msg || "회원가입 실패" };
    }
  }

  function logout() {
    clearToken();
    setUser(null);
  }

  return (
    <AuthCtx.Provider value={{ user, hydrated, login, register, logout }}>
      {children}
    </AuthCtx.Provider>
  );
}
```

3) Topbar: 로그인 상태 반영 (예시)

```javascript
// src/components/Topbar.jsx
import React from "react";
import { useAuth } from "./AuthProvider";
import { Link } from "react-router-dom";

export default function Topbar() {
  const { user, hydrated, logout } = useAuth();

  return (
    <div className="w-full h-12 border-b bg-white px-4 flex items-center">
      <div className="font-semibold">PCS Simulation</div>
      <div className="ml-auto flex items-center gap-3">
        {!hydrated ? null : user ? (
          <>
            <span className="text-sm text-slate-700">안녕하세요, <b>{user.name}</b>님</span>
            <button onClick={logout} className="text-sm px-3 py-1 rounded-md border hover:bg-slate-50">로그아웃</button>
          </>
        ) : (
          <>
            <Link to="/login" className="text-sm px-3 py-1 rounded-md border hover:bg-slate-50">로그인</Link>
            <Link to="/register" className="text-sm px-3 py-1 rounded-md border hover:bg-slate-50">회원가입</Link>
          </>
        )}
      </div>
    </div>
  );
}
```

4) 로그인/회원가입 페이지

```javascript
// src/pages/auth/Login.jsx
import React, { useState } from "react";
import { useAuth } from "../../components/AuthProvider";
import { useNavigate } from "react-router-dom";

export default function Login() {
  const { login } = useAuth();
  const nav = useNavigate();
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [msg, setMsg] = useState("");

  async function onSubmit(e) {
    e.preventDefault();
    setMsg("");
    const res = await login({ email, password });
    if (res.ok) nav("/");
    else setMsg(res.message || "로그인 실패");
  }

  return (
    <div className="max-w-md mx-auto mt-10 p-6 border rounded-xl bg-white">
      <h1 className="text-lg font-semibold mb-4">로그인</h1>
      {msg && <div className="mb-3 text-sm text-rose-600">{msg}</div>}
      <form onSubmit={onSubmit} className="space-y-3">
        <input className="w-full h-10 border rounded px-3" placeholder="이메일" value={email} onChange={e=>setEmail(e.target.value)} />
        <input className="w-full h-10 border rounded px-3" placeholder="비밀번호" type="password" value={password} onChange={e=>setPassword(e.target.value)} />
        <button className="w-full h-10 rounded bg-indigo-600 text-white">로그인</button>
      </form>
    </div>
  );
}
```

```javascript
// src/pages/auth/Register.jsx
import React, { useState } from "react";
import { useAuth } from "../../components/AuthProvider";
import { useNavigate } from "react-router-dom";

export default function Register() {
  const { register } = useAuth();
  const nav = useNavigate();
  const [name, setName] = useState("");
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [msg, setMsg] = useState("");

  async function onSubmit(e) {
    e.preventDefault();
    setMsg("");
    const res = await register({ name, email, password });
    if (res.ok) nav("/");
    else setMsg(res.message || "회원가입 실패");
  }

  return (
    <div className="max-w-md mx-auto mt-10 p-6 border rounded-xl bg-white">
      <h1 className="text-lg font-semibold mb-4">회원가입</h1>
      {msg && <div className="mb-3 text-sm text-rose-600">{msg}</div>}
      <form onSubmit={onSubmit} className="space-y-3">
        <input className="w-full h-10 border rounded px-3" placeholder="이름" value={name} onChange={e=>setName(e.target.value)} />
        <input className="w-full h-10 border rounded px-3" placeholder="이메일" value={email} onChange={e=>setEmail(e.target.value)} />
        <input className="w-full h-10 border rounded px-3" placeholder="비밀번호(8자 이상 추천)" type="password" value={password} onChange={e=>setPassword(e.target.value)} />
        <button className="w-full h-10 rounded bg-indigo-600 text-white">가입하기</button>
      </form>
    </div>
  );
}
```

5) 라우팅에 연결

```javascript
// src/App.jsx (예시)
import React from "react";
import { Routes, Route } from "react-router-dom";
import AuthProvider from "./components/AuthProvider";
import Topbar from "./components/Topbar";
import Login from "./pages/auth/Login";
import Register from "./pages/auth/Register";
import PumpSimulatorV2 from "./pages/simulations/PumpSimulatorV2";

export default function App() {
  return (
    <AuthProvider>
      <Topbar />
      <Routes>
        <Route path="/" element={<PumpSimulatorV2 />} />
        <Route path="/login" element={<Login />} />
        <Route path="/register" element={<Register />} />
      </Routes>
    </AuthProvider>
  );
}
```

### 백엔드(FastAPI 예시)

JWT 발급/검증만 갖춘 최소 엔드포인트. (이미 백엔드가 있다면 경로/필드만 맞춰줘도 됨)

```python
# backend/auth_api.py
from fastapi import APIRouter, HTTPException, Depends
from pydantic import BaseModel, EmailStr
from datetime import datetime, timedelta
import jwt

router = APIRouter(prefix="/api/auth", tags=["auth"])
JWT_SECRET = "change-me"
JWT_ALG = "HS256"
ACCESS_TTL = timedelta(days=7)

# 데모용 in-memory "DB"
USERS = {}  # email -> {"name":..., "email":..., "password":...}

class LoginIn(BaseModel):
    email: EmailStr
    password: str

class RegisterIn(BaseModel):
    name: str
    email: EmailStr
    password: str

def make_token(payload: dict):
    to_encode = {**payload, "exp": datetime.utcnow() + ACCESS_TTL}
    return jwt.encode(to_encode, JWT_SECRET, algorithm=JWT_ALG)

def verify_token(token: str):
    try:
        return jwt.decode(token, JWT_SECRET, algorithms=[JWT_ALG])
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Token expired")
    except Exception:
        raise HTTPException(status_code=401, detail="Invalid token")

def get_current_user(authorization: str = ""):
    if not authorization.startswith("Bearer "):
        raise HTTPException(status_code=401, detail="Missing token")
    token = authorization.split(" ", 1)[1].strip()
    data = verify_token(token)
    email = data.get("email")
    u = USERS.get(email)
    if not u:
        raise HTTPException(status_code=404, detail="User not found")
    return u

@router.post("/register")
def register(body: RegisterIn):
    if body.email in USERS:
        raise HTTPException(status_code=400, detail="User already exists")
    if len(body.password) < 6:
        raise HTTPException(status_code=400, detail="Password too weak")
    USERS[body.email] = {"name": body.name, "email": body.email, "password": body.password}
    return {"ok": True}

@router.post("/login")
def login(body: LoginIn):
    u = USERS.get(body.email)
    if not u:
        raise HTTPException(status_code=404, detail="User not found")
    if u["password"] != body.password:
        raise HTTPException(status_code=401, detail="Invalid credentials: password")
    token = make_token({"email": u["email"]})
    return {"token": token}

@router.get("/me")
def me(authorization: str = Depends(lambda authorization: authorization)):
    user = get_current_user(authorization)
    return {"name": user["name"], "email": user["email"]}

# backend/main.py (라우터 포함)
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from auth_api import router as auth_router

app = FastAPI()

# Vite 프론트 도메인/포트에 맞춰 조정
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5173", "http://pcssimulation.samsungds.net"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(auth_router)
# 기존 /api-v2/calculate 등도 여기에 include_router(...)
```

### 연결 체크리스트

1. 프론트 빌드 전:

* AuthProvider가 App의 최상단을 감싸는지
* Topbar가 useAuth() 사용해서 상태를 표시하는지
* 라우팅에 /login, /register 추가했는지

2. 백엔드:

* /api/auth/login, /api/auth/register, /api/auth/me 경로가 위와 일치하는지
* CORS allow_origins에 프론트 주소 추가했는지

3. 실패 메시지 세분화:

* 존재 X → “존재하지 않는 계정”
* 비밀번호 오류 → “비밀번호가 올바르지 않습니다”
* 중복 이메일 → “이미 등록된 이메일입니다”
* 약한 비번 → “비밀번호가 너무 약합니다”

필요하면 비밀번호 해시(bcrypt), DB 연동(SQLModel/SQLAlchemy), 이메일 검증, 이메일 인증(OTP)도 바로 확장해줄게요.
