---
name: base-tsam-create
description: OpenFrame Base 테스트에 필요한 TSAM VSAM 데이터셋과 copybook을 준비하고 정리한다. copybook 배치, AUTO_CREATE_COPYBOOK 자동 생성 추적, 레이아웃 크기와 키 경계 검증, idcams CLUSTER·AIX·PATH 생성·삭제와 LIBGEN이 필요할 때 사용한다.
---

# Base TSAM Create

테스트가 요구하는 정확한 데이터셋만 `idcams`로 생성하거나 삭제한다. 이 스킬에 내장된 명령 기준과 Base `idcams` 매뉴얼을 사용하되, 대상 테스트의 이름·조직·키·레코드 길이를 먼저 확인한다.

## 작업 위치와 사전 확인

1. 루트 `AGENTS.md`와 `.agents/openframe.local.yaml`에서 작업 대상에 맞는 환경 프로필을 선택하고, 환경 파일에서 읽은 `$SOURCE_BASE`에서 작업한다.
2. 다음 값을 추측하지 말고 확인한다.

```bash
pwd
test -n "$OPENFRAME_HOME"
command -v idcams
command -v listcat
```

3. 테스트의 JCL, COBOL, copybook과 데이터 준비 절차를 읽어 필요한 카탈로그 항목을 표로 정리한다.

- CLUSTER: 이름, 조직(`KS`, `ES`, `RR`), 평균/최대 레코드 길이
- KSDS: 키 길이와 offset
- AIX: 이름, base cluster, 키 길이와 offset, 필요 시 레코드 길이
- PATH: 이름과 연결할 AIX
- LIBGEN: 대상 유형과 `-rn`, `-bi` 등 명시된 옵션

## Copybook 준비와 레이아웃 검증

TSAM은 테이블을 생성할 레코드 레이아웃으로 `.cpy` copybook을 사용한다. 사용할 copybook은 반드시 `$OPENFRAME_HOME/tsam/copybook`에 있어야 한다.

먼저 필요한 TSAM 디렉터리를 확인한다. `$OPENFRAME_HOME` 값이 올바른 것을 확인한 후 없는 디렉터리는 직접 생성하고 생성 사실을 작업 결과에 남긴다.

```bash
for tsam_dir in \
  "$OPENFRAME_HOME/tsam/copybook" \
  "$OPENFRAME_HOME/tsam/temp" \
  "$OPENFRAME_HOME/tsam/lib"
do
  test -d "$tsam_dir" || mkdir -p -- "$tsam_dir"
done
```

테스트가 제공하는 copybook이 있으면 대상 파일을 명시적으로 복사하고, 복사된 파일을 다시 확인한다. 넓은 위치에서 `*.cpy`를 무조건 복사하지 않는다.

```bash
copybook_source=/path/to/RECORD.cpy
copybook_name=RECORD.cpy
copybook_dir="$OPENFRAME_HOME/tsam/copybook"

test -f "$copybook_source"
cp -- "$copybook_source" "$copybook_dir/$copybook_name"
cmp -- "$copybook_source" "$copybook_dir/$copybook_name"
```

`$common-ofconfig-manage`를 사용하여 `AUTO_CREATE_COPYBOOK` 설정을 `ofconfig list -n "$OPENFRAME_NODENAME"`으로 확인한다. subject와 section을 알고 있으면 `-s`, `-sec`, `-k AUTO_CREATE_COPYBOOK`으로 조회 범위를 좁힌다. 테스트를 위해 이 값을 바꾸면 해당 스킬의 스냅샷·복원 절차를 반드시 따른다.

- 설정이 `NO`이면 copybook이 없을 때 생성 작업을 시작하지 말고 올바른 파일을 배치한다.
- 설정이 `YES`이면 copybook이 없어도 TSAM이 새 copybook을 만들 수 있다. 생성 전 파일 존재 여부를 기록하고, 생성 후 새 파일이 만들어졌는지 확인한다.
- 자동 생성된 copybook을 사용했다면 최종 보고에 `AUTO_CREATE_COPYBOOK=YES`였다는 사실, 생성된 파일의 전체 경로, 실제 사용한 파일명을 반드시 포함한다.

`idcams define` 전에 copybook의 물리적 레이아웃을 계산하고 다음 조건을 검증한다.

1. COPY 확장, `OCCURS`, `REDEFINES`, `PIC`, `USAGE`에 따른 실제 저장 크기를 반영하여 전체 레코드 크기를 계산한다. PIC에 보이는 자릿수만 단순 합산하지 않는다.
2. copybook 전체 레이아웃 크기와 `idcams -l average,maximum`으로 지정할 레코드 길이를 같게 맞춘다.
3. KSDS와 AIX의 키 구간을 `[offset, offset + length)`로 계산한다.
4. 키 시작 offset과 키 끝 offset이 모두 copybook 필드 경계에 놓이는지 확인한다.
5. 하나의 키가 서로 인접한 여러 필드를 포함하는 것은 허용한다. 키 경계가 어떤 필드의 중간을 잘라서는 안 된다.

검증 결과를 다음 형태로 남긴 뒤 생성한다.

```text
copybook: RECORD.cpy
layout-size: 80
idcams-lrecl: 80,80
key: offset=0, length=12, fields=KEY-A(4)+KEY-B(8)
key-boundary: valid
```

## 생성

의존성 순서대로 CLUSTER, LIBGEN이 필요한 항목, AIX, PATH를 생성한다. 조직별 필수 옵션은 다음과 같다.

- ESDS: `-o ES -l average,maximum`
- KSDS: `-o KS -k length,offset -l average,maximum`
- RRDS: `-o RR -l average,maximum`
- AIX: `-r base-cluster -k length,offset`과 필요 시 `-l average,maximum`
- PATH: `-p alternate-index`
- LIBGEN: `idcams libgen type -n name`과 필요 시 `-rn`, `-bi`

다음 명령을 기준으로 실제 이름과 속성을 대입한다.

```bash
idcams define CL -n "$esds_name" -o ES -l "$average_lrecl,$maximum_lrecl"
idcams define CL -n "$ksds_name" -o KS -k "$key_length,$key_offset" -l "$average_lrecl,$maximum_lrecl"
idcams define CL -n "$rrds_name" -o RR -l "$average_lrecl,$maximum_lrecl"

idcams libgen CL -n "$cluster_name" -rn "$rownum"

idcams define AIX -n "$aix_name" -r "$base_cluster" -k "$aix_key_length,$aix_key_offset"
idcams define PATH -n "$path_name" -p "$aix_name"
```

`libgen`은 테스트가 shared object 재생성을 요구할 때만 실행한다. `-rn`은 READ NEXT에 사용할 ROWNUM이고, `-bi`는 bulk insert 시 한 번에 처리할 레코드 수이다. 옵션을 임의로 추가하거나 바꾸지 않는다. 명령 하나가 실패하면 후속 의존 항목을 계속 만들지 말고 오류를 해결한 뒤 다시 실행한다.

## 삭제와 재생성

삭제는 생성의 역순인 PATH, AIX, CLUSTER 순서로 수행한다. Base cluster를 먼저 지워 AIX의 shared object가 남는 상황을 피한다.

```bash
idcams delete PATH -n "$path_name"
idcams delete AIX -n "$aix_name"
idcams delete CL -n "$cluster_name"
```

삭제 전에 `listcat -n <name>`으로 존재 여부와 유형을 확인한다. 존재하지 않는 항목의 삭제 오류는 정리 단계에서만 명시적으로 구분하고, 권한·DB·카탈로그 오류를 성공으로 취급하지 않는다. `-F`는 연결 항목까지 삭제할 의도가 확인된 경우에만 사용한다.

재생성할 때는 정확한 대상만 삭제한 후 생성 절차를 다시 수행한다. 다른 테스트나 공용 데이터셋은 건드리지 않는다.

## 검증과 오류 처리

각 항목을 상세 조회하여 조직, 키, 레코드 길이, 연결 관계를 확인한다.

```bash
listcat -l -n "$cluster_name"
listcat -l -n "$aix_name"
listcat -l -n "$path_name"
```

생성 중 오류가 발생하면 다음 순서로 원인을 확인하고 사용자에게 진단 결과를 보고한다.

1. 사용하려는 `.cpy` 파일이 `$OPENFRAME_HOME/tsam/copybook`에 있는지 확인한다.
2. `$OPENFRAME_HOME/tsam/copybook`, `$OPENFRAME_HOME/tsam/temp`, `$OPENFRAME_HOME/tsam/lib`가 모두 존재하는지 확인한다.
3. 실제 사용된 copybook의 전체 레이아웃 크기와 `idcams -l` 정의가 일치하는지 확인한다.
4. `idcams -k length,offset`으로 지정한 키의 시작과 끝이 copybook 필드 경계에 맞는지 확인한다.
5. 화면 출력과 `$TMAXDIR/log`, `$OPENFRAME_HOME/log`를 확인하고, OpenFrame 오류 코드는 `oferror <error-code>`로 조회한다.

copybook 형태와 관련된 문제는 자동으로 임의 수정하지 말고 사용자에게 원인과 가능한 수정 방안을 제안한다.

- 전체 크기 불일치: `idcams -l`을 실제 레이아웃 크기에 맞추거나, 의도한 레코드 길이에 맞게 copybook의 `PIC`, `OCCURS`, filler 구성을 조정한다.
- 키 시작 또는 끝이 필드 중간에 위치: `-k`의 offset/length를 필드 경계에 맞추거나, 업무 키가 온전한 필드 조합이 되도록 copybook 필드 구성을 조정한다.
- 자동 생성된 copybook의 레이아웃이 의도와 다름: 자동 생성본을 사용자에게 알리고, 명시적인 `.cpy`를 제공하거나 생성본을 검토·수정한 뒤 재생성하도록 제안한다.

copybook 형태 이외의 문제는 가능한 범위에서 직접 해결하고 생성 명령을 다시 실행한다. 예를 들어 copybook 경로가 잘못되었으면 올바른 위치로 복사하고, 필수 디렉터리가 없으면 생성하며, 권한·환경·카탈로그 문제는 로그와 설정을 확인하여 수정한다. 해결하지 못한 경우에만 확인한 증거와 남은 차단 원인을 사용자에게 보고한다.

최종 보고에는 생성·삭제한 카탈로그 항목, 실행한 `idcams` 명령, 사용한 copybook 경로, 레이아웃 크기와 `-l` 비교, 키 필드 경계 검증 결과를 포함한다. 자동 생성된 copybook을 사용했다면 그 사실을 별도로 명시한다.

옵션이 불확실하면 다음 매뉴얼을 확인한다.

- `{{manual_base}}/openframe_base/docs/modules/tool-reference-guide/pages/ds/sect-idcams.adoc`
- `{{manual_base}}/openframe_base/docs/modules/tool-reference-guide/pages/ds/sect-listcat.adoc`
