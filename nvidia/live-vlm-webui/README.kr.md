# Live VLM WebUI

> 웹캠 스트리밍을 사용한 실시간 Vision Language Model 상호 작용

## 목차

- [개요](#개요)
- [지침](#지침)
  - [명령줄 옵션](#명령줄-옵션)
  - [SSL 인증서 수락](#ssl-인증서-수락)
  - [카메라 권한 부여](#카메라-권한-부여)
  - [성능 최적화 팁](#성능-최적화-팁)
- [문제 해결](#문제-해결)

---

## 개요

## 기본 아이디어

Live VLM WebUI는 실시간 Vision Language Model(VLM) 상호 작용 및 벤치마킹을 위한 범용 웹 인터페이스입니다. 웹캠을 모든 VLM 백엔드(Ollama, vLLM, SGLang 또는 클라우드 API)로 직접 스트리밍하고 실시간 AI 기반 분석을 받을 수 있습니다. 이 도구는 VLM 모델 테스트, 다양한 하드웨어 구성 전반의 성능 벤치마킹 및 비전 AI 기능 탐색에 적합합니다.

인터페이스는 WebRTC 기반 비디오 스트리밍, 통합 GPU 모니터링, 사용자 정의 가능한 프롬프트 및 여러 VLM 백엔드에 대한 지원을 제공합니다. DGX Spark의 강력한 Blackwell GPU와 원활하게 작동하여 인상적인 속도로 실시간 비전 추론을 가능하게 합니다.

## 달성할 내용

DGX Spark에서 다음을 수행할 수 있는 완전한 실시간 비전 AI 테스트 환경을 설정합니다:

- 웹캠 비디오를 스트리밍하고 웹 브라우저를 통해 즉각적인 VLM 분석 받기
- 다양한 비전 언어 모델(Gemma 3, Llama Vision, Qwen VL 등) 테스트 및 비교
- 모델이 비디오 프레임을 처리하는 동안 실시간으로 GPU 및 시스템 성능 모니터링
- 다양한 사용 사례(객체 감지, 장면 설명, OCR, 안전 모니터링)에 대한 프롬프트 사용자 정의
- 웹 브라우저로 네트워크의 모든 장치에서 인터페이스 액세스

## 시작하기 전에 알아야 할 사항

- Linux 명령줄 및 터미널 작업에 대한 기본 익숙함
- pip를 사용한 Python 패키지 설치에 대한 기본 지식
- REST API 및 HTTP를 통한 서비스 통신 방법에 대한 기본 지식
- 웹 브라우저 및 네트워크 액세스(IP 주소, 포트)에 대한 익숙함
- 선택 사항: Vision Language Model 및 그 기능에 대한 지식(도움이 되지만 필수는 아님)

## 전제 조건

**하드웨어 요구 사항:**
- 웹캠(노트북 내장 카메라, USB 카메라 또는 카메라가 있는 원격 브라우저)
- Python 패키지 및 모델 다운로드를 위한 최소 10GB 이상의 사용 가능한 저장 공간

**소프트웨어 요구 사항:**
- DGX OS가 설치된 DGX Spark
- Python 3.10 이상(`python3 --version`으로 확인)
- pip 패키지 관리자(`pip --version`으로 확인)
- PyPI에서 Python 패키지를 다운로드하기 위한 네트워크 액세스
- 로컬로 실행되는 VLM 백엔드(Ollama가 가장 쉬움) 또는 클라우드 API 액세스
- `https://<SPARK_IP>:8090`에 대한 웹 브라우저 액세스

**VLM 백엔드 옵션:**
1. **Ollama**(초보자에게 권장) - 설치 및 사용이 쉬움
2. **vLLM** - 프로덕션 워크로드를 위한 더 높은 성능
3. **SGLang** - 대안 고성능 백엔드
4. **NIM** - 최적화된 성능을 위한 NVIDIA Inference Microservices
5. **클라우드 API** - NVIDIA API Catalog, OpenAI 또는 기타 OpenAI 호환 API

## 부속 파일

모든 소스 코드 및 문서는 [Live VLM WebUI GitHub 저장소](https://github.com/NVIDIA-AI-IOT/live-vlm-webui)에서 찾을 수 있습니다.

패키지는 pip를 통해 직접 설치되므로 기본 설치에는 추가 파일이 필요하지 않습니다.

## 시간 및 위험

* **예상 시간:** 20-30분(Ollama 설치 및 모델 다운로드 포함)
  * Live VLM WebUI를 pip를 통해 설치하는 데 5분
  * Ollama를 설치하고 모델을 다운로드하는 데 10-15분(모델 크기에 따라 다름)
  * 구성 및 테스트에 5분
* **위험 수준:** 낮음
  * Python 패키지는 사용자 공간에 설치되어 시스템과 격리됨
  * 시스템 수준 변경이 필요하지 않음
  * 웹 인터페이스 기능을 위해 포트 8090에 액세스할 수 있어야 함
  * 자체 서명된 SSL 인증서에는 브라우저 보안 예외가 필요함
* **롤백:** `pip uninstall live-vlm-webui`로 Python 패키지를 제거합니다. Ollama는 표준 패키지 제거로 제거할 수 있습니다. DGX Spark 구성에 대한 영구적인 변경이 없습니다.
* **최종 업데이트:** 01/02/2026
  * 첫 번째 출판

## 지침

## 1단계. VLM 백엔드로 Ollama 설치

먼저 Vision Language Model을 제공하기 위해 Ollama를 설치하십시오. Ollama는 DGX Spark에서 로컬로 모델을 실행/제공하는 가장 쉬운 옵션 중 하나입니다.

```bash
## Ollama 설치
curl -fsSL https://ollama.com/install.sh | sh

## 설치 확인
ollama --version
```

Ollama는 자동으로 시스템 서비스로 시작되고 Blackwell GPU를 감지합니다.

이제 비전 언어 모델을 다운로드하십시오. 빠른 테스트를 위해 `gemma3:4b`로 시작하는 것이 좋습니다:

```bash
## 경량 모델 다운로드(테스트 권장)
ollama pull gemma3:4b

## 시도할 수 있는 대체 모델:
## ollama pull llama3.2-vision:11b    # 때때로 더 나은 품질, 느림
## ollama pull qwen2.5-vl:7b          #
```

모델 다운로드는 네트워크 속도 및 모델 크기에 따라 5-15분이 걸릴 수 있습니다.

Ollama가 작동하는지 확인하십시오:

```bash
## Ollama API에 액세스할 수 있는지 확인
curl http://localhost:11434/v1/models
```

예상 출력은 다운로드한 모델을 나열하는 JSON 응답을 표시해야 합니다.

## 2단계. Live VLM WebUI 설치

pip를 사용하여 Live VLM WebUI를 설치하십시오:

```bash
pip install live-vlm-webui
```

설치는 필요한 모든 Python 종속성을 다운로드하고 `live-vlm-webui` 명령을 설치합니다.

이제 서버를 시작하십시오:

```bash
## 웹 서버 시작
live-vlm-webui
```

서버는 다음을 수행합니다:
- HTTPS용 SSL 인증서 자동 생성(웹캠 액세스에 필요)
- 포트 8090에서 WebRTC 서버 시작
- Blackwell GPU를 자동으로 감지

서버가 시작되고 다음과 같은 출력을 표시합니다:

```
Starting Live VLM WebUI...
Generating SSL certificates...
GPU detected: NVIDIA GB10 Blackwell

Access the WebUI at:
  Local URL:   https://localhost:8090
  Network URL: https://<YOUR_SPARK_IP>:8090

Press Ctrl+C to stop the server
```

### 명령줄 옵션

Live VLM WebUI는 사용자 정의를 위한 여러 명령줄 옵션을 지원합니다:

```bash
## 다른 포트 지정
live-vlm-webui --port 8091

## 사용자 정의 SSL 인증서 사용
live-vlm-webui --ssl-cert /path/to/cert.pem --ssl-key /path/to/key.pem

## 기본 API 엔드포인트 변경
live-vlm-webui --api-base http://localhost:8000/v1

## 백그라운드에서 실행(선택 사항)
nohup live-vlm-webui > live-vlm.log 2>&1 &
```

## 3단계. 웹 인터페이스 액세스

웹 브라우저를 열고 다음으로 이동하십시오:

```
https://<YOUR_SPARK_IP>:8090
```

`<YOUR_SPARK_IP>`를 DGX Spark의 IP 주소로 바꾸십시오. 다음을 사용하여 찾을 수 있습니다:

```bash
hostname -I | awk '{print $1}'
```

**중요:** 최신 브라우저에서는 웹캠 액세스를 위해 보안 연결이 필요하므로 `https://`(not `http://`)를 사용해야 합니다.

### SSL 인증서 수락

애플리케이션은 자체 서명된 SSL 인증서를 사용하므로 브라우저에 보안 경고가 표시됩니다. 이는 예상되고 안전합니다.

**Chrome/Edge:**
1. "**고급**" 버튼 클릭
2. "**\<YOUR_SPARK_IP\> (안전하지 않음)로 이동**" 클릭

**Firefox:**
1. "**고급...**" 클릭
2. "**위험을 감수하고 계속**" 클릭

### 카메라 권한 부여

메시지가 표시되면 웹사이트가 카메라에 액세스하도록 허용하십시오. 웹캠 스트림이 인터페이스에 나타나야 합니다.

> [!TIP]
> **원격 액세스 권장:** 최상의 경험을 위해 동일한 네트워크의 노트북 또는 PC에서 웹 인터페이스에 액세스하십시오. DGX Spark에 로컬로 액세스하는 것보다 더 나은 브라우저 성능과 내장 웹캠 액세스를 제공합니다.

## 4단계. VLM 설정 구성

인터페이스는 로컬 VLM 백엔드를 자동으로 감지합니다. 왼쪽 사이드바의 **VLM API Configuration** 섹션에서 구성을 확인하십시오:

**API Endpoint:** `http://localhost:11434/v1`(Ollama)을 표시해야 합니다

**Model Selection:** 드롭다운을 클릭하고 다운로드한 모델(예: `gemma3:4b`)을 선택하십시오

**선택적 설정:**
- **Max Tokens:** 응답 길이 제어(기본값: 512, 더 빠른 응답을 위해 100-200으로 줄임)
- **Frame Processing Interval:** 분석 사이에 건너뛸 프레임 수(기본값: 30프레임, 느린 속도를 위해 증가)

### 성능 최적화 팁

DGX Spark Blackwell GPU에서 최상의 성능을 얻으려면:

- **모델 선택:** `gemma3:4b`는 1-2초/프레임을 제공하고 `llama3.2-vision:11b`는 더 느린 속도를 제공합니다.
- **Frame Interval:** 편안한 시청을 위해 60프레임(30fps에서 2초) 이상으로 설정
- **Max Tokens:** 더 빠른 응답을 위해 100으로 줄임

> [!NOTE]
> DGX Spark는 GPU와 CPU 간의 동적 메모리 공유를 가능하게 하는 통합 메모리 아키텍처(UMA)를 사용합니다.
> UMA를 활용하도록 업데이트되는 많은 애플리케이션에서 여전히 DGX Spark의 메모리 용량 내에서도
> 메모리 문제가 발생할 수 있습니다. 그런 경우 다음과 같이 버퍼 캐시를 수동으로 플러시하십시오:
```bash
sudo sh -c 'sync; echo 3 > /proc/sys/vm/drop_caches'
```

## 5단계. 비디오 분석 시작

녹색 "**Start Camera and Start VLM Analysis**" 버튼을 클릭하십시오.

인터페이스는 다음을 수행합니다:
1. WebRTC를 통해 웹캠 스트리밍 시작
2. 프레임 처리 및 VLM으로 전송 시작
3. AI 분석 결과를 실시간으로 표시
4. 하단에 GPU/CPU/RAM 메트릭 표시

다음을 볼 수 있습니다:
- 오른쪽의 **라이브 비디오 피드**(미러 토글 포함)
- 비디오 또는 정보 상자에 오버레이된 **VLM 분석 결과**
- 지연 시간 및 프레임 수를 표시하는 **성능 메트릭**
- Blackwell GPU 사용률 및 VRAM 사용량을 표시하는 **GPU 모니터링**

DGX Spark의 Blackwell GPU를 사용하면 `gemma3:4b` 및 `llama3.2-vision:11b`에 대해 **프레임당 1-2초**의 추론 시간을 볼 수 있습니다.

## 6단계. 프롬프트 사용자 정의

왼쪽 사이드바 하단의 **Prompt Editor**를 사용하면 VLM이 분석하는 내용을 사용자 정의할 수 있습니다.

**Quick Prompts** - 사용할 준비가 된 8개의 사전 설정:
- **Scene Description** - "Describe what you see in this image in one sentence."
- **Object Detection** - "List all objects you can see in this image, separated by commas."
- **Activity Recognition** - "Describe the person's activity and what they are doing."
- **Safety Monitoring** - "Are there any safety hazards visible? Answer with 'ALERT: description' or 'SAFE'."
- **OCR / Text Recognition** - "Read and transcribe any text visible in the image."
- 그리고 더...

**Custom Prompts** - 직접 입력:

실시간 CSV 출력을 위해 이것을 시도하십시오(다운스트림 애플리케이션에 유용):

```
List all objects you can see in this image, separated by commas.
Do not include explanatory text. Output only the comma-separated list.
```

VLM은 즉시 다음 프레임 분석에 새 프롬프트 사용을 시작합니다. 이를 통해 라이브 결과를 보면서 프롬프트를 반복하고 개선할 수 있는 실시간 "프롬프트 엔지니어링"이 가능합니다.

## 7단계. 다양한 모델 테스트(선택 사항)

모델을 비교하고 싶으십니까? 다른 모델을 다운로드하고 전환하십시오:

```bash
## 다른 모델 다운로드
ollama pull llama3.2-vision:11b

## 모델이 웹 인터페이스의 Model 드롭다운에 나타납니다
```

웹 인터페이스에서:
1. VLM 분석 중지(실행 중인 경우)
2. **Model** 드롭다운에서 새 모델을 선택합니다
3. VLM 분석을 다시 시작합니다

DGX Spark의 Blackwell GPU에서 모델 간의 추론 속도 및 품질을 비교하십시오.

## 8단계. 성능 모니터링

하단 섹션에는 실시간 시스템 메트릭이 표시됩니다:

- **GPU Usage** - Blackwell GPU 사용률 백분율
- **VRAM Usage** - GPU 메모리 소비
- **CPU Usage** - 시스템 CPU 사용률
- **System RAM** - 메모리 사용량

이러한 메트릭을 사용하여:
- 동일한 하드웨어에서 다양한 모델 벤치마킹
- 성능 병목 현상 식별
- 사용 사례에 맞게 설정 최적화

## 9단계. 정리

완료되면 서버가 실행 중인 터미널에서 `Ctrl+C`로 서버를 중지하십시오.

Live VLM WebUI를 완전히 제거하려면:

```bash
pip uninstall live-vlm-webui
```

Ollama 설치 및 다운로드한 모델은 향후 사용을 위해 사용 가능한 상태로 유지됩니다.

Ollama도 제거하려면(선택 사항):

```bash
## Ollama 제거
sudo systemctl stop ollama
sudo rm /usr/local/bin/ollama
sudo rm -rf /usr/share/ollama

## Ollama 모델 제거(선택 사항)
rm -rf ~/.ollama
```

## 10단계. 다음 단계

이제 Live VLM WebUI가 실행되고 있으므로 다음 사용 사례를 탐색하십시오:

**모델 벤치마킹:**
- DGX Spark에서 여러 모델(Gemma 3, Llama Vision, Qwen VL) 테스트
- 추론 지연 시간, 정확도 및 GPU 사용률 비교
- 구조화된 출력 기능(JSON, CSV) 평가

**애플리케이션 프로토타입 제작:**
- 자체 VLM 애플리케이션 구축을 위한 참조로 웹 인터페이스 사용
- 로봇 비전을 위해 ROS 2와 통합
- 보안 모니터링을 위해 RTSP IP 카메라에 연결(베타 기능)

**클라우드 API 통합:**
- 로컬 Ollama에서 클라우드 API(NVIDIA API Catalog, OpenAI)로 전환
- 엣지와 클라우드 추론 성능 및 비용 비교
- 하이브리드 배포 테스트

NVIDIA API Catalog 또는 다른 클라우드 API를 사용하려면:

1. **VLM API Configuration** 섹션에서 **API Base URL**을 다음으로 변경하십시오:
   - NVIDIA API Catalog: `https://integrate.api.nvidia.com/v1`
   - OpenAI: `https://api.openai.com/v1`
   - 기타: 사용자 정의 엔드포인트

2. 나타나는 필드에 **API Key**를 입력하십시오

3. 드롭다운에서 모델을 선택하십시오(목록은 API에서 가져옴)

**고급 구성:**
- 더 높은 처리량을 위해 vLLM, SGLang 또는 NIM 백엔드 사용
- 최적화된 NVIDIA 특정 성능을 위해 NIM 설정
- 특정 사용 사례에 맞게 Python 백엔드 사용자 정의

더 고급 사용법은 GitHub의 [전체 문서](https://github.com/NVIDIA-AI-IOT/live-vlm-webui/tree/main/docs)를 참조하십시오.

최신 알려진 문제에 대해서는 [DGX Spark 사용자 가이드](https://docs.nvidia.com/dgx/dgx-spark/known-issues.html) 및 [Live VLM WebUI 문제 해결 가이드](https://github.com/NVIDIA-AI-IOT/live-vlm-webui/blob/main/docs/troubleshooting.md)를 참조하십시오.

## 문제 해결

| 증상 | 원인 | 해결 방법 |
|---------|-------|-----|
| pip install이 "error: externally-managed-environment" 표시 | Python 3.12+는 시스템 전체 pip 설치를 방지 | 가상 환경 사용: `python3 -m venv live-vlm-env && source live-vlm-env/bin/activate && pip install live-vlm-webui` |
| 브라우저에 "연결이 비공개가 아닙니다" 경고 표시 | 애플리케이션이 자체 서명된 SSL 인증서 사용 | "고급" → "\<IP\>(안전하지 않음)로 이동" 클릭 - 이는 안전하고 예상되는 동작입니다 |
| 카메라에 액세스할 수 없거나 "권한 거부" | 브라우저는 웹캠 액세스를 위해 HTTPS 필요 | `https://`(not `http://`)를 사용하고 있는지 확인하십시오. 자체 서명된 인증서 경고를 수락하고 메시지가 표시되면 카메라 권한을 부여하십시오 |
| "VLM 연결 실패" 또는 "연결 거부" | Ollama 또는 VLM 백엔드가 실행되지 않음 | `curl http://localhost:11434/v1/models`로 Ollama가 실행 중인지 확인하십시오. 실행 중이 아니면 `sudo systemctl start ollama`로 시작하십시오 |
| VLM 응답이 매우 느림(프레임당 >5초) | 사용 가능한 VRAM에 비해 모델이 너무 크거나 구성이 잘못됨 | 더 작은 모델을 시도하십시오(`gemma3:4b` 대신 더 큰 모델). Frame Processing Interval을 60+ 프레임으로 증가시키십시오. Max Tokens을 100-200으로 줄이십시오 |
| GPU 통계가 모든 메트릭에 대해 "N/A" 표시 | NVML을 사용할 수 없거나 GPU 드라이버 문제 | `nvidia-smi`로 GPU 액세스를 확인하십시오. NVIDIA 드라이버가 올바르게 설치되어 있는지 확인하십시오 |
| 모델 드롭다운에 "사용 가능한 모델 없음" | API 엔드포인트가 잘못되었거나 모델이 다운로드되지 않음 | Ollama의 경우 API 엔드포인트가 `http://localhost:11434/v1`인지 확인하십시오. `ollama pull gemma3:4b`로 모델을 다운로드하십시오 |
| 서버가 "port already in use"로 시작하지 못함 | 포트 8090이 이미 다른 서비스에서 사용 중 | 충돌하는 서비스를 중지하거나 `--port` 플래그를 사용하여 다른 포트를 지정하십시오: `live-vlm-webui --port 8091` |
| 네트워크의 원격 브라우저에서 액세스할 수 없음 | 방화벽이 포트 8090을 차단하거나 잘못된 IP 주소 | 방화벽이 포트 8090을 허용하는지 확인하십시오: `sudo ufw allow 8090`. `hostname -I` 명령에서 올바른 IP를 사용하십시오 |
| 비디오 스트림이 지연되거나 고정됨 | 네트워크 문제 또는 브라우저 성능 | Chrome 또는 Edge 브라우저를 사용하십시오. 로컬이 아닌 네트워크의 별도 PC에서 액세스하십시오. 네트워크 대역폭을 확인하십시오 |
| 예기치 않은 언어로 분석 결과 | 모델이 다국어를 지원하고 프롬프트에서 언어를 감지 | 프롬프트에서 출력 언어를 명시적으로 지정하십시오: "Answer in English: describe what you see" |
| 종속성 오류로 pip install 실패 | Python 패키지 버전 충돌 | `--user` 플래그와 함께 설치해 보십시오: `pip install --user live-vlm-webui` |
| 설치 후 `live-vlm-webui` 명령을 찾을 수 없음 | 바이너리 경로가 PATH에 없음 | `~/.local/bin`을 PATH에 추가하십시오: `export PATH="$HOME/.local/bin:$PATH"` 그런 다음 `source ~/.bashrc` 실행 |
| 카메라가 작동하지만 VLM 분석 결과가 나타나지 않고 브라우저에 InvalidStateError 표시 | 원격 시스템에서 SSH 포트 포워딩을 통해 액세스 | WebRTC는 직접 네트워크 연결이 필요하고 SSH 터널을 통해 작동하지 않습니다(SSH는 TCP만 전달하고 WebRTC는 UDP가 필요함). **해결 방법 1**: 서버와 동일한 네트워크의 브라우저에서 웹 UI에 직접 액세스하십시오. **해결 방법 2**: 서버 시스템의 브라우저를 직접 사용하십시오. **해결 방법 3**: X11 포워딩(`ssh -X`)을 사용하여 브라우저를 원격으로 표시하십시오 |
