# 제품별 OFCOBOL 컴파일 확인

경로는 `$SOURCE_BASE` 기준이다. 스크립트는 버전·체크아웃마다 다시 읽고 현재 환경과 대조한다. 아래 패턴은 제품별 선택 기준이며 전체 스크립트 실행을 지시하지 않는다.

## TSAM

`base/src/ds/tsam/test/Makefile`의 `mvs`, `msp`, `xsp`, `vos`, `copybook` target을 확인한다.

- MVS는 기본 `ofcob`와 `-L$OPENFRAME_HOME/lib -ltextfh`를 사용한다.
- MSP/XSP는 `--enable-XSP`, VOS3는 `--enable-VOS`를 사용한다.
- OS target에는 여러 모듈 컴파일 뒤 `SYS1.COBLIB`로 이동하고 `SYS1.JCLLIB`에 JCL을 복사하는 단계가 포함된다. `copybook` target도 설치 경로에 파일을 쓴다.
- 출력명은 소스 파일명과 다를 수 있다. 예를 들어 `TSAM.TEST.WRITE.cob`의 산출물은 `TSWRITE`다. XSP는 `jcl/xsp`, 다른 OS는 `jcl/others`를 대조한다.

## HiDB

다음 경로의 실제 컴파일 분기와 배치 동작을 확인한다.

- `ims/src/hidb/test/GSAM_TEST/{BSAM,BSAM_VB,ESDS}/COB/compile.sh`
- 각 GSAM 테스트의 `CHKP_TEST/COB/compile.sh`
- `ims/src/hidb/test/SHISAM_TEST/COB/comp.sh`
- `ims/src/hidb/test/SHISAM_TEST/CHKP_TEST/COB/compile.sh`

`ofsys.COMPILER.COBOL_COMPILER`의 `VALUE=OFCOBOL`일 때 다음 OFCOBOL 옵션을 적용한다. `libcobsw.so`가 `libswofcob.so`를 가리키는지 확인하되 컴파일을 통과시키려고 공용 링크를 임의 변경하지 않는다.

OFCOBOL 분기는 `OFCOBCPY`에 테스트 copybook 디렉터리를 추가하고 다음 패턴을 사용한다.

```bash
ofcob -o TESTPGM.so -g -U TESTPGM.cob \
  -L"$OPENFRAME_HOME/lib" -ltextfh -lhidb \
  --enable-cbltdli --notrunc
```

`--check-index`는 지원 여부를 조회하는 스크립트가 있으므로 설치된 도움말을 확인한다. `--force-trace`도 해당 테스트에 필요하고 지원될 때만 사용한다. HiDB 예제는 MVS 방식이며 다른 OS에서 스크립트가 존재한다는 사실만으로 해당 OS runtime 지원을 단정하지 않는다.

SHISAM `comp.sh`는 `ofcbpp -i ... -o ... -copypath ...` 후 `.cbl`을 컴파일한다. 하드코딩된 `CPY_PATH`를 현재 copybook 위치로 확인한다. GSAM은 `dlupdate` 후 산출물을 삭제하고, SHISAM도 `dlupdate`를 수행하므로 컴파일만 확인할 때는 필요한 단계만 분리한다.

## NDB

### Fujitsu AIM 계열

컴파일러는 `ofsys.COMPILER.COBOL_COMPILER`의 현재 `VALUE`로 선택한다. `OFCOBOL`이면 Fujitsu 소스·copybook·AIM API 의존성을 확인하고 `ofcob --enable-XSP`로 컴파일한다. `NETCOBOL`이면 `common-netcobol-compile`을 적용한다. NetCOBOL의 `-WC` 옵션을 OFCOBOL에 그대로 넘기지 않는다.

NDB 테스트 준비 절차에는 ADL 저장, schema drop/create/dmlgen/truncate, `subsc`, 배치가 포함될 수 있다. 컴파일 요청만으로 DB 초기화·데이터 삭제·스키마 생성을 실행하지 않는다. copybook·subschema 생성이 필요하면 이미 생성된 파일과 요청 범위를 먼저 확인한다.

### Hitachi XDM/SD 계열

`ndb/src/db/dml/test/XDM_SD/*/compile.sh`를 확인한다. 대표 OFCOBOL 명령은 `-U`, `--enable-VOS`, OpenFrame의 `-ltextfh -lconcli -ltsys -lxdmtondb`, Tmax의 `-L$TMAXDIR/lib -lcli`를 사용한다. VOS 전처리 전제는 이 스킬의 VOS3 검증을 동일하게 적용한다.

- Tibero client: `-L$TB_HOME/client/lib -ltscli`
- Oracle client: `-L$ORACLE_LIB_HOME -lclntsh`

7.3 스크립트는 Tibero 링크를 사용한다. 7.4 스크립트는 `ndb/make/config.local`의 `DATABASE_SWITCH_ORACLE`을 읽어 분기할 수 있다. 프로필 `rdb`와 실제 분기·라이브러리가 일치하는지 확인한다. `config.local`의 위치는 현재 스크립트에서 확인하며, 불일치를 요청 없이 수정하지 않는다.

출력 `.so`를 다른 호출명으로 복사한 후 `SYS1.USERLIB`에 배치하는 스크립트가 있으므로 `PROGRAM-ID`, 외부 호출명과 최종 멤버명을 함께 확인한다. 이 Hitachi 전용 패턴을 MVS·MSP·XSP의 일반 NDB 컴파일 옵션으로 사용하지 않는다.
