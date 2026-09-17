# MVS 실제 검증 기록

XSP의 별도 빌드·설치 결과는 [XSP 실측 기록](../../openframe7-setup/references/xsp-validation.md)을 참고한다. 아래 MVS 결과와 합쳐 모든 OS의 검증 성공으로 해석하지 않는다.

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
