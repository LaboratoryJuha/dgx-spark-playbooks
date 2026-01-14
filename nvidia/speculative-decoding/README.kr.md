# 추측적 디코딩

> Spark에서 빠른 추론을 위한 추측적 디코딩 설정 방법 학습

## 목차

- [개요](#개요)
- [지침](#지침)
  - [옵션 1: EAGLE-3](#옵션-1-eagle-3)
  - [옵션 2: 드래프트 타겟](#옵션-2-드래프트-타겟)
- [문제 해결](#문제-해결)

---

## 개요

## 기본 개념

추측적 디코딩은 **작고 빠른 모델**을 사용하여 여러 토큰을 미리 초안으로 작성한 다음, **더 큰 모델**이 이를 빠르게 검증하거나 조정하도록 하여 텍스트 생성 속도를 높입니다.
이렇게 하면 큰 모델이 모든 토큰을 단계별로 예측할 필요가 없어 출력 품질을 유지하면서 지연 시간이 줄어듭니다.

## 달성할 목표

두 가지 접근 방식인 EAGLE-3와 드래프트-타겟을 사용하여 NVIDIA Spark에서 TensorRT-LLM을 사용한 추측적 디코딩을 탐색합니다.
이 예제들은 출력 품질을 유지하면서 대규모 언어 모델 추론을 가속화하는 방법을 보여줍니다.

## 시작하기 전에 알아야 할 사항

- Docker 및 컨테이너화된 애플리케이션 경험
- 추측적 디코딩 개념에 대한 이해
- TensorRT-LLM 서빙 및 API 엔드포인트에 대한 익숙함
- 대규모 언어 모델을 위한 GPU 메모리 관리에 대한 지식

## 필수 사항

- 충분한 GPU 메모리를 사용할 수 있는 NVIDIA Spark 장치
- GPU 지원이 활성화된 Docker

  ```bash
  docker run --gpus all nvcr.io/nvidia/tensorrt-llm/release:1.2.0rc6 nvidia-smi
  ```
- 모델 액세스를 위한 활성 HuggingFace 토큰
- 모델 다운로드를 위한 네트워크 연결


## 시간 및 위험도

* **기간:** 설정에 10-20분, 모델 다운로드에 추가 시간 (네트워크 속도에 따라 다름)
* **위험:** 대형 모델로 인한 GPU 메모리 고갈, 컨테이너 레지스트리 액세스 문제, 다운로드 중 네트워크 시간 초과
* **롤백:** Docker 컨테이너를 중지하고 선택적으로 다운로드된 모델 캐시를 정리합니다.
* **마지막 업데이트:** 01/02/2026
  * 최신 컨테이너 v1.2.0rc6로 업그레이드
  * GPT-OSS-120B를 사용한 EAGLE-3 추측적 디코딩 예제 추가

## 지침

## 1단계. Docker 권한 구성

sudo 없이 컨테이너를 쉽게 관리하려면 `docker` 그룹에 속해 있어야 합니다. 이 단계를 건너뛰면 sudo를 사용하여 Docker 명령을 실행해야 합니다.

새 터미널을 열고 Docker 액세스를 테스트합니다. 터미널에서 다음을 실행합니다:

```bash
docker ps
```

권한 거부 오류(Docker 데몬 소켓에 연결하는 동안 권한 거부와 같은)가 표시되면 docker 그룹에 사용자를 추가하여 sudo 없이 명령을 실행할 수 있도록 합니다.

```bash
sudo usermod -aG docker $USER
newgrp docker
```

## 2단계. 환경 변수 설정

다운스트림 서비스를 위한 환경 변수를 설정합니다:

 ```bash
export HF_TOKEN=<your_huggingface_token>
 ```

## 3단계. 추측적 디코딩 방법 실행

### 옵션 1: EAGLE-3

다음 명령을 실행하여 EAGLE-3 추측적 디코딩을 실행합니다:

```bash
docker run \
  -e HF_TOKEN=$HF_TOKEN \
  -v $HOME/.cache/huggingface/:/root/.cache/huggingface/ \
  --rm -it --ulimit memlock=-1 --ulimit stack=67108864 \
  --gpus=all --ipc=host --network host \
  nvcr.io/nvidia/tensorrt-llm/release:1.2.0rc6 \
  bash -c '
    hf download openai/gpt-oss-120b && \
    hf download nvidia/gpt-oss-120b-Eagle3-long-context \
        --local-dir /opt/gpt-oss-120b-Eagle3/ && \
    cat > /tmp/extra-llm-api-config.yml <<EOF
enable_attention_dp: false
disable_overlap_scheduler: false
enable_autotuner: false
cuda_graph_config:
    max_batch_size: 1
speculative_config:
    decoding_type: Eagle
    max_draft_len: 5
    speculative_model_dir: /opt/gpt-oss-120b-Eagle3/

kv_cache_config:
    free_gpu_memory_fraction: 0.9
    enable_block_reuse: false
EOF
    export TIKTOKEN_ENCODINGS_BASE="/tmp/harmony-reqs" && \
    mkdir -p $TIKTOKEN_ENCODINGS_BASE && \
    wget -P $TIKTOKEN_ENCODINGS_BASE https://openaipublic.blob.core.windows.net/encodings/o200k_base.tiktoken && \
    wget -P $TIKTOKEN_ENCODINGS_BASE https://openaipublic.blob.core.windows.net/encodings/cl100k_base.tiktoken
    trtllm-serve openai/gpt-oss-120b \
      --backend pytorch --tp_size 1 \
      --max_batch_size 1 \
      --extra_llm_api_options /tmp/extra-llm-api-config.yml'
```

서버가 실행되면 다른 터미널에서 API 호출을 하여 테스트합니다:

```bash
## 완성 엔드포인트 테스트
curl -X POST http://localhost:8000/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "openai/gpt-oss-120b",
    "prompt": "Solve the following problem step by step. If a train travels 180 km in 3 hours, and then slows down by 20% for the next 2 hours, what is the total distance traveled? Show all intermediate calculations and provide a final numeric answer.",
    "max_tokens": 300,
    "temperature": 0.7
  }'
```

**EAGLE-3 추측적 디코딩의 주요 특징**

- **더 간단한 배포** — 별도의 드래프트 모델을 관리하는 대신 EAGLE-3는 내부적으로 추측 토큰을 생성하는 내장 드래프팅 헤드를 사용합니다.

- **더 나은 정확도** — 모델의 여러 레이어에서 특징을 융합하여 드래프트 토큰이 수락될 가능성이 높아져 낭비되는 계산이 줄어듭니다.

- **더 빠른 생성** — 포워드 패스당 여러 토큰이 병렬로 검증되어 자기회귀 추론의 지연 시간이 줄어듭니다.

### 옵션 2: 드래프트 타겟

다음 명령을 실행하여 드래프트 타겟 추측적 디코딩을 설정하고 실행합니다:

```bash
docker run \
  -e HF_TOKEN=$HF_TOKEN \
  -v $HOME/.cache/huggingface/:/root/.cache/huggingface/ \
  --rm -it --ulimit memlock=-1 --ulimit stack=67108864 \
  --gpus=all --ipc=host --network host nvcr.io/nvidia/tensorrt-llm/release:1.2.0rc6 \
  bash -c "
#    # 모델 다운로드
    hf download nvidia/Llama-3.3-70B-Instruct-FP4 && \
    hf download nvidia/Llama-3.1-8B-Instruct-FP4 \
    --local-dir /opt/Llama-3.1-8B-Instruct-FP4/ && \

#    # 구성 파일 생성
    cat <<EOF > extra-llm-api-config.yml
print_iter_log: false
disable_overlap_scheduler: true
speculative_config:
  decoding_type: DraftTarget
  max_draft_len: 4
  speculative_model_dir: /opt/Llama-3.1-8B-Instruct-FP4/
kv_cache_config:
  enable_block_reuse: false
EOF

#    # TensorRT-LLM 서버 시작
    trtllm-serve nvidia/Llama-3.3-70B-Instruct-FP4 \
      --backend pytorch --tp_size 1 \
      --max_batch_size 1 \
      --kv_cache_free_gpu_memory_fraction 0.9 \
      --extra_llm_api_options ./extra-llm-api-config.yml
  "
```

서버가 실행되면 다른 터미널에서 API 호출을 하여 테스트합니다:

```bash
## 완성 엔드포인트 테스트
curl -X POST http://localhost:8000/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "nvidia/Llama-3.3-70B-Instruct-FP4",
    "prompt": "Explain the benefits of speculative decoding:",
    "max_tokens": 150,
    "temperature": 0.7
  }'
```

**드래프트-타겟의 주요 특징:**

- **효율적인 리소스 사용**: 8B 드래프트 모델이 70B 타겟 모델을 가속화
- **유연한 구성**: 최적화를 위해 조정 가능한 드래프트 토큰 길이
- **메모리 효율적**: 메모리 풋프린트를 줄이기 위해 FP4 양자화 모델 사용
- **호환 가능한 모델**: 일관된 토큰화를 가진 Llama 계열 모델 사용

## 4단계. 정리

완료되면 Docker 컨테이너를 중지합니다:

```bash
## 컨테이너 찾기 및 중지
docker ps
docker stop <container_id>

## 선택 사항: 캐시에서 다운로드된 모델 정리
## rm -rf $HOME/.cache/huggingface/hub/models--*gpt-oss*
```

## 5단계. 다음 단계

- 다양한 `max_draft_len` 값 (1, 2, 3, 4, 8) 실험
- 토큰 수락률 및 처리량 개선 모니터링
- 다양한 프롬프트 길이 및 생성 매개변수 테스트
- 추측적 디코딩에 대해 [여기](https://nvidia.github.io/TensorRT-LLM/advanced/speculative-decoding.html)에서 더 읽어보세요.

## 문제 해결

| 증상 | 원인 | 해결 방법 |
|---------|--------|-----|
| "CUDA out of memory" 오류 | GPU 메모리 부족 | `kv_cache_free_gpu_memory_fraction`을 0.9로 줄이거나 더 많은 VRAM을 가진 장치 사용 |
| 컨테이너 시작 실패 | Docker GPU 지원 문제 | `nvidia-docker`가 설치되어 있고 `--gpus=all` 플래그가 지원되는지 확인 |
| 모델 다운로드 실패 | 네트워크 또는 인증 문제 | HuggingFace 인증 및 네트워크 연결 확인 |
| URL에 대한 게이트 리포지토리에 액세스할 수 없음 | 특정 HuggingFace 모델은 액세스가 제한됨 | [HuggingFace 토큰](https://huggingface.co/docs/hub/en/security-tokens)을 재생성하고 웹 브라우저에서 [게이트 모델](https://huggingface.co/docs/hub/en/models-gated#customize-requested-information)에 대한 액세스 요청 |
| 서버가 응답하지 않음 | 포트 충돌 또는 방화벽 | 포트 8000이 사용 가능하고 차단되지 않았는지 확인 |

> [!NOTE]
> DGX Spark는 GPU와 CPU 간의 동적 메모리 공유를 가능하게 하는 통합 메모리 아키텍처(UMA)를 사용합니다.
> 많은 애플리케이션이 여전히 UMA를 활용하도록 업데이트 중이므로 DGX Spark의 메모리 용량 내에 있어도
> 메모리 문제가 발생할 수 있습니다. 이 경우 다음을 사용하여 수동으로 버퍼 캐시를 플러시하세요:
```bash
sudo sh -c 'sync; echo 3 > /proc/sys/vm/drop_caches'
```
