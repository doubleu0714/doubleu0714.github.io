# 블로그

GitHub Pages + Astro([AstroPaper](https://github.com/satnaing/astro-paper) 테마) 블로그. 배포 주소: https://doubleu0714.github.io

## 글 작성

- 위치: `src/content/posts/<slug>.md` (파일명이 URL이 됨: `/posts/<slug>/`). `_`로 시작하는 파일/폴더는 무시됨
- frontmatter 필수: `title`, `description`, `pubDatetime` (예: `2026-11-07T10:00:00+09:00`)
- 선택: `tags`, `modDatetime`, `featured`(홈 추천 글), `draft`(true면 비공개), `ogImage`
- 태그 규칙: 첫 태그는 분류로 `개발` 또는 `일상` 중 하나, 그 뒤에 주제 태그(예: `Spring`)
- 이미지는 `src/assets/`에 두고 상대경로 또는 `@/assets/...`로 참조
- 스키마: `src/content.config.ts`, 사이트 설정: `astro-paper.config.ts`, 한국어 UI 문구: `src/i18n/lang/ko.ts`
- 소개 페이지: `src/content/pages/about.md`

## 글쓰기 규칙

- **모든 기술 글에는 비개발자도 이해할 수 있는 예시를 곁들인다.** 본문의 맥락상 예시가 필요한 곳에 일상생활 비유(식당, 택배, 도서관 등)를 넣고, 그다음 기술 설명과 코드를 보여준다.
- 전문 용어는 처음 나올 때 한 줄로 풀어서 설명한다.
- **모든 코드 예제는 Java가 아닌 Kotlin으로 작성한다.** 빌드 스크립트도 Gradle Kotlin DSL(`build.gradle.kts`)을 쓰고, 코드 파일 경로는 `src/main/kotlin`, `src/test/kotlin`을 기준으로 한다. 코드 블록의 언어 표기는 `kotlin`으로 한다.
- **기술 글 초반(도입부 바로 다음)에 "왜 이 기술을 쓰는가"를 5 Whys로 작성한다.** "왜?"를 다섯 번 이어 물으며 근본 이유까지 내려간다. **5 Whys에는 비유·예시를 넣지 않는다.** **글 본문과 제목에는 "5 Whys"라는 단어를 쓰지 않는다.** 제목은 "왜 OOO를 쓸까?" 형태로 쓴다. 제목 아래에 방법을 설명하는 안내 문장("왜?를 다섯 번 물어보겠습니다" 등)은 쓰지 않고 바로 첫 질문으로 시작한다.
  - 5 Whys에는 반드시 두 가지가 포함되어야 한다: ① **왜 이 기술을 사용해야 하는지**, ② **무엇을 극복하기 위해 이 기술이 탄생했는지**(기술 이전의 방식과 그 한계).
  - 마지막 정리도 "사용하는 이유"와 "극복한 문제"를 각각 한 줄로 쓴다.

## 개발

- Node 22 필요 (`nvm use`)
- `npm run dev` 로컬 미리보기, `npm run build` 빌드 확인 (검색은 빌드 후에만 동작)
- `main`에 푸시하면 `.github/workflows/deploy.yml`이 자동 배포

@AGENTS.md
