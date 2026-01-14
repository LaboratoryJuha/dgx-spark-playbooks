# SGLang 추론 서버

> DGX Spark에서 SGLang 설치 및 사용

## 목차

- [개요](#개요)
  - [시간 및 위험도](#시간-및-위험도)
- [지침](#지침)
- [문제 해결](#문제-해결)

---

## 개요

## 기본 개념

SGLang는 대규모 언어 모델과 비전 언어 모델을 효율적으로 실행하도록 설계된 빠른 서빙 프레임워크로, 백엔드 런타임과 프론트엔드 언어를 공동 설계하여 모델과의 상호 작용을 더 빠르고 제어 가능하게 만듭니다. 이 설정은 Blackwell 아키텍처를 갖춘 단일 NVIDIA Spark 장치에서 최적화된 NVIDIA SGLang NGC 컨테이너를 사용하여 모든 종속성이 사전 설치된 GPU 가속 추론을 제공합니다.

## 달성할 목표

NVIDIA Spark 장치에서 서버 및 오프라인 추론 모드 모두에서 SGLang를 배포하여 DeepSeek-V2-Lite와 같은 모델을 사용한 텍스트 생성, 챗 완성 및 비전 언어 작업을 지원하는 고성능 LLM 서빙을 가능하게 합니다.

## 시작하기 전에 알아야 할 사항

- Linux 시스템의 터미널 환경에서 작업
- Docker 컨테이너 및 컨테이너 관리에 대한 기본 이해
- NVIDIA GPU 드라이버 및 CUDA 툴킷 개념에 대한 익숙함
- HTTP API 엔드포인트 및 JSON 요청/응답 처리 경험

## 필수 사항

- Blackwell 아키텍처를 갖춘 NVIDIA Spark 장치
- Docker Engine 설치 및 실행 중: `docker --version`
- NVIDIA GPU 드라이버 설치: `nvidia-smi`
- NVIDIA Container Toolkit 구성: `docker run --rm --gpus all lmsysorg/sglang:spark nvidia-smi`
- 충분한 디스크 공간 (>20GB 사용 가능): `df -h`
- NGC 컨테이너를 다운로드하기 위한 네트워크 연결: `ping nvcr.io`

## 보조 파일

- 오프라인 추론 Python 스크립트는 [여기 GitHub에서](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main/nvidia/sglang/assets/offline-inference.py) 찾을 수 있습니다

## 모델 지원 매트릭스

다음 모델은 Spark의 SGLang와 함께 지원됩니다. 나열된 모든 모델은 사용 가능하며 즉시 사용할 수 있습니다:

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

### 시간 및 위험도

* **예상 시간:** 초기 설정 및 검증에 30분
* **위험도:** 낮음 - 최소한의 구성으로 사전 구축되고 검증된 SGLang 컨테이너 사용
* **롤백:** `docker stop` 및 `docker rm` 명령으로 컨테이너 중지 및 제거
* **마지막 업데이트:** 01/02/2026
    * 모델 지원 매트릭스 추가

## 지침

## 1단계. 시스템 필수 사항 확인

계속하기 전에 NVIDIA Spark 장치가 모든 요구 사항을 충족하는지 확인합니다. 이 단계는 호스트 시스템에서 실행되며 Docker, GPU 드라이버 및 컨테이너 툴킷이 올바르게 구성되었는지 확인합니다.

```bash
## Docker 설치 확인
docker --version

## NVIDIA GPU 드라이버 확인
nvidia-smi

## Docker GPU 지원 확인
docker run --rm --gpus all lmsysorg/sglang:spark nvidia-smi

## 사용 가능한 디스크 공간 확인
df -h /
```

권한 거부 오류(Docker 데몬 소켓에 연결하는 동안 권한 거부와 같은)가 표시되면 docker 그룹에 사용자를 추가하여 sudo 없이 명령을 실행할 수 있도록 합니다.

```bash
sudo usermod -aG docker $USER
newgrp docker
```

## 2단계. SGLang 컨테이너 가져오기

최신 SGLang 컨테이너를 다운로드합니다. 이 단계는 호스트에서 실행되며 네트워크 연결에 따라 몇 분 정도 걸릴 수 있습니다.


```bash
## SGLang 컨테이너 가져오기
docker pull lmsysorg/sglang:spark

## 이미지가 다운로드되었는지 확인
docker images | grep sglang
```

## 3단계. 서버 모드용 SGLang 컨테이너 시작

HTTP API 액세스를 활성화하기 위해 서버 모드에서 SGLang 컨테이너를 시작합니다. 이는 컨테이너 내에서 추론 서버를 실행하여 클라이언트 연결을 위해 포트 30000에 노출합니다.

```bash
## GPU 지원 및 포트 매핑으로 컨테이너 시작
docker run --gpus all -it --rm \
  -p 30000:30000 \
  -v /tmp:/tmp \
  lmsysorg/sglang:spark \
  bash
```

## 4단계. SGLang 추론 서버 시작

컨테이너 내에서 지원되는 모델로 HTTP 추론 서버를 시작합니다. 이 단계는 Docker 컨테이너 내에서 실행되며 SGLang 서버 데몬을 시작합니다.

```bash
## DeepSeek-V2-Lite 모델로 추론 서버 시작
python3 -m sglang.launch_server \
  --model-path deepseek-ai/DeepSeek-V2-Lite \
  --host 0.0.0.0 \
  --port 30000 \
  --trust-remote-code \
  --tp 1 \
  --attention-backend flashinfer \
  --mem-fraction-static 0.75 &

## 서버 초기화 대기
sleep 30

## 서버 상태 확인
curl http://localhost:30000/health
```

## 5단계. 클라이언트-서버 추론 테스트

호스트 시스템의 새 터미널에서 SGLang 서버 API를 테스트하여 올바르게 작동하는지 확인합니다. 이는 서버가 요청을 수락하고 응답을 생성하는지 검증합니다.

```bash
## curl로 테스트
curl -X POST http://localhost:30000/generate \
  -H "Content-Type: application/json" \
  -d '{
      "text": "What does NVIDIA love?",
      "sampling_params": {
          "temperature": 0.7,
          "max_new_tokens": 100
      }
  }'
```

## 6단계. Python 클라이언트 API 테스트

SGLang 서버에 대한 프로그래밍 방식 액세스를 테스트하기 위해 간단한 Python 스크립트를 만듭니다. 이는 호스트 시스템에서 실행되며 SGLang를 애플리케이션에 통합하는 방법을 보여줍니다.

```python
import requests

## 서버에 프롬프트 전송
response = requests.post('http://localhost:30000/generate', json={
  'text': 'What does NVIDIA love?',
  'sampling_params': {
      'temperature': 0.7,
      'max_new_tokens': 100,
  },
})

print(f"Response: {response.json()['text']}")
```

## 7단계. 설치 검증

서버 및 오프라인 모드 모두가 올바르게 작동하는지 확인합니다. 이 단계는 완전한 SGLang 설정을 검증하고 안정적인 작동을 보장합니다.

```bash
## 서버 모드 확인 (호스트에서)
curl http://localhost:30000/health
curl -X POST http://localhost:30000/generate -H "Content-Type: application/json" \
  -d '{"text": "Hello", "sampling_params": {"max_new_tokens": 10}}'

## 컨테이너 로그 확인
docker ps
docker logs <CONTAINER_ID>
```

## 8단계. 정리 및 롤백

리소스를 정리하기 위해 컨테이너를 중지하고 제거합니다. 이 단계는 시스템을 원래 상태로 되돌립니다.

> [!WARNING]
> 이렇게 하면 모든 SGLang 컨테이너가 중지되고 임시 데이터가 제거됩니다.

```bash
## 모든 SGLang 컨테이너 중지
docker ps | grep sglang | awk '{print $1}' | xargs docker stop

## 중지된 컨테이너 제거
docker container prune -f

## SGLang 이미지 제거 (선택 사항)
docker rmi lmsysorg/sglang:spark
```

## 9단계. 다음 단계

SGLang가 성공적으로 배포되면 이제 다음을 수행할 수 있습니다:

- `/generate` 엔드포인트를 사용하여 HTTP API를 애플리케이션에 통합
- `--model-path` 매개변수를 변경하여 다양한 모델 실험
- `--tp` (텐서 병렬) 설정을 조정하여 여러 GPU를 사용하도록 확장
- 선택한 컨테이너 오케스트레이션 플랫폼을 사용하여 프로덕션 워크로드 배포

## 문제 해결

일반적인 문제 및 해결 방법:

| 증상 | 원인 | 해결 방법 |
|---------|-------|-----|
| GPU 오류로 컨테이너 시작 실패 | NVIDIA 드라이버/툴킷 누락 | nvidia-container-toolkit 설치, Docker 재시작 |
| 서버가 404 또는 연결 거부로 응답 | 서버가 완전히 초기화되지 않음 | 60초 대기, 컨테이너 로그 확인 |
| 모델 로딩 중 메모리 부족 오류 | GPU 메모리 부족 | 더 작은 모델 사용 또는 --tp 매개변수 증가 |
| 모델 다운로드 실패 | 네트워크 연결 문제 | 인터넷 연결 확인, 다운로드 재시도 |
| /tmp 액세스 권한 거부 | 볼륨 마운트 문제 | 전체 경로 사용: -v /tmp:/tmp 또는 전용 디렉토리 생성 |

> [!NOTE]
> DGX Spark는 GPU와 CPU 간의 동적 메모리 공유를 가능하게 하는 통합 메모리 아키텍처(UMA)를 사용합니다.
> 많은 애플리케이션이 여전히 UMA를 활용하도록 업데이트 중이므로 DGX Spark의 메모리 용량 내에 있어도
> 메모리 문제가 발생할 수 있습니다. 이 경우 다음을 사용하여 수동으로 버퍼 캐시를 플러시하세요:
```bash
sudo sh -c 'sync; echo 3 > /proc/sys/vm/drop_caches'
```
