# 교인용 알림 구독 페이지

[[15. 푸시 알림(WhatsApp 대안) 설계 - OneSignal 연동 (2026-09-05)]]의 "컴포넌트 A"에 해당하는 실제 코드. 교인이 이 페이지를 열고 "알림 받기"만 누르면 앱 설치 없이 웹 푸시 구독이 완료된다.

## 파일 구성

- `index.html` — 유일한 화면. 교회명/로고/버튼 하나.
- `manifest.json` — PWA 설치(홈 화면 추가) 메타데이터.
- `OneSignalSDKWorker.js` — OneSignal이 요구하는 서비스워커 파사드 파일(그대로 두면 됨, 수정 불필요).
- `icon-192.png`, `icon-512.png` — **아직 없음, 배포 전 추가 필요** (아래 "남은 작업" 참고).

## 아직 안 한 것 — 배포 전 필요

1. **아이콘 2개 추가**: `icon-192.png`(192×192), `icon-512.png`(512×512). 종 모양이나 교회 로고 기반으로 제작해 이 폴더에 넣고 `manifest.json`/`index.html`의 `apple-touch-icon` 경로와 맞추면 됨. 지금은 파일이 없어 홈 화면 추가 시 아이콘이 깨져 보일 수 있음(기능 자체는 아이콘 없이도 동작함).
2. **GitHub Pages 배포**: 이 `subscribe/` 폴더를 별도 저장소(예: `church-companion-subscribe`)로 만들거나, 기존 `church-companion` 저장소에 `docs/` 폴더로 넣고 GitHub Pages를 활성화. 빌드서버(git 사용 가능)에서 처리 필요 — 이 Windows PC는 git이 없음([[no-git-node-on-this-pc]]).
3. **Cloudflare Worker 릴레이 제작**: 아직 없음 — 다음 단계 작업.
4. **church-companion 앱에 Settings 필드 + 발송 화면 추가**: 아직 없음 — 그 다음 단계.

## 교회별 링크 만드는 법

배포 후 URL에 파라미터만 바꿔서 교회마다 같은 페이지를 재사용한다:

```
https://<배포주소>/subscribe/?app=<OneSignal App ID>&church=은혜교회&logo=https://.../logo.png
```

- `app` (필수): 그 교회가 OneSignal에서 무료로 발급받은 App ID
- `church` (선택): 화면에 표시할 교회 이름. 생략하면 "우리 교회"로 표시
- `logo` (선택): 교회 로고 이미지 URL. 생략하면 🔔 아이콘으로 표시

이 링크를 QR코드로 만들어 주보에 인쇄하거나, 카톡/왓츠앱으로 한 번만 공유하면 된다.

## 로컬에서 테스트하는 법

OneSignal 웹푸시는 **HTTPS(또는 localhost)에서만** 동작한다(서비스워커 보안 요건). 로컬 테스트 시:

```
npx serve subscribe
```

또는 이 PC 기준 Node가 없으므로, 빌드서버에서 `python3 -m http.server`나 `npx serve`로 띄운 뒤 `http://<빌드서버IP>:포트`로 접속 — 단, 이건 `http://`라서 **서비스워커가 등록되지 않는다**. 실제 푸시 동작 테스트는 GitHub Pages(HTTPS) 배포 후에만 정확히 확인 가능하다. localhost(`http://localhost:포트`)는 예외적으로 보안 컨텍스트로 취급되므로, 로컬 PC에서 직접 띄운 `localhost`라면 테스트 가능.

## iOS 관련 주의사항 (코드에 이미 반영됨)

- iOS Safari는 **홈 화면에 추가된 상태(Standalone)에서만** 알림 권한 요청이 가능하다. `index.html`은 이를 감지해서(`navigator.standalone`), 아직 홈 화면에 없으면 버튼을 비활성화하고 "공유 → 홈 화면에 추가" 안내를 자동으로 보여준다.
- **iOS 16.4 미만은 웹 푸시 자체가 불가능** — 이 경우 사용자에게 별도 안내가 필요하나, 현재 코드는 버전 감지까지는 하지 않음(브라우저가 `Notification`/`serviceWorker` API 자체를 지원 안 하면 에러 메시지로 걸러지긴 하나, "왜 안 되는지"를 iOS 버전 기준으로 명시하진 않음 — 필요시 추가 가능).

## OneSignal App ID는 어디서 발급받나

각 교회가 [onesignal.com](https://onesignal.com)에서 무료 계정을 만들고 "Web Push" 앱을 하나 생성하면 App ID와 REST API Key가 발급된다. App ID는 이 구독 페이지 링크에, REST API Key는 (아직 안 만든) Cloudflare Worker 릴레이 설정에 들어간다 — REST API Key는 절대 이 구독 페이지나 church-companion 앱 코드에 직접 넣지 않는다(설계 문서의 "보안 이슈" 항목 참고).
