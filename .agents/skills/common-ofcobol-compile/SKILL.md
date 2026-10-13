---
name: common-ofcobol-compile
description: OpenFrame COBOL(ofcob)로 MVS·MSP·XSP·VOS3 테스트 프로그램을 OS별 옵션과 copybook·링크 설정에 맞춰 컴파일하고 테스트 라이브러리에 배치한다. TSAM·HiDB·NDB COBOL 테스트 모듈 빌드 또는 JCL 실행용 프로그램 준비에 사용한다. NetCOBOL 컴파일은 common-netcobol-compile을 사용한다.
---

# Common OFCOBOL Compile

대상 테스트의 Makefile·컴파일 스크립트와 JCL을 대조해 `ofcob`로 빌드한다. 제품별 옵션과 스크립트 함정은 [references/product-compile.md](references/product-compile.md)에서 해당 제품 항목을 읽는다.

## 환경과 도구 확인

1. 루트 `AGENTS.md`의 프로필 선택·접속·인증·환경 적용 절차를 따른다. 프로필의 `version`, `os`, `rdb`를 사용하고 `$SOURCE_BASE` 아래 대상 제품과 더 가까운 `AGENTS.md`도 적용한다.
2. 아래 컴파일러 라우팅을 먼저 수행한다. `VALUE=OFCOBOL`인 경우에만 이후 OFCOBOL 도구 검증과 컴파일 절차를 진행한다.
3. `env_path` 파일의 OFCOBOL 관련 설정만 확인한다. `OFCOB_HOME`이 환경 파일 또는 명시적으로 불러오는 하위 환경 파일에 정의되어 있어야 한다. 이전 셸에서 상속된 값만으로 준비 완료로 판단하지 않는다. 환경 파일 전체나 DB 접속 문자열은 출력하지 않는다.
4. 환경 파일 적용 후 설치 루트·실행파일·라이브러리·parser 설정을 확인한다.

```bash
test -n "${OFCOB_HOME:-}" && test -d "$OFCOB_HOME" || exit 1
test -x "$OFCOB_HOME/bin/ofcob" || exit 1
test -d "$OFCOB_HOME/lib" || exit 1
command -v ofcob || exit 1
printf 'OFCOB_HOME=%s\nCOBPARSER_HOME=%s\nOFCOBCPY=%s\n' \
  "$OFCOB_HOME" "${COBPARSER_HOME:-}" "${OFCOBCPY:-}"
ofcob -h
```

`PATH`의 `ofcob`가 선택한 설치본과 일치하는지 확인한다(심볼릭 링크는 실제 경로를 비교한다). 환경 파일의 `PATH`, `LD_LIBRARY_PATH`, `COBPARSER_HOME` 설정을 확인하고 parser 경로가 필요한 설치본에서는 그 디렉터리 존재도 검증한다. copybook 경로는 대상 소스의 `COPY`와 실제 파일을 기준으로 확인한다. 누락되거나 다른 설치본을 가리키면 원인을 보고하고 컴파일을 중단한다. 개인 환경 파일을 요청 없이 수정하지 않는다.

## 컴파일러 라우팅

환경 파일 적용 후 `common-ofconfig-manage`로 현재 노드의 `ofsys.COMPILER.COBOL_COMPILER`를 상세 조회한다.

```bash
test -n "${OPENFRAME_NODENAME:-}" || exit 1
ofconfig list -n "$OPENFRAME_NODENAME" -s ofsys -sec COMPILER -k COBOL_COMPILER -l
```

`DEFAULT_VALUE`가 아니라 현재 `VALUE`로 컴파일러를 선택한다.

| 조회한 VALUE | 적용 절차 |
| --- | --- |
| `OFCOBOL` | 이 스킬의 `ofcob` 절차 및 OS별 옵션 |
| `NETCOBOL` | `common-netcobol-compile`의 전처리·NetCOBOL 절차 |
| `MFCOBOL` | 대상 환경의 Micro Focus 컴파일 절차를 확인해 적용 |

조회 실패·빈 값·알 수 없는 값은 기본값이나 설치된 바이너리로 대신 판단하지 않고 원인을 보고한다. 컴파일러 선택을 위해 설정을 변경하지 않는다. 스크립트의 컴파일러 호출이 `VALUE`와 다르면 그대로 실행하지 말고 설정에 맞는 컴파일 명령을 구성한다. 이후 소스 사양·OS·copybook·제품별 링크 옵션을 확인한다.

## OS별 컴파일 선택

| 프로필 OS | 컴파일 모드 | TSAM Makefile target |
| --- | --- | --- |
| MVS | 일반 `ofcob`, Fujitsu/Hitachi 옵션 없음 | `mvs` |
| MSP | Fujitsu 옵션 `--enable-XSP` | `msp` |
| XSP | Fujitsu 옵션 `--enable-XSP` | `xsp` |
| VOS3 | Hitachi 옵션 `--enable-VOS` | `vos` |

옵션 지원은 설치된 `ofcob -h`로 확인한다. MSP도 `--enable-XSP`를 사용하며 확인되지 않은 `--enable-MSP`를 만들지 않는다. OS 옵션과 제품별 링크·전처리 옵션은 별도로 결정한다. XDM처럼 특정 사양을 전제로 한 소스를 다른 OS에서 발견해도 현재 OS 옵션으로 자동 변환하거나 해당 스크립트를 그대로 실행하지 않는다.

### VOS3 추가 전제

환경 파일에서 `OFCBPPH_HOME`이 명시적으로 설정되어 있고, 해당 `bin`·`lib`가 각각 `PATH`·`LD_LIBRARY_PATH`에 반영되는지 확인한다.

```bash
test -n "${OFCBPPH_HOME:-}" && test -d "$OFCBPPH_HOME" || exit 1
test -x "$OFCBPPH_HOME/bin/ofcbpph" || exit 1
test -d "$OFCBPPH_HOME/lib" || exit 1
command -v ofcbpph || exit 1
```

`PATH`에서 찾은 실행파일과 `$OFCBPPH_HOME/bin/ofcbpph`의 실제 경로도 대조한다. 환경 변수만 있거나 바이너리만 검색되는 상태는 충분하지 않다. Hitachi 모드는 `ofcbpph` 전처리 단계를 사용하므로 준비가 안 되면 컴파일하지 않는다. 기본 패턴은 원본 COBOL을 `ofcob --enable-VOS`에 전달하는 방식이며, 수동 전처리는 대상 절차가 요구할 때만 수행한다.

## 전처리와 컴파일

- 소스, `PROGRAM-ID`, copybook, 출력 모듈명, JCL 호출명·DD·프로그램 라이브러리를 대조한다. main/subprogram과 shared object 여부는 실제 대상 스크립트를 따른다. HiDB·NDB 예제의 `-U`를 모든 TSAM 프로그램에 일괄 추가하지 않는다.
- `ofcbpp`, `ofcbppf`, `ofcbpph`를 서로 바꿔 사용하지 않는다. 별도 전처리가 필요하면 해당 도구와 copybook 경로를 확인하고 성공 RC와 새 출력 파일을 확인한 뒤 컴파일한다.
- `-L`, `-l`, `OFCOBCPY`, DB client 경로, encoding·추가 옵션은 대상 Makefile·스크립트 및 설치 버전의 지원 여부로 결정한다. 과거 절대 경로나 다른 DB의 라이브러리를 복사하지 않는다.
- TSAM의 OS별 명령 구성 예시는 다음과 같다. 공개된 `textfh` 라이브러리 이름과 링크 옵션을 사용하며, 실제 적용 전 대상 Makefile·매뉴얼과 현재 설치본을 확인한다. MSP/XSP에도 MVS의 링크 옵션을 무조건 덧붙이지 않는다.

```bash
# MVS
ofcob TESTPGM.cob -g -o TESTPGM -L"$OPENFRAME_HOME/lib" -ltextfh
# MSP 또는 XSP
ofcob TESTPGM.cob --enable-XSP -g -o TESTPGM
# VOS3: 위의 OFCBPPH 검증을 먼저 완료한다.
ofcob TESTPGM.cob --enable-VOS -g -o TESTPGM -L"$OPENFRAME_HOME/lib" -ltextfh
```

이는 OS별 선택 예제다. 실제 실행에는 대상 소스와 모듈명을 사용한다. 컴파일 RC가 0이고 해당 실행에서 생성한 산출물이 비어 있지 않은지 확인한다. 실패 시 기존 산출물을 새 성공 결과로 사용하거나 배치하지 않는다.

## 배치와 검증

Makefile·스크립트에 `mv`, `cp`, `dlupdate`, 생성 파일 삭제, 스키마 변경이 포함되는지 먼저 확인한다. 컴파일만 요청받았으면 컴파일 단계만 분리한다. OS target의 부작용을 확인하기 전에는 단순 컴파일 확인을 위해 전체 target을 실행하지 않는다.

배치가 요청 범위에 있으면 실제 프로그램 라이브러리의 catalog·volume·물리 경로와 호출 멤버명을 확인한다. 기존 멤버 교체 범위와 복구 방법을 확인하고 컴파일이 모두 성공한 뒤 배치한다. `dlupdate`는 사용법·임시 라이브러리 전제·성공 RC를 확인한다. 예제의 `SYS1.COBLIB`, `SYS1.USERLIB`, `PDS.P...`가 현재 환경에 있다고 가정하지 않는다.

산출물은 `test -s`, `file`, 필요하면 `ldd`로 형식·아키텍처·누락 라이브러리를 확인한다. JOB 실행까지 요청받았으면 OS별 JCL 작성 스킬과 `batch-job-run`으로 JOB 상태·STEP RC·SPOOL을 확인한다. HiDB·NDB runtime 검증은 해당 테스트 절차와 요청 범위를 따른다. 컴파일 성공과 실행 성공을 구분한다.

완료 보고에는 선택 프로필·OS·DB, 조회 노드와 `ofsys.COMPILER.COBOL_COMPILER`의 `VALUE`, 사용한 스크립트와 입력·copybook, 전처리·컴파일 명령 및 RC, 산출물·배치 경로, 실행 검증 결과 또는 미실행 범위를 포함한다. 실패 시 첫 오류와 실패 단계를 보고한다.
