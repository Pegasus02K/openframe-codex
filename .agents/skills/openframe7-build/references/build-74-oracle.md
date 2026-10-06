# OpenFrame 7.4 Oracle 빌드

프로필이 `version: "7.4"`, `rdb: oracle`일 때 [7.4 통합 빌드](build-74.md)에 추가로 적용한다. OS·DB 접속값은 선택한 local YAML 프로필을 따른다. 아래 MVS 실측을 다른 OS의 성공으로 일반화하지 않는다.

## 독립 환경과 클라이언트

- 사용자가 이름 선택을 맡기면 리모트의 기존 환경 파일과 소스·설치 디렉터리를 조사하여 충돌 없는 이름을 정한다. local YAML의 env_path는 요청에 따라 예제로만 참조하고 YAML은 수정하지 않는다. 신규 파일을 적용한 뒤 SOURCE_BASE/OPENFRAME_HOME/TMAXDIR/TCACHE_HOME의 realpath를 확인한다. 압축 해제 전 TMAXDIR와 TCACHE_HOME도 신규 설치 아래인지 확인한다.
- 예제 환경의 Oracle·OFCOBOL·ProSort·LLVM 경로를 실제 파일과 비교한다. ORACLE_HOME/bin의 sqlplus/proc, ORACLE_LIB_HOME의 libclntsh, ORACLE_HEADER_HOME의 sqlca.h/sqlcpr.h/oratypes.h/oci.h를 확인한다. 전체 클라이언트의 precomp/public만 지정하면 oratypes.h가 없어 C 컴파일이 실패할 수 있다. 이번 배포본의 sdk/include에는 필요한 헤더가 함께 있었다.
- Oracle client lib를 LD_LIBRARY_PATH에 추가하고 unixODBC DSN의 ServerName과 YAML의 접속 대상을 비교한다. DSN에 있는 예제 UserID를 신규 DB 계정으로 간주하지 않는다. SQL*Plus와 실제 ODBC SELECT USER 결과가 지정 계정인지 확인한다. 암호는 인증 입력 또는 출력하지 않는 stdin으로 전달한다.
- DB 초기화는 지정 스키마의 객체 수 0개를 먼저 확인한다. 기존 객체가 있으면 create/init/테스트 reset을 진행하지 않고 그 스키마의 용도를 확인한다.

## configure 결과를 실제 값으로 검증

루트 ofconfigure.sh의 현재 옵션과 config.sample을 확인하고 MVS의 Oracle/OFCOBOL/ProSort 조합을 생성한다. rc1에서는 `--set`을 전달해도 토큰이 아닌 활성 기본값과 주석 후보에 없는 버전 값은 바뀌지 않았다. 생성 파일의 활성 줄을 검증하고 필요한 **config.local만** 보완한다.

| 파일 | 이번 MVS Oracle 실측의 활성 설정 |
| --- | --- |
| base/make/config.local | DATABASE_SWITCH_ORACLE='YES', COBOL_SWITCH_OFCOBOL='YES', ORACLE_VERSION=23, SORT_ENGINE_SELECT='PROSORT' |
| batch/make/config.local | ORACLE_VERSION=23, COBOL_COMPILER='OFCOBOL', DB_CONNECT_TYPE_SELECT='UNIXODBC', DBDUMP_UTILITY_SELECT='ORACLE', OPTION_PLILIB='N.A.' |
| ims/make/config.local | DATABASE_SELECT='ORACLE_ESQL', COBOL_COMPILER_SELECT='OFCOBOL' |
| osi/make/config.local | COBOL_COMPILER='OFCOBOL', PLI_COMPILER='N.A.', OSI_USE_MQ='NO' |

코드가 소비하는 변수명이 기준이다. HiDB의 COBOL_COMPILER_SELECT를 COBOL_COMPILER로 대신하지 않는다. ORACLE_VERSION 누락은 드라이버 이름·링크 판정에도 영향을 준다. 설치 libtdbconnsw.so의 실제 대상이 libtdbconnora23.so의 현재 버전 산출물인지 확인한다. 예제 전체 config로 최신 파일을 덮어쓰지 않는다.

## Pro*C 전처리와 설치

- 루트 ofrelease.sh 실행 후 실제 Makefile의 precomp target을 확인한다. 이번 rc1 MVS 대상은 base/src/ds/tsam, base/src/tdbconnsw/tdbconn_ora, ims/src/common/base, ims/src/hidb/dml, ims/src/hidb/hidb였다. 다른 커밋에는 대상 목록을 재검증한다.
- 생성된 Oracle cfg의 SYS_INCLUDE에 오래된 GCC 경로가 남을 수 있다. `gcc -print-file-name=include`와 실제 존재를 비교하고 생성된 로컬 cfg를 보완한다. SOURCE_BASE 및 ORACLE_HEADER_HOME 참조와 현재 Oracle 옵션은 보존한다. 필요한 전처리가 실패하면 해결 후 다시 make precomp를 실행한다.
- TSAM은 소스의 tsam_oracle.cfg뿐 아니라 설치 scripts/tsam_oracle.cfg도 사용한다. 소스에서 성공한 cfg가 설치본에도 적용됐는지 확인한 뒤 idcams DEFINE/LIBGEN을 수행한다.
- 루트 `make install OS=mvs`를 먼저 수행한다. Tmax 헤더에 TDL_RTLD_GLOBAL이 없어 OSC가 실패하면 원인을 기록하고 호환 배포본 또는 제외 범위를 사용자 지시로 결정한다. 제품 소스 수정으로 우회하지 않는다. 이번 승인 범위의 최종 명령은 `make install OS=mvs COMPONENTS="base batch tacf ims osi"`였다. OSC 제외를 Oracle의 일반 기본값으로 만들지 않는다.
- dev는 MVS configure 대상에 없으므로 oflicgen은 --only dev 및 루트 ofrelease.sh dev로 준비한다. `INSTALL_DIR="$OPENFRAME_HOME/bin"`을 명시해 공용 HOME/bin을 덮어쓰지 않는다. 라이선스 생성에는 실제 hostname/hostid 및 도구가 요구하는 옵션을 지정한다.

## Oracle 실행 검증으로 연결

[Oracle 설치 차이와 실측](../../openframe7-setup/references/validation-74-oracle.md)을 읽고 openframe7-setup의 공통 초기화·기동·JOB·TSAM·HiDB 검증을 수행한다. 빌드 성공만으로 실행 성공을 판정하지 않는다.

- Batch tjclrun/SYSLIB/LIB_PATH는 JOB 프로세스의 라이브러리 경로를 다시 설정한다. 셸의 LD_LIBRARY_PATH가 정상이어도 이 설정에 `${ORACLE_HOME}/lib`가 없으면 Oracle에 링크된 COBOL이 libclntsh를 못 찾아 RC 127로 끝난다. 신규 설치의 활성 openframe_batch.conf와 ofconfig의 실제 VALUE에 반영하고 JOB을 재검증한다.
- 초기 import 후 파일·DB와 설정 캐시가 달라지는지 확인한다. `||` 형식의 DEFAULT_VALUE와 VALUE를 구분하여 실제 VALUE를 수정한다. 이번 재import는 DB export에 0x845가 보여도 서버가 이전 0x805를 사용했으며, 신규 인스턴스를 중지한 뒤 `ofconfig load`와 재기동으로 해소됐다. 정상 ofconfig update는 캐시도 갱신하므로 매번 load를 강제하지 않는다.
- Tmax APPDIR에서 TCache의 TPFMAGENT와 OpenFrame 서버를 찾을 수 있는지 확인한다. 이번 tar는 TPFMAGENT를 appbin64에 풀었고 APPDIR은 appbin이었다. 신규 appbin에서 해당 바이너리에 링크하여 검증했다. 공유 설치 디렉터리를 이동하지 않는다.
- ORA-12516은 연결 거부 증거로 기록한다. 단일 SQL*Plus/ODBC 연결 성공, 실제 서버 기동과 JOB 연결 성공을 구분하고 최신 로그로 재검증한다. 조회된 processes 한도만으로 원인을 단정하거나 기존 인스턴스·DB의 한도를 임의 변경하지 않는다.
