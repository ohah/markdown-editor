# 제품 범위

이 문서는 이 에디터가 **무엇을 만들고, 무엇을 아직 하지 않는지** 정하는 단일 출처다. 기술 선택의 이유는 [decisions.md](decisions.md), 구현 순서는 [implementation-plan.md](implementation-plan.md)를 본다.

## 제품 방향

이 에디터는 원본 마크다운을 직접 다루는 **파워유저용 데스크탑 도구**다. 핵심 사용자는 문서 저장소, 개발 노트, 기술 문서, 개인 지식 베이스를 일반 파일과 Git 중심으로 관리하며, 렌더된 화면보다 **원본 텍스트 보존과 빠른 편집감**을 더 중요하게 본다.

기본 입장은 다음과 같다.

- 원천은 항상 마크다운 텍스트다. 렌더 뷰는 파생 결과다.
- 마크다운 문법을 숨기는 워드프로세서가 아니라, 마크다운을 빠르게 읽고 편집하게 돕는 도구다.
- 초기 호환성 기준은 GitHub Flavored Markdown 수준으로 둔다([markdown-compatibility.md](markdown-compatibility.md)).
- Obsidian/Typora 대비 우위 주장은 시작 속도, 메모리, 바이너리 크기, 입력 레이턴시를 실측해 말한다([performance-budget.md](performance-budget.md)).

## 프로토타입 범위

앱 실행 프로토타입까지는 **macOS 로컬 실행만 완료 기준**이다. 이 단계의 목표는 멀티플랫폼 검증이 아니라 다음 경계를 실제 앱에서 통과시키는 것이다.

- Tauri 앱이 macOS에서 실행된다.
- 파일을 열고, CM6 소스 모드에서 편집하고, 저장한다.
- 편집 텍스트가 원본 그대로 디스크에 반영된다.
- 최소 성능 벤치 하니스가 같은 머신에서 반복 측정 가능한 형태로 붙는다.

Windows/Linux는 제품 목표에서 제외하지 않는다. 다만 Linux WebKitGTK와 Windows WebView2 검증은 프로토타입 이후 단계로 미룬다.

## v1 MVP

v1은 파워유저가 실제 문서 작업에 쓸 수 있는 최소 제품이다.

- 단일 파일 열기/편집/저장
- 소스 모드, 라이브 프리뷰, 리딩 뷰
- GitHub Flavored Markdown 수준의 렌더링
- 워크스페이스 파일 트리와 검색
- 기본 설정, 테마, 단축키
- 시작/메모리/입력 레이턴시 벤치 리포트

## 명시적 비목표

초기에는 다음을 하지 않는다.

- Typora식 트리 원천 WYSIWYG 워드프로세서
- Obsidian 수준 플러그인 생태계
- 실시간 협업 편집
- 클라우드 동기화, 계정, 모바일 앱
- 자체 노트 데이터베이스나 독자 파일 포맷
- Obsidian/Logseq 전용 문법 전체 호환

## 백로그

다음은 중요하지만 프로토타입 완료 조건은 아니다.

- crash recovery, autosave, 백업 파일, 저장 충돌 병합 UI
- Windows/Linux 자동 E2E와 배포 패키징
- 서명, 자동 업데이트, 릴리스 채널
- PDF/문서 익스포트
- GitHub 외 Markdown 방언 확장
