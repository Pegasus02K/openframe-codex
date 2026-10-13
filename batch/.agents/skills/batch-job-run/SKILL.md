---
name: batch-job-run
description: 테스트 JCL을 실제 SYS1.JCLLIB 경로에 복사하고 tjesmgr로 제출한 뒤 PS, PSJOB, PODD를 사용해 JOB 상태, STEP 반환 코드, SPOOL 결과를 검증한다. OpenFrame Batch 테스트 JOB을 실행하거나 실패 원인을 조사해야 할 때 사용한다.
---

# Batch Job Run

JCL을 올바른 `SYS1.JCLLIB`에 배치하고 `tjesmgr`로 제출한 뒤 JOB 상태뿐 아니라 STEP RC와 SPOOL 내용까지 검증한다.

## 사전 확인

1. 루트 `AGENTS.md`의 프로필 선택·접속·인증·환경 적용 절차를 따른다. 제품 작업은 환경 파일에서 읽은 `$SOURCE_BASE`에서 수행한다.
2. 실행할 JCL 파일과 멤버 이름을 확인한다. JCL이 호출하는 프로그램, 프로그램 라이브러리, 입력 데이터셋이 준비되어 있는지 확인한다.
   - 실행 의도를 기록한다. 성능 분석 이슈라면 이슈·리포트의 실행 시간, 측정 구간, 데이터 규모와 환경 등 비교 조건을 확인하고 이번 JOB의 시간 측정 기준을 정한다. 보고된 실행 시간이나 사용자와 정한 기준이 없으면 비교 기준이 없음을 알리고 임의의 제한 시간을 만들지 않는다.
3. 환경과 서비스 상태를 확인한다.

```bash
test -n "$OPENFRAME_HOME"
command -v tjesmgr
printf '%s\n' "$OPENFRAME_NODENAME"
tmadmin
```

`tmadmin`에서는 필요한 서버가 준비 상태인지 확인하고 `quit`으로 나온다. 서비스를 임의로 기동하거나 종료하지 않는다. 요청 범위에 포함될 때만 `tmboot` 또는 `tmdown`을 사용한다.

## JCL 배치

실제 `SYS1.JCLLIB` 디렉터리를 확인한다. 이 환경의 일반 경로는 `$OPENFRAME_HOME/volume_DEFVOL/SYS1.JCLLIB`이지만, 여러 volume이 있거나 JCL/설정이 다른 경로를 가리키면 해당 경로를 사용한다.

```bash
find "$OPENFRAME_HOME" -maxdepth 3 -type d -name SYS1.JCLLIB -print

jcl_source=/path/to/TESTJCL
jcl_member=TESTJCL
jcl_library="$OPENFRAME_HOME/volume_DEFVOL/SYS1.JCLLIB"
test -f "$jcl_source"
test -d "$jcl_library"
test ! -e "$jcl_library/$jcl_member" || exit 1
cp -- "$jcl_source" "$jcl_library/$jcl_member"
cmp -- "$jcl_source" "$jcl_library/$jcl_member"
```

확장자 제거 여부와 멤버명은 대상 테스트의 기존 Makefile/JCL 관례를 따른다. 예제는 새 멤버 배치용이다. 기존 멤버 교체는 요청 범위와 복구 방법을 확인한 뒤 수행한다. 없는 `SYS1.JCLLIB` 디렉터리를 임의로 만들지 않는다.

## JOB 제출

환경에서 사용하는 짧은 형식으로 제출하고 출력된 JOB ID를 기록한다.

```bash
tjesmgr r "$jcl_member"
```

필요하면 동등한 RUN 형식과 노드를 사용한다.

```bash
tjesmgr RUN "$jcl_member" NODE="$OPENFRAME_NODENAME"
```

제출 메시지에서 `JOB000n` 형태의 JOB ID를 확보한다. submit 오류가 있으면 JOB ID를 추측하지 않는다.

## 결과 확인

PTY가 있는 동일한 `tjesmgr` 대화형 세션에서 상태와 결과를 확인한다.

```text
$ tjesmgr
> ps
> psjob JOB000n
> podd @J di=<dd-index>
> quit
```

- `ps`에서 대상 JOB의 현재 상태를 확인하고 종료될 때까지 필요한 간격으로 다시 확인한다.
- 성능 분석 JOB은 제출 시각과 실제 실행 시작·종료 시각을 가능한 범위에서 JOB 메타데이터 또는 로그로 확보한다. 시간의 출처·측정 구간·단위를 기록하고, 제출 후 대기를 포함한 경과 시간과 실제 실행 시간을 구분한다. 실제 실행 시각을 확인할 수 없으면 미확인으로 남긴다.
- 대기 중 경과 시간이 이슈·리포트에 보고된 실행 시간 또는 사용자와 정한 기준에 비해 과도하면 사용자에게 JOB ID·이름, 현재 상태, 경과 시간, 비교 기준과 조건 차이를 알리고 계속 기다릴지 종료할지 묻는다. 완료될 때까지 알림을 미루지 않는다. 답변 대기 중에는 상태를 관찰하고 종료 지시 없이 임의로 종료하지 않는다. 종료 지시를 받으면 대상 JOB과 현재 상태를 다시 확인하고 해당 OS·설치 버전의 TJES 명령으로 종료한 뒤 최종 상태를 확인한다.
- `psjob JOB000n`에서 최종 상태, 각 STEP의 RC, SPOOL LIST를 확인한다.
- `podd`는 같은 세션에서 `psjob` 또는 `pospool`을 먼저 실행한 후 사용한다.
- `di`는 `PSJOB`/`POSPOOL`의 SPOOL LIST에 표시된 `NO` 즉 DD index이다. EXEC STEP 순번과 혼동하지 않는다.
- 환경의 `@J` 현재 JOB 별칭이 동작하지 않으면 `podd JOB000n di=<dd-index>`를 사용한다. DD 이름으로 확인할 때는 `dn=<dd-name>`을 사용한다.
- `PODD`가 `vi` 같은 별도 뷰어를 열면 출력 확인 후 먼저 뷰어를 종료하고 다음 TJES 명령을 보낸다. `vi`에서는 `Esc`, `:q!`, Enter로 저장 없이 종료한다. 뷰어가 열린 상태에서 다음 `PSJOB`/`PODD`를 입력하지 않는다. 일반 파이프 입력은 PTY 요구로 실패할 수 있고, 독립적인 `tjesmgr podd ...`는 콘솔 전용 오류가 날 수 있다.

`DONE`은 JCL이 허용한 RC 범위에서 종료되었다는 뜻일 뿐 업무 결과의 성공을 보장하지 않는다. 반드시 STEP RC와 기대한 DISPLAY, 레코드, 오류 메시지가 담긴 SPOOL을 함께 확인한다.

## 실패 처리와 보고

- `ERROR`, 비정상 RC, 예상과 다른 SPOOL이 있으면 실패한 STEP과 DD를 특정한다.
- 화면 출력과 `$TMAXDIR/log`, `$OPENFRAME_HOME/log`를 확인한다.
- OpenFrame 오류 코드는 `oferror <error-code>`로 상세 내용을 조회한다.
- JCL, 프로그램 모듈, TSAM 입력 준비 문제를 구분한다. COBOL 빌드는 루트의 컴파일러 라우팅에 따라 `common-ofcobol-compile` 또는 `common-netcobol-compile`을 적용하고, TSAM 준비는 `base-tsam-create`로 요청 범위의 선행 작업만 수행한다.

최종 보고에는 배치한 JCL 경로, JOB ID, 최종 상태, STEP별 RC, 확인한 SPOOL DD와 핵심 결과를 포함한다.

세션에서 이슈 분석·테스트를 위해 실행한 모든 JOB을 재실행·중단 건까지 실행 건별로 다음 표에 정리한다. 성능 분석 이슈에서는 실행 시간 열을 포함하고 측정 출처·시작과 종료 기준·단위를 함께 설명한다. 일반 이슈에서는 실행 시간 열을 생략할 수 있다.

| JOB 번호(ID) | JOB 이름 | 실행 의도 | 결과 | 실행 시간(성능 분석) |
| --- | --- | --- | --- | --- |
| 실제 제출된 ID | 확인한 JOB 이름 | 재현·기준 측정·회귀 확인 등 | 상태, STEP RC, SPOOL 핵심 결과와 검증 여부 | 측정한 실행 시간; 실행 중·중단 시 현재까지의 경과 시간 |

JOB ID가 발급되지 않은 제출 실패는 ID를 추측하지 않고 별도로 보고한다. 실행 중·중단·SPOOL 미확인 상태를 성공으로 보고하지 않는다.

명령이 불확실하면 다음 매뉴얼을 다시 확인한다.

- 현재 `BATCH_OS_TYPE`에 따라 MVS는 `tjes-guide`, MSP는 `msp-tjes-guide`, VOS3는 `vos-tjes-guide`, XSP는 `xsp-tjes-guide`를 선택한다.
- `{{manual_base}}/openframe_batch/docs/modules/<선택한 TJES 모듈>/pages/chapter-tjesmgr-commands.adoc`
- `{{manual_base}}/openframe_batch/docs/modules/<선택한 TJES 모듈>/pages/chapter-job-management.adoc`
- `{{manual_base}}/openframe_batch/docs/modules/batch-installation-guide/pages/chapter-verifying-installation.adoc`
