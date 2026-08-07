---
name: common-netcobol-compile
description: 후지쯔 사양 COBOL 테스트 프로그램을 ofcbppf로 전처리하고 NetCOBOL로 shared object로 컴파일한 뒤 OpenFrame 테스트 프로그램 라이브러리에 배치한다. COBOL 테스트 소스를 빌드하거나 SYS1.COBLIB 등의 실행 경로에 설치해야 할 때 사용한다.
---

# Common NetCOBOL Compile

대상 테스트의 Makefile과 JCL을 기준으로 후지쯔 COBOL을 전처리, 컴파일, 배치한다. 별도 샘플 파일에 의존하지 않고 이 스킬에 내장된 전처리·링크·컴파일 옵션을 기본 패턴으로 사용한다.

## 사전 확인

1. VM에 SSH로 접속하고 `of73xsp-pegasus` 컨테이너의 `/opt/oframe/ofsrc`에서 작업한다.
2. 소스 파일, copybook, `PROGRAM-ID`, JCL의 프로그램명과 프로그램 라이브러리를 확인한다.
3. 도구와 환경을 확인한다.

```bash
test -n "$OPENFRAME_HOME"
command -v ofcbppf
command -v cobol
printf '%s\n' "$COBDIR" "$COBCOPY"
```

4. 대상 디렉터리에 Makefile이 있으면 전처리 옵션, 링크 라이브러리, `-WC` 옵션, 산출물 이름을 우선 적용한다. 아래 기본 옵션으로 덮어쓰지 않는다.

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

파일 입출력 전처리를 사용하므로 `-ltcobfh`를 누락하지 않는다. 테스트가 AIM, NDB 등 추가 기능을 사용하면 원본 Makefile과 매뉴얼에 명시된 라이브러리만 추가한다. `PROGRAM-ID`와 JCL의 호출명에 맞춰 main program의 산출물 이름을 정한다. 서브 프로그램은 해당 테스트의 shared object 명명 규칙을 따른다.

여러 소스를 빌드할 때는 각 `ofcbppf` 성공을 확인한 뒤 해당 파일을 컴파일한다. 하나라도 실패하면 배치 단계로 넘어가지 않는다.

## 테스트 라이브러리에 배치

JCL의 `PRGLIB` 또는 테스트 Makefile에서 실제 목적지를 확인한다. 기본 프로그램 라이브러리 후보는 `$OPENFRAME_HOME/volume_DEFVOL/SYS1.COBLIB`이지만, 다른 volume이나 라이브러리를 임의로 같은 경로로 간주하지 않는다.

```bash
program_library="$OPENFRAME_HOME/volume_DEFVOL/SYS1.COBLIB"
test -d "$program_library"
mv -- TESTPGM "$program_library/TESTPGM"
test -s "$program_library/TESTPGM"
```

기존 모듈을 교체할 경우 대상이 해당 테스트의 모듈인지 확인한다. 실행 중인 시스템에 동적 반영이 필요하면 무조건 덮어쓰지 말고 `dlupdate` 적용 여부를 확인한다.

## 검증과 문제 해결

- 전처리 파일과 최종 shared object가 비어 있지 않은지 확인한다.
- 최종 모듈의 이름과 배치 경로가 JCL과 일치하는지 확인한다.
- 실행 테스트가 포함된 요청이면 `$batch-job-run`으로 JCL을 실행하여 STEP RC와 SPOOL까지 확인한다.
- 실패 시 컴파일러의 첫 오류, `$TMAXDIR/log`, `$OPENFRAME_HOME/log`를 확인하고 OpenFrame 오류는 `oferror <error-code>`로 조회한다.

명령이 불확실하면 다음 매뉴얼을 확인한다.

- `C:\Work\srcs\Tmax\manual\openframe_common\docs\modules\migration-guide\pages\mvs\sect-batch-application.adoc`
