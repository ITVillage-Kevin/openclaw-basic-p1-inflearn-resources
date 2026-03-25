# OpenClaw 트러블슈팅 가이드 – 환경변수 & WSL 연동

## 1. WSL2에서 `docker` 명령어를 찾지 못하는 오류

### 증상

WSL2 Ubuntu에서 다음과 같은 오류가 발생한다.

> The command 'docker' could not be found in this WSL 2 distro.  
> We recommend to activate the WSL integration in Docker Desktop settings.

강의 실습 때는 잘 됐는데, 재부팅 후 `docker compose up -d`를 실행했을 때 위 에러가 뜨는 경우가 많다.

### 원인

- Docker Desktop은 실행 중이지만, **WSL2 Ubuntu와의 통합(WSL integration)이 잠시 끊어진 상태**  
- Windows / Docker Desktop / WSL 업데이트나 재부팅 이후에  
  - 어떤 WSL 배포판(Ubuntu 등)과 연동할지 설정이 꼬였거나  
  - 해당 배포판에 `docker` 클라이언트가 다시 주입되지 않은 상태

### 해결 방법

1. Docker Desktop이 실행 중인지 확인  
   - 시스템 트레이의 고래 아이콘이 `Running` 상태인지 확인한다.

2. WSL2에서 `docker` 인식 여부 확인  
   - WSL2 Ubuntu 터미널에서 다음을 실행한다.
     ```bash
     docker --version
     ```
   - 위 에러가 그대로 나온다면 WSL 통합 문제로 본다.

3. Docker Desktop에서 WSL Integration 재설정  
   - Docker Desktop → `Settings` → `Resources` → `WSL integration` 이동  
   - `Use the WSL 2 based engine` 옵션이 켜져 있는지 확인  
   - 사용 중인 Ubuntu 배포판 옆 토글을 껐다가 다시 켠다.  
   - 필요하면 `Refetch distros` 버튼을 눌러 WSL 배포판 목록을 새로고침한 뒤, 해당 Ubuntu에 체크하고 `Apply & Restart` 한다.

4. WSL 자체를 한 번 종료 후 다시 시작  
   - Windows PowerShell(또는 CMD)에서:
     ```bash
     wsl --shutdown
     ```
   - 이후 Ubuntu 터미널을 다시 열고, 다시 한 번:
     ```bash
     docker --version
     docker compose version
     ```
     으로 정상 출력을 확인한다.

<br/>

### 재발 시 체크리스트

같은 문제가 다시 발생할 때마다 아래 순서를 따라 점검한다.

- Docker Desktop이 먼저 실행되어 있는가?  
- WSL2 Ubuntu에서 `docker --version`이 정상적으로 동작하는가?  
- Docker Desktop → `Settings` → `Resources` → `WSL integration`에서  
  - WSL2 엔진 사용이 켜져 있는가?  
  - 내가 사용하는 Ubuntu 배포판 옆 토글이 켜져 있는가?  
  - 필요 시 `Refetch distros` 후 다시 체크했는가?  
- 그래도 안 되면 `wsl --shutdown` 후 Docker Desktop을 재시작하고 다시 시도한다.


---

## 2. `auth-profiles.json`에서 `google_api_key` 하드코딩 제거 시 “API 키 없음” 오류

### 증상

- WSL2 Ubuntu에서 강의대로 API 키를 환경변수로 export 했다.
- `echo` 명령으로 값이 잘 보인다.  
- `auth-profiles.json`에서 `key: ""`로 비워 두면 OpenClaw가 “API 키가 없다”고 에러를 낸다.
- 다시 `key`에 직접 API 키를 넣으면 정상 동작한다.

<br/>

### 핵심 개념

- **Windows 환경변수는 사용하지 않는다.**  
  이 강의에서는 *오직* **WSL2 Ubuntu 내부에서 export 한 값**만 사용한다.
- WSL에서 `echo`로 보이는 값은 “WSL 셸 안에 있는 값”일 뿐이고,  
  OpenClaw가 실제로 보는 값은 **Docker 컨테이너 내부의 환경변수**이다.
- `auth-profiles.json`의 `"key": ""`는  
  “이 파일에는 키를 하드코딩하지 않고, 환경변수에서 가져오겠다”는 의미로 이해하면 된다.

### 2-1. WSL2에서 API 키 설정

WSL2 Ubuntu 터미널에서: vi 또는 nano 편집기로 ~/.bashrc를 연 상태에서:

```bash
export GEMINI_API_KEY="your-real-key"

echo "$GEMINI_API_KEY"

```

여기까지는 **WSL 세션 안에 값이 잘 들어갔는지** 확인하는 단계이다.
<br/>

### 2-2. docker-compose.yml에서 컨테이너로 env 전달

`docker-compose.yml`의 `environment` 섹션에 API 키를 넘기는 설정이 반드시 있어야 한다.

```yaml
services:
  openclaw-gateway:
    environment:
      - GEMINI_API_KEY=${GEMINI_API_KEY}
```

이 설정이 없으면, WSL2에서 `export`를 해도 **컨테이너 내부에서는 해당 환경변수를 전혀 볼 수 없다.**  
이 상태에서 `auth-profiles.json`의 `"key": ""`로 비워두면, OpenClaw는 “파일에도 없고 env에도 없다”라고 판단하여 “API 키 없음” 오류를 낸다.

<br/>

### 2-3. 컨테이너 내부에서 실제로 env가 보이는지 확인

컨테이너 안에 들어가서 직접 `echo`로 확인하는 것이 가장 확실하다.

```bash
# WSL2 Ubuntu에서, 컨테이너 셸 접속
docker compose exec openclaw-gateway /bin/bash

# 컨테이너 내부에서 API 키 확인
echo "$GEMINI_API_KEY"
```

- 키 값이 출력되면 → 환경변수가 컨테이너 내부까지 정상적으로 전달된 상태.  
- 아무 것도 출력되지 않으면 → 컨테이너 안에는 아직 키가 없는 상태다.

이 경우 다음을 다시 확인해야 한다.

- WSL2에서 올바른 셸(현재 세션)에서 `export` 했는가?  
- `docker-compose.yml`의 `environment`에 해당 변수가 포함되어 있는가?  
- 환경변수를 바꾼 뒤 `docker compose down && docker compose up -d`로 컨테이너를 재시작했는가?

<br/>

### 2-4. auth-profiles.json과의 관계

- `auth-profiles.json`에서 `"key": ""`로 비워두면, OpenClaw는 **환경변수에서 키를 찾으려고 시도**한다.  
- 컨테이너 내부에 `GEMINI_API_KEY`가 제대로 설정되어 있다면 이 상태에서도 정상 동작해야 한다.  
- 반대로 컨테이너 `echo`가 비어 있으면, `key` 하드코딩 때만 동작하고 `key: ""`에서는 항상 “API 키 없음” 오류가 나는 것이 정상적인 동작이다.

<br/>

### 재발 시 체크리스트

아래를 순서대로 점검하면 대부분의 API 키 관련 문제를 잡을 수 있다.

- WSL2 Ubuntu 내부에서 `export GEMINI_API_KEY=...`를 했는가?
- `source` 명령어를 이용해서 WSL2 Ubuntu 환경변수에 영구 저장되도록 했는가?
- `docker-compose.yml`의 `environment:` 섹션에 해당 변수를 컨테이너로 넘기는 설정이 있는가?  
- 환경변수를 변경한 뒤 `docker compose down` → `docker compose up -d`로 컨테이너를 다시 띄웠는가?  
- 환경변수를 변경한 뒤 WSL2 Ubuntu 터미널을 껏다가 다시 켰는가?
- `docker compose exec openclaw-gateway /bin/bash`로 컨테이너에 들어가 `echo $GEMINI_API_KEY`를 했을 때 실제 키 값이 출력되는가?  
- 위 단계가 모두 OK인 상태에서 `auth-profiles.json`의 `"key": ""`를 사용하면, API 키가 없는 오류가 나지 않아야 한다.