# 두 개의 Spark를 위한 NCCL

> 두 개의 Spark에 NCCL 설치 및 테스트

## 목차

- [개요](#개요)
- [두 개의 Spark에서 실행](#두-개의-spark에서-실행)
- [문제 해결](#문제-해결)

---

## 개요

## 기본 아이디어

NCCL(NVIDIA Collective Communication Library)은 여러 노드에 걸친 고성능 GPU 간 통신을 가능하게 합니다. 이 안내서는 Blackwell 아키텍처가 있는 DGX Spark 시스템에서 멀티 노드 분산 학습을 위한 NCCL을 설정합니다. 네트워킹을 구성하고, Blackwell 지원으로 NCCL을 소스에서 빌드하고, 노드 간 통신을 검증합니다.

## 달성할 내용

분산 학습 워크로드를 위해 검증된 네트워크 성능 및 적절한 GPU 토폴로지 감지와 함께 DGX Spark 시스템 간 고대역폭 GPU 통신을 가능하게 하는 작동하는 멀티 노드 NCCL 환경을 갖추게 됩니다.

## 시작하기 전에 알아야 할 사항

- Linux 네트워크 구성 및 netplan 작업
- MPI(Message Passing Interface) 개념에 대한 기본 이해
- SSH 키 관리 및 비밀번호 없는 인증 설정

## 전제 조건

- 두 개의 DGX Spark 시스템
- Connect two Sparks 플레이북 완료
- NVIDIA 드라이버 설치: `nvidia-smi`
- CUDA 툴킷 사용 가능: `nvcc --version`
- Root/sudo 권한: `sudo whoami`

## 시간 및 위험

* **소요 시간**: 설정 및 검증에 30분
* **위험 수준**: 중간 - 네트워크 구성 변경 포함
* **롤백**: NCCL 및 NCCL Tests 리포지토리를 DGX Spark에서 삭제할 수 있음
* **마지막 업데이트:** 12/15/2025
  * nccl 최신 버전 v2.28.9-1 사용

## 두 개의 Spark에서 실행

## 1단계. 네트워크 연결 구성

[Connect two Sparks](https://build.nvidia.com/spark/connect-two-sparks/stacked-sparks) 플레이북의 네트워크 설정 지침에 따라 DGX Spark 노드 간 연결을 설정합니다.

여기에는 다음이 포함됩니다:
- 물리적 QSFP 케이블 연결
- 네트워크 인터페이스 구성(자동 또는 수동 IP 할당)
- 비밀번호 없는 SSH 설정
- 네트워크 연결 확인

## 2단계. Blackwell 지원으로 NCCL 빌드

Blackwell 아키텍처 지원으로 소스에서 NCCL을 빌드하기 위해 두 노드에서 이러한 명령을 실행합니다:

```bash
## 종속성 설치 및 NCCL 빌드
sudo apt-get update && sudo apt-get install -y libopenmpi-dev
git clone -b v2.28.9-1 https://github.com/NVIDIA/nccl.git ~/nccl/
cd ~/nccl/
make -j src.build NVCC_GENCODE="-gencode=arch=compute_121,code=sm_121"

## 환경 변수 설정
export CUDA_HOME="/usr/local/cuda"
export MPI_HOME="/usr/lib/aarch64-linux-gnu/openmpi"
export NCCL_HOME="$HOME/nccl/build/"
export LD_LIBRARY_PATH="$NCCL_HOME/lib:$CUDA_HOME/lib64/:$MPI_HOME/lib:$LD_LIBRARY_PATH"
```

## 3단계. NCCL 테스트 스위트 빌드

**두 노드**에서 NCCL 테스트 스위트를 컴파일합니다:

```bash
## NCCL 테스트 클론 및 빌드
git clone https://github.com/NVIDIA/nccl-tests.git ~/nccl-tests/
cd ~/nccl-tests/
make MPI=1
```

## 4단계. 활성 네트워크 인터페이스 및 IP 주소 찾기

먼저 사용 가능하고 활성화된 네트워크 포트를 식별합니다:

```bash
## 네트워크 포트 상태 확인
ibdev2netdev
```

예시 출력:
```
roceP2p1s0f0 port 1 ==> enP2p1s0f0np0 (Down)
roceP2p1s0f1 port 1 ==> enP2p1s0f1np1 (Up)
rocep1s0f0 port 1 ==> enp1s0f0np0 (Down)
rocep1s0f1 port 1 ==> enp1s0f1np1 (Up)
```

출력에서 "(Up)"으로 표시된 인터페이스를 사용합니다. 이 예에서는 **enp1s0f1np1**을 사용합니다. `enP2p<...>` 접두사로 시작하는 인터페이스는 무시하고 대신 `enp1<...>`로 시작하는 인터페이스만 고려할 수 있습니다.

활성화된 CX-7 인터페이스의 IP 주소를 찾아야 합니다. 두 노드에서 다음 명령을 실행하여 IP 주소를 찾고 다음 단계를 위해 기록해 두세요.
```bash
  ip addr show enp1s0f0np0
  ip addr show enp1s0f1np1
```

예시 출력:
```
## 이 예에서는 인터페이스 enp1s0f1np1을 사용합니다.
nvidia@dgx-spark-1:~$ ip addr show enp1s0f1np1
    4: enp1s0f1np1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
        link/ether 3c:6d:66:cc:b3:b7 brd ff:ff:ff:ff:ff:ff
        inet **169.254.35.62**/16 brd 169.254.255.255 scope link noprefixroute enp1s0f1np1
          valid_lft forever preferred_lft forever
        inet6 fe80::3e6d:66ff:fecc:b3b7/64 scope link
          valid_lft forever preferred_lft forever
```

이 예에서 노드 1의 IP 주소는 **169.254.35.62**입니다. 노드 2에 대해 프로세스를 반복합니다.

## 5단계. NCCL 통신 테스트 실행

> [!NOTE]
> 하나의 QSFP 케이블만으로도 전체 대역폭을 달성할 수 있습니다.
> 두 개의 QSFP 케이블이 연결된 경우 전체 대역폭을 얻으려면 네 개의 인터페이스 모두에 IP 주소를 할당해야 합니다.

NCCL 통신 테스트를 실행하기 위해 두 노드에서 다음 명령을 실행합니다. IP 주소와 인터페이스 이름을 이전 단계에서 찾은 것으로 교체하세요.

```bash
## 네트워크 인터페이스 환경 변수 설정(이전 단계의 Up 인터페이스 사용)
export UCX_NET_DEVICES=enp1s0f1np1
export NCCL_SOCKET_IFNAME=enp1s0f1np1
export OMPI_MCA_btl_tcp_if_include=enp1s0f1np1

## 두 노드에 걸쳐 all_gather 성능 테스트 실행(이전 단계에서 찾은 IP 주소로 교체)
mpirun -np 2 -H <노드 1의 IP>:1,<노드 2의 IP>:1 \
  --mca plm_rsh_agent "ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no" \
  -x LD_LIBRARY_PATH=$LD_LIBRARY_PATH \
  $HOME/nccl-tests/build/all_gather_perf
```

더 큰 버퍼 크기로 NCCL 설정을 테스트하여 200Gbps 대역폭을 더 많이 사용할 수도 있습니다.

```bash
## 네트워크 인터페이스 환경 변수 설정(활성 인터페이스 사용)
export UCX_NET_DEVICES=enp1s0f1np1
export NCCL_SOCKET_IFNAME=enp1s0f1np1
export OMPI_MCA_btl_tcp_if_include=enp1s0f1np1

## 두 노드에 걸쳐 all_gather 성능 테스트 실행
mpirun -np 2 -H <노드 1의 IP>:1,<노드 2의 IP>:1 \
  --mca plm_rsh_agent "ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no" \
  -x LD_LIBRARY_PATH=$LD_LIBRARY_PATH \
  $HOME/nccl-tests/build/all_gather_perf -b 16G -e 16G -f 2
```

참고: `mpirun` 명령의 IP 주소 뒤에 `:1`이 붙습니다. 예: `mpirun -np 2 -H 169.254.35.62:1,169.254.35.63:1`

## 7단계. 정리 및 롤백

```bash
## 네트워크 구성 롤백(필요한 경우)
rm -rf ~/nccl/
rm -rf ~/nccl-tests/
```

## 8단계. 다음 단계
NCCL 환경이 DGX Spark에서 멀티 노드 분산 학습 워크로드를 위해 준비되었습니다.
이제 TRT-LLM 또는 vLLM 추론과 같은 더 큰 분산 워크로드를 실행해 볼 수 있습니다.

## 문제 해결

## 두 개의 Spark에서 실행하기 위한 일반적인 문제

| 문제 | 원인 | 해결 방법 |
|-------|-------|----------|
| mpirun이 중단되거나 시간 초과됨 | SSH 연결 문제 | 1. 기본 SSH 연결 테스트: `ssh <remote_ip>`가 비밀번호 프롬프트 없이 작동해야 함<br>2. 간단한 mpirun 테스트 시도: `mpirun -np 2 -H <노드 1의 IP>:1,<노드 2의 IP>:1 hostname`<br>3. 모든 노드에 대해 SSH 키가 올바르게 설정되었는지 확인 |
| 네트워크 인터페이스를 찾을 수 없음 | 잘못된 인터페이스 이름 또는 다운 상태 | `ibdev2netdev`로 인터페이스 상태 확인 및 IP 구성 확인 |
| NCCL 빌드 실패 | OpenMPI와 같은 누락된 종속성 또는 잘못된 CUDA 버전 | CUDA 설치 및 필요한 라이브러리가 있는지 확인 |
