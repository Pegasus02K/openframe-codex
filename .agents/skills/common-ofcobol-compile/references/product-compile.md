# 제품별 OFCOBOL 컴파일 확인

경로는 `$SOURCE_BASE` 기준이다. 스크립트는 버전·체크아웃마다 다시 읽고 현재 환경과 대조한다. 아래 항목은 제품별 확인 기준이며 전체 스크립트 실행을 지시하지 않는다. 내부 코드나 스크립트의 처리 내용을 이 참조에 옮기지 않는다.

## TSAM

`base/src/ds/tsam/test/Makefile`의 `mvs`, `msp`, `xsp`, `vos`, `copybook` target을 확인한다.

- MVS의 기본 예제는 `ofcob`와 `-L$OPENFRAME_HOME/lib -ltextfh`를 사용한다. 실제 링크 옵션은 현재 Makefile·매뉴얼과 설치본에서 확인한다.
- MSP/XSP는 `--enable-XSP`, VOS3는 `--enable-VOS`를 사용한다.
- OS·copybook target의 컴파일 범위와 프로그램·JCL·copybook 배치 또는 삭제 여부를 현재 Makefile에서 확인한다. 컴파일만 요청받았으면 배치와 삭제 단계를 실행하지 않는다.
- 소스의 `PROGRAM-ID`, 산출물 이름과 JCL 호출명을 대조한다. XSP는 `jcl/xsp`, 다른 OS는 `jcl/others`를 확인한다.

## HiDB

다음 경로의 실제 컴파일 분기와 배치 동작을 확인한다.

- `ims/src/hidb/test/GSAM_TEST/{BSAM,BSAM_VB,ESDS}/COB/compile.sh`
- 각 GSAM 테스트의 `CHKP_TEST/COB/compile.sh`
- `ims/src/hidb/test/SHISAM_TEST/COB/comp.sh`
- `ims/src/hidb/test/SHISAM_TEST/CHKP_TEST/COB/compile.sh`

`ofsys.COMPILER.COBOL_COMPILER`의 `VALUE=OFCOBOL`일 때 현재 스크립트·매뉴얼에서 전처리 도구, `OFCOBCPY`, 출력 형식, 호출 인터페이스와 링크 옵션을 확인한다. `libcobsw.so`의 실제 대상이 `libswofcob.so` 등 선택한 OFCOBOL 설치본의 런타임과 일치하는지 확인하되, 컴파일을 통과시키려고 공용 링크를 임의 변경하지 않는다.

공개된 `textfh`, `hidb` 라이브러리를 사용하는 HiDB 컴파일 예제는 다음과 같다. `OFCOBCPY`의 테스트 copybook 경로와 현재 OS·설치본의 옵션을 확인한 뒤 적용한다.

```bash
ofcob -o TESTPGM.so -g -U TESTPGM.cob \
  -L"$OPENFRAME_HOME/lib" -ltextfh -lhidb \
  --enable-cbltdli --notrunc
```

`-U`, `--enable-cbltdli`, `--notrunc`, `--check-index`, `--force-trace` 등은 해당 테스트의 요구사항과 설치된 도움말에서 필요성과 지원 여부를 확인한 뒤 사용한다. 다른 OS에서 스크립트가 존재한다는 사실만으로 해당 OS runtime 지원을 단정하지 않는다.

copybook 경로, 전처리 입력·출력과 컴파일 입력을 대조한다. 설치·동적 반영·생성 파일 삭제가 포함되는지 먼저 확인하고, 컴파일만 요청받았으면 해당 단계만 분리한다.

## NDB

### Fujitsu AIM 계열

컴파일러는 `ofsys.COMPILER.COBOL_COMPILER`의 현재 `VALUE`로 선택한다. `OFCOBOL`이면 Fujitsu 소스·copybook·AIM API 의존성을 확인하고 `ofcob --enable-XSP`로 컴파일한다. `NETCOBOL`이면 `common-netcobol-compile`을 적용한다. NetCOBOL의 `-WC` 옵션을 OFCOBOL에 그대로 넘기지 않는다.

NDB 테스트 준비 절차에는 ADL 저장, schema drop/create/dmlgen/truncate, `subsc`, 배치가 포함될 수 있다. 컴파일 요청만으로 DB 초기화·데이터 삭제·스키마 생성을 실행하지 않는다. copybook·subschema 생성이 필요하면 이미 생성된 파일과 요청 범위를 먼저 확인한다.

### Hitachi XDM/SD 계열

`ndb/src/db/dml/test/XDM_SD/*/compile.sh`를 확인한다. OFCOBOL 명령 예제는 `-U`, `--enable-VOS`, OpenFrame의 `-L$OPENFRAME_HOME/lib -ltextfh -lconcli -ltsys -lxdmtondb`, Tmax의 `-L$TMAXDIR/lib -lcli`를 사용한다. 공개된 아티팩트 이름과 `base/tsys` API 사용 정보이며, 필요한 옵션과 라이브러리는 현재 스크립트·매뉴얼 및 설치본에서 검증한다. VOS 전처리 전제는 이 스킬의 VOS3 검증을 동일하게 적용한다.

- Tibero client: `-L$TB_HOME/client/lib -ltscli`
- Oracle client: `-L$ORACLE_LIB_HOME -lclntsh`

7.3은 Tibero, 7.4는 프로필의 `rdb`에 맞는 client를 사용한다. `ndb/make/config.local` 등 현재 빌드 설정의 `DATABASE_SWITCH_ORACLE` 값과 실제 링크 대상이 선택 DB와 일치하는지 확인한다. 설정 파일의 위치는 현재 스크립트에서 확인하며, 불일치를 요청 없이 수정하지 않는다.

`PROGRAM-ID`, 외부 호출명, 산출물 이름과 최종 라이브러리 멤버명을 함께 확인한다. 호출명에 맞춘 복사·배치가 필요한지는 현재 스크립트와 JCL에서 확인한다. Hitachi 전용 옵션을 MVS·MSP·XSP의 일반 NDB 컴파일 옵션으로 사용하지 않는다.
