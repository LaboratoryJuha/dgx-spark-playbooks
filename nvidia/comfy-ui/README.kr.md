# Comfy UI

> 이미지를 생성하기 위해 Comfy UI를 설치하고 사용하기

## 목차

- [개요](#개요)
- [지침](#지침)
- [문제 해결](#문제-해결)

---

## 개요

## 기본 아이디어

ComfyUI는 SDXL, Flux 및 기타 확산 기반 모델을 사용하여 AI 이미지 생성을 위한 오픈 소스 웹 서버 애플리케이션입니다. 여러 단계로 이미지 생성 및 편집 워크플로를 생성, 편집 및 실행할 수 있는 브라우저 기반 UI가 있습니다. 이러한 생성 및 편집 단계(예: 모델 로드, 텍스트 추가 또는 샘플링)는 UI에서 노드로 구성할 수 있으며, 노드를 와이어로 연결하여 워크플로를 형성합니다.

ComfyUI는 추론을 위해 호스트의 GPU를 사용하므로 DGX Spark에 설치하고 장치에서 직접 모든 이미지 생성 및 편집을 수행할 수 있습니다.

워크플로는 JSON 파일로 저장되므로 향후 작업, 협업 및 재현성을 위해 버전을 관리할 수 있습니다.

## 달성할 내용

NVIDIA DGX Spark 장치에 ComfyUI를 설치하고 구성하여 통합 메모리를 사용하여 대형 모델을 작업할 수 있습니다.

## 시작하기 전에 알아야 할 사항

- Python 가상 환경 및 패키지 관리 작업 경험
- 명령줄 작업 및 터미널 사용에 대한 익숙함
- 딥러닝 모델 배포 및 체크포인트에 대한 기본 이해
- 컨테이너 워크플로 및 GPU 가속 개념에 대한 지식
- 웹 서비스 액세스를 위한 네트워크 구성에 대한 이해

## 전제 조건

**하드웨어 요구 사항:**
-  NVIDIA Grace Blackwell GB10 Superchip 시스템
-  Stable Diffusion 모델용 최소 8GB GPU 메모리
-  최소 20GB 이상의 사용 가능한 저장 공간

**소프트웨어 요구 사항:**
- Python 3.8 이상 설치됨: `python3 --version`
- pip 패키지 관리자 사용 가능: `pip3 --version`
- Blackwell과 호환되는 CUDA 툴킷: `nvcc --version`
- Git 버전 관리: `git --version`
- Hugging Face에서 모델을 다운로드하기 위한 네트워크 액세스
- `<SPARK_IP>:8188` 포트에 대한 웹 브라우저 액세스

## 부속 파일

필요한 모든 자산은 [GitHub의 ComfyUI 저장소](https://github.com/comfyanonymous/ComfyUI)에서 찾을 수 있습니다

- `requirements.txt` - ComfyUI 설치를 위한 Python 종속성
- `main.py` - 기본 ComfyUI 서버 애플리케이션 진입점
- `v1-5-pruned-emaonly-fp16.safetensors` - Stable Diffusion 1.5 체크포인트 모델

## 시간 및 위험

* **예상 시간:** 30-45분 (모델 다운로드 포함)
* **위험 수준:** 중간
  * 모델 다운로드가 크며(~2GB) 네트워크 문제로 인해 실패할 수 있음
  * 웹 인터페이스 기능을 위해 포트 8188에 액세스할 수 있어야 함
* **롤백:** 가상 환경을 삭제하여 설치된 모든 패키지를 제거할 수 있습니다. 다운로드한 모델은 체크포인트 디렉토리에서 수동으로 제거할 수 있습니다.
* **최종 업데이트:** 11/10/2025
  * ComfyUI PyTorch를 CUDA 13.0으로 업데이트

## 지침

## 1단계. 시스템 전제 조건 확인

설치를 진행하기 전에 NVIDIA DGX Spark 장치가 요구 사항을 충족하는지 확인하십시오.

```bash
python3 --version
pip3 --version
nvcc --version
nvidia-smi
```

예상 출력은 Python 3.8+, pip 사용 가능, CUDA 툴킷 및 GPU 감지를 표시해야 합니다.

## 2단계. Python 가상 환경 생성

호스트 시스템에 ComfyUI를 설치할 것이므로 시스템 패키지와의 충돌을 피하기 위해 격리된 환경을 만들어야 합니다.

```bash
python3 -m venv comfyui-env
source comfyui-env/bin/activate
```

명령 프롬프트에 `(comfyui-env)`가 표시되는지 확인하여 가상 환경이 활성화되었는지 확인하십시오.

## 3단계. CUDA 지원으로 PyTorch 설치

CUDA 13.0 지원으로 PyTorch를 설치하십시오.

```bash
pip3 install torch torchvision --index-url https://download.pytorch.org/whl/cu130
```

이 설치는 Blackwell 아키텍처 GPU와 CUDA 13.0 호환성을 대상으로 합니다.

## 4단계. ComfyUI 저장소 복제

공식 저장소에서 ComfyUI 소스 코드를 다운로드하십시오.

```bash
git clone https://github.com/comfyanonymous/ComfyUI.git
cd ComfyUI/
```

## 5단계. ComfyUI 종속성 설치

ComfyUI 작동에 필요한 Python 패키지를 설치하십시오.

```bash
pip install -r requirements.txt
```

이것은 웹 인터페이스 구성 요소 및 모델 처리 라이브러리를 포함한 모든 필요한 종속성을 설치합니다.

## 6단계. Stable Diffusion 체크포인트 다운로드

체크포인트 디렉토리로 이동하여 Stable Diffusion 1.5 모델을 다운로드하십시오.

```bash
cd models/checkpoints/
wget https://huggingface.co/Comfy-Org/stable-diffusion-v1-5-archive/resolve/main/v1-5-pruned-emaonly-fp16.safetensors
cd ../../
```

다운로드는 약 2GB이며 네트워크 속도에 따라 몇 분이 걸릴 수 있습니다.

## 7단계. ComfyUI 서버 시작

네트워크 액세스가 활성화된 상태로 ComfyUI 웹 서버를 시작하십시오.

```bash
python main.py --listen 0.0.0.0
```

서버는 포트 8188의 모든 네트워크 인터페이스에 바인딩되어 다른 장치에서 액세스할 수 있습니다.

## 8단계. 설치 확인

ComfyUI가 올바르게 실행되고 웹 브라우저를 통해 액세스할 수 있는지 확인하십시오.

```bash
curl -I http://localhost:8188
```

예상 출력은 웹 서버가 작동 중임을 나타내는 HTTP 200 응답을 표시해야 합니다.

웹 브라우저를 열고 `http://<SPARK_IP>:8188`로 이동하십시오. 여기서 `<SPARK_IP>`는 장치의 IP 주소입니다.

## 9단계. 선택 사항 - 정리 및 롤백

설치를 완전히 제거해야 하는 경우 다음 단계를 따르십시오:

> [!WARNING]
> 이렇게 하면 설치된 모든 패키지와 다운로드한 모델이 삭제됩니다.

```bash
deactivate
rm -rf comfyui-env/
rm -rf ComfyUI/
```

설치 중에 롤백하려면 `Ctrl+C`를 눌러 서버를 중지하고 가상 환경을 제거하십시오.

## 10단계. 선택 사항 - 다음 단계

기본 이미지 생성 워크플로로 설치를 테스트하십시오:

1. `http://<SPARK_IP>:8188`에서 웹 인터페이스에 액세스
2. 기본 워크플로 로드 (자동으로 나타나야 함)
3. "Run"을 클릭하여 첫 번째 이미지 생성
4. 별도의 터미널에서 `nvidia-smi`로 GPU 사용량 모니터링

이미지 생성은 하드웨어 구성에 따라 30-60초 이내에 완료되어야 합니다.

## 문제 해결

| 증상 | 원인 | 해결 방법 |
|---------|-------|-----|
| PyTorch CUDA 사용 불가 | CUDA 버전이 잘못되었거나 드라이버 누락 | `nvcc --version`이 cu129와 일치하는지 확인하고 PyTorch를 재설치 |
| 모델 다운로드 실패 | 네트워크 연결 또는 저장 공간 | 인터넷 연결 확인, 20GB+ 사용 가능한 공간 확인 |
| 웹 인터페이스 액세스 불가 | 방화벽이 포트 8188 차단 | 포트 8188을 허용하도록 방화벽 구성, IP 주소 확인 |
| 버퍼 캐시를 수동으로 플러시한 후 GPU 메모리 부족 오류 | 모델에 대한 VRAM 부족 | 더 작은 모델 사용 또는 CPU 폴백 모드 활성화 |

> [!NOTE]
> DGX Spark는 GPU와 CPU 간의 동적 메모리 공유를 가능하게 하는 통합 메모리 아키텍처(UMA)를 사용합니다.
> UMA를 활용하도록 업데이트되는 많은 애플리케이션에서 여전히 DGX Spark의 메모리 용량 내에서도
> 메모리 문제가 발생할 수 있습니다. 그런 경우 다음과 같이 버퍼 캐시를 수동으로 플러시하십시오:
```bash
sudo sh -c 'sync; echo 3 > /proc/sys/vm/drop_caches'
```


최신 알려진 문제에 대해서는 [DGX Spark 사용자 가이드](https://docs.nvidia.com/dgx/dgx-spark/known-issues.html)를 참조하십시오.
