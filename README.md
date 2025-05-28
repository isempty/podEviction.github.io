주요 요약
AWS에서는 물리적 코어 하나의 두 하이퍼스레드가 일반적으로 한 쌍으로 VM에 할당됩니다.
OpenStack에서는 설정에 따라 하이퍼스레드가 한 쌍으로 묶일 수도, 그렇지 않을 수도 있습니다.
하이퍼스레딩 코어를 한 쌍으로 묶는 기술은 CPU 핀닝입니다.
AWS의 하이퍼스레딩 할당
AWS EC2 인스턴스에서는 대부분의 인스턴스 유형에서 하이퍼스레딩이 활성화되어 있으며, 각 vCPU는 물리적 CPU 코어의 한 하이퍼스레드를 나타냅니다. 예를 들어, c5.xlarge 인스턴스는 4개의 vCPU를 가지며, 이는 2개의 물리적 코어에서 각각 2개의 하이퍼스레드가 할당된 것으로 보입니다. AWS 문서에 따르면, "Each vCPU is a full hardware thread"로, 이는 하이퍼스레딩이 활성화된 경우 물리적 코어당 두 개의 vCPU가 쌍으로 할당된다는 것을 암시합니다. 따라서 AWS에서는 물리적 코어 하나의 두 하이퍼스레드가 한 쌍으로 VM에 할당되는 것으로 보입니다.
OpenStack의 하이퍼스레딩 할당
OpenStack은 KVM을 사용하며, 하이퍼스레딩 코어의 할당은 설정에 따라 달라질 수 있습니다. OpenStack에서는 libvirt 드라이버를 통해 CPU 핀닝을 사용하여 vCPU를 특정 pCPU에 고정할 수 있습니다. 이는 하이퍼스레드를 쌍으로 묶거나 분리할 수 있음을 의미합니다. 예를 들어, VM의 CPU 토폴로지를 정의하여 하이퍼스레드를 쌍으로 묶을 수 있으며, 이는 플레이버(Flavor) 설정이나 인스턴스 생성 시 조정 가능합니다. 따라서 OpenStack에서는 하이퍼스레딩 코어가 한 쌍으로 제공될 수도 있고, 그렇지 않을 수도 있습니다.
하이퍼스레딩 코어를 한 쌍으로 묶는 기술
하이퍼스레딩 코어를 한 쌍으로 묶는 기술은 CPU 핀닝입니다. KVM에서는 libvirt를 통해 VM의 XML 설정에서 CPU 토폴로지를 정의할 수 있으며, 이를 통해 vCPU를 특정 pCPU에 고정할 수 있습니다. 예를 들어, 물리적 코어 하나의 두 하이퍼스레드를 쌍으로 묶으려면 해당 vCPU를 해당 pCPU에 핀핑하면 됩니다. OpenStack에서도 이러한 설정이 가능하며, 관리자는 필요에 따라 조정할 수 있습니다.
상세 보고서
소개
본 보고서는 KVM 환경에서 하이퍼스레딩이 활성화된 경우, 물리적 코어 하나가 두 개의 논리적 코어로 동작하며, 이를 VM에 어떻게 할당하는지에 대한 기술적 세부 사항을 다룹니다. 특히, AWS와 OpenStack 환경에서의 하이퍼스레딩 코어 할당 방식과, 하이퍼스레딩 코어를 한 쌍으로 묶는 기술에 대해 자세히 설명합니다. 현재 시간은 2025년 5월 28일 수요일 오후 5시 54분 KST입니다.
AWS에서의 하이퍼스레딩 코어 할당
AWS EC2 인스턴스에서는 대부분의 인스턴스 유형에서 하이퍼스레딩(Simultaneous Multithreading, SMT)이 활성화되어 있습니다. 각 vCPU는 물리적 CPU 코어의 한 하이퍼스레드를 나타내며, 이는 AWS 문서 CPU options for Amazon EC2 instances에서 확인할 수 있습니다. 문서에 따르면, "Each vCPU is a thread of a CPU core, except for T2 instances, M7a instances, Apple silicon Mac instances, and 64-bit ARM platforms such as instances powered by AWS Graviton processors." 이는 대부분의 인스턴스에서 vCPU가 하이퍼스레드임을 나타냅니다.
예를 들어, c5.xlarge 인스턴스는 4개의 vCPU를 가지며, 이는 2개의 물리적 코어에서 각각 2개의 하이퍼스레드가 할당된 것으로 보입니다. AWS 문서 Supported CPU options for Amazon EC2 instance types에서 인스턴스 유형별 CPU 설정을 확인할 수 있으며, 표를 통해 하이퍼스레딩이 활성화된 경우 기본적으로 threads per core가 2로 설정되어 있음을 알 수 있습니다. 이는 물리적 코어 하나의 두 하이퍼스레드가 한 쌍으로 VM에 할당된다는 것을 암시합니다.
또한, AWS Compute Blog의 포스트 Disabling Intel Hyper-Threading Technology on Amazon EC2 Windows Instances에서 m4.2xlarge 인스턴스가 4개의 물리적 프로세서와 각 프로세서당 2개의 스레드로 8개의 vCPU를 가지는 예시를 보여주며, 이는 하이퍼스레딩이 활성화된 경우 vCPU가 물리적 코어의 하이퍼스레드 쌍으로 할당됨을 확인할 수 있습니다.
따라서, AWS에서는 사용자의 진술대로 물리적 코어 하나의 두 하이퍼스레드가 한 쌍으로 묶여 VM에 할당됩니다.
OpenStack에서의 하이퍼스레딩 코어 할당
OpenStack은 KVM을 기반으로 하며, 하이퍼스레딩 코어의 할당은 설정에 따라 유연하게 조정될 수 있습니다. OpenStack 문서 VirtDriverGuestCPUMemoryPlacement에서 "If the host has hyperthreading enabled, however, then it is desirable to expose hyperthreading to the guest and at the same time strictly set vCPU<->pCPU affinity even within the node"라고 명시되어 있습니다. 이는 하이퍼스레딩이 활성화된 경우 vCPU와 pCPU 간의 어피니티를 설정하여 하이퍼스레드를 쌍으로 묶거나 분리할 수 있음을 나타냅니다.
OpenStack에서는 libvirt 드라이버를 통해 KVM의 CPU 핀닝 기능을 사용하여 vCPU를 특정 pCPU에 고정할 수 있습니다. 이는 VM의 libvirt XML 설정에서 CPU 토폴로지를 정의하여 소켓, 코어, 그리고 코어당 스레드 수를 지정할 수 있음을 의미합니다. 예를 들어, 관리자는 플레이버(Flavor) 설정을 통해 하이퍼스레드를 쌍으로 묶어 VM에 할당하거나, 그렇지 않게 설정할 수 있습니다.
Stack Overflow 포스트 openstack - vCPUs mapped to CPU on multiple servers에서, OpenStack에서 하이퍼스레딩이 활성화된 경우 vCPU가 논리적 프로세서에 어떻게 매핑되는지에 대한 논의가 있으며, 이는 설정에 따라 달라질 수 있음을 나타냅니다. 따라서 OpenStack에서는 하이퍼스레딩 코어가 한 쌍으로 제공될 수도 있고, 그렇지 않을 수도 있으며, 이는 관리자의 설정에 따라 결정됩니다.
하이퍼스레딩 코어를 한 쌍으로 묶는 기술
하이퍼스레딩 코어를 한 쌍으로 묶는 기술은 CPU 핀닝(CPU Pinning)입니다. KVM에서는 libvirt를 통해 VM의 XML 설정에서 CPU 토폴로지를 정의할 수 있으며, 이는 KVM CPU 핀닝 문서에서 확인할 수 있습니다. CPU 핀닝은 vCPU를 특정 pCPU에 고정하여, 예를 들어 물리적 코어 하나의 두 하이퍼스레드를 쌍으로 묶어 VM에 할당할 수 있습니다.
예를 들어, <cpu mode='host-passthrough'> 설정을 통해 호스트의 CPU 구조를 그대로 VM에 노출하거나, <cpu mode='custom' match='exact'>를 사용하여 사용자 정의 CPU 토폴로지를 적용할 수 있습니다. 이는 vCPU 0을 pCPU 0(첫 번째 하이퍼스레드)에, vCPU 1을 pCPU 1(두 번째 하이퍼스레드)에 핀핑하여 물리적 코어 하나의 하이퍼스레드를 쌍으로 묶는 것을 가능하게 합니다.
OpenStack에서도 이러한 설정이 가능하며, Nova의 CPU 토폴로지 문서 CPU topologies에서 NUMA 호스트에서의 vCPU 배치와 관련된 세부 사항을 확인할 수 있습니다. 이는 관리자가 필요에 따라 하이퍼스레딩 코어를 쌍으로 묶거나 분리할 수 있음을 나타냅니다.
비교 표
다음 표는 AWS와 OpenStack에서의 하이퍼스레딩 코어 할당 방식을 요약합니다:
플랫폼
하이퍼스레딩 코어 할당 방식
설정 가능성
AWS
물리적 코어 하나의 두 하이퍼스레드가 한 쌍으로 기본 할당
SMT 비활성화 가능, 기본적으로 쌍으로 할당
OpenStack
설정에 따라 쌍으로 묶거나 분리 가능
CPU 핀닝 및 토폴로지 설정으로 유연하게 조정 가능
결론
AWS에서는 물리적 코어 하나의 두 하이퍼스레드가 일반적으로 한 쌍으로 VM에 할당되며, 이는 인스턴스 유형의 설계와 하이퍼스레딩 활성화 설정에 따른 것입니다. 반면, OpenStack에서는 KVM을 기반으로 하며, 설정에 따라 하이퍼스레딩 코어가 한 쌍으로 묶일 수도 있고, 그렇지 않을 수도 있습니다. 하이퍼스레딩 코어를 한 쌍으로 묶는 기술은 CPU 핀닝이며, 이는 KVM의 libvirt를 통해 구현됩니다.
주요 인용
CPU options for Amazon EC2 instances
Supported CPU options for Amazon EC2 instance types
Disabling Intel Hyper-Threading Technology on Amazon EC2 Windows Instances
VirtDriverGuestCPUMemoryPlacement OpenStack
openstack vCPUs mapped to CPU on multiple servers
CPU topologies OpenStack Nova
KVM CPU pinning libvirt
