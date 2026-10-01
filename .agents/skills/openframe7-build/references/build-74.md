# OpenFrame 7.4 통합 저장소 빌드

7.4 프로필에서만 적용한다. 공통 접속·설치·충돌 확인·검증은 기존 빌드/설치 스킬을 따른다. 7.3의 제품별 브랜치와 release 명령을 적용하지 않는다.

## 소스 준비

- 저장소: `http://192.168.51.106/openframe/openframe7/ofsrc.git`.
- env_path를 적용한 뒤 SOURCE_BASE가 지정된 신규 경로인지 확인하고 그 경로에 클론한다. SOURCE_BASE는 base의 부모인 통합 저장소 루트다.
- 지정 브랜치가 없으면 서버 기본 브랜치와 product_info의 7.4 버전을 확인한다. 기존 참고 설치본의 브랜치를 임의로 강제하지 않는다. 브랜치/커밋을 기록한다.
- 기존 경로는 덮어쓰지 않는다. 기존 저장소는 상태 확인 및 `git pull --ff-only`를 먼저 시도한다.
- 실제 트리의 AGENTS.md와 Makefile, ofconfigure.sh, ofrelease.sh, config.sample을 읽는다.

## 설정과 빌드

참고 7.4 트리에는 루트 ofconfigure.sh와 OS별 순서를 담당하는 Makefile이 있다. 체크아웃의 실제 옵션과 대상을 다시 확인한다.

- `ofconfigure.sh --os <소문자 os>` 및 `--set KEY=VALUE`로 실제 DB/COBOL/SORT를 선택한다. 선택할 값은 현재 config.sample에서 확인하며 미해결 토큰이나 사용하지 않는 컴파일러·DB 스위치를 남기지 않는다. 기존 설정을 다시 생성할 때 변경 내용과 백업을 확인한다.
- DB는 프로필 rdb에 맞춘다. Tibero의 tbpc cfg, Oracle의 proc cfg와 include/라이브러리 경로는 설치된 클라이언트에 맞춰 검증한다. 7.3 전처리 대상 목록을 7.4에 그대로 강제하지 않는다.
- 저장소 최상단에서 `./ofrelease.sh`를 실행한다. 제품별 release 스크립트를 직접 반복 실행하지 않는다. 루트 product_info와 모듈 module_info를 기준으로 생성한다.
- 필요한 실제 전처리 입력/target을 확인해 `make precomp`를 수행하며 실패하면 수정 후 재실행한다.
- 루트 `make list OS=<소문자 os>`로 대상을 확인하고 `make install OS=<소문자 os>`를 실행한다. 루트 Makefile이 없으면 실제 체크아웃의 제품 의존 순서를 확인한다.
- 사용자가 제품 제외를 명시하면 실제 Makefile의 COMPONENTS override 지원을 확인하고 그 범위에 한정한다. 이번 MVS 실측의 OSC 제외는 `COMPONENTS="base batch tacf ims osi"`였다. 이를 모든 7.4 환경의 기본 제외로 만들지 않는다.
- 참고 트리의 MVS 대상은 base/batch/tacf/ims/osc/osi다. 7.3의 OSI 헤더 전용 규칙과 다르다. MSP/XSP의 Batch/AIM 순환 의존 처리도 루트 Makefile을 우선한다.
- 라이선스 생성 도구의 실제 경로와 지원 제품을 확인한다. DB 초기화와 서비스·데이터셋·JOB·TSAM·HiDB 검증은 openframe7-setup으로 이어간다.

Oracle 참고 환경은 옵션과 구조의 참고 자료다. Tibero 선택 시 Oracle 접속값이나 client 경로를 그대로 복사하지 않는다. 참고 설치본의 수정 파일·서버·DB는 보존한다. 설치/실행 검증 전 신규 스키마 여부와 모든 충돌을 다시 확인한다.

## 2026-10-01 실측에서 확인한 설정 보완

- rb_74 `b5ce2ef69837a1af325d332cfa7f3a8593d95bdc`, MVS/Tibero 7, GCC 11에서 확인했다. 다른 커밋의 지원 여부는 현재 파일로 재검증한다.
- ofconfigure.sh의 `--set TIBERO_VERSION=7`은 샘플 후보에 6만 있어서 반영되지 않았다. 생성 후 base/batch의 config.local에 `TIBERO_VERSION=7`이 활성화되었는지 확인하고 필요하면 해당 로컬 파일만 보완한다. `libtdbconntbr7` 설치도 확인한다.
- HiDB config.sample의 기본 `COBOL_COMPILER_SELECT='MFCOBOL'`은 OFCOBOL을 사용하는 환경에 맞지 않았다. 실제 Makefile 사용 변수와 설치된 컴파일러를 확인한 뒤 config.local의 해당 값만 조정했다.
- dev는 MVS 기본 configure 대상에서 빠지므로 oflicgen 빌드 전 루트 `ofconfigure.sh --platform linux-x86_64 --only dev`와 `ofrelease.sh dev`로 설정을 준비했다.
- Tmax tar에는 lib64만 있었으므로 신규 TMAXDIR에 `lib -> lib64` 링크가 필요했다. license/log 디렉터리도 실제 배포본을 확인해 생성한다.
- OSC는 Tmax5.0SP2Fix4 헤더에 TDL_RTLD_GLOBAL이 없어 실패했다. 사용자 지시로 제외했으며 제품 코드를 수정하지 않았다. OSI는 빌드·설치에 성공했다.
- 최종 루트 `make all OS=mvs COMPONENTS="base batch tacf ims osi"` 성공. Base/Batch/TACF/HiDB/OSI 설치 성공. 실행 검증 범위는 [7.4 설치 실측](../../openframe7-setup/references/validation-74.md)을 따른다.

## rc1 MSP/XSP/VOS3 빌드에서 확인한 추가 보완

- `--set`이 생성 config.local의 활성 값에 반영되었는지 확인한다. rc1의 Base/Batch에는 TIBERO_VERSION 활성 줄이 없고 NDB COBOL_COMPILER는 토큰이 아닌 NETCOBOL 값이어서 옵션 전달만으로 원하는 설정이 되지 않았다. 신규 로컬 파일에 Tibero 7/OFCOBOL 값을 보완한 후 최종 빌드를 다시 수행한다.
- DB 드라이버는 설치된 libtdbconnsw.so의 실제 링크 대상으로 확인한다. 라이브러리는 `.so.64.7_4_0_0_0` 같은 버전 접미사를 사용하므로 고정 `.so` 파일명만 검사하지 않는다.
- dev/make/rules.tool은 파일 target에도 설치 복사를 포함하며 기본 INSTALL_DIR은 HOME/bin이다. 기존 공용 도구를 보존하려면 `make -C dev/tool/oflicgen INSTALL_DIR="$OPENFRAME_HOME/bin"`으로 설치 경로를 지정한다. 실제 세 환경에서 이 변수 지정 후 재빌드하여 복사 경로와 기존 MVS 바이너리 보존을 확인했다. 파일 target 선택이나 BIN_DIR 지정만으로 설치 복사가 생략된다고 가정하지 않는다.
- OS별 최종 빌드·전처리·링크 검사와 실행 검증 결과와 미검증 범위는 [7.4 MSP/XSP/VOS3 실측](../../openframe7-setup/references/validation-74-non-mvs.md)을 참고한다.
