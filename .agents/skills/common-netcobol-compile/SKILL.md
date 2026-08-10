---
name: common-netcobol-compile
description: 후지쯔 사양 COBOL 테스트 프로그램을 ofcbppf로 전처리하고 NetCOBOL shared object로 컴파일해 OpenFrame 테스트 프로그램 라이브러리에 배치한다. COBOL 파일 OPEN 재현 프로그램 작성, SYS1.COBLIB 등으로 테스트 모듈 설치, JCL 실행용 COBOL 빌드가 필요할 때 사용한다.
---

# Common NetCOBOL Compile

대상 테스트의 Makefile과 JCL을 기준으로 후지쯔 COBOL을 전처리, 컴파일, 배치한다. 파일 속성이나 file handler 동작을 조사할 때는 유틸리티로 파일을 대신 만들지 않고 실제 COBOL `OPEN` 경로를 타는 최소 프로그램을 사용한다.

## 사전 확인

1. 루트 `AGENTS.md`와 `.agents/openframe.local.yaml`에서 선택한 환경 프로필로 진입하고 `{{source_base}}`에서 작업한다.
2. 소스 파일, copybook, `PROGRAM-ID`, COBOL `SELECT`의 DD 이름, JCL의 프로그램명·DD·`PRGLIB`를 대조한다.
3. 도구와 환경을 확인한다.

```bash
test -n "$OPENFRAME_HOME"
command -v ofcbppf
command -v cobol
printf '%s\n' "$COBDIR" "$COBCOPY"
```

4. 대상 디렉터리에 Makefile이 있으면 전처리 옵션, 링크 라이브러리, `-WC` 옵션, 산출물 이름을 우선 적용한다. 아래 기본 옵션으로 덮어쓰지 않는다.

## 파일 OPEN 재현 프로그램 설계

RECFM, LRECL, DISP 같은 생성 속성이나 `tcobfh`/`acobfh` 동작을 확인할 때 다음 원칙으로 전용 프로그램을 만든다.

- 하나의 `SELECT`와 하나의 `FD`만 두고 JCL DD 이름을 정확히 맞춘다.
- 재현 대상에 필요한 `OPEN OUTPUT`, 최소 한 번의 `WRITE`, `CLOSE`만 수행한다.
- 각 I/O 직후 두 자리 FILE STATUS를 `DISPLAY`하고 00이 아니면 명확한 RC로 끝낸다.
- `CLOSE WITH LOCK`, 두 번째 OPEN, 기존 샘플의 업무 로직처럼 결과 판정을 흐릴 동작을 넣지 않는다.
- `FD`의 record 크기와 JCL의 LRECL을 일치시킨다.
- 프로그램명과 데이터셋 qualifier는 각각의 길이 제한을 확인한다.

KQCAMS 등 별도 유틸리티로 데이터셋을 만들면 COBOL file handler 경로를 우회할 수 있다. 조사 대상이 COBOL OPEN이면 유틸리티 결과를 재현 증거로 사용하지 않는다. 기존 프로그램을 재사용해야 한다면 첫 대상 연산의 FILE STATUS와 후속 실패를 분리해 보고한다.

## 전처리

입력 파일마다 `ofcbppf`를 실행한다.

```bash
ofcbppf -i TESTPGM.cob
test -s ofcbppf_TESTPGM.cob
```

명시적인 출력 이름, copybook 경로, main/submodule 또는 debug 옵션이 필요하면 원본 Makefile이나 `ofcbppf` 사용법에 따라 `-o`, `-copypath`, `-s`, `-t` 등을 추가한다. 전처리가 실패하면 생성된 파일로 컴파일을 시도하지 말고 첫 오류부터 해결한다.

## NetCOBOL 컴파일

후지쯔 NetCOBOL 테스트 프로그램의 기본 패턴은 다음과 같다.

```bash
cobol -dy -shared \
  -o TESTPGM ofcbppf_TESTPGM.cob \
  -L"$OPENFRAME_HOME/lib" -laimap -ltcobfh \
  -WC"STD1(JIS1),NOALPHAL,SRF(FIX),NSPCOMP(ASP),DLOAD,RSV(V125),CHECK,MODE(CCVS)"
test -s TESTPGM
```

파일 입출력 전처리를 사용하므로 `-ltcobfh`를 누락하지 않는다. 테스트가 AIM, NDB 등 추가 기능을 사용하면 원본 Makefile과 매뉴얼에 명시된 라이브러리만 추가한다. `PROGRAM-ID`와 JCL 호출명에 맞춰 main program 산출물 이름을 정한다. 서브 프로그램은 해당 테스트의 shared object 명명 규칙을 따른다.

여러 소스를 빌드할 때는 각 `ofcbppf` 성공을 확인한 뒤 해당 파일을 컴파일한다. 하나라도 실패하면 배치 단계로 넘어가지 않는다.

## 테스트 라이브러리에 배치

JCL의 `PRGLIB` 또는 테스트 Makefile에서 실제 목적지를 확인한다. 예제의 `PROD.BATCHLIB`이 현재 환경에 존재한다고 가정하지 않는다. 기본 후보가 `SYS1.COBLIB`이어도 실제 catalog, volume, 물리 경로를 확인한 뒤 사용한다.

배치 전에 다음을 확인한다.

- JCL의 `PRGLIB` 데이터셋이 catalog에 존재하는가
- 프로그램 멤버 이름이 JCL 호출명과 같은가
- 기존 멤버가 있다면 이번 테스트에서 교체해도 되는가
- 설치 도구가 요구하는 임시 라이브러리와 설정이 존재하는가

특히 `dlupdate`를 사용할 때는 먼저 사용법과 환경을 확인하고, 환경이 `SYS1.TEMPLIB` 같은 임시 라이브러리를 요구하면 그 catalog와 경로를 사전 점검한다. 전제 조건이 없는 상태에서 새 PDS를 임의로 만들거나 설치를 계속하지 않는다.

테스트 Makefile이나 설치 스크립트가 있으면 그 절차를 우선한다. 직접 파일 배치가 해당 환경에서 검증된 방식일 때만 정확한 경로를 사용한다.

```bash
program_library="$OPENFRAME_HOME/volume_DEFVOL/SYS1.COBLIB"
test -d "$program_library"
test -s TESTPGM
test ! -e "$program_library/TESTPGM" || printf 'existing member: %s\n' "$program_library/TESTPGM"
```

기존 멤버 교체는 명시적 범위 확인과 복구 방안을 마련한 뒤 수행한다. 이 스킬은 확인되지 않은 라이브러리로의 `mv`를 기본 동작으로 삼지 않는다. 실행 중 시스템에 동적 반영이 필요하면 `dlupdate` 적용 여부와 성공 결과까지 확인한다.

## 실행 검증과 문제 해결

- 전처리 파일과 최종 shared object가 비어 있지 않은지 확인한다.
- 최종 모듈 이름과 배치 경로가 JCL과 일치하는지 확인한다.
- `$batch-job-run`으로 절대 JCL 경로를 제출하고 JOB 번호, STEP RC, SPOOL, 생성 데이터셋을 확인한다.
- OPEN 재현에서는 JOB 최종 RC만 보지 말고 각 I/O의 FILE STATUS와 `dslist -a` 결과를 함께 확인한다.
- 실패 시 컴파일러의 첫 오류, `$TMAXDIR/log`, `$OPENFRAME_HOME/log`를 확인하고 OpenFrame 오류는 `oferror <error-code>`로 조회한다.

명령이 불확실하면 다음 매뉴얼을 확인한다.

- `{{manual_base}}/openframe_common/docs/modules/migration-guide/pages/mvs/sect-batch-application.adoc`
