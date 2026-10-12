---
name: openframe7-setup
description: OpenFrame 7의 신규 설치 환경 파일과 Tmax/TCache를 준비하고 DB 초기화, 제품 설정, 볼륨/PDS, TJES 기동 및 데이터셋·배치 잡 검증을 수행한다. MVS/MSP/XSP/VOS3 신규 설치에 사용하며 소스 빌드는 openframe7-build와 연계한다.
---

# OpenFrame 7 설치

환경 선택·접속·인증은 루트 `AGENTS.md`를 따른다. 신규 설치 대상과 초기화·기동·테스트 범위를 현재 세션에서 확인하고 소스 빌드는 `openframe7-build`를 함께 사용한다.

## 설치 전 확인

1. 새 환경 파일을 적용하고 소스·설치·Tmax·TCache 경로가 의도한 신규 대상인지 확인한다. 기존 환경과 개인 YAML은 요청 없이 변경하지 않는다.
2. [Tibero 환경 템플릿](assets/env_openframe7.sh.template)은 구조 참고용이다. 선택한 DB와 현재 설치된 도구에 맞게 작성하며 Oracle 환경에 Tibero 설정이나 Oracle 변수 해제를 적용하지 않는다. 비밀번호는 환경 파일·템플릿·버전 관리 파일에 넣지 않는다.
3. Tmax·TCache·DB client·COBOL·Sort 및 필요한 도구의 실제 배포본·라이선스를 확인한다. 경로가 없으면 추측하지 말고 사용자에게 요청한다. 추가 패키지는 실패 증거·필요한 패키지와 설치 명령을 제시하고 허가 후 설치·재검증한다.
4. 선택한 DB의 client·전처리기·헤더·라이브러리와 ODBC 드라이버/DSN을 확인하고 실제 접속을 검증한다. 접속 대상·계정은 프로필과 현재 사용자 지시를 따른다. 암호는 인증 프롬프트 또는 출력 없는 stdin으로 전달한다.
5. 초기화 대상 스키마가 신규 설치용인지, 기존 객체가 없는지와 필요한 테이블스페이스를 확인한다. 기존 환경에 init/import를 실행하지 않는다. 계정·테이블스페이스 생성이 필요하면 사용자와 확인한다.

## 설치와 검증

[설치 상세 절차](references/setup.md)를 읽고 현재 버전·OS·DB의 실제 설치 파일과 매뉴얼에 맞춰 수행한다.

- 기동 전에 포트·Tmax/TCache 키·`ds.DATASET_RESOURCE.DATASET_SHMKEY`가 기존 환경과 겹치지 않도록 설정한다. 실행 중 자원뿐 아니라 중지 환경의 참조 값도 비교하고, 키 표기를 정수로 정규화하며 Tmax 연속 키 범위를 포함한다.
- DB 접속값·ENPASSWD는 출력하지 않고 설정·로그의 접근 권한을 제한한다. 신규 설치의 초기 import와 기존 환경의 임시 설정 변경을 구분한다. 임시 변경은 `common-ofconfig-manage`로 원본 확보·복원한다.
- JCL 작성은 현재 OS에 맞는 `jcl-write-run-general` 또는 `jcl-write-run-xsp`, 실제 제출·JOB/STEP/SPOOL 확인은 `batch-job-run`을 사용한다.
- 설치 파일·DB 초기화·설정 적용·서비스 기동·데이터셋·최소 JOB·관련 제품 테스트를 각각 검증한다. 서비스 RDY나 JOB RC만으로 전체 기능 성공을 판단하지 않는다.
- 환경·경로·포트·실제 JOB ID·검증 명령·결과·미검증 범위는 해당 세션에서 사용자에게 보고한다. 장애별 복구 조치와 결과는 현재 증거와 사용자 요청으로 결정하고 공통 지침에 세션 이력을 남기지 않는다.
