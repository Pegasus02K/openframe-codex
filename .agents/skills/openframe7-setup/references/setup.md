# 설치 상세 절차

## 디렉터리와 바이너리

OPENFRAME_HOME 아래 공통 디렉터리와 선택한 OS의 디렉터리를 만든다. `mkdir -p`가 `tsam`, `hidb`, `ndb` 부모 디렉터리도 함께 생성한다. `schema`는 HiDB의 hdgensch 실행 전에 필요하다.

```sh
mkdir -p "$OPENFRAME_HOME"/{lib,bin,data,config,scripts,log/sys,log/data,log/cmd,license,shared/SMF,volume_DEFVOL,volume_100000,volume_200000,util,temp,spool,outputq,spbackup,schema,tsam/lib,tsam/temp,tsam/copybook}
# MVS에서만 실행
mkdir -p "$OPENFRAME_HOME"/{hidb/lib,hidb/temp,hidb/copybook}
# MSP, XSP, VOS3에서만 실행
mkdir -p "$OPENFRAME_HOME"/{ndb/lib,ndb/temp,ndb/copybook}
```

TMAXDIR와 TCACHE_HOME을 별도 새 설치 아래 만들고 배포 tar의 내부 경로를 먼저 확인한다. Tmax를 TMAXDIR에, TCache를 TCACHE_HOME와 TMAXDIR 양쪽에 푼다. Tmax license.dat을 TMAXDIR/license에 복사한다.

압축 해제나 appbin 이동 전에 새 환경 파일을 적용하고, `realpath`로 해석한 SOURCE_BASE/OPENFRAME_HOME/TMAXDIR/TCACHE_HOME이 승인된 새 경로와 일치하는지 확인한다. 예제 환경에서 상속된 변수를 대상으로 설치하지 않는다. 새 설치 준비와 기존 경로의 변경을 같은 단계로 섞지 않는다.

`appbin -> appbin64`, `lib -> lib64` 링크를 확인한다. 실측 Tmax tar에는 appbin 실디렉터리, TCache tar에는 appbin64가 있었다. appbin을 보존 이름으로 옮기고 파일을 appbin64에 합친 뒤 링크를 만든다. 기존 파일을 무조건 덮어쓰거나 삭제하지 않는다. 실측 TCache tar는 bin64/lib64이므로 환경이 bin/lib를 쓰면 링크를 만들거나 환경 경로를 맞춘다. 로그 디렉터리도 새 설치에 둔다.

환경은 OPENFRAME_NODENAME, OPENFRAME_INSTANCE_NAME, SOURCE_BASE, OPENFRAME_HOME, OPENFRAME_BIT, TMAXDIR, TMAX_HOST_ADDR/PORT, FDLFILE, TB_HOME, TB_SID/USERID, ODBCINI/ODBCSYSINI, OFCOB_HOME/COBPARSER_HOME, LLVM_HOME, PROSORT_HOME, TCACHE_HOME/TCACHECONF, PATH/LD_LIBRARY_PATH 등을 예제와 비교한다. 실제 라이브러리와 라이선스가 없는 경로를 복제하지 않는다.

base/src/server/resource/oframe.m.local의 HOSTNAME, 모든 Tmax 경로, TPORTNO/RACPORT, DOMAIN/NODE SHMKEY를 새 환경에 맞춘다. 포트·키는 매번 충돌 확인 후 선택한다. 예제의 DOMAINID와 NODE명도 실제 환경에 맞는지 확인한다.

이후 openframe7-build의 제품 빌드·설치와 라이선스 생성을 완료한다. 최신 소스는 서버를 OPENFRAME_HOME/server에 설치하므로 Tmax APPDIR에서 실행 파일을 찾는지 확인한다. 실측 설치는 APPDIR의 개별 서버 이름을 OPENFRAME_HOME/server의 산출물로 symlink했다. 예제 환경은 server 디렉터리 자체를 tmax/appbin으로 연결한다. 두 형태를 섞어 파일을 덮어쓰지 말고 선택한 형태에서 ldd와 서버 경로를 검증한다.

## DB 설정과 초기화

설치 config/*.sample을 대응하는 활성 파일로 복사한다. 실측 dbconn.conf의 TSAM/SYS1/RDBII/HIDB 및 XSP의 AIM_CLIENT/NDB_CLIENT/NDB_BACKUP 등 사용 섹션에 DATABASE/USERNAME/ENPASSWD를 적용한다. `enpasswd <username> <password>`의 출력은 캡처해서 ENPASSWD에 기록하고 화면이나 로그에 남기지 않는다. DB 계정·암호가 서로 다른 섹션은 각 실제 접속 계약을 따른다.

MVS에서는 `$OPENFRAME_HOME/config/openframe_hidb.conf`를 import하기 전에 `hidb / DEFAULT_USER / ENPASSWD` 값을 `enpasswd ROOT SYS1`로 생성한 암호화 값으로 수정한다. 도구의 안내 문구가 아닌 실제 암호화 결과만 해당 키에 반영하고, 결과값은 화면이나 로그에 남기지 않는다. 수정한 파일을 HiDB 설정 import에 사용한다. 이미 설치된 테스트 환경을 보완할 때는 파일뿐 아니라 `ofconfig update -n "$OPENFRAME_NODENAME" -s hidb -sec DEFAULT_USER -k ENPASSWD -v "$encrypted_value"`로 해당 키만 갱신하고, 재조회 값을 출력 없이 비교한다. 이 값은 설치에 필요한 영구 설정으로 유지한다.

pfmtcache_batch.cfg.sample을 pfmtcache.cfg로 복사하고 SHMKEY를 독립 값으로 바꾼다. 기본 pfmtcache.cfg.sample은 Batch용 캐시 전체를 포함하지 않을 수 있다.

빈 스키마임을 확인한 뒤:

```sh
baseinit create -t DEFVOL
batchinit create -t DEFVOL
tacfinit create -t DEFVOL
# MVS
hidbinit create -t DEFVOL
# MSP/XSP
ndbinit create -t DEFVOL
aiminit create -st DEFVOL -lt DEFVOL
# VOS3
ndbinit create -t DEFVOL
```

OS에 해당하는 명령만 실행한다. **실측 MVS에서는 hidbinit 전에 추가 준비가 필요하다.** 공통 baseinit/batchinit/tacfinit을 완료한 뒤 `pfmtcacheadmin -c`, base/batch/tacf/hidb 설정 import, `mkdir -p "$OPENFRAME_HOME/hidb/lib"`를 먼저 수행하고 hidbinit을 실행한다. hidbinit이 HIDB_OBJECT_DIR를 읽고 하위 dlilibs를 만들기 때문이다. TCache를 이미 생성했다면 기동 단계에서 다시 초기화하지 않는다. MVS에서는 ims/scripts/init.sql을 해당 DB 계정으로 실행하고 생성 PSM의 유효 상태를 조회한다. SQL 클라이언트 종료 코드만으로 성공을 판단하지 않는다. 실측 Tibero tbsql은 `-s /nolog`를 잘못된 CONNECT로 처리했으므로 `tbsql -s`에 CONNECT/SQL을 stdin으로 공급했다. CONNECT에 암호가 들어가므로 echo/spool을 사용하지 않는다.

설정 import 전에 독립 SHMKEY의 신규 TCache를 `pfmtcacheadmin -c`로 한 번 생성한다. 앞 단계에서 생성했다면 반복하지 않는다. 활성 openframe_base.conf의 BATCH_OS_TYPE을 실제 OS로 맞춘 뒤 `ofconfig import -f <file> -n "$OPENFRAME_NODENAME"`으로 base/batch/tacf를 가져온다. MVS는 hidb, MSP/XSP는 ndb/aim, VOS3는 ndb를 추가한다. NDB의 OS 설정은 MSP/XSP에 맞춘다. 원래 예제 NODE1을 고정하지 않는다. Tmax 기동 후 `ofconfig list -n "$OPENFRAME_NODENAME" -k BATCH_OS_TYPE -l`로 확인한다.

XSP 실측에서는 TCache 생성 전 `aiminit`이 DB 객체 생성을 마친 뒤 설정 캐시 오류를 출력했다. RC 0만으로 오류가 없었다고 판단하지 않는다. 초기화 로그·DB 객체와 초기화 후 서버 상태를 구분해 확인하고, 이미 생성된 스키마에 `aiminit create`를 무조건 다시 실행하지 않는다. TCache 생성과 설정 import를 마친 뒤에도 오류가 남는지 확인한다.

`oferror insert -p <file>`로 공통 errcode_base.msg, errcode_batch.msg, errcode_TACF.msg를 적재한다. MVS는 errcode_imsx.msg, MSP/XSP는 errcode_ndb7.msg/errcode_aim.msg, VOS3는 errcode_ndb7.msg를 추가한다. 실제 설치 파일 경로를 찾고 각각 결과를 확인한다.

## 기동과 볼륨/PDS

```sh
# 초기화 단계에서 아직 생성하지 않은 신규 TCache만 생성한다.
# pfmtcacheadmin -c
tmboot
volmgr define device -dn 0001 -dt 3380 -ms 2048 -N
volmgr define group -dn 0001 -dg SYSDA -N
volmgr define device -dn 0000 -dt FFFF -ms 2048 -N
volmgr define volume -v DEFVOL -dn 0001 -i -N
volmgr define volume -v 100000 -dn 0001 -i -N
volmgr define volume -v 200000 -dn 0001 -i -N
volmgr define volume -v VSPOOL -p "$OPENFRAME_HOME/spool" -dn 0000 -s
```

볼륨 디렉터리와 DB 테이블스페이스가 먼저 준비되어야 한다. 실측 기본 디렉터리는 volume_DEFVOL/volume_100000/volume_200000이다. 설정에 따른 실제 매핑을 확인하며 이미 있는 볼륨을 초기화하지 않는다.

pdsgen으로 DEFVOL에 SYS1.JCLLIB, SYS1.COBLIB, SYS1.USERLIB, SYS1.PROCLIB, SYS1.MACLIB을 만든다. MVS는 IMS.DBDLIB, IMS.PSBLIB, IMS.ACBLIB을 추가한다.

`tjesinit`은 확인 프롬프트를 읽는다. 사용자가 승인한 신규 초기화에서는 `printf 'Y\n' | tjesinit`처럼 입력을 별도로 전달해 실행 중인 셸 스크립트의 나머지 줄을 소비하지 않게 한다. 완료 메시지와 후속 명령 실행도 검증한다. 이후 새 인스턴스만 `tmdown`, `tmboot`하고 `tjesmgr boot`를 확인한다. 기존 환경의 tmadmin 상태를 읽어 미기동 서버 비교 기준을 남긴다. 재기동 직전 TMAXDIR와 포트가 새 인스턴스인지 검증한다.

MVS 또는 XSP에서 TSO가 예제처럼 RDY여야 하거나 `obmtsmgr` 로그가 DEFAULT_PROC/맵 부재를 나타내면 SYS1.TSOMAP PDS를 생성하고 scripts/INITPROC를 SYS1.PROCLIB/INITPROC로 복사한다. 기존 PDS/멤버를 덮어쓰지 않는다. scripts 디렉터리에서 아래처럼 실행한다.

```sh
cd "$OPENFRAME_HOME/scripts"
for name in INIT LOGIN LOGOFF NEWPASS FIMPMAP FEXPMAP; do
  tsomapgen -m IPF -l SYS1.TSOMAP "$name"
  test -s "$OPENFRAME_HOME/volume_DEFVOL/SYS1.TSOMAP/$name.map" || exit 1
done
tmboot -s obmtsmgr
```

`-m IPF`를 생략하거나 입력을 절대 경로로 주면 파싱 성공·RC 0이어도 map 파일이 생성되지 않을 수 있다. 실제 파일과 서버 RDY를 검증한다.

OPENFRAME_HOME/scripts의 .sample을 활성 파일로 복사해 환경에 맞춘다. TSAM 테스트에는 tsam/{copybook,lib,temp} 등 설정이 가리키는 실제 디렉터리와 tsam_compile.sh 실행 경로도 확인한다.

## 실제 성공 테스트

1. 충돌 없는 테스트 dataset을 dscreate로 만들고 dslist/listcat으로 확인한다. spfedit의 지원 모드로 내용을 확인하고 dsdelete로 삭제한 뒤 카탈로그 제거를 확인한다. 사용자 보존 요청이 있으면 영구 보존용 결과 dataset과 삭제 검증용 dataset을 구분한다.
2. jcl-write-run 계열 및 batch-job-run에 따라 IEFBR14를 제출한다. JOB ID, 최종 상태, 해당 OS/버전의 정상 STEP RC, JESMSG 등 출력 DD를 확인한다. XSP 실측 IEFBR14는 소스가 성공 시 10을 반환하므로 0을 일괄 강제하지 않는다.
3. base/src/ds/tsam/test/create.sh을 읽고 TEST.* 삭제/생성 범위를 확인한다. 신규 스키마와 테스트 경로에서 실행하고 해당 OS의 make target으로 컴파일한다. 실측 target은 mvs/msp/xsp/vos이다.
4. 쓰기→읽기 순서와 AIX/PATH, VB 테스트의 입력 의존성을 JCL에서 확인한다. 각각 JOB/STEP/SPOOL와 실제 레코드·카탈로그를 확인한다. DONE만으로 성공 판정하지 않는다. XSP의 OSAMFRUN에서는 애플리케이션 RC 0이 JOB/STEP RC 10으로 표시될 수 있다. 실제 SPOOL의 `Execution AP(...) done - RC(0), STATUS(R)`와 정상 종료, 오류 부재를 함께 확인한다. 상태/RC를 기대값으로 하드코딩해 결과 파일에 쓰지 말고 실제 출력에서 추출한다. 실패 시 자동 연속 제출을 멈추고 미실행 항목을 따로 기록한다.
5. 환경 파일, 소스, 설치본, 로그와 JCL/SPOOL을 보존하고 실패한 테스트를 구분해 보고한다.

PTY UI가 TERM=dumb에서 보이지 않으면 해당 세션에 TERM=xterm을 지정한다. spfedit은 `-b`로 실제 레코드를 확인하고 F3으로 종료한다.

XSP `make xsp`는 `TSAM.TEST.WRITEX`, `KREADX`, `EREADX`, `PATHX`, `KAIXX`, `EAIXX`, `WRITEVBX`, `READVBX`, `WRITEVBX2`, `READVBX2` 멤버를 배치한다(모두 `TSAM.TEST.` 접두사). MVS 멤버명을 그대로 제출하지 않는다. `create.sh`는 초기 DELETE의 미존재 오류를 허용하므로 스크립트 종료 코드 외에 각 DEFINE/LIBGEN 결과와 카탈로그도 확인한다.

### MVS HiDB 정상 동작 테스트

MVS에서는 HiDB 기본 사용자 ENPASSWD 설정, HiDB 초기화 및 PSM 적재, 서버 기동을 완료한 뒤 다음을 실행한다. 환경 파일을 적용한 셸에서 `$SOURCE_BASE`를 확인하고 `ims/src/hidb/test/si_create.sh`와 `si_truncate.sh`의 대상을 먼저 읽는다. `si_truncate.sh`는 EXHIDAM 및 EXHIIX1~5 테스트 데이터를 초기화한다.

`-L → -Q → -I → -D → -R` 이후 `si_truncate.sh`로 초기화하고 `-L → -I → -SI`를 실행한다. `-SI`의 기대 데이터는 앞선 삭제·갱신 테스트 직후 상태와 다르므로 초기화와 재적재를 생략하지 않는다.

다음 Bash 예시는 명령별 종료 코드와 성공 메시지를 함께 검사한다. 재실행 로그는 별도 디렉터리에 보존한다.

```bash
cd "$SOURCE_BASE/ims/src/hidb/test" || exit 1
umask 077
mkdir -p "$OPENFRAME_HOME/validation" || exit 1
hidb_log_dir=$(mktemp -d "$OPENFRAME_HOME/validation/hidb-run.XXXXXX") || exit 1
run_hidam() {
    local opt="$1" marker log_file
    shift
    log_file=$(mktemp "$hidb_log_dir/test-${opt#-}.XXXXXX.log") || return 1
    ./test_hidam "$opt" > "$log_file" 2>&1 || return 1
    if grep -Fq 'Error!!!' "$log_file"; then return 1; fi
    for marker in "$@"; do
        grep -Fq "$marker" "$log_file" || return 1
    done
}
make > "$hidb_log_dir/make.log" 2>&1 || exit 1
bash -e si_create.sh > "$hidb_log_dir/si-create.log" 2>&1 || exit 1
run_hidam -L 'hidam_test_initial_load() success.' || exit 1
run_hidam -Q 'hidam_test_get_unique() success.' 'hidam_test_get_next() success.' 'hidam_test_get_nextp() success.' || exit 1
run_hidam -I 'hidam_test_insert() success.' || exit 1
run_hidam -D 'hidam_test_delete() success.' || exit 1
run_hidam -R 'hidam_test_replace() success.' || exit 1
bash -e si_truncate.sh > "$hidb_log_dir/si-truncate.log" 2>&1 || exit 1
run_hidam -L 'hidam_test_initial_load() success.' || exit 1
run_hidam -I 'hidam_test_insert() success.' || exit 1
run_hidam -SI 'hidam_test_secondary_index() success.' || exit 1
```

실측 test_hidam은 내부 오류 처리 뒤 disconnect 결과로 반환값이 덮여 실패해도 RC 0일 수 있다. 따라서 RC와 각 `hidam_test_*() success.` 메시지를 모두 확인한다. `DLI function failed - II/GE/GB/DA`는 테스트가 기대한 중복·조회 종료·잘못된 갱신 케이스일 수 있으므로 문자열만으로 실패 판정하지 않는다. `Error!!!`, 기대값 불일치 또는 최종 성공 메시지 부재는 실패로 판정하고 해당 출력과 소스의 기대 상태를 비교한다.

비정상 결과가 있으면 실패한 명령과 원인을 기록하고 해결한 뒤 필요한 준비 단계부터 재검증한다. 명령별 결과를 설치 검증 보고에 포함하고 실제 실행하지 않은 항목은 미검증으로 표시한다.
