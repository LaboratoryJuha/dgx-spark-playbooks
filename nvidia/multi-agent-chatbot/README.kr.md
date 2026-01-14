# 다중 에이전트 챗봇 구축 및 배포

> 다중 에이전트 챗봇 시스템을 배포하고 Spark에서 에이전트와 대화

## 목차

- [개요](#개요)
- [지침](#지침)
- [문제 해결](#문제-해결)

---

## 개요

## 기본 아이디어

이 플레이북은 DGX Spark를 사용하여 완전히 로컬인 다중 에이전트 시스템을 프로토타입, 구축 및 배포하는 방법을 보여줍니다.
128GB의 통합 메모리로 DGX Spark는 여러 LLM과 VLM을 병렬로 실행할 수 있어 에이전트 간 상호 작용을 가능하게 합니다.

핵심은 gpt-oss-120B로 구동되는 수퍼바이저 에이전트로, 코딩, 검색 증강 생성(RAG) 및 이미지 이해를 위한 전문화된 다운스트림 에이전트를 조정합니다.
DGX Spark의 인기 있는 AI 프레임워크 및 라이브러리에 대한 즉시 사용 가능한 지원 덕분에 개발 및 프로토타입 제작이 빠르고 마찰이 없습니다.
함께 이러한 구성 요소는 복잡한 다중 모달 워크플로가 로컬 고성능 하드웨어에서 효율적으로 실행될 수 있음을 보여줍니다.

## 달성할 내용

DGX Spark에서 실행되는 풀스택 다중 에이전트 챗봇 시스템을 갖게 되며 로컬 웹 브라우저를 통해 액세스할 수 있습니다.
설정에는 다음이 포함됩니다:
- llama.cpp 서버 및 TensorRT LLM 서버를 사용한 LLM 및 VLM 모델 제공
- 모델 추론 및 문서 검색 모두에 대한 GPU 가속
- gpt-oss-120B로 구동되는 수퍼바이저 에이전트를 사용한 다중 에이전트 시스템 오케스트레이션
- 수퍼바이저 에이전트를 위한 도구로서의 MCP(Model Context Protocol) 서버

## 전제 조건

-  DGX Spark 장치가 설정되어 있고 액세스 가능
-  DGX Spark GPU에서 실행 중인 다른 프로세스가 없음
-  모델 다운로드를 위한 충분한 디스크 공간

> [!NOTE]
> 이 데모는 기본적으로 DGX Spark의 128GB 메모리 중 ~120GB를 사용합니다.
> `nvidia-smi`를 사용하여 Spark에서 다른 워크로드가 실행되고 있지 않은지 확인하거나 gpt-oss-20B와 같은 더 작은 수퍼바이저 모델로 전환하십시오.


## 시간 및 위험

* **예상 시간**: 30분에서 1시간
* **위험:**
  * Docker 권한 문제로 인해 사용자 그룹 변경 및 세션 재시작이 필요할 수 있음
  * 설정에는 gpt-oss-120B(~63GB), Deepseek-Coder:6.7B-Instruct(~7GB) 및 Qwen3-Embedding-4B(~4GB)에 대한 모델 파일 다운로드가 포함되며 네트워크 속도에 따라 30분에서 2시간이 걸릴 수 있음
* **롤백**: 제공된 정리 명령을 사용하여 Docker 컨테이너를 중지하고 제거합니다.
* **최종 업데이트**: 11/20/2025
  * DGX Spark에서 llama.cpp를 실행하는 중요한 명령 수정

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

```bash
git clone https://github.com/NVIDIA/dgx-spark-playbooks
cd dgx-spark-playbooks/nvidia/multi-agent-chatbot/assets
```

## 3단계. 모델 다운로드 스크립트 실행

```bash
chmod +x model_download.sh
./model_download.sh
```

설정 스크립트는 HuggingFace에서 모델 GGUF 파일을 가져옵니다.
가져오는 모델 파일에는 gpt-oss-120B(~63GB), Deepseek-Coder:6.7B-Instruct(~7GB) 및 Qwen3-Embedding-4B(~4GB)가 포함됩니다.
네트워크 속도에 따라 30분에서 2시간이 걸릴 수 있습니다.


## 4단계. 애플리케이션을 위한 도커 컨테이너 시작

```bash
  docker compose -f docker-compose.yml -f docker-compose-models.yml up -d --build
```
이 단계는 기본 llama.cpp 서버 이미지를 빌드하고 모델을 제공하는 데 필요한 모든 도커 서비스, 백엔드 API 서버 및 프론트엔드 UI를 시작합니다.
이 단계는 네트워크 속도에 따라 10분에서 20분이 걸릴 수 있습니다.
모든 컨테이너가 준비되고 정상 상태가 될 때까지 기다립니다.

```bash
watch 'docker ps --format "table {{.ID}}\t{{.Names}}\t{{.Status}}"'
```

## 5단계. 프론트엔드 UI 액세스

브라우저를 열고 다음으로 이동하십시오: http://localhost:3000

> [!NOTE]
> SSH 연결을 통해 원격 GPU에서 이것을 실행하는 경우 새 터미널 창에서 다음 명령을 실행하여 localhost:3000에서 UI에 액세스하고 UI가 localhost:8000에서 백엔드와 통신할 수 있도록 해야 합니다.

>```ssh -L 3000:localhost:3000 -L 8000:localhost:8000  username@IP-address```

## 6단계. 샘플 프롬프트 시도

프론트엔드의 타일을 클릭하여 수퍼바이저 및 다른 에이전트를 시도하십시오.

**RAG 에이전트**:
RAG 에이전트에 대한 예제 프롬프트를 시도하기 전에 링크로 이동하여 PDF를 로컬 파일 시스템에 다운로드하고 왼쪽 사이드바의 "Context" 아래에 있는 녹색 "Upload Documents" 버튼을 클릭한 다음 "Select Sources" 섹션의 상자를 확인하여 예제 PDF 문서 [NVIDIA Blackwell Whitepaper](https://images.nvidia.com/aem-dam/Solutions/geforce/blackwell/nvidia-rtx-blackwell-gpu-architecture.pdf)를 컨텍스트로 업로드하십시오.

## 8단계. 정리 및 롤백

컨테이너를 완전히 제거하고 리소스를 확보하는 단계입니다.

multi-agent-chatbot 프로젝트의 루트 디렉토리에서 다음 명령을 실행하십시오:

```bash
docker compose -f docker-compose.yml -f docker-compose-models.yml down

docker volume rm "$(basename "$PWD")_postgres_data"
```

## 9단계. 다음 단계

- 다중 에이전트 챗봇 시스템으로 다양한 프롬프트를 시도해 보십시오.
- 저장소의 지침에 따라 다양한 모델을 시도해 보십시오.
- 수퍼바이저 에이전트를 위한 도구로 새로운 MCP(Model Context Protocol) 서버를 추가해 보십시오.

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
