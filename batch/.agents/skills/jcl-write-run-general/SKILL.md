---
name: jcl-write-run-general
description: OpenFrame MVS·MSP·VOS3의 공통 JOB/EXEC/DD 문법으로 JCL을 작성·수정하고, 실제 제출 시 JOB 상태·STEP RC·SPOOL·업무 결과를 검증한다. 세 OS의 배치 테스트 JCL 작성, JCL 문법 오류 수정 및 실행 검증에 사용한다. XSP JCL은 jcl-write-run-xsp를 사용한다.
---

# MVS·MSP·VOS3 JCL 작성 및 실행

세 OS의 기본 문법과 작성 절차를 함께 관리한다. OS별 스킬을 미리 분리하지 않으며, 특수 기능이 필요할 때 해당 OS 매뉴얼과 설치 버전의 지원 여부를 확인한다. MVS 실행 성공을 MSP·VOS3 실행 검증으로 보고하지 않는다.

## 환경 확인

루트 `AGENTS.md`와 `.agents/openframe.local.yaml`의 선택된 프로필을 따르고, 선택 환경에서 `env_path`를 적용한 뒤 `$SOURCE_BASE`, `$OPENFRAME_HOME`과 실제 작업 경로를 확인한다. 새 SSH 명령이나 셸에서도 다시 적용한다.

`$common-ofconfig-manage`의 조회 절차로 현재 값을 확인한다.

```bash
node_name="${OPENFRAME_NODENAME:?OPENFRAME_NODENAME is not set}"
ofconfig list -n "$node_name" -k BATCH_OS_TYPE -l
```

- `VALUE`가 `MVS`, `MSP`, `VOS3` 중 하나여야 한다. 기본값이나 환경 파일 이름을 현재 값으로 간주하지 않는다.
- `XSP`이면 `$jcl-write-run-xsp`로 라우팅한다. 명시된 대상 OS와 실제 환경이 다르면 사용자에게 확인한다.
- 조회 실패·알 수 없는 값이면 실행을 중단하고 환경 문제를 확인한다. 테스트를 위해 `BATCH_OS_TYPE`을 변경하지 않는다.
- 실제 제출과 결과 확인에는 [batch-job-run](../batch-job-run/SKILL.md)을 함께 사용한다. 작성만 요청되면 제출하지 않는다.

## 작성

1. JOB명, 프로그램, STEP 순서·조건, DD 이름, 입력·출력, 예상 RC와 업무 결과를 정한다. 실제 `SYS1.JCLLIB`의 검증된 JCL과 프로그램별 매뉴얼을 먼저 확인한다. 기존 멤버는 읽기 자료로 사용하고 덮어쓰지 않는다.
2. [공통 문법과 OS별 매뉴얼 경로](references/general-jcl-syntax.md)를 읽는다. `JOB → EXEC → DD` 구조, 계속행, 인라인 입력 종료, 데이터셋 수명과 조건부 실행을 점검한다.
3. 프로그램별 DD와 제어문은 해당 OS의 Utility Reference Guide에서 확인한다. COBOL 실행은 기존 성공 JCL의 `JOBLIB`/`STEPLIB`, 프로그램명 및 DD 계약을 따른다. 컴파일이 필요할 때만 적합한 컴파일 절차를 사용하며, MVS라는 이유로 Fujitsu용 전처리나 XSP 실행 래퍼를 추가하지 않는다.
4. 실행 경로만 확인하려면 [무할당 IEFBR14](assets/general-iefbr14.jcl)를 사용한다. DD 입력·출력까지 확인하려면 [인라인 복사](assets/general-inline-copy.jcl)를 사용한다. JOB명과 멤버명은 기존 자원과 충돌하지 않게 정한다.

템플릿의 JOB CLASS·MSGCLASS는 기본값을 사용한다. 기본값이 해당 환경에서 실행·출력 가능한지 검증된 JCL과 설정으로 확인하고 필요한 경우 명시한다. 특정 환경의 클래스, 볼륨, 계정, 프로그램 라이브러리를 공통 스킬에 고정하지 않는다.

## 제출과 검증

- `batch-job-run`에 따라 서비스 상태, 실제 JCL 라이브러리와 프로그램 탐색 경로를 확인한다. 신규 생성·갱신·삭제 DD는 대상과 정상/이상 종료 시 `DISP`를 검토한다. `IEFBR14`도 DD 할당·후처리를 수행하므로 데이터셋 조작이 없는 프로그램이라고 가정하지 않는다.
- 기존 멤버를 덮어쓰지 않고 배치하고 원본과 비교한다. `tjesmgr r <member>`가 반환한 실제 JOB ID를 기록한다.
- `PSJOB`에서 최종 JOB 상태와 STEP별 RC를 확인한다. `COND` 등으로 생략된 STEP은 기대한 조건에 의한 것인지 `JESMSG` 등에서 확인한다.
- 동일한 PTY의 `tjesmgr` 세션에서 `PSJOB`/`POSPOOL` 후 `PODD`로 출력 DD를 읽는다. 같은 DD명이 여러 STEP에 있으면 SPOOL LIST의 `NO`를 `di=`로 지정한다. 파이프 입력이나 독립적인 `tjesmgr podd ...`는 설치본에 따라 지원되지 않는다.
- 인라인 복사 템플릿은 `COPY`, `DELIM`의 RC 0, 각 `SYSUT2`의 입력 레코드 두 개, `SKIP`의 조건부 생략을 확인한다. `INPJCL`에 문자열이 있는 것만으로 출력 성공을 판정하지 않는다.
- 실제 업무 요청이면 추가로 데이터셋·카탈로그·레코드·DB 등 요청한 최종 결과를 확인한다. `DONE`이나 RC만으로 대체하지 않는다.
- 테스트 멤버와 임시 파일은 이번 작업에서 만든 정확한 경로만 정리한다. 사용자가 보존을 요청한 산출물과 기존 멤버, JOB SPOOL은 보존한다.

## 실패와 보고

환경/서비스 실패, JCL 파싱 실패, 프로그램 탐색 실패, DD 할당 실패, 프로그램 오류, 결과 불일치를 구분한다. 오류 코드는 `oferror`와 해당 시각 로그로 확인한다. 문법 오류를 해결하려고 서비스 재기동이나 OS 설정 변경을 임의로 수행하지 않는다.

보고에는 확인한 OS, JCL 경로, JOB ID, 최종 상태, STEP별 RC·생략 사유, 확인한 출력 DD와 업무 결과, 정리한 자원을 포함한다. [MVS 검증 기록](references/mvs-validation.md)은 템플릿의 실제 검증 범위와 재현 기준이 필요할 때 읽는다.
