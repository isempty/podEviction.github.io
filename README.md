사용자의 질문을 바탕으로 SUSE 12에서 Pacemaker를 사용하며 active-standby 설정에서 standby 노드를 가동할 때 active 노드가 SBD(STONITH Block Device)에 의해 fencing되는 문제의 원인과 대책을 자세히 설명하겠습니다. Fencing은 클러스터에서 데이터 무결성을 보호하기 위해 비정상 노드를 격리하거나 재부팅하는 메커니즘이며, 여기서는 SBD가 이를 수행합니다. 아래에 문제의 주요 원인과 해결 방법을 단계별로 정리했습니다.
문제 상황 분석
환경: SUSE 12, Pacemaker, active-standby 설정, SBD 사용.

증상: Standby 노드를 가동하면 active 노드가 SBD에 의해 fencing(강제 재부팅)됨.

의미: Standby 노드가 클러스터에 정상적으로 합류하지 못하거나, active 노드가 이를 비정상 상황으로 인식해 fencing을 유발하는 것으로 보임.

가능한 원인
Pacemaker 클러스터 설정 오류
Standby 노드가 클러스터에 제대로 정의되거나 통합되지 않아 active 노드가 이를 위협(예: split-brain)으로 판단할 가능성.

발생 이유: Standby 노드가 클러스터 멤버십에 정상적으로 참여하지 못하면 active 노드가 fencing을 통해 클러스터 안정성을 유지하려 함.

SBD 장치 접근 문제
SBD는 공유 스토리지를 통해 노드 간 메시지를 교환하며, standby 노드가 SBD 장치에 접근하지 못하면 active 노드가 이를 감지하고 fencing을 유발.

발생 이유: SBD 장치가 양쪽 노드에서 정상적으로 인식되지 않거나 접근 권한 문제 발생.

네트워크 통신 문제
노드 간 네트워크 연결이 불안정하거나 잘못 설정되어 active 노드가 standby 노드를 감지하지 못함.

발생 이유: Pacemaker는 Corosync를 통해 통신하며, 네트워크 단절 시 fencing을 통해 데이터 손상을 방지하려 함.

Watchdog 설정 오류
SBD는 watchdog 장치를 사용해 노드가 응답하지 않을 경우 재부팅을 유도. Watchdog 설정이 잘못되면 예기치 않은 fencing 발생.

발생 이유: SBD 데몬이 watchdog을 주기적으로 "feeding"하지 못하면 노드가 강제로 재부팅됨.

2노드 클러스터의 Quorum 문제
2노드 클러스터에서 quorum 설정이 적절하지 않으면 노드 간 연결이 끊길 때 fencing이 발생할 수 있음.

발생 이유: 기본적으로 2노드 클러스터는 quorum을 유지하기 어려워 split-brain 방지를 위해 fencing에 의존.

대책 및 해결 방법
1. Pacemaker 설정 점검
목표: Standby 노드가 클러스터에 정상적으로 통합되도록 설정 확인.

방법:
클러스터 상태 확인:
bash

crm status

두 노드가 모두 "online"으로 표시되는지 확인.

Standby 노드 설정 확인:
Standby 노드가 올바르게 정의되었는지 확인하고, 필요 시 수동으로 활성화:
bash

crm node standby <standby_node_name>
crm node online <standby_node_name>

제약 조건 점검:
crm configure show로 리소스 및 위치 제약 조건을 확인해 standby 노드가 비정상적으로 배제되지 않았는지 점검.

2. SBD 설정 및 접근성 확인
목표: SBD 장치가 양쪽 노드에서 정상적으로 작동하도록 보장.

방법:
SBD 설정 파일 확인:
/etc/sysconfig/sbd에서 장치 경로 확인:

SBD_DEVICE="/dev/sbd_device"

SBD 장치 상태 점검:
양쪽 노드에서 접근 가능 여부 확인:
bash

sbd -d /dev/sbd_device dump

오류가 있으면 스토리지 연결 및 권한 점검.

다중 SBD 장치 고려:
장치 하나에 의존하지 않도록 2~3개의 SBD 장치를 설정해 장애 시 안정성 확보.

3. 네트워크 설정 점검
목표: 노드 간 통신이 안정적으로 유지되도록 설정.

방법:
Corosync 상태 확인:
bash

corosync-cfgtool -s

"ring" 상태가 "active"이고 오류가 없는지 확인.

방화벽 점검:
UDP 포트 5404, 5405가 열려 있는지 확인:
bash

firewall-cmd --list-ports

네트워크 연결 테스트:
ping으로 노드 간 연결 확인.

4. Watchdog 설정 점검
목표: Watchdog이 정상 작동하도록 설정.

방법:
Watchdog 장치 확인:
/etc/sysconfig/sbd에서 설정 확인:

SBD_WATCHDOG_DEV=/dev/watchdog

타임아웃 설정 점검:
SBD_WATCHDOG_TIMEOUT 값이 적절한지 확인 (예: 5~10초).

테스트:
SBD 서비스 중지 후 노드가 재부팅되는지 확인:
bash

systemctl stop sbd

5. 2노드 클러스터 Quorum 조정
목표: Quorum 문제로 인한 fencing 방지.

방법:
QDevice 추가:
2노드 클러스터에서 quorum을 보장하려면 QDevice 설정 (SUSE 12 이후 버전 지원 여부 확인 필요).

Quorum 정책 조정:
crm configure edit로 no-quorum-policy=ignore 설정 가능, 단 split-brain 위험이 있으므로 주의.

6. Fencing 메커니즘 테스트
목표: Fencing 동작이 의도대로 작동하는지 확인.

방법:
수동 fencing 테스트:
bash

stonith_admin -F <node_name>

장애 시뮬레이션:
네트워크 또는 SBD 장치 연결을 끊고 클러스터 반응 확인.

7. 로그 분석
목표: 구체적인 오류 원인 파악.

방법:
클러스터 로그 확인:
/var/log/cluster/corosync.log

/var/log/pacemaker/pacemaker.log

주요 키워드 검색:
"fencing", "sbd", "timeout" 관련 메시지 확인.

추가 고려 사항
SUSE 12 버전 한계: 오래된 버전이므로 최신 서비스 팩 및 Pacemaker 업데이트 적용 권장.

SBD 장치 다중화: 단일 장치 장애를 대비해 다중 SBD 장치 사용 권장.

업그레이드 고려: 지속적인 문제가 있다면 최신 SUSE 버전으로 업그레이드 검토.

결론
SUSE 12에서 Pacemaker와 SBD를 사용하는 active-standby 설정에서 standby 노드 가동 시 active 노드가 fencing되는 문제는 주로 클러스터 설정, SBD 장치 접근성, 네트워크 문제, 또는 watchdog 설정 오류에서 비롯됩니다. 위의 단계별 점검과 해결 방법을 통해 원인을 파악하고, 로그 분석 및 테스트를 병행하면 문제를 해결할 수 있을 것입니다. 추가적인 오류 메시지나 로그가 있다면 이를 공유해 주시면 더 구체적인 도움을 드릴 수 있습니다.

15 웹페이지

