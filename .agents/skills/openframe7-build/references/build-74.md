# OpenFrame 7.4 통합 저장소 빌드

7.4 프로필에 적용한다. 7.3 제품별 저장소·release 절차와 혼용하지 않는다.

## 소스 준비

- 통합 저장소의 SSH 주소는 `ssh://git@192.168.51.106:2022/openframe/openframe7/ofsrc.git`이며 클론 자체가 `$SOURCE_BASE`다. 사용 전에 실행 환경의 접근 권한과 현재 clone 주소를 확인한다.
- 대상 브랜치는 사용자 지시와 현재 원격 저장소에서 확인하고 `product_info` 등 메타데이터로 제품 버전을 검증한다. 기본 브랜치나 특정 커밋을 고정하지 않는다.
- 기존 경로는 덮어쓰지 않고 루트 지침에 따라 상태 확인·pull을 수행한다.
- 실제 트리의 `AGENTS.md`, Makefile, `ofconfigure.sh`, `ofrelease.sh`, 설정 샘플을 읽는다.

## 설정·전처리·빌드

- `ofconfigure.sh`의 현재 옵션으로 OS·DB·COBOL·Sort와 플랫폼을 선택한다. 설정 재생성 전 기존 로컬 설정을 보존한다.
- 옵션 전달만으로 적용되었다고 판단하지 않고 생성된 `config.local`의 활성 값과 실제 Makefile 소비 변수를 확인한다. 보완이 필요하면 해당 로컬 설정만 수정한다.
- 저장소 루트의 `ofrelease.sh`를 사용하며 대상 선택은 현재 도구의 지원 옵션을 따른다.
- 선택 DB 전처리 cfg와 실제 입력/target을 확인해 필요한 `make precomp`를 수행한다.
- 루트 Makefile이 지원하는 OS별 목록·설치 target으로 의존 제품과 빌드 순서를 확인하고 설치한다. 순환 의존성도 해당 Makefile의 처리를 따른다.
- 제품 제외는 현재 요청 범위와 Makefile의 override 지원을 확인해 적용한다. 특정 환경의 실패를 모든 빌드의 기본 제외 조건으로 만들지 않는다.
- 설치된 DB 연결 드라이버는 심볼릭 링크의 실제 대상과 의존 라이브러리로 확인한다. 버전 접미사가 있는 산출물을 고정 파일명 검사로 누락하지 않는다.
- 라이선스 도구의 설정·빌드 target을 확인하고 설치 목적지를 `$OPENFRAME_HOME` 아래로 명시해 공용 도구를 보존한다.

Oracle 프로필은 [Oracle 클라이언트·전처리](build-74-oracle.md)를 추가로 읽는다. 초기화·기동·실행 검증은 `openframe7-setup`을 따른다.
