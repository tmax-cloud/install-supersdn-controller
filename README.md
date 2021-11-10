# SuperSDN/v2 Controller (작성 중)

# 목차  
[1. SDN 장비 설정](#1-sdn-장비-설정)  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[1-1. SDN Switch 설정](#1-1-sdn-Switch-설정)  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[1-2. Web UI 설정](#1-2-web-ui-설정)  
[2. 노드 설정](#2-노드-설정)  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[2-1. Network Interface 설정](#2-1-network-interface-설정)  
[3. SDN Controller Container로 기동](#3-sdn-controller-container로-기동)  
[4. SDN 운영기 설치](#4-sdn-운영기-설치)  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[4-1. SDN Controller 설치](#4-1-sdn-controller-설치)  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[4-2. (TBD)다른 SDN module 설치](#4-2tbd-다른-sdn-module-설치)  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[4-3. GW VIP, SNAT public IP를 관리하는 모듈 설치 이후](#4-3-gw-vip-snat-public-ip를-관리하는-모듈-설치-이후)  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[4-4. keepalived 설정](#4-4-keepalived 설정)  
[5. 사용법](#5-사용법)  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[5-1. ProxyARP CR](#5-1-proxyarp-cr)  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[5-2. Monitor](#5-2-monitor)  



# 1. SDN 장비 설정

## 1-1. SDN Switch 설정

- minicom console 연결 [[참조]](https://pinocc.tistory.com/156) (window에서는 putty 권장)
    - speed: 115200 설정하고 serial 접속
    - id/password = admin/password
- SDN 모드 및 IP 세팅
    - #sudo picos_boot
    - SDN 모드 선택, static IP 선택, switch의 management IP (eth0)를 172.31.1.100/24 설정
    - GW설정, UI설정 (O), SNMP설정 (X)
- 세팅 적용 및 재시작
    - #sudo reboot

## 1-2. Web UI 설정

- Web UI로 접속
    - SDN SW의 management port와 laptop을 연결
    - laptop의 ip를 172.31.1.200/24 로 설정
    - 172.31.1.100 으로 접속
        - id/password = admin/password
- br0 설정
    - br0 하단 메뉴 중 Controllers 설정
        
        [https://lh6.googleusercontent.com/gplU6QxhhlbdtKqSx4bhqusEU0dMP1Tuuv3kKeO6Ky7LieEhhZb1-Q4VFM0DuXpPYSEXFP0d01PH93HzpJ4V1_Bl1_Kg4RVoDsuqcJx9gVEXOoTFnclgF3-oLhVBFLb-9w7j48dO](https://lh6.googleusercontent.com/gplU6QxhhlbdtKqSx4bhqusEU0dMP1Tuuv3kKeO6Ky7LieEhhZb1-Q4VFM0DuXpPYSEXFP0d01PH93HzpJ4V1_Bl1_Kg4RVoDsuqcJx9gVEXOoTFnclgF3-oLhVBFLb-9w7j48dO)
        
        - Method : TCP , Connection Mode : out-of-band, IP : 172.31.1.1, Port:6633
        - Method : TCP , Connection Mode : out-of-band, IP : 172.31.1.2, Port:6633
        - 세팅한 IP가 sdn controller(=TCNC)를 가지고 있는 노드의 인터페이스 IP임
    - br0 하단 메뉴 중 Ports 설정
        - add port를 통해 사용할 port 설정 (trunk or access)
        - vlan을 사용하지 않는 경우 uplink port에 대해서도 trunk로 설정
        - 그 외의 경우: trunk로 설정, tags :1

# 2. 노드 설정

- Network infra 구성
    - 노드의 대역은 10.0.0.0/16 , pod 대역은 10.244.0.0/16를 기준으로 작성
    - SDN 망을 관리하기 위한 노드(=SDN Controller가 뜨는 노드)는 두개가 존재함
    - 외부 인터넷을 사용하기 위한 Gateway용 노드가 존재함
    - 클러스터 내부에서 외부와 통신하기 위해서는 Gateway용 노드에서 NAT를 거치게 됨
    - (TBD) ~~외부 노출을 위한 NAT rule 설정은 Gateway agent가, Active Gateway 노드 선별은 SDN watcher가 담당함~~
    - 일반 노드의 default gateway는 가상 default gateway 주소(GW VIP)로 설정
    - Gateway 노드의 default gateway는 실제 물리 gateway로 설정함
    - k8s 노드들은 k8s Master VIP를 통해 k8s API에 접근함. k8s Master VIP는 keepalived를 통해 failover를 수행함.
- 설치 시나리오
    - 노드간 통신과, 외부 통신(SuperSDN 모듈들의 이미지를 다운받기 위함)을 위해 SDN Controller를 container로 기동 해야 함
        - 이 때, 1) 모든 노드, 2) GW VIP, 3) SNAT할 Public IP, 4) k8s Master VIP는 ProxyARP가 등록 되어야 함(tcn_rcfg.json 사용)
        - GW 노드에서 SNAT rule이 내려가 있어야 함(iptables 사용)
    - 모든 SuperSDN 모듈들의 설치가 완료되고나서, SDN Controller를 k8s 위에서 기동 시킨 이후 container로 띄운 HyperSDN Controller를 종료시켜야 함
    - SDN Controller가 최소 하나는 기동 하고 있어야 노드간 통신이 동작하기 때문에, 설치 과정시 주의를 요함.
    - 설치 순서 : Container로 SDN Controller 기동 → k8s 설치 → Pod으로 SDN Controller 하나 기동 → 다른 SuperSDN 모듈들 설치, CustomResource로 ProxyARP 등록 → Container로 기동 중인 SDN Controller 중지 → Pod으로 나머지 SDN Controller 기동

## 2-1. Network interface 설정

- 노드 재기동 시 IP/PMAC 관리를 위한 설정 (/etc/sysconfig/network-scripts/ifcfg-enp1s0, ifcfg-enp2s0,...)
    - default Gateway 설정
        - 일반 노드의 경우 Gateway를 임의의 가상 default gateway 주소(예제에서는 10.0.0.1)로 설정.
        - Gw 노드의 경우 Gateway를 실제 물리 Gateway 주소로 설정함.(예제에서는 172.22.2.1)
    - data 통신용 interface의 경우 PMAC 규칙에 따라 PMAC 발급
        - PMAC 정책
            - fc:AA:AA:BB:00:00
            - **AA:AA : 연결된 스위치의 dpid 끝 4자리**
            - **BB : 연결된 스위치의 포트번호**
    - 인터페이스 모두 ONBOOT=”yes”는 필수로 설정
    - SDN data통신용 interface 파일 - 일반 노드의 경우
    
    ```bash
    TYPE=Ethernet
    PROXY_METHOD=none
    BROWSER_ONLY=no
    BOOTPROTO=none
    DEFROUTE=yes
    IPV4_FAILURE_FATAL=no
    IPV6INIT=yes
    IPV6_AUTOCONF=yes
    IPV6_DEFROUTE=yes
    IPV6_FAILURE_FATAL=no
    IPV6_ADDR_GEN_MODE=stable-privacy
    NAME=enp3s0
    UUID=eda2fa5d-53e8-4eba-a968-c45cc091748c
    DEVICE=enp3s0
    ONBOOT=yes
    IPADDR=10.0.9.1
    PREFIX=16
    IPV6_PRIVACY=no
    MACADDR="FC:C7:C1:0D:00:00"
    GATEWAY=10.0.0.1
    DNS1=8.8.8.8
    ```
    
    - SDN data통신용 interface 파일 - GW 노드의 경우
    
    ```bash
    TYPE=Ethernet
    PROXY_METHOD=none
    BROWSER_ONLY=no
    BOOTPROTO=none
    DEFROUTE=yes
    IPV4_FAILURE_FATAL=no
    IPV6INIT=yes
    IPV6_AUTOCONF=yes
    IPV6_DEFROUTE=yes
    IPV6_FAILURE_FATAL=no
    IPV6_ADDR_GEN_MODE=stable-privacy
    NAME=enp3s0
    UUID=d439aca5-c6c1-4141-b5ba-97cdaa1a2997
    DEVICE=enp3s0
    ONBOOT=yes
    IPADDR=10.0.9.4
    PREFIX=16
    IPV6_PRIVACY=no
    MACADDR="FC:C0:C1:0E:00:00"
    GATEWAY=172.23.2.1
    DNS1=8.8.8.8
    ```
    
    - sdn controller pod가 뜰 대상 노드(2대)의 경우에만 control용 인터페이스 설정을 추가.
    
    ```bash
    TYPE=Ethernet
    PROXY_METHOD=none
    BROWSER_ONLY=no
    BOOTPROTO=static
    DEFROUTE=yes
    IPV4_FAILURE_FATAL=no
    IPV6INIT=yes
    IPV6_AUTOCONF=yes
    IPV6_DEFROUTE=yes
    IPV6_FAILURE_FATAL=no
    IPV6_ADDR_GEN_MODE=stable-privacy
    NAME=enp2s0
    UUID=0445494e-23cd-4e99-953f-2a5924ec2fe6
    DEVICE=enp2s0
    ONBOOT=yes
    IPADDR=172.31.32.1
    PREFIX=16
    ```
    
    - (optional) SDN module을 headless service를 통해 접근할 경우 DNS 설정을 하나 더 추가해야 함.
    
    ```bash
    TYPE=Ethernet
    PROXY_METHOD=none
    BROWSER_ONLY=no
    BOOTPROTO=none
    DEFROUTE=yes
    IPV4_FAILURE_FATAL=no
    IPV6INIT=yes
    IPV6_AUTOCONF=yes
    IPV6_DEFROUTE=yes
    IPV6_FAILURE_FATAL=no
    IPV6_ADDR_GEN_MODE=stable-privacy
    NAME=enp3s0
    UUID=eda2fa5d-53e8-4eba-a968-c45cc091748c
    DEVICE=enp3s0
    ONBOOT=yes
    IPADDR=10.0.9.1
    PREFIX=16
    IPV6_PRIVACY=no
    MACADDR="FC:C7:C1:0D:00:00"
    GATEWAY=10.0.0.1
    DNS1=10.96.0.10
    DNS2=8.8.8.8
    ```
    
    - 파일 수정 후 service network restart로 네트워크 설정 적용.

# 3. SDN Controller Container로 기동

- docker를 사용해 SDN Controller를 구동시키고, 이를 통해 노드의 내부-외부 통신을 뚫고 k8s를 설치.
- 사전에 노드의 IP, k8s VIP, GW VIP, SNAT Public IP에 대한 PMAC을 적은 tcn_rcfg.json 파일 작성.
    
    ```bash
    {       "lsnr_port" : 6633,
            "up_link_ports":[{"port":1}],
            "phy_gw":[{"ip_addr" : "172.23.2.1", "mac_addr":"60:9c:9f:93:68:f0"}],
            "allowed_pmac_hosts" : [{"ip_addr" : "10.0.0.1", "mac_addr" : "fc:c0:c1:0e:00:00"},
                                    {"ip_addr" : "10.0.0.2", "mac_addr" : "fc:c7:c1:0d:00:00"},
                                    {"ip_addr" : "10.0.9.1", "mac_addr" : "fc:c7:c1:0d:00:00"},
                                    {"ip_addr" : "10.0.9.2", "mac_addr" : "fc:c7:c1:0e:00:00"},
                                    {"ip_addr" : "10.0.9.3", "mac_addr" : "fc:c0:c1:0d:00:00"},
                                    {"ip_addr" : "10.0.9.4", "mac_addr" : "fc:c0:c1:0e:00:00"},
                                    {"ip_addr" : "10.0.9.5", "mac_addr" : "fc:2d:c1:0d:00:00"},
                                    {"ip_addr" : "172.22.2.2", "mac_addr" : "fc:c0:c1:0e:00:00"},
                                    {"ip_addr" : "172.22.2.3", "mac_addr" : "fc:2d:c1:0d:00:00"}],
            "cm_gid" : 4000
    }
    ```
    
    - "lsnr_port" : SDN Switch에서 Controller로 접속할 때 사용하는 포트. SDN Switch 설정에서 잡아주었던 포트 그대로 작성.
    - “ip_link_ports” : SDN Switch에서 업링크로 설정된 포트
    - "phy_gw" : 실제 물리 gateway의 주소와 MAC 주소
    - "allowed_pmac_hosts" : Controller가 기동하면서 등록할 ProxyARP Entry
        - Container로 기동되는 SDN Controller에서 등록 해주어야 할 ProxyARP Entry
            - 클러스터 내 노드들의 IP와 PMAC
            - SNAT 해줄 Public IP와 GW 노드의 PMAC 작성
            - GW VIP와 GW 노드의 PMAC 작성
            - k8s Master VIP와 k8s Master를 Init할 노드의 PMAC
    - 10.0.9.1 ~ 10.0.9.5은 노드, 10.0.0.1은 GW VIP, 10.0.0.2는 k8s Master VIP, 172.22.2.2, 3는 SNAT Public IP로 작성한 예제
- podman을 통해 SDN Controller 이미지를 설치할 노드에 불러오고, 다음의 커맨드 실행
    - podman run -d --privileged --network host -v /root/tcn_rcfg.json:/opt/tcnc/config/tcn_rcfg.json -p 6633:6633 tmaxcloudck/hypersdn-controller-v2:v0.0.1
- 노드로 통신이 되는지 확인 후, GW 노드에 public 통신을 위한 SNAT룰을 수동으로 내림
- (TBD) 수동으로 내린 SNAT룰은 GW Agent가 기동할 때 지울 수 있도록 custom chain(NAT-POSTROUTING)에 룰을 내린다.
    
    ```bash
    iptables -t nat -N NAT-PREROUTING
    iptables -t nat -N NAT-POSTROUTING
    iptables -t nat -A PREROUTING -j NAT-PREROUTING 
    iptables -t nat -A POSTROUTING -j NAT-POSTROUTING
    iptables -t nat -A NAT-POSTROUTING -s 10.0.0.0/8 ! -d 10.0.0.0/8 -o {GW 노드의 data plane interface 이름} -j MASQUERADE
    ```
    
- 외부 통신이 되는지 확인
- ip addr add {k8s Master VIP}/32 dev {SDN Data Plane interface}로 쿠버네티스 클러스터 설치

# 4. SDN 운영기 설치

## 4-1. SDN Controller 설치

- hypersdn-system namespace 생성
    
    ```bash
    apiVersion: v1
    kind: Namespace
    metadata:
      name: hypersdn-system
    ```
    
- config-1, config-2 생성
    
    ```bash
    apiVersion: v1
    data:
      tcn_rcfg.json: |
            {       "lsnr_port" : 6633,
                    "priority" : 100,
                    "up_link_ports":[{"port":1}],
                    "cm_port" : 8080,
                    "cm_ip_addr" : "172.31.32.2",
                    "phy_gw":[{"ip_addr" : "172.23.2.1", "mac_addr":"60:9c:9f:93:68:f0"}],
                    "allowed_pmac_hosts" : []
            }
    kind: ConfigMap
    metadata:
      creationTimestamp: null
      name: hypersdn-rcfg-1
      namespace: hypersdn-system
    ```
    
    ```bash
    apiVersion: v1
    data:
      tcn_rcfg.json: |
            {       "lsnr_port" : 6633,
                    "priority" : 99,
                    "up_link_ports":[{"port":1}],
                    "cm_port" : 8080,
                    "cm_ip_addr" : "172.31.32.1",
                    "phy_gw":[{"ip_addr" : "172.23.2.1", "mac_addr":"60:9c:9f:93:68:f0"}],
                    "allowed_pmac_hosts" : []
                    }
    kind: ConfigMap
    metadata:
      creationTimestamp: null
      name: hypersdn-rcfg-2
      namespace: hypersdn-system
    ```
    
    - "priority" : leader가 될 쪽에 100, follwer가 될 쪽에 99 할당
    - "cm_port" : 컨트롤러간 연결에 사용 될 포트. 8080으로 설정
    - "cm_ip_addr" : 컨트롤러간 연결에 사용 될 IP. 상대방 컨트롤러가 뜰 노드의 IP로 설정.
- ProxyARP CustomResourceDefinition 생성
    
    ```bash
    apiVersion: apiextensions.k8s.io/v1
    kind: CustomResourceDefinition
    metadata:
      name: proxyarps.hypersdn.tmaxanc.com
    spec:
      group: hypersdn.tmaxanc.com
      names:
        kind: ProxyARP
        listKind: ProxyARPList
        plural: proxyarps
        singular: proxyarp
        shortNames:
        - pa
      scope: Namespaced
      versions:
      - name: v1alpha1
        additionalPrinterColumns:
        - name: PMAC
          type: string
          jsonPath: .spec.host.pmac
        - name: TargetNodeIP
          type: string
          jsonPath: .spec.vip.targetnodeip
        - name: status
          type: string
          jsonPath: .status.register
        subresources:
          status: {}
        schema:
          openAPIV3Schema:
            properties:
              apiVersion:
                type: string
              kind:
                type: string
              metadata:
                type: object
              spec:
                type: object
                properties:
                  host:
                    type: object
                    properties:
                      pmac:
                        type: string
                        pattern: ^(fc|FC):([0-9a-fA-F]{2}:){4}([0-9a-fA-F]{2})$
                    required:
                      - pmac
                  vip:
                    type: object
                    properties:
                      targetnodeip:
                        type: string
                        pattern: ^(?:(?:25[0-5]|2[0-4][0-9]|1[0-9][0-9]|[1-9][0-9]|[0-9])\.){3}(?:25[0-5]|2[0-4][0-9]|1[0-9][0-9]|[1-9][0-9]|[0-9])$
                    required:
                      - targetnodeip
                oneOf:
                  - required: ["host"]
                  - required: ["vip"]
                type: object
              status:
                type: object
                properties:
                  register:
                    type: string
                    description: show ProxyARP entry is registered on controller or not
            type: object
        served: true
        storage: true
    ```
    
- ProxyARP CustomResource 생성
    
    ```bash
    apiVersion: hypersdn.tmaxanc.com/v1alpha1
    kind: ProxyARP
    metadata:
      name: 10.0.9.1
      namespace: hypersdn-system
    spec:
      host:
        pmac: fc:c7:c1:0d:00:00
    ---
    apiVersion: hypersdn.tmaxanc.com/v1alpha1
    kind: ProxyARP
    metadata:
      name: 10.0.9.2
      namespace: hypersdn-system
    spec:
      host:
        pmac: fc:c7:c1:0e:00:00
    ---
    apiVersion: hypersdn.tmaxanc.com/v1alpha1
    kind: ProxyARP
    metadata:
      name: 10.0.9.3
      namespace: hypersdn-system
    spec:
      host:
        pmac: fc:c0:c1:0d:00:00
    ---
    apiVersion: hypersdn.tmaxanc.com/v1alpha1
    kind: ProxyARP
    metadata:
      name: 10.0.9.4
      namespace: hypersdn-system
    spec:
      host:
        pmac: fc:c0:c1:0e:00:00
    ---
    apiVersion: hypersdn.tmaxanc.com/v1alpha1
    kind: ProxyARP
    metadata:
      name: 10.0.9.5
      namespace: hypersdn-system
    spec:
      host:
        pmac: fc:2d:c1:0d:00:00
    
    ```
    
    - SNAT public IP, k8s Master VIP를 제외한 노드의 ProxyARP Entry만 작성
- Controller-1, Controller-2 작성
    
    ```bash
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: hypersdn-controller-1
      namespace: hypersdn-system
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: hypersdn-controller
      template:
        metadata:
          labels:
            app: hypersdn-controller
        spec:
          hostNetwork: true
          serviceAccountName: hypersdn-controller-service-account
          containers:
          - name: hypersdn-controller
            image: tmaxcloudck/hypersdn-controller-v2:v0.0.1
            ports:
            - containerPort: 8080
            - containerPort: 6633
            volumeMounts:
            - mountPath: /opt/tcnc/config
              name: hypersdn-volume-1 
              readOnly: true
            securityContext:
              privileged: true
          volumes:
          - configMap:
              defaultMode: 420
              name: hypersdn-rcfg-1
            name: hypersdn-volume-1
          affinity: 
            nodeAffinity: 
              requiredDuringSchedulingIgnoredDuringExecution:
                nodeSelectorTerms: 
                - matchExpressions: 
                  - key: kubernetes.io/hostname 
                    operator: In 
                    values: 
                    - cksdn1-1
    ```
    
    ```bash
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: hypersdn-controller-2
      namespace: hypersdn-system
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: hypersdn-controller
      template:
        metadata:
          labels:
            app: hypersdn-controller
        spec:
          hostNetwork: true
          serviceAccountName: hypersdn-controller-service-account
          containers:
          - name: hypersdn-controller
            image: tmaxcloudck/hypersdn-controller-v2:v0.0.1
            ports:
            - containerPort: 8080
            - containerPort: 6633
            volumeMounts:
            - mountPath: /opt/tcnc/config
              name: hypersdn-volume-2
              readOnly: true
            securityContext:
              privileged: true
          volumes:
          - configMap:
              defaultMode: 420
              name: hypersdn-rcfg-2
            name: hypersdn-volume-2
          affinity:
            nodeAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                nodeSelectorTerms:
                - matchExpressions:
                  - key: kubernetes.io/hostname
                    operator: In
                    values:
                    - cksdn1-2
    ```
    
    - deployment의 name은 hypersdn-controller-1, hypersdn-controller-2로 다르게 작성
    - volumeMounts와 configmap도 각각 맞게 작성, node affinitiy도 뜰 노드에 맞게 작성
    - **컨테이너로 띄운 SDN 컨트롤러와 다른 노드에 컨트롤러를 기동**
    - kubectl get pods, logs로 정상 동작 확인
        - ProxyARP CR이 정상적으로 컨트롤러에 등록되었는지 확인
        
        ```bash
        root@cksdn1-1 deploy]# kubectl get pa -A
        NAMESPACE         NAME         PMAC                TARGETNODEIP   STATUS
        hypersdn-system   10.0.0.1                         10.0.9.4       SUCCESS
        hypersdn-system   10.0.0.2                         10.0.9.1       SUCCESS
        hypersdn-system   10.0.9.1     fc:c7:c1:0d:00:00                  SUCCESS
        hypersdn-system   10.0.9.2     fc:c7:c1:0e:00:00                  SUCCESS
        hypersdn-system   10.0.9.3     fc:c0:c1:0d:00:00                  SUCCESS
        hypersdn-system   10.0.9.4     fc:c0:c1:0e:00:00                  SUCCESS
        hypersdn-system   10.0.9.5     fc:2d:c1:0d:00:00                  SUCCESS
        hypersdn-system   172.23.2.2   fc:c0:c1:0e:00:00                  SUCCESS
        hypersdn-system   172.23.2.3   fc:2d:c1:0d:00:00                  SUCCESS
        ```
        

## 4-2.(TBD) 다른 SDN module 설치

## 4-3. GW VIP, SNAT public IP를 관리하는 모듈 설치 이후

- 1) 컨테이너로 띄운 SDN 컨테이너를 종료.
- 2) pod으로 다시 컨트롤러를 실행.
- 노드 추가 시 해당 노드의 IP와 MAC을 ProxyARP CR로 생성 후 노드 추가 진행

## 4-4. keepalived 설정

- keepalived.conf
    
    ```bash
    global_defs {
        script_user root root
        enable_script_security off
    }
    
    vrrp_instance VI_1 {
    state BACKUP
    interface enp3s0
    virtual_router_id 22
    priority 100
    nopreempt
    advert_int 1
    unicast_peer {
            10.0.9.2
            10.0.9.3
    }
    authentication {
            auth_type PASS
            auth_pass 1111
            }
    virtual_ipaddress {
            10.0.0.2
            }
    notify /etc/keepalived/notify-script.sh
    
    }
    ```
    
    - k8s Master 3대로 구축 했을 때, nopreempt 옵션을 주고 state는 전부 BACKUP, priority는 1씩 차이나게 세팅
    - virtual_router_id와 advert_int값은 모두 같아야 함
    - unicaset_peer에 다른 k8s Master들의 SDN data plane에서 사용하는 ip를 적고, virtual_ipaddress에는 k8s master VIP를 적음.
    - k8s master VIP failover를 위해 notify-script 실행 설정
- notify-script.sh
    
    ```bash
    #add to keepalived.conf > "notify /etc/keepalived/notify-script.sh"
    #chmod +x notify-script.sh
    #disable selinux
    
    TYPE=$1
    NAME=$2
    STATE=$3
    
    case $STATE in
            "MASTER")
                    source /etc/environment
                    index=1
                    count=10
                    while [ $index -le $count ]
                    do
                            RESULT=$(curl -X PUT "$SDN_CON_1:8080/vip/10.0.0.2/targetNodeIP/10.0.9.1" -w "%{http_code}\n" -o /dev/null -s --connect-timeout 1)
                            if [ $RESULT -eq 200 ]; then
                                    echo "Service call to $SDN_CON_1 was successful at $(date)" | tee -a /etc/keepalived/curlLog
                                    break
                            fi
                            echo "Service call to $SDN_CON_1 was failed at $(date)" | tee -a /etc/keepalived/curlLog
    
                            RESULT=$(curl -X PUT "$SDN_CON_2:8080/vip/10.0.0.2/targetNodeIP/10.0.9.1" -w "%{http_code}\n" -o /dev/null -s --connect-timeout 1)
                            if [ $RESULT -eq 200 ]; then
                                    echo "Service call to $SDN_CON_2 was successful at $(date)" | tee -a /etc/keepalived/curlLog
                                    break
                            fi
                            echo "Service call to $SDN_CON_2 was failed at $(date)" | tee -a /etc/keepalived/curlLog
                            sleep 3
                            ((index++))
                    done
                    exit 0
                    ;;
            "BACKUP")
                    source /etc/environment
                    SDN_CON_ON=$(kubectl get po -n hypersdn-system -o wide | grep hypersdn-controller | awk 'NR == 1 {print $6}')
    
                    index=1
                    count=10
    
                    while [ $index -le $count ]
                    do
                            if [ -z "${SDN_CON_ON}" ]; then
                                    echo "SDN Controller was not active at $(date)" | tee -a /etc/keepalived/curlLog
                            else
                                    break
                            fi
                            sleep 3
                            ((index++))
                            SDN_CON_ON=$(kubectl get po -n hypersdn-system -o wide | grep hypersdn-controller | awk 'NR == 1 {print $6}')
                    done
    
                    if [ -z "${SDN_CON_1}" ]; then
                            SDN_CON_TEMP=$(kubectl get po -n hypersdn-system -o wide | grep hypersdn-controller | awk 'NR == 1 {print $6}')
                            if [ ! -z "${SDN_CON_TEMP}" ]; then
                                    echo SDN_CON_1=$SDN_CON_TEMP | tee -a /etc/environment
                            fi
    
                    fi
                    if  [ -z "${SDN_CON_2}" ]; then
                            SDN_CON_TEMP=$(kubectl get po -n hypersdn-system -o wide | grep hypersdn-controller | awk 'NR == 2 {print $6}')
                            if [ ! -z "${SDN_CON_TEMP}" ]; then
                                    echo SDN_CON_2=$SDN_CON_TEMP | tee -a /etc/environment
                            fi
                    fi
                    exit 0
                    ;;
            "FAULT")
                    echo "fault" | tee -a /etc/keepalived/faultLog
                    exit 0
                    ;;
            *)        echo "unknown state"
                    exit 1
                    ;;
    esac
    ```
    
    - RESULT=$(curl -X PUT "$SDN_CON_1:8080/vip/**{k8s Master VIP}**/targetNodeIP/**{keepalived가 구동되는 노드의 SDN DataPlane IP}**" -w "%{http_code}\n" -o /dev/null -s --connect-timeout 1) 형식에 맞게 두 번의 curl 호출 url을 작성
    - keepalived가 BACKUP state일 때는 /etc/environment에 SDN Controller가 떠있는 두 노드의 IP를 환경변수로 저장
    - MASTER state로 진입 했을 때 해당 환경변수를 사용해 k8s Master VIP의 MAC 주소를 바꾸는 service call을 양 컨트롤러에 번갈아가면서 시도함
    - 시도 횟수는 count 값으로 조정 가능, timeout 값은 --connect-timeout 인자로 조정 가능.
    - Master, BACKUP State로 진입 했을 때 시도하는 것들의 성공, 실패 결과가 모두 /etc/keepalived/curlLog에 저장됨.

# 5. 사용법

## 5-1. ProxyARP CR

- 일반 노드 등록 시
    
    ```bash
    apiVersion: hypersdn.tmaxanc.com/v1alpha1
    kind: ProxyARP
    metadata:
      name: 10.0.9.1
      namespace: hypersdn-system
    spec:
      host:
        pmac: fc:c7:c1:0d:00:00
    ```
    
    - name을 노드의 IP로, spec.host.pmac에 해당 노드의 PMAC 주소를 적음
- VIP 등록 시
    
    ```bash
    apiVersion: hypersdn.tmaxanc.com/v1alpha1
    kind: ProxyARP
    metadata:
      name: 10.0.0.1
      namespace: hypersdn-system
    spec:
      vip:
        targetnodeip: 10.0.9.4
    ```
    
    - name은 VIP의 IP로, spec.vip.targetnodeip에 해당 VIP가 포워딩 되어야 할 노드의 실제 IP주소를 적음
    - SDN Controller에서 해당 CR을 처리하여 {VIP - targetnodeip주소를 가진 노드의 PMAC}으로 ProxyARP Entry를 등록
- kubectl get pa -n hypersdn-system
    
    ```bash
    [root@cksdn1-1 deploy]# kubectl get pa -n hypersdn-system
    NAME         PMAC                TARGETNODEIP   STATUS
    10.0.0.1                         10.0.9.4       SUCCESS
    10.0.0.2                         10.0.9.1       SUCCESS
    10.0.0.9                         10.0.9.2       SUCCESS
    10.0.9.1     fc:c7:c1:0d:00:00                  SUCCESS
    10.0.9.2     fc:c7:c1:0e:00:00                  SUCCESS
    10.0.9.3     fc:c0:c1:0d:00:00                  SUCCESS
    10.0.9.4     fc:c0:c1:0e:00:00                  SUCCESS
    10.0.9.5     fc:2d:c1:0d:00:00                  SUCCESS
    172.23.2.2   fc:c0:c1:0e:00:00                  SUCCESS
    172.23.2.3   fc:2d:c1:0d:00:00                  SUCCESS
    ```
    
    - host의 경우 NAME과 PMAC 필드만 나오고, VIP의 경우 NAME과 TARGETNODEIP만 출력됨. 실제로 컨트롤러에 등록된 경우 STATUS에 SUCCESS로 표기됨.

## 5-2. Monitor

- SDN 스위치에 등록되어 있는 Resource를 조회 가능
- curl -X GET "{SDN Controller가 떠 있는 노드 IP:8080}/{Resource 이름}"으로 조회 가능

1. 헬스 체크 
- curl -X GET "localhost:8080/health"
- 정상이면 "alive":true 출력

2. 스위치 정보 출력
- curl -X GET "localhost:8080/switches"

```bash
{"Flag":"SW_GW","Ip":"172.31.32.101","Dpid":2037944993637449921,"DpidHex":"1c483c2c998bc0c1","RemotePortNo":44285,"GID":{"GenerationID":"5493"},"GID_SET":true,"DesiredMastership":true,"CurrentMastership":true}
{"Flag":"SW_NORMAL","Ip":"172.31.32.200","Dpid":2037944993637450625,"DpidHex":"1c483c2c998bc381","RemotePortNo":54948,"GID":{"GenerationID":"5493"},"GID_SET":true,"DesiredMastership":true,"CurrentMastership":true}
{"Flag":"SW_GW","Ip":"172.31.32.100","Dpid":2037944993637451713,"DpidHex":"1c483c2c998bc7c1","RemotePortNo":52243,"GID":{"GenerationID":"5493"},"GID_SET":true,"DesiredMastership":true,"CurrentMastership":true}
{"Flag":"SW_NORMAL","Ip":"172.31.32.102","Dpid":2037910622927662529,"DpidHex":"1c481cea0b992dc1","RemotePortNo":37137,"GID":{"GenerationID":"5493"},"GID_SET":true,"DesiredMastership":true,"CurrentMastership":true}
{"Flag":"SW_NORMAL","Ip":"172.31.32.201","Dpid":2038081599334387585,"DpidHex":"1c48b86a97960f81","RemotePortNo":60388,"GID":{"GenerationID":"5493"},"GID_SET":true,"DesiredMastership":true,"CurrentMastership":true}
```

- Flag : uplink가 있는 스위치의 경우 SW_GW로 표기, 이외의 경우 SW_NORMAL로 표기
- DesiredMastership : 컨트롤러간 switch mastership을 분배 할 때 해당 컨트롤러가 갖기로 정해지면 true
- CurrentMastership : 실제로 해당 컨트롤러가 switch의 mastership을 가지고 있으면 true

3. 포트 리스트 출력
- curl -X GET "localhost:8080/switches/{DpidHex}/ports"

```bash
{"Type":"UNKNOWN","Desc":{"PortNo":13,"HwAddr":"PCyZi8DB","Name":"Z2UtMS8xLzEzAAAAAAAAAA==","Config":0,"State":0,"Curr":10272,"Advertised":10282,"Supported":10282,"Peer":2095,"CurrSpeed":1000000,"MaxSpeed":1000000},"sw_dpid":2037944993637449921}
{"Type":"UNKNOWN","Desc":{"PortNo":14,"HwAddr":"PCyZi8DB","Name":"Z2UtMS8xLzE0AAAAAAAAAA==","Config":0,"State":0,"Curr":10272,"Advertised":10282,"Supported":10282,"Peer":2095,"CurrSpeed":1000000,"MaxSpeed":1000000},"sw_dpid":2037944993637449921}
{"Type":"TO_SW","Desc":{"PortNo":6,"HwAddr":"PCyZi8DB","Name":"Z2UtMS8xLzYAAAAAAAAAAA==","Config":0,"State":0,"Curr":10272,"Advertised":10282,"Supported":10282,"Peer":2090,"CurrSpeed":1000000,"MaxSpeed":1000000},"sw_dpid":2037944993637449921}
{"Type":"TO_GW","Desc":{"PortNo":1,"HwAddr":"PCyZi8DB","Name":"Z2UtMS8xLzEAAAAAAAAAAA==","Config":0,"State":0,"Curr":10272,"Advertised":10282,"Supported":10282,"Peer":2095,"CurrSpeed":1000000,"MaxSpeed":1000000},"sw_dpid":2037944993637449921}
{"Type":"TO_SW","Desc":{"PortNo":5,"HwAddr":"PCyZi8DB","Name":"Z2UtMS8xLzUAAAAAAAAAAA==","Config":0,"State":0,"Curr":10272,"Advertised":10282,"Supported":10282,"Peer":2090,"CurrSpeed":1000000,"MaxSpeed":1000000},"sw_dpid":2037944993637449921}
```

- Type : Host 타입인 경우 UNKNOWN이나 TO_HOST, 스위치간 링크면 TO_SW, 업링크인 포트면 TO_GW
- State : 0이거나 4 = LIVE ,  2 = BLOCKED , 1 = LINK_DOWN

4. 특정 포트 정보 출력
- curl -X GET "localhost:8080/switches/{DpidHex}/ports/{portNo}"

```bash
{"Type":"TO_GW","Desc":{"PortNo":1,"HwAddr":"PCyZi8DB","Name":"Z2UtMS8xLzEAAAAAAAAAAA==","Config":0,"State":0,"Curr":10272,"Advertised":10282,"Supported":10282,"Peer":2095,"CurrSpeed":1000000,"MaxSpeed":1000000},"sw_dpid":2037944993637449921}
```

5. 링크 정보 출력
- curl -X GET "localhost:8080/links"

```bash
{"Src":"1c483c2c998bc0c1","Dst":"1c483c2c998bc381","Sport":5,"Dport":6,"Cost":1,"Remote":false}
{"Src":"1c483c2c998bc0c1","Dst":"1c48b86a97960f81","Sport":6,"Dport":6,"Cost":1,"Remote":false}
{"Src":"1c483c2c998bc381","Dst":"1c483c2c998bc0c1","Sport":6,"Dport":5,"Cost":1,"Remote":false}
{"Src":"1c483c2c998bc381","Dst":"1c483c2c998bc7c1","Sport":5,"Dport":5,"Cost":1,"Remote":false}
{"Src":"1c483c2c998bc7c1","Dst":"1c48b86a97960f81","Sport":6,"Dport":5,"Cost":1,"Remote":false}
{"Src":"1c483c2c998bc7c1","Dst":"1c483c2c998bc381","Sport":5,"Dport":5,"Cost":1,"Remote":false}
{"Src":"1c481cea0b992dc1","Dst":"1c48b86a97960f81","Sport":6,"Dport":7,"Cost":1,"Remote":false}
{"Src":"1c48b86a97960f81","Dst":"1c483c2c998bc7c1","Sport":5,"Dport":6,"Cost":1,"Remote":false}
{"Src":"1c48b86a97960f81","Dst":"1c481cea0b992dc1","Sport":7,"Dport":6,"Cost":1,"Remote":false}
{"Src":"1c48b86a97960f81","Dst":"1c483c2c998bc0c1","Sport":6,"Dport":6,"Cost":1,"Remote":false}
```

6. ARP Host Entry 출력
- curl -X GET "localhost:8080/arpHosts"

```bash
{"HwAddr":"fc:c0:c1:0e:00:00","IP":"10.0.0.1"}
{"HwAddr":"fc:c7:c1:0d:00:00","IP":"10.0.0.2"}
{"HwAddr":"fc:c7:c1:0e:00:00","IP":"10.0.0.9"}
{"HwAddr":"fc:c7:c1:0d:00:00","IP":"10.0.9.1"}
{"HwAddr":"fc:c7:c1:0e:00:00","IP":"10.0.9.2"}
{"HwAddr":"fc:c0:c1:0d:00:00","IP":"10.0.9.3"}
{"HwAddr":"fc:c0:c1:0e:00:00","IP":"10.0.9.4"}
{"HwAddr":"fc:c0:c1:0d:00:00","IP":"10.0.9.40"}
{"HwAddr":"fc:2d:c1:0d:00:00","IP":"10.0.9.5"}
{"HwAddr":"fc:c0:c1:0e:00:00","IP":"172.23.2.2"}
{"HwAddr":"fc:2d:c1:0d:00:00","IP":"172.23.2.3"}
```

7. Physical Gateway 출력
- curl -X GET "localhost:8080/phyGWs"

```bash
{"HwAddr":"60:bc:ef:93:68:f2","IP":"172.24.1.1"}
{"HwAddr":"60:9c:9f:93:68:f0","IP":"172.23.2.1"}
```
