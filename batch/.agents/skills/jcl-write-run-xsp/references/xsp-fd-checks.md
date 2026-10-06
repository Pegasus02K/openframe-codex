# XSP FD 작성·실패 점검

FD를 새로 작성하거나 FCPY로 디스크/테이프 복사를 재현할 때 사용한다. 문법은 `xsp-jcl-reference-guide/pages/jcl/sect-fd-statement.adoc`, FCPY의 FD·제어문은 `xsp-utility-reference-guide/pages/ds/sect-fcpy.adoc`에서 확인한다.

## 이번 실패에서 확인한 구분

2026-10-06 OPENFRAME-830, OpenFrame 7.3 XSP/Tibero의 실제 INPJCL/SYSMSG를 확인했다.

| 실패 | 확인한 증거 | 작성·준비 단계의 개선 |
| --- | --- | --- |
| `DISP=SHR` | JOB00004~00006: `Keyword:DISP`, `JCL keyword value is invalid - SHR`, `Jcl parsing error` | 기존 입력에는 불필요한 DISP를 생략하거나 XSP 의미에 맞게 `KEEP` 등을 선택한다. MVS의 SHR/OLD/NEW/MOD를 XSP DISP로 옮기지 않는다. |
| `LIST=SOUT` | JOB00012: `volm_get_volume_group()`, `unit=SOUT`, `rc=-6104`, LIST FD 할당 실패 | 장치와 SOUT 출력 클래스를 분리해 `LIST=DA,SOUT=<확인한 클래스>`로 작성한다. 파싱 통과로 장치 그룹 존재가 검증되지는 않는다. |
| 준비되지 않은 테이프 볼륨 | JOB00010: U02 `volm_get_ample_volume()`, `rc=-22016` | 논리 볼륨 정의만으로 준비 완료를 판단하지 않는다. 물리 테이프 연결과 실제 경로를 확인한다. 이 오류는 JCL 파싱 실패와 구분한다. |

`DISP` 생략 시 기본값 KEEP, CAT는 STEP 종료 시 카탈로그 등록이다. 이것을 신규 데이터셋 생성이나 공유 사용 요청과 동일하게 해석하지 않는다.

## FCPY 작성 예시

아래 이름·볼륨·클래스는 자리표시자에 해당하는 예시이며 그대로 제출하지 않는다. 실제 존재하는 입력, 충돌하지 않는 출력 이름, 확인한 볼륨/장치 그룹/클래스로 바꾼다.

```text
\ JOB COPYCHK
\COPY EX FCPY,RSIZE=256
\ PARA / FCPY IN=U01,OUT=U02,COUNT=YES
\ FD U01=DA,FILE=USER.INPUT
\ FD U02=DA,FILE=USER.OUTPUT,VOL=DISK01,
             CYL=(1,1,RLSE),DISP=CAT
\ FD LIST=DA,SOUT=A
\ JEND
```

테이프 출력은 실제 등록된 장치 그룹(이번 테스트는 VTL)과 논리 테이프 볼륨을 U02에 지정한다. `VTL` 그룹이 모든 환경에 존재한다고 가정하지 않는다. `volmgr list group`, 논리 볼륨 조회 및 `volmgr list volume -t`로 대상 장치·물리 테이프 연결·경로를 확인한다. 신규 환경 준비가 요청 범위에 포함된 경우에만 기존 자원과 충돌하지 않는 이름으로 정의한다.

## 검증 순서와 한계

- 작성한 FD와 유틸리티 매뉴얼을 대조한 뒤, 문법이 불확실하면 실제 SYS1.JCLLIB에 배치하고 `tjesmgr SCAN <멤버>` 결과의 JOB ID와 SYSMSG를 확인한다. SCAN은 JOBQ/SPOOL을 생성하지만 유틸리티를 실행하지 않는다.
- SCAN의 DONE은 FD 할당·압축·복사가 성공했다는 뜻이 아니다. 실제 RUN에서 JOB 상태, STEP RC, LIST 레코드 건수/NORMAL END와 요청된 파일 결과를 확인한다.
- 이번 수정 JCL은 JOB00017(VTL 출력), JOB00018(DA 디스크 출력), JOB00020(VTL 입력)에서 20만 레코드와 원본 일치를 확인했다. STEP R0010은 해당 설치의 관찰값이며 다른 유틸리티·환경의 정상 RC로 고정하지 않는다.
- RUN 전 파싱·할당 실패를 대상 제품 결함 재현으로 보고하지 않는다. 실패 FD와 오류 계층을 먼저 특정하고, 생성된 출력이 있다면 재시도 전에 그 상태를 확인한다.
- 테스트 자원만 정리하고 기존 데이터셋/JCL/설정 및 JOB SPOOL을 임의로 삭제하지 않는다.

## 보완 후 검증 기록

스킬 사본에 `quick_validate.py`를 실행해 `Skill is valid!`를 확인했다. 문법 예제는 FD/FCPY 매뉴얼 및 위 실제 성공·실패 SPOOL과 대조했다. 추가 SCAN 비교는 실행을 시도했으나 `tjesmgr`가 응답하지 않아 중단했고, 별도 `NODESTATUS`도 10초 제한에서 종료 코드 124로 끝났다. 추가 SCAN 성공을 검증했다고 보고하지 않는다. 명령이 응답하지 않을 때는 제출 실패나 JOB 미생성으로 단정하지 말고, 환경·통신 상태와 이미 생성된 JOB 유무를 확인한 후 재시도한다. 검증만을 위해 기존 서비스를 임의로 재기동하지 않는다.
