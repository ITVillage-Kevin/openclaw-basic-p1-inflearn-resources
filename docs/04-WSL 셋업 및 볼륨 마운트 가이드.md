# [가이드] Windows + WSL2 + Docker 환경 구성 및 볼륨 마운트

이 문서는 최적의 개발 환경을 구축하기 위한 **WSL2(Ubuntu) \+ Docker Desktop** 세팅 가이드입니다.

## 1. WSL2 + Ubuntu 설치

### WSL2 설치 (PowerShell)

1. **PowerShell(관리자 권한)** 을 실행합니다.  
2. 아래 명령어를 입력하고 완료되면 **PC를 재부팅** 합니다.  
   wsl \--install

3. 재부팅 후 나타나는 Ubuntu 터미널 창에서 사용할 **사용자 이름(username)** 과 **비밀번호**를 설정합니다.

## 2. Docker Desktop + WSL2 연동

### 2-1. Docker 설정 변경

1. **Docker Desktop**을 실행합니다.  
2. **Settings (톱니바퀴 아이콘)** \-\> **General**:  
   * Use the WSL 2 based engine 항목이 체크되어 있는지 확인합니다.  
3. **Settings** \-\> **Resources** \-\> **WSL Integration**:  
   * Enable integration with my default WSL distro를 켭니다.  
   * 아래 리스트에서 **Ubuntu** 스위치를 **ON**으로 활성화합니다.  
4. **Apply & Restart**를 클릭합니다.

### **2-2. 연동 확인**

Ubuntu 터미널(시작 메뉴 \-\> Ubuntu 실행)에서 다음 명령어가 정상 작동하는지 확인합니다.

docker \--version  
docker compose version

---

<br/>

## 3. WSL2(Linux) 내부에 프로젝트 폴더 생성

**중요:** 반드시 /mnt/c/나 /mnt/d/가 아닌, 리눅스 고유 경로인 **홈 디렉터리(\~)** 아래에 폴더를 만들어야 합니다.

\# 1\. 리눅스 홈 디렉터리로 이동  
cd \~

\# 2\. 데모용 디렉터리 생성  
mkdir \-p \~/demo-p1/config \~/openclaw-demo/workspace

\# 3\. 작업 디렉터리로 이동  
cd \~/demo-p1

---

<br/>


## 4. 설정 파일 작성 (.env & docker-compose.yml)

Ubuntu 터미널에서 nano나 vim을 사용하여 파일을 작성합니다.

참고: nano나 vim 사용이 익숙하지 않으신 분들은 Ubuntu 터미널에서 아래 명령어를 이용해서 VS Code 편집기를 연 후에 아래의 내용을 복사/붙여넣기 합니다.

```bash
cd \~/demo-p1  
code .
```

### 4-1. .env 파일

nano .env

(아래 내용을 복사해서 붙여넣고 Ctrl+O, Enter, Ctrl+X로 저장)

```text
# 1. 서버 접속 설정
OPENCLAW_GATEWAY_PORT=3000
OPENCLAW_GATEWAY_TOKEN=demo_token_123
OPENCLAW_GATEWAY_MODE=local

# 2. 경로 설정
OPENCLAW_CONFIG_DIR=./config
OPENCLAW_WORKSPACE_DIR=./workspace
```

### 4-2. docker-compose.yml 파일

nano docker-compose.yml

```yaml
services:  
  openclaw-gateway:  
    image: ghcr.io/openclaw/openclaw:latest  
    container_name: openclaw-gateway  
    environment:  
      - NODE_ENV=production  
      - OPENCLAW_GATEWAY_TOKEN=${OPENCLAW_GATEWAY_TOKEN}
      - OPENCLAW_GATEWAY_MODE=${OPENCLAW\_GATEWAY_MODE}
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

## 5. 컨테이너 실행 및 확인
```bash
cd ~/demo-p1

# 컨테이너 실행  
docker-compose up -d

# 컨테이너 상태 확인 (에러 여부 체크)  
docker-compose ps
```
- 컨테이너의 status에서 에러가 보이지 않는다면 컨테이너가 정상적으로 실행이 된겁니다.
- 또한 Docker Desktop UI에서 컨테이너에 `초록색` 아이콘이 표시되면 역시 컨테이너가 정상적으로 실행이 된 상태입니다.

---

<br/>


## 6 개발 도구 활용 (VS Code)

WSL2 환경을 마치 윈도우처럼 편하게 사용하는 방법입니다.
Ubuntu 터미널에서 nano 또는 vim 텍스트 편집기 사용에 익숙하지 않은 분들은 아래 방법으로 VS Code 편집기를 열어서 사용하세요.

### 6-1. VS Code에서 열기 (가장 추천)

Ubuntu 터미널에서 해당 폴더로 이동한 뒤 명령어를 입력합니다.

```bash
cd \~/demo-p1  
code .
```
- VS Code 편집기에서 우분투 터미널 내의 `demo-p1` 폴더의 파일과 하위 폴더들이 보이면 성공입니다.

---

<br/>


## 7. 핵심 요약 (이것만은 지켜주세요!)
1. 경로 엄수: 모든 작업은 /mnt/c/가 아닌 `~/`(리눅스 홈) 디렉터리에서 수행합니다.
2. 명령어 실행: Docker 관련 명령어는 반드시 **Ubuntu 터미널**에서 실행합니다.
3. 마운트 금지: D:\\...와 같은 윈도우 드라이브 경로를 docker-compose.yml의 볼륨으로 직접 연결하지 마세요. (권한 에러의 주원인입니다.)

실습 중 권한 에러가 계속된다면, docker compose down 후 ~/demo-p1 내부의 config, workspace 폴더 삭제 후 다시 시도해 보시기 바랍니다.