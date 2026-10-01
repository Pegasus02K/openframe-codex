# MSP 실제 검증 기록

2026-09-17 설치 후 2026-09-23 최종 검증. RHEL 9 / GCC 11 / Tibero 7 / Tmax 5.0 SP2 Fix4 / TCache 2.4 / OFCOBOL.
기존 MSP/MVS/XSP 환경을 보존하고 빈 별도 스키마에 설치했다. 추가 OS 패키지 설치나 제품 소스 수정은 없었다.

## 판정과 경로

Base/Batch/TACF/NDB/AIM 빌드, DB 초기화, 설정 import, 볼륨/PDS, Tmax/TJES 기동, 데이터셋과 최소 JOB은 성공했다. TSAM 회귀는 READVB에서 실패했으므로 설치 성공과 전체 TSAM 성공을 구분한다.

- 환경 파일: `$HOME/env_openframe_msp_test`
- 소스: `$HOME/ofsrc_msp_test`
- 설치: `$HOME/openframe_msp_test`
- 로그: `$OPENFRAME_HOME/validation`
- JCL: `$OPENFRAME_HOME/volume_DEFVOL/SYS1.JCLLIB`
- SPOOL: `$OPENFRAME_HOME/spool/JOB00001`~`JOB00010`

Tmax 포트 9560/RACPORT 9610, DOMAIN/NODE SHMKEY 97000/97010, TCache 0x70065는 이 실행에서 기존 환경과 `ss`/`ipcs`로 충돌을 확인한 값이다. 재사용 시 다시 확인한다. DB·Git 인증값은 환경 파일, 로그, 스킬에 저장하지 않았다.

## 빌드

| 제품 | 브랜치 | 커밋 | 결과 |
|---|---|---|---|
| Base | rb_73 | c1638ab979a1edf9b109ccbdba4a1282b904c009 | make install 성공 |
| Batch | rb_73_FS2 | 08e69ba96cf93e92c61de49faf60661c6813192c | make install 성공 |
| TACF | rb_73 | edc54855371858de9f51f7ea631aeae1a115cfbc | make install 성공 |
| NDB | rb_73 | d7d798065f1fe1fe4c77fcb7de238e7334c9004e | make install 성공 |
| AIM | rb_73 | f6fad3e9e72987b3e32752bf5a1fc82cb2519c4f | make install 성공 |
| dev | master | 496b5914327961ae5f56f82c36f7ed3172d84bd6 | oflicgen 빌드 성공 |

Base 첫 빌드는 `base/src/server/resource/oframe.m.local` 부재로 실패했다. XSP 실측 파일의 구조를 사용하되 신규 MSP 경로·포트·SHMKEY로 만든 뒤 재빌드했다. oflicgen 첫 빌드는 `dev/make/cflags.local` 부재로 실패했고 현재 `cflags.linux64`를 복사한 뒤 성공했다. Base/Batch/TACF/NDB/AIM/NDB2 라이선스 생성과 최종 `tmconfig_aim` 적용에 성공했다.

Base TSAM 및 Tibero 연결 모듈 4곳과 NDB common/dml/meta의 `make precomp`가 성공했다. Base → Batch → TACF → NDB → AIM 설치도 모두 성공했다. MSP 설정은 Base `JCL_SWITCH_MSP`, Batch `BATCH_OS_TYPE=MSP`, NDB/AIM `COBOL_COMPILER=OFCOBOL`, NDB `USE_DB_SELECT=AIM_DB`를 사용했다.

## 초기화와 기동

- DB 사전 확인에서 user_objects 0개, DEFVOL/100000/200000 테이블스페이스 존재를 확인했다.
- baseinit/batchinit/tacfinit 후 TCache 생성과 base/batch/tacf/ndb/aim 설정 import를 먼저 수행하고 ndbinit/aiminit을 실행했다. XSP 실측의 aiminit 설정 캐시 오류는 재현되지 않았다.
- 오류코드 5종, DEFVOL/100000/200000/VSPOOL, 시스템 PDS, tjesinit과 TJES boot가 성공했다.
- `BATCH_OS_TYPE`의 DEFAULT_VALUE와 VALUE 모두 MSP로 확인했다.
- obmtsmgr의 INITPROC 부재는 SYS1.TSOMAP, INITPROC와 6개 IPF 맵 생성 후 해소했다.
- 장기간 남은 테스트 인스턴스는 서비스가 RDY여도 `ofconfig` TPETIME과 `tjesmgr` 대기가 발생했다. 사용자가 재기동한 기존 MSP와 프로세스 경로·시작 시각을 구분한 뒤 테스트 인스턴스만 `tmdown -i -y`, `tmboot`, `tjesmgr boot`하여 복구했다.
- 최종 25개 항목 중 21개 RDY. `ofrsmlog`, `aimdtssv`, `aimapsvr`, `OIVPMQN`은 NRDY였고 같은 시점의 기존 MSP도 동일했다. 기능 성공으로 해석하지 않는다.

## 데이터셋과 최소 JOB

- TEST.MSP.DELETE를 생성·조회·삭제했고 `dslist`의 `Total 0 entries`로 삭제를 확인했다. 이 버전의 dslist는 0건에도 RC 0을 반환했다.
- TEST.MSP.DATA에 `OPENFRAME MSP INSTALLATION VERIFIED`를 적재하고 dsmigout 첫 레코드로 왕복 내용을 확인했다. FB 80 출력은 레코드 뒤에 패딩을 포함하므로 파일 전체 byte 비교는 사용하지 않았다.
- `PGM=IEFBR14`인 MSPCHK/JOB00001은 프로그램 파일이 없어 Error(A00016), STEP A0016이었다.
- MSP 설치 파일은 `$OPENFRAME_HOME/util/KDJBR14`이다. `PGM=KDJBR14`인 MSPCHK2/JOB00002는 Done(R00000), SMOKE R0000이었고 SYSMSG에서 프로그램 실행 RC 0을 확인했다.

## TSAM 결과

첫 `create.sh`는 설치본 `$OPENFRAME_HOME/scripts/tsam_tibero.cfg`의 오래된 GCC include 때문에 `TBR-9130`/`TBR-9108`로 실패했다. 소스에서 검증한 cfg를 설치본에 적용한 뒤 모든 DEFINE/LIBGEN이 성공했다. `make msp`도 성공했다.

| 작업 | JOB ID | 결과 |
|---|---|---|
| WRITE | JOB00003 | Done(R00010), STEP R0010, 애플리케이션 RC 0 |
| KREAD | JOB00004 | 동일 정상 종료 |
| EREAD | JOB00005 | 동일 정상 종료 |
| PATH | JOB00006 | 동일 정상 종료 |
| KAIX | JOB00007 | 동일 정상 종료 |
| EAIX | JOB00008 | 동일 정상 종료 |
| WRITEVB | JOB00009 | 동일 정상 종료 |
| READVB | JOB00010 | Error(A00050), STEP A0050, boundary violation |
| WRITEVB2 / READVB2 | 미제출 | READVB 실패 후 연속 실행 중단 |

MSP `make msp`는 OFCOBOL `--enable-XSP`를 사용하며 SPOOL에 `AIM OS TYPE : [XSP]`가 표시됐다. 환경 판정은 이 문자열이 아니라 확인된 `BATCH_OS_TYPE=MSP`를 기준으로 했다. 정상 작업은 SPOOL의 `Execution AP(...) done - RC(0), STATUS(R)`와 `OSAMFRUN finish - STATUS:R, RC:10`을 함께 확인했다.

READVB는 TC01에서 `TCOBFH: a boundary violation exists`와 `AIM0150E`를 출력했다. XSP에서 확인된 것과 같은 실제 실행 실패이며 해결·성공 재검증하지 않았다. 실패 시점 DB 레코드는 TEST_CASE1/2/3/4가 100/200/100/100, TEST_CASE1_VB 3, TEST_CASE2_VB 0이었다. user_objects 419개는 모두 VALID였다.

증거는 `$OPENFRAME_HOME/validation`에 보존했다. build/precomp/init/config/errcode/volume/TJES 로그, JOB별 submit·PSJOB, `tsam-results.tsv`, `tsam-db-counts.log`, `tmadmin-final.log`, `reference-tmadmin.log`, `build-commits.tsv`를 포함한다. 설치 성공과 READVB 회귀 실패 및 미실행 VB2를 구분해 재사용한다.
