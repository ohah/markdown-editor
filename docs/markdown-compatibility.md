# 마크다운 호환성

초기 목표는 **GitHub Flavored Markdown(GFM) 수준**의 편집·렌더링이다. 이 문서는 어떤 문법을 기준으로 삼고, 무엇을 의도적으로 미루는지 정한다.

## 기준 레퍼런스

우선순위는 다음과 같다.

1. [github/cmark-gfm](https://github.com/github/cmark-gfm) — GitHub의 GFM 확장 CommonMark 구현과 테스트 스위트.
2. [GitHub Flavored Markdown Spec](https://github.github.com/gfm/) — 공개 GFM 명세.
3. [GitHub Docs: Basic writing and formatting syntax](https://docs.github.com/github/writing-on-github/getting-started-with-writing-and-formatting-on-github/basic-writing-and-formatting-syntax) — 실제 GitHub 사용자 문서 기준.

`github/cmark-gfm`은 동작 오라클로 쓴다. 테스트 fixture를 가져와 저장소에 포함하려면 라이선스와 출처 표기를 확인한 뒤 별도 PR에서 처리한다([references.md](references.md)).

## 초기 지원

| 영역 | 상태 | 비고 |
| --- | --- | --- |
| CommonMark 기본 문법 | 지원 | heading, paragraph, emphasis, blockquote, list, code, link, image 등 |
| GFM table | 지원 | `remark-gfm`과 렌더 스냅샷으로 검증 |
| GFM task list | 지원 | 체크박스 렌더. 편집 토글 UX는 후속 기능 |
| GFM strikethrough | 지원 | `~~text~~` |
| GFM autolink literal | 지원 | URL/email 자동 링크 |
| Raw HTML | 제한 지원 | 최종 sanitize를 통과한 안전한 subset만 렌더([security-model.md](security-model.md)) |

## 초기 미지원 / 백로그

| 영역 | 상태 | 이유 |
| --- | --- | --- |
| Obsidian wikilink/backlink | 백로그 | GFM 기준 밖. 워크스페이스 링크 모델이 필요 |
| YAML frontmatter 의미 처리 | 백로그 | 초기에는 텍스트 보존이 우선. 메타데이터 인덱싱은 후속 |
| GitHub alerts/admonitions | 백로그 | GitHub UX에는 있지만 GFM core 기준과 분리해 판단 |
| Mermaid | 백로그 | 보안·성능·sanitize schema가 필요 |
| 수식(KaTeX/MathJax) | 백로그 | GFM core 기준 밖 |
| PDF/HTML 익스포트 옵션 | 백로그 | 리딩 뷰 안정화 이후 |
| GitHub issue/PR mention 해석 | 미지원 | 로컬 파일 편집기에서 GitHub 서버 컨텍스트가 없음 |

## 검증 원칙

- 렌더 스냅샷은 GFM fixture를 기준으로 만든다.
- Lezer 기반 라이브 프리뷰와 unified 기반 리딩 뷰가 눈에 띄게 다른 결과를 내면 문법별로 허용/수정 여부를 기록한다.
- 원문 마크다운 서식 보존이 렌더 예쁨보다 우선한다.
- 동작이 GitHub와 다르면 의도한 차이인지 버그인지 PR에 남긴다.
