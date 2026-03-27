# Vercel이란?

- 프론트엔드 정적 파일을 전 세계 CDN에 배포해주는 호스팅 플랫폼
    - CDN: 웹사이트 파일(이미지, 코드 등)을 전 세계 여러 곳에 미리 복사해 두었다가, 사용자에게 가장 가까운 서버에서 빠르게 보내주는 “전달망”이에요.[1](https://www.notion.so/Flutter-web-Vercel-33061db7c45b800f8a5cdc16903f1eef?pvs=21)
- Vercel은 Next.js를 만든 팀이 운영하고 있으며, Next.js 외에도 React, Vue, Svelte, 그리고 Flutter Web 같은 정적 결과물이라면 무엇이든 올릴 수 있다.
- 가장 큰 장점은 배포가 단순하다는 것이다. CLI 한 줄이면 수십 초 안에 전 세계 어디서나 접속 가능한 URL이 생김

# 핵심 개념

### Deployment vs Domain

- Vercel은 배포할 때마다 고유한 URL을 자동 생성한다.
    
    ```bash
    Deployment URL  →  my-app-abc123xyz-username.vercel.app
    Domain          →  my-app.vercel.app  또는  mycustomdomain.com
    ```
    
    - Deployment URL은 해당 배포 버전의 고유 주소로 영구적으로 유지된다.
    - Domain은 이 Deployment URL중 하나를 가리키는 포인터 역할을 한다.
    - —prod 플래그로 배포하면 Domain이 자동으로 최신 Deployment를 가리킨다.
    - 플래그 없이 배포하면 preview 배포가 되어 Domain은 갱신되지 않는다.

### Production vs Preview

|  | Production | Preview |
| --- | --- | --- |
| 명령어 | vercel —prod | vercel |
| Domain 갱신 | 됨 | 안 됨 (임시 URL만 생성) |
| 용도 | 실제 서비스 배포 | 기능 테스트, PR 확인 |
- GitHub 연동 시 main 브랜치에 push하면 Production 배포, PR을 열면 Preview 배포가 자동으로 실행

# CLI

- 설치 및 인증
    
    ```bash
    npm i -g vercel # 전역 설치
    npx vercel login # 로그인
    npx vercel whoami # 현재 로그인된 계정 확인
    ```
    
- 배포
    
    ```bash
    npx vercel . # 현재 폴더 Preview 배포
    npx vercel . --prod # Production 배포
    npx vercel . --prod --yes # 대화 없이 자동 배포
    ```
    
    - 처음 배포하면 프로젝트 연결 여부를 묻는다.
    - 기존 프로젝트에 연결할지, 새로 만들지 선택할 수 있다.
- 배포 목록
    
    ```bash
    npx vercel ls # 현재 프로젝트 배포 목록
    npx vercel ls --scope [팀/계정ID] # 특정 계정의 배포 목록
    npx vercel inspect [deployment-url] # 특정 배포 상세 정보
    ```
    
- Domain 연결 (alias)
    
    ```bash
    npx vercel alias set [deployment-url] [domain]
    ```
    
    - —prod 배포를 했는데도 Domain이 갱신 안 될 때 수동으로 연결한다.
    - Flutter Web 프로젝트를 배포하면서 이 상황을 꽤 자주 겪었다.
        
        ```bash
        # 실제로 썼던 명령어
        npx vercel alias set promptmaster-abc123.vercel.app promjang.vercel.app
        ```
        
- 배포 승격
    
    ```bash
    npx vercel promote [deployment-url]
    ```
    
    - 이미 배포된 버전 중 특정 시점으로 롤백하거나 승격할 때 사용
    - 현재 Production인 배포에 이 명령어를 실행하면 409 에러가 나며 이미 Production이라고 알려줌
- 프로젝트 관리
    
    ```bash
    npx vercel project ls # 프로젝트 목록
    npx vercel project rm [이름] # 프로젝트 삭제
    ```
    
    - Settings가 꼬였을 때 프로젝트를 삭제하고 새로 만드는게 더 빠를 때가 있다.

# vercel.json

- 프로젝트 루트에 두는 설정 파일다. 단, CLI로 npx vercel . 을 실행할 때는 해당 폴더 안에 있어야 적용된다.
- rewrites
    - SPA(Single Page Application)에서 필수 설정이다.
    - React, Vue, Flutter Web 처럼 클라이언트 사이드 라우팅을 쓰는 앱은 /about, /dashboard 같은 경로로 접속하면 Vercel이 해당 파일을 찾으려다 404를 반환한다.
    - 모든 요청을 index.html 로.보내면 앱이 라우팅을 직접 처리하게 되낟.
        
        ```json
        {
          "rewrites": [
            { "source": "/(.*)", "destination": "/index.html" }
          ]
        }
        ```
        
- headers
    - HTTP 응답 헤더를 경로별로 설정
    - 캐시 정책, 보안 헤더(CSP, CORS 등)를 여기서 제어한다.
        
        ```json
        {
          "headers": [
            {
              "source": "/index.html",
              "headers": [
                { "key": "Cache-Control", "value": "no-cache, no-store, must-revalidate" }
              ]
            },
            {
              "source": "/(.*\\.js)",
              "headers": [
                { "key": "Cache-Control", "value": "public, max-age=31536000, immutable" }
              ]
            }
          ]
        }
        ```
        
    - Flutter Web은 flutter_service_worker.js 가 브라우저에 앱을 캐시해둔다.
    - 배포해도 브라우저가 구버전을 계속 보여주는 문제가 생기는데, index.html과 서비스 워커 파일에 no-cache를 설정하면 해결된다.
    - 해시값이 포함된 JS 파일들은 파일명 자체가 바뀌므로 immutable로 둬도 안전하다.
- buildCommand / outputDirectiory
    
    ```json
    {
      "buildCommand": "npm run build",
      "outputDirectory": "dist"
    }
    ```
    
    - Vercel이 직접 빌드할 때 사용할 명령어와 결과 폴더를 지정한다.
    - 이미 빌드된 결과물만 올릴 때는 두 항목을 비워두면 된다.
    - Flutter Web 처럼 Vercel 서버에서 빌드가 불가능한 경우, 로컬에서 빌드 후 build/web 폴더만 올리는 방식을 쓴다.
    - 이때 buildCommand가 설정되어 있으면 Vercel이 빌드를 시도하다 실패하므로 반드시 비워야 한다.

# Project Settings

- Vercel 대시보드 → 프로젝트 → Settings → Build and Deployment

| 항목 | 설명 |
| --- | --- |
| Framework Presey | 프레임워크 자동 감지, Next.js, Vite, CRA 등 자동 설정. 해당 없으면 Other |
| Build Command | 배포 시 실행할 빌드 명령어. 정적 파일 배포 시 비움 |
| Output Directory | 빌드 결과 폴더. . 으로 설정하면 현재 폴더 전체 |
| Root Directory | 소스 코드 위치. 모노레포에서 특정 폴더만 배포할 때 사용 |
- Root Directory 주의사항
    - 모노레포처럼 레포 안에 여러 프로젝트가 있을 때 특정 폴드를 지정하는 설정이다.
    - 한번 설정하면 UI에서 비우기가 까다롭다.
    - 설정이 꼬였을 때 가장 빠른 해결법은 프로젝트를 삭제하고 CLI로 새로 만드는 것이다.

# 환경 변수

- 대시보드에서 설정
    - Settings → Environment Variables에서 키-값 형태로 추가
    - Production / Preview / Deplopment 환경별로 다른 값을 설정할 수 있다.
- CLI로 주입
    
    ```bash
    vercel env add [변수명] # 환경 변수 추가 (대화형)
    vercel env ls # 환경 변수 목록
    vercel env rm [변수명] # 환경 변수 삭제
    ```
    
- Flutter Web의 경우
    - Flutter는 Dart 컴파일 시점에 변수를 코드에 삽입하는 —dart-define 방식을 쓴다.
    - 런타임에 환경 변수를 읽는 일반 웹과 달리 빌드 명령어에 직접 넣어야 한다.
        
        ```bash
        flutter build web --release \
          --dart-define=API_URL=https://api.example.com
        ```
        

# 도메인 연결

### [vercel.app](http://vercel.app) 서브도메인

- 무료 플랜에서 기본으로 제공되는 *.vercel.app 도메인이다.
- 대시보드는 Domains 탭에서 원하는 이름으로 추가할 수 있다.

### 커스텀 도메인

- 구매한 도메인을 연결할 때는 도메인 레지스트라에서 DNS 설정이 필요하다.
- Vercel이 안내하는 A 레코드 또는 CNAME을 추가하면 자동으로 SSL 인증서까지 발급해준다.

# 자동화

- 배포 과정을 스크립트로 묶어두면 매번 명령어를 칠 필요가 없다.
- Flutter Web 배포에 쓰는 스크립트 예시다.

```bash
#!/bin/bash
# ~/deploy.sh

cd ~/path/to/project

# 빌드
flutter build web --release \
  --dart-define=API_URL=https://api.example.com

# 배포용 vercel.json 복사 (buildCommand 없는 버전)
cp vercel-deploy.json build/web/vercel.json

# 배포
cd build/web
npx vercel . --prod --yes

echo "완료!"
```

```bash
chmod +x ~/deploy.sh   # 최초 1회 실행 권한 부여
~/deploy.sh            # 이후 배포할 때마다
```

- 배포용 vercel.json과 개발용 vercel.json을 분리하는 이유는 개발용에는 buildCommand가 포함되어 있을 수 있는데 빌드된 결과물 폴더에서 배포할 때 이 설정이 있으면 Vercel이 다시 빌드를 시도하기 대문이다.
