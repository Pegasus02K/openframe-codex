# OpenFrame Base 추가 지침

## 적용 범위
`$SOURCE_BASE/base/**`에 적용한다. 환경·Git·증거·보고와 여러 제품의 지침 적용은 루트 `AGENTS.md`를 따른다.

## 담당 기능과 매뉴얼
Base는 데이터셋 생성·I/O·런타임 할당, CPM 및 OpenFrame schema를 통한 데이터 변환과 커맨드라인 도구를 담당한다.

- 매뉴얼: `{{manual_base}}/openframe_base`
- 생성·삭제·조회: `dscreate`, `dsdelete`, `dslist`, `listcat`
- 내용 조회: `dsview`, `spfedit`
- 마이그레이션: `dsmigin`, `dsmigout`, `cobgensch`

## 변경과 검증
- Jira의 데이터셋·공용 API·도구 관련 정보는 현재 소스·설정과 대조한다.
- 공통 라이브러리, 컴파일 옵션, API·헤더·설정 변경은 실제 사용처와 의존 제품의 호환성을 확인하고 관련 빌드·테스트를 수행한다.
- 빌드 산출물이나 생성 파일은 직접 수정하지 않는다.
- C/C++ 코드 수정은 `common-unit-test-write`로 관련 단위 테스트를 작성·실행하고 gcov로 변경 코드를 검증한다. 기존 테스트가 없으면 새 테스트를 작성하되, 단위 테스트 불가·과도한 코드 길이·새 모킹 필요 조건에서는 해당 스킬에 따라 사용자에게 추가 지침을 묻는다. 제품 동작의 최소 재현 검증은 단위 테스트를 보완한다.
