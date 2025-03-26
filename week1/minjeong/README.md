### 홈서버를 구축해보자!

- 목적
    - AWS를 사용하면서 직접 온프레미스 환경을 경험하고 싶어졌다.
    - 온프레미스 환경 경험 + 간단한 웹서버 구축을 목표로 진행한다.
- 환경
    - 그램 2019 노트북 사용
    - Proxmox 설치 → 우분투 VM 생성

<br>

### Proxmox

- Hypervisor
    - Hypervisor는 VM의 생성 및 관리를 맡는 가상화 소프트웨어
    - 가상화: 물리적인 리소스를 가상 리소스로 분할해 사용
    - 서버 가상화 방식: 한 대의 물리적 하드웨어 서버 → 여러 대의 가상 서버(VM)로 분할
- Hypervisor 유형
    - 1형: 호스트 시스템 위에 OS 없이 바로 실행 → Proxmox
    - 2형: 호스트 OS 위에서 실행 → VMware, VirtualBox
- 구축하기
    1. Proxmox iso 파일 다운로드
    2. USB 드라이브 설정
        - 부팅 가능한 USB 드라이브 만들기
            - USB에 iso 파일을 넣고 rufus를 다운받아 이미지를 구움
        - USB에 Ventoy 설치 후 iso 파일을 넣어 사용하기
            - 여러 운영체제를 하나의 USB 내에서 선택 가능
    3. 이후 BIOS에 진입해 부팅 우선순위를 USB 부팅으로 변경 후 Proxmox 설치

- 윈도우 OS 및 데이터가 손실?
    - 문제 상황: 윈도우 OS를 유지한 상태에서 Proxmox OS를 설치하려 했는데, Proxmox 설치 후 윈도우가 실행되지 않는다.
    - 과정
        1. 윈도우에서 파티션을 156GB(윈도우용) + 100GB(Proxmox용)으로 나눔
        2. 부팅 우선순위를 USB 부팅으로 변경, ventoy가 적용된 USB에서 Proxmox iso 이미지를 통해 Proxmox 설치
        3. 이후 부팅 우선순위를 윈도우 부트 매니저로 바꿨는데도 불구하고 윈도우가 실행되지 않고 proxmox만 실행됨
    - 원인: proxmox 설치 과정에서 “기존의 모든 파티션과 데이터가 손실됨” 문구 확인
        - 설치 과정에서 파티션 경계 삭제 → 윈도우 os 및 데이터 삭제
        - 윈도우는 삭제되어도 BIOS에는 윈도우 부트 매니저가 남아있을 수 있음

<br>

### Tailscale

- VPN(Virtual Private Network, 가상 사설망)
    - 사설망 구축
        - 전용망 구축
        - 인터넷(공중망)을 사용해 사설망과 같은 환경 구축 = 가상 사설망
    - 인터넷(공중망) 상에서 구축되는 논리적인 전용망 → 가상의 터널 사용
- VPN 유형
    - Client-to-Site
        - 사용자가 VPN 클라이언트를 통해 VPN 서버에 연결 요청
        - 사용자 → VPN Client → ISP → VPN Server → Internet
            - 사용자 → ISP → VPN Server까지 터널링 생성
            - ISP는 트래픽 확인 불가
            - 사용자가 VPN 서버로 보낸 모든 트래픽이 암호화
            - VPN 서버는 사용자의 트래픽을 복호화해 인터넷 상으로 트래픽 전달
    - Site-to-Site
        - 두 네트워크 간 연결
    - m2m(machine to machine)
- Tailscale
    - WireGuard 프로토콜 기반 Mesh VPN
    - m2m VPN
    - Zero Trust 보안: 네트워크 내부/외부와 무관하게 항상 신뢰하지 않음
- Tailscale 사용
    - 홈서버 및 원격 접속할 노트북에 모두 Tailscale을 설치한다.
    - 설치 후 Tailscale에서 IP를 확인한다.
    - IP와 8006 포트를 통해 Proxmox 페이지로 진입한다.

<br><hr><br>

### Network

- Bridge
    - VM이 독립적인 IP로 동작
- Host-Only
    - 외부 네트워크와 격리된 내부 폐쇄망 구성
- NAT
    - 호스트가 NAT 역할
    - VM이 내부 IP 주소 사용
    - 모든 VM에 같은 IP 할당

|  | Bridge | Host-Only | NAT |
| --- | --- | --- | --- |
| 호스트 - VM | ✅ | ✅ | 🔺 |
| VM - VM | ✅ | ✅ | ❌ |
| 인터넷 | ✅ | ❌ | ✅ |

<br>

### DHCP

- DHCP(Dynamic Host Configuration Protocol)
    - 네임 서버 주소, IP 주소, 게이트웨이 주소를 자동 할당
    - 영구적인 할당이 아닌 임대 개념의 할당
        - dhclient.leases 파일에 임대받은 IP 대역이 기록되어 있음
    - DHCP 서버 + 클라이언트로 구성
    - 라우터(공유기) 내에 DHCP 서버 내장 - 인터넷을 사용하는 기계에 DHCP 클라이언트 내장
- 구축 과정
    - 우분투 OS를 사용하는 VM 2대를 사용하여 각각 DHCP 서버, 클라이언트로 구축한다.
    - DHCP 서버는 DHCP 패키지(isc-dhcp-server) 설치 및 설정, 고정 IP 할당이 필요하다.
    
<details>
<summary>구축</summary>
<div markdown="1">

- DHCP 서버
    - DHCP 패키지 설치
        
        ```bash
        sudo apt-get update
        sudo apt-get install isc-dhcp-server
        ```
        
    - 고정 IP 할당 필요
        
        ```bash
        vim /etc/netplan/00-installer-config.yaml
        ```
        
        ```yaml
        network:
            version: 2
            ethernets:
            ens18:
                addresses: [192.168.1.20/24]
                routes:
                    - to: default
                    via: 192.168.1.1
                nameservers:
                    addresses: [8.8.8.8, 8.8.4.4]
        ```
        
        ```bash
        sudo netplan apply
        ```
        
    - DHCP 설정
        - dhcpd.conf 설정
            
            ```bash
            sudo vim /etc/dhcp/dhcpd.conf
            ```
            
            ```bash
            # option domain-name 부분 주석 처리
            
            subnet 192.168.0.0 netmask 255.255.255.0 {
                range 192.168.0.2 192.168.0.254;
                option routers 192.168.0.1;
                option subnet-mask 255.255.255.0;
            }
            ```
            
        - NIC 설정
            
            ```bash
            vim /etc/default/isc-dhcp-server
            ```
            
            ```bash
            INTERFACESv4="ens18"
            INTERFACESv6="ens18"
            ```
            
    - DHCP 서버 재시작
        
        ```bash
        sudo systemctl start isc-dhcp-server
        ```
        
    
- DHCP 클라이언트
    - dchp 서버로부터 IP를 할당받도록 설정
        
        ```bash
        vim /etc/netplan/00-installer-config.yaml
        ```
        
        ```yaml
        network:
            version: 2
            ethernets:
            ens18:
                dhcp4: true
        ```
        
        ```bash
        sudo netplan apply
        ```
        
    - dhcp 서버로부터 IP 갱신
        
        ```bash
        sudo dhclient -r ens18
        sudo dhclient ens18
        ```
        

- 클라이언트가 새로운 IP 값이 아닌 기존의 IP 값을 계속 받아옴
    - DHCP 리스 파일을 삭제 → 새로운 IP 값을 받아옴
        
        ```bash
        sudo rm /var/lib/dhcp/dhclient.leases
        ```
</div>
</details>