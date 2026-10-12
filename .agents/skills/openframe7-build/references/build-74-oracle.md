# OpenFrame 7.4 Oracle 클라이언트·전처리

`version: "7.4"`, `rdb: oracle` 프로필에서 [통합 빌드](build-74.md)에 추가로 적용한다.

## 클라이언트와 설정

- 실제 Oracle client의 `sqlplus`·`proc`, 헤더와 라이브러리를 확인한다. `ORACLE_HOME`·헤더·라이브러리 변수는 현재 설치 구조에 맞추고 접속 대상과 DSN의 계약을 확인한다.
- Pro*C 및 C 컴파일에 필요한 헤더가 모두 존재하는지 확인한다. 배포본 종류에 따라 include 위치가 다를 수 있으므로 특정 SDK 경로를 고정하지 않는다.
- 생성 설정의 DB·컴파일러·Oracle 버전 값과 실제 Makefile 소비 변수를 확인한다. 기존 환경의 전체 설정으로 현재 샘플을 덮어쓰지 않는다.
- DB 연결 드라이버의 실제 링크 대상이 선택 Oracle client 버전과 일치하는지 확인한다.

## 전처리와 실행 준비

- 현재 Makefile의 Oracle 전처리 입력과 target을 확인하고 해당 모듈에서 `make precomp`를 수행한다.
- Pro*C cfg의 GCC·Oracle include 경로를 실제 설치와 비교하고 생성 소스와 오류 출력을 확인한다. 실패하면 수정 후 재실행한다.
- 설치 scripts가 별도 cfg를 사용하면 소스 전처리 성공 여부와 별개로 설치본 cfg도 확인한다.
- JOB 프로세스의 라이브러리 탐색 설정과 서버의 실제 실행 경로를 확인한다. 셸에서 연결·로딩이 성공해도 서비스나 JOB에서 같은 결과가 보장되지는 않는다.
- 초기화·설정 적용·기동·JOB·TSAM·HiDB 검증은 `openframe7-setup`을 따른다. Oracle 연결 오류의 원인은 해당 세션의 로그·DB 상태로 조사하며 기존 DB나 공유 서비스 설정을 임의 변경하지 않는다.
