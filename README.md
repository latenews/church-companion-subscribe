# 교인용 알림 구독 페이지

[[15. 푸시 알림(WhatsApp 대안) 설계 - OneSignal 연동 (2026-09-05)]]의 "컴포넌트 A"에 해당하는 실제 코드. 교인이 이 페이지를 열고 "알림 받기"만 누르면 앱 설치 없이 웹 푸시 구독이 완료된다.

## 2026-09-05: 실기기 검증 완료, 배포처 변경됨

**실제 배포 주소(운영 중)**: `https://church-companion-subscribe.pages.dev/` — **GitHub Pages가 아니라 Cloudflare Pages**로 서빙 중. 이유: GitHub Pages 프로젝트 사이트처럼 도메인의 서브패스(`username.github.io/repo/`)에서 호스팅하면 OneSignal 웹푸시의 서비스워커 등록이 항상 도메인 루트를 시도하다 404로 실패하는 근본적 비호환 문제가 실측으로 확인됨(자세한 원인은 [[15. 푸시 알림(WhatsApp 대안) 설계 - OneSignal 연동 (2026-09-05)]]의 트러블슈팅 6번 참고). GitHub 저장소(`latenews/church-companion-subscribe`)는 소스 보관용으로 계속 쓰되, 실제 서빙은 Cloudflare Pages 기준으로 한다.

안드로이드 실기기(Chrome)로 구독 → 발송 → 알림 수신·클릭까지 전체 확인 완료.

## 파일 구성

- `index.html` — 유일한 화면. 교회명/로고/버튼 하나. (영어로 작성됨 — 남아공 사용 대상이라 한국어 아님)
- `manifest.json` — PWA 설치(홈 화면 추가) 메타데이터.
- `OneSignalSDKWorker.js` — 파일명은 관례상 이렇게 두지만, **내용은 `importScripts("https://cdn.onesignal.com/sdks/web/v16/OneSignalSDK.sw.js")`여야 함**(v16 기준 CDN 파일명이 `OneSignalSDKWorker.js`가 아니라 `OneSignalSDK.sw.js`로 바뀌었음 — 처음에 이 착오로 한 차례 404를 겪었으니 다른 프로젝트에 복사할 때 주의).
- `icon-192.png`, `icon-512.png` — 마스터 아이콘(`X:\작업\개발\교인 관리 앱\아이콘\마스터_아이콘_1024.png`)에서 리사이즈해 적용 완료.

## 새 교회 온보딩 시 해야 할 일 (체크리스트)

1. 그 교회가 OneSignal 무료 계정을 만들고 **Web Push** 앱 생성 (Site URL은 반드시 `https://church-companion-subscribe.pages.dev` 그대로 — 이 정적 페이지는 여러 교회가 URL 파라미터로 공유하는 구조라 교회마다 새로 만들 필요 없음. **단, OneSignal 앱 자체는 교회마다 별도로 만들어야 함**)
2. Keys & IDs에서 App ID, REST API Key 확인 — REST API Key 발급 화면에서 **"IP Allowlist" 체크박스가 꺼져 있는지 반드시 확인**(기본값이 켜짐+빈 목록이라 모든 요청이 차단되는 함정이 있었음)
3. [[relay]] 폴더 README를 따라 그 교회 전용 Cloudflare Worker 배포 + 시크릿 3종 설정
4. 이 페이지 링크(`?app=<그 교회 App ID>&church=<교회명>`)를 교인들에게 공유

## 교회별 링크 만드는 법

URL에 파라미터만 바꿔서 교회마다 같은 페이지를 재사용한다:

```
https://church-companion-subscribe.pages.dev/?app=<OneSignal App ID>&church=Grace Church&logo=https://.../logo.png
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
