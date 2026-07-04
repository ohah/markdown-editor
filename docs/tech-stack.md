# 기술 스택

이 문서는 확정된 기술 선택과 버전 정책의 단일 출처다. 각 선택의 *이유*는 [설계 의사결정 기록](decisions.md)을 본다.

## 확정 스택

| 영역 | 선택 | 비고 |
| --- | --- | --- |
| 셸(데스크탑 런타임) | **Tauri 2.x** | OS 웹뷰 사용 → 작은 바이너리·낮은 메모리·빠른 시작. 백엔드는 Rust. |
| 프론트엔드 프레임워크 | **React 19** | UI 크롬(사이드바·탭·팔레트·설정)용. 무거운 편집은 CM6이 담당. |
| 빌드 도구 | **Vite** | 프론트엔드 dev 서버/번들러. Tauri가 이를 감싼다. |
| 패키지 매니저/런타임 | **Bun** | 워크스페이스(모노레포), 스크립트 러너, 테스트 러너 후보. |
| 프론트엔드 아키텍처 | **Feature-Sliced Design(FSD)** | 레이어/슬라이스/세그먼트 규칙은 [frontend-fsd.md](frontend-fsd.md). |
| 에디터 엔진 | **CodeMirror 6** | 텍스트를 진실의 원천으로 두는 편집 표면. 라이브 프리뷰는 자체 decoration. |
| 인에디터 파서 | **Lezer(markdown)** | CM6 `@codemirror/lang-markdown`에 내장. 증분 파싱·하이라이트. |
| 렌더/익스포트 파서 | **unified(remark + rehype)** | 리딩 뷰·HTML 익스포트용. Web Worker에서 실행. |
| JS/TS 린트 | **oxlint** | 빠른 린터. FSD 경계는 `steiger`(옵션)로 별도 강제. |
| JS/TS 포맷 | **oxfmt** | Oxc 포매터(Prettier 호환). 미성숙 이슈 발견 시 한계로 보고. |
| Rust 빌드/테스트 | **cargo** | `src-tauri`와 공용 crate. |
| Rust 린트 | **clippy** | `cargo clippy`. CI 게이트에 포함. |
| Rust 포맷 | **rustfmt** | `cargo fmt`. clippy는 린터라 포맷과 별도. |
| 단위 테스트(FE) | **Vitest** | Vite 네이티브. 커버리지(v8) 지원. |
| 단위 테스트(Rust) | **cargo test** | 커버리지는 `cargo-llvm-cov`. |
| E2E | **WebdriverIO + tauri-driver** | Linux/Windows CI. **macOS는 tauri-driver 미지원**(한계, [testing-strategy.md](testing-strategy.md)). |

## 파서 선택에 대한 주의

사용자 요청의 "rehype"는 unified 생태계로 해석한다. 실제 데이터 흐름은 다음과 같이 역할이 나뉜다.

```text
마크다운 텍스트
  --(remark-parse)-->  mdast(마크다운 AST)
  --(remark-rehype)--> hast(HTML AST)
  --(rehype-* 플러그인)--> 변환/보안(sanitize)
  --(rehype-stringify)--> HTML  (리딩 뷰 / 익스포트)
```

즉 마크다운 *파싱*은 `remark`, HTML *변환·렌더*는 `rehype`가 담당한다. 편집 화면의 실시간 하이라이트는 이 파이프라인이 아니라 CM6 내장 Lezer가 담당한다(성능상 증분 파서가 필요). 상세는 [parser-pipeline.md](parser-pipeline.md).

## 버전 정책

- 모든 의존성은 **최신 안정 버전**을 기본으로 채택한다(사용자 요청).
- 정확한 버전은 lockfile(`bun.lock`, `Cargo.lock`)에 **핀**으로 고정한다. "latest"를 런타임에 떠도는 상태로 두지 않는다.
- 스캐폴딩 시점에 실제로 설치된 버전을 이 문서의 아래 표에 기록한다(현재는 계획 단계라 미기록).
- 메이저 업그레이드(예: Tauri 2→3, React 19→20)는 [decisions.md](decisions.md)에 ADR로 남긴 뒤 진행한다.

### 설치된 버전(스캐폴딩 후 기록)

| 패키지 | 버전 | 기록일 |
| --- | --- | --- |
| _(미기록 — 초기 스캐폴딩 PR에서 채운다)_ | | |

## 툴 버전 핀(선택)

Bun·Rust toolchain·Node 버전을 재현 가능하게 고정하려면 `mise`(또는 `rust-toolchain.toml` + `.bun-version`)를 쓴다. 도입 여부는 스캐폴딩 시 결정하고 [development-commands.md](development-commands.md)에 명령을 기록한다.
