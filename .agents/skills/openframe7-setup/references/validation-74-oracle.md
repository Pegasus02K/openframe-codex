# OpenFrame 7.4 MVS Oracle 설치 차이와 실측

Oracle 프로필에서 [공통 설치](setup.md)와 [Oracle 빌드](../../openframe7-build/references/build-74-oracle.md)에 추가하여 적용한다. local YAML의 속성을 그대로 사용하고, 사용자가 신규 이름 선택을 맡기면 지정 env_path를 예제로 읽어 별도 환경 파일을 만든다. YAML이나 예제 환경 파일은 변경하지 않는다.

## 설치 시 확인할 차이

- unixODBC DSN의 ServerName과 실제 SQL*Plus 접속 대상이 일치하는지 확인한다. rdb_connect_string에 sqlplus 명령 접두사가 있으면 데이터로 파싱하고 중복 실행하지 않는다. 계정·암호는 평문 파일/URL/명령 로그에 넣지 않는다.
- 신규 스키마에만 baseinit/batchinit/tacfinit/hidbinit을 실행한다. HiDB scripts/init.sql 적재 후 HIDB_SAVE_LARGE_AREA의 VALID 및 user_objects의 비유효 객체 수를 확인한다.
- DATASET_SHMKEY는 활성 IPC와 중지 환경의 설정까지 대조한다. `||` 파일의 실제 VALUE 칸을 변경하고 import 뒤 캐시 적용도 재검증한다. 이번 재import는 DB 값과 캐시 값이 달라 신규 인스턴스를 중지하고 ofconfig load 후 재기동했다. 정상 update 뒤에는 불필요한 load를 추가하지 않는다.
- Oracle JOB에는 tjclrun/SYSLIB/LIB_PATH의 `${ORACLE_HOME}/lib`가 필요하다. 설치 파일과 실행 설정을 함께 보완하고 첫 실패 JOB과 재실행 결과를 구분한다.
- TCache tar의 appbin64/TPFMAGENT를 실제 Tmax APPDIR에서 실행할 수 있어야 한다. 새 설치에만 경로를 맞춘다.
- TSAM source/설치 scripts의 Oracle cfg 모두 GCC와 Oracle header 경로를 검증한다. DEFINE/LIBGEN 및 copybook 배치 후 COBOL/JCL을 준비한다.
- MVS HiDB NO_INDEX_TABLE=YES이면 EXHIDAM2/EXHIDASI2와 실제 생성 테이블을 사용한다. 테스트 순서는 -L/-Q/-I/-D/-R, EXHIDAM reset, -L/-I/-SI다. RC와 각 성공 메시지 및 Error!!! 부재를 함께 검사한다.
- SPOOL은 동일 PTY에서 PSJOB 후 PODD를 수행한다. 이번 설치에서는 대문자 `DI=`와 실제 SPOOL LIST의 index로 SYSOUT을 읽었다. lowercase dn=으로 출력이 보이지 않으면 성공으로 간주하지 않고 실제 DD index를 확인한다. viewer 임시 변경은 common-ofconfig-manage로 원본을 확보하고 복원·재조회한다.

## 2026-10-01 검증 결과

RHEL 9/GCC 11, OpenFrame 7.4 rc1 `081cbe845f59ea5459e9e45dd04c06192d349a49`, MVS, Oracle client Pro*C 23.26.1, Tmax 5.0 SP2 Fix4, TCache 2.4. 추가 OS 패키지 설치와 제품 소스 수정 없이 수행했다.

리모트 환경은 `$HOME/env_of74_mvs_ora_codex_20261001`, 소스는 `$HOME/of74/ofsrc/of74mvsora_codex_20261001`, 설치는 `$HOME/of74/ofbin/of74mvsora_codex_20261001`이다. 지정 프로필의 Oracle 스키마가 객체 0개이고 DEFVOL/100000/200000 테이블스페이스가 존재함을 먼저 확인했다. 연결은 YAML의 지정값을 사용했으며 여기에는 인증값을 기록하지 않는다.

- Oracle 전처리 5개 디렉터리 성공. 최초 header 경로 누락을 sdk/include로 보완해 전처리와 컴파일을 재검증했다.
- Base/Batch/TACF/HiDB/OSI 빌드·설치 성공. 최초 전체 MVS 빌드의 OSC는 TDL_RTLD_GLOBAL 미정의로 실패했고 사용자 승인으로 제외했다. 최종 선택 루트 make install 성공.
- DB 초기화, 설정 import/load, HiDB PSM 적재, 공통 오류코드, 볼륨/PDS/TSO map, tjesinit 및 tjesmgr boot 완료.
- Tmax/TCache/Dataset 키는 각각 118000/118010, 0x74025, 0x845. 포트는 10220/10270. 기존 설정 72개와 활성 IPC/포트를 대조했다. 이번 실측 값이므로 다른 환경에서 그대로 사용하지 않는다.
- 공통/TJES/HiDB 17개 서버 RDY. 실제 BATCH_OS_TYPE=MVS와 DATASET_SHMKEY=0x845 확인.
- TEST.OF74.ORA.DATA의 생성·카탈로그, import/export 1레코드 80바이트 byte 비교, 삭제 후 Total 0 entries 확인.
- ORASMK01 JOB00001: Done(R00000), SMOKE R0000, JESMSG 확인.
- TSAM 최초 WRITE JOB00002는 libclntsh 탐색 실패로 Error(R00127), STEP R0127. Oracle JOB LIB_PATH 보완 후 WRITE/KREAD/EREAD/PATH/KAIX/EAIX/WRITEVB/READVB/WRITEVB2/READVB2 JOB00003~JOB00012 모두 Done(R00000), STEP00 R0000. 동일 PTY의 PSJOB/PODD로 모든 SYSOUT 확인.
- 실제 TEST_CASE1/2/3/4 레코드 수 100/200/100/100, TEST_CASE1_VB/TEST_CASE2_VB 각 3. VB 읽기 SYSOUT에서 A 문자열 길이 8/10/12와 AAAAAAAAXXXX/BBBBAAAAAAAA/CCCCAAAAAAXX 확인.
- HiDB -L/-Q/-I/-D/-R, EXHIDAM reset, -L/-I/-SI 총 8회 성공. 각 hidam_test_*() success. 메시지와 Error!!! 부재 확인.
- 최종 HIDB_SAVE_LARGE_AREA VALID, 비유효 DB 객체 0개. SPOOL viewer를 원래 값으로 복원하고 재조회했다.

초기/재기동 중 ORA-12516이 관찰됐고 일부 서버 초기화가 실패했다. 단일 SQL*Plus/ODBC는 통과했으며 설정 캐시·Dataset 키·TCache 경로·설치 자원 보완과 최종 개별 서버 재기동 후 실행 검증을 완료했다. ORA-12516의 DB 서버 측 원인은 확정하지 않았다. 기존 서비스나 DB 서버 설정은 변경하지 않았다.

증거는 신규 설치의 validation 아래 build-install*.log, build-selected-final.log/rc, precomp*.log, *init.log, import*.log, config-load.log, collision-check.log, tmadmin*.log, dataset-test.log, tsam-{create,build,result-*}.log, spool-review.typescript, hidb-*.log, viewer-restored.log와 실제 JOB SPOOL/JCL이다. OSC 실행, OSI 트랜잭션 및 MVS 이외 OS는 미검증이다.
