# FLUX.1 Dreambooth LoRA 미세 조정

> 사용자 정의 이미지 생성을 위해 Dreambooth LoRA를 사용하여 FLUX.1-dev 12B 모델 미세 조정

## 목차

- [개요](#개요)
- [지침](#지침)
- [문제 해결](#문제-해결)

---

## 개요

## 기본 아이디어

이 플레이북은 DGX Spark에서 사용자 정의 이미지 생성을 위해 다중 개념 Dreambooth LoRA(저순위 적응)를 사용하여 FLUX.1-dev 12B 모델을 미세 조정하는 방법을 보여줍니다.
128GB의 통합 메모리와 강력한 GPU 가속으로 DGX Spark는 Diffusion Transformer, CLIP Text Encoder, T5 Text Encoder 및 Autoencoder와 같은 여러 모델이 메모리에 로드된 이미지 생성 모델을 훈련하는 이상적인 환경을 제공합니다.

다중 개념 Dreambooth LoRA 미세 조정을 사용하면 FLUX.1에 새로운 개념, 캐릭터 및 스타일을 가르칠 수 있습니다. 훈련된 LoRA 가중치는 기존 ComfyUI 워크플로에 쉽게 통합할 수 있어 프로토타입 제작 및 실험에 적합합니다.
또한 이 플레이북은 DGX Spark가 메모리에 여러 모델을 로드할 뿐만 아니라 1024px 이상과 같은 고해상도 이미지를 훈련하고 생성할 수 있음을 보여줍니다.

## 달성할 내용

사용자 정의 개념으로 이미지를 생성할 수 있는 미세 조정된 FLUX.1 모델을 갖게 되며, ComfyUI 워크플로에 즉시 사용할 수 있습니다.
설정에는 다음이 포함됩니다:
- Dreambooth LoRA 기술을 사용한 FLUX.1-dev 모델 미세 조정
- 사용자 정의 개념("tjtoy" 장난감 및 "sparkgpu" GPU)에 대한 훈련
- 고해상도 1K 확산 훈련 및 추론
- 직관적인 시각적 워크플로를 위한 ComfyUI 통합
- 재현 가능한 환경을 위한 Docker 컨테이너화

## 전제 조건

-  DGX Spark 장치가 설정되어 있고 액세스 가능
-  DGX Spark GPU에서 실행 중인 다른 프로세스가 없음
-  모델 다운로드를 위한 충분한 디스크 공간
-  NVIDIA Docker 설치 및 구성됨


## 시간 및 위험

* **소요 시간**:
  * 초기 설정 모델 다운로드 시간 30-45분
  * dreambooth LoRA 훈련 1-2시간
* **위험:**
  * Docker 권한 문제로 인해 사용자 그룹 변경 및 세션 재시작이 필요할 수 있음
  * 레시피는 최상의 결과를 위해 하이퍼파라미터 조정 및 고품질 데이터 세트가 필요함
* **롤백**: Docker 컨테이너를 중지하고 제거하고 필요한 경우 다운로드한 모델을 삭제합니다.
* **최종 업데이트:** 11/07/2025
  * 사소한 수정

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

## 2단계. 저장소 복제

터미널에서 저장소를 복제하고 flux-finetuning 디렉토리로 이동하십시오.

```bash
git clone https://github.com/NVIDIA/dgx-spark-playbooks
```

## 3단계. 모델 다운로드

FLUX.1-dev 모델은 게이트되어 있으므로 액세스 권한을 부여받아야 합니다. [모델 카드](https://huggingface.co/black-forest-labs/FLUX.1-dev)로 이동하여 약관에 동의하고 체크포인트에 대한 액세스 권한을 얻으십시오.
`HF_TOKEN`이 아직 없는 경우 [여기](https://huggingface.co/docs/hub/en/security-tokens)의 지침에 따라 생성하십시오. 다음 명령에서 생성된 토큰을 교체하여 시스템을 인증하십시오.

```bash
export HF_TOKEN=<YOUR_HF_TOKEN>
cd dgx-spark-playbooks/nvidia/flux-finetuning/assets
sh download.sh
```
다운로드 스크립트는 인터넷 속도에 따라 약 30-45분이 걸릴 수 있습니다.

이미 미세 조정된 LoRA가 있는 경우 `models/loras` 안에 배치하십시오. 아직 없는 경우 자세한 내용은 `6단계. 훈련` 섹션으로 진행하십시오.

## 4단계. 기본 모델 추론

관심 있는 2가지 개념인 Toy Jensen과 DGX Spark에서 기본 FLUX.1 모델을 사용하여 이미지를 생성하는 것으로 시작하겠습니다.

```bash
## 추론 도커 이미지 빌드
docker build -f Dockerfile.inference -t flux-comfyui .

## ComfyUI 컨테이너 시작 (flux-finetuning/assets 안에 있는지 확인)
## `torchaudio`에 대한 가져오기 오류는 무시할 수 있습니다
sh launch_comfyui.sh
```
`http://localhost:8188`에서 ComfyUI에 액세스하여 기본 모델로 이미지를 생성합니다. 기존 템플릿을 선택하지 마십시오.

ComfyUI의 왼쪽 패널에서 워크플로 섹션을 찾으십시오(또는 `w` 누르기). 열면 두 개의 기존 워크플로가 로드되어 있어야 합니다. 기본 Flux 모델의 경우 `base_flux.json` 워크플로를 로드하겠습니다. json을 로드하면 ComfyUI가 워크플로를 로드하는 것을 볼 수 있습니다.

`CLIP Text Encode (Prompt)` 블록에 프롬프트를 제공하십시오. 예를 들어 `Toy Jensen holding a DGX Spark in a datacenter`를 사용하겠습니다. 고해상도 1024px 이미지를 만드는 것은 계산 집약적이므로 생성에 ~3분이 걸릴 것으로 예상할 수 있습니다.

기본 모델을 가지고 놀고 나면 2가지 가능한 다음 단계가 있습니다.
* 이미 `models/loras/` 안에 미세 조정된 LoRA가 있는 경우 `7단계. 미세 조정된 모델 추론` 섹션으로 건너뛰십시오.
* 사용자 정의 개념에 대한 LoRA를 훈련하려면 먼저 훈련을 진행하기 전에 ComfyUI 추론 컨테이너가 종료되었는지 확인하십시오. `Ctrl+C` 키를 눌러 종료할 수 있습니다.

> [!NOTE]
>  시스템에서 추가로 점유된 메모리를 지우려면 컨테이너 외부에서 ComfyUI 서버를 중단한 후 다음 명령을 실행하십시오.
```bash
sudo sh -c 'sync; echo 3 > /proc/sys/vm/drop_caches'
```

## 5단계. 데이터 세트 준비

FLUX.1-dev 12B 모델에서 Dreambooth LoRA 미세 조정을 수행할 데이터 세트를 준비하겠습니다.

이 플레이북에서는 이미 2가지 개념인 Toy Jensen과 DGX Spark의 데이터 세트를 준비했습니다. 이 데이터 세트는 Google 이미지를 통해 액세스할 수 있는 공개 자산 모음입니다. 이러한 개념으로 이미지를 생성하려면 `data.toml` 파일을 수정할 필요가 없습니다.

**TJToy 개념**
- **트리거 문구**: `tjtoy toy`
- **훈련 이미지**: 공개 도메인에서 사용 가능한 Toy Jensen 피규어의 고품질 이미지 6장
- **사용 사례**: 다양한 장면에서 특정 장난감 캐릭터가 등장하는 이미지 생성

**SparkGPU 개념**
- **트리거 문구**: `sparkgpu gpu`
- **훈련 이미지**: 공개 도메인에서 사용 가능한 DGX Spark GPU 이미지 7장
- **사용 사례**: 다양한 컨텍스트에서 특정 GPU 디자인이 등장하는 이미지 생성

사용자 정의 개념으로 이미지를 생성하려면 생성하려는 모든 개념의 데이터 세트와 각 개념에 대해 약 5-10개의 이미지를 준비해야 합니다.

각 개념에 해당하는 이름으로 폴더를 만들고 `flux_data` 디렉토리 안에 배치하십시오. 우리의 경우 `sparkgpu`와 `tjtoy`를 개념으로 사용하고 각각에 몇 개의 이미지를 배치했습니다.

이제 선택한 개념을 반영하도록 `flux_data/data.toml` 파일을 수정하겠습니다. `[[datasets.subsets]]` 아래의 `image_dir` 및 `class_tokens` 필드를 수정하여 각 개념에 대한 항목을 업데이트/생성해야 합니다. 미세 조정의 성능을 향상시키려면 개념 이름에 클래스 토큰(`toy` 또는 `gpu`와 같은)을 추가하는 것이 좋은 관행입니다.

## 6단계. 훈련

다음 명령을 실행하여 훈련을 시작하십시오. 훈련 스크립트는 약 90분의 훈련 후에 DreamBooth 개념을 효과적으로 캡처하는 이미지를 생성하는 기본 구성을 사용합니다. 이 train 명령은 자동으로 체크포인트를 `models/loras/` 디렉토리에 저장합니다.

```bash
## 추론 도커 이미지 빌드
docker build -f Dockerfile.train -t flux-train .

## 훈련 트리거
sh launch_train.sh
```

## 7단계. 미세 조정된 모델 추론

이제 미세 조정된 LoRA를 사용하여 이미지를 생성해 보겠습니다!

```bash
## ComfyUI 컨테이너 시작 (flux-finetuning/assets 안에 있는지 확인)
## `torchaudio`에 대한 가져오기 오류는 무시할 수 있습니다
sh launch_comfyui.sh
```
`http://localhost:8188`에서 ComfyUI에 액세스하여 미세 조정된 모델로 이미지를 생성합니다. 기존 템플릿을 선택하지 마십시오.

ComfyUI의 왼쪽 패널에서 워크플로 섹션을 찾으십시오(또는 `w` 누르기). 열면 두 개의 기존 워크플로가 로드되어 있어야 합니다. 미세 조정된 Flux 모델의 경우 `finetuned_flux.json` 워크플로를 로드하겠습니다. json을 로드하면 ComfyUI가 워크플로를 로드하는 것을 볼 수 있습니다.

`CLIP Text Encode (Prompt)` 블록에 프롬프트를 제공하십시오. 이제 미세 조정된 모델의 프롬프트에 사용자 정의 개념을 통합해 보겠습니다. 예를 들어 `tjtoy toy holding sparkgpu gpu in a datacenter`를 사용하겠습니다. 고해상도 1024px 이미지를 만드는 것은 계산 집약적이므로 생성에 ~3분이 걸릴 것으로 예상할 수 있습니다.

기본 모델과 달리 미세 조정된 모델은 단일 이미지에서 여러 개념을 생성할 수 있습니다. 또한 ComfyUI는 생성된 이미지의 모양과 느낌을 조정하고 변경할 수 있는 여러 필드를 노출합니다.

## 문제 해결

| 증상 | 원인 | 해결 방법 |
|---------|--------|-----|
| URL에 대한 게이트된 리포지토리에 액세스할 수 없음 | 특정 HuggingFace 모델에 제한된 액세스가 있음 | [HuggingFace 토큰](https://huggingface.co/docs/hub/en/security-tokens)을 재생성하고 웹 브라우저에서 [게이트된 모델](https://huggingface.co/docs/hub/en/models-gated#customize-requested-information)에 대한 액세스 요청 |

> [!NOTE]
> DGX Spark는 GPU와 CPU 간의 동적 메모리 공유를 가능하게 하는 통합 메모리 아키텍처(UMA)를 사용합니다.
> UMA를 활용하도록 업데이트되는 많은 애플리케이션에서 여전히 DGX Spark의 메모리 용량 내에서도
> 메모리 문제가 발생할 수 있습니다. 그런 경우 다음과 같이 버퍼 캐시를 수동으로 플러시하십시오:
```bash
sudo sh -c 'sync; echo 3 > /proc/sys/vm/drop_caches'
```
