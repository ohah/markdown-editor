# PR 체크리스트

모든 PR은 아래를 만족한다. 규칙 본문은 [필수 프로젝트 규칙](project-rules.md)을 단일 출처로 두고, 이 문서는 PR 제출 시 빠뜨리지 않도록 하는 체크리스트다.

## 제출 전 확인

제출 전 검증 명령(`bun run check`, `cargo clippy/fmt/test`, `git diff --check`)의 단일 출처는 [development-commands.md 완료 전 확인](development-commands.md)이다. 여기서 명령을 중복 나열하지 않는다.

> 위 검사 중 **린트·유닛 테스트는 push 시 pre-push 훅이 자동으로 강제**한다([development-commands.md Git 훅](development-commands.md)). 훅이 최소 강제 게이트이고, CI는 유닛 테스트만 돌린다(나머지 백로그, [testing-strategy.md](testing-strategy.md)).

## PR 본문에 포함할 것

PR 제목과 본문은 한국어로 작성한다. 코드 식별자·명령어·고유명사는 필요한 경우 원문을 유지한다([project-rules.md 기본 규칙](project-rules.md)).

```text
## 무엇을 / 왜
- 이 변경이 지키는 사용자 편집 경험은 무엇인가

## 전략 영향 평가
- 성능 예산(입력 레이턴시/시작/메모리)에 영향이 있는가? 있으면 벤치 방향
- 아키텍처 경계(편집 엔진 격리, 텍스트 원천, Worker 렌더, FSD 방향)를 지키는가
- 전략을 바꿔야 한다면: decisions.md를 먼저 갱신했는가, 사용자와 논의했는가

## 테스트 / 검증
- 추가/변경한 테스트와 그것이 증명하는 것
- 자동 검증이 불가능한 부분과 그 이유 + 수행한 수동 검증
  (예: macOS E2E 불가 → 수동 흐름 확인, 한글 IME → 수동 조합 확인)

## 문서 정합성
- 코드와 달라진 docs/를 갱신했는가
```

## 전략 영향 평가 기준

- **성능**: hot path(타이핑·라이브 프리뷰·대용량 스크롤·리딩 뷰 렌더)를 건드리면 [성능 예산](performance-budget.md) 벤치의 방향을 확인한다. 회귀면 근거와 완화책을 남긴다.
- **경계**: 편집 엔진(`packages/editor`)의 CM6 세부가 앱 레이어로 새지 않았는지, 텍스트 원천 모델을 우회하지 않았는지, 전체 렌더를 메인 스레드에 넣지 않았는지, FSD import 방향을 어기지 않았는지 확인한다.
- **보안**: HTML 렌더 경로에 최종 sanitize를 우회하지 않았는지, 플러그인 schema 확장을 검증했는지, IPC 경로 검증을 건너뛰지 않았는지 확인한다.

## 문서와 코드가 달라졌을 때

- 코드가 설계 문서와 달라지면, 임의로 코드를 밀지 말고 **문서(특히 [decisions.md](decisions.md))를 먼저 갱신**하거나 사용자에게 보고한다.
- "구현하다 보니 설계가 틀렸다"가 드러나면 그 자체를 보고한다 — 한계를 숨기고 진행하지 않는다.

## 커밋 / 브랜치

커밋 컨벤션(conventional-commit·소문자), 브랜치 형식(`type/kebab-설명`), `main` 직접 푸시 금지와 변경 범위 한정 규칙은 [필수 프로젝트 규칙 기본 규칙](project-rules.md)을 단일 출처로 둔다.
