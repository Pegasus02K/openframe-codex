# 공통 JCL 문법과 매뉴얼 라우팅

## 매뉴얼 선택

기준 루트는 `.agents/openframe.local.yaml`의 `manual_base` 아래 `openframe_batch/docs/modules`이다. 실제 디렉터리와 파일 존재를 확인한 후 읽는다.

| BATCH_OS_TYPE | JCL | 유틸리티 | TJES |
| --- | --- | --- | --- |
| MVS | `jcl-reference-guide` | `utility-reference-guide` | `tjes-guide` |
| MSP | `msp-jcl-reference-guide` | `msp-utility-reference-guide` | `msp-tjes-guide` |
| VOS3 | `vos-jcl-reference-guide` | `vos-utility-reference-guide` | `vos-tjes-guide` |

MVS·MSP의 기본 제어문은 각 JCL 모듈의 `pages/jcl/`, VOS3는 `pages/os-jcl/`에 있다. 필요한 문장에 따라 `sect-job-statement.adoc`, `sect-exec-statement.adoc`, `sect-dd-statement.adoc`, `sect-special-dd-statement.adoc`, `sect-proc-statement.adoc`, `sect-pend-statement.adoc`를 읽는다.

형식과 계속행은 MVS·MSP의 `pages/jcl-intro/sect-jcl-type.adoc`와 `sect-jcl-statement.adoc`, VOS3의 `pages/chapter-introduction.adoc` 및 그 include 문서를 확인한다. 제출·출력은 해당 TJES 모듈의 `pages/chapter-tjesmgr-commands.adoc`, `chapter-job-management.adoc`에서 연결된 문서를 읽는다.

IEFBR14와 IEBGENER는 해당 Utility 모듈의 `pages/etc/sect-iefbr14.adoc`, `pages/ds/sect-iebgener.adoc`를 확인한다. 매뉴얼이 문법 검사만 지원한다고 명시한 오퍼랜드를 실제 기능이 구현된 것으로 취급하지 않는다.

## 기본 구조

```jcl
//MYJOB    JOB
//STEP01   EXEC PGM=IEFBR14
//
```

- 제어문은 1~2열의 `//`, 명칭은 3열부터 시작한다. 새 템플릿은 영문자로 시작하는 영문자·숫자 8자 이내의 JOB/STEP/DD 이름을 사용한다. STEP명은 JOB 내에서 고유하게 둔다.
- JOB 뒤에 하나 이상의 EXEC, 각 EXEC 뒤에 해당 STEP의 DD를 둔다. `JOBLIB DD`는 첫 EXEC 이전, `STEPLIB DD`는 해당 STEP 안에 둔다.
- `//*`는 주석, `/*`는 인라인 데이터의 단락문, `//`만 있는 공문은 JOB 종료이다. 입력 파일 EOF도 마지막 JOB을 끝낼 수 있다. XSP의 `EX`, `FD`, `JEND`를 혼용하지 않는다.
- `EXEC PGM=`의 프로그램명과 COBOL 라이브러리는 실제 설치 및 성공 JCL로 확인한다. PROC 호출은 프로그램 실행과 구분한다.

## 계속행

제어문의 의미 있는 내용은 71열 안에서 끝내고 탭 대신 공백을 사용한다. 새 JCL에서는 쉼표로 오퍼랜드를 나누는 계속행을 우선한다.

```jcl
//OUTPUT   DD DSN=USER.RESULT,DISP=(NEW,CATLG,DELETE),
//            DCB=(RECFM=FB,LRECL=80,BLKSIZE=800)
```

쉼표를 71열 이전에 두고 다음 행은 `//` 뒤 명칭 없이 4~16열 사이에서 오퍼랜드를 시작한다. 72열 계속 표시와 인용 문자열 계속은 해당 OS 매뉴얼을 확인한다. 73~80열은 임의 영역이므로 오퍼랜드를 두지 않는다.

## DD와 인라인 입력

| 목적 | 기본 형태 | 확인 사항 |
| --- | --- | --- |
| 기존 입력 | `DD DSN=USER.INPUT,DISP=SHR` | 실제 카탈로그·프로그램 접근 요구 |
| 신규 출력 | `DD DSN=USER.OUTPUT,DISP=(NEW,CATLG,DELETE)` | 이름 충돌, DCB·SPACE·볼륨, 종료 시 처리 |
| SPOOL 출력 | `DD SYSOUT=*` | JOB 메시지 클래스의 출력 처리 |
| 빈 입력/버릴 출력 | `DD DUMMY` | 프로그램이 요구하는 DD 이름 |
| 일반 인라인 입력 | `DD *` 뒤 데이터와 `/*` | 데이터가 제어문으로 해석되지 않는지 |
| 제어문 형태의 입력 | `DD DATA,DLM=ZZ` 뒤 데이터와 `ZZ` | 구분자와 실제 데이터의 충돌 |

MVS·MSP 매뉴얼의 애스터리스크 절은 `DD *`에 DLM을 사용할 수 없다고 명시한다. 공통 템플릿은 `DD DATA,DLM=...`를 사용한다. `//`나 `/*`로 시작하는 데이터를 보존할 때는 해당 입력·구분자를 실제 출력에서 확인한다. 종료 구분자 뒤에 다음 STEP/DD를 배치한다.

`DISP=(상태,정상종료처리,이상종료처리)`를 구분한다. `SHR/OLD/NEW/MOD`, `KEEP/DELETE/CATLG/UNCATLG/PASS`의 의미와 생략 시 기본 처리를 확인한다. 임시 데이터셋 `&&name`과 STEP 간 PASS, `*.STEP.DD` 역참조는 카탈로그된 영구 데이터셋과 동일하게 취급하지 않는다. 할당을 검증해야 하는 요청에는 SPOOL 복사만으로 검증을 대신하지 않는다.

## STEP 조건과 특수 기능

`EXEC COND=(코드,연산자,이전STEP)`는 조건이 참이면 현재 STEP을 건너뛴다. 예를 들어 `COND=(0,EQ,COPY)`는 COPY가 RC 0일 때 현재 STEP을 생략한다. 여러 조건, `EVEN`, `ONLY`, JOB 수준 COND는 적용 범위와 ABEND 동작을 매뉴얼에서 확인한다.

PROC/PEND, 심볼릭 파라미터, DD 오버라이드, 조건문, JES2/JES/JSS3 제어문, MSP 매크로, VOS3 전용 오퍼랜드 등이 필요한 경우 해당 OS 문서를 추가로 읽고 최소 사례를 검증한다. 기본 스킬을 OS별로 복제하지 않고 실제 차이가 생긴 기능만 별도 참조로 분리한다.
