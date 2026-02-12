# 🌐 Office AI Suite - Document Translator

AI 번역(Gemini API)과 문서 스타일링(Meiryo 폰트 통합)을 한 번에 해결하는 웹 기반 도구입니다.

## ✨ 주요 기능
- **AI 번역**: 한국어 ↔ 일본어 양방향 번역 (Google Gemini API 활용).
- **자동 폰트 패치**: 일본어 출력 시 최적화된 **Meiryo** 폰트를 자동 적용.
- **이미지 교체 리뷰**: PPTX 내 이미지를 확인하고 번역된 문서에서 강조(Glow 효과)할 대상을 선택 가능.
- **보안 중심**: API 키는 서버로 전송되지 않으며, 사용자의 브라우저 내에서만 활용됩니다.

## 🚀 시작하기

### 1. 깃허브 호스팅 (GitHub Pages)
이 프로젝트는 정적 HTML로 구성되어 있어 깃허브 페이지를 통해 간편하게 호스팅할 수 있습니다.
1. 이 리포지토리를 본인의 GitHub 계정으로 **Push**합니다.
2. 리포지토리 설정(**Settings**) > **Pages** 탭으로 이동합니다.
3. Build and deployment에서 `main` 브랜치를 선택하고 **Save**를 클릭합니다.
4. 몇 분 후 제공되는 URL을 통해 접속합니다.

### 2. 로컬 실행
1. `index.html` 파일을 브라우저로 엽니다.
2. Gemini API 키를 입력하고 번역할 파일을 업로드합니다.

## ⚠️ 주의사항
- **보안**: `API.txt` 등의 파일은 절대 리포지토리에 포함하지 마세요 (`.gitignore`에 추가됨).
- **API 키**: 도구 실행 시 직접 입력창에 키를 입력하여 사용하세요.

## 🛠 기술 스택
- **Frontend**: HTML5, CSS3 (Vanilla), JavaScript
- **Libraries**: [JSZip](https://stuk.github.io/jszip/)
- **AI Client**: Google Gemini API
