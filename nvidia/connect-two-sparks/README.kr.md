# 두 개의 Spark 연결

> 두 개의 Spark 장치를 연결하고 추론 및 미세 조정을 위해 설정

## 목차

- [개요](#개요)
- [두 개의 Spark에서 실행](#두-개의-spark에서-실행)
- [문제 해결](#문제-해결)

---

## 개요

## 기본 아이디어

200GbE 직접 QSFP 연결을 사용하여 고속 노드 간 통신을 위해 두 개의 DGX Spark 시스템을 구성합니다. 이 설정은 네트워크 연결을 설정하고 SSH 인증을 구성하여 여러 DGX Spark 노드에서 분산 워크로드를 가능하게 합니다.

## 달성할 내용

QSFP 케이블로 두 개의 DGX Spark 장치를 물리적으로 연결하고, 클러스터 통신을 위한 네트워크 인터페이스를 구성하고, 노드 간 비밀번호 없는 SSH를 설정하여 기능적인 분산 컴퓨팅 환경을 만듭니다.

## 시작하기 전에 알아야 할 사항

- 분산 컴퓨팅 개념에 대한 기본 이해
- 네트워크 인터페이스 구성 및 netplan 작업
- SSH 키 관리 경험

## 전제 조건

- 두 개의 DGX Spark 시스템
- 두 장치 간의 직접 200GbE 연결을 위한 QSFP 케이블 1개
- 두 시스템 모두에 SSH 액세스 가능
- 두 시스템 모두에 대한 루트 또는 sudo 액세스: `sudo whoami`
- 두 시스템 모두에서 동일한 사용자 이름

## 부속 파일

이 플레이북에 필요한 모든 파일은 [여기 GitHub에서](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main/nvidia/connect-two-sparks/) 찾을 수 있습니다

- [**discover-sparks.sh**](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main/nvidia/connect-two-sparks/assets/discover-sparks) 자동 노드 검색 및 SSH 키 배포를 위한 스크립트

## 시간 및 위험

- **소요 시간:** 검증을 포함하여 1시간

- **위험 수준:** 중간 - 네트워크 재구성 포함

- **롤백:** netplan 구성을 제거하거나 IP 할당을 제거하여 네트워크 변경을 되돌릴 수 있습니다

- **최종 업데이트:** 11/24/2025
  * 사소한 수정

## 두 개의 Spark에서 실행

## 1단계. 두 시스템에서 동일한 사용자 이름 확인

두 시스템 모두에서 사용자 이름을 확인하고 동일한지 확인하십시오:

```bash
## 현재 사용자 이름 확인
whoami
```

사용자 이름이 일치하지 않으면 두 시스템 모두에 새 사용자(예: nvidia)를 만들고 새 사용자로 로그인하십시오:

```bash
## nvidia 사용자 생성 및 sudo 그룹에 추가
sudo useradd -m nvidia
sudo usermod -aG sudo nvidia

## nvidia 사용자의 비밀번호 설정
sudo passwd nvidia

## nvidia 사용자로 전환
su - nvidia
```

## 2단계. 물리적 하드웨어 연결

각 장치의 QSFP 인터페이스를 사용하여 두 DGX Spark 시스템 간에 QSFP 케이블을 연결합니다. 이것은 고속 노드 간 통신에 필요한 200GbE 직접 연결을 설정합니다. 두 노드 간의 연결 시 다음과 같은 출력을 볼 수 있습니다: 이 예제에서 'Up'으로 표시되는 인터페이스는 **enp1s0f1np1** / **enP2p1s0f1np1**입니다(각 물리적 포트에는 두 개의 이름이 있음).

예제 출력:
```bash
## 두 노드 모두에서 QSFP 인터페이스 가용성 확인
nvidia@dxg-spark-1:~$ ibdev2netdev
roceP2p1s0f0 port 1 ==> enP2p1s0f0np0 (Down)
roceP2p1s0f1 port 1 ==> enP2p1s0f1np1 (Up)
rocep1s0f0 port 1 ==> enp1s0f0np0 (Down)
rocep1s0f1 port 1 ==> enp1s0f1np1 (Up)
```

> [!NOTE]
> 인터페이스 중 어느 것도 'Up'으로 표시되지 않으면 QSFP 케이블 연결을 확인하고 시스템을 재부팅한 후 다시 시도하십시오.
> 'Up'으로 표시되는 인터페이스는 두 노드를 연결하는 데 사용하는 포트에 따라 다릅니다. 각 물리적 포트에는 두 개의 이름이 있습니다. 예를 들어 enp1s0f1np1과 enP2p1s0f1np1은 동일한 물리적 포트를 나타냅니다. enP2p1s0f0np0 및 enP2p1s0f1np1은 무시하고 enp1s0f0np0 및 enp1s0f1np1만 사용하십시오.

## 3단계. 네트워크 인터페이스 구성

네트워크 인터페이스를 설정하려면 옵션 중 하나를 선택하십시오. 옵션 1과 2는 상호 배타적입니다.

> [!NOTE]
> QSFP 케이블 1개만으로 전체 대역폭을 달성할 수 있습니다.
> 두 개의 QSFP 케이블이 연결된 경우 전체 대역폭을 얻으려면 네 개의 인터페이스 모두에 IP 주소를 할당해야 합니다.
> 아래 옵션 1은 QSFP 케이블 1개가 연결된 경우에만 사용할 수 있습니다.

**옵션 1: 자동 IP 할당 (QSFP 케이블 1개가 연결된 경우에만 사용 가능)**

링크 로컬 주소 지정을 위해 두 DGX Spark 노드 모두에서 netplan을 사용하여 네트워크 인터페이스를 구성하십시오:

```bash
## netplan 구성 파일 생성
sudo tee /etc/netplan/40-cx7.yaml > /dev/null <<EOF
network:
  version: 2
  ethernets:
    enp1s0f0np0:
      link-local: [ ipv4 ]
    enp1s0f1np1:
      link-local: [ ipv4 ]
EOF

## 적절한 권한 설정
sudo chmod 600 /etc/netplan/40-cx7.yaml

## 구성 적용
sudo netplan apply
```

**옵션 2: netplan 구성 파일을 사용한 수동 IP 할당**

노드 1:
```bash
## netplan 구성 파일 생성
sudo tee /etc/netplan/40-cx7.yaml > /dev/null <<EOF
network:
  version: 2
  ethernets:
    enp1s0f0np0:
      addresses:
        - 192.168.100.10/24
      dhcp4: no
    enp1s0f1np1:
      addresses:
        - 192.168.200.12/24
      dhcp4: no
    enP2p1s0f0np0:
      addresses:
        - 192.168.100.14/24
      dhcp4: no
    enP2p1s0f1np1:
      addresses:
        - 192.168.200.16/24
      dhcp4: no
EOF

## 적절한 권한 설정
sudo chmod 600 /etc/netplan/40-cx7.yaml

## 구성 적용
sudo netplan apply
```

노드 2:
```bash
## netplan 구성 파일 생성
sudo tee /etc/netplan/40-cx7.yaml > /dev/null <<EOF
network:
  version: 2
  ethernets:
    enp1s0f0np0:
      addresses:
        - 192.168.100.11/24
      dhcp4: no
    enp1s0f1np1:
      addresses:
        - 192.168.200.13/24
      dhcp4: no
    enP2p1s0f0np0:
      addresses:
        - 192.168.100.15/24
      dhcp4: no
    enP2p1s0f1np1:
      addresses:
        - 192.168.200.17/24
      dhcp4: no
EOF

## 적절한 권한 설정
sudo chmod 600 /etc/netplan/40-cx7.yaml

## 구성 적용
sudo netplan apply
```


**옵션 3: 명령줄을 사용한 수동 IP 할당**

> [!NOTE]
> 이 옵션을 사용하면 시스템을 재부팅할 경우 인터페이스에 할당된 IP가 변경됩니다.

먼저 사용 가능하고 활성화된 네트워크 포트를 식별하십시오:

```bash
## 네트워크 포트 상태 확인
ibdev2netdev
```

예제 출력:
```
roceP2p1s0f0 port 1 ==> enP2p1s0f0np0 (Down)
roceP2p1s0f1 port 1 ==> enP2p1s0f1np1 (Up)
rocep1s0f0 port 1 ==> enp1s0f0np0 (Down)
rocep1s0f1 port 1 ==> enp1s0f1np1 (Up)
```

출력에서 "(Up)"으로 표시되는 인터페이스를 사용하십시오. 이 예제에서는 **enp1s0f1np1**을 사용합니다. `enP2p<...>`로 시작하는 인터페이스는 무시하고 `enp1<...>`로 시작하는 인터페이스만 사용할 수 있습니다.

노드 1:
```bash
## 고정 IP 할당 및 인터페이스 활성화.
sudo ip addr add 192.168.100.10/24 dev enp1s0f1np1
sudo ip link set enp1s0f1np1 up
```

노드 2에 대해 동일한 프로세스를 반복하되 IP **192.168.100.11/24**를 사용하십시오. `ibdev2netdev` 명령을 사용하여 올바른 인터페이스 이름을 사용해야 합니다.
```bash
## 고정 IP 할당 및 인터페이스 활성화.
sudo ip addr add 192.168.100.11/24 dev enp1s0f1np1
sudo ip link set enp1s0f1np1 up
```

각 노드에서 다음 명령을 실행하여 두 노드 모두에서 IP 할당을 확인할 수 있습니다:
```bash
## 출력에서 "(Up)"으로 표시되는 인터페이스로 enp1s0f1np1을 교체하십시오, enp1s0f0np0 또는 enp1s0f1np1
ip addr show enp1s0f1np1
```

## 4단계. 비밀번호 없는 SSH 인증 설정

#### 옵션 1: SSH 자동 구성

노드 중 하나에서 DGX Spark [**discover-sparks.sh**](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main/nvidia/connect-two-sparks/assets/discover-sparks) 스크립트를 실행하여 SSH를 자동으로 검색하고 구성하십시오:

```bash
bash ./discover-sparks
```

다른 IP 및 노드 이름으로 아래와 유사한 예상 출력. 스크립트를 처음 실행하면 각 노드에 대한 비밀번호를 입력하라는 메시지가 표시됩니다.
```
Found: 169.254.35.62 (dgx-spark-1.local)
Found: 169.254.35.63 (dgx-spark-2.local)

Setting up bidirectional SSH access (local <-> remote nodes)...
You may be prompted for your password for each node.

SSH setup complete! Both local and remote nodes can now SSH to each other without passwords.
```

> [!NOTE]
> 오류가 발생하면 아래 옵션 2를 따라 수동으로 SSH를 구성하고 문제를 디버그하십시오.

#### 옵션 2: 수동으로 검색 및 SSH 구성

활성화된 CX-7 인터페이스의 IP 주소를 찾아야 합니다. 두 노드 모두에서 다음 명령을 실행하여 IP 주소를 찾고 다음 단계를 위해 기록해 두십시오.
```bash
  ip addr show enp1s0f0np0
  ip addr show enp1s0f1np1
```

예제 출력:
```
## 이 예제에서는 인터페이스 enp1s0f1np1을 사용하고 있습니다.
nvidia@dgx-spark-1:~$ ip addr show enp1s0f1np1
    4: enp1s0f1np1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
        link/ether 3c:6d:66:cc:b3:b7 brd ff:ff:ff:ff:ff:ff
        inet **169.254.35.62**/16 brd 169.254.255.255 scope link noprefixroute enp1s0f1np1
          valid_lft forever preferred_lft forever
        inet6 fe80::3e6d:66ff:fecc:b3b7/64 scope link
          valid_lft forever preferred_lft forever
```

이 예제에서 노드 1의 IP 주소는 **169.254.35.62**입니다. 노드 2에 대해 프로세스를 반복하십시오.

두 노드 모두에서 다음 명령을 실행하여 비밀번호 없는 SSH를 활성화하십시오:
```bash
## SSH 공개 키를 두 노드 모두에 복사합니다. 이전 단계에서 찾은 IP 주소로 교체하십시오.
ssh-copy-id -i ~/.ssh/id_rsa.pub <username>@<IP for Node 1>
ssh-copy-id -i ~/.ssh/id_rsa.pub <username>@<IP for Node 2>
```

## 5단계. 다중 노드 통신 확인

기본 다중 노드 기능을 테스트하십시오:

```bash
## 노드 간 호스트 이름 해석 테스트
ssh <IP for Node 1> hostname
ssh <IP for Node 2> hostname
```

## 6단계. 정리 및 롤백

> [!WARNING]
> 이 단계는 네트워크 구성을 재설정합니다.

```bash
## 네트워크 구성 롤백 (옵션 1 사용 시)
sudo rm /etc/netplan/40-cx7.yaml
sudo netplan apply

## 네트워크 구성 롤백 (옵션 2 사용 시)
sudo ip addr del 192.168.100.10/24 dev enp1s0f0np0  # 3단계에서 사용한 인터페이스 이름으로 조정합니다.
sudo ip addr del 192.168.100.11/24 dev enp1s0f0np0  # 3단계에서 사용한 인터페이스 이름으로 조정합니다.
```

## 문제 해결

| 증상 | 원인 | 해결 방법 |
|---------|-------|-----|
| "Network unreachable" 오류 | 네트워크 인터페이스가 구성되지 않음 | netplan 구성을 확인하고 `sudo netplan apply` |
| SSH 인증 실패 | SSH 키가 제대로 배포되지 않음 | `./discover-sparks`를 다시 실행하고 비밀번호 입력 |
| 클러스터에서 노드 2가 보이지 않음 | 네트워크 연결 문제 | QSFP 케이블 연결 확인, IP 구성 확인 |
