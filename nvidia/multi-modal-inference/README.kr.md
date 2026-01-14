# 멀티모달 추론

> TensorRT를 사용한 멀티모달 추론 설정

## 목차

- [개요](#개요)
- [지침](#지침)
- [문제 해결](#문제-해결)

---

## 개요

## 기본 아이디어

멀티모달 추론은 단일 모델 파이프라인 내에서 **텍스트, 이미지 및 오디오**와 같은 다양한 데이터 유형을 결합하여 더 풍부한 출력을 생성하거나 해석합니다.
한 번에 하나의 입력 유형을 처리하는 대신, 멀티모달 시스템은 **텍스트-이미지 생성**, **이미지 캡셔닝** 또는 **비전-언어 추론**을 가능하게 하는 공유 표현을 갖습니다.

GPU에서 이는 더 빠르고 높은 정확도의 결과를 위해 **모달리티 간 병렬 처리**를 가능하게 하여 언어와 비전을 결합하는 작업을 수행합니다.

## 달성할 내용

TensorRT를 사용하여 NVIDIA Spark에서 GPU 가속 멀티모달 추론 기능을 배포하여 여러 정밀도 형식(FP16, FP8, FP4)에서 최적화된 성능으로 Flux.1 및 SDXL 확산 모델을 실행합니다.

## 시작하기 전에 알아야 할 사항

- Docker 컨테이너 및 GPU 패스스루 작업
- 모델 최적화를 위한 TensorRT 사용
- Hugging Face 모델 허브 인증 및 다운로드
- GPU 워크로드를 위한 명령줄 도구
- 확산 모델 및 이미지 생성에 대한 기본 이해

## 전제 조건

- Blackwell GPU 아키텍처가 있는 NVIDIA Spark 장치
- Docker가 설치되어 현재 사용자가 액세스할 수 있음
- NVIDIA Container Runtime이 구성됨
- Black Forest Labs 모델에 대한 액세스 권한이 있는 Hugging Face 계정 [FLUX.1-dev](https://huggingface.co/black-forest-labs/FLUX.1-dev) 및 [FLUX.1-dev-onnx](https://huggingface.co/black-forest-labs/FLUX.1-dev-onnx)
- 두 FLUX.1 모델 리포지토리에 대한 액세스 권한이 구성된 Hugging Face [토큰](https://huggingface.co/settings/tokens)
- FP16 Flux.1 Schnell 작업을 위해 최소 48GB VRAM 사용 가능
- GPU 액세스 확인: `nvidia-smi`
- Docker GPU 통합 확인: `docker run --rm --gpus all nvcr.io/nvidia/pytorch:25.11-py3 nvidia-smi`

## 부속 파일

필요한 모든 파일은 TensorRT 리포지토리 [GitHub에서](https://github.com/NVIDIA/TensorRT) 찾을 수 있습니다
- [**requirements.txt**](https://github.com/NVIDIA/TensorRT/blob/main/demo/Diffusion/requirements.txt) - TensorRT 데모 환경을 위한 Python 종속성
- [**demo_txt2img_flux.py**](https://github.com/NVIDIA/TensorRT/blob/main/demo/Diffusion/demo_txt2img_flux.py) - Flux.1 모델 추론 스크립트
- [**demo_txt2img_xl.py**](https://github.com/NVIDIA/TensorRT/blob/main/demo/Diffusion/demo_txt2img_xl.py) - SDXL 모델 추론 스크립트
- **TensorRT 리포지토리** - 확산 데모 코드 및 최적화 도구 포함

## 시간 및 위험

- **소요 시간**: 모델 다운로드 및 최적화 단계에 따라 45-90분

- **위험**:
  - 대용량 모델 다운로드가 시간 초과될 수 있음
  - 높은 VRAM 요구 사항으로 인해 OOM 오류가 발생할 수 있음
  - 양자화된 모델은 품질 저하를 보일 수 있음

- **롤백**:
  - HuggingFace 캐시에서 다운로드한 모델 제거
  - 그런 다음 컨테이너 환경 종료

* **마지막 업데이트:** 12/22/2025
  * 최신 pytorch 컨테이너 버전 nvcr.io/nvidia/pytorch:25.11-py3로 업그레이드
  * 모델 액세스를 위한 HuggingFace 토큰 설정 지침 추가
  * docker 컨테이너 권한 설정 지침 추가

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

## 2단계. TensorRT 컨테이너 환경 시작

GPU 액세스 및 HuggingFace 캐시 마운팅과 함께 NVIDIA PyTorch 컨테이너를 시작합니다. 이는 필요한 모든 종속성이 사전 설치된 TensorRT 개발 환경을 제공합니다.

```bash
docker run --gpus all --ipc=host --ulimit memlock=-1 \
--ulimit stack=67108864 -it --rm --ipc=host \
-v $HOME/.cache/huggingface:/root/.cache/huggingface \
nvcr.io/nvidia/pytorch:25.11-py3
```

## 3단계. TensorRT 리포지토리 클론 및 설정

TensorRT 리포지토리를 다운로드하고 확산 모델 데모를 위한 환경을 구성합니다.

```bash
git clone https://github.com/NVIDIA/TensorRT.git -b main --single-branch && cd TensorRT
export TRT_OSSPATH=/workspace/TensorRT/
cd $TRT_OSSPATH/demo/Diffusion
```

## 4단계. 필수 종속성 설치

모델 양자화 및 최적화를 위한 NVIDIA ModelOpt 및 기타 종속성을 설치합니다.

```bash
## OpenGL 라이브러리 설치
apt update
apt install -y libgl1 libglu1-mesa libglib2.0-0t64 libxrender1 libxext6 libx11-6 libxrandr2 libxss1 libxcomposite1 libxdamage1 libxfixes3 libxcb1

pip install nvidia-modelopt[torch,onnx]
sed -i '/^nvidia-modelopt\[.*\]=.*/d' requirements.txt
pip3 install -r requirements.txt
pip install onnxconverter_common
```

오픈 모델에 액세스하기 위해 HuggingFace 토큰을 설정합니다.
```bash
export HF_TOKEN = <YOUR_HUGGING_FACE_TOKEN>
```

## 5단계. Flux.1 Dev 모델 추론 실행

다양한 정밀도 형식으로 Flux.1 Dev 모델을 사용하여 멀티모달 추론을 테스트합니다.

**하위 단계 A. BF16 양자화 정밀도**

```bash
python3 demo_txt2img_flux.py "a beautiful photograph of Mt. Fuji during cherry blossom" \
  --hf-token=$HF_TOKEN --download-onnx-models --bf16
```

**하위 단계 B. FP8 양자화 정밀도**

```bash
python3 demo_txt2img_flux.py "a beautiful photograph of Mt. Fuji during cherry blossom" \
  --hf-token=$HF_TOKEN --quantization-level 4 --fp8 --download-onnx-models
```

**하위 단계 C. FP4 양자화 정밀도**

```bash
python3 demo_txt2img_flux.py "a beautiful photograph of Mt. Fuji during cherry blossom" \
  --hf-token=$HF_TOKEN --fp4 --download-onnx-models
```

## 6단계. Flux.1 Schnell 모델 추론 실행

다양한 정밀도 형식으로 더 빠른 Flux.1 Schnell 변형을 테스트합니다.

> [!WARNING]
> FP16 Flux.1 Schnell은 네이티브 내보내기를 위해 >48GB VRAM이 필요합니다

**하위 단계 A. FP16 정밀도(높은 VRAM 요구 사항)**

```bash
python3 demo_txt2img_flux.py "a beautiful photograph of Mt. Fuji during cherry blossom" \
  --hf-token=$HF_TOKEN --version="flux.1-schnell"
```

**하위 단계 B. FP8 양자화 정밀도**

```bash
python3 demo_txt2img_flux.py "a beautiful photograph of Mt. Fuji during cherry blossom" \
  --hf-token=$HF_TOKEN --version="flux.1-schnell" \
  --quantization-level 4 --fp8 --download-onnx-models
```

**하위 단계 C. FP4 양자화 정밀도**

```bash
python3 demo_txt2img_flux.py "a beautiful photograph of Mt. Fuji during cherry blossom" \
  --hf-token=$HF_TOKEN --version="flux.1-schnell" \
  --fp4 --download-onnx-models
```

## 7단계. SDXL 모델 추론 실행

다양한 정밀도 형식으로 비교를 위해 SDXL 모델을 테스트합니다.

**하위 단계 A. BF16 정밀도**

```bash
python3 demo_txt2img_xl.py "a beautiful photograph of Mt. Fuji during cherry blossom" \
  --hf-token=$HF_TOKEN --version xl-1.0 --download-onnx-models
```

**하위 단계 B. FP8 양자화 정밀도**

```bash
python3 demo_txt2img_xl.py "a beautiful photograph of Mt. Fuji during cherry blossom" \
  --hf-token=$HF_TOKEN --version xl-1.0 --download-onnx-models --fp8
```

## 8단계. 추론 출력 검증

모델이 이미지를 성공적으로 생성했는지 확인하고 성능 차이를 측정합니다.

```bash
## 출력 디렉토리에서 생성된 이미지 확인
ls -la *.png *.jpg 2>/dev/null || echo "이미지 파일을 찾을 수 없습니다"

## CUDA 액세스 가능 여부 확인
nvidia-smi

## TensorRT 버전 확인
python3 -c "import tensorrt as trt; print(f'TensorRT 버전: {trt.__version__}')"
```

## 9단계. 정리 및 롤백

다운로드한 모델을 제거하고 컨테이너 환경을 종료하여 디스크 공간을 확보합니다.

> [!WARNING]
> 이렇게 하면 캐시된 모든 모델과 생성된 이미지가 삭제됩니다

```bash
## 컨테이너 종료
exit

## HuggingFace 캐시 제거(선택 사항)
rm -rf $HOME/.cache/huggingface/
```

## 10단계. 다음 단계

검증된 설정을 사용하여 사용자 정의 이미지를 생성하거나 멀티모달 추론을 애플리케이션에 통합합니다. 다양한 프롬프트를 시도하거나 확립된 TensorRT 환경으로 모델 미세 조정을 탐색하세요.

## 문제 해결

| 증상 | 원인 | 해결 방법 |
|---------|-------|-----|
| "CUDA out of memory" 오류 | 모델에 대한 VRAM 부족 | FP8/FP4 양자화 사용 또는 더 작은 모델 사용 |
| "Invalid HF token" 오류 | HuggingFace 토큰 누락 또는 만료됨 | 유효한 토큰 설정: `export HF_TOKEN=<YOUR_TOKEN>` |
| URL에 대한 게이트된 리포지토리에 액세스할 수 없음 | 특정 HuggingFace 모델에는 제한된 액세스 권한이 있음 | [HuggingFace 토큰](https://huggingface.co/docs/hub/en/security-tokens)을 재생성하고 웹 브라우저에서 [게이트된 모델](https://huggingface.co/docs/hub/en/models-gated#customize-requested-information)에 대한 액세스를 요청하세요 |
| 모델 다운로드 시간 초과 | 네트워크 문제 또는 속도 제한 | 명령 재시도 또는 모델 사전 다운로드 |

> [!NOTE]
> DGX Spark는 GPU와 CPU 간에 동적 메모리 공유를 가능하게 하는 통합 메모리 아키텍처(UMA)를 사용합니다.
> 많은 애플리케이션이 여전히 UMA를 활용하도록 업데이트 중이므로 DGX Spark의 메모리 용량 내에 있더라도
> 메모리 문제가 발생할 수 있습니다. 이런 경우 다음과 같이 버퍼 캐시를 수동으로 플러시하세요:
```bash
sudo sh -c 'sync; echo 3 > /proc/sys/vm/drop_caches'
```
