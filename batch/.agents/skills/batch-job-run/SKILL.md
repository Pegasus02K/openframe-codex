---
name: batch-job-run
description: 테스트 JCL을 실제 SYS1.JCLLIB 경로에 복사하고 tjesmgr로 제출한 뒤 PS, PSJOB, PODD를 사용해 JOB 상태, STEP 반환 코드, SPOOL 결과를 검증한다. OpenFrame Batch 테스트 JOB을 실행하거나 실패 원인을 조사해야 할 때 사용한다.
---

# Batch Job Run

JCL을 올바른 `SYS1.JCLLIB`에 배치하고 `tjesmgr`로 제출한 뒤 JOB 상태뿐 아니라 STEP RC와 SPOOL 내용까지 검증한다.

## 사전 확인

1. VM에 SSH로 접속하고 `of73xsp-pegasus` 컨테이너의 `/opt/oframe/ofsrc`에서 작업한다.
2. 실행할 JCL 파일과 멤버 이름을 확인한다. JCL이 호출하는 프로그램, 프로그램 라이브러리, 입력 데이터셋이 준비되어 있는지 확인한다.
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
cp -- "$jcl_source" "$jcl_library/$jcl_member"
cmp -- "$jcl_source" "$jcl_library/$jcl_member"
```

확장자 제거 여부와 멤버명은 대상 테스트의 기존 Makefile/JCL 관례를 따른다. 없는 `SYS1.JCLLIB` 디렉터리를 임의로 만들지 않는다.

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
- `psjob JOB000n`에서 최종 상태, 각 STEP의 RC, SPOOL LIST를 확인한다.
- `podd`는 같은 세션에서 `psjob` 또는 `pospool`을 먼저 실행한 후 사용한다.
- `di`는 `PSJOB`/`POSPOOL`의 SPOOL LIST에 표시된 `NO` 즉 DD index이다. EXEC STEP 순번과 혼동하지 않는다.
- 환경의 `@J` 현재 JOB 별칭이 동작하지 않으면 `podd JOB000n di=<dd-index>`를 사용한다. DD 이름으로 확인할 때는 `dn=<dd-name>`을 사용한다.

`DONE`은 JCL이 허용한 RC 범위에서 종료되었다는 뜻일 뿐 업무 결과의 성공을 보장하지 않는다. 반드시 STEP RC와 기대한 DISPLAY, 레코드, 오류 메시지가 담긴 SPOOL을 함께 확인한다.

## 실패 처리와 보고

- `ERROR`, 비정상 RC, 예상과 다른 SPOOL이 있으면 실패한 STEP과 DD를 특정한다.
- 화면 출력과 `$TMAXDIR/log`, `$OPENFRAME_HOME/log`를 확인한다.
- OpenFrame 오류 코드는 `oferror <error-code>`로 상세 내용을 조회한다.
- JCL, 프로그램 모듈, TSAM 입력 준비 문제를 구분하고 관련 스킬인 `$common-netcobol-compile` 또는 `$base-tsam-create`로 필요한 선행 작업만 수행한다.

최종 보고에는 배치한 JCL 경로, JOB ID, 최종 상태, STEP별 RC, 확인한 SPOOL DD와 핵심 결과를 포함한다.

명령이 불확실하면 다음 매뉴얼을 다시 확인한다.

- `C:\Work\srcs\Tmax\manual\openframe_batch\docs\modules\xsp-tjes-guide\pages\chapter-tjesmgr-commands.adoc`
- `C:\Work\srcs\Tmax\manual\openframe_batch\docs\modules\xsp-tjes-guide\pages\chapter-job-management.adoc`
- `C:\Work\srcs\Tmax\manual\openframe_batch\docs\modules\batch-installation-guide\pages\chapter-verifying-installation.adoc`
