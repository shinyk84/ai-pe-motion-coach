# AI 체육 동작 코치

초등 체육수업에서 카메라로 동작을 연습할 수 있는 정적 웹앱입니다.

- 서비스: <https://shinyk84.github.io/ai-pe-motion-coach/>
- 저장소: <https://github.com/shinyk84/ai-pe-motion-coach>
- 개발 인수인계: [DEVELOPMENT.md](./DEVELOPMENT.md)

## 포함된 동작

- 스쿼트
- 런지
- 팔 벌려 뛰기
- 양팔 올리기

각 동작은 MediaPipe Pose가 찾은 2차원 관절 위치를 바탕으로 간단한 피드백과 반복 횟수를 제공합니다. 전문적인 운동 또는 의료 진단 도구가 아닙니다.

## 다른 컴퓨터에서 시작하기

Git을 설치한 뒤 터미널에서 다음 명령을 실행합니다.

```powershell
git clone https://github.com/shinyk84/ai-pe-motion-coach.git
cd ai-pe-motion-coach
python -m http.server 8000
```

브라우저에서 <http://localhost:8000>을 엽니다. 처음 커밋할 때 Git이 작성자 정보를 요구하면 해당 컴퓨터에서 다음과 같이 설정합니다.

```powershell
git config user.name "shinyk84"
git config user.email "GitHub에 등록한 이메일"
```

작업 전에는 `git pull`, 작업 후에는 아래 순서로 GitHub에 반영합니다.

```powershell
git status
git add index.html README.md DEVELOPMENT.md
git commit -m "변경 내용을 설명하는 메시지"
git push
```

## 로컬 실행

카메라는 보안 연결에서만 안정적으로 사용할 수 있으므로 파일을 직접 열기보다 로컬 웹 서버를 사용합니다.

```powershell
python -m http.server 8000
```

브라우저에서 `http://localhost:8000`을 엽니다. 같은 와이파이의 스마트폰에서 PC의 IP 주소를 HTTP로 여는 방식은 카메라 권한이 제한될 수 있으므로, 스마트폰 테스트에는 위의 GitHub Pages HTTPS 주소를 사용합니다.

## 배포

`main` 브랜치 최상위 폴더가 GitHub Pages에 연결되어 있습니다. `index.html`이 저장소 최상위에 있어 별도 빌드 과정은 필요하지 않으며, `main`에 푸시하면 잠시 후 서비스에 자동 반영됩니다.

## 개인정보

현재 구현은 영상을 녹화하거나 서버로 업로드하지 않습니다. 카메라 영상은 브라우저에서 실시간 자세 분석에만 사용합니다.
