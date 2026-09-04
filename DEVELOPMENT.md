# 개발 인수인계

최종 기록일: 2026-09-04

## 현재 상태

- GitHub 저장소: <https://github.com/shinyk84/ai-pe-motion-coach>
- 배포 주소: <https://shinyk84.github.io/ai-pe-motion-coach/>
- 기본 브랜치: `main`
- 배포: GitHub Pages, `main` 브랜치의 `/` 경로
- 앱 형태: 빌드 과정이 없는 단일 정적 HTML 웹앱
- 외부 라이브러리: jsDelivr에서 불러오는 MediaPipe Camera Utils 0.3 및 Pose 0.5
- 원본 스쿼트 예제는 로컬에만 보관하며 `.gitignore`로 배포에서 제외

현재 제공하는 동작은 스쿼트, 런지, 팔 벌려 뛰기, 양팔 올리기입니다. 전면·후면 카메라를 선택할 수 있고, 동작별 피드백과 성공 횟수를 표시합니다. 영상 저장 또는 서버 업로드 기능은 없습니다.

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

- 화면: 동작 선택, 카메라 선택, 시작 버튼, 캔버스, 피드백, 측정 카드
- 공통 계산: `angle`, `distance`, `visible`, `bestSide`
- 동작 판정: `evaluateSquat`, `evaluateLunge`, `evaluateJumpingJack`, `evaluateArmRaise`
- 동작 설정: `profiles`
- 화면 표시: `setFeedback`, `drawPose`, `onResults`
- 카메라 생명주기: `startCamera`, `stopCamera`
- 횟수 판정: `phase`가 목표 자세에 도달한 뒤 시작 자세로 돌아올 때 1회 증가

MediaPipe 관절 번호는 코드의 `LEFT`, `RIGHT` 객체에 모아 두었습니다. 인식 가능성은 각 관절의 `visibility` 값으로 확인합니다.

## 새 동작 추가 방법

1. `movementSelect`에 새 `<option>`을 추가합니다.
2. `evaluate새동작(landmarks)` 판정 함수를 만듭니다.
3. 필요한 관절이 충분히 보이는지 `visible`로 먼저 확인합니다.
4. 목표 각도 또는 관절 간 거리 비율을 계산합니다.
5. `phase`로 시작·목표·완료 상태를 구분하고, 완료 시 `success()`를 호출합니다.
6. `profiles`에 측정 항목 이름, 촬영 안내, 판정 함수를 등록합니다.
7. PC와 스마트폰에서 정면·측면, 전면·후면 카메라를 모두 시험합니다.

설정 예시:

```javascript
profiles.newMovement = {
  primary: '주요 측정값',
  secondary: '보조 상태',
  guide: '학생에게 보여 줄 촬영 위치 안내',
  evaluate: evaluateNewMovement
};
```

빠른 스윙이나 던지기처럼 순서와 속도가 중요한 동작은 한 프레임의 각도만으로 판정하지 말고, 여러 프레임의 관절 위치와 시간을 저장하는 동작 단계 분석을 추가해야 합니다.

## 수정 후 확인 사항

- `index.html`의 인라인 JavaScript 문법 검사
- 390px 안팎의 세로형 모바일 화면에서 가로 스크롤이 생기지 않는지 확인
- Android Chrome과 iPhone Safari에서 카메라 권한 확인
- 전면 카메라 화면 반전과 후면 카메라 방향 확인
- 동작을 바꾸면 횟수와 `phase`가 초기화되는지 확인
- 관절이 화면 밖에 있거나 가려졌을 때 성공으로 잘못 세지 않는지 확인
- 한 번의 동작이 여러 회로 중복 집계되지 않는지 확인
- `git status`가 깨끗한지 확인한 후 푸시
- GitHub Pages 배포가 성공하고 서비스 주소가 HTTP 200으로 열리는지 확인

## 판정 기준과 한계

현재 각도 범위는 초기 교육용 기준값입니다. 학생의 체형, 옷, 촬영 거리, 카메라 높이와 방향에 따라 결과가 달라질 수 있으므로 실제 수업 표본으로 기준을 조정해야 합니다.

- 2차원 영상이므로 깊이 방향 움직임을 정확히 알 수 없음
- 한 학생만 화면에 들어오는 상황을 전제로 함
- 가림, 헐렁한 옷, 낮은 조도에서 관절 인식이 불안정할 수 있음
- 결과는 수업 보조 피드백이며 운동·의료 진단이 아님
- 통증이나 부상 위험을 단정하는 문구는 사용하지 않음

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

- 교사가 판정 각도와 안내 문구를 화면에서 설정하는 기능
- 학년 또는 난이도별 동작 기준
- 음성 안내 켜기·끄기
- 준비 카운트다운과 연습 시간 설정
- 수업용 QR 코드와 전체 화면 모드
- 여러 프레임을 사용하는 던지기·스윙 단계 분석
- 실제 학생 영상을 저장하지 않는 범위의 익명 성공 횟수 통계

학생 영상이나 개인별 결과를 저장하는 기능을 추가할 경우에는 구현 전에 촬영 동의, 개인정보 처리, 저장 기간과 접근 권한을 먼저 결정해야 합니다.
