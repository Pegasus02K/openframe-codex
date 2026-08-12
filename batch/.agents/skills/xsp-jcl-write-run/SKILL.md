---
name: xsp-jcl-write-run
description: OpenFrame의 BATCH_OS_TYPE을 확인해 XSP 환경에서만 Fujitsu 형식 XSP JCL을 작성하고, 기존 batch-job-run 절차로 제출한 뒤 JOB 상태, STEP RC, SPOOL과 요청된 업무 결과를 검증한다. XSP용 JOB, EX, FD, SYSIN/DATA/JEND 문 작성, MVS 형식 JCL의 XSP 변환, XSP 배치 테스트 JCL 생성·실행 또는 XSP JCL 문법 오류 수정이 필요할 때 사용한다.
---

# XSP JCL 작성 및 실행

현재 환경이 XSP인지 확인하고 Fujitsu 형식 JCL을 작성한 뒤 실제 배치 실행 결과까지 검증한다. MVS, MSP, VOS3 문법을 XSP 문법으로 가장하지 않는다.

## 필수 경계

1. 루트 `AGENTS.md`와 `.agents/openframe.local.yaml`에서 환경 프로필을 선택하고 해당 `env_path`를 적용한다.
2. `$OPENFRAME_NODENAME`, `$SOURCE_BASE`, `$OPENFRAME_HOME`, `ofconfig`, `tjesmgr`를 확인한다.
3. 다음 상세 조회의 `VALUE`가 정확히 `XSP`일 때만 계속한다.

```bash
node_name="${OPENFRAME_NODENAME:?OPENFRAME_NODENAME is not set}"
ofconfig list -n "$node_name" -k BATCH_OS_TYPE -l
```

- 노드 이름을 `NODE1`로 고정하지 않는다.
- 값이 `MVS`, `MSP`, `VOS3`이거나 조회가 실패하면 JCL을 작성하거나 제출하지 않는다. 확인된 값과 필요한 해당 OS 스킬을 보고한다.
- 설정을 바꾸어 XSP 테스트를 만들지 않는다.
- 실제 제출과 결과 확인에는 반드시 `$batch-job-run`을 함께 사용한다.

## 작업 절차

### 1. 요구사항을 구조화하기

JOB명, 실행 프로그램, STEP 순서와 조건, FD 이름, 입력·출력 데이터셋, 인라인 입력, 예상 STEP RC, 확인할 SPOOL 또는 최종 데이터셋 결과를 정리한다. 빠진 정보는 기존 JCL, 프로그램 소스, 유틸리티 매뉴얼에서 확인하고 추측하지 않는다.

프로그램이나 유틸리티별 FD와 제어문은 `{{manual_base}}/openframe_batch/docs/modules/xsp-utility-reference-guide`에서 먼저 찾는다. COBOL 모듈 준비가 필요하면 `$common-netcobol-compile`을 사용한다.

### 2. XSP 문법으로 작성하기

JCL을 작성하거나 MVS 형식을 변환하기 전에 [XSP JCL 문법](references/xsp-jcl-syntax.md)을 읽는다. 특히 다음 규칙을 지킨다.

- 제어문 첫 칸에 `\`를 둔다.
- 기본 순서를 `JOB` -> `EX` -> 해당 STEP의 `FD` -> `JEND`로 둔다.
- JOB명, STEP명, 프로그램명, FD명은 문서의 길이와 문자 제한을 지킨다.
- `//... JOB`, `EXEC PGM=`, `//DD`를 그대로 쓰지 않는다.
- JOB과 STEP에 식별 가능한 이름을 명시하고 종료를 분명히 하기 위해 `JEND`를 작성한다.
- 인라인 데이터의 종료 문자를 데이터 일부와 혼동하지 않는다.

부작용 없는 실행 확인만 필요하면 [XSP IEFBR14 템플릿](assets/xsp-iefbr14.jcl)을 복사해 JOB명과 STEP명을 작업에 맞게 바꾼다. 실제 업무 JCL에는 필요한 FD만 추가한다.

### 3. 제출 전 점검하기

1. 첫 칸의 `\`, 문장 순서, 이름 길이, 계속행, 인라인 데이터 종료를 확인한다.
2. 프로그램이 PATH/BIN_PATH 또는 지정한 `PRGLIB`에서 실제로 해석되는지 확인한다.
3. 데이터셋과 볼륨을 실제 카탈로그 및 환경에서 확인한다.
4. 신규 데이터셋이나 삭제·갱신 FD가 있으면 이름, `DISP`, 볼륨과 정리 계획을 다시 확인한다.
5. 기존 `SYS1.JCLLIB` 멤버와 같은 이름이면 명시적 승인 없이 덮어쓰지 않는다.
6. 요청된 결과가 STEP RC만으로 충분한지 판단하고, 파일·카탈로그·DB 결과가 기준이면 별도 검증 명령을 정한다.

### 4. 실제 실행하고 검증하기

`$batch-job-run`의 절차에 따라 실제 `SYS1.JCLLIB`을 확인하고 JCL을 배치한 뒤 `tjesmgr`로 제출한다. 제출 후 다음을 모두 확인한다.

1. 제출 메시지의 실제 JOB ID
2. `PS` 또는 `PSJOB`의 최종 JOB 상태
3. 모든 STEP의 상태와 RC
4. `PSJOB`/`POSPOOL` 이후 `PODD`로 확인한 관련 SPOOL
5. 요청이 요구한 데이터셋, 카탈로그, 레코드 또는 애플리케이션 결과

`DONE`만으로 성공을 판단하지 않는다. 테스트용 JCL 멤버와 임시 파일은 정확한 경로를 확인한 뒤 정리하되, 기존 파일과 JOB SPOOL을 임의로 삭제하지 않는다.

## 실패 분류

- 제출 전에 실패하면 XSP 문법, 멤버 경로, 환경 또는 TJES 상태 문제로 분류한다.
- JOB이 `ERROR`/`FLUSH`이면 실패 STEP과 SPOOL DD를 특정하고 JCL 파싱, 프로그램 탐색, FD 할당을 구분한다.
- STEP RC가 예상과 다르면 해당 프로그램의 정상 RC 정의를 매뉴얼과 SPOOL에서 확인한다.
- JOB과 STEP이 성공해도 업무 결과가 다르면 JCL 성공과 결과 실패를 분리해 보고한다.
- OpenFrame 오류 코드는 `oferror <error-code>`로 확인하고 `$TMAXDIR/log`, `$OPENFRAME_HOME/log`의 같은 시각 로그를 조사한다.

## 최종 보고

다음을 간결하게 보고한다.

- 확인한 노드와 `BATCH_OS_TYPE`
- 작성·배치한 JCL 경로와 핵심 문장
- JOB ID, 최종 상태, STEP별 RC
- 확인한 SPOOL DD와 핵심 출력
- 요청된 최종 결과와 성공·실패 판단
- 생성·변경·정리한 테스트 자원
