# DGX Spark에서 Unsloth

> Unsloth를 사용한 최적화된 파인튜닝

## 목차

- [개요](#개요)
- [지침](#지침)
- [문제 해결](#문제-해결)

---

## 개요

## 기본 개념

- **성능 우선**: 표준 방법에 비해 훈련 속도가 빨라지고 (예: 단일 GPU에서 2배, 다중 GPU 설정에서 최대 30배) 메모리 사용량이 줄어듭니다.
- **커널 수준 최적화**: 핵심 계산은 사용자 정의 커널(예: Triton 사용)과 수작업으로 최적화된 수학으로 구축되어 처리량과 효율성을 높입니다.
- **양자화 및 모델 형식**: 동적 양자화(4비트, 16비트) 및 GGUF 형식을 지원하여 풋프린트를 줄이면서 정확도를 유지하는 것을 목표로 합니다.
- **광범위한 모델 지원**: 많은 LLM(LLaMA, Mistral, Qwen, DeepSeek 등)과 함께 작동하며 훈련, 파인튜닝, Ollama, vLLM, GGUF, Hugging Face와 같은 형식으로 내보내기를 허용합니다.
- **간소화된 인터페이스**: 최소한의 상용구로 모델을 파인튜닝할 수 있도록 사용하기 쉬운 노트북과 도구를 제공합니다.

## 달성할 목표

NVIDIA Spark 장치에서 대규모 언어 모델의 최적화된 파인튜닝을 위해 Unsloth를 설정하여 LoRA 및 QLoRA와 같은 효율적인 매개변수 효율적 파인튜닝 방법을 통해 메모리 사용량을 줄이면서 최대 2배 빠른 훈련 속도를 달성합니다.

## 시작하기 전에 알아야 할 사항

- pip 및 가상 환경을 사용한 Python 패키지 관리
- Hugging Face Transformers 라이브러리 기본 사항 (모델, 토크나이저, 데이터셋 로딩)
- GPU 기본 사항 (CUDA/GPU 대 CPU, VRAM 제약, 장치 가용성)
- LLM 훈련 개념에 대한 기본 이해 (손실 함수, 체크포인트)
- 프롬프트 엔지니어링 및 기본 모델 상호 작용에 대한 익숙함
- 선택 사항: LoRA/QLoRA 매개변수 효율적 파인튜닝 지식

## 필수 사항

- Blackwell GPU 아키텍처를 갖춘 NVIDIA Spark 장치
- `nvidia-smi`가 GPU 정보 요약을 표시
- CUDA 13.0 설치: `nvcc --version`
- 모델 및 데이터셋 다운로드를 위한 인터넷 액세스

## 보조 파일

Python 테스트 스크립트는 [여기 GitHub에서](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main/nvidia/unsloth/assets/test_unsloth.py) 찾을 수 있습니다


## 시간 및 위험도

* **기간**: 초기 설정 및 테스트 실행에 30-60분
* **위험**:
  * Triton 컴파일러 버전 불일치로 인한 컴파일 오류 발생 가능
  * CUDA 툴킷 구성 문제로 인해 커널 컴파일이 방지될 수 있음
  * 더 작은 모델의 메모리 제약으로 배치 크기 조정 필요
* **롤백**: `pip uninstall unsloth torch torchvision`으로 패키지 제거.
* **마지막 업데이트:** 12/15/2025
  * PyTorch 컨테이너 및 Python 종속성을 최신 버전으로 업그레이드

## 지침

## 1단계. 필수 사항 확인

NVIDIA Spark 장치에 필요한 CUDA 툴킷 및 GPU 리소스가 있는지 확인합니다.

```bash
nvcc --version
```
출력은 CUDA 13.0을 표시해야 합니다.

```bash
nvidia-smi
```
출력은 GPU 정보 요약을 표시해야 합니다.

## 2단계. 컨테이너 이미지 가져오기
```bash
docker pull nvcr.io/nvidia/pytorch:25.11-py3
```

## 3단계. Docker 시작
```bash
docker run --gpus all --ulimit memlock=-1 -it --ulimit stack=67108864 --entrypoint /usr/bin/bash --rm nvcr.io/nvidia/pytorch:25.11-py3
```

## 4단계. Docker 내에서 종속성 설치

```bash
pip install transformers peft hf_transfer "datasets==4.3.0" "trl==0.26.1"
pip install --no-deps unsloth unsloth_zoo bitsandbytes
```

## 5단계. Python 테스트 스크립트 생성

[여기](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main/nvidia/unsloth/assets/test_unsloth.py)에서 테스트 스크립트를 컨테이너에 curl합니다.

```bash
curl -O https://raw.githubusercontent.com/NVIDIA/dgx-spark-playbooks/refs/heads/main/nvidia/unsloth/assets/test_unsloth.py
```

이 테스트 스크립트를 사용하여 간단한 파인튜닝 작업으로 설치를 검증합니다.


## 6단계. 검증 테스트 실행

테스트 스크립트를 실행하여 Unsloth가 올바르게 작동하는지 확인합니다.

```bash
python test_unsloth.py
```

터미널 창에서 예상 출력:
- "Unsloth: Will patch your computer to enable 2x faster free finetuning"
- 60단계 동안 손실이 감소하는 훈련 진행률 표시줄
- 완료를 보여주는 최종 훈련 메트릭

## 7단계. 다음 단계

`test_unsloth.py` 파일을 업데이트하여 자신의 모델 및 데이터셋으로 테스트합니다:

```python
## 32행을 모델 선택으로 교체
model_name = "unsloth/Meta-Llama-3.1-8B-bnb-4bit"

## 8행에서 사용자 정의 데이터셋 로드
dataset = load_dataset("your_dataset_name")

## 61행에서 훈련 매개변수 args 조정
per_device_train_batch_size = 4
max_steps = 1000
```

다음을 포함한 고급 사용 지침은 https://github.com/unslothai/unsloth/wiki를 방문하세요:
- [vLLM용 GGUF 형식으로 모델 저장](https://github.com/unslothai/unsloth/wiki#saving-to-gguf)
- [체크포인트에서 계속 훈련](https://github.com/unslothai/unsloth/wiki#loading-lora-adapters-for-continued-finetuning)
- [사용자 정의 챗 템플릿 사용](https://github.com/unslothai/unsloth/wiki#chat-templates)
- [평가 루프 실행](https://github.com/unslothai/unsloth/wiki#evaluation-loop---also-fixes-oom-or-crashing)

## 문제 해결

> [!NOTE]
> DGX Spark는 GPU와 CPU 간의 동적 메모리 공유를 가능하게 하는 통합 메모리 아키텍처(UMA)를 사용합니다.
> 많은 애플리케이션이 여전히 UMA를 활용하도록 업데이트 중이므로 DGX Spark의 메모리 용량 내에 있어도
> 메모리 문제가 발생할 수 있습니다. 이 경우 다음을 사용하여 수동으로 버퍼 캐시를 플러시하세요:
```bash
sudo sh -c 'sync; echo 3 > /proc/sys/vm/drop_caches'
```
