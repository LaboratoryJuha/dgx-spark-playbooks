# 비디오 검색 및 요약(VSS) 에이전트 구축

> Spark에서 VSS Blueprint 실행

## 목차

- [개요](#개요)
- [지침](#지침)
- [문제 해결](#문제-해결)

---

## 개요

## 기본 아이디어

NVIDIA의 비디오 검색 및 요약(VSS) AI Blueprint를 배포하여 비전 언어 모델, 대형 언어 모델 및 검색 증강 생성을 결합한 지능형 비디오 분석 시스템을 구축합니다. 이 시스템은 원시 비디오 콘텐츠를 비디오 요약, Q&A 및 실시간 알림을 통해 실시간 실행 가능한 통찰력으로 변환합니다. 완전히 로컬인 Event Reviewer 배포 또는 원격 모델 엔드포인트를 사용하는 하이브리드 배포를 설정합니다.

## 달성할 내용

Blackwell 아키텍처를 사용하는 NVIDIA Spark 하드웨어에 NVIDIA의 VSS AI Blueprint를 배포하며, 두 가지 배포 시나리오 중에서 선택합니다: VSS Event Reviewer(VLM 파이프라인이 포함된 완전히 로컬) 또는 Standard VSS(원격 LLM/임베딩 엔드포인트가 있는 하이브리드 배포). 여기에는 Alert Bridge, VLM Pipeline, Alert Inspector UI, Video Storage Toolkit 및 자동화된 비디오 분석 및 이벤트 검토를 위한 선택적 DeepStream CV 파이프라인 설정이 포함됩니다.

## 시작하기 전에 알아야 할 사항

- NVIDIA Docker 컨테이너 및 컨테이너 레지스트리 작업
- 공유 네트워크로 Docker Compose 환경 설정
- 환경 변수 및 인증 토큰 관리
- 비디오 처리 및 분석 워크플로에 대한 기본 이해

## 전제 조건

- ARM64 아키텍처 및 Blackwell GPU를 사용하는 NVIDIA Spark 장치
- NVIDIA DGX OS 7.2.3 이상
- 드라이버 버전 580.95.05 이상 설치됨: `nvidia-smi | grep "Driver Version"`
- CUDA 버전 13.0 설치됨: `nvcc --version`
- Docker 설치 및 실행 중: `docker --version && docker compose version`
- [NGC API Key](https://org.ngc.nvidia.com/setup/api-keys)를 사용하는 NVIDIA Container Registry에 대한 액세스
- [선택 사항] 원격 모델 엔드포인트용 NVIDIA API Key(하이브리드 배포만 해당)
- 비디오 처리를 위한 충분한 저장 공간(`/tmp/`에 >10GB 권장)

## 부속 파일

- [VSS Blueprint GitHub 저장소](https://github.com/NVIDIA-AI-Blueprints/video-search-and-summarization) - 메인 코드베이스 및 Docker Compose 구성
- [샘플 CV 감지 파이프라인](https://github.com/NVIDIA-AI-Blueprints/video-search-and-summarization/tree/main/examples/cv-event-detector) - 이벤트 검토자 워크플로를 위한 참조 CV 파이프라인
- [VSS 공식 문서](https://docs.nvidia.com/vss/latest/index.html) - 완전한 시스템 문서

## 시간 및 위험

* **소요 시간:** 초기 설정 30-45분, 비디오 처리 검증을 위한 추가 시간
* **위험:**
  * 컨테이너 시작은 대형 모델 다운로드로 인해 리소스 집약적이고 시간이 오래 걸릴 수 있음
  * 공유 네트워크가 이미 존재하는 경우 네트워크 구성 충돌
  * 원격 API 엔드포인트에 속도 제한 또는 연결 문제가 있을 수 있음(하이브리드 배포)
* **롤백:** `docker compose down`으로 모든 컨테이너를 중지하고, `docker network rm vss-shared-network`로 공유 네트워크를 제거하고, 임시 미디어 디렉토리를 정리합니다.
* **최종 업데이트:** 10/18/2025
  * 필요한 OS 및 드라이버 버전 업데이트
  * 완전히 로컬인 VSS 배포 지침 추가

## 지침

## 1단계. 환경 요구 사항 확인

시스템이 하드웨어 및 소프트웨어 전제 조건을 충족하는지 확인하십시오.

```bash
## 드라이버 버전 확인
nvidia-smi | grep "Driver Version"
## 예상 출력: Driver Version: 580.82.09 이상

## CUDA 버전 확인
nvcc --version
## 예상 출력: release 13.0

## Docker가 실행 중인지 확인
docker --version && docker compose version
```

## 2단계. Docker 구성

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


또한 NVIDIA Container Runtime을 사용할 수 있도록 Docker를 구성하십시오.

```bash
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker

##샘플 워크로드를 실행하여 설정 확인
sudo docker run --rm --runtime=nvidia --gpus all ubuntu nvidia-smi
```

## 3단계. VSS 저장소 복제

NVIDIA의 공개 GitHub에서 비디오 검색 및 요약 저장소를 복제하십시오.

```bash
## VSS AI Blueprint 저장소 복제
git clone https://github.com/NVIDIA-AI-Blueprints/video-search-and-summarization.git
cd video-search-and-summarization
```

## 4단계. 캐시 클리너 스크립트 실행

컨테이너 작업 중 메모리 사용을 최적화하기 위해 시스템 캐시 클리너를 시작하십시오.

```bash
## 다른 터미널에서 캐시 클리너 스크립트를 시작합니다.
## 또는 명령 끝에 " &"를 추가하여 백그라운드에서 실행합니다.
sudo sh deploy/scripts/sys_cache_cleaner.sh
```

## 5단계. Docker 공유 네트워크 설정

VSS 서비스와 CV 파이프라인 컨테이너 간에 공유될 Docker 네트워크를 만드십시오.

```bash
## 공유 네트워크 생성(Docker 구성에 따라 sudo가 필요할 수 있음)
docker network create vss-shared-network
```

> [!WARNING]
> 네트워크가 이미 존재하는 경우 오류가 표시될 수 있습니다. 필요한 경우 먼저 `docker network rm vss-shared-network`로 제거하십시오.

## 6단계. NVIDIA Container Registry로 인증

[NGC API Key](https://org.ngc.nvidia.com/setup/api-keys)를 사용하여 NVIDIA의 컨테이너 레지스트리에 로그인하십시오.

> [!NOTE]
> NVIDIA 계정이 아직 없는 경우 계정을 만들고 [개발자 프로그램](https://developer.nvidia.com/nvidia-developer-program)에 등록해야 합니다.

```bash
## NVIDIA Container Registry에 로그인
docker login nvcr.io
## Username: $oauthtoken
## Password: <PASTE_NGC_API_KEY_HERE>
```

## 7단계. 배포 시나리오 선택

요구 사항에 따라 두 가지 배포 옵션 중에서 선택하십시오:

| 배포 시나리오  | VLM (Cosmos-Reason1-7B) | LLM (Llama 3.1 70B) | 임베딩/리랭커 | CV 파이프라인 |
|----------------------|--------------------------|---------------------|--------------------|-------------|
| VSS Event Reviewer   | 로컬                    | 사용 안 함            | 사용 안 함           | 로컬       |
| Standard VSS (하이브리드)| 로컬                   | 원격              | 원격             | 선택 사항    |

Event Reviewer의 경우 **옵션 A**로, Standard VSS의 경우 **옵션 B**로 진행하십시오.

## 8단계. 옵션 A

**[VSS Event Reviewer](https://docs.nvidia.com/vss/latest/content/vss_event_reviewer.html) (완전히 로컬)**

**8.1 Event Reviewer 디렉토리로 이동**

Event Reviewer Docker Compose 구성이 포함된 디렉토리로 변경하십시오.

```bash
cd deploy/docker/event_reviewer/
```

**8.2 NGC API Key 구성**

NGC API Key로 환경 파일을 업데이트하십시오. `.env` 파일을 직접 편집하거나 다음 명령을 실행하여 수행할 수 있습니다:

```bash
## .env 파일을 편집하고 NGC_API_KEY를 업데이트
echo "NGC_API_KEY=<YOUR_NGC_API_KEY>" >> .env
```

**8.3 VSS 이미지 경로 업데이트**

`.env`에서 `VSS_IMAGE`를 `nvcr.io/nvidia/blueprint/vss-engine-sbsa:2.4.0`으로 업데이트하십시오.

```bash
## .env 파일을 편집하고 VSS_IMAGE를 업데이트
echo "VSS_IMAGE=nvcr.io/nvidia/blueprint/vss-engine-sbsa:2.4.0" >> .env
```

**8.4 VSS Event Reviewer 서비스 시작**

Alert Bridge, VLM Pipeline, Alert Inspector UI 및 Video Storage Toolkit을 포함한 완전한 VSS Event Reviewer 스택을 시작하십시오.

```bash
## ARM64 및 SBSA 최적화로 VSS Event Reviewer 시작
IS_SBSA=1 IS_AARCH64=1 ALERT_REVIEW_MEDIA_BASE_DIR=/tmp/alert-media-dir docker compose up
```

> [!NOTE]
> 이 단계는 컨테이너가 풀되고 서비스가 초기화되므로 몇 분이 걸립니다. VSS 백엔드는 추가 시작 시간이 필요합니다. 그 동안 새 터미널에서 다음 단계로 진행하십시오.

**8.5 CV Event Detector 디렉토리로 이동**

새 터미널 세션에서 컴퓨터 비전 이벤트 감지기 구성으로 이동하십시오.

```bash
cd video-search-and-summarization/examples/cv-event-detector
```

**8.6 NV_CV_EVENT_DETECTOR_IMAGE 이미지 경로 업데이트**

`.env`에서 `NV_CV_EVENT_DETECTOR_IMAGE`를 `nvcr.io/nvidia/blueprint/nv-cv-event-detector-sbsa:2.4.0`으로 업데이트하십시오.

```bash
## .env 파일을 편집하고 NV_CV_EVENT_DETECTOR_IMAGE를 업데이트
echo "NV_CV_EVENT_DETECTOR_IMAGE=nvcr.io/nvidia/blueprint/nv-cv-event-detector-sbsa:2.4.0" >> .env
```

**8.7 DeepStream CV 파이프라인 시작**

DeepStream 컴퓨터 비전 파이프라인 및 CV UI 서비스를 시작하십시오.

```bash
## ARM64 및 SBSA 최적화로 CV 파이프라인 시작
IS_SBSA=1 IS_AARCH64=1 ALERT_REVIEW_MEDIA_BASE_DIR=/tmp/alert-media-dir docker compose up
```

**8.8 서비스 초기화 대기**

사용자 인터페이스에 액세스하기 전에 모든 컨테이너가 완전히 초기화될 때까지 기다리십시오.

```bash
## 컨테이너 상태 모니터링
docker ps
## 모든 컨테이너가 "Up" 상태를 표시하고 VSS 백엔드 로그(vss-engine-sbsa:2.4.0)가 준비 상태 "Uvicorn running on http://0.0.0.0:7860"를 표시하는지 확인
## 총 8개의 컨테이너가 있어야 합니다:
## nvcr.io/nvidia/blueprint/nv-cv-event-detector-ui:2.4.0
## nvcr.io/nvidia/blueprint/nv-cv-event-detector-sbsa:2.4.0
## nginx:alpine
## nvcr.io/nvidia/blueprint/vss-alert-inspector-ui:2.4.0
## nvcr.io/nvidia/blueprint/alert-bridge:0.19.0-multiarch
## nvcr.io/nvidia/blueprint/vss-engine-sbsa:2.4.0
## nvcr.io/nvidia/blueprint/vst-storage:2.1.0-25.07.1
## redis/redis-stack-server:7.2.0-v9
```

**8.9 Event Reviewer 배포 검증**

웹 인터페이스에 액세스하여 성공적인 배포 및 기능을 확인하십시오.

```bash
## CV UI 접근성 테스트(기본값: localhost)
curl -I http://localhost:7862
## 예상: HTTP 200 응답

## Alert Inspector UI 접근성 테스트(기본값: localhost)
curl -I http://localhost:7860
## 예상: HTTP 200 응답

## Spark를 원격 또는 액세서리 모드에서 실행하는 경우 'localhost'를 Spark 장치의 IP 주소 또는 호스트 이름으로 교체하십시오.
## Spark의 IP 주소를 찾으려면 Spark 시스템에서 다음 명령을 실행하십시오:
hostname -I
## 또는 호스트 이름을 가져오려면:
hostname
## 그런 다음 'localhost' 대신 IP/호스트 이름을 사용하십시오. 예:
## curl -I http://<SPARK_IP_OR_HOSTNAME>:7862
```

브라우저에서 다음 URL을 여십시오:
- `http://localhost:7862` - CV 파이프라인을 시작하고 모니터링하기 위한 CV UI
- `http://localhost:7860` - 클립을 보고 VLM 결과를 검토하기 위한 Alert Inspector UI

> [!NOTE]
> 이제 10단계로 진행할 수 있습니다.

## 9단계. 옵션 B

**[Standard VSS](https://docs.nvidia.com/vss/latest/content/architecture.html) (하이브리드 배포)**

이 하이브리드 배포에서는 [build.nvidia.com](https://build.nvidia.com/)의 NIM을 사용합니다. 또는 [VSS 원격 배포 가이드](https://docs.nvidia.com/vss/latest/content/installation-remote-docker-compose.html)의 지침에 따라 고유한 호스팅 엔드포인트를 구성할 수 있습니다.

> [!NOTE]
> 더 작은 LLM(Llama 3.1 8B)을 사용하는 완전히 로컬인 배포도 가능합니다.
> 완전히 로컬인 VSS 배포를 설정하려면 [VSS 문서의 지침](https://docs.nvidia.com/vss/latest/content/vss_dep_docker_compose_arm.html#local-deployment-single-gpu-dgx-spark)을 따르십시오.

**9.1 NVIDIA API Key 가져오기**

- https://build.nvidia.com/explore/discover에 로그인하십시오.
- 페이지에서 **Get API Key**를 검색하고 클릭하십시오.

**9.2 원격 LLM 배포 디렉토리로 이동**

```bash
cd deploy/docker/remote_llm_deployment/
```

**9.3 환경 변수 구성**

API 키 및 배포 기본 설정으로 환경 파일을 업데이트하십시오. `.env` 파일을 직접 편집하거나 다음 명령을 실행하여 수행할 수 있습니다:

```bash
## 필요한 키로 .env 파일 편집
echo "NVIDIA_API_KEY=<YOUR_NVIDIA_API_KEY>" >> .env
echo "NGC_API_KEY=<YOUR_NGC_API_KEY>" >> .env
echo "DISABLE_CV_PIPELINE=true" >> .env  # CV를 활성화하려면 false로 설정
echo "INSTALL_PROPRIETARY_CODECS=false" >> .env  # CV를 활성화하려면 true로 설정
```

**9.4 VSS 이미지 경로 업데이트**

`.env`에서 `VIA_IMAGE`를 `nvcr.io/nvidia/blueprint/vss-engine-sbsa:2.4.0`으로 업데이트하십시오.

```bash
## .env 파일을 편집하고 VIA_IMAGE를 업데이트
echo "VIA_IMAGE=nvcr.io/nvidia/blueprint/vss-engine-sbsa:2.4.0" >> .env
```

**9.5 모델 구성 검토**

config.yaml 파일에 올바른 원격 엔드포인트가 포함되어 있는지 확인하십시오. NIM의 경우 `https://integrate.api.nvidia.com/v1`로 설정해야 합니다.

```bash
## config.yaml에서 모델 서버 엔드포인트 확인
cat config.yaml | grep -A 10 "model"
```

**9.6 Standard VSS 배포 시작**

```bash
## 하이브리드 배포로 Standard VSS 시작
docker compose up
```

> [!NOTE]
> 이 단계는 컨테이너가 풀되고 서비스가 초기화되므로 몇 분이 걸립니다. VSS 백엔드는 추가 시작 시간이 필요합니다.

**9.7 Standard VSS 배포 검증**

성공적인 배포를 확인하기 위해 VSS UI에 액세스하십시오.

```bash
## VSS UI 접근성 테스트
## Spark 장치에서 로컬로 실행하는 경우 localhost를 사용하십시오:
curl -I http://localhost:9100
## 예상: HTTP 200 응답

## Spark가 원격/액세서리 모드에서 실행 중인 경우 'localhost'를 Spark 장치의 IP 주소 또는 호스트 이름으로 교체하십시오.
## Spark의 IP 주소를 찾으려면 Spark 터미널에서 다음 명령을 실행하십시오:
hostname -I
## 또는 호스트 이름을 가져오려면:
hostname
## 그런 다음 접근성을 테스트하십시오(<SPARK_IP_OR_HOSTNAME>을 실제 값으로 교체):
curl -I http://<SPARK_IP_OR_HOSTNAME>:9100
```

브라우저에서 `http://localhost:9100`을 열어 VSS 인터페이스에 액세스하십시오.

## 10단계. 비디오 처리 워크플로 테스트

배포에 따라 비디오 분석 파이프라인이 작동하는지 확인하기 위한 기본 테스트를 실행하십시오. UI에는 업로드 및 테스트를 위해 미리 채워진 몇 가지 예제 비디오가 함께 제공됩니다

**Event Reviewer 배포의 경우**

Event Reviewer 워크플로에 액세스하고 사용하려면 [여기](https://docs.nvidia.com/vss/latest/content/vss_event_reviewer.html#vss-alert-inspector-ui)의 단계를 따르십시오.
- `http://localhost:7862`에서 CV UI에 액세스하여 비디오를 업로드하고 처리
- `http://localhost:7860`에서 Alert Inspector UI에서 결과 모니터링

**Standard VSS 배포의 경우**

VSS UI - 파일 요약, Q&A 및 알림을 탐색하려면 [여기](https://docs.nvidia.com/vss/latest/content/ui_app.html)의 단계를 따르십시오.
- `http://localhost:9100`에서 VSS 인터페이스에 액세스
- 비디오를 업로드하고 요약 기능 테스트

## 11단계. 정리 및 롤백

VSS 배포를 완전히 제거하고 시스템 리소스를 확보하려면:

> [!WARNING]
> 이렇게 하면 모든 처리된 비디오 데이터 및 분석 결과가 삭제됩니다.

```bash
## Event Reviewer 배포의 경우
cd deploy/docker/event_reviewer/
IS_SBSA=1 IS_AARCH64=1 ALERT_REVIEW_MEDIA_BASE_DIR=/tmp/alert-media-dir docker compose down
cd ../../examples/cv-event-detector/
IS_SBSA=1 IS_AARCH64=1 ALERT_REVIEW_MEDIA_BASE_DIR=/tmp/alert-media-dir docker compose down

## Standard VSS 배포의 경우
cd deploy/docker/remote_llm_deployment/
docker compose down

## 공유 네트워크 제거(Event Reviewer를 사용하는 경우)
docker network rm vss-shared-network

## 임시 미디어 파일 정리 및 캐시 클리너 중지
rm -rf /tmp/alert-media-dir
sudo pkill -f sys_cache_cleaner.sh
```

## 12단계. 다음 단계

VSS를 배포한 후 다음을 수행할 수 있습니다:

**Event Reviewer 배포:**
- 포트 7862의 CV UI를 통해 비디오 파일 업로드
- 자동화된 이벤트 감지 및 검토 모니터링
- 포트 7860의 Alert Inspector UI에서 분석 결과 검토
- 사용자 정의 이벤트 감지 규칙 및 임계값 구성

**Standard VSS 배포:**
- 포트 9100에서 전체 VSS 기능에 액세스
- 비디오 요약 및 Q&A 기능 테스트
- 지식 그래프 및 그래프 데이터베이스 구성
- 기존 비디오 처리 워크플로와 통합

## 문제 해결

| 증상 | 원인 | 해결 방법 |
|---------|--------|-----|
| "pull access denied"로 컨테이너 시작 실패 | nvcr.io 자격 증명 누락 또는 잘못됨 | 유효한 자격 증명으로 `docker login nvcr.io` 재실행 |
| 네트워크 생성 실패 | 같은 이름의 기존 네트워크 | `docker network rm vss-shared-network`를 실행한 다음 재생성 |
| 서비스 통신 실패 | 잘못된 환경 변수 | `IS_SBSA=1 IS_AARCH64=1`이 올바르게 설정되었는지 확인 |
| 웹 인터페이스에 액세스할 수 없음 | 서비스가 아직 시작 중이거나 포트 충돌 | 2-3분 기다리고 `docker ps`로 컨테이너 상태 확인 |

> [!NOTE]
> DGX Spark는 GPU와 CPU 간의 동적 메모리 공유를 가능하게 하는 통합 메모리 아키텍처(UMA)를 사용합니다.
> UMA를 활용하도록 업데이트되는 많은 애플리케이션에서 여전히 DGX Spark의 메모리 용량 내에서도
> 메모리 문제가 발생할 수 있습니다. 그런 경우 다음과 같이 버퍼 캐시를 수동으로 플러시하십시오:
```bash
sudo sh -c 'sync; echo 3 > /proc/sys/vm/drop_caches'
```
