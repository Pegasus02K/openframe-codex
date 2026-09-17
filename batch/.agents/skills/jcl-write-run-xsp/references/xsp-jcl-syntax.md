# XSP JCL 문법

## 기준 매뉴얼

- 개요와 기술 형식: `{{manual_base}}/openframe_batch/docs/modules/xsp-jcl-reference-guide/pages/jcl-intro/sect-jcl-introduction.adoc`
- JOB 문: `{{manual_base}}/openframe_batch/docs/modules/xsp-jcl-reference-guide/pages/jcl/sect-job-statement.adoc`
- EX 문: `{{manual_base}}/openframe_batch/docs/modules/xsp-jcl-reference-guide/pages/jcl/sect-ex-statement.adoc`
- FD 문: `{{manual_base}}/openframe_batch/docs/modules/xsp-jcl-reference-guide/pages/jcl/sect-fd-statement.adoc`
- JEND 문: `{{manual_base}}/openframe_batch/docs/modules/xsp-jcl-reference-guide/pages/jcl/sect-jend-statement.adoc`
- 인라인 입력: 같은 디렉터리의 `sect-sysin-statement.adoc`, `sect-data-statement.adoc`, `sect-end-statement.adoc`
- IEFBR14: `{{manual_base}}/openframe_batch/docs/modules/xsp-utility-reference-guide/pages/etc/sect-iefbr14.adoc`
- 제출과 결과 확인: `{{manual_base}}/openframe_batch/docs/modules/xsp-tjes-guide/pages/chapter-tjesmgr-commands.adoc`, `chapter-job-management.adoc`

프로그램별 FD, 제어문과 정상 RC는 `xsp-utility-reference-guide`의 해당 프로그램 문서를 추가로 읽어 확인한다.

## 기본 형식

XSP JCL은 첫 번째 칸의 `\`로 제어문을 시작한다. 명칭은 두 번째 칸부터 쓰고, 명칭을 생략하면 `\` 뒤에 공백을 둔다.

```text
\ JOB XSPJOB,ML=1
\STEP1 EX IEFBR14
\ JEND
```

기본 기술 순서는 다음과 같다.

```text
JOB [JOB 범위 FD]
  EX [STEP 범위 FD와 인라인 데이터]
  EX [STEP 범위 FD와 인라인 데이터]
JEND
```

- JOB 이름은 8자 이내의 기호명칭으로 작성한다.
- STEP 이름과 일반 문장 명칭은 8자 이내이며 영문자로 시작한다.
- EX의 프로그램 이름은 필수이고 8자 이내이다. `*`는 DUMMY 프로그램을 뜻한다.
- FD 이름은 8자 이내이며 영문자로 시작한다.
- 73~80번째 칸은 임의 영역이고 81번째 칸 이후는 무시되므로 의미 있는 문장은 71번째 칸 안에서 마친다.
- 오퍼랜드는 쉼표로 구분하고 위치 오퍼랜드를 키워드 오퍼랜드보다 먼저 쓴다.
- 긴 오퍼랜드는 71번째 칸 이전의 쉼표로 계속을 표시하고 다음 줄 첫 칸을 공백으로 둔다.

## 주요 문장 대응

| 목적 | XSP 형식 | 주의점 |
| --- | --- | --- |
| JOB 시작 | `\ JOB jobname` | JOB명을 명시한다. |
| STEP 시작 | `\step EX program` | MVS의 `EXEC PGM=`이 아니다. |
| 파일 정의 | `\ FD ddname=device,...` | MVS의 `DD`가 아니다. |
| 인라인 FD | `\ FD SYSIN=*` | 뒤의 데이터는 단락문 또는 다음 제어문 전까지 입력이다. |
| JOB 종료 | `\ JEND` | 생략 가능하더라도 명시한다. |

프로그램은 다음 순서로 찾는다.

1. STEP에 지정된 `PRGLIB` 데이터셋의 멤버
2. `tjclrun`의 `SYSLIB/BIN_PATH`
3. `BIN_PATH`가 없으면 `obmjinit` 기동 환경의 `PATH`

Shared Object 형식 COBOL 애플리케이션은 환경의 실행 규약과 `OSAMFRUN` 필요 여부를 기존 JCL 및 프로그램 배치 절차에서 확인한다.

## FD 예제

직접 액세스 데이터셋을 입력 FD로 지정하는 기본 모양은 다음과 같다. 실제 장치명, 파일명, 볼륨과 `DISP`는 카탈로그 및 프로그램 요구사항에서 확인한다.

```text
\INPUT FD INPUT=DA,FILE=USER.INPUT,DISP=SHR
```

인라인 입력의 기본 모양은 다음과 같다.

```text
\SYSIN FD SYSIN=*
control statement
\/
```

단락문 `\/`가 실제 환경과 해당 JCL의 인라인 입력 종료 규칙에 맞는지 기존 성공 JCL 또는 SYSIN 매뉴얼에서 확인한다.

## MVS 형식에서 변환할 때

문자열 치환만 하지 말고 의미 단위로 다시 작성한다.

```text
//XSPJOB JOB ...       -> \ JOB XSPJOB,...
//STEP1  EXEC PGM=PGM1 -> \STEP1 EX PGM1
//INPUT  DD ...        -> \INPUT FD INPUT=... 또는 \ FD INPUT=...
```

`CLASS`, `MSGCLASS`, `MSGLEVEL`, `DSN`, `UNIT`, `SPACE`, `DCB`, `DISP`를 이름만 보고 옮기지 않는다. XSP JOB/FD 매뉴얼에서 지원되는 대응 오퍼랜드와 실제 의미를 확인한다.

## 최소 실행 확인

`IEFBR14`는 별도 FD나 제어문이 없는 테스트용 유틸리티이므로 JCL 형식과 TJES 실행 경로만 확인할 때 사용한다. XSP 유틸리티 매뉴얼에는 완료 코드 0으로 기술되어 있지만, 설치 버전과 환경에 따라 표시되는 정상 RC가 다를 수 있으므로 숫자를 스킬에 고정하지 않는다.

```text
\ JOB XSPCHK
\SMOKE EX IEFBR14
\ JEND
```

이 결과에서도 제출 성공, JOB 최종 상태, `SMOKE` STEP RC와 관련 SPOOL을 모두 확인한다. 예를 들어 2026-08-12의 OpenFrame 7.3 XSP 테스트 환경에서는 설치된 `/opt/oframe/util/IEFBR14`가 직접 실행과 TJES에서 모두 RC 10을 반환했고, `Done(R00010)`, `STATUS=R`, `ABEND=0`으로 정상 종료했다. 다른 환경에는 이 관찰값을 그대로 적용하지 말고 그 환경의 유틸리티, JOB 상태와 SPOOL을 다시 확인한다.
