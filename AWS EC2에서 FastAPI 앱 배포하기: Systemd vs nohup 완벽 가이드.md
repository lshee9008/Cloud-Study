# AWS EC2에서 FastAPI 앱 배포하기: Systemd vs nohup 완벽 가이드

## 들어가며

로컬에서는 잘 돌아가던 FastAPI 앱을 AWS EC2에 배포했는데 SSH 연결을 끊으면 서버가 죽는 경험, 다들 한 번쯤 있으시죠? 이번 포스팅에서는 **프로덕션 환경에서 안정적으로 서비스를 운영하는 방법**을 다룹니다.

---

## 목차
1. [백그라운드 실행이 필요한 이유](#1-백그라운드-실행이-필요한-이유)
2. [nohup: 간단한 백그라운드 실행](#2-nohup-간단한-백그라운드-실행)
3. [Systemd: 프로덕션급 서비스 관리](#3-systemd-프로덕션급-서비스-관리)
4. [실전 적용 가이드](#4-실전-적용-가이드)
5. [트러블슈팅](#5-트러블슈팅)

---

## 1. 백그라운드 실행이 필요한 이유

### 문제 상황

```bash
# SSH로 접속해서 앱 실행
ubuntu@ec2:~$ python3 main.py
INFO: Uvicorn running on http://0.0.0.0:8000

# SSH 연결 끊으면?
# → 앱도 함께 종료됨 💀
```

**왜 이런 일이 발생할까?**

일반적으로 터미널에서 실행한 프로세스는 해당 터미널 세션에 종속됩니다. SSH 연결이 끊어지면 터미널이 닫히고, 함께 실행 중이던 프로세스도 SIGHUP 신호를 받아 종료됩니다.

### 해결 방법 비교

| 방법 | 난이도 | 안정성 | 프로덕션 적합성 |
|------|--------|--------|----------------|
| **nohup** | ⭐ 쉬움 | ⭐⭐ 보통 | ❌ 비추천 |
| **screen/tmux** | ⭐⭐ 보통 | ⭐⭐ 보통 | ❌ 비추천 |
| **Systemd** | ⭐⭐⭐ 어려움 | ⭐⭐⭐⭐⭐ 최고 | ✅ 강력 추천 |
| **Docker** | ⭐⭐⭐⭐ 복잡 | ⭐⭐⭐⭐⭐ 최고 | ✅ 추천 |
| **PM2** | ⭐⭐ 보통 | ⭐⭐⭐⭐ 좋음 | ✅ 추천 (Node.js) |

---

## 2. nohup: 간단한 백그라운드 실행

### nohup이란?

**no hang up**의 약자로, SIGHUP 신호를 무시하고 프로세스를 백그라운드에서 실행하는 명령어입니다.

### 기본 사용법

```bash
# 가상환경 활성화
source ~/venv/bin/activate

# nohup으로 백그라운드 실행
nohup python3 main.py > app.log 2>&1 &
```

**명령어 해석:**
- `nohup`: SIGHUP 신호 무시
- `python3 main.py`: 실행할 명령어
- `> app.log`: 표준 출력을 app.log로 리다이렉트
- `2>&1`: 표준 에러도 표준 출력으로 합침
- `&`: 백그라운드 실행

### uvicorn 직접 실행

```bash
# 기본 실행
nohup uvicorn main:app --host 0.0.0.0 --port 8000 > app.log 2>&1 &

# 워커 프로세스 여러 개로 실행 (성능 향상)
nohup uvicorn main:app --host 0.0.0.0 --port 8000 --workers 2 > app.log 2>&1 &

# 로그 분리 버전
nohup uvicorn main:app --host 0.0.0.0 --port 8000 \
  > app.log 2> error.log &
```

### 프로세스 관리

```bash
# 실행 중인 프로세스 확인
ps aux | grep uvicorn
ps aux | grep python3

# 포트 사용 확인
sudo netstat -tulpn | grep 8000
# 또는
sudo ss -tulpn | grep 8000

# 로그 실시간 확인
tail -f app.log

# 프로세스 종료 (PID가 12345라고 가정)
kill 12345

# 강제 종료
kill -9 12345

# 이름으로 종료
pkill -f uvicorn
```

### 편의를 위한 스크립트 작성

```bash
# start.sh 생성
cat > start.sh << 'EOF'
#!/bin/bash
cd ~
source venv/bin/activate
nohup python3 main.py > app.log 2>&1 &
echo "✅ 서버 시작됨 (PID: $!)"
sleep 2
tail -20 app.log
EOF

chmod +x start.sh
```

```bash
# stop.sh 생성
cat > stop.sh << 'EOF'
#!/bin/bash
PID=$(pgrep -f "python3 main.py")
if [ -z "$PID" ]; then
    echo "❌ 실행 중인 서버가 없습니다"
else
    kill $PID
    echo "🛑 서버 중지됨 (PID: $PID)"
fi
EOF

chmod +x stop.sh
```

### 📊 nohup 사용이 적합한 경우

✅ **이럴 때 사용하세요:**
- 빠른 테스트/데모 목적
- 개발 환경에서 임시 실행
- 간단한 배치 작업
- 1-2일 정도만 돌릴 임시 서버

❌ **이럴 때는 피하세요:**
- 프로덕션 환경
- 24/7 운영이 필요한 서비스
- 자동 재시작이 필요한 경우
- 여러 서비스를 함께 관리해야 할 때

### ⚠️ nohup의 단점

1. **자동 재시작 없음**
   - 앱이 크래시되면 수동으로 다시 실행해야 함
   - 메모리 누수로 죽어도 자동 복구 안 됨

2. **서버 재부팅 시 자동 실행 불가**
   - EC2 재부팅 후 수동으로 실행해야 함
   - crontab @reboot 설정 필요 (추가 작업)

3. **로그 관리 어려움**
   - 로그 파일이 무한정 커질 수 있음
   - 로그 로테이션 직접 구현 필요

4. **프로세스 관리 불편**
   - PID 추적 번거로움
   - 여러 서비스 동시 관리 어려움

5. **의존성 관리 없음**
   - Ollama나 DB가 먼저 실행되어야 하는 경우 순서 보장 안 됨

### 💡 주의사항

```bash
# ❌ 잘못된 사용
nohup python3 main.py &  # 로그가 nohup.out에 쌓여서 나중에 찾기 어려움

# ✅ 올바른 사용
nohup python3 main.py > app.log 2>&1 &  # 명시적으로 로그 파일 지정

# ❌ 권한 문제 발생 가능
nohup python3 main.py > /var/log/app.log 2>&1 &  # /var/log는 root 권한 필요

# ✅ 홈 디렉토리 사용
nohup python3 main.py > ~/app.log 2>&1 &
```

---

## 3. Systemd: 프로덕션급 서비스 관리

### Systemd란?

Linux 시스템의 **init 시스템**(부팅 시 첫 번째로 실행되는 프로세스)이자 **서비스 관리자**입니다. Ubuntu 16.04 이후 기본 채택되었으며, 현대적인 Linux 배포판 대부분이 사용합니다.

### Systemd의 핵심 기능

1. **서비스 생명주기 관리**: 시작, 중지, 재시작, 상태 확인
2. **의존성 관리**: 다른 서비스 실행 후 시작 설정 가능
3. **자동 재시작**: 크래시 시 자동 복구
4. **로그 관리**: journald를 통한 중앙집중식 로깅
5. **리소스 제어**: CPU, 메모리 사용량 제한 가능

### 서비스 파일 생성

```bash
sudo nano /etc/systemd/system/newsapp.service
```

```ini
[Unit]
Description=News AI FastAPI Application
After=network.target ollama.service

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/home/ubuntu
Environment="PATH=/home/ubuntu/venv/bin"
ExecStart=/home/ubuntu/venv/bin/python3 /home/ubuntu/main.py
Restart=always
RestartSec=10
StandardOutput=append:/home/ubuntu/app.log
StandardError=append:/home/ubuntu/error.log

[Install]
WantedBy=multi-user.target
```

### 설정 파일 상세 분석

#### **[Unit] 섹션 - 서비스 메타데이터**

| 옵션 | 설명 | 예시 |
|------|------|------|
| `Description` | 서비스에 대한 설명 | News AI FastAPI Application |
| `After` | 이 서비스보다 먼저 시작될 서비스 지정 | network.target, ollama.service |
| `Wants` | 선택적 의존성 (실패해도 실행) | postgresql.service |
| `Requires` | 필수 의존성 (실패 시 같이 실패) | mysql.service |
| `Before` | 이 서비스 후에 시작될 서비스 지정 | nginx.service |

**실전 예시:**

```ini
# 네트워크 준비 완료 + Ollama 실행 후 시작
After=network.target ollama.service

# PostgreSQL이 필수인 경우
Requires=postgresql.service
After=postgresql.service
```

#### **[Service] 섹션 - 실행 방식**

| 옵션 | 설명 | 가능한 값 |
|------|------|-----------|
| `Type` | 프로세스 타입 | simple, forking, oneshot, notify |
| `User` | 실행할 사용자 | ubuntu, www-data, root |
| `Group` | 실행할 그룹 | ubuntu, www-data |
| `WorkingDirectory` | 작업 디렉토리 (절대경로) | /home/ubuntu |
| `Environment` | 환경 변수 설정 | PATH, PYTHONPATH, API_KEY |
| `EnvironmentFile` | 환경 변수 파일 로드 | /etc/myapp/env |
| `ExecStart` | 시작 명령어 (절대경로 필수) | /usr/bin/python3 main.py |
| `ExecStartPre` | 시작 전 실행할 명령어 | /usr/bin/check-db.sh |
| `ExecStartPost` | 시작 후 실행할 명령어 | /usr/bin/notify-slack.sh |
| `ExecStop` | 중지 명령어 | /usr/bin/graceful-shutdown.sh |
| `ExecReload` | 재로드 명령어 | /usr/bin/kill -HUP $MAINPID |
| `Restart` | 재시작 정책 | no, always, on-failure |
| `RestartSec` | 재시작 대기 시간 (초) | 10 |
| `StandardOutput` | 표준 출력 위치 | journal, file, append:file |
| `StandardError` | 표준 에러 위치 | journal, file, append:file |
| `TimeoutStartSec` | 시작 타임아웃 | 90 |
| `TimeoutStopSec` | 중지 타임아웃 | 30 |

**Type 상세 설명:**

```ini
# simple: 기본값, ExecStart가 메인 프로세스 (FastAPI, Flask 등)
Type=simple

# forking: 백그라운드로 fork하는 데몬 (Nginx, Apache 등)
Type=forking
PIDFile=/var/run/myapp.pid

# oneshot: 한 번 실행 후 종료 (초기화 스크립트)
Type=oneshot
RemainAfterExit=yes

# notify: sd_notify()로 준비 완료 알림 (고급)
Type=notify
```

**Restart 정책 비교:**

```ini
# 절대 재시작 안 함
Restart=no

# 항상 재시작 (정상 종료 포함) - 가장 많이 사용
Restart=always

# 에러로 종료된 경우만 재시작
Restart=on-failure

# 비정상 종료 시만 재시작 (timeout, watchdog, signal)
Restart=on-abnormal

# 특정 종료 코드에서만 재시작
Restart=on-failure
RestartPreventExitStatus=0 2
```

**환경 변수 설정 방법:**

```ini
# 방법 1: 직접 설정
Environment="PATH=/home/ubuntu/venv/bin"
Environment="API_KEY=secret123"
Environment="DB_HOST=localhost"

# 방법 2: 파일에서 로드 (.env 파일 사용)
EnvironmentFile=/home/ubuntu/.env

# /home/ubuntu/.env 파일 내용:
# API_KEY=secret123
# DB_HOST=localhost
# DEBUG=false
```

#### **[Install] 섹션 - 부팅 설정**

| 옵션 | 설명 |
|------|------|
| `WantedBy=multi-user.target` | 일반적인 서버 부팅 시 자동 실행 |
| `WantedBy=graphical.target` | GUI 환경에서 자동 실행 |
| `WantedBy=network.target` | 네트워크 준비 후 자동 실행 |

### 고급 설정 예시

#### 1. 리소스 제한 설정

```ini
[Service]
# 메모리 제한 (2GB)
MemoryLimit=2G
MemoryMax=2G

# CPU 사용률 제한 (50%)
CPUQuota=50%

# 파일 디스크립터 제한
LimitNOFILE=65536

# 프로세스 수 제한
TasksMax=100
```

#### 2. 보안 강화 설정

```ini
[Service]
# 프라이빗 /tmp 디렉토리 사용
PrivateTmp=yes

# 읽기 전용 루트 파일시스템
ReadOnlyPaths=/

# 쓰기 가능한 경로만 지정
ReadWritePaths=/home/ubuntu/newsapp

# 네트워크 네임스페이스 격리
PrivateNetwork=no

# 특정 시스템 콜 차단
SystemCallFilter=~@clock @debug @module @mount @obsolete @privileged @raw-io @reboot @swap
```

#### 3. 헬스체크 설정

```ini
[Service]
# 타임아웃 설정
TimeoutStartSec=90
TimeoutStopSec=30

# Watchdog (앱이 일정 시간 응답 없으면 재시작)
WatchdogSec=30

# 시작 전 DB 연결 확인
ExecStartPre=/usr/bin/check-database.sh

# 시작 후 Slack 알림
ExecStartPost=/usr/bin/curl -X POST https://hooks.slack.com/...
```

#### 4. 다중 서비스 의존성 예시

```ini
[Unit]
Description=News AI FastAPI Application
After=network.target
Requires=postgresql.service redis.service
After=postgresql.service redis.service ollama.service
Wants=ollama.service

[Service]
# PostgreSQL, Redis 필수, Ollama는 선택
# 순서: network → postgresql, redis → ollama → newsapp
```

### Systemd 명령어 전체

```bash
# 설정 파일 리로드 (수정 후 반드시 실행!)
sudo systemctl daemon-reload

# 서비스 시작
sudo systemctl start newsapp

# 서비스 중지
sudo systemctl stop newsapp

# 서비스 재시작 (stop → start)
sudo systemctl restart newsapp

# 설정 리로드 (중지 없이)
sudo systemctl reload newsapp

# 상태 확인
sudo systemctl status newsapp

# 부팅 시 자동 실행 활성화
sudo systemctl enable newsapp

# 자동 실행 비활성화
sudo systemctl disable newsapp

# 서비스 활성화 + 즉시 시작
sudo systemctl enable --now newsapp

# 의존 관계 확인
systemctl list-dependencies newsapp

# 전체 서비스 목록
systemctl list-units --type=service

# 실패한 서비스 확인
systemctl --failed
```

### 로그 확인 명령어

```bash
# 실시간 로그 (tail -f와 유사)
sudo journalctl -u newsapp -f

# 마지막 50줄
sudo journalctl -u newsapp -n 50

# 오늘 로그만
sudo journalctl -u newsapp --since today

# 특정 시간 범위
sudo journalctl -u newsapp --since "2026-01-09 10:00:00" --until "2026-01-09 12:00:00"

# 우선순위 필터 (에러만)
sudo journalctl -u newsapp -p err

# 부팅별 로그
sudo journalctl -u newsapp -b 0  # 현재 부팅
sudo journalctl -u newsapp -b -1  # 이전 부팅

# 페이지 없이 출력
sudo journalctl -u newsapp --no-pager

# JSON 형식으로 출력
sudo journalctl -u newsapp -o json-pretty
```

### 📊 Systemd 사용이 적합한 경우

✅ **반드시 사용하세요:**
- 프로덕션 환경
- 24/7 운영 서비스
- 서버 재부팅 후 자동 실행 필요
- 크래시 시 자동 재시작 필요
- 여러 서비스의 의존 관계 관리
- 중앙집중식 로그 관리 필요
- 리소스 사용량 제어 필요

✅ **특히 이런 경우:**
- 데이터베이스 의존성이 있는 앱
- 다른 서비스와 연동이 많은 경우
- 기업/상용 서비스
- 모니터링/알림 시스템 연동
- 보안이 중요한 환경

### ⚠️ Systemd의 단점

1. **학습 곡선**
   - 설정 파일 문법 익히는데 시간 필요
   - 옵션이 많아서 초보자에게 복잡함

2. **디버깅 난이도**
   - 에러 메시지가 불친절할 때 있음
   - 권한, 경로 문제 파악이 어려울 수 있음

3. **플랫폼 종속성**
   - Linux 전용 (macOS, Windows 불가)
   - systemd가 없는 구형 시스템은 사용 불가

4. **과도한 기능**
   - 간단한 스크립트 실행에는 오버스펙
   - 빠른 테스트에는 nohup이 더 빠름

### 💡 주의사항

#### 1. 절대경로 필수

```ini
# ❌ 상대경로 사용 - 에러 발생!
ExecStart=python3 main.py
ExecStart=venv/bin/python3 main.py

# ✅ 절대경로 사용
ExecStart=/home/ubuntu/venv/bin/python3 /home/ubuntu/main.py
```

#### 2. 환경변수 문제

```ini
# ❌ 가상환경 활성화 명령어 사용 불가
ExecStart=source venv/bin/activate && python3 main.py

# ✅ PATH에 가상환경 추가
Environment="PATH=/home/ubuntu/venv/bin:/usr/local/bin:/usr/bin"
ExecStart=/home/ubuntu/venv/bin/python3 /home/ubuntu/main.py
```

#### 3. 로그 파일 권한

```bash
# 로그 파일 미리 생성 및 권한 설정
touch ~/app.log ~/error.log
chmod 644 ~/app.log ~/error.log
chown ubuntu:ubuntu ~/app.log ~/error.log
```

#### 4. daemon-reload 필수

```bash
# 서비스 파일 수정 후 반드시 실행!
sudo systemctl daemon-reload
sudo systemctl restart newsapp
```

#### 5. 포트 충돌 확인

```bash
# 기존 프로세스 종료 후 systemd 시작
sudo systemctl stop newsapp
pkill -f python3
sudo systemctl start newsapp
```

---

## 4. 실전 적용 가이드

### Step 1: 환경 준비

```bash
# Python 및 필수 도구 설치
sudo apt update
sudo apt install -y python3 python3-pip python3-venv

# 프로젝트 디렉토리 생성
mkdir ~/newsapp
cd ~/newsapp

# 가상환경 생성
python3 -m venv venv
source venv/bin/activate

# 패키지 설치
pip install fastapi uvicorn ollama sqlalchemy beautifulsoup4 aiohttp apscheduler
```

### Step 2: Ollama 설치

```bash
# Ollama 설치
curl -fsSL https://ollama.com/install.sh | sh

# 모델 다운로드
ollama pull gemma3:270m

# 확인
ollama list
```

### Step 3: 보안 그룹 설정

AWS 콘솔에서:

1. **인바운드 규칙**
   - 유형: 사용자 지정 TCP
   - 포트: 8000
   - 소스: 0.0.0.0/0 (또는 특정 IP)

2. **아웃바운드 규칙**
   - 유형: 모든 트래픽
   - 대상: 0.0.0.0/0
   - (크롤링을 위해 필수!)

### Step 4: Systemd 서비스 설정

```bash
# 서비스 파일 생성
sudo nano /etc/systemd/system/newsapp.service
```

```ini
[Unit]
Description=News AI FastAPI Application
After=network.target ollama.service
Wants=ollama.service

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/home/ubuntu
Environment="PATH=/home/ubuntu/venv/bin"
ExecStart=/home/ubuntu/venv/bin/python3 /home/ubuntu/main.py
Restart=always
RestartSec=10
StandardOutput=append:/home/ubuntu/app.log
StandardError=append:/home/ubuntu/error.log

# 리소스 제한 (선택사항)
MemoryMax=2G
CPUQuota=80%

# 타임아웃 설정
TimeoutStartSec=120
TimeoutStopSec=30

[Install]
WantedBy=multi-user.target
```

### Step 5: 서비스 시작

```bash
# 로그 파일 생성
touch ~/app.log ~/error.log

# daemon 리로드
sudo systemctl daemon-reload

# 서비스 활성화 및 시작
sudo systemctl enable --now newsapp

# 상태 확인
sudo systemctl status newsapp

# 로그 확인
tail -f ~/app.log
```

### Step 6: 테스트

```bash
# 로컬 테스트
curl http://localhost:8000/news

# 외부 테스트 (브라우저 또는 로컬 PC에서)
curl http://YOUR-EC2-PUBLIC-IP:8000/news

# 자동 재시작 테스트
sudo systemctl restart newsapp
sleep 5
curl http://localhost:8000/news

# 재부팅 테스트
sudo reboot
# 재접속 후
sudo systemctl status newsapp
```

---

## 5. 트러블슈팅

### 문제 1: 서비스가 시작되지 않음

**증상:**
```bash
$ sudo systemctl status newsapp
● newsapp.service - News AI FastAPI Application
     Active: failed (Result: exit-code)
```

**해결 방법:**

```bash
# 1. 자세한 로그 확인
sudo journalctl -u newsapp -n 100 --no-pager

# 2. 설정 파일 문법 검증
sudo systemd-analyze verify newsapp.service

# 3. 직접 실행해서 에러 확인
cd ~
source venv/bin/activate
/home/ubuntu/venv/bin/python3 /home/ubuntu/main.py

# 4. 권한 확인
ls -la ~/main.py
ls -la ~/venv/bin/python3
```

**흔한 원인:**
- 절대경로 미사용
- 가상환경 PATH 설정 누락
- 파일 권한 문제
- 포트 이미 사용 중

### 문제 2: 크롤링이 안 됨

**증상:**
```python
# 로그에 이런 에러
TimeoutError: [Errno 110] Connection timed out
```

**해결 방법:**

```bash
# 1. 아웃바운드 연결 테스트
curl https://news.nate.com

# 2. DNS 확인
nslookup news.nate.com

# 3. 보안 그룹 아웃바운드 규칙 확인
# AWS 콘솔 → EC2 → 보안 그룹 → 아웃바운드 규칙
# 0.0.0.0/0 으로 모든 트래픽 허용되어 있는지 확인
```

### 문제 3: Ollama 연결 실패

**증상:**
```python
ollama.exceptions.ConnectionError: Ollama is not running
```

**해결 방법:**

```bash
# 1. Ollama 서비스 확인
sudo systemctl status ollama

# 2. 재시작
sudo systemctl restart ollama

# 3. 모델 확인
ollama list

# 4. 서비스 의존성 추가 (newsapp.service에)
[Unit]
After=ollama.service
Requires=ollama.service
```

### 문제 4: 메모리 부족

**증상:**
```bash
# 시스템이 느려지거나 OOM Killer 발동
dmesg | grep -i "out of memory"
```

**해결 방법:**

```bash
# 1. Swap 메모리 추가
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swap
