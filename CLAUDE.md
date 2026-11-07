# 블로그

GitHub Pages + Astro 블로그. 배포 주소: https://doubleu0714.github.io

## 글 작성

- 위치: `src/content/blog/<slug>.md` (파일명이 URL이 됨: `/blog/<slug>/`)
- frontmatter 필수: `title`, `description`, `pubDate` (YYYY-MM-DD). 선택: `updatedDate`, `heroImage`
- 이미지는 `src/assets/`에 두고 상대경로로 참조 (예: `../../assets/foo.png`)
- 스키마: `src/content.config.ts`

## 글쓰기 규칙

- **모든 기술 글에는 비개발자도 이해할 수 있는 예시를 곁들인다.** 본문의 맥락상 예시가 필요한 곳에 일상생활 비유(식당, 택배, 도서관 등)를 넣고, 그다음 기술 설명과 코드를 보여준다.
- 전문 용어는 처음 나올 때 한 줄로 풀어서 설명한다.
- **기술 글 초반(도입부 바로 다음)에 "왜 이 기술을 쓰는가"를 5 Whys로 작성한다.** "왜?"를 다섯 번 이어 물으며 근본 이유까지 내려간다. **5 Whys에는 비유·예시를 넣지 않는다.**
  - 5 Whys에는 반드시 두 가지가 포함되어야 한다: ① **왜 이 기술을 사용해야 하는지**, ② **무엇을 극복하기 위해 이 기술이 탄생했는지**(기술 이전의 방식과 그 한계).
  - 마지막 정리도 "사용하는 이유"와 "극복한 문제"를 각각 한 줄로 쓴다.

## 개발

- Node 22 필요 (`nvm use`)
- `npm run dev` 로컬 미리보기, `npm run build` 빌드 확인
- `main`에 푸시하면 `.github/workflows/deploy.yml`이 자동 배포

@AGENTS.md
