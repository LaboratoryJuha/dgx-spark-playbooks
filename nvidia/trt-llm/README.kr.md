# 추론을 위한 TRT LLM

> DGX Spark에서 TensorRT-LLM 설치 및 사용

## 목차

- [개요](#개요)
- [단일 Spark](#단일-spark)
- [두 개의 Spark에서 실행](#두-개의-spark에서-실행)
  - [1단계. 네트워크 연결 구성](#1단계-네트워크-연결-구성)
  - [2단계. Docker 권한 구성](#2단계-docker-권한-구성)
  - [3단계. OpenMPI hostfile 생성](#3단계-openmpi-hostfile-생성)
  - [4단계. 두 노드에서 컨테이너 시작](#4단계-두-노드에서-컨테이너-시작)
  - [5단계. 컨테이너가 실행 중인지 확인](#5단계-컨테이너가-실행-중인지-확인)
  - [6단계. 주 컨테이너에 hostfile 복사](#6단계-주-컨테이너에-hostfile-복사)
  - [7단계. 컨테이너 참조 저장](#7단계-컨테이너-참조-저장)
  - [8단계. 구성 파일 생성](#8단계-구성-파일-생성)
  - [9단계. 모델 다운로드](#9단계-모델-다운로드)
  - [10단계. 모델 제공](#10단계-모델-제공)
  - [11단계. API 서버 검증](#11단계-api-서버-검증)
  - [12단계. 정리 및 롤백](#12단계-정리-및-롤백)
  - [13단계. 다음 단계](#13단계-다음-단계)
- [TensorRT-LLM용 Open WebUI](#tensorrt-llm용-open-webui)
  - [1단계. TRT-LLM과 함께 Open WebUI를 사용하기 위한 전제 조건 설정](#1단계-trt-llm과-함께-open-webui를-사용하기-위한-전제-조건-설정)
  - [2단계. Open WebUI 컨테이너 시작](#2단계-open-webui-컨테이너-시작)
  - [3단계. Open WebUI 인터페이스 액세스](#3단계-open-webui-인터페이스-액세스)
  - [4단계. 정리 및 롤백](#4단계-정리-및-롤백)
- [문제 해결](#문제-해결)

---

## 개요

## 기본 아이디어

**NVIDIA TensorRT-LLM (TRT-LLM)**은 NVIDIA GPU에서 대형 언어 모델(LLM) 추론을 최적화하고 가속화하기 위한 오픈 소스 라이브러리입니다.

개발자가 더 낮은 지연 시간과 더 높은 처리량으로 LLM을 제공할 수 있도록 고도로 효율적인 커널, 메모리 관리 및 텐서, 파이프라인 및 시퀀스 병렬 처리와 같은 병렬 처리 전략을 제공합니다.

TRT-LLM은 Hugging Face 및 PyTorch와 같은 프레임워크와 통합되어 최첨단 모델을 대규모로 배포하기가 더 쉬워집니다.


## 달성할 내용

DGX Spark에서 TensorRT-LLM을 설정하여 대형 언어 모델을 최적화하고 배포하며, 커널 수준 최적화, 효율적인 메모리 레이아웃 및 고급 양자화를 통해 표준 PyTorch
추론보다 훨씬 높은 처리량과 낮은 지연 시간을 달성합니다.

## 시작하기 전에 알아야 할 사항

- Python 숙련도 및 PyTorch 또는 유사한 ML 프레임워크에 대한 경험
- CLI 도구 및 Docker 컨테이너 실행을 위한 명령줄 사용
- VRAM, 배칭 및 양자화(FP16/INT8)를 포함한 GPU 개념에 대한 기본 이해
- NVIDIA 소프트웨어 스택(CUDA Toolkit, 드라이버)에 대한 익숙함
- 추론 서버 및 컨테이너화된 환경에 대한 경험

## 전제 조건

- DGX Spark 장치
- CUDA 12.x와 호환되는 NVIDIA 드라이버: `nvidia-smi`
- Docker 설치 및 GPU 지원 구성: `docker run --rm --gpus all nvcr.io/nvidia/tensorrt-llm/release:1.2.0rc6 nvidia-smi`
- 모델 액세스를 위한 토큰이 있는 Hugging Face 계정: `echo $HF_TOKEN`
- 충분한 GPU VRAM(70B 모델의 경우 40GB+ 권장)
- 모델 및 컨테이너 이미지 다운로드를 위한 인터넷 연결
- 네트워크: OpenAI 호환 제공을 위한 호스트의 TCP 포트 8355(LLM) 및 8356(VLM) 열기

## 부속 파일

필요한 모든 자산은 [여기 GitHub에서](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main) 찾을 수 있습니다

- [**trtllm-mn-entrypoint.sh**](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main/nvidia/trt-llm/assets/trtllm-mn-entrypoint.sh) — 다중 노드 설정을 위한 컨테이너 엔트리포인트 스크립트

## 모델 지원 매트릭스

Spark의 TensorRT-LLM에서 지원되는 모델은 다음과 같습니다. 나열된 모든 모델을 사용할 수 있으며 사용할 준비가 되어 있습니다:

| 모델 | 양자화 | 지원 상태 | HF 핸들 |
|-------|-------------|----------------|-----------|
| **GPT-OSS-20B** | MXFP4 | ✅ | `openai/gpt-oss-20b` |
| **GPT-OSS-120B** | MXFP4 | ✅ | `openai/gpt-oss-120b` |
| **Llama-3.1-8B-Instruct** | FP8 | ✅ | `nvidia/Llama-3.1-8B-Instruct-FP8` |
| **Llama-3.1-8B-Instruct** | NVFP4 | ✅ | `nvidia/Llama-3.1-8B-Instruct-FP4` |
| **Llama-3.3-70B-Instruct** | NVFP4 | ✅ | `nvidia/Llama-3.3-70B-Instruct-FP4` |
| **Qwen3-8B** | FP8 | ✅ | `nvidia/Qwen3-8B-FP8` |
| **Qwen3-8B** | NVFP4 | ✅ | `nvidia/Qwen3-8B-FP4` |
| **Qwen3-14B** | FP8 | ✅ | `nvidia/Qwen3-14B-FP8` |
| **Qwen3-14B** | NVFP4 | ✅ | `nvidia/Qwen3-14B-FP4` |
| **Qwen3-32B** | NVFP4 | ✅ | `nvidia/Qwen3-32B-FP4` |
| **Phi-4-multimodal-instruct** | FP8 | ✅ | `nvidia/Phi-4-multimodal-instruct-FP8` |
| **Phi-4-multimodal-instruct** | NVFP4 | ✅ | `nvidia/Phi-4-multimodal-instruct-FP4` |
| **Phi-4-reasoning-plus** | FP8 | ✅ | `nvidia/Phi-4-reasoning-plus-FP8` |
| **Phi-4-reasoning-plus** | NVFP4 | ✅ | `nvidia/Phi-4-reasoning-plus-FP4` |
| **Qwen3-30B-A3B** | NVFP4 | ✅ | `nvidia/Qwen3-30B-A3B-FP4` |
| **Llama-4-Scout-17B-16E-Instruct** | NVFP4 | ✅ | `nvidia/Llama-4-Scout-17B-16E-Instruct-FP4` |
| **Qwen3-235B-A22B (두 개의 Spark만 해당)** | NVFP4 | ✅ | `nvidia/Qwen3-235B-A22B-FP4` |

> [!NOTE]
> NVFP4 양자화 문서를 사용하여 좋아하는 모델에 대한 고유한 NVFP4 양자화 체크포인트를 생성할 수 있습니다. 이를 통해 NVIDIA에서 이미 게시하지 않은 모델에 대해서도 NVFP4 양자화의 성능 및 메모리 이점을 활용할 수 있습니다.

참고: 모든 모델 아키텍처가 NVFP4 양자화를 지원하는 것은 아닙니다.

## 시간 및 위험

* **소요 시간**: 설정 및 API 서버 배포에 45-60분
* **위험 수준**: 중간 - 컨테이너 풀 및 모델 다운로드가 네트워크 문제로 인해 실패할 수 있음
* **롤백**: 추론 서버를 중지하고 다운로드한 모델을 제거하여 리소스를 확보합니다.
* **최종 업데이트:** 01/02/2026
  * 두 개의 Spark에서 TRT-LLM 실행 워크플로 개선
  * 최신 TRT-LLM 컨테이너 v1.2.0rc6으로 업그레이드

## 단일 Spark

## 1단계. Docker 권한 구성

sudo 없이 컨테이너를 쉽게 관리하려면 `docker` 그룹에 있어야 합니다. 이 단계를 건너뛰면 sudo로 Docker 명령을 실행해야 합니다.

새 터미널을 열고 Docker 액세스를 테스트하십시오. 터미널에서 다음을 실행하십시오:

```bash
docker ps
```

권한 거부 오류(Docker 데몬 소켓에 연결하는 동안 권한 거부와 같은 것)가 표시되면 sudo로 명령을 실행할 필요가 없도록 docker 그룹에 사용자를 추가하십시오.

```bash
sudo usermod -aG docker $USER
newgrp docker
```

## 2단계. 환경 전제 조건 확인

Spark 장치에 모델 및 컨테이너를 다운로드하는 데 필요한 GPU 액세스 및 네트워크 연결이 있는지 확인하십시오.

```bash
## GPU 가시성 및 드라이버 확인
nvidia-smi

## Docker GPU 지원 확인
docker run --rm --gpus all nvcr.io/nvidia/tensorrt-llm/release:1.2.0rc6 nvidia-smi

```

## 3단계. 환경 변수 설정

```bash
## 모델 액세스를 위해 `HF_TOKEN` 설정.
export HF_TOKEN=<your-huggingface-token>

export DOCKER_IMAGE="nvcr.io/nvidia/tensorrt-llm/release:1.2.0rc6"
```

## 4단계. TensorRT-LLM 설치 확인

GPU 액세스를 확인한 후 컨테이너 내부에서 TensorRT-LLM을 가져올 수 있는지 확인하십시오.

```bash
docker run --rm -it --gpus all \
  $DOCKER_IMAGE \
  python -c "import tensorrt_llm; print(f'TensorRT-LLM version: {tensorrt_llm.__version__}')"
```

예상 출력:
```
[TensorRT-LLM] TensorRT-LLM version: 1.2.0rc6
TensorRT-LLM version: 1.2.0rc6
```

## 5단계. 캐시 디렉토리 생성

후속 실행 시 모델을 다시 다운로드하지 않도록 로컬 캐싱을 설정하십시오.

```bash
## Hugging Face 캐시 디렉토리 생성
mkdir -p $HOME/.cache/huggingface/
```

## 6단계. quickstart_advanced로 설정 검증

이 빠른 시작은 모델 로딩, 추론 엔진 초기화 및 실제 텍스트 생성을 통한 GPU 실행을 테스트하여 TensorRT-LLM 설정을 엔드 투 엔드로 검증합니다. 추론 API 서버를 시작하기 전에 모든 것이 작동하는지 확인하는 가장 빠른 방법입니다.

**LLM 빠른 시작 예제**

#### Llama 3.1 8B Instruct
```bash
export MODEL_HANDLE="nvidia/Llama-3.1-8B-Instruct-FP4"

docker run \
  -e MODEL_HANDLE=$MODEL_HANDLE \
  -e HF_TOKEN=$HF_TOKEN \
  -v $HOME/.cache/huggingface/:/root/.cache/huggingface/ \
  --rm -it --ulimit memlock=-1 --ulimit stack=67108864 \
  --gpus=all --ipc=host --network host \
  $DOCKER_IMAGE \
  bash -c '
    hf download $MODEL_HANDLE && \
    python examples/llm-api/quickstart_advanced.py \
      --model_dir $MODEL_HANDLE \
      --prompt "Paris is great because" \
      --max_tokens 64
    '
```

#### GPT-OSS 20B
```bash
export MODEL_HANDLE="openai/gpt-oss-20b"

docker run \
  -e MODEL_HANDLE=$MODEL_HANDLE \
  -e HF_TOKEN=$HF_TOKEN \
  -v $HOME/.cache/huggingface/:/root/.cache/huggingface/ \
  --rm -it --ulimit memlock=-1 --ulimit stack=67108864 \
  --gpus=all --ipc=host --network host \
  $DOCKER_IMAGE \
  bash -c '
    export TIKTOKEN_ENCODINGS_BASE="/tmp/harmony-reqs" && \
    mkdir -p $TIKTOKEN_ENCODINGS_BASE && \
    wget -P $TIKTOKEN_ENCODINGS_BASE https://openaipublic.blob.core.windows.net/encodings/o200k_base.tiktoken && \
    wget -P $TIKTOKEN_ENCODINGS_BASE https://openaipublic.blob.core.windows.net/encodings/cl100k_base.tiktoken && \
    hf download $MODEL_HANDLE && \
    python examples/llm-api/quickstart_advanced.py \
      --model_dir $MODEL_HANDLE \
      --prompt "Paris is great because" \
      --max_tokens 64
    '
```

#### GPT-OSS 120B
```bash
export MODEL_HANDLE="openai/gpt-oss-120b"

docker run \
  -e MODEL_HANDLE=$MODEL_HANDLE \
  -e HF_TOKEN=$HF_TOKEN \
  -v $HOME/.cache/huggingface/:/root/.cache/huggingface/ \
  --rm -it --ulimit memlock=-1 --ulimit stack=67108864 \
  --gpus=all --ipc=host --network host \
  $DOCKER_IMAGE \
  bash -c '
    export TIKTOKEN_ENCODINGS_BASE="/tmp/harmony-reqs" && \
    mkdir -p $TIKTOKEN_ENCODINGS_BASE && \
    wget -P $TIKTOKEN_ENCODINGS_BASE https://openaipublic.blob.core.windows.net/encodings/o200k_base.tiktoken && \
    wget -P $TIKTOKEN_ENCODINGS_BASE https://openaipublic.blob.core.windows.net/encodings/cl100k_base.tiktoken && \
    hf download $MODEL_HANDLE && \
    python examples/llm-api/quickstart_advanced.py \
      --model_dir $MODEL_HANDLE \
      --prompt "Paris is great because" \
      --max_tokens 64
    '
```

## 7단계. quickstart_multimodal로 설정 검증

**VLM 빠른 시작 예제**

이것은 이미지 이해를 통한 추론을 실행하여 비전-언어 모델 기능을 보여줍니다. 이 예제는 멀티모달 입력을 사용하여 텍스트 및 비전 처리 파이프라인을 모두 검증합니다.

#### Phi-4-multimodal-instruct

이 모델은 파라미터 효율적인 미세 조정을 사용하므로 LoRA(Low-Rank Adaptation) 구성이 필요합니다. `--load_lora` 플래그는 멀티모달 지시 따르기를 위해 기본 모델을 적응시키는 LoRA 가중치 로딩을 활성화합니다.
```bash
export MODEL_HANDLE="nvidia/Phi-4-multimodal-instruct-FP4"

docker run \
  -e MODEL_HANDLE=$MODEL_HANDLE \
  -e HF_TOKEN=$HF_TOKEN \
  -v $HOME/.cache/huggingface/:/root/.cache/huggingface/ \
  --rm -it --ulimit memlock=-1 --ulimit stack=67108864 \
  --gpus=all --ipc=host --network host \
  $DOCKER_IMAGE \
  bash -c '
  python3 examples/llm-api/quickstart_multimodal.py \
    --model_type phi4mm \
    --model_dir $MODEL_HANDLE \
    --modality image \
    --media "https://huggingface.co/datasets/YiYiXu/testing-images/resolve/main/seashore.png" \
    --prompt "What is happening in this image?" \
    --load_lora \
    --auto_model_name Phi4MMForCausalLM
  '
```


> [!NOTE]
> 다운로드 또는 첫 실행 중에 호스트 OOM이 발생하면 호스트(컨테이너 외부)에서 OS 페이지 캐시를 확보하고 다시 시도하십시오:
```bash
sudo sh -c 'sync; echo 3 > /proc/sys/vm/drop_caches'
```

## 8단계. OpenAI 호환 API로 LLM 제공

trtllm-serve를 통해 OpenAI 호환 API로 제공:

#### Llama 3.1 8B Instruct
```bash
export MODEL_HANDLE="nvidia/Llama-3.1-8B-Instruct-FP4"

docker run --name trtllm_llm_server --rm -it --gpus all --ipc host --network host \
  -e HF_TOKEN=$HF_TOKEN \
  -e MODEL_HANDLE="$MODEL_HANDLE" \
  -v $HOME/.cache/huggingface/:/root/.cache/huggingface/ \
  $DOCKER_IMAGE \
  bash -c '
    hf download $MODEL_HANDLE && \
    cat > /tmp/extra-llm-api-config.yml <<EOF
print_iter_log: false
kv_cache_config:
  dtype: "auto"
  free_gpu_memory_fraction: 0.9
cuda_graph_config:
  enable_padding: true
disable_overlap_scheduler: true
EOF
    trtllm-serve "$MODEL_HANDLE" \
      --max_batch_size 64 \
      --trust_remote_code \
      --port 8355 \
      --extra_llm_api_options /tmp/extra-llm-api-config.yml
  '
```

#### GPT-OSS 20B
```bash
export MODEL_HANDLE="openai/gpt-oss-20b"

docker run --name trtllm_llm_server --rm -it --gpus all --ipc host --network host \
  -e HF_TOKEN=$HF_TOKEN \
  -e MODEL_HANDLE="$MODEL_HANDLE" \
  -v $HOME/.cache/huggingface/:/root/.cache/huggingface/ \
  $DOCKER_IMAGE \
  bash -c '
    export TIKTOKEN_ENCODINGS_BASE="/tmp/harmony-reqs" && \
    mkdir -p $TIKTOKEN_ENCODINGS_BASE && \
    wget -P $TIKTOKEN_ENCODINGS_BASE https://openaipublic.blob.core.windows.net/encodings/o200k_base.tiktoken && \
    wget -P $TIKTOKEN_ENCODINGS_BASE https://openaipublic.blob.core.windows.net/encodings/cl100k_base.tiktoken && \
    hf download $MODEL_HANDLE && \
    cat > /tmp/extra-llm-api-config.yml <<EOF
print_iter_log: false
kv_cache_config:
  dtype: "auto"
  free_gpu_memory_fraction: 0.9
cuda_graph_config:
  enable_padding: true
disable_overlap_scheduler: true
EOF
    trtllm-serve "$MODEL_HANDLE" \
      --max_batch_size 64 \
      --trust_remote_code \
      --port 8355 \
      --extra_llm_api_options /tmp/extra-llm-api-config.yml
  '
```

최소 OpenAI 스타일 채팅 요청. 별도의 터미널에서 실행하십시오.

```bash
curl -s http://localhost:8355/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "'"$MODEL_HANDLE"'",
    "messages": [{"role": "user", "content": "Paris is great because"}],
    "max_tokens": 64
  }'
```

## 9단계. 정리 및 롤백

테스트가 완료되면 다운로드한 모델 및 컨테이너를 제거하여 공간을 확보하십시오.

> [!WARNING]
> 이렇게 하면 모든 캐시된 모델이 삭제되며 향후 실행을 위해 다시 다운로드해야 할 수 있습니다.

```bash
## Hugging Face 캐시 제거
sudo chown -R "$USER:$USER" "$HOME/.cache/huggingface"
rm -rf $HOME/.cache/huggingface/

## Docker 이미지 정리
docker image prune -f
docker rmi $DOCKER_IMAGE
```

## 두 개의 Spark에서 실행

### 1단계. 네트워크 연결 구성

[두 개의 Spark 연결](https://build.nvidia.com/spark/connect-two-sparks/stacked-sparks) 플레이북의 네트워크 설정 지침에 따라 DGX Spark 노드 간 연결을 설정하십시오.

여기에는 다음이 포함됩니다:
- 물리적 QSFP 케이블 연결
- 네트워크 인터페이스 구성(자동 또는 수동 IP 할당)
- 비밀번호 없는 SSH 설정
- 네트워크 연결 확인

### 2단계. Docker 권한 구성

sudo 없이 컨테이너를 쉽게 관리하려면 `docker` 그룹에 있어야 합니다. 이 단계를 건너뛰면 sudo로 Docker 명령을 실행해야 합니다.

새 터미널을 열고 Docker 액세스를 테스트하십시오. 터미널에서 다음을 실행하십시오:

```bash
docker ps
```

권한 거부 오류(Docker 데몬 소켓에 연결하는 동안 권한 거부와 같은 것)가 표시되면 sudo로 명령을 실행할 필요가 없도록 docker 그룹에 사용자를 추가하십시오.

```bash
sudo usermod -aG docker $USER
newgrp docker
```

두 노드 모두에서 이 단계를 반복하십시오.

### 3단계. OpenMPI hostfile 생성

MPI 작업을 위해 두 노드의 IP 주소로 hostfile을 만드십시오. 각 노드에서 네트워크 인터페이스의 IP 주소를 가져오십시오:

```bash
ip a show enp1s0f0np0
```

또는 두 번째 인터페이스를 사용하는 경우:

```bash
ip a show enp1s0f1np1
```

IP 주소를 찾으려면 `inet` 줄을 찾으십시오(예: `192.168.1.10/24`).

주 노드에서 수집된 IP로 hostfile `~/openmpi-hostfile`을 만드십시오:

```bash
cat > ~/openmpi-hostfile <<EOF
192.168.1.10
192.168.1.11
EOF
```

IP 주소를 실제 노드 IP로 교체하십시오.

### 4단계. 두 노드에서 컨테이너 시작

**각 노드**(주 및 작업자)에서 다음 명령을 실행하여 TRT-LLM 컨테이너를 시작하십시오:

```bash
docker run -d --rm \
  --name trtllm-multinode \
  --gpus '"device=all"' \
  --network host \
  --ulimit memlock=-1 \
  --ulimit stack=67108864 \
  --device /dev/infiniband:/dev/infiniband \
  -e UCX_NET_DEVICES="enp1s0f0np0,enp1s0f1np1" \
  -e NCCL_SOCKET_IFNAME="enp1s0f0np0,enp1s0f1np1" \
  -e OMPI_MCA_btl_tcp_if_include="enp1s0f0np0,enp1s0f1np1" \
  -e OMPI_MCA_orte_default_hostfile="/etc/openmpi-hostfile" \
  -e OMPI_MCA_rmaps_ppr_n_pernode="1" \
  -e OMPI_ALLOW_RUN_AS_ROOT="1" \
  -e OMPI_ALLOW_RUN_AS_ROOT_CONFIRM="1" \
  -v ~/.cache/huggingface/:/root/.cache/huggingface/ \
  -v ~/.ssh:/tmp/.ssh:ro \
  nvcr.io/nvidia/tensorrt-llm/release:1.2.0rc6 \
  sh -c "curl https://raw.githubusercontent.com/NVIDIA/dgx-spark-playbooks/refs/heads/main/nvidia/trt-llm/assets/trtllm-mn-entrypoint.sh | sh"
```

> [!NOTE]
> 주 및 작업자 노드 **모두**에서 이 명령을 실행해야 합니다.

### 5단계. 컨테이너가 실행 중인지 확인

각 노드에서 컨테이너가 실행 중인지 확인하십시오:

```bash
docker ps
```

다음과 유사한 출력이 표시되어야 합니다:

```
CONTAINER ID   IMAGE                                                 COMMAND                  CREATED          STATUS          PORTS     NAMES
abc123def456   nvcr.io/nvidia/tensorrt-llm/release:1.2.0rc6         "sh -c 'curl https:…"    10 seconds ago   Up 8 seconds              trtllm-multinode
```

### 6단계. 주 컨테이너에 hostfile 복사

주 노드에서 OpenMPI hostfile을 컨테이너에 복사하십시오:

```bash
docker cp ~/openmpi-hostfile trtllm-multinode:/etc/openmpi-hostfile
```

### 7단계. 컨테이너 참조 저장

주 노드에서 편의를 위해 변수에 컨테이너 이름을 저장하십시오:

```bash
export TRTLLM_MN_CONTAINER=trtllm-multinode
```

### 8단계. 구성 파일 생성

주 노드에서 컨테이너 내부에 구성 파일을 생성하십시오:

```bash
docker exec $TRTLLM_MN_CONTAINER bash -c 'cat <<EOF > /tmp/extra-llm-api-config.yml
print_iter_log: false
kv_cache_config:
  dtype: "auto"
  free_gpu_memory_fraction: 0.9
cuda_graph_config:
  enable_padding: true
EOF'
```

### 9단계. 모델 다운로드

다음 명령을 사용하여 모델을 다운로드할 수 있습니다. `nvidia/Qwen3-235B-A22B-FP4`를 원하는 모델로 교체할 수 있습니다.

```bash
## 모델 다운로드를 위해 huggingface 토큰을 지정해야 합니다.
export HF_TOKEN=<your-huggingface-token>

docker exec \
  -e MODEL="nvidia/Qwen3-235B-A22B-FP4" \
  -e HF_TOKEN=$HF_TOKEN \
  -it $TRTLLM_MN_CONTAINER bash -c 'mpirun -x HF_TOKEN bash -c "hf download $MODEL"'
```

### 10단계. 모델 제공

주 노드에서 TensorRT-LLM 서버를 시작하십시오:

```bash
docker exec \
  -e MODEL="nvidia/Qwen3-235B-A22B-FP4" \
  -e HF_TOKEN=$HF_TOKEN \
  -it $TRTLLM_MN_CONTAINER bash -c '
    mpirun -x HF_TOKEN trtllm-llmapi-launch trtllm-serve $MODEL \
      --tp_size 2 \
      --backend pytorch \
      --max_num_tokens 32768 \
      --max_batch_size 4 \
      --extra_llm_api_options /tmp/extra-llm-api-config.yml \
      --port 8355'
```

이렇게 하면 포트 8355에서 TensorRT-LLM 서버가 시작됩니다. 그런 다음 OpenAI 호환 API 형식을 사용하여 `http://localhost:8355`에 추론 요청을 할 수 있습니다.

> [!NOTE]
> `UCX  WARN  network device 'enp1s0f0np0' is not available, please use one or more of`와 같은 경고가 표시될 수 있습니다. 추론이 성공하면 이 경고를 무시할 수 있습니다. 이는 두 개의 CX-7 포트 중 하나만 사용되고 다른 하나는 사용되지 않는 것과 관련이 있기 때문입니다.

**예상 출력:** 서버 시작 로그 및 준비 메시지.

### 11단계. API 서버 검증

서버가 실행되면 CURL 요청으로 테스트할 수 있습니다. 주 노드에서 다음을 실행하십시오:

```bash
curl -s http://localhost:8355/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "nvidia/Qwen3-235B-A22B-FP4",
    "messages": [{"role": "user", "content": "Paris is great because"}],
    "max_tokens": 64
  }'
```

**예상 출력:** 생성된 텍스트 완료가 포함된 JSON 응답.

### 12단계. 정리 및 롤백

**각 노드**에서 컨테이너를 중지하고 제거하십시오. 각 노드에 SSH로 연결하고 다음을 실행하십시오:

```bash
docker stop trtllm-multinode
```

> [!WARNING]
> 이렇게 하면 모든 추론 데이터 및 성능 보고서가 제거됩니다. 필요한 경우 정리하기 전에 필요한 파일을 복사하십시오.

각 노드에서 다운로드한 모델을 제거하여 디스크 공간을 확보하십시오:

```bash
rm -rf $HOME/.cache/huggingface/hub/models--nvidia--Qwen3*
```

### 13단계. 다음 단계

이제 DGX Spark 클러스터에 다른 모델을 배포할 수 있습니다.

## TensorRT-LLM용 Open WebUI

### 1단계. TRT-LLM과 함께 Open WebUI를 사용하기 위한 전제 조건 설정

단일 노드 또는 다중 노드 구성에서 TensorRT-LLM 추론 서버를 설정한 후
Open WebUI를 배포하여 Open WebUI를 통해 모델과 상호 작용할 수 있습니다. 설정하려면 다음 사항만
확인하면 됩니다

- TensorRT-LLM 추론 서버가 실행 중이고 http://localhost:8355에서 액세스 가능
- Docker 설치 및 구성(이전 단계 참조)
- DGX Spark에서 포트 3000 사용 가능

### 2단계. Open WebUI 컨테이너 시작

TensorRT-LLM 추론 서버가 실행 중인 DGX Spark 노드에서 다음 명령을 실행하십시오.
다중 노드 설정의 경우 이것은 주 노드가 됩니다.

> [!NOTE]
> OpenAI 호환 API 서버에 다른 포트를 사용한 경우 `OPENAI_API_BASE_URL="http://localhost:8355/v1"`을 TensorRT-LLM 추론 서버의 IP 및 포트와 일치하도록 조정하십시오.

```bash
docker run \
  -d \
  -e OPENAI_API_BASE_URL="http://localhost:8355/v1" \
  -v open-webui:/app/backend/data \
  --network host \
  --add-host=host.docker.internal:host-gateway \
  --name open-webui \
  --restart always \
  ghcr.io/open-webui/open-webui:main
```

이 명령은:
- http://localhost:8355에서 TensorRT-LLM용 OpenAI 호환 API 서버에 연결
- http://localhost:8080에서 Open WebUI 인터페이스에 대한 액세스 제공
- Docker 볼륨에 채팅 데이터 유지
- 자동 컨테이너 재시작 활성화
- 최신 Open WebUI 이미지 사용

### 3단계. Open WebUI 인터페이스 액세스

웹 브라우저를 열고 다음으로 이동하십시오:

```
http://localhost:8080
```

http://localhost:8080에서 Open WebUI 인터페이스가 표시되어야 하며 여기에서 다음을 수행할 수 있습니다:
- 배포된 모델과 채팅
- 모델 매개변수 조정
- 채팅 기록 보기
- 모델 구성 관리

왼쪽 상단 모서리의 드롭다운 메뉴에서 모델을 선택할 수 있습니다. 배포된 모델과 함께 Open WebUI를 사용하기 시작하는 데 필요한 모든 것입니다.

> [!NOTE]
> 원격 시스템에서 액세스하는 경우 localhost를 DGX Spark의 IP 주소로 교체하십시오.

### 4단계. 정리 및 롤백
> [!WARNING]
> 이렇게 하면 모든 채팅 데이터가 제거되며 향후 실행을 위해 다시 업로드해야 할 수 있습니다.

다음 명령을 사용하여 컨테이너를 제거하십시오:
```bash
docker stop open-webui
docker rm open-webui
docker volume rm open-webui
docker rmi ghcr.io/open-webui/open-webui:main
```

## 문제 해결

## 단일 Spark에서 실행하는 일반적인 문제

| 증상 | 원인 | 해결 방법 |
|---------|-------|-----|
| URL에 대한 게이트된 리포지토리에 액세스할 수 없음 | 특정 HuggingFace 모델에 제한된 액세스가 있음 | [HuggingFace 토큰](https://huggingface.co/docs/hub/en/security-tokens)을 재생성하고 웹 브라우저에서 [게이트된 모델](https://huggingface.co/docs/hub/en/models-gated#customize-requested-information)에 대한 액세스 요청 |
| 가중치 로딩 중 OOM(예: [Nemotron Super 49B](https://huggingface.co/nvidia/Llama-3_3-Nemotron-Super-49B-v1_5)) | 병렬 가중치 로딩 메모리 압력 | `export TRT_LLM_DISABLE_LOAD_WEIGHTS_IN_PARALLEL=1` |
| "CUDA out of memory" | 모델에 GPU VRAM 부족 | `free_gpu_memory_fraction: 0.9` 또는 배치 크기를 줄이거나 더 작은 모델 사용 |
| "Model not found" 오류 | HF_TOKEN 무효 또는 모델 액세스 불가 | 토큰 및 모델 권한 확인 |
| 컨테이너 풀 시간 초과 | 네트워크 연결 문제 | 풀 재시도 또는 로컬 미러 사용 |
| tensorrt_llm 가져오기 실패 | 컨테이너 런타임 문제 | Docker 데몬 재시작 및 재시도 |

## 두 개의 Spark에서 실행하는 일반적인 문제

| 증상 | 원인 | 해결 방법 |
|---------|-------|-----|
| MPI 호스트 이름 테스트가 단일 호스트 이름 반환 | 네트워크 연결 문제 | 두 노드 모두 연결 가능한 IP 주소에 있는지 확인 |
| HuggingFace 다운로드 시 "Permission denied" | HF_TOKEN 무효 또는 누락 | 유효한 토큰 설정: `export HF_TOKEN=<TOKEN>` |
| URL에 대한 게이트된 리포지토리에 액세스할 수 없음 | 특정 HuggingFace 모델에 제한된 액세스가 있음 | [HuggingFace 토큰](https://huggingface.co/docs/hub/en/security-tokens)을 재생성하고 웹 브라우저에서 [게이트된 모델](https://huggingface.co/docs/hub/en/models-gated#customize-requested-information)에 대한 액세스 요청 |
| "CUDA out of memory" 오류 | GPU 메모리 부족 | `--max_batch_size` 또는 `--max_num_tokens` 줄이기 |
| 컨테이너가 즉시 종료됨 | 엔트리포인트 스크립트 누락 | `trtllm-mn-entrypoint.sh` 다운로드가 성공하고 실행 가능한 권한이 있는지 확인하고, 노드에서 컨테이너를 이미 실행하고 있지 않은지 확인하십시오. 포트 2233이 이미 사용 중인 경우 엔트리포인트 스크립트가 시작되지 않습니다. |
| Error response from daemon: error while validating Root CA Certificate | 시스템 시계가 동기화되지 않았거나 인증서가 만료됨 | NTP 서버와 동기화하도록 시스템 시간 업데이트 `sudo timedatectl set-ntp true`|
| "invalid mount config for type 'bind'" | 엔트리포인트 스크립트 누락 또는 실행 불가 | `docker inspect <container_id>`를 실행하여 전체 오류 메시지를 확인하십시오. 홈 디렉토리에 `trtllm-mn-entrypoint.sh`가 있는지 확인(`ls -la $HOME/trtllm-mn-entrypoint.sh`)하고 실행 가능한 권한이 있는지 확인하십시오(`chmod +x $HOME/trtllm-mn-entrypoint.sh`) |
| "task: non-zero exit (255)" | 오류 코드 255로 컨테이너 종료 | `docker ps -a --filter "name=trtllm-multinode_trtllm"`로 컨테이너 로그를 확인하여 컨테이너 ID를 가져온 다음 `docker logs <container_id>`로 자세한 오류 메시지를 확인하십시오 |
| "insufficien..."로 "Pending" 상태에 Docker 상태 중단 | GPU 액세스를 위해 Docker 데몬이 올바르게 구성되지 않음 | 2-4단계가 성공적으로 완료되었는지 확인하고 `/etc/docker/daemon.json`에 올바른 GPU 구성이 포함되어 있는지 확인하십시오 |

> [!NOTE]
> DGX Spark는 GPU와 CPU 간의 동적 메모리 공유를 가능하게 하는 통합 메모리 아키텍처(UMA)를 사용합니다.
> UMA를 활용하도록 업데이트되는 많은 애플리케이션에서 여전히 DGX Spark의 메모리 용량 내에서도
> 메모리 문제가 발생할 수 있습니다. 그런 경우 다음과 같이 버퍼 캐시를 수동으로 플러시하십시오:
```bash
sudo sh -c 'sync; echo 3 > /proc/sys/vm/drop_caches'
```
