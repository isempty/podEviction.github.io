# podEviction.github.io

### 직접 답변

- Kubernetes는 pod eviction 전에 특정 동작을 직접적으로 지원하지 않지만, 커스텀 컨트롤러를 통해 이를 구현할 수 있습니다.  
- OVS pod의 flow cache 정리와 같은 작업은 pod 삭제 이벤트를 모니터링하고 노드에서 실행하는 방식으로 가능합니다.  
- 디스크 압력(예: /var/spool 꽉 참)을 줄이기 위한 리소스 관리도 중요합니다.  

#### 배경 설명  
Kubernetes에서 pod는 리소스 부족(예: 디스크 공간)으로 인해 eviction될 수 있습니다. 특히 /var/spool이 90% 이상 차면 pod가 강제로 종료되며, OVS pod가 eviction되면 flow cache가 남아 네트워크 문제가 발생할 수 있습니다. 이를 해결하려면 eviction 전에 특정 동작(예: flow cache 정리)을 삽입하거나, 특정 pod의 종료 방식을 변경하고 싶으신 것 같습니다.  

#### 가능한 해결책  
Kubernetes는 기본적으로 pod eviction 전에 커스텀 동작을 직접 지원하지 않지만, 다음 방법으로 원하는 결과를 얻을 수 있습니다:  
- **커스텀 컨트롤러 구현**: pod 삭제 이벤트를 감지하고, OVS pod라면 노드에서 flow cache를 정리하는 스크립트를 실행하는 컨트롤러를 작성하세요. 예를 들어, Kubernetes API를 사용해 pod 삭제를 모니터링하고, `ovs-ofctl` 명령으로 flow를 삭제할 수 있습니다.  
- **리소스 관리 최적화**: /var/spool의 디스크 사용을 줄이기 위해 로그 회전 설정이나 리소스 요청/한계를 조정하세요. 이는 eviction 자체를 줄이는 데 도움을 줄 수 있습니다.  
- **우선순위 설정**: OVS pod에 높은 우선순위를 부여해 eviction 가능성을 낮출 수 있습니다.  

이 방법들은 개발 및 설정이 필요하지만, 네트워크 안정성을 높이는 데 효과적입니다. 자세한 구현은 아래 보고서에서 더 설명드리겠습니다.  

---

### 보고서

Kubernetes 환경에서 /var/spool 디스크 사용이 90% 이상으로 차면서 pod eviction이 발생하고, 특히 Open vSwitch(OVS) pod가 eviction되면 flow cache가 남아 네트워크 연결 문제가 발생하는 상황을 해결하려면 특정 동작을 삽입하거나 pod 종료 방식을 변경하는 것이 필요합니다. 이 보고서는 문제의 배경, 가능한 해결책, 그리고 세부 구현 방안을 다룹니다.

#### 문제 배경
- **/var/spool의 역할**: Linux에서 /var/spool은 스풀링 데이터(예: 프린터 큐, 이메일 등)를 저장하는 디렉토리로, 처리 대기 중인 데이터를 임시로 보관합니다. 이 디렉토리가 꽉 차면 디스크 압력이 발생해 Kubernetes 노드에서 리소스 부족으로 pod eviction이 트리거됩니다.  
- **pod eviction의 원인**: Kubernetes의 kubelet은 노드의 디스크 공간, 메모리, CPU 사용량을 모니터링하며, 특정 임계값(threshold)을 초과하면 pod을 강제로 종료합니다. 이는 노드 안정성을 유지하기 위한 메커니즘입니다.  
- **OVS pod와 flow cache 문제**: OVS pod가 eviction되면, 해당 pod와 관련된 flow cache가 노드의 OVS 데몬에 남아 있을 수 있습니다. 이는 네트워크 연결 문제(예: 쿼럼 서버로만 통신 가능, 다른 세션 연결 끊김)를 유발할 수 있습니다.  

#### 가능한 해결책
Kubernetes는 기본적으로 pod eviction 전에 커스텀 동작을 삽입하는 기능을 제공하지 않지만, 다음 방법으로 원하는 결과를 얻을 수 있습니다:

1. **커스텀 컨트롤러 구현**  
   - Kubernetes API를 사용해 pod 삭제 이벤트를 모니터링하는 컨트롤러를 작성할 수 있습니다.  
   - OVS pod(예: `app=ovs` 레이블이 있는 pod)를 식별하고, 해당 pod가 삭제될 때 노드에서 flow cache를 정리하는 스크립트를 실행합니다.  
   - 예를 들어, `ovs-ofctl del-flows <bridge-name> "table=<table-id>,reg0=<pod-id>"` 명령으로 flow를 삭제할 수 있습니다.  
   - 이 컨트롤러는 Kubernetes client-go 라이브러리를 사용해 informer를 설정하고, pod 삭제 이벤트를 처리하며, 노드에서 명령을 실행하기 위해 privileged pod이나 DaemonSet을 사용할 수 있습니다.  

2. **리소스 관리 최적화**  
   - /var/spool의 디스크 사용을 줄이기 위해 로그 회전(log rotation) 설정을 강화하거나, 불필요한 데이터를 정기적으로 정리하세요.  
   - Kubernetes의 kubelet eviction 임계값을 조정해 디스크 압력 발생 시 pod eviction을 지연시킬 수 있습니다. 예를 들어, `eviction-hard`와 `eviction-soft` 설정을 조정하세요 ([Kubernetes Node-Pressure Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/)).  
   - pod의 리소스 요청(request)과 한계(limit)를 적절히 설정해 리소스 사용을 최적화하세요. 예를 들어, 비필수 pod은 리소스 요청을 80-90%로 설정해 노드 오버커밋을 허용할 수 있습니다 ([Ultimate Guide Of Pod Eviction On Kubernetes](https://devtron.ai/blog/ultimate-guide-of-pod-eviction-on-kubernetes/)).  

3. **우선순위 및 QoS 설정**  
   - OVS pod에 높은 우선순위를 부여해 다른 pod보다 늦게 eviction되도록 설정하세요. Kubernetes의 PriorityClass를 사용해 pod의 우선순위를 정의할 수 있습니다 ([Scheduling, Preemption and Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/)).  
   - QoS(Quality of Service) 클래스를 Guaranteed로 설정해 리소스 보장을 강화하고, eviction 가능성을 낮출 수 있습니다.  

4. **기존 도구 활용**  
   - 사용 중인 OVS CNI 플러그인(예: k-vswitch, OVN-Kubernetes)이 pod 삭제 시 flow cache 정리 기능을 제공하는지 확인하세요. 만약 제공하지 않는다면, 커스텀 컨트롤러가 필요합니다.  
   - 예를 들어, k-vswitch는 OVS 기반 네트워킹을 지원하며, 네트워크 정책을 프로그래밍하지만, pod eviction 시 flow cache 정리와 관련된 명시적 기능은 문서에 명시되지 않았습니다 ([k-vswitch GitHub](https://github.com/k-vswitch/k-vswitch)).  

#### 세부 구현 고려사항
- **커스텀 컨트롤러의 실행 환경**: 컨트롤러는 클러스터 내에서 Deployment나 StatefulSet으로 실행되며, 노드에서 명령을 실행하려면 privileged pod이나 SSH 접근이 필요할 수 있습니다. 보안상 주의가 필요합니다.  
- **flow cache 식별**: OVS의 flow cache는 pod의 네트워크 인터페이스와 관련된 flow 항목을 포함하므로, 삭제 시 pod의 UID나 네트워크 인터페이스 정보를 기반으로 식별해야 합니다.  
- **자동화 및 모니터링**: pod eviction 이벤트를 모니터링하기 위해 Prometheus와 Alertmanager를 설정하고, 알림을 기반으로 정리 작업을 트리거할 수 있습니다.  

#### 표: Kubernetes에서 pod eviction 관리 방법 비교

| 방법                  | 장점                                      | 단점                                      | 적합성                     |
|-----------------------|------------------------------------------|------------------------------------------|----------------------------|
| 커스텀 컨트롤러 구현   | 유연성 높음, 특정 동작 커스터마이징 가능 | 개발 및 유지보수 필요, 보안 고려사항 있음 | OVS flow cache 정리에 적합 |
| 리소스 관리 최적화     | 간단, 클러스터 안정성 향상               | 근본적 해결 아님, eviction 방지만 가능    | 디스크 압력 완화에 적합     |
| 우선순위 및 QoS 설정   | eviction 가능성 낮춤, 간단한 설정         | 모든 상황에서 효과적 아님                | 중요 pod 보호에 적합        |
| 기존 도구 활용         | 빠른 구현 가능, 표준화된 접근법           | 기능 제한 가능, 커스터마이징 어려움      | CNI 플러그인 지원 시 적합  |

#### 결론
Kubernetes는 pod eviction 전에 특정 동작을 직접 지원하지 않지만, 커스텀 컨트롤러를 통해 OVS pod의 flow cache 정리와 같은 작업을 구현할 수 있습니다. 또한, /var/spool 디스크 사용을 줄이고, pod 우선순위를 설정해 eviction 가능성을 낮추는 방법도 효과적입니다. 이 접근법은 네트워크 안정성을 높이고, 클러스터의 전반적인 성능을 개선하는 데 기여할 것입니다.

---

### 주요 인용

- [Kubernetes Node-Pressure Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/)
- [Scheduling, Preemption and Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/)
- [Ultimate Guide Of Pod Eviction On Kubernetes](https://devtron.ai/blog/ultimate-guide-of-pod-eviction-on-kubernetes/)
- [k-vswitch GitHub](https://github.com/k-vswitch/k-vswitch)





### 주요 요약
- Kubernetes에서 pod eviction 전에 특정 동작을 삽입하거나 특정 pod의 종료 방식을 변경하는 것은 가능하지만, 추가적인 설정과 구현이 필요합니다.
- OVS pod의 flow cache를 정리하려면 커스텀 컨트롤러를 사용해 pod 삭제 이벤트를 감지하고, `ovs-ofctl` 명령어를 실행할 수 있습니다.
- 디스크 압력을 줄이거나 pod 우선순위를 설정해 eviction 가능성을 낮출 수도 있습니다.

---

### 간단한 설명

#### OVS Flow Cache 정리 명령어 설명
`ovs-ofctl del-flows <bridge-name> "table=<table-id>,reg0=<pod-id>"` 명령어는 OVS 브리지에서 특정 pod와 관련된 flow를 삭제하는 데 사용됩니다. 여기서:
- `<bridge-name>`: OVS 브리지 이름(예: k-vswitch0).
- `<table-id>`: flow가 저장된 테이블 ID(설정에 따라 다름).
- `<pod-id>`: pod를 식별하는 값, 보통 pod의 IP 주소나 UID로 설정됩니다.

이 명령어는 pod 삭제 시 해당 pod의 flow cache를 정리해 네트워크 문제를 방지합니다. 예를 들어, pod의 IP 주소가 10.244.1.5라면, `ovs-ofctl del-flows k-vswitch0 "reg0=10.244.1.5"`로 flow를 삭제할 수 있습니다.

#### 구현 단계
1. **OVS 브리지 이름 확인**: `ovs-vsctl show` 명령어로 브리지 이름(예: k-vswitch0)을 확인하세요.
2. **Table ID 확인**: `ovs-ofctl dump-flows <bridge-name>`로 flow 테이블을 확인해 pod 관련 flow가 있는 테이블을 찾습니다.
3. **Pod ID 매핑**: 삭제된 pod의 IP 주소(status.podIP)를 가져와 `<pod-id>`로 사용합니다.
4. **커스텀 컨트롤러 설정**: DaemonSet을 만들어 각 노드에서 pod 삭제 이벤트를 감지하고, `ovs-ofctl` 명령어를 실행합니다.

이 과정을 통해 pod eviction 전에 flow cache를 정리할 수 있습니다. 그러나 정확한 설정은 클러스터 환경에 따라 달라질 수 있으니, 테스트 후 적용하세요.

---

### 상세 보고서

Kubernetes 환경에서 /var/spool 디스크 사용률이 90%를 초과하면 pod eviction이 발생하며, 특히 Open vSwitch(OVS) pod이 eviction될 경우 flow cache가 남아 네트워크 연결 문제가 발생할 수 있습니다. 이를 해결하기 위해 pod eviction 전에 특정 동작(예: flow cache 정리)을 삽입하거나, 특정 pod의 종료 방식을 변경하려면 커스텀 컨트롤러를 구현하거나 리소스 관리 방식을 최적화해야 합니다. 아래는 이 과정에 대한 상세한 분석과 구현 방법입니다.

#### 배경 및 문제 분석
- **/var/spool의 역할**: Linux에서 /var/spool은 스풀링 데이터(예: 프린터 큐, 이메일 등)를 저장하는 디렉토리로, 디스크 공간이 부족하면 Kubernetes 노드에서 리소스 부족으로 pod eviction이 트리거됩니다.
- **Pod Eviction 메커니즘**: Kubernetes의 kubelet은 디스크 공간, 메모리, CPU 사용량을 모니터링하며, 특정 임계값을 초과하면 pod을 강제로 종료합니다. 이 과정에서 OVS pod이 eviction되면 flow cache가 남아 네트워크 연결 문제(예: 쿼럼 서버로만 통신, 다른 세션 연결 끊김)를 유발할 수 있습니다.
- **Flow Cache 문제**: OVS pod이 삭제될 때 flow cache가 제대로 정리되지 않으면, 네트워크 트래픽이 잘못된 경로로 전달되어 네트워크 안정성을 저하시킵니다.

#### OVS Flow Cache 정리 명령어 상세 설명
사용자가 언급한 `ovs-ofctl del-flows <bridge-name> "table=<table-id>,reg0=<pod-id>"` 명령어는 OVS 브리지에서 특정 조건을 만족하는 flow를 삭제하는 데 사용됩니다. 이 명령어의 각 구성 요소는 다음과 같습니다:
- **`<bridge-name>`**: OVS 브리지 이름으로, k-vswitch와 같은 CNI 플러그인에서는 보통 "k-vswitch0"로 설정됩니다. 이를 확인하려면 `ovs-vsctl show` 명령어를 실행해 브리지 이름을 확인할 수 있습니다.
- **`<table-id>`**: OVS는 여러 테이블(0~254)을 지원하며, 각 테이블은 특정 flow 처리 단계에 사용됩니다. Pod 관련 flow는 보통 table 0이나 다른 특정 테이블에 저장됩니다. 이를 확인하려면 `ovs-ofctl dump-flows <bridge-name>` 명령어로 현재 flow를 확인하고, pod와 관련된 flow가 있는 테이블을 식별해야 합니다.
- **`<pod-id>`**: Pod를 식별하는 값으로, 보통 pod의 UID, IP 주소, 또는 다른 고유 식별자로 설정됩니다. 사용자가 "reg0=<pod-id>"라고 명시했으므로, reg0 레지스터에 pod의 고유 식별자가 저장된 것으로 보입니다. 예를 들어, pod의 IP 주소(예: 10.244.1.5)를 64비트 값으로 변환해 사용하거나, pod UID를 해시화한 값을 사용할 수 있습니다.

이 명령어는 특정 테이블에서 reg0 값이 `<pod-id>`와 일치하는 모든 flow를 삭제합니다. 예를 들어, pod의 IP가 10.244.1.5이고, table 0에 flow가 저장되었다면, `ovs-ofctl del-flows k-vswitch0 "table=0,reg0=10.244.1.5"`로 flow를 삭제할 수 있습니다.

#### 구현 방법: 처음부터 단계별 설명
다음은 `ovs-ofctl del-flows <bridge-name> "table=<table-id>,reg0=<pod-id>"` 명령어를 사용해 pod 삭제 시 flow cache를 정리하는 과정을 단계별로 설명합니다.

1. **환경 준비**
   - Kubernetes 클러스터에서 OVS가 설치되어 있어야 하며, k-vswitch와 같은 OVS 기반 CNI 플러그인을 사용 중이라고 가정합니다.
   - 각 노드에서 `ovs-ofctl` 명령어를 실행할 수 있는 권한이 필요합니다. 이를 위해 DaemonSet을 privileged 모드로 실행해야 합니다.

2. **OVS 브리지 이름 확인**
   - 각 노드에서 `ovs-vsctl show` 명령어를 실행해 OVS 브리지 이름을 확인합니다. 예를 들어, k-vswitch에서는 보통 "k-vswitch0"가 사용됩니다.
   - 확인 명령어 예: `ovs-vsctl show | grep Bridge`, 출력에서 브리지 이름(예: k-vswitch0)을 찾습니다.

3. **Table ID 및 Pod ID 매핑 확인**
   - `ovs-ofctl dump-flows <bridge-name>` 명령어로 현재 flow를 확인합니다. 예: `ovs-ofctl dump-flows k-vswitch0`.
   - 출력에서 pod와 관련된 flow를 찾고, reg0 값이 어떻게 설정되었는지 확인합니다. 예를 들어, "table=0, actions=set(reg0=12345), resubmit(,1)"과 같은 flow가 있다면, reg0=12345가 특정 pod와 연결된 것으로 보입니다.
   - Pod ID는 보통 pod의 IP 주소(status.podIP)로 간주할 수 있습니다. 예를 들어, pod의 IP가 10.244.1.5라면, reg0=10.244.1.5로 설정된 flow를 삭제 대상으로 합니다.
   - Table ID는 pod 관련 flow가 있는 테이블 번호로, 보통 table 0 또는 다른 특정 테이블입니다. 만약 특정 테이블을 모르겠다면, 모든 테이블에서 삭제하려면 "table=<table-id>"를 생략하고 `ovs-ofctl del-flows k-vswitch0 "reg0=<pod-ip>"`로 실행할 수 있습니다.

4. **Pod 삭제 이벤트 감지**
   - Kubernetes API를 사용해 pod 삭제 이벤트를 감지해야 합니다. 이를 위해 Go 언어와 client-go 라이브러리를 사용한 커스텀 컨트롤러를 작성하거나, DaemonSet을 통해 각 노드에서 로컬로 처리할 수 있습니다.
   - 예를 들어, DaemonSet을 사용하면 각 노드에서 실행되는 pod이 해당 노드에서 삭제된 pod을 감지할 수 있습니다. 이를 위해 RBAC 권한을 설정해 pod 목록을 조회할 수 있도록 합니다.

5. **Flow Cache 정리 실행**
   - 삭제된 pod의 IP 주소를 가져옵니다. Kubernetes API에서 DELETED 이벤트의 마지막 상태를 통해 status.podIP를 얻을 수 있습니다.
   - 해당 노드에서 `ovs-ofctl del-flows <bridge-name> "reg0=<pod-ip>"` 명령어를 실행합니다. 예: `ovs-ofctl del-flows k-vswitch0 "reg0=10.244.1.5"`.
   - 만약 특정 테이블만 삭제하려면, `ovs-ofctl del-flows k-vswitch0 "table=0,reg0=10.244.1.5"`와 같이 table을 명시할 수 있습니다.

6. **보안 및 권한 설정**
   - DaemonSet pod은 hostNetwork: true와 privileged: true로 설정해 노드의 OVS에 접근할 수 있도록 합니다.
   - RBAC를 통해 Kubernetes API 접근 권한을 부여합니다. 예: ClusterRole과 ClusterRoleBinding을 설정해 pod 목록 조회 및 감시 권한을 부여합니다.

7. **테스트 및 검증**
   - 테스트 환경에서 pod을 강제로 eviction 시키고, `ovs-ofctl dump-flows k-vswitch0`로 flow가 제대로 삭제되었는지 확인합니다.
   - 네트워크 연결 문제를 모니터링해 flow cache 정리 후 문제가 해결되었는지 검증합니다.

#### 추가 고려사항
- **CNI DEL 호출 문제**: Pod eviction 시 CNI DEL 명령어가 제대로 호출되지 않을 수 있습니다. 이 경우, 커스텀 컨트롤러가 추가적인 flow 정리 로직을 보완할 수 있습니다.
- **Flow 구조 분석**: OVS flow는 여러 테이블에 걸쳐 있을 수 있으므로, 모든 관련 flow를 삭제하려면 테이블별로 검토가 필요합니다. 예를 들어, table 0에서 in_port를 기반으로 reg0를 설정하고, 이후 테이블에서 reg0를 사용한다면, 모든 테이블에서 reg0=<pod-id>를 기준으로 삭제해야 할 수 있습니다.
- **대안 접근법**: Flow를 in_port 기준으로 삭제하는 방법도 있습니다. 이를 위해 ovsdb에서 pod와 연결된 OVS 포트를 조회한 후, `ovs-ofctl del-flows <bridge-name> "in_port=<port-no>"`로 삭제할 수 있습니다. 그러나 pod 삭제 시 OVS 포트가 이미 제거되었다면, 이 방법은 제한적일 수 있습니다.

#### 리소스 관리 최적화
- **디스크 압력 줄이기**: /var/spool 디스크 사용을 줄이기 위해 로그 회전 설정을 강화하거나 불필요한 데이터를 정기적으로 정리하세요. 예: logrotate 설정을 조정해 로그 파일 크기를 제한합니다.
- **Kubelet 설정 조정**: `eviction-hard`와 `eviction-soft` 설정을 변경해 디스크 압력 임계값을 높입니다. 예: `/etc/kubernetes/kubelet.conf`에서 `eviction-hard: 'nodefs.available<5%,nodefs.inodesFree<5%'`로 설정.
- **Pod 리소스 설정**: 비필수 pod의 리소스 요청(request)과 한계(limit)를 조정해 노드 오버커밋을 허용합니다. 예: CPU 요청을 80-90%로 설정.

#### 우선순위 및 QoS 설정
- **PriorityClass 사용**: OVS pod에 높은 우선순위를 부여해 eviction 가능성을 낮춥니다. 예: PriorityClass YAML을 작성해 value를 1000000으로 설정하고, OVS pod에 적용합니다.
- **QoS 클래스**: OVS pod의 QoS를 Guaranteed로 설정해 리소스 보장을 강화합니다. 예: CPU와 Memory 요청과 한계를 동일하게 설정.

#### 표: Kubernetes에서 pod eviction 관리 방법 비교

| 방법                  | 장점                                      | 단점                                      | 적합성                     |
|-----------------------|------------------------------------------|------------------------------------------|----------------------------|
| 커스텀 컨트롤러 구현   | 유연성 높음, 특정 동작 커스터마이징 가능 | 개발 및 유지보수 필요, 보안 고려사항 있음 | OVS flow cache 정리에 적합 |
| 리소스 관리 최적화     | 간단, 클러스터 안정성 향상               | 근본적 해결 아님, eviction 방지만 가능    | 디스크 압력 완화에 적합     |
| 우선순위 및 QoS 설정   | eviction 가능성 낮춤, 간단한 설정         | 모든 상황에서 효과적 아님                | 중요 pod 보호에 적합        |

#### 결론
Kubernetes에서 pod eviction 전에 특정 동작을 삽입하거나 특정 pod의 종료 방식을 변경하려면 커스텀 컨트롤러를 구현하거나 리소스 관리 방식을 최적화하는 것이 필요합니다. `ovs-ofctl del-flows <bridge-name> "table=<table-id>,reg0=<pod-id>"` 명령어를 사용해 OVS flow cache를 정리할 수 있으며, 이를 위해 DaemonSet과 Kubernetes API를 활용한 구현이 효과적입니다. 또한, 디스크 압력 줄이기와 pod 우선순위 설정으로 eviction 가능성을 낮출 수 있습니다. 클러스터 환경에 따라 테스트 후 적용하세요.

---

### 주요 인용
- [Kubernetes Node-Pressure Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/)
- [Scheduling, Preemption and Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/)
- [Ultimate Guide Of Pod Eviction On Kubernetes](https://devtron.ai/blog/ultimate-guide-of-pod-eviction-on-kubernetes/)
- [k-vswitch GitHub](https://github.com/k-vswitch/k-vswitch)
