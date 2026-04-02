# [1부 전용] macOS + Docker + OpenClaw + Gemini 셋업 가이드
> 이 문서는 1부 강의에서 도커 설치부터 Gemini API 키까지의 OpenClaw 전체 셋업 과정에 어려움을 겪는 분들을 위한 가이드 문서입니다.
> 이 문서는 **1부를 macOS에서 진행하는 수강생**이  
> Docker + OpenClaw + Gemini 셋업을 마치고,  
> **대시보드 접속 → 페어링 → 채팅 첫 인사**까지 도달하는 것을 목표로 합니다.
> 1부 방식에 맞춰 **토큰은 .env 파일**, **Gemini API 키는 온보딩 시 직접 붙여넣어 `auth-profiles.json`에 저장**합니다.

---

<br/>

## 0. 전체 흐름 한눈에 보기

1. Docker Desktop for Mac 설치  
2. OpenClaw 프로젝트 폴더 생성  
3. `.env` 파일에 토큰/경로/포트 설정  
4. `docker-compose.yml` 작성 및 컨테이너 실행  
5. `openclaw.json` 최소 설정 추가  
6. 온보딩(onboard)으로 Gemini 연결 (키 직접 붙여넣기)  
7. 브라우저에서 대시보드 접속 → 페어링 → 채팅 첫 인사

---

<br/>


## 1. Docker Desktop 설치 (macOS)

### 1-1. 사전 확인

- 칩셋: Apple Silicon(M1/M2/M3) 또는 Intel  
- macOS 버전: Docker Desktop 최소 요구사항 이상  
- RAM: 8GB 이상 권장

<br/>


### 1-2. 설치

1. Docker 공식 사이트 접속 → Download Docker Desktop for Mac.  
2. 칩셋(Apple / Intel)에 맞는 `.dmg` 다운로드.  
3. `.dmg` 열고 Docker 아이콘을 Applications 폴더로 드래그.  
4. Docker Desktop 실행 후  
   - 로그인: **Continue without signing in** 가능.  
   - 권한 관련 안내는 모두 허용.

<br/>


### 1-3. 설치 확인

Terminal에서:

```bash
docker --version
docker run hello-world
```

`Hello from Docker!` 나오면 설치 성공.

---

<br/>


## 2. macOS 프로젝트 폴더 만들기

> macOS에서는 별도의 WSL이 없으므로,  
> 일반 사용자 홈 디렉터리 아래에 프로젝트 폴더를 만든다.

```bash
cd ~

mkdir -p demo-p1/config demo-p1/workspace

cd demo-p1
```

이 폴더가 1부 실습 루트 디렉터리.

---

<br/>


## 3. .env 파일에 토큰/경로/포트 설정 (1부 방식)

```bash
cd ~/demo-p1
nano .env
```

내용:

```env
OPENCLAW_GATEWAY_TOKEN=your_secure_token_here

OPENCLAW_GATEWAY_MODE=local
OPENCLAW_GATEWAY_PORT=3000

OPENCLAW_CONFIG_DIR=./config
OPENCLAW_WORKSPACE_DIR=./workspace
```

- 토큰은 충분히 복잡한 문자열 권장.  
- `.env`는 Git 커밋 대상에서 제외.

저장 후 종료.

---

<br/>


## 4. docker-compose.yml 작성

```bash
cd ~/demo-p1
nano docker-compose.yml
```

내용:

```yaml
services:
  openclaw-gateway:
    image: ghcr.io/openclaw/openclaw:latest
    container_name: openclaw-gateway
    env_file:
      - .env
    environment:
      - NODE_ENV=production
      - OPENCLAW_GATEWAY_TOKEN=${OPENCLAW_GATEWAY_TOKEN}
      - OPENCLAW_GATEWAY_MODE=${OPENCLAW_GATEWAY_MODE}
    volumes:
      - ${OPENCLAW_CONFIG_DIR}:/home/node/.openclaw
      - ${OPENCLAW_WORKSPACE_DIR}:/home/node/workspace
    ports:
      - "${OPENCLAW_GATEWAY_PORT}:18789"
    restart: unless-stopped
    command: [
      "node", "dist/index.js",
      "gateway",
      "--allow-unconfigured",
      "--bind", "lan",
      "--port", "18789",
      "--token", "${OPENCLAW_GATEWAY_TOKEN}"
    ]
```

---

<br/>


## 5. 최소 설정 파일(openclaw.json) 추가

```bash
cd ~/demo-p1/config
nano openclaw.json
```

내용:

```json
{
  "gateway": {
    "mode": "local",
    "bind": "loopback",
    "controlUi": {
      "allowedOrigins": [
        "http://localhost:3000",
        "http://127.0.0.1:3000"
      ]
    }
  }
}
```

---

## 6. 컨테이너 실행 및 온보딩

### 6-1. 컨테이너 실행

```bash
cd ~/demo-p1

docker compose up -d

docker compose logs -f   # 선택
```

에러가 없다면 게이트웨이 기동 완료.

<br/>


### 6-2. 온보딩(onboard)으로 Gemini 연결 (키 직접 입력)

```bash
cd ~/demo-p1
docker compose exec openclaw-gateway node dist/index.js onboard
```

선택 흐름:

- Onboarding mode → QuickStart  
- Config handling → Update values  
- Provider → Google (Gemini API key + OAuth)  
- Auth method → Google Gemini API key  
- How do you want to provide this API key?  
  - **Paste API key now** 선택 →  
    Google AI Studio에서 발급받은 **실제 Gemini API 키**를 그대로 붙여넣기.  
- Default model → google/gemini-2.5-flash 권장.

`Onboarding complete` 메시지가 뜨면 완료.  
이때 컨테이너 내부 `auth-profiles.json`에 API 키가 저장되어 있고, 1부에서는 이 상태 그대로 사용한다.

---

## 7. 대시보드 접속 → 페어링 → 첫 인사

### 7-1. 대시보드 접속

브라우저에서:

```text
http://localhost:3000?token=YOUR_TOKEN_HERE
```

- `YOUR_TOKEN_HERE` = `.env`의 `OPENCLAW_GATEWAY_TOKEN`.

Pairing 관련 메시지가 보이면 통신/토큰은 정상.

### 7-2. 기기 페어링

Terminal에서:

```bash
cd ~/demo-p1

docker compose exec openclaw-gateway node dist/index.js devices list
```

`pending` 항목의 `RequestID`를 확인한 뒤:

```bash
docker compose exec openclaw-gateway node dist/index.js devices approve <RequestID>
```

컨테이너 재시작:

```bash
docker compose stop
docker compose start
```

브라우저에서 새로고침하면 채팅 UI 등장.

<br/>


### 7-3. 첫 인사

채팅창에:

> 안녕? 너는 누구니? 앞으로 나를 어떻게 도와줄 수 있어?

정상 응답이 돌아오면, macOS에서 1부에 필요한 전체 셋업이 끝난 상태다.

---

<br/>


## 8. 셋업 중 자주 나오는 에러 메시지 정리 (macOS)

### 8-1. `docker: command not found` / `docker: command not found` (Terminal)

**의미**  
- macOS 터미널에서 Docker CLI를 찾지 못하는 상태.  
- Docker Desktop이 설치/실행되지 않았거나, PATH 설정이 꼬였을 가능성.

**체크리스트**  
1. Docker Desktop 앱이 실제로 설치되어 있는지 확인.  
2. 상단 메뉴 막대에 고래 아이콘이 떠 있는지 확인 (실행 여부).  
3. Docker Desktop을 완전히 종료 후 다시 실행 → 터미널을 새로 열고 `docker --version` 재확인.  
4. 여전히 안 되면 Docker Desktop 재설치.

---

<br/>


### 8-2. 브라우저에서 `ERR_CONNECTION_REFUSED` / `이 사이트에 연결할 수 없음`

**의미**  
- `http://localhost:3000`으로 접속했지만 해당 포트에 서비스가 없거나, 컨테이너가 죽어 있는 상태.

**체크리스트**  
1. 터미널에서:

   ```bash
   cd ~/demo-p1
   docker compose ps
   ```

   - `openclaw-gateway`가 `Up`인지 확인.  
2. 로그 확인:

   ```bash
   docker compose logs -f
   ```

   - 포트 바인딩, 설정 경로 오류 등 메시지 확인.  
3. `.env`에서 `OPENCLAW_GATEWAY_PORT` 값을 3001 등 다른 포트로 바꾸고:

   ```bash
   docker compose down
   docker compose up -d
   ```

   후 `http://localhost:3001`로 재접속 시도.

---

<br/>


### 8-3. `permission denied` / 볼륨 마운트 권한 오류

**의미**  
- macOS의 특정 디렉터리는 Docker가 접근 권한을 자동으로 받지 못해, 마운트 시 `permission denied`가 발생할 수 있음.

**체크리스트**  
1. 프로젝트 폴더를 **사용자 홈 디렉터리** 아래(예: `~/demo-p1`)에 두었는지 확인.  
2. 외장 디스크나 시스템 보호 폴더(Desktop, Documents 등)를 직접 마운트하려 했다면,  
   Docker Desktop → Settings → Resources → File sharing(또는 비슷한 메뉴)에서 해당 경로 공유 허용.  
3. 권한 문제 해결 후 `docker compose down && docker compose up -d` 재실행.

---

<br/>


### 8-4. 대시보드에서 `Unauthorized` / 토큰 관련 에러

**의미**  
- 브라우저에서 전달한 `token` 값이 게이트웨이 토큰과 일치하지 않음.

**체크리스트**  
1. `.env`에서:

   ```env
   OPENCLAW_GATEWAY_TOKEN=...
   ```

   값을 다시 확인.  
2. 브라우저 주소창의 `token=` 값이 위 문자열과 정확히 같은지 확인.  
3. `.env`를 수정했다면:

   ```bash
   docker compose down
   docker compose up -d
   ```

   으로 컨테이너 재기동 후 다시 접속.

---

<br/>

### 8-5. `devices list` 시 pending 없음 / 페어링 안 됨

**의미**  
- 브라우저에서 Pairing Required 메시지가 보이는데, CLI에서 `pending` 요청이 안 잡히는 상황.

**체크리스트**  
1. 브라우저 URL의 `token=` 값과 `.env`의 `OPENCLAW_GATEWAY_TOKEN`이 일치하는지 확인.  
2. `devices list`를 실행하는 위치가 실제 프로젝트 폴더(`~/demo-p1`)인지 확인.  
3. `docker compose logs`에서 pairing 관련 경고/에러가 있는지 확인.  
4. 브라우저를 완전히 닫고, 새 탭에서 다시 `http://localhost:3000?token=...`로 접속 후 재시도.

---

<br/>


### 8-6. 온보딩 후 Gemini 관련 인증 에러 (`auth_permanent`, `invalid API key` 등)

**의미**  
- OpenClaw가 Gemini API 호출 시 인증 실패.  
- 잘못된 키, 비활성화된 키, 잘못 복사된 키(공백 포함) 가능성이 있다.

**체크리스트**  
1. Google AI Studio에서 **새 API 키**를 발급받는다.  
2. `docker compose exec openclaw-gateway node dist/index.js onboard`를 다시 실행해,  
   - `Paste API key now` 단계에서 새 키를 정확히 붙여넣는다.  
3. 온보딩이 끝난 뒤:

   ```bash
   docker compose down
   docker compose up -d
   ```
   으로 컨테이너를 재시작.  
4. 여전히 문제가 지속되면, 강의 Q&A에  
   - 에러 메시지 전문 또는 스크린샷  
   - 사용한 모델 이름  
   을 함께 첨부해 질문 남기기.