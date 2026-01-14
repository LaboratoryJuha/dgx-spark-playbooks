# 추론을 위한 vLLM

> DGX Spark에서 vLLM 설치 및 사용

## 목차

- [개요](#개요)
- [지침](#지침)
- [두 개의 Spark에서 실행](#두-개의-spark에서-실행)
  - [11단계. (선택 사항) 405B 추론 서버 시작](#11단계-선택-사항-405b-추론-서버-시작)
- [문제 해결](#문제-해결)

---

## 개요

## 기본 아이디어

vLLM은 대형 언어 모델을 효율적으로 실행하도록 설계된 추론 엔진입니다. 핵심 아이디어는 LLM을 제공할 때 **처리량을 최대화하고 메모리 낭비를 최소화**하는 것입니다.

- GPU 메모리 부족 없이 긴 시퀀스를 처리하기 위해 **PagedAttention**이라는 메모리 효율적인 주의 알고리즘을 사용합니다.
- **연속 배칭**을 통해 새 요청을 이미 처리 중인 배치에 추가하여 GPU를 완전히 활용할 수 있습니다.
- OpenAI API용으로 구축된 애플리케이션이 거의 또는 전혀 수정 없이 vLLM 백엔드로 전환할 수 있도록 **OpenAI 호환 API**가 있습니다.

## 달성할 내용

Blackwell 아키텍처를 사용하는 DGX Spark에서 vLLM 고처리량 LLM 제공을 설정하며,
사전 빌드된 Docker 컨테이너를 사용하거나 ARM64용 사용자 정의 LLVM/Triton
지원으로 소스에서 빌드합니다.

## 시작하기 전에 알아야 할 사항

- Docker로 컨테이너를 빌드하고 구성하는 경험
- CUDA 툴킷 설치 및 버전 관리에 대한 익숙함
- Python 가상 환경 및 패키지 관리에 대한 이해
- CMake 및 Ninja를 사용하여 소스에서 소프트웨어 빌드에 대한 지식
- Git 버전 제어 및 패치 관리에 대한 경험

## 전제 조건

- ARM64 프로세서 및 Blackwell GPU 아키텍처를 사용하는 DGX Spark 장치
- CUDA 13.0 툴킷 설치됨: `nvcc --version`은 CUDA 툴킷 버전을 표시합니다.
- Docker 설치 및 구성됨: `docker --version` 성공
- NVIDIA Container Toolkit 설치됨
- Python 3.12 사용 가능: `python3.12 --version` 성공
- Git 설치됨: `git --version` 성공
- 패키지 및 컨테이너 이미지를 다운로드하기 위한 네트워크 액세스


## 모델 지원 매트릭스

Spark의 vLLM에서 지원되는 모델은 다음과 같습니다. 나열된 모든 모델을 사용할 수 있으며 사용할 준비가 되어 있습니다:

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
| **Qwen2.5-VL-7B-Instruct** | NVFP4 | ✅ | `nvidia/Qwen2.5-VL-7B-Instruct-FP4` |
| **Phi-4-multimodal-instruct** | FP8 | ✅ | `nvidia/Phi-4-multimodal-instruct-FP8` |
| **Phi-4-multimodal-instruct** | NVFP4 | ✅ | `nvidia/Phi-4-multimodal-instruct-FP4` |
| **Phi-4-reasoning-plus** | FP8 | ✅ | `nvidia/Phi-4-reasoning-plus-FP8` |
| **Phi-4-reasoning-plus** | NVFP4 | ✅ | `nvidia/Phi-4-reasoning-plus-FP4` |


> [!NOTE]
> Phi-4-multimodal-instruct 모델은 vLLM을 시작할 때 `--trust-remote-code`가 필요합니다.

> [!NOTE]
> NVFP4 양자화 문서를 사용하여 좋아하는 모델에 대한 고유한 NVFP4 양자화 체크포인트를 생성할 수 있습니다. 이를 통해 NVIDIA에서 이미 게시하지 않은 모델에 대해서도 NVFP4 양자화의 성능 및 메모리 이점을 활용할 수 있습니다.

참고: 모든 모델 아키텍처가 NVFP4 양자화를 지원하는 것은 아닙니다.

## 시간 및 위험

* **소요 시간:** Docker 접근 방식의 경우 30분
* **위험:** 컨테이너 레지스트리 액세스에는 내부 자격 증명이 필요함
* **롤백:** 컨테이너 접근 방식은 비파괴적입니다.
* **최종 업데이트:** 01/02/2026
  * 지원되는 모델 매트릭스 추가 (25.11-py3)
  * 클러스터 설정 지침 개선

## 지침

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

## 2단계. vLLM 컨테이너 이미지 풀

https://catalog.ngc.nvidia.com/orgs/nvidia/containers/vllm에서 최신 컨테이너 빌드를 찾으십시오

```bash
export LATEST_VLLM_VERSION=<latest_container_version>

## 예제
## export LATEST_VLLM_VERSION=25.11-py3

docker pull nvcr.io/nvidia/vllm:${LATEST_VLLM_VERSION}
```

## 3단계. 컨테이너에서 vLLM 테스트

컨테이너를 시작하고 테스트 모델로 vLLM 서버를 시작하여 기본 기능을 확인하십시오.

```bash
docker run -it --gpus all -p 8000:8000 \
nvcr.io/nvidia/vllm:${LATEST_VLLM_VERSION} \
vllm serve "Qwen/Qwen2.5-Math-1.5B-Instruct"
```

예상 출력에는 다음이 포함되어야 합니다:
- 모델 로딩 확인
- 포트 8000에서 서버 시작
- GPU 메모리 할당 세부 정보

다른 터미널에서 서버를 테스트하십시오:

```bash
curl http://localhost:8000/v1/chat/completions \
-H "Content-Type: application/json" \
-d '{
    "model": "Qwen/Qwen2.5-Math-1.5B-Instruct",
    "messages": [{"role": "user", "content": "12*17"}],
    "max_tokens": 500
}'
```

예상 응답에는 `"content": "204"` 또는 유사한 수학 계산이 포함되어야 합니다.

## 4단계. 정리 및 롤백

컨테이너 접근 방식의 경우(비파괴적):

```bash
docker rm $(docker ps -aq --filter ancestor=nvcr.io/nvidia/vllm:${LATEST_VLLM_VERSION})
docker rmi nvcr.io/nvidia/vllm
```


CUDA 12.9를 제거하려면:

```bash
sudo /usr/local/cuda-12.9/bin/cuda-uninstaller
```

## 5단계. 다음 단계

- **프로덕션 배포:** 특정 모델 요구 사항에 맞게 vLLM 구성
- **성능 튜닝:** 워크로드에 맞게 배치 크기 및 메모리 설정 조정
- **모니터링:** 프로덕션 사용을 위한 로깅 및 메트릭 수집 설정
- **모델 관리:** 추가 모델 형식 및 양자화 옵션 탐색

## 두 개의 Spark에서 실행

## 1단계. 네트워크 연결 구성

[두 개의 Spark 연결](https://build.nvidia.com/spark/connect-two-sparks) 플레이북의 네트워크 설정 지침에 따라 DGX Spark 노드 간 연결을 설정하십시오.

여기에는 다음이 포함됩니다:
- 물리적 QSFP 케이블 연결
- 네트워크 인터페이스 구성(자동 또는 수동 IP 할당)
- 비밀번호 없는 SSH 설정
- 네트워크 연결 확인

## 2단계. 클러스터 배포 스크립트 다운로드

두 노드 모두에서 vLLM 클러스터 배포 스크립트를 가져오십시오. 이 스크립트는 분산 추론에 필요한 Ray 클러스터 설정을 조율합니다.

```bash
## 두 노드 모두에서 다운로드
wget https://raw.githubusercontent.com/vllm-project/vllm/refs/heads/main/examples/online_serving/run_cluster.sh
chmod +x run_cluster.sh
```

## 3단계. NGC에서 NVIDIA vLLM 이미지 풀

먼저 NGC에서 풀하도록 docker를 구성해야 합니다
처음으로 docker를 사용하는 경우 다음을 실행하십시오:
```bash
sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker
```

이후 `sudo`를 사용하지 않고 docker 명령을 실행할 수 있어야 합니다.


```bash
docker pull nvcr.io/nvidia/vllm:25.11-py3
export VLLM_IMAGE=nvcr.io/nvidia/vllm:25.11-py3
```


## 4단계. Ray 헤드 노드 시작

노드 1에서 Ray 클러스터 헤드 노드를 시작하십시오. 이 노드는 분산 추론을 조정하고 API 엔드포인트를 제공합니다.

```bash
## 노드 1에서 헤드 노드 시작

## 고속 인터페이스의 IP 주소 가져오기
## ibdev2netdev의 "(Up)"을 표시하는 인터페이스 사용(enp1s0f0np0 또는 enp1s0f1np1)
export MN_IF_NAME=enp1s0f1np1
export VLLM_HOST_IP=$(ip -4 addr show $MN_IF_NAME | grep -oP '(?<=inet\s)\d+(\.\d+){3}')

echo "Using interface $MN_IF_NAME with IP $VLLM_HOST_IP"

bash run_cluster.sh $VLLM_IMAGE $VLLM_HOST_IP --head ~/.cache/huggingface \
  -e VLLM_HOST_IP=$VLLM_HOST_IP \
  -e UCX_NET_DEVICES=$MN_IF_NAME \
  -e NCCL_SOCKET_IFNAME=$MN_IF_NAME \
  -e OMPI_MCA_btl_tcp_if_include=$MN_IF_NAME \
  -e GLOO_SOCKET_IFNAME=$MN_IF_NAME \
  -e TP_SOCKET_IFNAME=$MN_IF_NAME \
  -e RAY_memory_monitor_refresh_ms=0 \
  -e MASTER_ADDR=$VLLM_HOST_IP
```


## 5단계. Ray 작업자 노드 시작

노드 2를 작업자 노드로 Ray 클러스터에 연결하십시오. 이것은 텐서 병렬 처리를 위한 추가 GPU 리소스를 제공합니다.

```bash
## 노드 2에서 작업자로 연결

## 인터페이스 이름 설정(노드 1과 동일)
export MN_IF_NAME=enp1s0f1np1

## 노드 2의 자체 IP 주소 가져오기
export VLLM_HOST_IP=$(ip -4 addr show $MN_IF_NAME | grep -oP '(?<=inet\s)\d+(\.\d+){3}')

## 중요: HEAD_NODE_IP를 노드 1의 IP 주소로 설정
## 노드 1에서 이 값을 가져와야 합니다(노드 1에서 실행: echo $VLLM_HOST_IP)
export HEAD_NODE_IP=<NODE_1_IP_ADDRESS>

echo "Worker IP: $VLLM_HOST_IP, connecting to head node at: $HEAD_NODE_IP"

bash run_cluster.sh $VLLM_IMAGE $HEAD_NODE_IP --worker ~/.cache/huggingface \
  -e VLLM_HOST_IP=$VLLM_HOST_IP \
  -e UCX_NET_DEVICES=$MN_IF_NAME \
  -e NCCL_SOCKET_IFNAME=$MN_IF_NAME \
  -e OMPI_MCA_btl_tcp_if_include=$MN_IF_NAME \
  -e GLOO_SOCKET_IFNAME=$MN_IF_NAME \
  -e TP_SOCKET_IFNAME=$MN_IF_NAME \
  -e RAY_memory_monitor_refresh_ms=0 \
  -e MASTER_ADDR=$HEAD_NODE_IP
```
> **참고:** `<NODE_1_IP_ADDRESS>`를 노드 1의 실제 IP 주소, 특히 [두 개의 Spark 연결](https://build.nvidia.com/spark/connect-two-sparks) 플레이북에서 구성된 QSFP 인터페이스 nep1s0f1np1로 교체하십시오.

## 6단계. 클러스터 상태 확인

두 노드 모두 인식되고 Ray 클러스터에서 사용 가능한지 확인하십시오.

```bash
## 노드 1(헤드 노드)에서
## vLLM 컨테이너 이름 찾기(node-<random_number>가 됨)
export VLLM_CONTAINER=$(docker ps --format '{{.Names}}' | grep -E '^node-[0-9]+$')
echo "Found container: $VLLM_CONTAINER"

docker exec $VLLM_CONTAINER ray status
```

예상 출력은 사용 가능한 GPU 리소스가 있는 2개의 노드를 표시합니다.

## 7단계. Llama 3.3 70B 모델 다운로드

Hugging Face로 인증하고 권장되는 프로덕션 준비 모델을 다운로드하십시오.

```bash
## `ray status`가 실행된 동일한 컨테이너 내에서 다음을 실행
hf auth login
hf download meta-llama/Llama-3.3-70B-Instruct
```

## 8단계. Llama 3.3 70B용 추론 서버 시작

두 노드에서 텐서 병렬 처리를 사용하여 vLLM 추론 서버를 시작하십시오.

```bash
## 노드 1에서 컨테이너에 들어가 서버 시작
export VLLM_CONTAINER=$(docker ps --format '{{.Names}}' | grep -E '^node-[0-9]+$')
docker exec -it $VLLM_CONTAINER /bin/bash -c '
  vllm serve meta-llama/Llama-3.3-70B-Instruct \
    --tensor-parallel-size 2 --max_model_len 2048'
```

## 9단계. 70B 모델 추론 테스트

샘플 추론 요청으로 배포를 확인하십시오.

```bash
## 노드 1 또는 외부 클라이언트에서 테스트
curl http://localhost:8000/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "meta-llama/Llama-3.3-70B-Instruct",
    "prompt": "Write a haiku about a GPU",
    "max_tokens": 32,
    "temperature": 0.7
  }'
```

예상 출력에는 생성된 하이쿠 응답이 포함됩니다.

## 10단계. (선택 사항) Llama 3.1 405B 모델 배포

> [!WARNING]
> 405B 모델은 프로덕션 사용을 위한 메모리 여유가 부족합니다.

테스트 목적으로만 양자화된 405B 모델을 다운로드하십시오.

```bash
## 노드 1에서 양자화된 모델 다운로드
huggingface-cli download hugging-quants/Meta-Llama-3.1-405B-Instruct-AWQ-INT4
```

### 11단계. (선택 사항) 405B 추론 서버 시작

대형 모델에 대해 메모리 제약 매개변수로 서버를 시작하십시오.

```bash
## 노드 1에서 제한된 매개변수로 시작
export VLLM_CONTAINER=$(docker ps --format '{{.Names}}' | grep -E '^node-[0-9]+$')
docker exec -it $VLLM_CONTAINER /bin/bash -c '
  vllm serve hugging-quants/Meta-Llama-3.1-405B-Instruct-AWQ-INT4 \
    --tensor-parallel-size 2 --max-model-len 256 --gpu-memory-utilization 1.0 \
    --max-num-seqs 1 --max_num_batched_tokens 256'
```

## 12단계. (선택 사항) 405B 모델 추론 테스트

제한된 매개변수로 405B 배포를 확인하십시오.

```bash
curl http://localhost:8000/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "hugging-quants/Meta-Llama-3.1-405B-Instruct-AWQ-INT4",
    "prompt": "Write a haiku about a GPU",
    "max_tokens": 32,
    "temperature": 0.7
  }'
```

## 13단계. 배포 검증

분산 추론 시스템에 대한 포괄적인 검증을 수행하십시오.

```bash
## Ray 클러스터 상태 확인
export VLLM_CONTAINER=$(docker ps --format '{{.Names}}' | grep -E '^node-[0-9]+$')
docker exec $VLLM_CONTAINER ray status

## 서버 상태 엔드포인트 확인
curl http://192.168.100.10:8000/health

## 두 노드 모두에서 GPU 사용률 모니터링
nvidia-smi
export VLLM_CONTAINER=$(docker ps --format '{{.Names}}' | grep -E '^node-[0-9]+$')
docker exec $VLLM_CONTAINER nvidia-smi --query-gpu=memory.used,memory.total --format=csv
```

## 14단계. 다음 단계

클러스터 모니터링을 위한 Ray 대시보드에 액세스하고 추가 기능을 탐색하십시오:

```bash
## Ray 대시보드 사용 가능:
http://<head-node-ip>:8265

## 프로덕션을 위해 구현 고려:
## - 상태 확인 및 자동 재시작
## - 장기 실행 서비스를 위한 로그 회전
## - 재시작 간 지속적인 모델 캐싱
## - 대체 양자화 방법(FP8, INT4)
```

## 문제 해결

## 단일 Spark에서 실행하는 일반적인 문제

| 증상 | 원인 | 해결 방법 |
|---------|--------|-----|
| CUDA 버전 불일치 오류 | 잘못된 CUDA 툴킷 버전 | 정확한 설치 프로그램을 사용하여 CUDA 12.9 재설치 |
| 컨테이너 레지스트리 인증 실패 | 무효 또는 만료된 GitLab 토큰 | 새 인증 토큰 생성 |
| SM_121a 아키텍처가 인식되지 않음 | LLVM 패치 누락 | SM_121a 패치가 LLVM 소스에 적용되었는지 확인 |

## 두 개의 Spark에서 실행하는 일반적인 문제
| 증상 | 원인 | 해결 방법 |
|---------|--------|-----|
| 노드 2가 Ray 클러스터에 표시되지 않음 | 네트워크 연결 문제 | QSFP 케이블 연결 확인, IP 구성 확인 |
| URL에 대한 게이트된 리포지토리에 액세스할 수 없음 | 특정 HuggingFace 모델에 제한된 액세스가 있음 | [HuggingFace 토큰](https://huggingface.co/docs/hub/en/security-tokens)을 재생성하고 웹 브라우저에서 [게이트된 모델](https://huggingface.co/docs/hub/en/models-gated#customize-requested-information)에 대한 액세스 요청 |
| 모델 다운로드 실패 | 인증 또는 네트워크 문제 | `huggingface-cli login`을 다시 실행하고 인터넷 액세스 확인 |
| URL에 대한 게이트된 리포지토리에 액세스할 수 없음 | 특정 HuggingFace 모델에 제한된 액세스가 있음 | HuggingFace 토큰을 재생성하고 웹 브라우저에서 게이트된 모델에 대한 액세스 요청 |
| 405B로 CUDA 메모리 부족 | GPU 메모리 부족 | 70B 모델 사용 또는 max_model_len 매개변수 줄이기 |
| 컨테이너 시작 실패 | ARM64 이미지 누락 | ARM64 지침에 따라 vLLM 이미지 재빌드 |

> [!NOTE]
> DGX Spark는 GPU와 CPU 간의 동적 메모리 공유를 가능하게 하는 통합 메모리 아키텍처(UMA)를 사용합니다.
> UMA를 활용하도록 업데이트되는 많은 애플리케이션에서 여전히 DGX Spark의 메모리 용량 내에서도
> 메모리 문제가 발생할 수 있습니다. 그런 경우 다음과 같이 버퍼 캐시를 수동으로 플러시하십시오:
```bash
sudo sh -c 'sync; echo 3 > /proc/sys/vm/drop_caches'
```
