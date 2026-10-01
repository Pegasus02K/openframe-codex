# OpenFrame 7.4 MSP·XSP·VOS3 Tibero 빌드 실측

## 적용 범위

2026-10-01, RHEL 9/GCC 11, Tibero 7 FS02 PS05, Tmax5.0SP2Fix4, TCache 2.4, OFCOBOL에서 실행했다. 기존 MVS와 7.3 환경을 보존하도록 소스·설치·캐시 경로를 분리했다. 추가 OS 패키지 설치와 제품 소스 수동 수정은 없었다.

저장소 기본 브랜치는 `rc1`, 커밋은 `081cbe845f59ea5459e9e45dd04c06192d349a49`였다. product_info에서 7.4를 확인했다. 이전 MVS는 `rb_74`의 다른 커밋으로 검증했으므로 OS 차이와 커밋 차이를 함께 고려한다. 통합 저장소를 새로 다운로드한 뒤 XSP/VOS3는 그 클론에서 `git clone --no-hardlinks`로 별도 저장소를 만들고 origin을 원래 서버 URL로 설정했다. 기존 MVS 작업 트리의 생성 파일과 로컬 설정은 복사하지 않았다.

| OS | 환경 파일 | 소스 | 설치 | 결과 |
| --- | --- | --- | --- | --- |
| MSP | `$HOME/env_of74_msp_tb` | `$HOME/ofsrc/of74msp` | `$HOME/ofbin/of74msp` | Base/Batch/TACF/NDB/AIM 최종 make install 성공 |
| XSP | `$HOME/env_of74_xsp_tb` | `$HOME/ofsrc/of74xsp` | `$HOME/ofbin/of74xsp` | Base/Batch/TACF/NDB/AIM 최종 make install 성공 |
| VOS3 | `$HOME/env_of74_vos_tb` | `$HOME/ofsrc/of74vos` | `$HOME/ofbin/of74vos` | Base/Batch/TACF/NDB/OSD 최종 make install 성공 |

세 환경 모두 로그는 `$OPENFRAME_HOME/validation`이다. VOS3의 환경/경로 이름은 `vos`, 루트 Makefile의 OS 값은 `vos3`이다. `TB_USERID`는 각각 `p2tb7msp2`, `p2tb7xsp2`, `p2tb7vos2`로 설정했다. 환경 파일에는 DB 암호를 저장하지 않는다. 접속정보는 stdin으로 전달하고 설치 dbconn.conf에는 제품이 요구하는 ENPASSWD만 설정했다.

## 빌드에서 확인한 보완

- 루트 `ofconfigure.sh --platform linux-x86_64 --os <msp|xsp|vos3>`와 루트 `ofrelease.sh`를 사용했다. 루트 `make install OS=<os>`가 MSP/XSP의 Batch→AIM 순환 의존성을 2회 Batch 빌드로 처리했다. 이를 수동 제품 순서나 제품 코드 변경으로 대체하지 않았다.
- `--set TIBERO_VERSION=7`을 전달해도 rc1의 생성 config.local에는 활성 버전 지정이 없었다. Base/Batch의 **로컬** config.local에 `TIBERO_VERSION=7`을 추가하고 재빌드했다. 최종 `libtdbconnsw.so`는 세 설치 모두 `libtdbconntbr7.so.64.7_4_0_0_0`를 가리켰다. 라이브러리의 버전 접미사 때문에 고정 파일명 `libtdbconntbr7.so`의 존재만 검사하지 않는다.
- NDB config.sample의 `COBOL_COMPILER='NETCOBOL'`은 토큰이 아니어서 `--set COBOL_COMPILER=OFCOBOL`만으로 변경되지 않았다. 생성된 ndb/make/config.local의 활성 값을 OFCOBOL로 맞췄다. USE_DB_SELECT는 MSP/XSP `AIM_DB`, VOS3 `XDM_SD`로 맞추고 루트 OS 프로필로 최종 재빌드했다.
- 생성된 Tibero cfg의 GCC include는 설치된 `gcc -print-file-name=include` 결과로 맞췄다. TSAM 실행 준비 시 **설치본** scripts/tsam_tibero.cfg도 같은 설정인지 확인해야 한다.
- 실제 `make precomp` 성공 경로는 Base의 `src/ds/tsam`, `src/tdbconnsw/tdbconn_tbr`, `tdbconn_tboci`, `tdbconn_tbodbc`와 NDB의 `src/common/common`, `src/db/dml`, `src/meta`였다. 없는 디렉터리를 건너뛴 것을 전처리 성공으로 기록하지 않는다.
- dev는 OS 기본 configure 대상 밖이다. `ofconfigure.sh --platform linux-x86_64 --only dev`, `ofrelease.sh dev`를 수행했다. 이 커밋의 dev/make/rules.tool은 파일 target에도 설치 복사를 포함하며 기본 INSTALL_DIR은 `$HOME/bin`이다. `make -C dev/tool/oflicgen INSTALL_DIR="$OPENFRAME_HOME/bin"`으로 신규 설치 경로를 지정한다. 최초 빌드의 공용 경로 변경은 저장된 이전 MVS 바이너리로 복원하고 byte 비교했다. 세 환경에서 INSTALL_DIR 지정 후 강제 재빌드로 실제 복사 경로와 기존 MVS 공용 바이너리 보존을 확인했다. 파일 target 선택이나 BIN_DIR 지정만으로 공용 경로 변경을 막을 수 있다고 가정하지 않는다.
- Base/Batch/TACF/NDB2 및 MSP/XSP AIM, VOS3 OSD 라이선스를 생성했다. ELF 의존 검사에서 MSP 308개, XSP 311개, VOS3 287개 모두 누락 라이브러리 0개였다. 이는 로딩 의존성 검사이며 서비스 동작 검증을 대신하지 않는다.
- 실제 설치된 최소 유틸리티는 MSP `KDJBR14`, XSP `IEFBR14`, VOS3 `JDJDUMMY`다. JOB 작성 전에 현재 설치 경로를 다시 확인한다.

## 충돌 확인과 실행 검증 상태

기존 소스/설치/컨테이너 설정의 Tmax·TCache·DATASET_SHMKEY와 현재 `ipcs -m`, `ss -ltn`을 비교했다. 중지 환경의 포트와 키도 포함하고 Tmax 연속 키 범위를 확인했다. 아래 값은 이번 신규 설치의 예약값이며 다른 작업의 기본값으로 사용하지 않는다. 기동 직전에 재확인한다.

| OS | TPORTNO / RACPORT | DOMAIN / NODE SHMKEY | TCache | DATASET_SHMKEY |
| --- | --- | --- | --- | --- |
| MSP | 10020 / 10070 | 110000 / 110010 | 0x73035 | 0x835 |
| XSP | 10030 / 10080 | 111000 / 111010 | 0x73045 | 0x836 |
| VOS3 | 10040 / 10090 | 112000 / 112010 | 0x73055 | 0x837 |

**빌드·DB 초기화·기본 실행 검증 완료**다. 사전에 세 신규 계정의 user_objects=0 및 지정 tablespace를 확인했다. baseinit/batchinit/tacfinit/ndbinit, MSP/XSP aiminit, OS별 설정 import, TCache/Tmax/TJES 기동, 볼륨/PDS, INITPROC와 TSO 맵 생성을 수행했다. 기존 MVS 또는 7.3 계정과 서비스를 초기화하거나 재기동하지 않았다.

| 항목 | MSP | XSP | VOS3 |
| --- | --- | --- | --- |
| 최소 JOB | KDJBR14, Done(R00000), STEP R0000 | IEFBR14, Done(R00010), STEP R0010 | JDJDUMMY, Done(R00000), STEP R0000 |
| TSAM WRITE/KREAD/EREAD/PATH/KAIX/EAIX/WRITEVB/READVB/WRITEVB2/READVB2 | 10/10 통과, STEP R0010 | 10/10 통과, STEP R0010 | 10/10 통과, STEP R0000 |
| DB 객체 / invalid 객체 | 421 / 0 | 421 / 0 | 243 / 0 |
| 서버 RDY / MIN=0 NRDY 항목 | 23 / 2 | 23 / 2 | 16 / 5 |

세 환경 모두 PS 데이터셋 생성·카탈로그·삭제와 FB80 입력→dsmigin→dsmigout의 80바이트 일치 및 삭제 후 카탈로그 0건을 확인했다. TSAM 최종 레코드는 CASE1/2/3/4 각각 100/200/100/100, CASE1_VB/CASE2_VB 각각 3이다. JOB00001은 최소 JOB, JOB00002~11은 TSAM이다. JOB 상태·STEP RC와 실제 PSJOB/PODD 출력 DD를 함께 확인했다. MSP/XSP TSAM은 실행 애플리케이션 RC 0과 STATUS R도 확인했으며 AIM0150E/boundary violation/Error!!!가 없었다. 과거 7.3의 READVB 오류는 이 7.4 조합에서 재현되지 않았지만 수정 원인은 별도로 분석하지 않았다.

## 실행 중 확인한 OS별 보완

- VOS3 TSAM COBOL 전처리에는 `OFCBPPH_HOME`이 필요했다. 현재 호스트의 실행파일을 확인한 후 신규 VOS 환경에 `export OFCBPPH_HOME=$HOME/ofcbpph/install` 및 해당 bin/lib 경로를 추가하고 `make vos`를 재실행해 성공했다. 다른 호스트에서는 실제 설치 경로를 먼저 확인한다.
- MSP/XSP aimdtssv는 MIN=1이면서 NRDY였고 로그에 DTS DIR=(NONE)의 stat 실패가 있었다. 신규 환경마다 `$OPENFRAME_HOME/aim/dts`를 만들고 `aimdtssv.DTS.DIR`을 설정/import한 뒤 해당 서버를 기동하여 RDY를 확인했다. 공용 AIM 서버는 건드리지 않는다. DTS 이벤트/온라인 트랜잭션은 별도 검증 항목이다.
- XSP의 WRITEVBX2/READVBX2는 마지막 qualifier가 9자로 `tjesmgr r`의 데이터셋명 검사에서 거부됐다. 원본 멤버는 보존하고 동일 바이트를 `TSAM.TEST.WVBX2`, `TSAM.TEST.RVBX2`라는 8자 이하 별칭으로 복사하여 제출했다. 앞서 통과한 8개 JOB은 다시 실행하지 않고 VB2만 이어서 실행했다.
- XSP IEFBR14를 직접 실행하면 RC 0이지만 이번 Batch 래퍼의 최소 JOB/STEP은 10이다. SYSMSG의 `step done with RC=10, STATUS=R`와 정상 종료를 함께 확인했다. MSP 최소 JOB의 0과 MSP TSAM의 10도 구분한다.
- PODD는 실제 PTY의 동일 tjesmgr 콘솔에서 PSJOB 이후 실행했다. VIEWER를 일시적으로 `/usr/bin/cat &FILEPATH`로 바꾸고 finally에서 원래 `/usr/bin/vi -w&ROWCOUNT -R &FILEPATH`로 복원했다. 모든 JOB 완료 후 ofconfig를 재조회해 복원을 확인했다.

## 검증 범위의 한계

MSP/XSP의 aimapsvr/OIVPMQN과 VOS3의 OSDBASE_A/OSDSCHD_B/OSDCLI_B/OSDMPP_B/OSDBMP_B는 MIN=0인 미기동 온라인 영역이다. AIM/OSD 온라인 트랜잭션, NDB 애플리케이션 트랜잭션 및 VOS3 보조 XDM JDBC 도구 초기화는 미검증이다. OSD 라이선스·설정 import와 NDB 초기화 성공을 OSD 영역/XDM 전체 기능 성공으로 기록하지 않는다. 현재 xdminit는 별도의 JDBC properties와 명령/tablespace 인자가 필요하므로 일반 dbconn.conf만으로 실행 가능하다고 가정하지 않는다.

증거: source-commit.log, reserved-keys.log, configure*.log, release*.log, build-install-final.log, precomp-*.log, build-oflicgen-isolated.log, license-*.log, link-check.json, db-precheck.log, *init.log, tmadmin-final.log, dataset-test.log, jobs-results.json, spool-*.json, db-final-check.json, viewer-final-query.log, final-results.json. MSP의 environment-plan.json에 세 환경의 예약값을 보존했다.
