# 제품별 빌드 상세

## 저장소와 브랜치

저장소 루트: http://192.168.51.106/openframe/openframe7

| 제품 | 브랜치 | 빌드 대상 |
|---|---|---|
| base | rb_73 | 공통 |
| batch | rb_73_FS2 | 공통 |
| tacf | rb_73 | 공통 |
| ims | rb_72 | MVS |
| ndb | rb_73 | MSP/XSP AIM DB. VOS3 XDM/SD 브랜치는 실제 예제로 확인 |
| aim | rb_73 | MSP/XSP |
| dev | 서버 기본 브랜치 확인(실측 master) | oflicgen |

요청된 소스는 base.git, batch.git, tacf.git, ims.git, ndb.git, aim.git, dev.git이다. 선택 OS에 불필요한 제품은 클론과 빌드를 구분한다. MVS HiDB의 추가 의존성은 osi.git rb_72 헤더이며 OSI는 빌드하지 않는다.

필요한 보조 제품이 없는 경우 사용자가 허용한 빌드 소스는 같은 루트의 prosort.git, ofcobol.git, ofcbpph.git이다. 예제 검증은 이미 설치된 ProSort/OFCOBOL을 사용했다. 보조 제품 신규 빌드는 별도 실제 절차를 확인하며 미검증 명령을 성공 사례로 쓰지 않는다.

## 실제 디렉터리와 로컬 설정

현재 Base/HiDB는 `src/` 계층을 사용한다. 오래된 `base/ds`, `base/server`, `base/tool` 경로를 고정하지 말고 find/rg로 실제 경로를 확인한다.

| 제품 | ofrelease.sh | cflags.local 원본 |
|---|---|---|
| base | scripts/ofrelease.sh | make/cflags.linux-x86_64 |
| batch | ofrelease.sh | make/cflags.linux-x86_64 |
| tacf | ofrelease.sh | make/cflags.linux-x86_64 |
| ims | scripts/ofrelease.sh | make/cflags.linux64 |
| ndb | ofrelease.sh | make/cflags.linux64 |
| aim | script/ofrelease.sh | make/cflags.linux64 |
| dev | ofrelease.sh | make/cflags.linux64 |

제품 루트에서 스크립트를 실행한다. ofrelease.sh가 HOME/bin의 dist/patch 스크립트를 덮어쓰는지 제품별로 확인하고, 기존 파일을 보존한 뒤 성공·실패 여부와 무관하게 복원한다. 새 경로의 build_info 헤더는 해당 스크립트로 생성한다.

MVS + Tibero 7 예제에서 확인한 config.local의 활성 값:

```makefile
# base
SORT_ENGINE_SELECT='PROSORT'
COBOL_SWITCH_OFCOBOL='YES'
DATABASE_SWITCH_TIBERO='YES'
JCL_SWITCH_MVS='YES'
TIBERO_VERSION=7
RDBII_SELECT='TIBERO'
# batch
BATCH_OS_TYPE='MVS'
COBOL_COMPILER='OFCOBOL'
OPTION_PLILIB='N.A.'
DBDUMP_UTILITY_SELECT='TIBERO'
SORT_ENGINE_SELECT='PROSORT'
BATCH_USE_TSCMDSVR='NO'
DB_CONNECT_TYPE_SELECT='UNIXODBC'
DB2HPU_UTILITY_SELECT='ESQL'
TIBERO_VERSION=7
# tacf
TACF_TCONFIG_SELECT='WITH_BATCH'
# ims
COBOL_COMPILER='OFCOBOL'
DATABASE_SELECT='TIBERO_ESQL'
```

상호 배타적 옵션과 @...@ 대체 값을 남기지 않는다. 위 목록 외 옵션은 해당 config.sample과 정상 예제의 차이를 확인한다. 다른 OS에는 해당 BATCH_OS_TYPE/JCL 스위치를 적용하고 NDB OS 옵션도 실제 예제로 확인한다.

Batch rb_73_FS2의 GCC 10+ 빌드는 예제처럼 cflags.local에 `CFLAGS_COMMON += -fcommon`이 필요하다. 최신 플랫폼 cflags의 링크 옵션은 유지한다.

### XSP 옵션과 순서

실측 XSP는 Base → Batch → TACF → NDB → AIM 순서로 `make install`에 성공했다. Base의 JCL 스위치는 `JCL_SWITCH_XSP='YES'`, Batch는 `BATCH_OS_TYPE='XSP'`로 선택하고 다른 OS 스위치를 함께 활성화하지 않는다. NDB/AIM은 `SORT_ENGINE_SELECT='PROSORT'`, `COBOL_COMPILER='OFCOBOL'`, NDB는 추가로 `USE_DB_SELECT='AIM_DB'`를 사용했다. 나머지 공통 옵션은 위 예제와 현재 sample을 비교한다. config.sample의 `NETCOBOL` 기본값을 그대로 두지 말고 실제 컴파일러와 맞춘다.

## Tibero 전처리

cfg 이름이 반드시 tibero.cfg인 것은 아니다. 실제 sample을 복사하고 GCC include 한 줄을 `/usr/bin/gcc -print-file-name=include`의 존재하는 절대 경로로 바꾼다. 다른 include 값은 예제와 비교해 유지한다.

- base/src/ds/tsam/tsam_tibero.cfg.sample
- base/src/tdbconnsw/tdbconn_{tbr,tboci,tbodbc}/tbpc.cfg.sample
- batch/util/mvs/{dsntb,tbunload,inztb}/tbpc.cfg.sample
- ims/config/tibero.cfg.sample
- 설치 scripts의 tsam_tibero.cfg.sample 및 hidb_tibero.cfg.sample
- NDB는 실제 cfg.sample 검색 결과와 해당 예제를 사용한다.

실측 RHEL 9 GCC 경로는 `/usr/lib/gcc/x86_64-redhat-linux/11/include`이다. 재사용 환경에서 이 값을 고정하지 않는다.

Base 루트에는 `precomp` target이 없을 수 있다. 실측 대상은 `base/src/ds/tsam`, `base/src/tdbconnsw/tdbconn_{tbr,tboci,tbodbc}`였다. NDB는 `config/tibero.cfg`를 준비한 뒤 `src/common/common`, `src/db/dml`, `src/meta`의 `make precomp`를 수행했다.

NDB rb_73 실측에서는 `src/db/rstd`의 오래된 `precomp`가 없는 `ndb_rstd.pc`를 참조했다. 현재 SOURCES의 `ndb_rstd.c`가 일반 C 소스인지, 남은 `.pc`가 실제 빌드 대상인지 먼저 확인한다. 전처리할 입력이 없는 target은 미적용 사유와 실패 로그를 남기고 실제 빌드로 검증한다. 이를 전처리 성공으로 기록하거나 가짜 입력을 만들지 않는다. 필요한 입력이 누락된 경우에는 빌드를 중단하고 확인한다.

## 라이선스와 최종 Tmax 설정

`make -C "$SOURCE_BASE/dev/tool/oflicgen" INSTALL_DIR="$OPENFRAME_HOME/bin"`으로 HOME/bin 기존 도구 덮어쓰기를 피할 수 있다. Base의 liboflic이 먼저 필요하다.

OPENFRAME_HOME/license에서 `oflicgen -c -T <product> -N -R -H "$(hostname)"`을 실행한다. 공통은 Base/Batch/TACF, MVS는 HIDB, MSP/XSP는 AIM/NDB2, VOS3는 NDB2를 추가한다. 제품명과 실제 생성 파일을 확인한다.

TMAXDIR/config의 가장 최근 tmconfig_*를 확인한 뒤 tmconfig로 복사한다. 현재 빌드에는 build/tmax/config 산출물도 있으므로 설치 위치를 확인한다. MVS 최종 파일은 실측 tmconfig_ims였다. 잘못된 제품 또는 오래된 설정을 선택하지 않도록 생성시각과 포함 서버를 함께 검증한다.

XSP 실측의 최종 설정은 `tmconfig_aim`이었다. NDB 설정은 `ndb/build/tmax/config`, AIM 설정은 `aim/build`에서 생성됐다. AIM의 설치 target이 `$TMAXDIR/config/tmconfig_aim`을 실제 생성했는지 확인한다.
