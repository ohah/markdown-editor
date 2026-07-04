# 보안 모델

마크다운 파일은 신뢰할 수 없는 입력으로 취급한다. 이 에디터는 로컬 데스크탑 앱이지만, 문서는 인터넷에서 복사되거나 저장소를 통해 들어올 수 있으므로 렌더 경로에서 스크립트 실행을 허용하지 않는다.

## 기본 정책

- 리딩 뷰와 익스포트 HTML은 최종 주입 직전에 sanitize를 거친다.
- raw HTML은 기본적으로 안전한 subset만 허용한다.
- `<script>`, 이벤트 핸들러 속성, 위험한 URL scheme, 임의 inline style은 허용하지 않는다.
- 라이브 프리뷰는 CM6 decoration으로 표현하고, 사용자 HTML을 실행하지 않는다.
- 로그와 테스트 아티팩트에는 파일 본문, 홈 경로, 토큰 같은 민감정보를 그대로 남기지 않는다.

## 렌더 파이프라인 권장 순서

```text
마크다운 텍스트
  -> remark-parse
  -> remark-gfm
  -> remark/rehype 플러그인
  -> rehype-sanitize  (최종 안전 경계)
  -> rehype-stringify
  -> 리딩 뷰 주입 / 익스포트
```

sanitize는 가능한 한 마지막 변환 단계에 둔다. sanitize 이후에 HTML/HAST를 새로 만드는 플러그인을 실행하면 보안 경계를 우회할 수 있기 때문이다.

## 플러그인 정책

새 렌더 플러그인을 추가할 때는 다음을 PR에 남긴다.

- 생성하는 HTML 태그와 속성
- 필요한 sanitize schema 확장
- 성능 영향
- 악성 입력 fixture와 기대 출력

Mermaid, KaTeX, 코드 하이라이트처럼 HTML/SVG를 생성하는 플러그인은 최종 sanitize 전 단계에서 실행한다. 플러그인이 생성한 태그가 필요하면 schema를 좁게 확장하고 sanitize 회귀 테스트를 추가한다.

## 리소스 로딩

초기에는 외부 리소스 로딩을 보수적으로 다룬다.

- `javascript:` URL은 항상 거부한다.
- 워크스페이스 밖 `file://` 접근은 허용하지 않는다.
- 원격 이미지는 별도 설정 전까지 기본 허용 여부를 구현 시점에 사용자와 다시 결정한다.
- 로컬 이미지는 경로 검증을 통과한 워크스페이스 내부 파일만 렌더한다.

## Tauri 경계

Tauri IPC 커맨드는 필요한 기능만 노출한다. 파일 접근은 `crates/fs-engine`의 경로 검증을 통과해야 하며, 프론트엔드는 임의 파일 시스템 권한을 직접 갖지 않는다([tauri-shell.md](tauri-shell.md)).

## 검증

- sanitize 회귀 테스트: script, event handler, dangerous URL, SVG payload.
- 플러그인별 악성 fixture.
- IPC 경로 탈출 테스트.
- 로그 민감정보 제거 테스트.
