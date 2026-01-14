# Pytorch로 미세 조정하기

> Pytorch를 사용하여 로컬에서 모델 미세 조정

## 목차

- [개요](#개요)
- [지침](#지침)
- [두 개의 Spark에서 실행](#두-개의-spark에서-실행)
  - [1단계. 네트워크 연결 구성](#1단계-네트워크-연결-구성)
  - [2단계. Docker 권한 구성](#2단계-docker-권한-구성)
  - [3단계. NVIDIA Container Toolkit 설치 및 Docker 환경 설정](#3단계-nvidia-container-toolkit-설치-및-docker-환경-설정)
  - [4단계. 리소스 알림 활성화](#4단계-리소스-알림-활성화)
  - [5단계. Docker Swarm 초기화](#5단계-docker-swarm-초기화)
  - [6단계. 워커 노드 연결 및 배포](#6단계-워커-노드-연결-및-배포)
  - [7단계. Docker 컨테이너 ID 찾기](#7단계-docker-컨테이너-id-찾기)
  - [9단계. 구성 파일 조정](#9단계-구성-파일-조정)
  - [10단계. 미세 조정 스크립트 실행](#10단계-미세-조정-스크립트-실행)
  - [14단계. 정리 및 롤백](#14단계-정리-및-롤백)
- [문제 해결](#문제-해결)

---

## 개요

## 기본 아이디어

이 플레이북은 NVIDIA Spark 장치에서 대규모 언어 모델을 미세 조정하기 위한 Pytorch 설정 및 사용 방법을 안내합니다.

## 달성할 내용

NVIDIA Spark 장치에서 대규모 언어 모델(1-70B 파라미터)을 위한 완전한 미세 조정 환경을 구축합니다.
완료 시, 파라미터 효율적 미세 조정(PEFT) 및 지도 미세 조정(SFT)을 지원하는 작동 가능한 설치 환경을 갖추게 됩니다.

## 시작하기 전에 알아야 할 사항

- Pytorch에서 미세 조정에 대한 이전 경험
- Docker 작업


## 전제 조건
레시피는 특별히 DIGITS SPARK용입니다. OS와 드라이버가 최신인지 확인하세요.


## 부속 파일

미세 조정에 필요한 모든 파일은 [GitHub 리포지토리 여기](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main/nvidia/pytorch-fine-tune)의 폴더에 포함되어 있습니다.

## 시간 및 위험

* **예상 시간:** 설정 및 미세 조정 실행에 30-45분. 미세 조정 실행 시간은 모델 크기에 따라 다름
* **위험:** 모델 다운로드 크기가 클 수 있음(수 GB), ARM64 패키지 호환성 문제로 인해 문제 해결이 필요할 수 있음
* **마지막 업데이트:** 01/02/2025
  * 두 개의 Spark 분산 미세 조정 예제 추가

## 지침

## 1단계. Docker 권한 구성

sudo 없이 컨테이너를 쉽게 관리하려면 `docker` 그룹에 속해야 합니다. 이 단계를 건너뛰면 Docker 명령을 sudo로 실행해야 합니다.

새 터미널을 열고 Docker 액세스를 테스트합니다. 터미널에서 다음을 실행합니다:

```bash
docker ps
```

권한 거부 오류가 표시되면(Docker 데몬 소켓에 연결하는 동안 권한 거부와 같은 메시지), sudo 없이 명령을 실행할 수 있도록 사용자를 docker 그룹에 추가합니다.

```bash
sudo usermod -aG docker $USER
newgrp docker
```

## 2단계. 최신 Pytorch 컨테이너 가져오기

```bash
docker pull nvcr.io/nvidia/pytorch:25.11-py3
```

## 3단계. Docker 시작

```bash
docker run --gpus all -it --rm --ipc=host \
-v $HOME/.cache/huggingface:/root/.cache/huggingface \
-v ${PWD}:/workspace -w /workspace \
nvcr.io/nvidia/pytorch:25.11-py3
```

## 4단계. 컨테이너 내부에 종속성 설치

```bash
pip install transformers peft datasets trl bitsandbytes
```

## 5단계: Huggingface로 인증

```bash
hf auth login
##<huggingface 토큰 입력.
##<git 자격 증명에 대해 n 입력>
```

## 6단계: 미세 조정 레시피가 있는 git 리포지토리 클론

```bash
git clone https://github.com/NVIDIA/dgx-spark-playbooks
cd dgx-spark-playbooks/nvidia/pytorch-fine-tune/assets
```

## 7단계: 미세 조정 레시피 실행

Llama3-8B에서 LoRA를 실행하려면 다음 명령을 사용합니다:
```bash
python Llama3_8B_LoRA_finetuning.py
```

llama3-3B에서 전체 미세 조정을 실행하려면 다음 명령을 사용합니다:
```bash
python Llama3_3B_full_finetuning.py
```

## 두 개의 Spark에서 실행

### 1단계. 네트워크 연결 구성

[Connect two Sparks](https://build.nvidia.com/spark/connect-two-sparks/stacked-sparks) 플레이북의 네트워크 설정 지침에 따라 DGX Spark 노드 간 연결을 설정합니다.

여기에는 다음이 포함됩니다:
- 물리적 QSFP 케이블 연결
- 네트워크 인터페이스 구성(자동 또는 수동 IP 할당)
- 비밀번호 없는 SSH 설정
- 네트워크 연결 확인

### 2단계. Docker 권한 구성

sudo 없이 컨테이너를 쉽게 관리하려면 `docker` 그룹에 속해야 합니다. 이 단계를 건너뛰면 Docker 명령을 sudo로 실행해야 합니다.

새 터미널을 열고 Docker 액세스를 테스트합니다. 터미널에서 다음을 실행합니다:

```bash
docker ps
```

권한 거부 오류가 표시되면(Docker 데몬 소켓에 연결하는 동안 권한 거부와 같은 메시지), sudo 없이 명령을 실행할 수 있도록 사용자를 docker 그룹에 추가합니다.

```bash
sudo usermod -aG docker $USER
newgrp docker
```
### 3단계. NVIDIA Container Toolkit 설치 및 Docker 환경 설정

GPU 리소스를 제공할 각 노드(관리자 및 워커 모두)에 NVIDIA 드라이버와 NVIDIA Container Toolkit이 설치되어 있는지 확인합니다. 이 패키지를 통해 Docker 컨테이너가 호스트의 GPU 하드웨어에 액세스할 수 있습니다. NVIDIA Container Toolkit에 대한 [Docker 구성](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html#configuring-docker)을 포함하여 [설치 단계](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)를 완료했는지 확인합니다.

### 4단계. 리소스 알림 활성화

먼저 다음을 실행하여 GPU UUID를 찾습니다:
```bash
nvidia-smi -a | grep UUID
```

다음으로 Docker 데몬 구성을 수정하여 GPU를 Swarm에 알립니다. **/etc/docker/daemon.json**을 편집합니다:

```bash
sudo nano /etc/docker/daemon.json
```

nvidia 런타임 및 GPU UUID를 포함하도록 파일을 추가하거나 수정합니다(**GPU-45cbf7b3-f919-7228-7a26-b06628ebefa1**을 실제 GPU UUID로 교체):

```json
{
  "runtimes": {
    "nvidia": {
      "path": "nvidia-container-runtime",
      "runtimeArgs": []
    }
  },
  "default-runtime": "nvidia",
  "node-generic-resources": [
    "NVIDIA_GPU=GPU-45cbf7b3-f919-7228-7a26-b06628ebefa1"
    ]
}
```

**config.toml** 파일에서 swarm-resource 줄의 주석을 제거하여 Swarm에 GPU를 알리도록 NVIDIA Container Runtime을 수정합니다. 선호하는 텍스트 편집기(예: vim, nano...)를 사용하거나 다음 명령을 사용할 수 있습니다:
```bash
sudo sed -i 's/^#\s*\(swarm-resource\s*=\s*".*"\)/\1/' /etc/nvidia-container-runtime/config.toml
```

마지막으로 Docker 데몬을 다시 시작하여 모든 변경 사항을 적용합니다:
```bash
sudo systemctl restart docker
```

모든 노드에서 이 단계를 반복합니다.

### 5단계. Docker Swarm 초기화

기본으로 사용하려는 노드에서 다음 swarm 초기화 명령을 실행합니다
```bash
docker swarm init --advertise-addr $(ip -o -4 addr show enp1s0f0np0 | awk '{print $4}' | cut -d/ -f1) $(ip -o -4 addr show enp1s0f1np1 | awk '{print $4}' | cut -d/ -f1)
```

위의 일반적인 출력은 다음과 유사합니다:
```
Swarm initialized: current node (node-id) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token <worker-token> <advertise-addr>:<port>

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

### 6단계. 워커 노드 연결 및 배포

이제 클러스터의 워커 노드 설정을 진행할 수 있습니다. 모든 워커 노드에서 이 단계를 반복합니다.

각 워커 노드에서 docker swarm init이 제안한 명령을 실행하여 Docker swarm에 연결합니다
```bash
docker swarm join --token <worker-token> <advertise-addr>:<port>
```

두 노드 모두에서 미세 조정 스크립트 및 구성 파일이 포함된 디렉토리에 [**pytorch-ft-entrypoint.sh**](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main/nvidia/pytorch-fine-tune/assets/pytorch-ft-entrypoint.sh) 스크립트를 다운로드하고 다음 명령을 실행하여 실행 가능하게 만듭니다:

```bash
chmod +x $PWD/pytorch-ft-entrypoint.sh
```

기본 노드에서 [**docker-compose.yml**](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main/nvidia/pytorch-fine-tune/assets/docker-compose.yml) 파일을 이전 단계와 동일한 디렉토리에 다운로드하고 다음 명령을 실행하여 미세 조정 멀티 노드 스택을 배포합니다:
```bash
docker stack deploy -c $PWD/docker-compose.yml finetuning-multinode
```
> [!NOTE]
> 명령을 실행하는 디렉토리에 두 파일을 모두 다운로드했는지 확인하세요.

다음을 사용하여 워커 노드의 상태를 확인할 수 있습니다
```bash
docker stack ps finetuning-multinode
```

모든 것이 정상이면 다음과 유사한 출력이 표시되어야 합니다:
```
nvidia@spark-1b3b:~$ docker stack ps finetuning-multinode
ID             NAME                                IMAGE                              NODE         DESIRED STATE   CURRENT STATE            ERROR     PORTS
vlun7z9cacf9   finetuning-multinode_finetunine.1   nvcr.io/nvidia/pytorch:25.10-py3   spark-1d84   Running         Starting 2 seconds ago
tjl49zicvxoi   finetuning-multinode_finetunine.2   nvcr.io/nvidia/pytorch:25.10-py3   spark-1b3b   Running         Starting 2 seconds ago

```

> [!NOTE]
> "Current state"가 "Running"이 아닌 경우 자세한 내용은 문제 해결 섹션을 참조하세요.

### 7단계. Docker 컨테이너 ID 찾기

`docker ps`를 사용하여 Docker 컨테이너 ID를 찾을 수 있습니다. 아래와 같이 변수에 컨테이너 ID를 저장할 수 있습니다. 두 노드에서 이 명령을 실행합니다.
```bash
export FINETUNING_CONTAINER=$(docker ps -q -f name=finetuning-multinode)
```

### 9단계. 구성 파일 조정

멀티 노드 실행을 위해 2개의 구성 파일을 제공합니다:
- [**config_finetuning.yaml**](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main/nvidia/pytorch-fine-tune/assets/configs/config_finetuning.yaml) Llama3 3B의 전체 미세 조정에 사용됩니다.
- [**config_fsdp_lora.yaml**](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main/nvidia/pytorch-fine-tune/assets/configs/config_fsdp_lora.yaml) Llama3 8B 및 Llama3 70B의 LoRa 및 FSDP로 미세 조정에 사용됩니다.

이러한 구성 파일은 다음과 같이 조정해야 합니다:
- 각 노드의 순위에 따라 각 노드에서 `machine_rank`를 설정합니다. 마스터 노드의 순위는 `0`이어야 합니다. 두 번째 노드의 순위는 `1`입니다.
- 마스터 노드의 IP 주소를 사용하여 `main_process_ip`를 설정합니다. 두 구성 파일 모두 동일한 값을 갖도록 합니다. 메인 노드에서 `ifconfig`를 사용하여 CX-7 IP 주소의 올바른 값을 찾습니다.
- 메인 노드에서 사용할 수 있는 포트 번호를 설정합니다.

YAML 파일에서 채워야 하는 필드:

```bash
machine_rank: 0
main_process_ip: < TODO: IP 지정 >
main_process_port: < TODO: 포트 지정 >
```

모든 스크립트 및 구성 파일은 이 [**리포지토리**](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main/nvidia/pytorch-fine-tune/assets)에서 사용할 수 있습니다.

### 10단계. 미세 조정 스크립트 실행

이전 단계를 성공적으로 실행한 후 이 [**리포지토리**](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main/nvidia/pytorch-fine-tune/assets)에서 사용 가능한 미세 조정을 위한 `run-multi-llama_*` 스크립트 중 하나를 사용할 수 있습니다. 다음은 미세 조정에 LoRa를 사용하고 FSDP2를 사용하는 Llama3 70B의 예입니다.

```bash
## 모델 다운로드를 위해 huggingface 토큰을 지정해야 합니다.
export HF_TOKEN=<your-huggingface-token>

docker exec \
  -e HF_TOKEN=$HF_TOKEN \
  -it $FINETUNING_CONTAINER bash -c '
  bash /workspace/install-requirements;
  accelerate launch --config_file=/workspace/configs/config_fsdp_lora.yaml /workspace/Llama3_70B_LoRA_finetuning.py'
```

실행 중에 미세 조정의 진행률 표시줄은 메인 노드의 stdout에만 나타납니다. 이는 `accelerate`가 [여기](https://github.com/huggingface/accelerate/blob/main/src/accelerate/utils/tqdm.py#L25)에 설명된 대로 메인 프로세스에만 진행률을 표시하기 위해 `tqdm` 주위에 래퍼를 사용하므로 예상되는 동작입니다. 워커 노드에서 `nvidia-smi`를 사용하면 GPU가 사용 중임을 확인할 수 있습니다.

### 14단계. 정리 및 롤백

리더 노드에서 다음 명령을 사용하여 컨테이너를 중지하고 제거합니다:

```bash
docker stack rm finetuning-multinode
```

다운로드한 모델을 제거하여 디스크 공간 확보:

```bash
rm -rf $HOME/.cache/huggingface/hub/models--meta-llama* $HOME/.cache/huggingface/hub/datasets*
```

## 문제 해결

| 증상 | 원인 | 해결 방법 |
|---------|--------|-----|
| URL에 대한 게이트된 리포지토리에 액세스할 수 없음 | 특정 HuggingFace 모델에는 제한된 액세스 권한이 있음 | [HuggingFace 토큰](https://huggingface.co/docs/hub/en/security-tokens)을 재생성하고 웹 브라우저에서 [게이트된 모델](https://huggingface.co/docs/hub/en/models-gated#customize-requested-information)에 대한 액세스를 요청하세요 |
| 멀티 Spark 실행에서 오류 및 시간 초과 | 다양한 이유 | 추가 로깅 및 런타임 일관성 검사를 활성화하기 위해 다음 변수를 설정하는 것이 좋습니다 <br> `ACCELERATE_DEBUG_MODE=1`<br> `ACCELERATE_LOG_LEVEL=DEBUG`<br> `TORCH_CPP_LOG_LEVEL=INFO`<br> `TORCH_DISTRIBUTED_DEBUG=DETAIL`|
| task: non-zero exit (255) | 오류 코드 255로 컨테이너 종료 | `docker ps -a --filter "name=finetuning-multinode"`로 컨테이너 로그를 확인하여 컨테이너 ID를 가져온 다음 `docker logs <container_id>`로 자세한 오류 메시지 확인 |
|Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running? | Docker Swarm이 오래되었거나 연결할 수 없는 링크 로컬 IP 주소에 바인딩하려고 시도하여 Docker 데몬 충돌이 발생함 | Docker 중지 `sudo systemctl stop docker`<br> Swarm 상태 제거 `sudo rm -rf /var/lib/docker/swarm`<br> Docker 다시 시작 `sudo systemctl start docker`<br> 활성 인터페이스에서 유효한 광고 주소로 Swarm 다시 초기화|

> [!NOTE]
> DGX Spark는 GPU와 CPU 간에 동적 메모리 공유를 가능하게 하는 통합 메모리 아키텍처(UMA)를 사용합니다.
> 많은 애플리케이션이 여전히 UMA를 활용하도록 업데이트 중이므로 DGX Spark의 메모리 용량 내에 있더라도
> 메모리 문제가 발생할 수 있습니다. 이런 경우 다음과 같이 버퍼 캐시를 수동으로 플러시하세요:
```bash
sudo sh -c 'sync; echo 3 > /proc/sys/vm/drop_caches'
```
