---
name: openframe7-build
description: OpenFrame 7 소스를 지정된 BATCH OS와 환경 프로필에서 클론하고 제품별 설정, 전처리, 빌드 및 바이너리 설치를 수행한다. 신규 빌드 환경 준비나 MVS/MSP/XSP/VOS3 소스 빌드에 사용하며 DB 초기화와 서비스 검증은 openframe7-setup으로 이어간다.
---

# OpenFrame 7 빌드

선택한 프로필의 `version`(7.3/7.4), `os`(MVS/MSP/XSP/VOS3), `rdb`(tibero/oracle), `rdb_connect_string`을 읽는다. 명시적인 사용자 지시가 우선하며 필수 값이 없으면 질문한다. 예제 환경 이름만으로 선택하지 않는다. 7.3 + Oracle 조합은 지원하지 않는다. 최상위 version은 설정 스키마 버전으로 유지한다. 작업 저장소의 AGENTS.md와 선택된 환경의 실제 하위 지침을 적용한다.

## 환경과 사전 조건

- `.agents/openframe.local.yaml`을 데이터로 읽고 사용자가 지정한 프로필 또는 default_environment를 선택한다. type별 접속·인증을 따른다. 매 새 셸에서 env_path를 적용하고 SOURCE_BASE/OPENFRAME_HOME을 검증한 후 SOURCE_BASE로 이동한다.
- 새 설치 경로와 env_path가 프로필에 이미 지정되어 있으면 그 경로를 검증해 사용한다. 별도 테스트 환경이 필요한 경우 기존 env_path를 예제로 읽어 HOME에 새 환경 파일을 만든다. 새 경로가 정해지지 않았으면 확인하고 기존 소스·설치를 덮어쓰지 않는다. 초기 디렉터리를 만든 뒤 해당 파일을 적용하고 이후 모든 작업에 사용한다. 설정 YAML은 요청한 범위에서만 바꾼다.
- `openframe7-setup`의 [환경 템플릿](../openframe7-setup/assets/env_openframe7.sh.template)을 실제 예제와 비교해 사용한다. 개인 호스트·비밀번호를 공용 assets에 넣지 않는다.
- 필요한 Tmax, TCache, Tibero client 배포본은 **$HOME/packages**에서 찾는다. 없으면 경로를 질문한다. ProSort/OFCOBOL과 라이선스는 사용 가능한 예제 환경에서 확인한다. VOS3에는 ofcbpph도 확인한다.
- 패키지가 없다고 자동 설치하지 않는다. 실패 증거, 필요한 패키지명과 설치 명령을 제시해 사용자 허가를 받은 뒤 설치하고 발견한 의존성을 설치 스킬에 기록한다.
- 선택한 rdb의 연결은 필수다. `odbcinst -j`, `-q -d`, `-q -s`로 확인하고 부족한 정보는 질문한다. 접속 정보는 프로필의 rdb_connect_string과 사용자 최신 지시를 사용한다. Tibero는 tbsql/tbpc, Oracle은 sqlplus/proc와 설치된 client 라이브러리를 확인한다. 암호는 소스·로그·스킬에 넣지 않는다.

## 소스와 설정

공통 절차는 이 스킬에서 관리한다. 프로필의 `version`에 따라 7.3은 [7.3 제품별 저장소 빌드](references/build-73.md), 7.4는 [7.4 통합 저장소 빌드](references/build-74.md)를 읽고 해당 버전의 절차만 적용한다. 지원 환경이 7.4로 전환되어도 문서 이름을 바꾸지 않고 프로필의 `version`으로 선택한다. 기존 저장소는 branch/status를 확인한 뒤 `git pull --ff-only`를 먼저 시도하고 기존 변경을 보존한다. 인증 실패 시 사용자에게 요청한다. 인증은 대화형 프롬프트로 전달하며 credential 파일, URL, 환경 변수, 로그, 스킬에 저장하지 않는다.

Tmax/TCache 설치와 충돌 없는 환경 준비는 `openframe7-setup`의 선행 준비 절차를 먼저 수행한다. 7.3은 각 제품에서 ofrelease.sh를 실행하고 플랫폼 cflags/config를 준비한다. 7.4는 통합 저장소 루트에서 ofconfigure.sh의 실제 옵션을 확인해 설정한 뒤 루트 ofrelease.sh를 실행한다. 기존 예제의 옵션을 참고하되 최신 플랫폼 파일 전체를 오래된 예제로 덮어쓰지 않는다.

선택 DB의 전처리 cfg를 준비하고 실제 `.tbc`/`.pc` 입력과 Makefile의 대상 목록을 먼저 확인한다. 전처리가 필요한 모듈 디렉터리에서 `make precomp`를 실행하고 필요한 입력의 전처리가 실패하면 해결 후 재실행한다. 루트에 target이 없으면 실제 모듈 target을 확인한다. 7.3의 삭제된 입력 관련 판정은 [7.3 제품별 상세](references/build-73.md)를 따른다.

## 빌드와 판정

7.3은 Base → Batch → TACF → OS별 DB 제품 순으로 `make install`을 실행한다. 7.4는 루트 Makefile의 `make install OS=<소문자 os>`로 선택 OS의 의존 순서를 따른다. 로그와 종료 코드를 보존한다. Makefile의 실제 의존성이 우선이다. 상호의존성으로 첫 빌드가 실패하면 필요한 디렉터리로 이동해 선행 산출물을 만들고 재시도한다. 이를 해결하려고 제품 소스를 수정하거나 수정을 제안하지 않는다.

7.3 MVS HiDB rb_72가 osi_io.h를 참조하면 OSI rb_72 소스·헤더를 추가 클론한다. 이 7.3 절차에서는 **OSI 자체는 빌드·설치하지 않는다.** 7.4는 통합 저장소의 OS 프로필에 포함된 제품을 빌드한다. 다른 OS의 의존성은 실제 로그로 판단한다.

빌드 후 dev/tool/oflicgen으로 필요한 라이선스를 생성하고 최종 Tmax 설정을 준비한다. DB 초기화·기동·데이터셋/잡 검증은 `openframe7-setup`을 사용한다. 빌드 성공을 설치 검증 성공으로 보고하지 않는다. 사용자가 보존을 요청하면 소스, 환경 파일, 설치본, 로그, 테스트 JCL을 남긴다.

결과에 OS, 브랜치/커밋, 경로, 성공한 빌드, 실패와 재시도, 미검증 범위를 기록한다. 실측 결과는 [검증 기록](references/validation.md)에 따른다.

XSP에는 [XSP 실측 기록](../openframe7-setup/references/xsp-validation.md)의 빌드 옵션과 검증 한계도 확인한다.
MSP에는 [MSP 실측 기록](../openframe7-setup/references/msp-validation.md)의 빌드 옵션, 설치 유틸리티 이름과 검증 한계도 확인한다.
