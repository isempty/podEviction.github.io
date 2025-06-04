Red Hat Enterprise Linux (RHEL) 7.x에서 Linux 커널이 IP 충돌을 처리하는 방식은 기본적으로 Linux 커널의 네트워크 스택과 ARP(Address Resolution Protocol) 프로토콜에 의존하며, RHEL 7.x의 특정 설정과 동작을 중심으로 설명하겠습니다. RHEL 7.x는 3.10.x 계열의 커널을 사용하며, IP 충돌 감지 및 처리 과정은 일반적인 Linux 동작과 유사하지만, RHEL의 기본 설정과 관리 도구(NetworkManager, systemd 등)에 따라 약간의 차이가 있습니다. 아래에서 자세히 설명합니다.
1. RHEL 7.x 커널의 IP 충돌 처리 메커니즘
RHEL 7.x 커널은 네트워크 인터페이스에서 IP 충돌을 감지하고 처리하기 위해 ARP 프로토콜과 관련된 메커니즘을 사용합니다. IP 충돌은 동일한 네트워크 내에서 두 개 이상의 장치가 동일한 IP 주소를 사용할 때 발생하며, 커널은 이를 ARP 요청/응답을 통해 감지합니다.
1.1 ARP를 통한 충돌 감지
ARP 요청 및 응답:
네트워크 인터페이스가 활성화되거나 IP 주소가 설정될 때, 커널은 ARP 요청을 브로드캐스트하여 해당 IP 주소가 이미 사용 중인지 확인합니다.

다른 장치가 동일한 IP 주소로 ARP 응답을 보내면, 커널은 이를 IP 충돌로 간주합니다.

RHEL 7.x에서는 arp 서브시스템이 이를 처리하며, /proc/sys/net/ipv4/conf/*/ 디렉터리의 sysctl 설정을 통해 동작을 제어합니다.

Duplicate Address Detection (DAD):
RHEL 7.x에서 네트워크 인터페이스가 IP 주소를 설정할 때 DAD를 수행합니다. 이는 특히 IPv4와 IPv6 모두에서 적용되며, IPv4의 경우 ARP 요청을 통해 중복 주소를 확인합니다.

DAD 과정에서 충돌이 감지되면, 커널은 네트워크 인터페이스의 IP 주소를 활성화하지 않거나 비활성화할 수 있습니다.

1.2 커널의 동작
인터페이스 비활성화:
IP 충돌이 감지되면, 커널은 해당 네트워크 인터페이스의 IP 주소를 비활성화하거나 인터페이스 자체를 down 상태로 전환할 수 있습니다.

이는 net.ipv4.conf.<interface>.arp_notify 설정이 활성화된 경우 로그로 기록됩니다.

패킷 처리 오류:
충돌로 인해 패킷이 잘못된 장치로 라우팅되면, 커널은 패킷 드롭, 지연, 또는 라우팅 오류를 발생시킬 수 있습니다.

DHCP 환경:
DHCP 클라이언트(dhclient)가 사용 중인 경우, 충돌이 감지되면 커널은 DHCP 클라이언트에 이를 알리고 새로운 IP 주소를 요청하도록 유도합니다.

1.3 관련 sysctl 설정
RHEL 7.x에서 IP 충돌 처리와 관련된 주요 sysctl 파라미터는 다음과 같습니다:
net.ipv4.conf.<interface>.arp_ignore:
ARP 요청에 대한 응답 방식을 제어합니다.

기본값: 0 (모든 ARP 요청에 응답).

1로 설정하면, 요청이 해당 인터페이스의 IP 주소와 관련된 경우에만 응답합니다.

예: echo 1 > /proc/sys/net/ipv4/conf/eth0/arp_ignore

net.ipv4.conf.<interface>.arp_notify:
IP 충돌 감지 시 시스템 로그에 기록하도록 설정.

기본값: 0 (비활성화).

1로 설정하면 충돌 시 로그를 생성합니다.

예: echo 1 > /proc/sys/net/ipv4/conf/all/arp_notify

net.ipv4.conf.<interface>.arp_announce:
ARP 요청 시 사용할 IP 주소를 제어하여 충돌 가능성을 줄입니다.

기본값: 0.

2로 설정하면 가장 적절한 IP 주소를 선택하여 ARP 충돌을 최소화합니다.

이 설정들은 /etc/sysctl.conf 또는 /etc/sysctl.d/ 디렉터리에 저장하여 영구적으로 적용할 수 있습니다:
bash

echo "net.ipv4.conf.all.arp_notify = 1" >> /etc/sysctl.d/99-custom.conf
sysctl -p /etc/sysctl.d/99-custom.conf

2. 커널 로그
IP 충돌이 감지되면, RHEL 7.x 커널은 /var/log/messages 또는 dmesg에 관련 메시지를 기록합니다. 예시 로그는 다음과 같습니다:

Jun 04 12:55:10 hostname kernel: IPv4: Conflict detected for IP address 192.168.1.100 with MAC address 00:11:22:33:44:55
Jun 04 12:55:11 hostname kernel: eth0: IP conflict detected, interface disabled

설명:
첫 번째 줄은 IP 주소 192.168.1.100에 대해 다른 장치의 MAC 주소(00:11:22:33:44:55)가 응답했음을 나타냅니다.

두 번째 줄은 충돌로 인해 인터페이스(eth0)가 비활성화되었음을 나타냅니다.

로그는 journalctl -k 명령으로도 확인 가능:
bash

journalctl -k | grep -i conflict

3. NetworkManager와의 상호작용
RHEL 7.x에서는 기본적으로 NetworkManager가 네트워크 설정을 관리합니다. IP 충돌이 발생하면 NetworkManager는 커널의 충돌 감지 정보를 받아 동작을 조정합니다:
동작:
NetworkManager는 충돌을 감지하면 인터페이스를 재설정하거나 새로운 IP 주소를 요청(DHCP 환경)합니다.

GUI 또는 nmcli를 통해 사용자에게 충돌 경고를 표시할 수 있습니다.

로그:
/var/log/messages 또는 journalctl -u NetworkManager에 기록:

Jun 04 12:55:15 hostname NetworkManager[1234]: <warn>  [1622801715.1234] device (eth0): IP configuration conflict detected: 192.168.1.100
Jun 04 12:55:16 hostname NetworkManager[1234]: <info>  [1622801716.1234] device (eth0): Requesting new IP address

4. DHCP 환경에서의 처리
RHEL 7.x에서 DHCP 클라이언트(dhclient)는 IP 충돌을 감지하면 다음과 같은 동작을 수행합니다:
충돌 감지: DHCP 서버로부터 받은 IP 주소가 이미 사용 중인 경우, dhclient는 충돌을 감지하고 서버에 새로운 IP 주소를 요청합니다.

로그:

Jun 04 12:55:20 hostname dhclient[5678]: DHCP IP conflict detected for 192.168.1.100
Jun 04 12:55:21 hostname dhclient[5678]: Sending DHCPDECLINE for 192.168.1.100
Jun 04 12:55:22 hostname dhclient[5678]: Requesting new IP address

동작: dhclient는 DHCPDECLINE 메시지를 보내고 새로운 IP 주소를 요청합니다.

5. IP 충돌 감지 및 해결 도구
RHEL 7.x에서 IP 충돌을 감지하고 해결하기 위해 사용할 수 있는 도구는 다음과 같습니다:
arping:
IP 주소의 중복 여부를 확인:
bash

arping -D -I eth0 192.168.1.100

-D는 중복 주소 감지 모드, -I는 인터페이스를 지정.

tcpdump:
ARP 패킷을 캡처하여 충돌 확인:
bash

tcpdump -i eth0 arp

nmcli:
NetworkManager 상태 확인:
bash

nmcli device status
nmcli connection show

로그 확인:
journalctl 또는 /var/log/messages로 관련 로그 확인:
bash

journalctl -u NetworkManager
journalctl -k | grep -i conflict

6. 해결 방법
정적 IP 환경:
/etc/sysconfig/network-scripts/ifcfg-<interface> 파일을 수정하여 고유한 IP 주소를 설정:
bash

vi /etc/sysconfig/network-scripts/ifcfg-eth0
IPADDR=192.168.1.101

네트워크 재시작:
bash

systemctl restart network

DHCP 환경:
dhclient를 재실행하여 새로운 IP 주소를 요청:
bash

dhclient -r eth0 && dhclient eth0

NetworkManager:
nmcli로 인터페이스 재설정:
bash

nmcli con down eth0 && nmcli con up eth0

커널 설정 조정:
arp_notify 활성화로 충돌 로그 강화:
bash

echo "net.ipv4.conf.all.arp_notify = 1" >> /etc/sysctl.d/99-custom.conf
sysctl -p /etc/sysctl.d/99-custom.conf

7. 추가 고려 사항
RHEL 7.x의 기본 설정: RHEL 7.x는 보안과 안정성을 위해 기본적으로 arp_notify가 비활성화되어 있을 수 있으므로, 충돌 로그를 확인하려면 이를 명시적으로 활성화해야 합니다.

보안: IP 충돌은 ARP 스푸핑 공격의 징후일 수 있습니다. tcpdump 또는 wireshark로 ARP 트래픽을 분석하여 의심스러운 활동을 확인하세요.

SELinux: RHEL 7.x는 SELinux를 기본적으로 사용하므로, 네트워크 관련 작업 시 SELinux 정책이 영향을 미칠 수 있습니다. 필요 시 audit.log 확인:
bash

tail -f /var/log/audit/audit.log

요약
RHEL 7.x 커널은 ARP 프로토콜과 DAD를 통해 IP 충돌을 감지하며, 충돌 시 인터페이스를 비활성화하거나 로그를 기록합니다. 로그는 /var/log/messages 또는 journalctl에 기록되며, 예: IPv4: Conflict detected for IP address 192.168.1.100. NetworkManager와 dhclient는 충돌을 감지하고 새로운 IP 주소를 요청하며, arping, tcpdump 등으로 충돌을 진단할 수 있습니다. sysctl 설정(arp_notify, arp_ignore)으로 충돌 처리 동작을 조정 가능합니다.

