# MVS 템플릿 검증

2026-09-16, `ofconfig list -n "$OPENFRAME_NODENAME" -k BATCH_OS_TYPE -l`의 현재 `VALUE=MVS`인 OpenFrame 7.3 환경에서 검증했다. MSP·VOS3는 매뉴얼의 공통 문법을 확인했으며 런타임 검증은 수행하지 않았다. XSP 스킬은 이름과 라우팅만 변경했고 XSP 재실행은 하지 않았다.

## 실행 결과

| 템플릿/멤버 | JOB ID | 최종 상태 | STEP 결과 |
| --- | --- | --- | --- |
| `general-iefbr14.jcl` / `GNLSMK16` | JOB00004 | Done(R00000) | SMOKE R0000 |
| `general-inline-copy.jcl` / `GNLCPY16` | JOB00005 | Done(R00000) | COPY R0000, DELIM R0000, SKIP 조건부 생략 |

검증된 기존 `SYS1.JCLLIB/job1`의 `EXEC PGM=IEFBR14`와 COBOL JCL의 `JOBLIB DD DSN=SYS1.COBLIB,DISP=SHR` 구성을 참고했다. 기존 멤버는 변경하거나 재실행하지 않았다.

## 확인한 증거

- `tjesmgr r GNLSMK16`, `tjesmgr r GNLCPY16`의 반환 JOB ID와 `PSJOB` 최종 상태·STEP RC를 확인했다.
- 동일한 PTY의 `tjesmgr`에서 `PSJOB` 후 `PODD`를 사용했다. `PODD`가 여는 `vi` 뷰어는 매번 `Esc :q! Enter`로 닫고 다음 명령을 보냈다.
- JOB00004의 `JESMSG`(di=5)에서 정상 JOB 상태와 SMOKE 실행을 확인했다.
- JOB00005의 COPY `SYSUT2`(di=8)는 `GENERAL-JCL-RECORD-ONE`, `GENERAL-JCL-RECORD-TWO` 두 레코드였다.
- DELIM `SYSUT2`(di=10)는 `//THIS IS INPUT DATA`, `/*THIS IS ALSO INPUT DATA` 두 레코드였다. 출력의 FB 80 공백 패딩을 제외하고 입력과 정확히 일치함을 추가 확인했다.
- COPY·DELIM `SYSPRINT`(di=7,9)의 `TOTAL RECORD COUNT = 2`와 `JESMSG`의 각 SYSUT1 R:2 / SYSUT2 W:2를 확인했다.
- `SYSMSG`의 `JRN0030I`: `EXEC COND satisfied at step SKIP - the step is to be bypassed.`로 조건부 생략을 확인했다. PSJOB에 STEP이 없다는 사실만으로 판정하지 않았다.

무할당 실행, JOB 기본 클래스, 주석, 계속행, DCB, `DD *`, `DD DATA,DLM`, `DD DUMMY`, `SYSOUT=*`, 조건부 STEP 생략이 검증 범위다. 영구 데이터셋 할당·삭제, PROC, COBOL 프로그램 실행이나 OS별 특수 기능까지 검증한 것은 아니다. 이 기록의 JOB ID와 DD index를 새 실행에 재사용하지 않는다.

테스트 멤버는 검증 후 정리하고 JOB SPOOL은 보존한다. 재현 시 assets를 새 멤버명으로 복사하고 환경과 프로그램을 확인한 뒤 위 절차를 반복한다.
