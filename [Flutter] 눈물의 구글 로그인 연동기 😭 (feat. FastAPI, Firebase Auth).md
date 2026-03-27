안녕하세요! 현재 Flutter + FastAPI 조합으로 뉴스 요약 앱 **'Newsum'**을 개발하고 있는 대학생 개발자입니다. 👨‍💻

오늘은 저를 꼬박 2일 동안 잠 못 들게 했던... 'Flutter 앱에서 구글 로그인하고 백엔드(FastAPI)로 토큰 보내기' 성공 후기를 남겨보려고 합니다. 혹시 저처럼 [firebase_auth/internal-error]를 보고 멘붕에 빠진 분들이 계시다면, 이 글이 네비게이션이 되길 바랍니다.

## 🛠️ 개발 환경 및 목표

- App: Flutter (Provider 상태관리)
- Backend: Python FastAPI
- Auth: Firebase Authentication

Goal:

- 앱에서 구글 로그인 버튼 클릭
- Firebase에서 인증 토큰(ID Token) 발급
- 이 토큰을 FastAPI 서버로 전송하여 유저 검증 및 DB 저장

간단해 보이죠? 저도 그런 줄 알았습니다... 하지만 현실은 에러의 연속이었습니다.

## 🚨 1. 처절했던 삽질의 기록 (에러 모음집)

성공 방법을 바로 알려드리기 전에, 제가 겪었던 대표적인 에러 3가지와 그 해결 과정을 먼저 공유합니다. (이거 알면 시간 50% 단축 가능합니다.)

### 💥 에러 1: 이름 충돌 "GoogleSignIn 생성자가 없다고?"

user_provider.dart 파일에서 GoogleSignIn()을 호출했는데, 자꾸 The class 'GoogleSignIn' doesn't have an unnamed constructor 라는 에러가 떴습니다.

**원인:** 제가 lib 폴더 안에 패키지 이름이랑 똑같은 google_sign_in.dart라는 파일을 만들어 놨던 게 화근이었습니다. 컴파일러가 패키지가 아니라 제가 만든 파일을 참조하고 있었던 거죠.

**해결:**

- 파일 이름을 auth_service.dart로 변경.
- 확실하게 하기 위해 import 문에 별칭(alias)을 사용했습니다.

```dart
import 'package:google_sign_in/google_sign_in.dart' as signIn;
// ...
final signIn.GoogleSignIn _googleSignIn = signIn.GoogleSignIn();
```

### 💥 에러 2: 로그인 버튼 누르면 앱이 '픽' 꺼짐 (iOS Crash)

iOS 시뮬레이터에서 구글 로그인 버튼만 누르면 아무런 로그도 없이 앱이 종료되었습니다.

**원인:** iOS는 외부 브라우저(구글 로그인 창)에 갔다가 다시 **우리 앱으로 돌아오는 길(URL Scheme)**을 Info.plist에 등록해줘야 하는데, 이걸 빼먹어서 갈 곳을 잃고 앱이 죽은 것이었습니다.

**해결:** GoogleService-Info.plist 파일 내의 REVERSE_CLIENT_ID 값을 찾아 Info.plist에 추가했습니다. (자세한 방법은 아래 '성공 가이드'에!)

### 💥 에러 3: [firebase_auth/internal-error] (제일 중요 ⭐)

앱이 꺼지진 않는데, 로그인이 안 되고 콘솔에 internal-error만 떴습니다. 코드는 완벽한데 도대체 왜?!

**원인:** Firebase 콘솔 설정 미스였습니다. Google 로그인을 켜긴 했는데, **'지원 이메일(Support Email)'**을 선택하지 않아서 발생한 문제였습니다.

**해결:** Firebase Console -> Authentication -> Sign-in method -> Google 설정에서 이메일 선택 후 저장. (이거 하나 때문에 반나절 날렸습니다...)

## 🚀 2. 이것만 따라 하면 성공! (완벽 설정 가이드)

수많은 시행착오 끝에 정리한 **'절대 실패하지 않는 정석 루트'**입니다. 웹사이트에서 수동으로 하지 말고, **터미널(CLI)**을 믿으세요.

### Step 1. FlutterFire CLI로 초기 설정 (한방에 끝내기)

Firebase 웹사이트에서 앱 등록하고 파일 다운받고... 그러지 마세요. 터미널에서 아래 명령어면 끝납니다.

```bash
# 혹시 기존 설정이 꼬였다면 과감하게 삭제
rm firebase.json
rm lib/firebase_options.dart

# 설정 시작
flutterfire configure
```

**[중요 포인트]**

플랫폼 선택 화면이 나오면 화살표 키로 이동해서 Space바를 눌러 **Android와 iOS 모두 체크(✔)**한 뒤 엔터를 눌러야 합니다! (그냥 엔터 치면 안드로이드만 됩니다.)

### Step 2. iOS 설정: URL Scheme 등록하기

flutterfire configure가 다 해주지만, 이 설정만큼은 직접 Info.plist에 넣어줘야 합니다.

- ID 찾기: 방금 생성된 lib/firebase_options.dart 파일을 엽니다.
- iosClientId 값을 찾습니다. (예: 123456-abcdefg.apps...)
- 뒤집기: 이 값을 거꾸로 뒤집습니다. -> com.googleusercontent.apps.123456-abcdefg
- 등록하기: ios/Runner/Info.plist 파일을 열고 아래 코드를 추가합니다.

```xml
<key>CFBundleURLTypes</key>
<array>
    <dict>
        <key>CFBundleTypeRole</key>
        <string>Editor</string>
        <key>CFBundleURLSchemes</key>
        <array>
            <string>com.googleusercontent.apps.123456-abcdefg...</string>
        </array>
    </dict>
</array>
```

### Step 3. Android 설정: SHA-1 지문 등록

안드로이드는 '디지털 지문'이 없으면 구글 로그인을 거부합니다.

- 키 뽑기: 터미널에 아래 명령어를 입력합니다.

```bash
keytool -list -v -keystore ~/.android/debug.keystore -alias androiddebugkey -storepass android -keypass android
```

- 등록하기: 출력된 SHA1: XX:XX... 코드를 복사해서, Firebase 콘솔 > 프로젝트 설정 > 내 앱(Android) > 디지털 지문 추가에 붙여넣습니다.
- 동기화: 터미널에서 flutterfire configure를 한 번 더 실행해서 최신 정보를 앱에 반영합니다.

## 🎉 3. 결과 및 마무리

위 과정을 모두 마치고 앱을 실행하니... 드디어!

```dart
// 터미널에 찍힌 감동의 로그
flutter: 🎉 로그인 성공 (신규유저: true)
```

구글 로그인 창이 뜨고, 계정을 선택하니 제 백엔드 서버(FastAPI)까지 토큰이 안전하게 전달되어 DB에 유저 정보가 저장되는 것을 확인했습니다.

처음엔 "로그인 하나 붙이는 게 이렇게 힘들 줄이야" 싶었지만, 덕분에 OAuth 인증 흐름과 **네이티브 앱 설정(Info.plist, Gradle)**에 대해 제대로 공부하게 된 것 같습니다.

혹시 저와 같은 에러로 고생하고 계신 분이 있다면, 꼭 지원 이메일 확인이랑 URL Scheme 설정을 다시 한번 체크해보세요!
