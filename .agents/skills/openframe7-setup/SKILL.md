---
name: openframe7-setup
description: OpenFrame 7의 신규 설치 환경 파일과 Tmax/TCache를 준비하고 DB 초기화, 제품 설정, 볼륨/PDS, TJES 기동 및 데이터셋·배치 잡 검증을 수행한다. MVS/MSP/XSP/VOS3 신규 설치에 사용하며 소스 빌드는 openframe7-build와 연계한다.
---

# OpenFrame 7 설치

BATCH OS는 사용자가 지정해야 한다. 미지정이면 질문하며 예제 파일명으로 결정하지 않는다. 실제 빌드는 `openframe7-build`를 함께 사용한다. MVS 검증을 다른 OS의 설치 성공으로 일반화하지 않는다.

## 설치 전 확인

1. 저장소 AGENTS.md, `.agents/openframe.local.yaml`, 선택된 프로필의 접속 방식과 env_path를 확인한다. 모든 새 셸에서 환경 파일을 적용하고 SOURCE_BASE/OPENFRAME_HOME을 검증한 후 SOURCE_BASE로 이동한다.
2. 신규 소스/설치 경로와 HOME의 새 환경 파일을 준비한다. [환경 템플릿](assets/env_openframe7.sh.template)은 정상 예제에서 추출한 구조이며 @...@를 실제 값으로 치환한 뒤 사용한다. 비밀번호는 템플릿이나 버전 관리 파일에 넣지 않는다.
3. **$HOME/packages**의 Tmax/TCache/Tibero client 바이너리를 확인한다. 없으면 위치를 질문한다. ProSort/OFCOBOL, VOS3 ofcbpph를 확인한다. Tmax/OFCOBOL/ProSort 라이선스는 packages에 없으면 예제 환경에서 재사용 가능 여부를 확인하고 없으면 질문한다.
4. `odbcinst -j`, `-q -d`, `-q -s`로 DB 드라이버/DSN을 확인하고 실제 연결을 검증한다. Tibero는 tibero_connect_string과 최신 사용자 지시를 따른다. DSN과 tbdsn.tbr 별칭도 확인한다. Oracle 정보가 없거나 연결할 수 없으면 질문한다.
5. 새 설치용 스키마가 비어 있는지 먼저 확인한다. 기존 환경의 스키마에 init/import를 실행하지 않는다. 별도 계정이 필요하면 사용자에게 요청한다. DEFVOL/100000/200000 테이블스페이스를 확인하고 없는 경우에만 생성 승인을 요청한다. DDL 예시는 `CREATE TABLESPACE "DEFVOL" DATAFILE 'DEFVOL.dbf' AUTOEXTEND ON;`이다.
6. 필요한 패키지는 설치 전에 사용자 허가를 받는다. 현재 MVS 실측 환경에는 GCC 11, make, bison, flex, bc, Python 3, unixODBC 개발 환경 및 vendor 도구가 이미 있었으며 추가 패키지를 설치하지 않았다. 이를 모든 호스트의 완전한 의존성 목록으로 취급하지 않는다. 새로 확인한 패키지는 승인·설치·재검증 결과와 함께 기록한다.

## 설치와 검증

[설치 상세 절차](references/setup.md)를 순서대로 수행한다. 기존 HOME 환경 파일을 개별 서브셸에서 적용해 포트, Tmax SHMKEY, TCache SHMKEY와 경로를 비교한다. ss/ipcs의 현재 사용량도 확인한다. 꺼진 환경의 설정값도 충돌 대상이다.

DB 접속값/ENPASSWD는 출력하지 않는다. 설치 config와 로그의 권한을 제한한다. 초기 import는 신규 설치의 명시적 작업이며, 기존 환경의 일시 설정 변경에는 `common-ofconfig-manage`의 원본 확보·복원 절차를 적용한다.

IEFBR14 작성은 MVS/MSP/VOS3에서 `jcl-write-run-general`, XSP에서 `jcl-write-run-xsp`를 사용하고 제출·JOB/STEP/SPOOL 검증은 `batch-job-run`을 함께 사용한다. 해당 스킬이 없으면 설치 위치를 검색하고 필요한 절차가 확보되기 전 실제 검증 완료로 보고하지 않는다.

설치 완료는 init/import뿐 아니라 tmboot, tjesmgr boot, dataset 도구, IEFBR14, TSAM 테스트의 실제 결과로 판단한다. MVS는 설치 상세 절차의 HiDB 기본 사용자 ENPASSWD 설정과 `test_hidam` 순차 테스트도 수행한다. 미기동 서버는 예제 환경의 실제 상태와 비교하고 이름·원인을 기록한다. 예상 외 서버 실패를 일괄 무시하지 않는다.

사용자에게 접속 후 source할 환경 파일, 소스/설치/로그 경로, 포트, JOB ID와 실패·미검증 범위를 알려준다. 사용자 요청 시 파일과 테스트 결과를 모두 보존한다. [실측 검증 기록](references/validation.md)을 참고한다.

XSP는 [XSP 실측 기록](references/xsp-validation.md)도 읽는다. 해당 버전의 JOB RC 10과 애플리케이션 RC 0을 구분하며, 확인된 VB 읽기 실패를 성공으로 일반화하지 않는다.
