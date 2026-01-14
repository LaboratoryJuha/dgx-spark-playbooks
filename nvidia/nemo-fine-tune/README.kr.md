# NeMo로 미세 조정하기

> NVIDIA NeMo를 사용하여 로컬에서 모델 미세 조정하기

## 목차

- [개요](#개요)
- [지침](#지침)
- [문제 해결](#문제-해결)

---

## 개요

## 기본 아이디어

이 플레이북은 NVIDIA Spark 장치에서 대규모 언어 모델 및 비전-언어 모델을 미세 조정하기 위한 NVIDIA NeMo AutoModel 설정 및 사용 방법을 안내합니다. NeMo AutoModel은 네이티브 PyTorch 지원과 함께 Hugging Face 모델에 대한 GPU 가속 엔드투엔드 학습을 제공하여 변환 지연 없이 즉시 미세 조정을 가능하게 합니다. 이 프레임워크는 단일 GPU에서 멀티 노드 클러스터까지의 분산 학습을 지원하며, ARM64 아키텍처와 Blackwell GPU 시스템을 위해 특별히 설계된 최적화된 커널과 메모리 효율적인 레시피를 제공합니다.

## 달성할 내용

NVIDIA Spark 장치에서 NeMo AutoModel을 사용하여 대규모 언어 모델(1-70B 파라미터)과 비전-언어 모델을 위한 완전한 미세 조정 환경을 구축합니다. 완료 시, Hugging Face 에코시스템과의 호환성을 유지하면서 파라미터 효율적 미세 조정(PEFT), 지도 미세 조정(SFT), FP8 정밀도 최적화가 적용된 분산 학습 기능을 지원하는 작동 가능한 설치 환경을 갖추게 됩니다.

## 시작하기 전에 알아야 할 사항

- Linux 터미널 환경 및 SSH 연결 작업
- Python 가상 환경 및 패키지 관리에 대한 기본 이해
- GPU 컴퓨팅 개념 및 CUDA 툴킷 사용에 대한 친숙함
- 컨테이너화된 워크플로우 및 Docker/Podman 작업 경험
- 머신 러닝 모델 학습 개념 및 미세 조정 워크플로우에 대한 이해

## 전제 조건

- Blackwell 아키텍처 GPU 액세스가 가능한 NVIDIA Spark 장치
- CUDA 툴킷 12.0+ 설치 및 구성됨: `nvcc --version`
- Python 3.10+ 환경 사용 가능: `python3 --version`
- 효율적인 모델 로딩 및 학습을 위한 최소 32GB 시스템 RAM
- 모델 및 패키지 다운로드를 위한 활성 인터넷 연결
- 리포지토리 클론을 위한 Git 설치: `git --version`
- NVIDIA Spark 장치에 대한 SSH 액세스 구성됨

## 부속 파일

플레이북에 필요한 모든 파일은 [GitHub에서](https://github.com/NVIDIA-NeMo/Automodel) 찾을 수 있습니다

## 시간 및 위험

* **소요 시간:** 완전한 설정 및 초기 모델 미세 조정에 45-90분
* **위험:** 모델 다운로드 크기가 클 수 있음(수 GB), ARM64 패키지 호환성 문제로 인해 문제 해결이 필요할 수 있음, 멀티 노드 구성으로 인해 분산 학습 설정 복잡도가 증가함
* **롤백:** 가상 환경을 완전히 제거할 수 있음; 패키지 설치 외에 호스트 시스템에 대한 시스템 레벨 변경은 없음
* **마지막 업데이트:** 12/22/2025
  * 최신 pytorch 컨테이너 버전 nvcr.io/nvidia/pytorch:25.11-py3로 업그레이드
  * docker 컨테이너 권한 설정 지침 추가

## 지침

## 1단계. 시스템 요구 사항 확인

NVIDIA Spark 장치가 [NeMo AutoModel](https://github.com/NVIDIA-NeMo/Automodel) 설치를 위한 전제 조건을 충족하는지 확인합니다. 이 단계는 호스트 시스템에서 실행되어 CUDA 툴킷 가용성 및 Python 버전 호환성을 확인합니다.

```bash
## CUDA 설치 확인
nvcc --version

## Python 버전 확인 (3.10+ 필요)
python3 --version

## GPU 접근성 확인
nvidia-smi

## 사용 가능한 시스템 메모리 확인
free -h
```

## 2단계. Docker 권한 구성

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

## 3단계. 컨테이너 이미지 가져오기

```bash
docker pull nvcr.io/nvidia/pytorch:25.11-py3
```

권한 거부 오류가 표시되면(Docker 데몬 소켓에 연결하는 동안 권한 거부와 같은 메시지), sudo 없이 명령을 실행할 수 있도록 사용자를 docker 그룹에 추가합니다.

```bash
sudo usermod -aG docker $USER
newgrp docker
```

## 4단계. Docker 시작

```bash
docker run \
  --gpus all \
  --ulimit memlock=-1 \
  -it --ulimit stack=67108864 \
  --entrypoint /usr/bin/bash \
  --rm nvcr.io/nvidia/pytorch:25.11-py3
```

## 5단계. 패키지 관리 도구 설치

효율적인 패키지 관리 및 가상 환경 격리를 위해 `uv`를 설치합니다. NeMo AutoModel은 종속성 관리 및 자동 환경 처리를 위해 `uv`를 사용합니다.

```bash
## uv 패키지 매니저 설치
pip3 install uv

## 설치 확인
uv --version
```

**시스템 설치가 실패하는 경우:**

```bash
## 현재 사용자만을 위해 설치
pip3 install --user uv

## 필요한 경우 PATH에 추가
export PATH="$HOME/.local/bin:$PATH"
```

## 6단계. NeMo AutoModel 리포지토리 클론

레시피와 예제에 액세스하기 위해 공식 NeMo AutoModel 리포지토리를 클론합니다. 이를 통해 다양한 모델 유형 및 학습 시나리오에 대한 즉시 사용 가능한 학습 구성을 제공합니다.

```bash
## 리포지토리 클론
git clone https://github.com/NVIDIA-NeMo/Automodel.git

## 리포지토리로 이동
cd Automodel
```

## 7단계. NeMo AutoModel 설치

가상 환경을 설정하고 NeMo AutoModel을 설치합니다. 안정성을 위해 휠 패키지 설치 또는 최신 기능을 위해 소스 설치 중에서 선택합니다.

**휠 패키지에서 설치(권장):**

```bash
## 가상 환경 초기화
uv venv --system-site-packages

## uv로 패키지 설치
uv sync --inexact --frozen --all-extras \
  --no-install-package torch \
  --no-install-package torchvision \
  --no-install-package triton \
  --no-install-package nvidia-cublas-cu12 \
  --no-install-package nvidia-cuda-cupti-cu12 \
  --no-install-package nvidia-cuda-nvrtc-cu12 \
  --no-install-package nvidia-cuda-runtime-cu12 \
  --no-install-package nvidia-cudnn-cu12 \
  --no-install-package nvidia-cufft-cu12 \
  --no-install-package nvidia-cufile-cu12 \
  --no-install-package nvidia-curand-cu12 \
  --no-install-package nvidia-cusolver-cu12 \
  --no-install-package nvidia-cusparse-cu12 \
  --no-install-package nvidia-cusparselt-cu12 \
  --no-install-package nvidia-nccl-cu12 \
  --no-install-package transformer-engine \
  --no-install-package nvidia-modelopt \
  --no-install-package nvidia-modelopt-core \
  --no-install-package flash-attn \
  --no-install-package transformer-engine-cu12 \
  --no-install-package transformer-engine-torch

## bitsandbytes 설치
CMAKE_ARGS="-DCOMPUTE_BACKEND=cuda -DCOMPUTE_CAPABILITY=80;86;87;89;90" \
CMAKE_BUILD_PARALLEL_LEVEL=8 \
uv pip install --no-deps git+https://github.com/bitsandbytes-foundation/bitsandbytes.git@50be19c39698e038a1604daf3e1b939c9ac1c342
```

## 8단계. 설치 확인

NeMo AutoModel이 올바르게 설치되고 액세스 가능한지 확인합니다. 이 단계는 설치를 검증하고 누락된 종속성을 확인합니다.

```bash
## NeMo AutoModel 가져오기 테스트
uv run --frozen --no-sync python -c "import nemo_automodel; print('✅ NeMo AutoModel 준비 완료')"

## 사용 가능한 예제 확인
ls -la examples/

## 다음은 예상 출력의 예시입니다(username 및 domain-users는 플레이스홀더입니다).
## $ ls -la examples/
## total 36
## drwxr-xr-x  9 username domain-users 4096 Oct 16 14:52 .
## drwxr-xr-x 16 username domain-users 4096 Oct 16 14:52 ..
## drwxr-xr-x  3 username domain-users 4096 Oct 16 14:52 benchmark
## drwxr-xr-x  3 username domain-users 4096 Oct 16 14:52 diffusion
## drwxr-xr-x 20 username domain-users 4096 Oct 16 14:52 llm_finetune
## drwxr-xr-x  3 username domain-users 4096 Oct 14 09:27 llm_kd
## drwxr-xr-x  2 username domain-users 4096 Oct 16 14:52 llm_pretrain
## drwxr-xr-x  6 username domain-users 4096 Oct 14 09:27 vlm_finetune
## drwxr-xr-x  2 username domain-users 4096 Oct 14 09:27 vlm_generate
```

## 9단계. 사용 가능한 예제 탐색

다양한 모델 유형 및 학습 시나리오에 대해 사용 가능한 사전 구성된 학습 레시피를 검토합니다. 이러한 레시피는 ARM64 및 Blackwell 아키텍처에 최적화된 구성을 제공합니다.

```bash
## LLM 미세 조정 예제 목록 보기
ls examples/llm_finetune/

## 예제 레시피 구성 보기
cat examples/llm_finetune/finetune.py | head -20
```

## 10단계. 샘플 미세 조정 실행
다음 명령은 LoRA 및 QLoRA를 사용한 전체 미세 조정(SFT), 파라미터 효율적 미세 조정(PEFT)을 수행하는 방법을 보여줍니다.

먼저, 게이트된 모델을 다운로드할 수 있도록 HF_TOKEN을 내보냅니다.

```bash
## 기본 LLM 미세 조정 예제 실행
export HF_TOKEN=<your_huggingface_token>
```
> [!NOTE]
> `<your_huggingface_token>`을 귀하의 개인 Hugging Face 액세스 토큰으로 교체하세요. 게이트된 모델을 다운로드하려면 유효한 토큰이 필요합니다.
>
> - 토큰 생성: [Hugging Face 토큰](https://huggingface.co/settings/tokens), 가이드는 [여기](https://huggingface.co/docs/hub/en/security-tokens)에서 확인할 수 있습니다.
> - 다운로드를 시도하기 전에 각 모델의 페이지에서 액세스를 요청하고 받아야 합니다(라이선스/약관에 동의).
>   - Llama-3.1-8B: [meta-llama/Llama-3.1-8B](https://huggingface.co/meta-llama/Llama-3.1-8B)
>   - Qwen3-8B: [Qwen/Qwen3-8B](https://huggingface.co/Qwen/Qwen3-8B)
>   - Mixtral-8x7B: [mistralai/Mixtral-8x7B](https://huggingface.co/mistralai/Mixtral-8x7B)
>
> 사용하는 다른 게이트된 모델에도 동일한 단계가 적용됩니다: Hugging Face에서 모델 카드를 방문하고, 액세스를 요청하고, 라이선스에 동의하고, 승인을 기다리세요.

**LoRA 미세 조정 예제:**

완전한 설정을 검증하기 위해 기본 미세 조정 예제를 실행합니다. 이는 테스트에 적합한 작은 모델을 사용한 파라미터 효율적 미세 조정을 보여줍니다.
아래 예제에서는 구성에 YAML을 사용하고 파라미터 재정의는 명령줄 인수로 전달됩니다.

```bash
## 기본 LLM 미세 조정 예제 실행
uv run --frozen --no-sync \
examples/llm_finetune/finetune.py \
-c examples/llm_finetune/llama3_2/llama3_2_1b_squad_peft.yaml \
--model.pretrained_model_name_or_path meta-llama/Llama-3.1-8B \
--packed_sequence.packed_sequence_size 1024 \
--step_scheduler.max_steps 100
```

이러한 재정의는 Llama-3.1-8B LoRA 실행이 예상대로 작동하도록 합니다:
- `--model.pretrained_model_name_or_path`: Hugging Face 모델 허브에서 미세 조정할 Llama-3.1-8B 모델 선택(가중치는 Hugging Face 토큰을 통해 가져옴).
- `--packed_sequence.packed_sequence_size`: 패킹된 시퀀스 학습을 활성화하기 위해 패킹된 시퀀스 크기를 1024로 설정.
- `--step_scheduler.max_steps`: 최대 학습 단계 수 설정. 데모 목적으로 100으로 설정했으며, 필요에 따라 조정하세요.


**QLoRA 미세 조정 예제:**

QLoRA를 사용하여 메모리 효율적인 방식으로 대형 모델을 미세 조정할 수 있습니다.

```bash
uv run --frozen --no-sync \
examples/llm_finetune/finetune.py \
-c examples/llm_finetune/llama3_1/llama3_1_8b_squad_qlora.yaml \
--model.pretrained_model_name_or_path meta-llama/Meta-Llama-3-70B \
--loss_fn._target_ nemo_automodel.components.loss.te_parallel_ce.TEParallelCrossEntropy \
--step_scheduler.local_batch_size 1 \
--packed_sequence.packed_sequence_size 1024 \
--step_scheduler.max_steps 100
```

이러한 재정의는 70B QLoRA 실행이 예상대로 작동하도록 합니다:
- `--model.pretrained_model_name_or_path`: 미세 조정할 70B 기본 모델 선택(가중치는 Hugging Face 토큰을 통해 가져옴).
- `--loss_fn._target_`: 대형 LLM의 텐서 병렬 학습과 호환되는 TransformerEngine-parallel 교차 엔트로피 손실 변형 사용.
- `--step_scheduler.local_batch_size`: GPU당 마이크로 배치 크기를 1로 설정하여 70B를 메모리에 맞춤; 전체 유효 배치 크기는 여전히 레시피의 그래디언트 누적 및 데이터/텐서 병렬 설정에 의해 결정됨.
- `--step_scheduler.max_steps`: 최대 학습 단계 수 설정. 데모 목적으로 100으로 설정했으며, 필요에 따라 조정하세요.
- `--packed_sequence.packed_sequence_size`: 패킹된 시퀀스 학습을 활성화하기 위해 패킹된 시퀀스 크기를 1024로 설정.

**전체 미세 조정 예제:**

GitHub에서 클론한 `Automodel` 디렉토리 내부에서 다음을 실행합니다:

```bash
uv run --frozen --no-sync \
examples/llm_finetune/finetune.py \
-c examples/llm_finetune/qwen/qwen3_8b_squad_spark.yaml \
--model.pretrained_model_name_or_path Qwen/Qwen3-8B \
--step_scheduler.local_batch_size 1 \
--step_scheduler.max_steps 100 \
--packed_sequence.packed_sequence_size 1024
```
이러한 재정의는 Qwen3-8B SFT 실행이 예상대로 작동하도록 합니다:
- `--model.pretrained_model_name_or_path`: Hugging Face 모델 허브에서 미세 조정할 Qwen/Qwen3-8B 모델 선택(가중치는 Hugging Face 토큰을 통해 가져옴). 다른 모델을 미세 조정하려면 이를 조정하세요.
- `--step_scheduler.max_steps`: 최대 학습 단계 수 설정. 데모 목적으로 100으로 설정했으며, 필요에 따라 조정하세요.
- `--step_scheduler.local_batch_size`: GPU당 마이크로 배치 크기를 1로 설정하여 메모리에 맞춤; 전체 유효 배치 크기는 여전히 레시피의 그래디언트 누적 및 데이터/텐서 병렬 설정에 의해 결정됨.


## 11단계. 성공적인 학습 완료 확인

체크포인트 디렉토리에 포함된 아티팩트를 검사하여 미세 조정된 모델을 검증합니다.

```bash
## 로그 및 체크포인트 출력 검사.
## LATEST는 최신 체크포인트를 가리키는 심볼릭 링크입니다.
## 체크포인트는 학습 중에 저장된 것입니다.
## 아래는 예상 출력의 예시입니다(username 및 domain-users는 플레이스홀더입니다).
ls -lah checkpoints/LATEST/

## $ ls -lah checkpoints/LATEST/
## total 32K
## drwxr-xr-x 6 username domain-users 4.0K Oct 16 22:33 .
## drwxr-xr-x 4 username domain-users 4.0K Oct 16 22:33 ..
## -rw-r--r-- 1 username domain-users 1.6K Oct 16 22:33 config.yaml
## drwxr-xr-x 2 username domain-users 4.0K Oct 16 22:33 dataloader
## drwxr-xr-x 2 username domain-users 4.0K Oct 16 22:33 model
## drwxr-xr-x 2 username domain-users 4.0K Oct 16 22:33 optim
## drwxr-xr-x 2 username domain-users 4.0K Oct 16 22:33 rng
## -rw-r--r-- 1 username domain-users 1.3K Oct 16 22:33 step_scheduler.pt
```

## 12단계. 정리 및 롤백(선택 사항)

필요한 경우 설치를 제거하고 원래 환경을 복원합니다. 이러한 명령은 설치된 모든 구성 요소를 안전하게 제거합니다.

> [!WARNING]
> 이렇게 하면 모든 가상 환경과 다운로드된 모델이 삭제됩니다. 중요한 학습 체크포인트를 백업했는지 확인하세요.

```bash
## 가상 환경 제거
rm -rf .venv

## 클론된 리포지토리 제거
cd ..
rm -rf Automodel

## uv 제거(--user로 설치한 경우)
pip3 uninstall uv

## Python 캐시 지우기
rm -rf ~/.cache/pip
```
## 13단계. 선택 사항: Hugging Face Hub에 미세 조정된 모델 체크포인트 게시

Hugging Face Hub에 미세 조정된 모델 체크포인트를 게시합니다.
> [!NOTE]
> 이것은 선택 사항이며 미세 조정된 모델을 사용하는 데 필요하지 않습니다.
> 다른 사람과 미세 조정된 모델을 공유하거나 다른 프로젝트에서 사용하려는 경우 유용합니다.
> 리포지토리를 클론하고 체크포인트를 사용하여 다른 프로젝트에서 미세 조정된 모델을 사용할 수도 있습니다.
> 다른 프로젝트에서 미세 조정된 모델을 사용하려면 Hugging Face CLI가 설치되어 있어야 합니다.
> `pip install huggingface-cli`를 실행하여 Hugging Face CLI를 설치할 수 있습니다.
> 자세한 내용은 [Hugging Face CLI 문서](https://huggingface.co/docs/huggingface_hub/en/guides/cli)를 참조하세요.

> [!TIP]
> `hf` 명령을 사용하여 미세 조정된 모델 체크포인트를 Hugging Face Hub에 업로드할 수 있습니다.
> 자세한 내용은 [Hugging Face CLI 문서](https://huggingface.co/docs/huggingface_hub/en/guides/cli)를 참조하세요.

```bash
## 미세 조정된 모델 체크포인트를 Hugging Face Hub에 게시
## <your_huggingface_username>/my-cool-model 네임스페이스 아래에 게시되며, 필요에 따라 이름을 조정하세요.
hf upload my-cool-model checkpoints/LATEST/model
```

> [!TIP]
> 위 명령은 사용한 HF_TOKEN에 Hugging Face Hub에 대한 쓰기 권한이 없으면 실패할 수 있습니다.
> 샘플 오류 메시지:
> ```bash
> akoumparouli@1604ab7-lcedt:/mnt/4tb/auto/Automodel8$ hf upload my-cool-model checkpoints/LATEST/model
> Traceback (most recent call last):
>   File "/home/akoumparouli/.local/lib/python3.10/site-packages/huggingface_hub/utils/_http.py", line 409, in hf_raise_for_status
>     response.raise_for_status()
>   File "/home/akoumparouli/.local/lib/python3.10/site-packages/requests/models.py", line 1024, in raise_for_status
>     raise HTTPError(http_error_msg, response=self)
> requests.exceptions.HTTPError: 403 Client Error: Forbidden for url: https://huggingface.co/api/repos/create
> ```
> 이를 해결하려면 *쓰기* 권한이 있는 액세스 토큰을 만들어야 합니다. 지침은 Hugging Face 가이드 [여기](https://huggingface.co/docs/hub/en/security-tokens)를 참조하세요.

## 14단계. 다음 단계

특정 미세 조정 작업을 위해 NeMo AutoModel 사용을 시작합니다. 제공된 레시피로 시작하고 모델 요구 사항 및 데이터셋에 따라 사용자 정의합니다.

```bash
## 사용자 정의를 위해 레시피 복사
cp recipes/llm_finetune/finetune.py my_custom_training.py

## 특정 모델 및 데이터에 대한 구성 편집
## 그런 다음 실행: uv run my_custom_training.py
```

더 많은 레시피, 문서 및 커뮤니티 예제는 [NeMo AutoModel GitHub 리포지토리](https://github.com/NVIDIA-NeMo/Automodel)를 탐색하세요. 사용자 정의 데이터셋 설정, 다양한 모델 아키텍처 실험, 더 큰 모델을 위한 멀티 노드 분산 학습으로 확장하는 것을 고려하세요.

## 문제 해결

| 증상 | 원인 | 해결 방법 |
|---------|--------|-----|
| `nvcc: command not found` | CUDA 툴킷이 PATH에 없음 | CUDA 툴킷을 PATH에 추가: `export PATH=/usr/local/cuda/bin:$PATH` |
| `pip install uv` 권한 거부 | 시스템 레벨 pip 제한 | `pip3 install --user uv`를 사용하고 PATH 업데이트 |
| 학습 중 GPU가 감지되지 않음 | CUDA 드라이버/런타임 불일치 | 드라이버 호환성 확인: `nvidia-smi` 및 필요한 경우 CUDA 재설치 |
| 학습 중 메모리 부족 | 사용 가능한 GPU 메모리에 비해 모델이 너무 큼 | 배치 크기 줄이기, 그래디언트 체크포인팅 활성화 또는 모델 병렬 처리 사용 |
| ARM64 패키지 호환성 문제 | ARM 아키텍처에 패키지를 사용할 수 없음 | 소스 설치 사용 또는 ARM64 플래그로 소스에서 빌드 |
| URL에 대한 게이트된 리포지토리에 액세스할 수 없음 | 특정 HuggingFace 모델에는 제한된 액세스 권한이 있음 | [HuggingFace 토큰](https://huggingface.co/docs/hub/en/security-tokens)을 재생성하고 웹 브라우저에서 [게이트된 모델](https://huggingface.co/docs/hub/en/models-gated#customize-requested-information)에 대한 액세스를 요청하세요 |

> [!NOTE]
> DGX Spark는 GPU와 CPU 간에 동적 메모리 공유를 가능하게 하는 통합 메모리 아키텍처(UMA)를 사용합니다.
> 많은 애플리케이션이 여전히 UMA를 활용하도록 업데이트 중이므로 DGX Spark의 메모리 용량 내에 있더라도
> 메모리 문제가 발생할 수 있습니다. 이런 경우 다음과 같이 버퍼 캐시를 수동으로 플러시하세요:
```bash
sudo sh -c 'sync; echo 3 > /proc/sys/vm/drop_caches'
```
