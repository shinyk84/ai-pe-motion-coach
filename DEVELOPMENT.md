# 개발 인수인계

최종 기록일: 2026-09-09 (1차 개선)

## 현재 상태

- GitHub 저장소: <https://github.com/shinyk84/ai-pe-motion-coach>
- 배포 주소: <https://shinyk84.github.io/ai-pe-motion-coach/>
- 기본 브랜치: `main`
- 배포: GitHub Pages, `main` 브랜치의 `/` 경로
- 앱 형태: 빌드 과정이 없는 단일 정적 HTML 웹앱
- 외부 라이브러리: jsDelivr에서 불러오는 MediaPipe Pose 0.5
  - 1차 개선에서 MediaPipe Camera Utils 0.3 의존성을 제거했습니다. 카메라 시작/정지, 프레임 전송 루프를 `getUserMedia` + `requestAnimationFrame`으로 직접 구현하여 캡처 해상도(`video.videoWidth/videoHeight`)를 그대로 읽고 강제 리사이즈 없이 캔버스 크기를 맞출 수 있게 했습니다.
- 원본 스쿼트 예제는 로컬에만 보관하며 `.gitignore`로 배포에서 제외

현재 제공하는 동작은 스쿼트, 런지, 팔 벌려 뛰기, 양팔 올리기입니다. 전면·후면 카메라를 선택할 수 있고, 동작별 피드백과 성공 횟수를 표시합니다. 영상 저장 또는 서버 업로드 기능은 없습니다.

## 1차 개선 (2026-09-09) 요약

- 카메라 화면 비율을 실제 입력 영상 비율에 맞춰 동적으로 처리 (4:3/640×480 고정 제거)
- 준비 카운트다운(5/10/15초, 기본 10초)과 "바로 시작"/"취소" 제공, 카운트다운 중 횟수 미집계
- 원거리 연습 모드: 카메라 화면 위에 큰 글자 피드백 + 모서리 성공 횟수 오버레이
- 프레임마다 갱신되던 피드백을 우선순위·최소 유지시간 기반으로 안정화
- 관절 visibility를 이용한 몸 전체 인식 안내 (사람 없음/화면 밖 잘림/신뢰도 낮음/정상)
- `profiles`에 `name`, `view`, `instructions`, `distanceHint` 추가, 동작별 안내 카드와 "설명 듣기" 버튼 제공
- 모바일 터치 영역(44px 이상), safe-area inset, prefers-reduced-motion 대응

## 파일 구성

```text
ai-pe-motion-coach/
├─ index.html       앱 화면, 스타일, 자세 판정 로직 전체
├─ README.md        사용자 및 기본 작업 안내
├─ DEVELOPMENT.md   개발 구조와 인수인계 기록
└─ .gitignore       로컬 전용 파일 제외 설정
```

## `index.html` 구조

별도 프레임워크 없이 HTML, CSS, JavaScript가 한 파일에 들어 있습니다.

- 화면: 설정(동작/카메라/준비 시간/음성/원거리 모드), 안내 카드, 캔버스(카운트다운·원거리 오버레이 포함), 피드백, 측정 카드
- 공통 계산: `angle`, `distance`, `visible`, `bestSide`
- 동작 판정: `evaluateSquat`, `evaluateLunge`, `evaluateJumpingJack`, `evaluateArmRaise`
- 동작 설정: `profiles`
- 몸 전체 인식: `assessBodyVisibility`, `updateVisibilityHint`
- 화면 표시: `requestFeedback`/`renderFeedback`, `drawPose`, `onResults`, `layoutStage`
- 카메라 생명주기: `openCamera`, `startFrameLoop`, `stopEverything`, `handleCameraError`
- 준비 카운트다운: `startCountdown`, `tickCountdown`, `skipCountdown`, `cancelCountdown`, `beginPractice`
- 횟수 판정: `phase`가 목표 자세에 도달한 뒤 시작 자세로 돌아올 때 1회 증가 (단, `appState === 'practice'`일 때만 판정)

MediaPipe 관절 번호는 코드의 `LEFT`, `RIGHT` 객체에 모아 두었습니다. 인식 가능성은 각 관절의 `visibility` 값으로 확인합니다.

### 화면 비율 처리 방식 (1차 개선)

기존에는 `<canvas width="640" height="480">`와 `.stage { aspect-ratio: 4/3 }`가 고정되어 있어 세로로 촬영하는 스마트폰 영상이 눌리거나 잘렸습니다. 지금은 다음 순서로 실제 영상 비율을 그대로 반영합니다.

1. `openCamera()`가 `getUserMedia`로 스트림을 받아 `video.srcObject`에 연결하고, `loadedmetadata` 이벤트(또는 이미 `readyState >= 2`인 경우 즉시)로 `video.videoWidth`/`video.videoHeight`가 채워질 때까지 기다립니다(`waitForVideoReady`).
2. 실제 폭/높이 비율을 `videoAspect`에 저장하고, `<canvas>`의 내부 픽셀 크기(`canvas.width`/`canvas.height`)를 `video.videoWidth`/`video.videoHeight`와 **완전히 동일하게** 맞춥니다. 따라서 `ctx.drawImage(results.image, 0, 0, canvas.width, canvas.height)`가 원본을 늘리거나 압축하지 않습니다.
3. `layoutStage()`가 `.stage-frame`의 실제 폭과 `window.innerHeight`를 기준으로 `videoAspect`를 유지하면서 들어갈 수 있는 최대 크기를 계산해 `.stage`의 인라인 `width`/`height`(px)를 직접 설정합니다. 폭 기준으로 계산한 높이가 뷰포트 높이의 68%(최대 760px)를 넘으면 높이 기준으로 다시 계산해 폭을 줄입니다. 이 방식으로 세로 영상이 데스크톱에서 과도하게 커지는 것을 막습니다(대신 CSS `aspect-ratio` 속성은 쓰지 않습니다. 고정폭 컨테이너에 `aspect-ratio`만 적용하면 세로 영상에서 높이 상한을 자연스럽게 줄 수 없기 때문입니다).
4. `<canvas>`의 CSS 크기는 `width:100%; height:100%`이며, `.stage`가 이미 `videoAspect`와 동일한 비율로 계산되어 있으므로 가로/세로 스케일이 항상 같습니다 → 왜곡이 발생하지 않습니다. 관절 좌표는 `landmark.x * canvas.width`, `landmark.y * canvas.height`로 그리므로 캔버스 내부 해상도가 영상과 동일한 한 항상 영상 위에 정확히 겹칩니다.
5. `window.resize`/`orientationchange` 이벤트에서 `layoutStage()`를 다시 호출해 화면 회전이나 창 크기 변경에 대응합니다. 카메라(전/후면) 전환 시에는 새 스트림의 `videoWidth`/`videoHeight`를 다시 읽어 `videoAspect`와 캔버스 크기를 갱신합니다.
6. 전면 카메라는 `canvas.mirror` 클래스(`transform: scaleX(-1)`)로 좌우 반전합니다. 영상과 관절선이 같은 `<canvas>` 위에 그려지므로 이 변환이 항상 함께 적용되어 어긋나지 않습니다.

### 피드백 우선순위와 안정화 (1차 개선)

기존에는 `setFeedback()`이 매 프레임 호출되어 문구가 너무 빨리 바뀌었습니다. 지금은 `requestFeedback(key, message, type, priority, opts)` 하나를 거쳐야만 화면/음성이 갱신됩니다.

- 우선순위(`PRIORITY`): `SYSTEM(100)` 카메라 오류 > `SUCCESS(90)` 성공 횟수 > `WARNING(70)` 중요한 자세 교정·몸 인식 경고 > `TARGET(55)` 목표 자세 도달 안내 > `INFO(40)` 일반 안내
- 유지시간(`holdDuration`): `SUCCESS`는 2000ms, 그 외는 1500ms(`HOLD_DEFAULT_MS`)
- 교체 규칙(`feedbackState`에 저장): 새 메시지의 우선순위가 현재보다 **높으면 즉시 교체**, **같거나 낮으면** 현재 메시지의 유지시간이 지난 뒤에만 교체. 완전히 동일한 `key`+메시지면 아무 것도 하지 않습니다(깜빡임 방지).
- 음성(`maybeSpeak`): 메시지가 실제로 바뀌었고, 마지막 발화 이후 2.5초(`SPEECH_GAP_MS`) 이상 지났을 때만 `speechSynthesis.speak()`를 호출합니다. `speechSynthesis.cancel()`도 이 시점에만 호출되므로 매 프레임 취소가 발생하지 않습니다.
- 성공(횟수 증가) 메시지는 `{ forceSpeak: true }`로 요청해 2.5초 간격 제한을 건너뜁니다. 성공 메시지는 매번 횟수가 달라져 텍스트가 고유하므로 반복 발화 문제가 없고, 반복 횟수 안내를 놓치지 않아야 하기 때문입니다. 카운트다운(`speakCountdown`)도 별도 경로로 매초 발화하며 2.5초 간격 제한을 받지 않습니다(마지막 3초 "3, 2, 1, 시작" 요구사항 때문).
- `renderFeedback()` 한 번의 호출이 하단 `#feedback`, 카운트다운 오버레이의 상태줄(`#countdownStatus`), 원거리 모드 오버레이(`#distanceMessage`/`#distanceIcon`)를 동시에 갱신합니다. 화면 노출 여부는 각 요소의 `hidden` 속성으로만 제어하므로 상태 로직은 한 곳에만 존재합니다.
- 동작 변경, 카메라 전환, 카메라 종료 시 `resetFeedbackState()`를 호출해 이전 상태(특히 오류/성공 메시지)가 새 시도에 남아있지 않도록 합니다.

### 준비 카운트다운 상태 (1차 개선)

`appState`는 `'idle' | 'countdown' | 'practice'` 세 가지 값을 가집니다.

1. `handleStartClick()` → `openCamera()`로 권한 요청과 영상 준비 대기 → `startCountdown()`
2. `startCountdown()`은 `appState = 'countdown'`으로 전환하고, 선택한 동작 이름·촬영 안내를 화면(`#countdownMovement`/`#countdownGuide`)과 음성(`speakCountdown`)으로 안내한 뒤 `settings.prepSeconds`부터 1초 간격(`setInterval`)으로 감소시킵니다. 남은 시간이 3 이하이면 매초 숫자를 음성으로도 안내하고, 0이 되면 "시작"을 안내한 뒤 0.5초 후 `beginPractice()`를 호출합니다.
3. `onResults()`는 `appState === 'practice'`이고 `assessBodyVisibility()`가 "정상"으로 판단할 때만 `profiles[...].evaluate()`(횟수/자세 판정)를 호출합니다. 즉 카운트다운 중에는 골격 오버레이와 몸 전체 인식 안내는 계속 보이지만 횟수는 절대 올라가지 않습니다.
4. 카운트다운 화면의 "바로 시작"(`skipCountdown`)은 타이머를 즉시 정리하고 `beginPractice()`를 호출합니다. "취소"(`cancelCountdown`)는 타이머를 정리하고 `stopEverything()`으로 카메라까지 종료합니다.
5. 연습 중 카메라(전/후면)를 바꾸면 현재 스트림을 정리하고 새 스트림으로 카운트다운부터 다시 시작합니다(해상도·비율이 달라질 수 있어 학생이 다시 자리를 잡을 시간을 주기 위함입니다). 동작만 바꾸는 경우에는 카메라를 재시작하지 않고 횟수·`phase`만 초기화합니다.

### profiles 구조 (1차 개선)

```javascript
profiles.squat = {
  name: '스쿼트',                 // 카운트다운/안내 카드 제목
  view: 'side',                  // 'side' | 'front' → 안내 카드의 촬영 방향 배지
  guide: '몸 전체가 보이도록 카메라 옆으로 서세요.', // 카운트다운 시작 시 음성/화면 안내
  distanceHint: '카메라 옆에서 2~3걸음 떨어져 몸 전체가 보이게 서요.', // 안내 카드의 권장 거리
  instructions: ['두 발을 어깨너비로 벌려 서요.', '엉덩이를 뒤로 빼며 천천히 앉아요.', '무릎을 펴며 천천히 일어나요.'], // 2~3단계 수행 방법
  primary: '무릎 각도', secondary: '상체 상태', // 측정 카드 라벨
  evaluate: evaluateSquat
};
```

새 동작을 추가할 때는 위 필드를 모두 채우고, `movementSelect`에 `<option>`을 추가한 뒤 `evaluate새동작`을 등록합니다(아래 "새 동작 추가 방법" 참고).

## 새 동작 추가 방법

1. `movementSelect`에 새 `<option>`을 추가합니다.
2. `evaluate새동작(landmarks)` 판정 함수를 만듭니다.
3. 필요한 관절이 충분히 보이는지 `visible`로 먼저 확인합니다.
4. 목표 각도 또는 관절 간 거리 비율을 계산합니다.
5. `phase`로 시작·목표·완료 상태를 구분하고, 완료 시 `success()`를 호출합니다.
6. `profiles`에 측정 항목 이름, 촬영 안내, 판정 함수를 등록합니다.
7. PC와 스마트폰에서 정면·측면, 전면·후면 카메라를 모두 시험합니다.

설정 예시(1차 개선 이후 구조 — 위 "profiles 구조" 참고):

```javascript
profiles.newMovement = {
  name: '새 동작',
  view: 'side',
  guide: '학생에게 보여 줄 촬영 위치 안내',
  distanceHint: '권장 촬영 거리 안내',
  instructions: ['1단계 설명', '2단계 설명', '3단계 설명'],
  primary: '주요 측정값',
  secondary: '보조 상태',
  evaluate: evaluateNewMovement
};
```

판정 함수 안에서는 `setFeedback` 대신 `requestFeedback(key, message, type, priority)`를 호출해야 새 우선순위/유지시간 규칙을 따릅니다. 일반 안내는 `PRIORITY.INFO`, 목표 자세 도달은 `PRIORITY.TARGET`, 중요한 자세 교정(균형이 심하게 무너진 경우 등)은 `PRIORITY.WARNING`을 사용하세요.

빠른 스윙이나 던지기처럼 순서와 속도가 중요한 동작은 한 프레임의 각도만으로 판정하지 말고, 여러 프레임의 관절 위치와 시간을 저장하는 동작 단계 분석을 추가해야 합니다.

## 수정 후 확인 사항

- `index.html`의 인라인 JavaScript 문법 검사
- 360px, 390px, 430px 세로형 모바일 화면과 768px 태블릿, 1280px 데스크톱에서 가로 스크롤이 생기지 않는지 확인
- Android Chrome과 iPhone Safari에서 카메라 권한 확인
- 전면 카메라 화면 반전과 후면 카메라 방향 확인, 세로/가로 입력 영상 모두 왜곡 없이 표시되는지 확인
- 영상과 관절선(스켈레톤)이 항상 겹치는지 확인
- 카운트다운 중에는 횟수가 절대 올라가지 않는지 확인
- 동작을 바꾸면 횟수와 `phase`가 초기화되는지 확인
- 관절이 화면 밖에 있거나 가려졌을 때 성공으로 잘못 세지 않는지 확인
- 한 번의 동작이 여러 회로 중복 집계되지 않는지 확인
- 원거리 모드에서 피드백 문구와 성공 횟수가 카메라 화면 위에서 크게 읽히는지 확인
- 같은 음성 문구가 반복 재생되지 않고, 발화 사이 최소 간격이 지켜지는지 확인
- JavaScript 콘솔에 오류가 없는지 확인
- `git status`가 깨끗한지 확인한 후 푸시
- GitHub Pages 배포가 성공하고 서비스 주소가 HTTP 200으로 열리는지 확인

## 판정 기준과 한계

현재 각도 범위는 초기 교육용 기준값입니다. 학생의 체형, 옷, 촬영 거리, 카메라 높이와 방향에 따라 결과가 달라질 수 있으므로 실제 수업 표본으로 기준을 조정해야 합니다.

- 2차원 영상이므로 깊이 방향 움직임을 정확히 알 수 없음
- 한 학생만 화면에 들어오는 상황을 전제로 함
- 가림, 헐렁한 옷, 낮은 조도에서 관절 인식이 불안정할 수 있음
- 결과는 수업 보조 피드백이며 운동·의료 진단이 아님
- 통증이나 부상 위험을 단정하는 문구는 사용하지 않음

### 1차 개선에서 확인된 한계

- 원거리 모드의 몸 전체 인식 안내(`assessBodyVisibility`)는 코·양쪽 발목·어깨·엉덩이의 `visibility`와 화면 가장자리 근접 여부만으로 판단하는 간단한 휴리스틱입니다. 조명이 나쁘거나 옷이 헐렁하면 "화면 밖" 안내가 과도하게 뜰 수 있어, 실제 교실 환경에서 임계값(0.04, 0.35~0.5 등)을 다시 조정해야 할 수 있습니다.
- `speechSynthesis`의 실제 목소리 품질과 지원 여부는 브라우저·OS·설치된 음성 엔진에 따라 달라집니다(예: 일부 Android 브라우저는 한국어 음성이 없을 수 있음). "음성 시험" 버튼으로 각 기기에서 사전 확인이 필요합니다.
- 카메라 소스를 전/후면으로 전환하면 연습 중이라도 카운트다운을 처음부터 다시 진행합니다. 이는 해상도·비율 변경에 학생이 대응할 시간을 주기 위한 의도적 설계이지만, 빠르게 카메라만 바꾸고 싶은 경우 다소 번거로울 수 있습니다.
- MediaPipe Camera Utils 대신 직접 구현한 `getUserMedia` + `requestAnimationFrame` 루프는 표준 브라우저 API만 사용하므로 대부분의 최신 브라우저에서 동작하지만, 일부 구형 브라우저의 `env(safe-area-inset-*)`/`dvh` 지원 차이는 실제 기기에서 추가로 확인이 필요합니다.

종목을 늘릴 때는 모든 동작에 하나의 공통 각도 기준을 적용하지 말고, 동작별 촬영 방향과 평가 관절을 별도로 설계합니다.

## Git 및 배포 절차

다른 컴퓨터에서 최초 한 번:

```powershell
git clone https://github.com/shinyk84/ai-pe-motion-coach.git
cd ai-pe-motion-coach
```

매 작업 시작과 종료:

```powershell
git pull
# 파일 수정 및 로컬 확인
git add index.html README.md DEVELOPMENT.md
git commit -m "변경 사항 요약"
git push
```

`main`에 푸시하면 GitHub Pages가 자동으로 다시 배포됩니다. 배포 설정은 GitHub 저장소의 `Settings → Pages`에서 확인할 수 있습니다.

둘 이상의 컴퓨터에서 동시에 수정했다면 푸시하기 전에 반드시 `git pull`을 실행합니다. 충돌이 발생했을 때 `--force`로 덮어쓰지 말고 충돌 부분을 확인해 병합합니다.

## 다음 개발 후보

1차 개선에서 음성 안내 켜기/끄기, 준비 카운트다운(시간 선택 포함)은 구현했습니다. 남은 후보:

- 교사가 판정 각도와 안내 문구를 화면에서 설정하는 기능
- 학년 또는 난이도별 동작 기준
- 수업용 QR 코드와 전체 화면 모드
- 여러 프레임을 사용하는 던지기·스윙 단계 분석
- 실제 학생 영상을 저장하지 않는 범위의 익명 성공 횟수 통계
- 새로운 종목(구기, 체조 등) 추가 및 영상 저장 기능 — 이번 1차 개선 범위에서는 제외

학생 영상이나 개인별 결과를 저장하는 기능을 추가할 경우에는 구현 전에 촬영 동의, 개인정보 처리, 저장 기간과 접근 권한을 먼저 결정해야 합니다.
