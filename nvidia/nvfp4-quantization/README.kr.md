# NVFP4 양자화

> TensorRT Model Optimizer를 사용하여 Spark에서 실행할 모델을 NVFP4로 양자화

## 목차

- [개요](#개요)
- [지침](#지침)
- [문제 해결](#문제-해결)

---

## 개요

## 기본 아이디어

NVFP4는 NVIDIA Blackwell GPU와 함께 도입된 4비트 부동 소수점 형식으로, 추론 워크로드의 메모리 대역폭 및 저장 요구 사항을 줄이면서 모델 정확도를 유지합니다.
균일한 INT4 양자화와 달리 NVFP4는 공유 지수와 컴팩트한 가수를 사용하여 부동 소수점 의미론을 유지하므로 더 높은 동적 범위와 더 안정적인 수렴이 가능합니다.
NVIDIA Blackwell Tensor Core는 FP16, FP8 및 FP4에 걸친 혼합 정밀도 실행을 기본적으로 지원하여 모델이 가중치 및 활성화에 FP4를 사용하면서 더 높은 정밀도(일반적으로 FP16)로 누적할 수 있습니다.
이 설계는 행렬 곱셈 중 양자화 오류를 최소화하고 미세 조정된 레이어별 양자화를 위한 TensorRT-LLM의 효율적인 변환 파이프라인을 지원합니다.

즉각적인 이점:
  - FP16 대비 메모리 사용량 약 3.5배, FP8 대비 약 1.8배 감소
  - FP8에 가까운 정확도 유지(일반적으로 <1% 손실)
  - 추론을 위한 속도 및 에너지 효율성 향상


## 달성할 내용

TensorRT-LLM 컨테이너 내에서 NVIDIA의 TensorRT Model Optimizer를 사용하여 DeepSeek-R1-Distill-Llama-8B 모델을 양자화하여 NVIDIA DGX Spark에 배포할 수 있는 NVFP4 양자화 모델을 생성합니다.

예제는 레이어 정밀도를 줄여 모델 크기를 약 2배로 줄이는 데 도움이 되는 NVIDIA FP4 양자화 모델을 사용합니다.
이 양자화 접근 방식은 상당한 처리량 향상을 제공하면서 정확도를 유지하는 것을 목표로 합니다. 그러나 양자화는 모델 정확도에 잠재적으로 영향을 미칠 수 있다는 점에 유의해야 합니다. 양자화된 모델이 사용 사례에 대해 허용 가능한 성능을 유지하는지 확인하기 위해 평가를 실행하는 것이 좋습니다.

## 시작하기 전에 알아야 할 사항

- Docker 컨테이너 및 GPU 가속 워크로드 작업
- 모델 양자화 개념 및 추론 성능에 미치는 영향에 대한 이해
- NVIDIA TensorRT 및 CUDA 툴킷 환경 경험
- Hugging Face 모델 리포지토리 및 인증에 대한 친숙함

## 전제 조건

- Blackwell 아키텍처 GPU가 있는 NVIDIA Spark 장치
- GPU 지원이 포함된 Docker 설치
- NVIDIA Container Toolkit 구성
- 모델 파일 및 출력을 위한 사용 가능한 저장 공간
- 대상 모델에 대한 액세스 권한이 있는 Hugging Face 계정

설정 확인:
```bash
## Docker GPU 액세스 확인
docker run --rm --gpus all nvcr.io/nvidia/tensorrt-llm/release:spark-single-gpu-dev nvidia-smi

## 충분한 디스크 공간 확인
df -h .
```

## 시간 및 위험

* **예상 소요 시간**: 네트워크 속도 및 모델 크기에 따라 45-90분
* **위험**:
  * 네트워크 문제 또는 Hugging Face 인증 문제로 인해 모델 다운로드가 실패할 수 있음
  * 양자화 프로세스는 메모리 집약적이며 GPU 메모리가 부족한 시스템에서 실패할 수 있음
  * 출력 파일이 크며(수 GB) 충분한 저장 공간이 필요함
* **롤백**: 출력 디렉토리 및 가져온 Docker 이미지를 제거하여 원래 상태로 복원
* **마지막 업데이트**: 12/15/2025
  * 8단계의 손상된 클라이언트 CURL 요청 수정
  * ModelOptimizer 프로젝트 이름 업데이트

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

## 2단계. 환경 준비

양자화된 모델 파일이 저장될 로컬 출력 디렉토리를 만듭니다. 이 디렉토리는 컨테이너가 종료된 후 결과를 유지하기 위해 컨테이너에 마운트됩니다.

```bash
mkdir -p ./output_models
chmod 755 ./output_models
```

## 3단계. Hugging Face로 인증

Hugging Face 인증 토큰을 설정하여 DeepSeek 모델에 대한 액세스 권한이 있는지 확인합니다.

```bash
## Hugging Face 토큰을 환경 변수로 내보내기
## 토큰을 다음에서 가져오기: https://huggingface.co/settings/tokens
export HF_TOKEN="your_token_here"
```

토큰은 모델 다운로드를 위해 컨테이너에서 자동으로 사용됩니다.

## 4단계. TensorRT Model Optimizer 컨테이너 실행

GPU 액세스, 멀티 GPU 워크로드에 최적화된 IPC 설정 및 모델 캐싱 및 출력 지속성을 위한 볼륨 마운트와 함께 TensorRT-LLM 컨테이너를 시작합니다.

```bash
docker run --rm -it --gpus all --ipc=host --ulimit memlock=-1 --ulimit stack=67108864 \
  -v "./output_models:/workspace/output_models" \
  -v "$HOME/.cache/huggingface:/root/.cache/huggingface" \
  -e HF_TOKEN=$HF_TOKEN \
  nvcr.io/nvidia/tensorrt-llm/release:spark-single-gpu-dev \
  bash -c "
    git clone -b 0.35.0 --single-branch https://github.com/NVIDIA/Model-Optimizer.git /app/TensorRT-Model-Optimizer && \
    cd /app/TensorRT-Model-Optimizer && pip install -e '.[dev]' && \
    export ROOT_SAVE_PATH='/workspace/output_models' && \
    /app/TensorRT-Model-Optimizer/examples/llm_ptq/scripts/huggingface_example.sh \
    --model 'deepseek-ai/DeepSeek-R1-Distill-Llama-8B' \
    --quant nvfp4 \
    --tp 1 \
    --export_fmt hf
  "
```

참고: 일부 환경에서 `pynvml.NVMLError_NotSupported: Not Supported` 오류가 발생할 수 있습니다. 이는 예상되며 결과에 영향을 미치지 않으며 향후 릴리스에서 수정될 예정입니다.
참고: 모델이 너무 큰 경우 메모리 부족 오류가 발생할 수 있습니다. 대신 더 작은 모델을 양자화해 볼 수 있습니다.

이 명령:
- 전체 GPU 액세스 및 최적화된 공유 메모리 설정으로 컨테이너 실행
- 출력 디렉토리를 마운트하여 양자화된 모델 파일 유지
- Hugging Face 캐시를 마운트하여 모델 재다운로드 방지
- 소스에서 TensorRT Model Optimizer 클론 및 설치
- NVFP4 양자화 파라미터로 양자화 스크립트 실행

## 5단계. 양자화 프로세스 모니터링

양자화 프로세스는 다음을 포함한 진행 정보를 표시합니다:
- Hugging Face에서 모델 다운로드 진행률
- 양자화 보정 단계
- 모델 내보내기 및 검증 단계

## 6단계. 양자화된 모델 검증

컨테이너가 완료된 후 양자화된 모델 파일이 성공적으로 생성되었는지 확인합니다.

```bash
## 출력 디렉토리 내용 확인
ls -la ./output_models/

## 모델 파일이 있는지 확인
find ./output_models/ -name "*.bin" -o -name "*.safetensors" -o -name "config.json"
```

출력 디렉토리에 모델 가중치 파일, 구성 파일 및 토크나이저 파일이 표시되어야 합니다.

## 7단계. 모델 로딩 테스트

먼저 양자화된 모델의 경로를 설정합니다:

```bash
## 양자화된 모델 디렉토리 경로 설정
export MODEL_PATH="./output_models/saved_models_DeepSeek-R1-Distill-Llama-8B_nvfp4_hf/"
```

이제 간단한 테스트를 사용하여 양자화된 모델이 제대로 로드되는지 확인합니다:

```bash
docker run \
  -e HF_TOKEN=$HF_TOKEN \
  -v $HOME/.cache/huggingface/:/root/.cache/huggingface/ \
  -v "$MODEL_PATH:/workspace/model" \
  --rm -it --ulimit memlock=-1 --ulimit stack=67108864 \
  --gpus=all --ipc=host --network host \
  nvcr.io/nvidia/tensorrt-llm/release:spark-single-gpu-dev \
  bash -c '
    python examples/llm-api/quickstart_advanced.py \
      --model_dir /workspace/model/ \
      --prompt "Paris is great because" \
      --max_tokens 64
    '
```

## 8단계. OpenAI 호환 API로 모델 제공
양자화된 모델로 TensorRT-LLM OpenAI 호환 API 서버를 시작합니다.
먼저 양자화된 모델의 경로를 설정합니다:

```bash
## 양자화된 모델 디렉토리 경로 설정
export MODEL_PATH="./output_models/saved_models_DeepSeek-R1-Distill-Llama-8B_nvfp4_hf/"

docker run \
  -e HF_TOKEN=$HF_TOKEN \
  -v "$MODEL_PATH:/workspace/model" \
  --rm -it --ulimit memlock=-1 --ulimit stack=67108864 \
  --gpus=all --ipc=host --network host \
  nvcr.io/nvidia/tensorrt-llm/release:spark-single-gpu-dev \
  trtllm-serve /workspace/model \
    --backend pytorch \
    --max_batch_size 4 \
    --port 8000
```

다음을 실행하여 클라이언트 CURL 요청으로 서버를 테스트합니다:

```bash
curl -X POST http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-ai/DeepSeek-R1-Distill-Llama-8B",
    "messages": [{"role": "user", "content": "What is artificial intelligence?"}],
    "max_tokens": 100,
    "temperature": 0.7,
    "stream": false
  }'
```

## 10단계. 정리 및 롤백

환경을 정리하고 생성된 파일을 제거하려면:

> [!WARNING]
> 이렇게 하면 양자화된 모든 모델 파일과 캐시된 데이터가 영구적으로 삭제됩니다.

```bash
## 출력 디렉토리 및 모든 양자화된 모델 제거
rm -rf ./output_models

## Hugging Face 캐시 제거(선택 사항)
rm -rf ~/.cache/huggingface

## Docker 이미지 제거(선택 사항)
docker rmi nvcr.io/nvidia/tensorrt-llm/release:spark-single-gpu-dev
```

## 11단계. 다음 단계

양자화된 모델이 이제 배포 준비가 되었습니다. 일반적인 다음 단계는 다음과 같습니다:
- 원본 모델과 비교하여 추론 성능 벤치마킹
- 양자화된 모델을 추론 파이프라인에 통합
- 프로덕션 서빙을 위해 NVIDIA Triton Inference Server에 배포
- 특정 사용 사례에 대한 추가 검증 테스트 실행

## 문제 해결

| 증상 | 원인 | 해결 방법 |
|---------|--------|-----|
| Hugging Face 액세스 시 "Permission denied" | HF 토큰 누락 또는 잘못됨 | 유효한 토큰으로 `huggingface-cli login` 실행 |
| 컨테이너가 CUDA 메모리 부족으로 종료됨 | GPU 메모리 부족 | 배치 크기 줄이기 또는 GPU 메모리가 더 많은 머신 사용 |
| 출력 디렉토리에서 모델 파일을 찾을 수 없음 | 볼륨 마운트 실패 또는 잘못된 경로 | `$(pwd)/output_models`가 올바르게 해석되는지 확인 |
| 컨테이너 내부에서 Git 클론 실패 | 네트워크 연결 문제 | 인터넷 연결을 확인하고 재시도 |
| 양자화 프로세스가 중단됨 | 컨테이너 리소스 제한 | Docker 메모리 제한 늘리기 또는 `--ulimit` 플래그 사용 |
| URL에 대한 게이트된 리포지토리에 액세스할 수 없음 | 특정 HuggingFace 모델에는 제한된 액세스 권한이 있음 | [HuggingFace 토큰](https://huggingface.co/docs/hub/en/security-tokens)을 재생성하고 웹 브라우저에서 [게이트된 모델](https://huggingface.co/docs/hub/en/models-gated#customize-requested-information)에 대한 액세스를 요청하세요 |

> [!NOTE]
> DGX Spark는 GPU와 CPU 간에 동적 메모리 공유를 가능하게 하는 통합 메모리 아키텍처(UMA)를 사용합니다.
> 많은 애플리케이션이 여전히 UMA를 활용하도록 업데이트 중이므로 DGX Spark의 메모리 용량 내에 있더라도
> 메모리 문제가 발생할 수 있습니다. 이런 경우 다음과 같이 버퍼 캐시를 수동으로 플러시하세요:
```bash
sudo sh -c 'sync; echo 3 > /proc/sys/vm/drop_caches'
```
