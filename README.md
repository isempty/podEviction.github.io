직접 답변
OpenStack에서 디스크를 삭제하면 호스트 커널에서 디스크가 여전히 존재하는 것처럼 I/O 오류가 발생할 수 있습니다. 이는 삭제 후 디바이스 정보가 완전히 정리되지 않았기 때문입니다.

호스트 커널은 디스크 파티션 정보를 /sys/block와 /dev에 저장하며, 삭제가 제대로 동기화되지 않으면 오류가 발생합니다.

OpenStack 삭제와 직접 삭제의 차이는 OpenStack이 클라우드 상태를 업데이트하지만 호스트 커널과 멀티패스 설정은 직접적으로 관리하지 않는 점입니다.

삭제 후 오류 발생 원인
OpenStack에서 디스크를 삭제하면 Cinder 서비스가 볼륨을 제거하지만, 호스트 커널과 멀티패스d(multipathd)는 즉시 이를 인식하지 못할 수 있습니다. 이는 디바이스 파일(/dev/sdX)이나 멀티패스 경로가 남아 있어, 커널이 접근 시도 시 "blk_update_request I/O error"나 "Buffer I/O error"와 같은 오류를 발생시킬 수 있습니다. 특히, 멀티패스 설정이 있는 경우 삭제된 볼륨의 경로가 정리되지 않으면 추가 오류가 발생할 수 있습니다.
호스트 커널의 디스크 관리 방식
호스트 커널은 디스크 파티션 정보를 메모리와 /sys/block, /dev에 저장합니다. 디스크가 물리적으로 제거되거나 접근 불가능해지면 udev 이벤트를 통해 업데이트됩니다. 하지만 OpenStack의 삭제는 소프트웨어적 작업으로, 커널이 이를 즉시 인식하지 못하면 디바이스 엔트리가 남아 오류를 유발할 수 있습니다.
OpenStack 삭제 vs 직접 삭제
OpenStack 삭제: 클라우드 상태(Cinder 데이터베이스)를 업데이트하고 스토리지 백엔드(LVM, iSCSI 등)를 제거하지만, 호스트 커널이나 멀티패스d의 업데이트는 자동으로 이루어지지 않을 수 있습니다.

직접 삭제: 호스트에서 수동으로 디스크를 제거하면 커널과 멀티패스d가 즉시 업데이트되지만, OpenStack 상태는 일관성 있게 유지되지 않을 수 있습니다.

이 문제는 커널 버전이나 OpenStack 설정에 따라 달라질 수 있으며, 관련 커뮤니티에서 비슷한 사례가 보고되었습니다. 예를 들어, 특정 커널 버전(예: Ubuntu 14.04.3, 3.19.x)에서 iSCSI와 멀티패스 관련 문제가 해결된 사례가 있습니다 

.
보고서
OpenStack와 KVM 가상화 환경에서 디스크 삭제 후 호스트 오류 분석
OpenStack와 KVM 가상화를 사용하는 Linux 환경에서 디스크 또는 VM OS를 OpenStack을 통해 삭제하면, 호스트 OS 로그에서 "blk_update_request I/O error"와 "Buffer I/O error"가 발생하며, 관련된 멀티패스d(multipathd)에서도 오류가 보고됩니다. 이는 디스크 삭제가 빈번한 작업임에도 불구하고, 호스트 커널이 해당 디스크 파티션 정보를 어떻게 관리하며, 왜 삭제 후에도 마치 디스크가 존재하는 것처럼 오류가 발생하는지에 대한 중요한 질문입니다. 또한, OpenStack을 통해 삭제하는 것과 직접 디스크를 삭제하는 것 간의 차이를 상세히 분석하겠습니다.
오류 발생 원인 및 배경
OpenStack에서 디스크(볼륨)를 삭제하면, Cinder 서비스가 해당 볼륨을 스토리지 백엔드에서 제거하는 과정을 수행합니다. 이 과정은 다음과 같습니다:
볼륨이 VM에 연결되어 있다면, Nova를 통해 분리(detach) 작업을 수행.

Cinder 데이터베이스에서 볼륨 정보를 삭제.

스토리지 백엔드(예: LVM, iSCSI)에서 물리적 또는 논리적 스토리지를 제거.

그러나 이 과정은 호스트 커널과 멀티패스d와의 동기화가 완벽하지 않을 수 있습니다. 특히, 호스트 커널은 디스크 파티션 정보를 /sys/block와 /dev에 저장하며, 디스크가 물리적으로 제거되거나 접근 불가능해지면 udev 이벤트를 통해 업데이트됩니다. 하지만 OpenStack의 삭제는 소프트웨어적 작업으로, 커널이 이를 즉시 인식하지 못하면 디바이스 엔트리가 남아 있을 수 있습니다. 이 경우, 커널이 해당 디바이스에 접근 시도 시 "blk_update_request I/O error" 또는 "Buffer I/O error"와 같은 오류가 발생합니다.
멀티패스d의 경우, SAN이나 iSCSI 스토리지와 같은 환경에서 여러 경로를 관리하며, 볼륨 삭제 후 해당 경로가 제대로 정리되지 않으면 멀티패스d가 여전히 접근을 시도하여 오류를 발생시킬 수 있습니다. 이는 특히 iSCSI 또는 멀티패스 설정이 활성화된 환경에서 자주 관찰됩니다.
호스트 커널의 디스크 파티션 정보 관리
Linux 커널은 블록 디바이스(디스크)를 관리하기 위해 블록 레이어를 사용하며, 디스크 파티션 정보는 다음과 같이 저장됩니다:
/sys/block: 각 블록 디바이스의 계층적 정보가 저장됨.

/dev: 디바이스 파일(예: /dev/sdX, /dev/vda)이 생성되어 사용자 공간에서 접근 가능.

디바이스 매퍼(Device Mapper): LVM과 같은 논리적 볼륨은 /dev/mapper/* 또는 /dev/dm-*로 나타남.

멀티패스d: SAN 환경에서는 /dev/mapper/mpathX와 같은 가상 디바이스를 생성하여 여러 경로를 통합 관리.

디스크가 제거되면, 커널은 udev 이벤트를 통해 디바이스 엔트리를 업데이트해야 합니다. 그러나 가상화 환경에서는 디스크 제거가 물리적 이벤트가 아니라 소프트웨어적 작업이므로, 커널이 이를 즉시 인식하지 못할 수 있습니다. 이는 삭제된 디바이스가 여전히 커널 메모리나 시스템 파일에 남아 있어, 접근 시도 시 I/O 오류를 유발할 수 있습니다.
유사 문제 검색 결과
관련 커뮤니티와 문서에서 유사한 문제가 보고되었습니다:
Ask OpenStack: VM filesystem shutting down에서는 VM에서 무거운 작업을 수행할 때 "blk_update_request I/O error"와 "Buffer I/O error"가 발생한 사례가 있으며, 이는 커널 버전(예: Ubuntu 14.04.3, 3.19.x)과 멀티패스 설정 문제로 추정됨.

OpenStack Operators Mailing List: Buffer I/O error on Cinder volumes에서는 Cinder 볼륨에서 Buffer I/O 오류가 발생했으며, 커널을 3.16.x로 다운그레이드하여 해결된 사례가 있음.

Red Hat Bugzilla: blk_update_request I/O error with qcow2 on Btrfs에서는 Btrfs 파일 시스템에서 qcow2 이미지와 관련된 유사한 오류가 보고됨.

이러한 사례들은 커널 버전, 스토리지 백엔드, 그리고 멀티패스 설정이 오류 발생에 중요한 역할을 한다는 점을 시사합니다.
OpenStack을 통해 삭제 vs 직접 삭제: 상세 분석
OpenStack을 통해 디스크를 삭제하는 것과 호스트에서 직접 삭제하는 것 간의 차이는 다음과 같습니다:
항목

OpenStack을 통해 삭제

직접 삭제 (호스트에서)

프로세스

OpenStack CLI(
openstack volume delete
) 또는 대시보드를 통해 실행. Cinder가 볼륨 분리 및 스토리지 백엔드 제거 수행.

lvremove
, 
wipefs
 등으로 호스트에서 수동 제거. 커널이 즉시 업데이트.

영향 범위

클라우드 상태(Cinder DB)와 스토리지 백엔드에 영향. 호스트 커널 및 멀티패스d는 직접 관리되지 않음.

호스트 커널과 멀티패스d가 즉시 업데이트되지만, OpenStack 상태는 일관성 유지되지 않음.

청소 과정

Cinder가 스토리지 백엔드 정리, 하지만 호스트 커널 및 멀티패스d 동기화는 보장되지 않음.

호스트 커널과 멀티패스d가 즉시 정리되지만, OpenStack 상태 업데이트 없음.

오류 처리

삭제 실패 시 "error_deleting" 상태에 머물 수 있으며, 수동 정리 필요.

호스트에서 오류 발생 가능성 낮음, 하지만 OpenStack 상태 불일치로 인한 문제 발생 가능.

이 차이는 OpenStack 삭제가 클라우드 관리 레벨에서 이루어지며, 호스트 커널과 멀티패스d의 저수준 작업은 별도로 관리되지 않는다는 점에서 기인합니다. 예를 들어, OpenStack 삭제 후 디바이스 파일이 남아 있으면, 커널이 이를 접근 시도하며 오류를 발생시킬 수 있습니다.
권장 조치
문제를 해결하기 위해 다음을 고려할 수 있습니다:
삭제 후 호스트에서 디바이스 파일(/dev/vda, /dev/mapper/mpathX 등)이 남아 있는지 확인하고, 필요 시 수동으로 제거(dmsetup remove 사용).

멀티패스d 로그를 확인하여 삭제된 볼륨과 관련된 경로 오류가 있는지 점검하고, multipath -r로 구성 재로딩.

커널 로그(/var/log/syslog, dmesg)에서 "blk_update_request I/O error" 또는 "Buffer I/O error"와 같은 오류를 모니터링.

iSCSI 또는 멀티패스 설정이 활성화된 경우, use_multipath_for_image_xfer 옵션을 비활성화하여 문제 해결 시도.

커널 버전과 OpenStack 릴리스가 알려진 문제(예: 3.19.x 커널의 iSCSI 문제)와 관련이 있는지 확인.

결론
OpenStack에서 디스크를 삭제한 후 호스트에서 I/O 오류가 발생하는 이유는 호스트 커널과 멀티패스d가 삭제를 즉시 인식하지 못하고, 디바이스 엔트리나 경로가 남아 있기 때문입니다. 호스트 커널은 디스크 파티션 정보를 /sys/block와 /dev에 저장하며, OpenStack 삭제는 이 과정과 동기화되지 않을 수 있습니다. OpenStack을 통해 삭제하는 것과 직접 삭제하는 것의 차이는 관리 레벨과 청소 과정의 차이로, 적절한 동기화와 청소가 중요합니다. 관련 커뮤니티 사례와 문서를 참고하여 문제를 해결할 수 있습니다.
주요 인용
Ask OpenStack: VM filesystem shutting down

OpenStack Operators Mailing List: Buffer I/O error on Cinder volumes

Red Hat Bugzilla: blk_update_request I/O error with qcow2 on Btrfs

OpenStack Cinder Documentation: Manage Volumes

36 웹페이지

