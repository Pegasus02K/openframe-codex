---
name: batch-job-run
description: OpenFrame 배치 JCL을 실행 환경에서 사전 점검하고 tjesmgr로 제출한 뒤 JOB 상태, STEP RC, SPOOL, 생성 데이터셋을 확인한다. JCL 재현 케이스 실행, COBOL 프로그램 호출, 배치 실패 분석, dslist 결과 검증이 필요할 때 사용한다.
---

# Batch Job Run

JCL을 실행 가능한 상태로 점검하고, 제출부터 SPOOL과 산출물 확인까지 하나의 JOB 번호로 추적한다. 단순 JOB 성공 여부보다 재현 대상 동작의 실제 증거를 우선한다.

## 사전 확인

1. 루트 `AGENTS.md`와 `.agents/openframe.local.yaml`에서 선택한 환경 프로필로 진입한다.
2. OpenFrame 노드와 런타임 도구를 확인한다.

```bash
test -n "$OPENFRAME_NODENAME"
command -v tjesmgr
command -v dslist
```

3. JCL의 프로그램명, `PRGLIB`, 입력 데이터셋, 출력 데이터셋, 예상 STEP RC를 기록한다.
4. 데이터셋 이름의 각 qualifier가 1~8자인지 확인한다. 재현용 이름도 이 제한을 지킨다.
5. 기존 데이터가 결과를 오염시키지 않도록 출력 데이터셋의 존재 여부를 확인한다. 기존 데이터셋은 사용자 승인 없이 삭제하거나 재사용하지 않는다.

```bash
dslist -a CODEX.FBC.OUTPUT
```

6. JCL이 호출하는 프로그램 라이브러리와 멤버가 실제로 존재하는지 확인한다. 예제 JCL의 `PROD.BATCHLIB` 같은 값을 그대로 가정하지 않는다.

## JCL 경로 준비

JMSVR가 읽을 수 있는 컨테이너 내부의 절대 경로를 사용한다. 현재 디렉터리 기준 상대 경로는 제출 클라이언트에서는 보여도 JMSVR에서 `TJES_ERR_JCL_PATH` 또는 `-9800`으로 실패할 수 있다.

```bash
jcl_path="/opt/oframe/ofsrc/.codex-repro/FBCREPRO"
test -f "$jcl_path"
test -r "$jcl_path"
case "$jcl_path" in
  /*) ;;
  *) printf 'JCL path must be absolute: %s\n' "$jcl_path" >&2; exit 1 ;;
esac
```

JCL에 출력 데이터셋 속성이 포함되었는지, COBOL `SELECT`의 DD 이름과 JCL의 DD 이름이 일치하는지 다시 확인한다.

## JOB 제출과 상태 확인

절대 경로로 제출하고 출력된 JOB 번호를 즉시 기록한다.

```bash
tjesmgr r "$jcl_path"
tjesmgr psjob JOB0001
```

JOB이 끝날 때까지 같은 JOB 번호로 `psjob`을 반복한다. `tjesmgr ps`는 환경에 따라 대화형 세션으로 남을 수 있으므로 자동화된 단발 조회에는 사용하지 않는다.

최종 상태에서 다음을 분리해 기록한다.

- JOB/STEP의 최종 RC
- 재현 대상 OPEN, WRITE, CLOSE 등의 FILE STATUS
- 기대한 데이터셋의 생성 여부와 속성
- 후속 로직 때문에 발생한 별도 실패

후속 OPEN이나 검증 로직이 실패해 STEP RC가 비정상이어도, 첫 번째 대상 연산이 성공했다면 그 SPOOL 행과 생성 결과를 별도 증거로 남긴다. 반대로 JOB RC 0만으로 대상 동작이 성공했다고 판단하지 않는다.

## SPOOL 확인

JOB 상세는 단발 명령으로 확인한다. `podd`가 필요한 상세 SPOOL은 `tjesmgr` 대화형 콘솔에서 JOB을 선택한 뒤 연다.

```text
$ tjesmgr
tjesmgr> psjob JOB0001
tjesmgr> podd @J di=2
```

`di` 값은 `psjob`에 표시된 대상 DD index를 사용한다. SPOOL 뷰어가 읽기 전용 Vim으로 열리면 `:q!`로 닫고, `tjesmgr`에서는 `quit`으로 종료한다. JOB 번호와 DD index를 추측하지 않는다.

다음 내용을 우선 확인한다.

- JCL 변환·파싱 오류
- 프로그램 탐색과 로드 오류
- COBOL FILE STATUS 또는 애플리케이션 DISPLAY
- STEP RC와 ABEND 정보
- 생성·할당 관련 OpenFrame 오류 코드

OpenFrame 오류 코드는 `oferror <error-code>`로 조회하고, 필요하면 `$TMAXDIR/log`와 `$OPENFRAME_HOME/log`를 같은 시간대 기준으로 확인한다.

## 산출물 검증

새 데이터셋 생성이 목적이면 실행 직후 카탈로그 속성을 확인한다.

```bash
dslist -a CODEX.FBC.OUTPUT
```

RECFM 같은 속성 재현에서는 다음을 함께 보존한다.

- JCL의 원래 속성 지정 행
- COBOL에서 해당 DD를 실제로 `OPEN OUTPUT`한 증거
- FILE STATUS 00 또는 대응 성공 로그
- `dslist -a`의 최종 속성

KQCAMS 같은 유틸리티가 만든 데이터셋과 COBOL file handler가 `OPEN`으로 만든 데이터셋은 생성 경로가 다를 수 있다. COBOL 런타임 문제를 재현할 때 유틸리티 생성 결과로 대체하지 않는다. 전용 COBOL 프로그램이 필요하면 `$common-netcobol-compile`을 사용한다.

## 정리 원칙

재현용 산출물만 정확한 이름으로 식별해 정리한다. 기존 라이브러리, 기존 데이터셋, 다른 JOB의 SPOOL은 삭제하지 않는다. 사용자에게 남겨야 할 최종 JCL 경로, JOB 번호, 데이터셋 이름을 보고한다.
