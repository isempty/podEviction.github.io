아래는 주어진 로그를 기반으로 `scp-srmpd001`과 `scp-srmpd002` 서버에서 발생한 이벤트를 시간대별로 정리하고 분석한 내용입니다. 이를 통해 어떤 에러가 발생했는지, 그로 인해 어떤 문제가 발생했는지, 그리고 원인이 무엇인지 자세히 설명하겠습니다. 마지막으로 타임라인 테이블을 제공하여 두 서버 간의 상호작용과 각 이벤트의 의미를 명확히 하겠습니다.

---

### **로그 분석 및 문제 파악**

#### **1. 주요 이벤트 타임라인 요약**
로그를 시간 순서대로 분석하면 다음과 같은 주요 이벤트가 두 서버에서 발생했습니다:

- **02:20:34**: `scp-srmpd002`에서 quorum(쿼럼) 손실이 발생하며 fencing(펜싱)이 시작됨. 이는 클러스터의 데이터 무결성을 유지하기 위한 초기 조치.
- **02:23:05**: `scp-srmpd001`에서 `mysbd01` 모니터링 작업이 타임아웃(OCF_TIMEOUT) 발생. 이후 복구 시도가 반복됨.
- **02:23:43~02:24:00**: `scp-srmpd001`에서 SBD(Storage-Based Death) servant가 반복적으로 "outdated"와 "healthy" 상태를 오감. 이는 스토리지 연결 불안정성을 시사.
- **02:25:06**: `scp-srmpd001`에서 `DB` 모니터링 작업이 타임아웃 발생. 이후 `DB`가 종료됨.
- **02:25:09~02:25:25**: `scp-srmpd002`에서 watchdog-fencing-query 실패와 SBD 상태 불명확 경고 발생.
- **02:27:01~02:27:08**: `mysbd01`과 `DB`를 다시 시작하려는 시도 성공.

#### **2. 발생한 에러와 문제**
- **Quorum 손실과 펜싱 (02:20:34, 반복 발생)**  
  - `On loss of quorum: Fence all remaining nodes` 메시지가 여러 번 나타남.  
  - **문제**: 클러스터가 쿼럼을 잃어 split-brain(분할 뇌) 상황을 방지하기 위해 펜싱이 실행됨. 이는 노드 간 통신 또는 리소스 상태 문제로 발생.
  - **영향**: 클러스터가 정상적으로 동작하지 못하고 리소스 복구가 필요해짐.

- **mysbd01 모니터링 타임아웃 (02:23:05)**  
  - `Transition 14431 action 55 (mysbd01_monitor_60000 on scp-srmpd001): expected 'ok' but got 'OCF_TIMEOUT'`  
  - **문제**: `scp-srmpd001`에서 `mysbd01` 리소스의 상태 확인이 60초 내에 완료되지 않음.  
  - **영향**: Pacemaker가 이를 실패로 간주하고 복구(Stop → Start)를 시도했으나, 이후 시작 작업도 타임아웃(02:23:31) 발생. 실패 카운트가 `INFINITY`로 설정되며 `mysbd01`을 `scp-srmpd001`에서 강제로 제외(02:23:34).

- **SBD Servant 불안정 (02:23:43~02:24:00)**  
  - `inquisitor_child: Servant ... is outdated (age: 4)`와 `is healthy (age: 0)`가 반복.  
  - **문제**: `scp-srmpd001`의 SBD 디바이스 연결이 불안정하여 주기적으로 응답 지연 발생.  
  - **영향**: SBD는 펜싱 메커니즘의 핵심이므로, 이 불안정성은 쿼럼 손실과 펜싱 실패의 원인이 될 수 있음.

- **DB 모니터링 타임아웃 (02:25:06)**  
  - `Result of monitor operation for DB on scp-srmpd001: Timed Out`  
  - **문제**: `scp-srmpd001`에서 `DB` 리소스 모니터링이 60초 내에 응답하지 않음.  
  - **영향**: `DB`가 종료되고 복구 시도 시작. 이는 `mysbd01` 문제와 연관될 가능성 높음.

- **펜싱 쿼리 실패 및 SBD 상태 불명확 (02:25:09~02:25:25)**  
  - `watchdog-fencing-query failed`와 `SBD Status is not clear` 발생.  
  - **문제**: 펜싱 메커니즘이 제대로 작동하지 않음.  
  - **영향**: 클러스터가 안정적으로 노드를 관리하지 못해 리소스 복구가 지연됨.

#### **3. 원인 분석**
위 에러들을 종합하면, **스토리지 연결 불안정**이 근본 원인으로 보입니다:
- **증거**:  
  - `scp-srmpd001`에서 SBD servant가 반복적으로 "outdated" 상태를 보임. 이는 iSCSI 스토리지 타겟(예: `ip-198.19.19.168:3260`)과의 연결에 문제가 있음을 나타냄.
  - `mysbd01`과 `DB` 모니터링 타임아웃은 스토리지 지연으로 리소스가 제때 응답하지 못한 결과일 가능성 높음.
  - `watchdog-fencing-query` 실패와 `SBD Status is not clear`는 SBD 디바이스 접근 문제와 직접 연관.
- **추정**: 네트워크 지연, iSCSI 설정 오류, 또는 스토리지 타겟의 부하/장애가 발생했을 가능성.
- **결론**: 스토리지 연결이 불안정해지며 SBD가 제 기능을 하지 못했고, 이는 쿼럼 손실, 리소스 타임아웃, 펜싱 실패로 이어짐.

#### **4. 해결 방안 제안**
- **스토리지 연결 점검**:  
  - `scp-srmpd001`과 iSCSI 타겟 간 네트워크 상태(ping, latency) 확인.
  - iSCSI 설정(타임아웃 값, 재시도 설정) 검토 및 최적화.
  - 스토리지 디바이스 상태(부하, 오류 로그) 확인.
- **SBD 설정 확인**:  
  - SBD 디바이스의 안정성 테스트 및 대체 디바이스 고려.
- **Pacemaker 설정 조정**:  
  - 모니터링 타임아웃 값(60000ms)을 상황에 맞게 조정하여 불필요한 실패 방지.
- **로그 추가 수집**:  
  - 네트워크 및 스토리지 관련 시스템 로그를 추가로 분석하여 상세 원인 파악.

---

### **타임라인 테이블**
아래는 `scp-srmpd001`과 `scp-srmpd002`의 이벤트를 시간대별로 정리한 테이블입니다. 각 이벤트의 의미를 마지막 열에 설명했습니다.

| **시간**      | **scp-srmpd001 이벤트**                                                                                          | **scp-srmpd002 이벤트**                                                                                             | **의미**                                                                                   |
|---------------|---------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------|
| 02:20:18      | -                                                                                                             | systemd[1]: Removed slice User Slice of UID 0.                                                               | 루틴 시스템 작업(중요도 낮음).                                                            |
| 02:20:34      | -                                                                                                             | pacemaker-controld: State transition S_IDLE -> S_POLICY_ENGINE<br>pacemaker-schedulerd: On loss of quorum: Fence all remaining nodes<br>Calculated transition 16747<br>Transition 16747 Complete<br>State transition S_TRANSITION_ENGINE -> S_IDLE | 쿼럼 손실로 펜싱 시작. 클러스터 상태 평가 후 정상 완료.                                     |
| 02:21:00      | -                                                                                                             | systemd[1]: Starting system activity accounting tool...                                                      | 루틴 시스템 작업(중요도 낮음).                                                            |
| 02:23:05      | -                                                                                                             | pacemaker-controld: mysbd01_monitor_60000 on scp-srmpd001: expected 'ok' but got 'OCF_TIMEOUT'<br>State transition S_IDLE -> S_POLICY_ENGINE<br>Setting fail-count-mysbd01#monitor_60000[scp-srmpd001]: 1<br>On loss of quorum: Fence all remaining nodes<br>Actions: Recover mysbd01 on scp-srmpd001 | `mysbd01` 모니터링 타임아웃 발생. 복구 시도 시작. 쿼럼 손실로 펜싱 재시작.                   |
| 02:23:08      | -                                                                                                             | pacemaker-controld: Initiating start operation mysbd01_start_0 on scp-srmpd001                               | `mysbd01` 시작 시도.                                                                      |
| 02:23:31      | -                                                                                                             | pacemaker-controld: mysbd01_start_0 on scp-srmpd001: expected 'ok' but got 'OCF_TIMEOUT'<br>On loss of quorum: Fence all remaining nodes<br>Actions: Recover mysbd01 on scp-srmpd001 | `mysbd01` 시작 타임아웃. 복구 재시도 및 펜싱 반복.                                         |
| 02:23:32      | -                                                                                                             | pacemaker-attrd: Setting fail-count-mysbd01#start_0[scp-srmpd001]: INFINITY                                  | `mysbd01` 시작 실패 카운트 무한대로 설정(반복 실패 인정).                                   |
| 02:23:34      | -                                                                                                             | pacemaker-schedulerd: Forcing mysbd01 away from scp-srmpd001 after 1000000 failures<br>Transition 16751 Complete<br>State transition S_TRANSITION_ENGINE -> S_IDLE | `mysbd01`을 `scp-srmpd001`에서 강제 제외. 클러스터 안정화 시도.                            |
| 02:23:43      | sbd[16857]: Servant ... is outdated (age: 4)                                                                 | -                                                                                                            | SBD 디바이스 응답 지연. 스토리지 연결 문제 시작.                                           |
| 02:23:45      | sbd[16857]: Servant ... is healthy (age: 0)                                                                  | -                                                                                                            | SBD 정상 복구. 그러나 불안정성 지속.                                                      |
| 02:23:48~57   | sbd[16857]: Servant outdated와 healthy 반복                                                                  | -                                                                                                            | SBD 연결이 불안정하게 반복(스토리지 문제 증거).                                           |
| 02:25:05      | systemd-journald[1462]: Suppressed 43314 messages from session-92448.scope                                   | -                                                                                                            | 로그 메시지 과다로 억제(시스템 부하 가능성).                                               |
| 02:25:06      | pacemaker-execd: DB_monitor_60000 timed out after 60000ms<br>pacemaker-controld: Result of monitor operation for DB: Timed Out<br>Requesting stop operation for DB | pacemaker-controld: DB_monitor_60000 on scp-srmpd001: expected 'ok' but got 'error'<br>On loss of quorum: Fence all remaining nodes<br>Actions: Recover DB on scp-srmpd001 | `DB` 모니터링 타임아웃. `scp-srmpd001`에서 문제 발생, `scp-srmpd002`에서 복구 지시.         |
| 02:25:07      | pgsql(DB)[2360162]: INFO: server shutting down                                                              | -                                                                                                            | `DB` 종료 완료.                                                                          |
| 02:25:09      | -                                                                                                             | pacemaker-attrd: fail-count 및 last-failure 초기화<br>pacemaker-controld: watchdog-fencing-query failed       | 실패 카운트 초기화 시도 중 펜싱 쿼리 실패(펜싱 메커니즘 문제).                             |
| 02:25:22      | -                                                                                                             | pacemaker-controld: watchdog-fencing-query failed                                                    | 펜싱 쿼리 재실패(지속적인 펜싱 문제).                                                    |
| 02:25:25      | -                                                                                                             | root[2501885]: [ClusterWarning] SBD Status is not clear                                              | SBD 상태 불명확 경고. 스토리지 및 펜싱 문제의 결과.                                       |
| 02:27:01      | -                                                                                                             | pacemaker-schedulerd: Actions: Start DB and mysbd01 on scp-srmpd001<br>pacemaker-controld: Initiating start operations | `DB`와 `mysbd01` 재시작 시도.                                                            |
| 02:27:08      | -                                                                                                             | pacemaker-controld: Transition 16754 Complete<br>State transition S_TRANSITION_ENGINE -> S_IDLE             | 재시작 성공. 클러스터 안정 상태로 복귀.                                                   |

---

### **최종 결론**
- **문제**: 스토리지 연결 불안정으로 인해 `mysbd01`과 `DB` 모니터링 타임아웃, 쿼럼 손실, 펜싱 실패가 연쇄적으로 발생.
- **원인**: `scp-srmpd001`의 SBD servant가 불안정하여 iSCSI 스토리지 연결에 문제가 있었음.
- **해결**: 스토리지 연결(네트워크, iSCSI 설정, 타겟 상태)을 점검하고 안정화해야 함.

이 분석을 통해 클러스터 문제를 해결하려면 스토리지 인프라에 대한 즉각적인 조치가 필요함을 알 수 있습니다.
