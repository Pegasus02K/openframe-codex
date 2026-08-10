---
name: common-ofconfig-manage
description: OpenFrame 테스트 중 환경설정을 ofconfig list로 조회하고, 필요한 기존 설정을 ofconfig update로 최소 범위에서 임시 변경한 뒤 성공·실패·중단 여부와 관계없이 원래 값으로 복원한다. 테스트 준비나 장애 분석에서 설정 확인, 상세 값·허용 범위 조회, 가역적 설정 변경 또는 rollback 검증이 필요할 때 반드시 사용한다.
---

# OpenFrame 설정 조회 및 가역 변경

`ofconfig`로 현재 노드의 설정을 확인하고, 테스트에 꼭 필요한 기존 키만 임시 변경한 뒤 원래 상태로 복원한다. 읽기만으로 목적을 달성할 수 있으면 설정을 변경하지 않는다.

## 기본 원칙

1. 루트 `AGENTS.md`와 `.agents/openframe.local.yaml`에서 선택한 환경 프로필로 진입한다.
2. 소스 작업은 선택한 환경의 `{{source_base}}`에서 수행한다.
3. 노드 이름을 추측하거나 `NODE1`로 고정하지 않는다. 모든 `list`, `update` 명령의 `-n`에는 `$OPENFRAME_NODENAME`을 사용한다.
4. subject, section, key 이름은 관련 제품의 configuration guide나 기존 설정 조회로 확인한다. 비슷해 보이는 이름을 추측하지 않는다.
5. 테스트용 변경은 기존 키의 `VALUE`에 한정한다. `add`, `delete`, `truncate`, `import` 또는 DB 직접 수정은 이 스킬의 기본 작업 범위가 아니다.
6. 변경 전에 반드시 원래 값을 확보하고 복원 가능성을 검증한다. 복원 계획이 없거나 원래 상태를 정확히 재현할 수 없으면 변경하지 않는다.
7. 설정값에 비밀번호, 토큰 또는 기타 비밀이 포함될 수 있다. 필요한 키만 좁게 조회하고 민감한 값은 사용자 보고나 로그에 노출하지 않는다.

환경을 먼저 확인한다.

```bash
pwd
test -n "$OPENFRAME_HOME"
node_name="${OPENFRAME_NODENAME:?OPENFRAME_NODENAME is not set}"
command -v ofconfig
```

환경변수나 `ofconfig`가 없으면 임의 값을 대입하지 말고 OpenFrame 환경을 올바르게 로드한 후 다시 확인한다.

## 명령 명세

설정 키는 `NODE / SUBJECT / SECTION / KEY` 계층으로 식별한다.

`list`는 `-n`만 지정해 노드 전체를 조회할 수 있고, `-s`, `-sec`, `-k`를 차례로 더해 범위를 좁힐 수 있다. `-l`은 다음 상세 정보를 표시한다.

- `TYPE`: 값 유형
- `DEFAULT_VALUE`: 기본값
- `VALUE`: 현재 저장값
- `AVAIL_VALUE`: 허용 값 또는 숫자 범위
- `DESCRIPTION`: 설정 설명

```bash
ofconfig list -n "$node_name"
ofconfig list -n "$node_name" -s "$subject"
ofconfig list -n "$node_name" -s "$subject" -sec "$section"
ofconfig list -n "$node_name" -s "$subject" -sec "$section" -k "$key" -l
```

`update`로 값을 바꾸려면 node, subject, section, key와 새 value가 모두 필요하다.

```bash
ofconfig update \
  -n "$node_name" \
  -s "$subject" \
  -sec "$section" \
  -k "$key" \
  -v "$new_value"
```

명령이 `COMPLETED SUCCESSFULLY.`를 출력하고 종료 코드가 0이어야 성공으로 판단한다. 성공 메시지만 믿지 말고 같은 키를 다시 `list -l`로 조회하여 실제 값을 확인한다.

`TYPE`과 `AVAIL_VALUE`를 보고 새 값을 검증한다.

- `TYPE=1`인 Y/N 값은 `YES` 또는 `NO`만 사용한다.
- `TYPE=2`인 문자열에 `AVAIL_VALUE`가 있으면 나열된 값 중 하나만 사용한다.
- `TYPE=3`인 숫자에 `AVAIL_VALUE`가 있으면 표시된 최솟값과 최댓값 범위 안의 숫자만 사용한다.

`ofconfig update`로 변경하면 설정 DB와 TCache가 자동으로 동기화되므로 별도의 `ofconfig load`를 실행하지 않는다. `load`는 DB를 외부에서 직접 수정했을 때의 동기화 명령이며, 이 스킬에서는 DB를 직접 수정하지 않는다.

## 조회 절차

1. 요청과 관련된 product, subject, section, key를 확인한다.
2. 정확한 key를 모르면 node 전체가 아니라 subject, section 순서로 조회 범위를 좁힌다.
3. 최종 키는 반드시 `-l`로 조회한다.
4. `VALUE`, `DEFAULT_VALUE`, `TYPE`, `AVAIL_VALUE`, `DESCRIPTION`을 해석하여 테스트 전제와 일치하는지 판단한다.
5. 조회만 수행했다면 명령, 노드, 키 경로와 판단 결과를 보고한다. 민감한 값은 가린다.

## 가역 변경 절차

다음 순서를 하나의 작업 단위로 수행한다.

### 1. 변경 전 스냅샷과 복원 계획 확보

대상 키를 `list -l`로 조회하고 아래 항목을 작업 기록에 보관한다.

```text
node: <OPENFRAME_NODENAME>
subject: <subject>
section: <section>
key: <key>
original-value: <value, secret이면 보고에서 redacted>
temporary-value: <test value, secret이면 보고에서 redacted>
reason: <test name or reason>
```

- 키가 존재하지 않으면 `update`하지 않는다. 임시 키 추가와 사후 삭제가 정말 필요하면 사용자 승인을 받아 별도 절차를 정한다.
- 원래 `VALUE`가 비어 있으면 중단한다. 빈 문자열은 `update -v ""`로 정확히 복원되지 않을 수 있으며, `DEFAULT_VALUE`를 명시적으로 쓰는 것은 원래 DB 상태의 복원이 아니다.
- 원래 값과 임시 값이 같으면 변경하지 않고 그대로 테스트한다.
- 허용 값 또는 범위를 벗어나면 변경하지 않고 올바른 후보를 제시한다.

### 2. 최소 범위 변경과 즉시 검증

변경 직전에 같은 키를 다시 조회하여 스냅샷 이후 다른 작업이 값을 바꾸지 않았는지 확인한다. 원래 값이 유지된 경우에만 `update`하고 즉시 재조회한다.

```bash
ofconfig update -n "$node_name" -s "$subject" -sec "$section" -k "$key" -v "$temporary_value"
ofconfig list   -n "$node_name" -s "$subject" -sec "$section" -k "$key" -l
```

실패하면 테스트를 시작하지 않는다. 일부 키를 이미 변경했다면 변경된 키부터 복원한다.

### 3. 테스트 실행

설정 적용이 검증된 뒤에만 테스트를 실행한다. 해당 설정이 프로세스 시작 시에만 읽히는지는 관련 configuration guide에서 확인한다. 재기동이 필요하더라도 테스트 범위에서 허용되지 않은 서비스 재기동을 임의로 수행하지 않는다.

### 4. 성공·실패·중단 시 복원

테스트 결과와 관계없이 정리 단계에서 복원한다. 복원 직전에 현재 값을 다시 조회한다.

- 현재 값이 이 작업의 임시 값과 같으면 원래 값으로 되돌린다.
- 현재 값이 임시 값과 다르면 다른 작업이 변경했을 수 있으므로 덮어쓰지 않는다. 충돌한 키, 예상 임시 값, 현재 값을 사용자에게 보고하되 민감한 값은 가린다.

```bash
ofconfig update -n "$node_name" -s "$subject" -sec "$section" -k "$key" -v "$original_value"
ofconfig list   -n "$node_name" -s "$subject" -sec "$section" -k "$key" -l
```

복원 명령의 종료 코드와 최종 `VALUE`가 스냅샷과 같은지 모두 확인한다. 복원이 실패하면 테스트 성공으로 보고하지 말고 즉시 원인과 현재 시스템 상태를 알린다.

## 여러 키 변경

1. 모든 대상 키의 스냅샷과 복원 가능성을 먼저 확인한다.
2. 키를 하나씩 변경하고 각각 즉시 검증한다.
3. 중간 변경이 실패하면 이미 변경한 키를 역순으로 복원한다.
4. 테스트가 끝나면 모든 키를 변경의 역순으로 복원한다.
5. 각 키의 현재 값이 해당 작업의 임시 값인지 확인한 뒤 복원하여 다른 작업의 변경을 덮어쓰지 않는다.
6. 마지막에 모든 키를 다시 조회하고 원래 값과 비교한다.

## 오류 처리

`ofconfig` 오류가 발생하면 화면 출력부터 보존하고 다음을 확인한다.

1. `$OPENFRAME_NODENAME`이 설정되어 있고 길이가 16자를 넘지 않는지 확인한다.
2. node, subject, section, key의 철자와 대소문자를 상세 조회 결과 및 관련 configuration guide와 비교한다.
3. 새 값이 `TYPE`과 `AVAIL_VALUE` 제약을 만족하는지 확인한다.
4. Tmax 서비스 상태와 사용자 권한을 확인한다. `ofconfig`는 설정 관리 서비스와 인증을 사용한다.
5. `$TMAXDIR/log`, `$OPENFRAME_HOME/log`에서 같은 시각의 오류를 확인한다.
6. OpenFrame 오류 코드는 `oferror <error-code>`로 상세 내용을 조회한다.

환경, 경로, 값 형식 또는 명령 인자 문제는 직접 해결하고 다시 시도한다. 권한 부족, 서비스 장애, 값 의미의 불확실성 또는 안전한 복원 불가 상태는 임의 우회하지 말고 확인한 증거와 필요한 조치를 사용자에게 보고한다.

## 최종 보고

다음을 간결하게 보고한다.

- 사용한 노드와 `SUBJECT / SECTION / KEY`
- 실행한 `list`와 `update` 작업
- 변경 전 값, 테스트 값, 복원 후 값의 일치 여부(민감한 값은 가림)
- 테스트 결과와 설정 복원 성공 여부
- 실패하거나 복원하지 못한 경우 핵심 원인과 현재 값 상태

## 기준 자료

명령 옵션이나 설정 의미가 불확실하면 아래 자료를 확인하고, 확인한 동작 규칙을 이 절차에 적용한다.

- 명령 매뉴얼: `{{manual_base}}/openframe_base/docs/modules/tool-reference-guide/pages/etc/sect-ofconfig.adoc`
- 설정 매뉴얼: 관련 제품 매뉴얼의 `configuration-guide`
- 명령 구현: `{{source_base}}/base/src/tool/ofconfig/ofconfig_main.c`
- 설정 공통 구현: `{{source_base}}/base/src/common/ofcom/ofcom_conf.c`
