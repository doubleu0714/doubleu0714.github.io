# 블로그

GitHub Pages + Astro 블로그. 배포 주소: https://doubleu0714.github.io

## 글 작성

- 위치: `src/content/blog/<slug>.md` (파일명이 URL이 됨: `/blog/<slug>/`)
- frontmatter 필수: `title`, `description`, `pubDate` (YYYY-MM-DD). 선택: `updatedDate`, `heroImage`
- 이미지는 `src/assets/`에 두고 상대경로로 참조 (예: `../../assets/foo.png`)
- 스키마: `src/content.config.ts`

## 개발

- Node 22 필요 (`nvm use`)
- `npm run dev` 로컬 미리보기, `npm run build` 빌드 확인
- `main`에 푸시하면 `.github/workflows/deploy.yml`이 자동 배포

@AGENTS.md
