# OpenFrame 7.4 설치 차이와 MVS/Tibero 실측

## 적용할 차이

- 프로필 version/os/rdb/rdb_connect_string으로 선택한다. 설치·환경 충돌 확인의 공통 절차는 setup.md를 따른다. Oracle은 7.4에서 선택 가능하지만 아래 기록은 Tibero 실측이다.
- 신규 설치 경로가 프로필 env_path에 이미 지정되어 있으면 그 경로를 검증해 사용한다. 소스는 통합 저장소 클론 자체가 SOURCE_BASE다.
- Tmax/TCache/client 배포본과 실제 링크 구조를 확인한다. 전용 TCache에는 log 디렉터리도 생성한다. shared package 라이브러리를 참고하더라도 테스트 인스턴스의 로그/캐시 설정은 독립 경로를 사용한다.
- ipcs의 16진수와 설정의 10진수를 `int(value, 0)` 등으로 정규화하고 현재·중지 환경의 키를 비교한다. Tmax의 연속 키 범위도 포함한다. 충돌 시 기존 IPC나 서비스를 제거하지 말고 신규 환경의 키를 변경한다.
- 현재 pdsgen 구문은 `pdsgen <dsname> <volser>`였다. 도구 help를 확인하고 사용한다.
- dsmigin -I는 신규 데이터셋을 할당하므로 dscreate한 같은 이름에 그대로 import하면 이미 카탈로그됨 오류가 난다. 이번 작업의 빈 테스트 데이터셋만 삭제해 제거를 확인하고 import 시 FB/LRECL/volume을 지정했다. dsmigout -X로 읽어 원본과 byte 비교했다.
- TSAM create.sh 전에 make copybook을 수행한다. 테스트 스크립트 RC만으로 성공 판정하지 않고 모든 DEFINE/LIBGEN 결과를 확인한다.
- HiDB si_create.sh 전에 `ims/src/hidb/test/copybook/`의 필요한 계층을 신규 설치 hidb/copybook에 배치한다.
- 7.4 기본 NO_INDEX_TABLE=YES에서 si_create.sh는 EXHIDAM2/EXHIDASI2 자료를 사용한다(내부 DBD 이름은 EXHIDAM). 이 모드는 별도 EXHIIX 테이블을 만들지 않는다. reset은 실제 테이블을 가진 EXHIDAM만 truncate한 후 -L/-I/-SI를 수행한다. 7.3의 모든 인덱스 DBD truncate를 그대로 적용하면 존재하지 않는 테이블 오류가 난다. NO_INDEX_TABLE=NO에서는 실제 생성된 테이블과 기존 reset 절차를 확인한다.
- SPOOL viewer 임시 변경이 필요하면 common-ofconfig-manage에 따라 원본 확보·현재값 확인·최소 변경·재조회·복원을 수행한다. VIEWER=/usr/bin/cat &FILEPATH로도 같은 PTY에서 PSJOB 후 PODD로 실제 SYSOUT을 읽을 수 있다.

## 2026-10-01 실측

RHEL 9/GCC 11, OpenFrame rb_74 `b5ce2ef69837a1af325d332cfa7f3a8593d95bdc`, MVS, Tibero 7 FS02 PS05, Tmax5.0SP2Fix4, TCache 2.4. 추가 OS 패키지 설치와 제품 코드 수정 없이 수행했다.

환경 파일은 `$HOME/env_of74_mvs_tb`, 소스는 `$HOME/ofsrc/of74mvs`, 설치는 `$HOME/ofbin/of74mvs`, 로그는 `$OPENFRAME_HOME/validation`이다. 지정된 소스·설치 경로는 시작 시 존재하지 않았다. 지정 DB 계정의 객체 수 0개를 확인한 뒤 초기화했다.

- Base/Batch/TACF/HiDB/OSI 빌드·설치 성공. OSC는 TDL_RTLD_GLOBAL 미정의로 실패한 뒤 사용자 지시로 제외했다. OSC 실행과 OSI 트랜잭션 실행은 검증하지 않았다.
- TSAM, Tibero/TBOCI/TBODBC 드라이버의 make precomp 성공. Tibero 7 버전 설정 보완 후 핵심 모듈 재빌드 성공.
- baseinit/batchinit/tacfinit/hidbinit, 설정 import, 오류코드 적재, 네 볼륨과 PDS, TSO map/INITPROC, tjesinit 및 tjesmgr boot 성공.
- HiDB HIDB_SAVE_LARGE_AREA VALID, 최종 비유효 DB 객체 0개.
- 공통/TJES/HiDB 서버 17개 모두 RDY. 실제 BATCH_OS_TYPE=MVS, DATASET_SHMKEY=0x825 확인.
- Tmax TPORTNO=9520, RACPORT=9570, DOMAIN/NODE SHMKEY=99000/99010, 전용 TCache=0x72025. 이 값들은 이번 실행에서 확인한 값이며 다른 환경의 기본값으로 사용하지 않는다. 최초 98010은 기존 0x17eda와 충돌해 변경했다.
- TEST.OF74.DATA 생성, import/export 1레코드 80바이트 byte 비교, 삭제 후 Total 0 entries 확인.
- OF74SMK JOB00001: Done(R00000), SMOKE R0000, 동일 PTY PODD JESMSG 확인.
- TSAM WRITE/KREAD/EREAD/PATH/KAIX/EAIX/WRITEVB/READVB/WRITEVB2/READVB2, JOB00002~JOB00011: 모두 Done(R00000), STEP00 R0000. 모든 SYSOUT을 동일 PTY의 PSJOB/PODD로 읽고 오류 부재를 확인했다.
- 실제 TEST_CASE1/2/3/4 레코드 수 100/200/100/100, TEST_CASE1_VB/TEST_CASE2_VB 각 3. VB 읽기의 실제 문자열과 길이를 출력 DD에서 확인했다.
- HiDB -L/-Q/-I/-D/-R 후 EXHIDAM reset, -L/-I/-SI 총 8회 성공. RC뿐 아니라 각 hidam_test_*() success. 및 Error!!! 부재로 판정했다. 인덱스 테이블 truncate의 최초 실패는 NO_INDEX_TABLE=YES 동작에 맞춰 reset 범위를 조정했다.
- SPOOL 검토 후 VIEWER가 원래 `/usr/bin/vi -w&ROWCOUNT -R &FILEPATH`로 복원된 것을 재조회했다.

증거: build-install-first.log, build-install.log(OSC 실패 포함), build-*-final.log, build-osi.log, build-selected-final.log, precomp-*.log, *init.log, import-*.log, tmboot-*.log, tmadmin-final.log, dataset-test-retry.log, tsam-{create-retry,build,results,spool-review,db-counts}.*, hidb-{si-create-retry,test,retest,reset}*.log. JOB SPOOL과 JCL은 신규 설치에 보존했다. Oracle 및 MVS 이외 OS는 이번 실행에서 미검증이다.
