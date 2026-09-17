# XSP 실제 검증 기록

2026-09-17, RHEL 9 / GCC 11 / Tibero 7 / Tmax 5.0 SP2 Fix4 / TCache 2.4 / OFCOBOL.
기존 예제 환경을 보존하고 빈 별도 스키마에 설치했다. 추가 OS 패키지 설치나 제품 소스 수정은 없었다.

## 판정과 경로

빌드·DB 초기화·Tmax/TJES 기동과 일반 배치 실행은 성공했다. 설치·기동 성공과 아래의 TSAM 회귀 테스트 실패를 구분한다. 사용자는 해당 테스트 오류를 설치 판정에서 별도로 취급하도록 요청했다. 이 요청을 다른 환경의 실패를 무시하는 일반 규칙으로 적용하지 않는다.

- 환경 파일: `$HOME/env_openframe_xsp_test`
- 소스: `$HOME/ofsrc_xsp_test`
- 설치: `$HOME/openframe_xsp_test`
- 로그: `$OPENFRAME_HOME/validation`
- JCL: `$OPENFRAME_HOME/volume_DEFVOL/SYS1.JCLLIB`
- SPOOL: `$OPENFRAME_HOME/spool/JOB00001`~`JOB00009`

Tmax 포트 9500/RACPORT 9550, DOMAIN/NODE SHMKEY 96000/96010, TCache 0x70055는 이번 환경에서 기존 HOME 환경 파일 및 ss/ipcs와 비교한 값이다. 재사용 시 충돌을 다시 확인한다. 개인 계정·비밀번호·Git 인증은 이 기록이나 공용 템플릿에 저장하지 않는다.

## 빌드

| 제품 | 브랜치 | 커밋 | 결과 |
|---|---|---|---|
| Base | rb_73 | 15c9cb3f4c0d05335864660a30e3047002eebd05 | make install 성공 |
| Batch | rb_73_FS2 | 08e69ba96cf93e92c61de49faf60661c6813192c | make install 성공 |
| TACF | rb_73 | edc54855371858de9f51f7ea631aeae1a115cfbc | make install 성공 |
| NDB | rb_73 | d7d798065f1fe1fe4c77fcb7de238e7334c9004e | make install 성공 |
| AIM | rb_73 | 56746a707c0cf1eaa31f5ddbb66919bc0992b38d | make install 성공 |
| dev | master | af9bb745f5d91303cff778dd3f145eba82e99578 | oflicgen 빌드 성공 |

Git 인증을 대화형으로 갱신한 뒤 한시적 credential cache로 clone/pull에 성공했다. 저장소 URL에는 인증 정보를 넣지 않았다. Base/Batch/TACF/AIM/NDB2 라이선스를 생성했고, 최종 `tmconfig_aim`을 `tmconfig`로 적용했다.

Base 루트 `make precomp`는 target 부재로 실패했다. 실제 TSAM 및 Tibero 연결 모듈 디렉터리의 전처리는 성공했다. NDB common/dml/meta 전처리도 성공했다. NDB rstd의 오래된 target은 없는 `ndb_rstd.pc`를 참조했으나 현재 빌드의 `ndb_rstd.c`는 일반 C 소스였다. 해당 target 자체를 성공으로 기록하지 않으며 실제 NDB 빌드 성공과 구분한다.

## 초기화와 기동

- DB 접속 성공, 초기 user_objects 0개 및 DEFVOL/100000/200000 테이블스페이스 존재 확인.
- baseinit/batchinit/tacfinit/ndbinit, aiminit의 객체 생성 및 초기 데이터 적재 완료.
- aiminit 종료 시 TCache 미생성에 따른 설정 캐시 오류가 출력됐다. 후속 TCache 생성·설정 import를 완료했으며 스키마 create를 반복하지 않았다.
- base/batch/tacf/ndb/aim 설정 import 및 오류코드 적재 성공. `BATCH_OS_TYPE`의 실제 VALUE는 XSP.
- DEFVOL/100000/200000/VSPOOL 및 시스템 PDS 생성, tjesinit, 신규 인스턴스 재기동 및 `tjesmgr boot` 성공.
- obmtsmgr의 INITPROC 부재 오류는 SYS1.TSOMAP, INITPROC, 6개 IPF 맵 생성 후 해소됐다. 후속 tmadmin에서 RDY 확인.
- 최종 확인한 25개 서버 항목 중 22개 RDY. aimapsvr/OIVPMQN은 MIN=0이며 NRDY. aimdtssv는 MIN=1이지만 `tpsvrinit fail`, `invalid param dbconn`으로 NRDY이며 예제 XSP에서도 NRDY였다. DTS 동작 성공으로 해석하지 않는다.
- 후속 `tjesmgr psjob JOB00009` 재조회는 `get password valid days failed`, `auth check failed`로 실패했다. 서버 목록의 RDY만으로 후속 명령의 사용 가능성을 보장하지 않는다. 아래 JOB 결과는 실행 당시 보존한 PSJOB/SPOOL로 확인했다. 재조회 인증 오류의 근본 원인은 미확정이다.

## 배치 결과

| 작업 | JOB ID | 실행 당시 결과 |
|---|---|---|
| IEFBR14 XSPCHK | JOB00001 | Done(R00010), SMOKE R0010, PTY PODD JESMSG 확인 |
| WRITEX | JOB00002 | Done(R00010), STEP R0010, 애플리케이션 RC 0 |
| KREADX | JOB00003 | 동일 정상 종료 |
| EREADX | JOB00004 | 동일 정상 종료 |
| PATHX | JOB00005 | 동일 정상 종료 |
| KAIXX | JOB00006 | 동일 정상 종료 |
| EAIXX | JOB00007 | 동일 정상 종료 |
| WRITEVBX | JOB00008 | 동일 정상 종료 |
| READVBX | JOB00009 | Error(A00050), STEP A0050, STATUS=A, ABEND=1 |
| WRITEVBX2 / READVBX2 | 미제출 | READVBX 실패 후 연속 실행 중단 |

XSP IEFBR14 소스는 성공 시 10을 반환하며 직접 실행도 RC 10이었다. TSAM 정상 작업의 LIST에는 `Execution AP(...) done - RC(0), STATUS(R)`와 `OSAMFRUN finish - STATUS:R, RC:10`이 기록됐다. MVS의 RC 0 판정을 그대로 사용해 첫 WRITE를 실패로 표시한 검증기 오류를 수정했다. WRITE를 중복 제출하지 않고 보존된 결과를 재판정했다.

READVBX는 `TC01`에서 `TCOBFH: a boundary violation exists` 및 `AIM0150E: boundary violation occurred`를 출력했다. 이는 단순히 정상 RC 10을 잘못 판정한 경우가 아니라 실제 JOB 실행 실패다. 원인 수정이나 성공 재검증은 수행하지 않았다.

TSAM의 실제 DB 레코드 수와 카탈로그 최종 일치 검증은 미실행이다. 별도의 dscreate/dslist/dsdelete 및 spfedit 검증은 실행 전 자동 승인 검토의 사용량 한도 오류로 차단됐다. 생성하려던 TEST.SETUP.* 데이터셋을 생성·삭제 성공으로 기록하지 않는다.

## 증거와 다음 확인

- `build-{base,batch,tacf,ndb,aim}.log`, `precomp-*.log`, `release-*.log`, `build-oflicgen.log`
- `db-precheck.log`, `{base,batch,tacf,ndb,aim}init.log`, `config-*.log`, `errcode-*.log`
- `volumes-pds.log`, `tjesinit.log`, `tjes-boot.log`, `os-type.log`, `tso-setup.log`
- `reference-tmadmin.log`, `tmadmin-first.log`, `tmadmin-final.log`
- `tsam-create.log`, `tsam-build.log`, `*-submit.log`, `*-JOB*-status.log`, `tsam-results.json`
- `spool/JOB00009/SYSMSG` 및 `ROOT.READVB.JOB00009.D000002`

tmadmin-final.log는 TSO 보완 전의 스냅샷이며 이후 조회에서 obmtsmgr RDY를 확인했다. 초기 검증 JSON은 실패 항목에도 기대 job_rc=10을 기록했으므로 원본 PSJOB/SYSMSG를 판정 근거로 사용한다. 향후 검증기는 실제 JOB 상태와 STEP RC를 파싱하고, 기대값과 실측값을 별도 필드로 보존한다.

설치 성공 이력과 회귀 테스트 완료 여부를 구분해서 재사용한다. 후속 검증 시 인증 오류를 먼저 확인하고, READVBX 원인을 좁힌 뒤 남은 VB2 및 데이터셋·레코드 검증을 실행한다. 기존 초기화나 이미 완료한 WRITE를 무조건 재실행하지 않는다.
