# MVS 실제 검증 기록

XSP의 별도 설치·테스트 결과는 [XSP 실측 기록](xsp-validation.md)을 참고한다. 아래 MVS 결과와 합쳐 모든 OS의 검증 성공으로 해석하지 않는다.

2026-09-16, RHEL 9 / GCC 11 / Tibero 7 / Tmax 5.0 SP2 Fix4 / TCache 2.4.
추가 OS 패키지 설치 없음. ProSort/OFCOBOL/Tibero client는 정상 예제 설치를 사용했다.
MVS만 실제 빌드·설치·실행했다. MSP/XSP/VOS3와 Oracle은 이번에 실행하지 않았다.

소스: `$HOME/ofsrc_mvs_test`, 설치: `$HOME/openframe_mvs_test`.
환경: `$HOME/env_openframe_mvs_test`, 증거: `$OPENFRAME_HOME/validation`.
기존 환경을 보존하고 신규 DB 스키마에 설치했다. 사용자 요청에 따라 파일과 SPOOL을 남겼다.

| 저장소 | 브랜치 | 커밋 |
|---|---|---|
| base | rb_73 | 15c9cb3f4c0d05335864660a30e3047002eebd05 |
| batch | rb_73_FS2 | c588e22bb9511da24c3b0ce2aece605b1951a1f8 |
| tacf | rb_73 | edc54855371858de9f51f7ea631aeae1a115cfbc |
| ims | rb_72 | 57f839a871755885b061dd4c136a7743954141e4 |
| ndb | rb_73 | d7d798065f1fe1fe4c77fcb7de238e7334c9004e |
| aim | rb_73 | 4067fece3bb8ae6c570f8a92b813f5643b6e6b6c |
| dev | master | af9bb745f5d91303cff778dd3f145eba82e99578 |
| osi | rb_72 | c3de8913ad5146ca905899e22c9851a81bc6e87f |

Base/Batch/TACF/HiDB `make install` 성공. OSI는 rb_72 클론만 했고 빌드하지 않았다. NDB/AIM도 이번 MVS 빌드 대상이 아니다.
oflicgen 빌드 및 Base/Batch/TACF/HIDB 라이선스 생성 성공.

재현 중 확인한 사항:
- HiDB의 osi_io.h 헤더 의존성을 OSI rb_72 소스 추가로 해결.
- hidbinit 전에 TCache, 설정 import, hidb/lib 디렉터리가 필요.
- 최신 서버 설치 경로와 Tmax APPDIR를 링크로 연결해야 함.
- tjesinit 확인 입력을 별도 stdin으로 공급해야 함.
- TSO 기본 PROC/맵 필요. tsomapgen은 -m IPF와 basename 입력을 사용하고 파일 생성 확인.

공통 init, HiDB init, 설정 import, 오류코드 적재, 네 볼륨, PDS, tjesinit, 재기동 및 tjesmgr boot 성공.
HIDB_SAVE_LARGE_AREA PSM VALID. user_objects의 비유효 객체 0개.
ofconfig로 BATCH_OS_TYPE=MVS 확인. 최종 tmadmin si의 17개 서버 모두 RDY.
Tmax 포트 9440/RACPORT 9490, DOMAIN/NODE SHMKEY 95000/95010, TCache 0x70045는 이 실행에서 충돌을 확인한 값이며 범용 기본값이 아니다.

## 테스트 증거

- dscreate/dslist/dsdelete 성공. 삭제 검증용 TEST.SETUP.DELETE는 삭제 후 0개.
- 보존용 TEST.SETUP.DATA는 FB LRECL 80, 한 레코드 `OPENFRAME MVS INSTALLATION VERIFIED`; dsmigin 및 spfedit -b로 내용 확인.
- IEFBR14 MVSSMOKE: JOB00001 Done(R00000), SMOKE R0000, PTY PODD JESMSG 확인.
- TSAM create.sh 및 make mvs 성공. 초기 DELETE의 기존 항목 없음 메시지는 빈 스키마에서 예상된 결과.
- TSAM WRITE/KREAD/EREAD/PATH/KAIX/EAIX/WRITEVB/READVB/WRITEVB2/READVB2: JOB00002~JOB00011 모두 Done(R00000), STEP R0000.
- 전체 프로그램 SPOOL의 내용과 오류 표시를 검토. KREAD/READVB는 PTY PODD SYSOUT으로도 확인. KSDS 키/레코드와 VB의 AAAAAAAAXXXX, AAAAAAAAAAXX, AAAAAAAAAAAA를 확인.
- 실제 DB TEST_CASE1/2/3/4 레코드 수 100/200/100/100, TEST_CASE1_VB/TEST_CASE2_VB 각 3. COBOL 쓰기 루프와 일치.

로그: build-*.log, hidbinit-final.log, hidb-psm.log, volumes-pds-tjesinit.log, tjesinit.log, tmadmin-final.log, tjes-session.log, spfedit-xterm.log, tsam-results.json, tsam-spool-review.json, tsam-db-counts.log.

이 기록은 해당 커밋·환경의 검증이며 향후 설치 성공을 보장하지 않는다. 매번 포트/키/DB·도구 상태와 실제 결과를 재확인한다.

## MVS HiDB 추가 검증 (2026-09-16)

동일한 보존 테스트 환경에서 다음을 실제 확인했다.

- openframe_hidb.conf의 hidb.DEFAULT_USER.ENPASSWD와 실행 중 설정을 enpasswd ROOT SYS1의 암호화 결과로 일치시켰다. 결과값은 문서에 기록하지 않았다.
- ims/src/hidb/test에서 make 성공. si_create.sh는 schema 디렉터리 부재로 처음 실패했으며 디렉터리 생성 후 재실행 성공.
- 공통 schema, tsam/{lib,temp,copybook}, MVS hidb/{lib,temp,copybook} 디렉터리 존재 확인. MSP/XSP/VOS3의 ndb/{lib,temp,copybook}은 설치 절차에 추가했으며 이번 MVS 환경에서 실행 검증하지 않았다.
- -L, -Q, -I, -D, -R 성공 후 바로 -SI를 실행하면 보조 인덱스 결과가 기대값과 달랐다. si_truncate.sh 후 -L, -I, -SI를 실행하자 모두 성공했다. 총 8회의 test_hidam 실행에서 각 단계의 성공 메시지를 확인했다.
- 내부 실패에도 test_hidam이 RC 0을 반환하는 사례를 확인했다. 최종 판정에는 RC뿐 아니라 hidam_test_*() success.와 Error!!! 부재를 사용했다. II/GE/GB/DA 상태는 테스트가 기대하는 음성 케이스와 구분했다.
- 검증 시작 시 설정 조회가 TPETIME으로 실패했다. DB 신규 접속은 정상이며 설정 서버 재기동 후 조회가 복구됐다. 근본 원인은 확정하지 않았으며 필수 설치 단계로 일반화하지 않는다.
- 최종 ofruisvr/hidbsvr RDY 확인. 제품 소스 수정 없이 설치 디렉터리·설정·테스트 순서를 보완했다.

증거는 `$OPENFRAME_HOME/validation/hidb`에 보존했다: make.log, si-create.log, si-create-retry.log, si-truncate.log, test-{L,Q,I,D,R,SI}.log, retest-{L,I,SI}.log, final-results.json, tmadmin-final.log. 최초 실패 로그와 이후 성공 로그를 함께 남겼다. 암호 관련 백업과 상세 로그는 접근 권한을 제한했다.
