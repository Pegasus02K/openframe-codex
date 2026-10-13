# XSP FD 작성·실패 점검

FD 문법은 `xsp-jcl-reference-guide/pages/jcl/sect-fd-statement.adoc`, FCPY의 FD·제어문은 `xsp-utility-reference-guide/pages/ds/sect-fcpy.adoc`에서 확인한다.

## 작성과 준비

- MVS의 상태·후처리 오퍼랜드를 이름만 보고 XSP `DISP`로 옮기지 않는다. `KEEP`·`CAT` 등 XSP의 의미와 생략 시 동작을 매뉴얼에서 확인한다.
- SPOOL 출력은 장치와 출력 클래스를 분리해 `LIST=DA,SOUT=<확인한 클래스>`처럼 작성한다. `LIST=SOUT`로 장치와 오퍼랜드를 혼동하지 않는다.
- 데이터셋·장치 그룹·논리 볼륨과 실제 경로를 확인한다. 테이프를 사용하면 물리 테이프 연결까지 확인하고 논리 볼륨 정의만으로 준비 완료를 판단하지 않는다.
- `volmgr` 등의 조회 명령은 현재 도구 사용법으로 확인한다. 신규 자원 정의는 사용자 요청 범위에서 기존 자원과 충돌하지 않는 대상만 수행한다.

## 문법 예시

아래 이름·볼륨·클래스는 예시다. 실제 입력, 충돌하지 않는 출력 이름과 확인한 자원으로 바꿔 사용한다.

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

## 검증과 실패 분류

- 실행 검증 또는 SCAN이 요청 범위에 포함되고 문법이 불확실하면 현재 `tjesmgr SCAN` 사용법으로 검사한다. 작성만 요청되면 정적 점검에 한정한다. SCAN도 JOBQ/SPOOL을 생성하므로 실제 JOB ID와 출력 결과를 확인한다.
- SCAN 성공은 프로그램 실행·FD 할당·복사 성공을 보장하지 않는다. 실제 RUN은 `batch-job-run`으로 JOB·STEP·SPOOL과 요청된 최종 결과를 검증한다.
- 파싱, 프로그램 탐색, 장치·볼륨 할당, 프로그램 실행, 결과 불일치 중 실패한 계층과 FD를 구분한다. 준비 실패를 대상 제품 결함으로 단정하지 않는다.
- 명령이 응답하지 않으면 JOB 생성 여부와 환경·통신 상태를 확인한 뒤 재시도한다. 조회 실패만으로 제출 실패·JOB 미생성으로 판단하거나 공유 서비스를 임의 재기동하지 않는다.
- 생성된 출력의 상태를 확인하고 재실행한다. 테스트 자원만 정리하며 기존 데이터셋·JCL·설정과 JOB SPOOL은 보존한다.
- 실제 입력·명령·JOB ID·오류·결과는 현재 세션에서 사용자와 확인해 처리한다. 개별 실행 사례는 이 참조에 누적하지 않는다.
