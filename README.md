# OpenFrame 7 개발 에이전트 지침

OpenFrame 7 개발·빌드·테스트·문제 해결을 AI 개발 에이전트와 함께 수행하기 위한 작업 지침과 재사용 가능한 스킬을 관리하는 프로젝트입니다. 환경 선택부터 제품별 검증과 결과 보고까지 일관된 절차를 제공하는 것을 목적으로 합니다.

이 저장소의 주요 내용은 `AGENTS.md`, `SKILL.md`, 참조 문서와 템플릿입니다. 실제 제품 소스 수정·빌드·런타임 검증은 개인 환경 프로필이 가리키는 OpenFrame 개발 환경에서 수행합니다.

## 목적

- 공통 작업 원칙과 제품별 추가 지침을 명확히 연결합니다.
- 제품 버전·BATCH OS·DB·접속 방식에 따라 작업 환경을 선택합니다.
- Jira 확인, 설정 조회, COBOL 컴파일, 단위 테스트, 빌드·설치와 JOB 검증 절차를 재사용합니다.
- 코드·설정·로그·메타데이터에 근거해 원인을 분석하고, 변경 전후와 검증 결과를 보고합니다.
- 기존 작업을 보존하고 개인 설정과 인증 정보의 공유를 방지합니다.

## 저장소 구성

```text
openframe-codex/
├── AGENTS.md                         # 공통 작업 지침과 라우팅
├── openframe.local.yaml.sample       # 개인 환경 설정 예시
├── .agents/skills/                   # 공통 스킬과 빌드·설치 절차
├── base/
│   ├── AGENTS.md                     # Base 추가 지침
│   └── .agents/skills/               # TSAM 데이터셋·copybook 준비
├── batch/
│   ├── AGENTS.md                     # Batch 유틸리티 추가 지침
│   └── .agents/skills/               # JCL 작성과 JOB 실행·검증
├── ims/AGENTS.md                     # HiDB 추가 지침
└── ndb/AGENTS.md                     # NDB 추가 지침
```

`openframe.local.yaml`은 체크아웃마다 작성하는 개인 설정이며 Git에서 제외됩니다. 스킬별로 필요한 `references/`, `assets/`, `agents/`에는 상세 절차, 예제·템플릿, 에이전트 표시 정보가 들어 있습니다.

## 지침과 스킬

[루트 AGENTS.md](AGENTS.md)는 환경 선택, Git 작업, 인증 정보 보호, 검증과 보고의 공통 기준입니다. 제품별 문서는 해당 범위의 추가 기준을 제공합니다.

| 제품 | 제품 소스에서의 적용 범위 | 주요 내용 |
| --- | --- | --- |
| [Base](base/AGENTS.md) | `$SOURCE_BASE/base/**` | 데이터셋·I/O·할당·공용 API·도구의 변경과 검증 |
| [Batch 유틸리티](batch/AGENTS.md) | `$SOURCE_BASE/batch/util/**` | JCL·종료 코드·SPOOL·데이터 부작용 검증 |
| [HiDB](ims/AGENTS.md) | `$SOURCE_BASE/ims/**` | IMS DB 대응 기능과 인터페이스·연동 검증 |
| [NDB](ndb/AGENTS.md) | `$SOURCE_BASE/ndb/**` | AIM DB·XDM/SD DB 대응 기능, SQL·정합성·성능 검증 |

작업할 때는 실제 소스 경로에 더 가까운 `AGENTS.md`도 확인합니다. `batch/AGENTS.md`의 제품별 추가 규칙은 `batch/util/**`에 한정됩니다.

| 작업 | 스킬 |
| --- | --- |
| OpenFrame 7 소스 빌드·바이너리 설치 | [openframe7-build](.agents/skills/openframe7-build/SKILL.md) |
| 신규 환경 준비·DB 초기화·서비스 검증 | [openframe7-setup](.agents/skills/openframe7-setup/SKILL.md) |
| TmaxSoft Jira 이슈 읽기 | [common-tmaxsoft-jira-read](.agents/skills/common-tmaxsoft-jira-read/SKILL.md) |
| OpenFrame 설정 조회·임시 변경·복원 | [common-ofconfig-manage](.agents/skills/common-ofconfig-manage/SKILL.md) |
| NetCOBOL 테스트 모듈 전처리·컴파일·배치 | [common-netcobol-compile](.agents/skills/common-netcobol-compile/SKILL.md) |
| OpenFrame COBOL 테스트 모듈 컴파일·배치 | [common-ofcobol-compile](.agents/skills/common-ofcobol-compile/SKILL.md) |
| C/C++ GoogleTest 작성·실행·gcov 검증 | [common-unit-test-write](.agents/skills/common-unit-test-write/SKILL.md) |
| TSAM 데이터셋·copybook 준비·정리 | [base-tsam-create](base/.agents/skills/base-tsam-create/SKILL.md) |
| MVS·MSP·VOS3 JCL 작성·수정 | [jcl-write-run-general](batch/.agents/skills/jcl-write-run-general/SKILL.md) |
| XSP JCL 작성·변환·수정 | [jcl-write-run-xsp](batch/.agents/skills/jcl-write-run-xsp/SKILL.md) |
| JOB 제출·상태·STEP RC·SPOOL 검증 | [batch-job-run](batch/.agents/skills/batch-job-run/SKILL.md) |

관련 작업을 시작하기 전에 해당 `SKILL.md` 전체와 필요한 참조 문서를 읽습니다. JCL을 실제 제출하면 OS별 작성 스킬과 `batch-job-run`을 함께 적용합니다. COBOL 컴파일러는 현재 노드의 `ofsys.COMPILER.COBOL_COMPILER` 설정에서 `VALUE`를 확인한 뒤 선택합니다.

## 시작하기

1. [AGENTS.md](AGENTS.md)를 읽고 작업 범위와 필요한 스킬을 확인합니다.
2. [설정 샘플](openframe.local.yaml.sample)을 루트의 `openframe.local.yaml`로 복사합니다. 기존 개인 설정이 있으면 덮어쓰지 않습니다.
3. 자신의 매뉴얼 경로, 환경 파일 경로와 접속 정보를 입력하고 `default_environment`를 지정합니다. 샘플 값은 실제 접속에 사용하지 않습니다.
4. 사용하는 AI 개발 에이전트에 환경과 작업을 요청합니다. 분석, 수정 제안, 구현, 빌드·테스트 중 필요한 범위를 명시합니다.

### 환경 프로필

`environments`는 환경 이름을 고유 키로 사용하는 맵입니다. 각 프로필에는 `type`, `env_path`, 제품 `version`, `os`, `rdb`를 정의합니다. 최상위 `version`은 설정 스키마 버전이며 제품 버전과 구분됩니다.

| 구분 | 선택 가능 값 또는 역할 |
| --- | --- |
| 제품 버전·DB | 7.3: Tibero / 7.4: Tibero 또는 Oracle |
| BATCH OS | MVS, MSP, XSP, VOS3 |
| `local` | 로컬 호스트에서 직접 작업 |
| `remote` | SSH 대상 서버에서 직접 작업 |
| `remote_container` | SSH 대상 서버의 지정 Docker 컨테이너에서 작업 |
| `env_path` | 작업 환경에서 적용할 환경 파일 |

위 항목은 프로필과 절차의 선택 기준입니다. 실제 사용 가능한 조합과 빌드·검증 조건은 해당 환경 및 버전별 참조 문서에서 확인합니다.

원격 프로필에는 SSH 대상, 인증 정보와 RDB 접속 문자열이 추가로 필요하며, `remote_container`에는 컨테이너 이름도 필요합니다. 소스·설치 루트는 YAML에 중복 정의하지 않고 `env_path` 적용 후 `$SOURCE_BASE`, `$OPENFRAME_HOME`에서 읽습니다.

환경 이름을 지정하면 해당 이름을 선택하고, 속성만 지정하면 명시한 모든 조건이 일치하는 프로필을 찾습니다. 후보가 여러 개이거나 없으면 사용자에게 확인합니다. 이름과 속성을 모두 생략했을 때만 기본 환경을 선택합니다.

제품 작업에는 OpenFrame 소스·설치 환경, Tmax, 선택한 RDBMS 및 작업별 도구가 필요합니다. 원격 접속에는 해당 호스트에서 유효한 인증과 권한이 필요합니다. 버전별 빌드 구조와 도구 요건은 `openframe7-build`의 참조 문서를 따릅니다.

### 요청 예시

아래 환경 이름은 설정 샘플의 예시입니다. 실제 요청에는 자신의 프로필 이름을 사용합니다.

```text
of73msp 환경에서 지정한 Jira 이슈의 원인을 분석해줘. 소스 수정은 하지 마.

7.3 MVS 환경에서 지정한 C/C++ 이슈를 수정하고 단위 테스트와 gcov를 검증해줘.

of74mvs_ora_container 환경에서 OpenFrame 7.4 빌드와 설치 검증을 수행해줘.

이 저장소의 지침과 스킬만 수정해줘. 제품 환경 변경은 하지 마.
```

## 기본 작업 흐름

1. **대상 확인**: 현재 브랜치와 작업 트리를 확인하고, 기존 변경을 보존한 채 `git pull --ff-only`를 시도합니다. 실패하면 원인과 원격 변경 반영 여부를 보고합니다.
2. **환경 확인**: 요청에 맞는 프로필을 선택하고 실제 실행 위치에서 `env_path`와 소스·설치 루트를 검증합니다. 지침·스킬·샘플·템플릿 관리 작업은 지정된 체크아웃에서 정적으로 검증합니다.
3. **근거 조사**: 관련 제품 지침과 스킬을 적용하고 이슈 내용, 현재 코드, 설정, 메타데이터와 로그를 대조합니다.
4. **변경과 검증**: 요청 범위의 파일만 수정하고 관련 빌드·테스트를 수행합니다. Jira C/C++ 수정에는 단위 테스트와 gcov 검증을 이어가며, JOB 실행은 상태·각 STEP RC·SPOOL·업무 결과를 함께 확인합니다.
5. **결과 보고**: `AS-IS → TO-BE`로 차이와 영향 범위를 설명하고 실행 명령, 성공·실패 결과 및 미검증 항목을 구분합니다. 실행한 JOB은 재실행·중단 건도 포함해 기록합니다.

Jira 소스 수정에서는 원래 브랜치에서 pull을 시도한 뒤 `<원래 브랜치>-<Jira ID>` 형식의 작업 브랜치를 사용합니다. 분석·제안 요청의 범위는 구현이나 상태를 변경하는 테스트로 확대하지 않습니다. 커밋은 사용자가 명시적으로 지시한 경우에만 생성합니다.

## 관리 원칙

- 개인 `openframe.local.yaml`, 실제 인증 정보와 개인 환경값은 공유 문서·샘플·템플릿에 포함하지 않습니다. 비밀번호는 명령 인자, URL, 환경 변수, 임시 파일이나 로그에 노출하지 않습니다.
- 지침과 스킬에는 재사용 가능한 절차와 판단 기준을 기록합니다. 특정 세션의 브랜치·JOB ID·실행 결과·우회 이력은 세션 보고와 구분합니다.
- 관찰한 사실, 코드·메타데이터에서 도출한 추론, 예상 효과와 미확인 사항을 구분합니다.
- 기존 변경과 무관한 파일을 보존하고 요청 범위에 필요한 최소 변경만 수행합니다.

구체적인 실행 기준은 루트·제품별 `AGENTS.md`와 해당 스킬 문서를 따릅니다.
