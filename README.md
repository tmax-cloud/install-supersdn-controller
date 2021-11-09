# install-supersdn-controller
SuperSDN/v2 Controller
# 설치 전 사전 설정

## SDN 장비 설정

1. SDN Switch 설정
    - minicom console 연결 [[참조]](https://pinocc.tistory.com/156) (window에서는 putty 권장)
        - speed: 115200 설정하고 serial 접속
        - id/password = admin/password
    - SDN 모드 및 IP 세팅
        - #sudo picos_boot
        - SDN 모드 선택, static IP 선택, switch의 management IP (eth0)를 172.31.1.100/24 설정
        - GW설정, UI설정 (O), SNMP설정 (X)
    - 세팅 적용 및 재시작
        - #sudo reboot
2. Web UI 설정
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
            - vlan을 사용하지 않기 때문에 uplink port에 대해서도 trunk로 설정
            - 그 외의 경우: trunk로 설정, tags :1
