# Spark의 NIM

> Spark에 NIM 배포

## 목차

- [개요](#개요)
  - [기본 아이디어](#기본-아이디어)
  - [달성할 내용](#달성할-내용)
  - [시작하기 전에 알아야 할 사항](#시작하기-전에-알아야-할-사항)
  - [전제 조건](#전제-조건)
  - [시간 및 위험](#시간-및-위험)
- [지침](#지침)
  - [2단계. NGC 인증 구성](#2단계-ngc-인증-구성)
- [문제 해결](#문제-해결)

---

## 개요

### 기본 아이디어

NVIDIA NIM은 NVIDIA GPU에서 빠르고 안정적인 AI 모델 서빙 및 추론을 위한 컨테이너화된 소프트웨어입니다. 이 플레이북은 DGX Spark 장치에서 LLM용 NIM 마이크로서비스를 실행하는 방법을 보여주며, 간단한 Docker 워크플로우를 통해 로컬 GPU 추론을 가능하게 합니다. NVIDIA 레지스트리로 인증하고, NIM 추론 마이크로서비스를 시작하고, 기능을 확인하기 위한 기본 추론 테스트를 수행합니다.

### 달성할 내용

DGX Spark 장치에서 NIM 컨테이너를 시작하여 텍스트 완성을 위한 GPU 가속 HTTP 엔드포인트를 노출합니다. 이 지침은 Llama 3.1 8B NIM과 함께 작동하는 것을 다루지만, DGX Spark용 [Qwen3-32 NIM](https://catalog.ngc.nvidia.com/orgs/nim/teams/qwen/containers/qwen3-32b-dgx-spark)을 포함한 추가 NIM을 사용할 수 있습니다([여기](https://docs.nvidia.com/nim/large-language-models/1.14.0/release-notes.html#new-language-models%20)에서 확인).

### 시작하기 전에 알아야 할 사항

- 터미널 환경에서 작업
- Docker 명령 및 GPU 활성화 컨테이너 사용
- REST API 및 curl 명령에 대한 기본 친숙함
- NVIDIA GPU 환경 및 CUDA에 대한 이해

### 전제 조건

- NVIDIA 드라이버가 설치된 DGX Spark 장치
  ```bash
  nvidia-smi
  ```
- NVIDIA Container Toolkit이 구성된 Docker, 지침 [여기](https://docs.nvidia.com/dgx/dgx-spark/nvidia-container-runtime-for-docker.html)
  ```bash
  docker run -it --gpus=all nvcr.io/nvidia/cuda:13.0.1-devel-ubuntu24.04 nvidia-smi
  ```
- [여기](https://ngc.nvidia.com/setup/api-key)에서 API 키가 있는 NGC 계정
  ```bash
  echo $NGC_API_KEY | grep -E '^[a-zA-Z0-9]{86}=='
  ```
- 모델 캐싱을 위한 충분한 디스크 공간(모델에 따라 다르며, 일반적으로 10-50GB)
  ```bash
  df -h ~
  ```


### 시간 및 위험

* **예상 시간:** 설정 및 검증에 15-30분
* **위험:**
  * 대용량 모델 다운로드는 네트워크 속도에 따라 상당한 시간이 걸릴 수 있음
  * GPU 메모리 요구 사항은 모델 크기에 따라 다름
  * 컨테이너 시작 시간은 모델 로딩에 따라 다름
* **롤백:** `docker stop <CONTAINER_NAME> && docker rm <CONTAINER_NAME>`으로 컨테이너를 중지하고 제거합니다. 디스크 공간 복구가 필요한 경우 `~/.cache/nim`에서 캐시된 모델을 제거합니다.
* **마지막 업데이트:** 12/22/2025
  * docker 컨테이너 버전을 cuda:13.0.1-devel-ubuntu24.04로 업데이트
  * docker 컨테이너 권한 설정 지침 추가

## 지침

## 1단계. 환경 전제 조건 확인

시스템이 GPU 활성화 컨테이너를 실행하기 위한 기본 요구 사항을 충족하는지 확인합니다.

```bash
nvidia-smi
docker --version
docker run --rm --gpus all nvcr.io/nvidia/cuda:13.0.1-devel-ubuntu24.04 nvidia-smi
```

권한 거부 오류가 표시되면(Docker 데몬 소켓에 연결하는 동안 권한 거부와 같은 메시지), sudo 없이 명령을 실행할 수 있도록 사용자를 docker 그룹에 추가합니다.

```bash
sudo usermod -aG docker $USER
newgrp docker
```

### 2단계. NGC 인증 구성

NGC API 키를 사용하여 NVIDIA 컨테이너 레지스트리에 대한 액세스를 설정합니다.

```bash
export NGC_API_KEY="<YOUR_NGC_API_KEY>"
echo "$NGC_API_KEY" | docker login nvcr.io --username '$oauthtoken' --password-stdin
```

## 3단계. NIM 컨테이너 선택 및 구성

NGC에서 특정 LLM NIM을 선택하고 모델 자산에 대한 로컬 캐싱을 설정합니다.

```bash
export CONTAINER_NAME="nim-llm-demo"
export IMG_NAME="nvcr.io/nim/meta/llama-3.1-8b-instruct-dgx-spark:latest"
export LOCAL_NIM_CACHE=~/.cache/nim
export LOCAL_NIM_WORKSPACE=~/.local/share/nim/workspace
mkdir -p "$LOCAL_NIM_WORKSPACE"
chmod -R a+w "$LOCAL_NIM_WORKSPACE"
mkdir -p "$LOCAL_NIM_CACHE"
chmod -R a+w "$LOCAL_NIM_CACHE"
```

## 4단계. NIM 컨테이너 시작

GPU 가속 및 적절한 리소스 할당으로 컨테이너화된 LLM 서비스를 시작합니다.

```bash
docker run -it --rm --name=$CONTAINER_NAME \
  --gpus all \
  --shm-size=16GB \
  -e NGC_API_KEY=$NGC_API_KEY \
  -v "$LOCAL_NIM_CACHE:/opt/nim/.cache" \
  -v "$LOCAL_NIM_WORKSPACE:/opt/nim/workspace" \
  -p 8000:8000 \
  $IMG_NAME
```

컨테이너는 첫 실행 시 모델을 다운로드하며 시작하는 데 몇 분이 걸릴 수 있습니다. 서비스가 준비되었음을 나타내는 시작 메시지를 확인하세요.

## 5단계. 추론 엔드포인트 검증

기능을 확인하기 위해 기본 완성 요청으로 배포된 서비스를 테스트합니다. 새 터미널에서 다음 curl 명령을 실행합니다.


```bash
curl -X 'POST' \
    'http://0.0.0.0:8000/v1/chat/completions' \
    -H 'accept: application/json' \
    -H 'Content-Type: application/json' \
    -d '{
      "model": "meta/llama-3.1-8b-instruct",
      "messages": [
        {
          "role":"system",
          "content":"detailed thinking on"
        },
        {
          "role":"user",
          "content":"Can you write me a song?"
        }
      ],
      "top_p": 1,
      "n": 1,
      "max_tokens": 15,
      "frequency_penalty": 1.0,
      "stop": ["hello"]

    }'

```

예상 출력은 생성된 텍스트가 포함된 완성 필드가 있는 JSON 응답이어야 합니다.

## 6단계. 정리 및 롤백

실행 중인 컨테이너를 제거하고 선택적으로 캐시된 모델 파일을 정리합니다.

> [!WARNING]
> 캐시된 모델을 제거하면 다음 실행 시 재다운로드가 필요합니다.

```bash
docker stop $CONTAINER_NAME
docker rm $CONTAINER_NAME
```

캐시된 모델을 제거하고 디스크 공간을 확보하려면:
```bash
rm -rf "$LOCAL_NIM_CACHE"
```

## 7단계. 다음 단계

작동하는 NIM 배포를 통해 다음을 수행할 수 있습니다:

- OpenAI 호환 인터페이스를 사용하여 API 엔드포인트를 애플리케이션에 통합
- NGC 카탈로그에서 사용 가능한 다양한 모델 실험
- 컨테이너 오케스트레이션 도구를 사용하여 배포 확장
- 리소스 사용량을 모니터링하고 컨테이너 리소스 할당 최적화

선호하는 HTTP 클라이언트 또는 SDK와 통합을 테스트하여 애플리케이션 구축을 시작하세요.

## 문제 해결

| 증상 | 원인 | 해결 방법 |
|---------|--------|-----|
| GPU 오류로 컨테이너 시작 실패 | NVIDIA Container Toolkit이 구성되지 않음 | nvidia-container-toolkit을 설치하고 Docker 재시작 |
| docker 로그인 중 "Invalid credentials" | NGC API 키 형식이 잘못됨 | NGC 포털에서 API 키 확인, 추가 공백이 없는지 확인 |
| 모델 다운로드가 중단되거나 실패함 | 네트워크 연결 또는 디스크 공간 부족 | 인터넷 연결 및 캐시 디렉토리의 사용 가능한 디스크 공간 확인 |
| API가 404 또는 연결 거부를 반환함 | 컨테이너가 완전히 시작되지 않았거나 잘못된 포트 | 컨테이너 시작 완료를 기다리고, 포트 8000에 액세스할 수 있는지 확인 |
| runtime not found | NVIDIA Container Toolkit이 제대로 구성되지 않음 | `sudo nvidia-ctk runtime configure --runtime=docker`를 실행하고 Docker 재시작 |

> [!NOTE]
> DGX Spark는 GPU와 CPU 간에 동적 메모리 공유를 가능하게 하는 통합 메모리 아키텍처(UMA)를 사용합니다.
> 많은 애플리케이션이 여전히 UMA를 활용하도록 업데이트 중이므로 DGX Spark의 메모리 용량 내에 있더라도
> 메모리 문제가 발생할 수 있습니다. 이런 경우 다음과 같이 버퍼 캐시를 수동으로 플러시하세요:
```bash
sudo sh -c 'sync; echo 3 > /proc/sys/vm/drop_caches'
```
