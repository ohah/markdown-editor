# 마크다운 파이프라인 (unified: remark / rehype)

마크다운 처리는 **두 경로로 이원화**한다. 편집 화면 실시간 하이라이트는 CM6 내장 Lezer가, 리딩 뷰·익스포트의 전체 렌더는 unified(remark→rehype)가 담당한다([decisions.md ADR-004](decisions.md)).

## 왜 이원화하는가

| 경로 | 파서 | 이유 |
| --- | --- | --- |
| 편집 화면(하이라이트·라이브 프리뷰) | **Lezer markdown**(CM6 내장) | 매 키 입력마다 재파싱 → 증분 파서 필수. |
| 리딩 뷰 / 익스포트(HTML·통계) | **unified(remark/rehype)** | 전체 렌더 + 풍부한 플러그인(GFM·수식·mermaid·sanitize). |

편집 화면에 unified를 쓰면 전체 재파싱 비용 때문에 타이핑이 막힌다. 반대로 리딩 뷰에 Lezer만 쓰면 HTML 변환·플러그인 생태계가 빈약하다. 그래서 나눈다.

## unified 파이프라인

```text
마크다운 텍스트
  --(remark-parse)------>  mdast (마크다운 AST)
  --(remark-gfm 등)------>  mdast (GFM: 표/체크박스/취소선/자동링크)
  --(remark-rehype)------>  hast  (HTML AST)
  --(rehype-* 플러그인)-->  hast  (수식/코드 하이라이트/각주 등)
  --(rehype-sanitize)---->  hast  (최종 XSS 차단, 기본 on)
  --(rehype-stringify)--->  HTML  (리딩 뷰 주입 / 파일 익스포트)
```

- **보안 기본값**: `rehype-sanitize`는 항상 켜고, 가능한 한 최종 HTML 직전 단계에 둔다. 신뢰할 수 없는 문서의 raw HTML/스크립트를 실행하지 않는다([security-model.md](security-model.md)).
- **소스맵**: 가능하면 hast 노드에 원문 위치를 실어, 리딩 뷰 클릭 → 소스 위치 스크롤(양방향 동기)을 지원한다.

## Web Worker 경계

전체 렌더는 무거우므로 **메인 스레드에서 돌리지 않는다**. `packages/markdown`을 순수 코어 + Worker 진입점으로 나눈다.

```text
packages/markdown/
  core.ts         순수 함수: text -> { html, hast, headings, stats } (Node/Worker 어디서나 테스트 가능)
  worker.ts       Worker 진입점: postMessage 계약(요청 id, 텍스트, 옵션) -> 결과
  client.ts       메인 스레드 측 클라이언트: 요청 디바운스 + 최신 요청만 반영(stale 취소)
```

메인 스레드 흐름:

```text
편집 텍스트 변경
  -> client.render(text) (디바운스 ~50-100ms)
  -> Worker에 postMessage
  -> Worker가 unified 렌더
  -> 결과 HTML/hast를 메인에 전달
  -> 리딩 뷰 갱신 (편집은 그동안 한 번도 안 막힘)
```

- **stale 취소**: 사용자가 계속 타이핑하면 이전 렌더 요청 결과는 버린다(최신 요청 id만 반영).
- **증분 목표(후속)**: 초기엔 전체 렌더로 시작하고, 병목이 확인되면 블록 단위 부분 렌더로 최적화한다(성급하게 하지 않는다).

## 플러그인 정책

- 코어(항상): `remark-parse`, `remark-gfm`, `remark-rehype`, `rehype-sanitize`, `rehype-stringify`.
- 옵션(지연 로드): 수식(KaTeX 계열), 다이어그램(mermaid), 코드 하이라이트. 무거운 것은 리딩 뷰 진입/해당 노드 등장 시에만 동적 import 한다.
- 새 플러그인은 (1) 성능 영향, (2) 생성하는 태그/속성, (3) sanitize schema 확장 필요 여부를 [security-model.md](security-model.md)와 PR에 기록하고 추가한다.

## 테스트

- `core.ts`는 순수 함수라 스냅샷 테스트가 쉽다: 입력 마크다운 → 기대 HTML/headings/stats([verification-matrix.md](verification-matrix.md)).
- sanitize 회귀(스크립트 주입 문서 → 스크립트 제거)는 보안 테스트로 고정한다.
- Worker 계약(postMessage 형식, stale 취소)은 클라이언트 단위 테스트로 검증한다.
