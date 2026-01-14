# LLaMA Factory

> LLaMA Factory로 모델 설치 및 미세 조정

## 목차

- [개요](#개요)
- [지침](#지침)
  - [4단계. 종속성과 함께 LLaMA Factory 설치](#4단계-종속성과-함께-llama-factory-설치)
- [문제 해결](#문제-해결)

---

## 개요

## 기본 아이디어
LLaMA Factory는 대형 언어 모델의 훈련 및 미세 조정 프로세스를 단순화하는 오픈 소스 프레임워크입니다. SFT, RLHF 및 QLoRA 기술과 같은 다양한 최첨단 방법에 대한 통합 인터페이스를 제공합니다. 또한 LLaMA, Mistral 및 Qwen과 같은 광범위한 LLM 아키텍처를 지원합니다. 이 플레이북은 NVIDIA Spark 장치에서 LLaMA Factory CLI를 사용하여 대형 언어 모델을 미세 조정하는 방법을 보여줍니다.

## 달성할 내용

Blackwell 아키텍처가 있는 NVIDIA Spark에 LLaMA Factory를 설정하여 LoRA, QLoRA 및 전체 미세 조정 방법을 사용하여 대형 언어 모델을 미세 조정합니다. 이를 통해 하드웨어별 최적화를 활용하면서 전문 도메인에 대한 효율적인 모델 적응이 가능합니다.

## 시작하기 전에 알아야 할 사항

- 구성 파일 편집 및 문제 해결을 위한 기본 Python 지식
- 셸 명령 실행 및 환경 관리를 위한 명령줄 사용
- PyTorch 및 Hugging Face Transformers 에코시스템에 대한 익숙함
- CUDA/cuDNN 설치 및 VRAM 관리를 포함한 GPU 환경 설정
- 미세 조정 개념: LoRA, QLoRA 및 전체 미세 조정 간의 트레이드오프 이해
- 데이터 세트 준비: 지시 조정을 위해 텍스트 데이터를 JSON 구조로 형식화
- 리소스 관리: GPU 제약 조건에 맞게 배치 크기 및 메모리 설정 조정

## 전제 조건

- Blackwell 아키텍처가 있는 NVIDIA Spark 장치

- CUDA 12.9 이상 버전 설치됨: `nvcc --version`

- GPU 액세스를 위해 Docker가 설치되고 구성됨: `docker run --gpus all nvcr.io/nvidia/pytorch:25.11-py3 nvidia-smi`

- Git 설치됨: `git --version`

- pip가 있는 Python 환경: `python --version && pip --version`

- 충분한 저장 공간(모델 및 체크포인트용 >50GB): `df -h`

- Hugging Face Hub에서 모델을 다운로드하기 위한 인터넷 연결

## 부속 파일

- 공식 LLaMA Factory 저장소: https://github.com/hiyouga/LLaMA-Factory

- NVIDIA PyTorch 컨테이너: https://catalog.ngc.nvidia.com/orgs/nvidia/containers/pytorch

- 예제 훈련 구성: `examples/train_lora/llama3_lora_sft.yaml`(저장소에서)

- 문서: https://llamafactory.readthedocs.io/en/latest/getting_started/data_preparation.html

## 시간 및 위험

* **소요 시간:** 초기 설정 30-60분, 모델 크기 및 데이터 세트에 따라 훈련 1-7시간.
* **위험:** 모델 다운로드에는 상당한 대역폭과 저장 공간이 필요합니다. 훈련은 상당한 GPU 메모리를 소비할 수 있으며 하드웨어 제약 조건에 대한 매개변수 조정이 필요할 수 있습니다.
* **롤백:** Docker 컨테이너 및 복제된 저장소를 제거합니다. 훈련 체크포인트는 로컬에 저장되며 저장 공간을 회수하기 위해 삭제할 수 있습니다.
* **최종 업데이트:** 12/15/2025
  * 최신 pytorch 컨테이너 버전 nvcr.io/nvidia/pytorch:25.11-py3으로 업그레이드

## 지침

## 1단계. 시스템 전제 조건 확인

NVIDIA Spark 시스템에 필요한 구성 요소가 설치되어 있고 액세스할 수 있는지 확인하십시오.

```bash
nvcc --version
docker --version
nvidia-smi
python --version
git --version
```

## 2단계. GPU 지원으로 PyTorch 컨테이너 시작

GPU 액세스 및 작업 공간 디렉토리 마운트와 함께 NVIDIA PyTorch 컨테이너를 시작하십시오.
> [!NOTE]
> 이 NVIDIA PyTorch 컨테이너는 CUDA 13을 지원합니다

```bash
docker run --gpus all --ipc=host --ulimit memlock=-1 -it --ulimit stack=67108864 --rm -v "$PWD":/workspace nvcr.io/nvidia/pytorch:25.11-py3 bash
```

## 3단계. LLaMA Factory 저장소 복제

공식 저장소에서 LLaMA Factory 소스 코드를 다운로드하십시오.

```bash
git clone --depth 1 https://github.com/hiyouga/LLaMA-Factory.git
cd LLaMA-Factory
```

### 4단계. 종속성과 함께 LLaMA Factory 설치

훈련 평가를 위한 메트릭 지원과 함께 패키지를 편집 가능 모드로 설치하십시오.

```bash
pip install -e ".[metrics]"
```

## 5단계. Pytorch CUDA 지원 확인.

PyTorch는 CUDA 지원과 함께 사전 설치되어 있습니다.

설치를 확인하려면:

```bash
python -c "import torch; print(f'PyTorch: {torch.__version__}, CUDA: {torch.cuda.is_available()}')"
```

## 6단계. 훈련 구성 준비

Llama-3에 대한 제공된 LoRA 미세 조정 구성을 검토하십시오.

```bash
cat examples/train_lora/llama3_lora_sft.yaml
```

## 7단계. 미세 조정 훈련 시작

> [!NOTE]
> 모델이 게이트되어 있는 경우 모델을 다운로드하려면 hugging face hub에 로그인하십시오.

사전 구성된 LoRA 설정을 사용하여 훈련 프로세스를 실행하십시오.

```bash
huggingface-cli login # 모델이 게이트되어 있는 경우
llamafactory-cli train examples/train_lora/llama3_lora_sft.yaml
```

예제 출력:
```bash
***** train metrics *****
  epoch                    =        3.0
  total_flos               = 22851591GF
  train_loss               =     0.9113
  train_runtime            = 0:22:21.99
  train_samples_per_second =      2.437
  train_steps_per_second   =      0.306
Figure saved at: saves/llama3-8b/lora/sft/training_loss.png
```

## 8단계. 훈련 완료 확인

훈련이 성공적으로 완료되고 체크포인트가 저장되었는지 확인하십시오.

```bash
ls -la saves/llama3-8b/lora/sft/
```


예상 출력은 다음을 표시해야 합니다:
- 최종 체크포인트 디렉토리(`checkpoint-21` 또는 유사)
- 모델 구성 파일(`config.json`, `adapter_config.json`)
- 손실 값이 감소하는 훈련 메트릭
- PNG 파일로 저장된 훈련 손실 플롯

## 9단계. 미세 조정된 모델로 추론 테스트

사용자 정의 프롬프트로 미세 조정된 모델을 테스트하십시오:

```bash
llamafactory-cli chat examples/inference/llama3_lora_sft.yaml
## Type: "Hello, how can you help me today?"
## Expect: Response showing fine-tuned behavior
```

## 10단계. 프로덕션 배포를 위해 모델 내보내기
```bash
llamafactory-cli export examples/merge_lora/llama3_lora_sft.yaml
```

## 11단계. 정리 및 롤백

> [!WARNING]
> 이렇게 하면 모든 훈련 진행 상황과 체크포인트가 삭제됩니다.

생성된 모든 파일을 제거하고 저장 공간을 확보하려면:

```bash
cd /workspace
rm -rf LLaMA-Factory/
docker system prune -f
```

Docker 컨테이너 변경 사항을 롤백하려면:
```bash
exit  # Exit container
docker container prune -f
```

## 문제 해결

| 증상 | 원인 | 해결 방법 |
|---------|--------|-----|
| 훈련 중 CUDA 메모리 부족 | GPU VRAM에 비해 배치 크기가 너무 큼 | `per_device_train_batch_size`를 줄이거나 `gradient_accumulation_steps`를 늘립니다 |
| URL에 대한 게이트된 리포지토리에 액세스할 수 없음 | 특정 HuggingFace 모델에 제한된 액세스가 있음 | [HuggingFace 토큰](https://huggingface.co/docs/hub/en/security-tokens)을 재생성하고 웹 브라우저에서 [게이트된 모델](https://huggingface.co/docs/hub/en/models-gated#customize-requested-information)에 대한 액세스 요청 |
| 모델 다운로드 실패 또는 느림 | 네트워크 연결 또는 Hugging Face Hub 문제 | 인터넷 연결을 확인하고 캐시된 모델에 대해 `HF_HUB_OFFLINE=1` 사용을 시도하십시오 |
| 훈련 손실이 감소하지 않음 | 학습률이 너무 높음/낮음 또는 데이터 부족 | `learning_rate` 매개변수를 조정하거나 데이터 세트 품질을 확인하십시오 |

> [!NOTE]
> DGX Spark는 GPU와 CPU 간의 동적 메모리 공유를 가능하게 하는 통합 메모리 아키텍처(UMA)를 사용합니다.
> UMA를 활용하도록 업데이트되는 많은 애플리케이션에서 여전히 DGX Spark의 메모리 용량 내에서도
> 메모리 문제가 발생할 수 있습니다. 그런 경우 다음과 같이 버퍼 캐시를 수동으로 플러시하십시오:
```bash
sudo sh -c 'sync; echo 3 > /proc/sys/vm/drop_caches'
```
