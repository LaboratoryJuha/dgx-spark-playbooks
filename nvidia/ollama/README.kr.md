# Ollama

> Ollama 설치 및 사용

## 목차

- [개요](#개요)
- [지침](#지침)
- [문제 해결](#문제-해결)

---

## 개요

## 기본 아이디어

이 플레이북은 NVIDIA Sync의 Custom Apps 기능을 사용하여 NVIDIA Spark 장치에서 실행되는 Ollama 서버에 대한 원격 액세스를 설정하는 방법을 보여줍니다. Spark 장치에 Ollama를 설치하고, NVIDIA Sync를 구성하여 SSH 터널을 만들고, 로컬 시스템에서 Ollama API에 액세스합니다. 이를 통해 네트워크에서 포트를 노출할 필요 없이 노트북에서 보안 SSH 터널을 통해 AI 추론을 가능하게 합니다.

## 달성할 내용

Blackwell 아키텍처가 있는 NVIDIA Spark에서 Ollama가 실행되고 로컬 노트북에서 API 호출을 통해 액세스할 수 있습니다. 이 설정을 통해 복잡한 네트워크 구성 없이 Spark 장치의 강력한 GPU 기능을 활용하여 대형 언어 모델 추론을 위해 Ollama API와 통신하는 애플리케이션을 구축하거나 로컬 시스템에서 도구를 사용할 수 있습니다.

## 시작하기 전에 알아야 할 사항

- SSH 연결 및 시스템 트레이 애플리케이션 작업
- API 테스트를 위한 터미널 명령 및 cURL에 대한 기본 익숙함
- REST API 개념 및 JSON 형식에 대한 이해
- 컨테이너 환경 및 GPU 가속 워크로드에 대한 경험

## 전제 조건

- 네트워크에 설정되고 연결된 DGX Spark 장치
- NVIDIA Sync가 설치되고 Spark에 연결됨
- API 호출 테스트를 위한 로컬 시스템의 터미널 액세스



## 시간 및 위험

* **소요 시간**: 초기 설정 10-15분, 모델 다운로드 2-3분(모델 크기에 따라 다름)

* **위험 수준**: 낮음 - 시스템 수준 변경 없음, 사용자 정의 앱을 중지하여 쉽게 되돌릴 수 있음

* **롤백**: NVIDIA Sync에서 사용자 정의 앱을 중지하고 필요한 경우 표준 패키지 제거로 Ollama를 제거

* **최종 업데이트:** 10/12/2025
  * 첫 번째 출판

## 지침

## 1단계. Ollama 설치 상태 확인

**설명**: NVIDIA Spark 장치에 Ollama가 이미 설치되어 있는지 확인합니다. NVIDIA Sync 터미널을 통해 Spark 장치에서 실행하여 설치가 필요한지 확인합니다.

```bash
ollama --version
```

버전 정보가 표시되면 3단계로 건너뛰십시오. "command not found"가 표시되면 2단계로 진행하십시오.

## 2단계. Spark 장치에 Ollama 설치

**설명**: 공식 설치 스크립트를 사용하여 Ollama를 다운로드하고 설치합니다. Spark 장치에서 실행되며 Ollama 바이너리 및 서비스 구성 요소를 설치합니다.

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

설치가 완료될 때까지 기다립니다. 설치가 성공했음을 나타내는 출력이 표시되어야 합니다.

## 3단계. 언어 모델 다운로드 및 확인

**설명**: Spark 장치에 언어 모델을 가져옵니다. 모델 파일을 다운로드하고 추론에 사용할 수 있도록 합니다. 예제는 Blackwell GPU에 최적화된 Qwen2.5 30B를 사용합니다.

```bash
ollama pull qwen2.5:32b
```

예상 출력:
```
pulling manifest
pulling 58574f2e94b9: 100% ████████████████████████████  18 GB
pulling 53e4ea15e8f5: 100% ████████████████████████████ 1.5 KB
pulling d18a5cc71b84: 100% ████████████████████████████  11 KB
pulling cff3f395ef37: 100% ████████████████████████████  120 B
pulling 3cdc64c2b371: 100% ████████████████████████████  494 B
verifying sha256 digest
writing manifest
success
```

## 4단계. NVIDIA Sync 설정 액세스

**설명**: 로컬 시스템에서 NVIDIA Sync 구성 인터페이스를 열어 새 사용자 정의 애플리케이션 터널을 추가합니다. 로컬 노트북/워크스테이션에서 실행됩니다.

1. 시스템 트레이/작업 표시줄에서 NVIDIA Sync 로고를 클릭합니다
2. 오른쪽 상단 모서리의 톱니바퀴 아이콘을 클릭하여 설정 창을 엽니다
3. "Custom" 탭을 클릭합니다

## 5단계. NVIDIA Sync에서 Ollama 사용자 정의 앱 구성

**설명**: 포트 11434에서 실행 중인 Ollama 서버에 대한 SSH 터널을 설정할 새 사용자 정의 애플리케이션 항목을 만듭니다. 이 구성은 로컬 시스템에서 실행됩니다.

1. "Add New" 버튼을 클릭합니다
2. 다음 값으로 양식을 작성합니다:
  - **Name**: `Ollama Server`
  - **Port**: `11434`
  - **Auto open in browser**: 선택 해제(이것은 웹 인터페이스가 아닌 API임)
  - **Start Script**: 비워 둠
3. "Add"를 클릭합니다

이제 새 Ollama Server 항목이 NVIDIA Sync 사용자 정의 앱 목록에 표시되어야 합니다.

## 6단계. SSH 터널 시작

**설명**: SSH 터널을 활성화하여 원격 Ollama 서버를 로컬 시스템에서 액세스할 수 있도록 합니다. localhost:11434에서 Spark 장치로의 보안 연결을 만듭니다.

1. 시스템 트레이/작업 표시줄에서 NVIDIA Sync 로고를 클릭합니다
2. "Custom" 섹션에서 "Ollama Server"를 클릭합니다

NVIDIA Sync에서 연결 상태 표시기가 표시되면 터널이 활성화됩니다.

## 7단계. API 연결 확인

**설명**: 로컬 시스템에서 Ollama API 연결을 테스트하여 터널이 올바르게 작동하는지 확인합니다. 로컬 노트북 터미널에서 실행됩니다.

```bash
curl http://localhost:11434/api/chat -d '{
  "model": "qwen2.5:32b",
  "messages": [{
    "role": "user",
    "content": "Write me a haiku about GPUs and AI."
  }],
  "stream": false
}'
```

예상 응답 형식:
```json
{
  "model": "qwen2.5:32b",
  "created_at": "2024-01-15T12:30:45.123Z",
  "message": {
    "role": "assistant",
    "content": "Silicon power flows\nThrough circuits, dreams become real\nAI awakens"
  },
  "done": true
}
```

## 8단계. 추가 API 엔드포인트 테스트

**설명**: 다른 Ollama API 기능을 확인하여 전체 작동을 보장합니다. 이러한 명령은 로컬 시스템에서 실행되며 다양한 API 기능을 테스트합니다.

모델 목록 테스트:
```bash
curl http://localhost:11434/api/tags
```

스트리밍 응답 테스트:
```bash
curl -N http://localhost:11434/api/chat -d '{
  "model": "qwen2.5:32b",
  "messages": [{"role": "user", "content": "Count to 5 slowly"}],
  "stream": true
}'
```

## 9단계. 정리 및 롤백

**설명**: 설정을 제거하고 원래 상태로 되돌리는 방법입니다.

터널을 중지하려면:
1. NVIDIA Sync를 열고 "Ollama Server"를 클릭하여 비활성화합니다

사용자 정의 앱을 제거하려면:
1. NVIDIA Sync 설정 → Custom 탭을 엽니다
2. "Ollama Server"를 선택하고 "Remove"를 클릭합니다

> [!WARNING]
> Spark 장치에서 Ollama를 완전히 제거하려면:

```bash
sudo systemctl stop ollama
sudo systemctl disable ollama
sudo rm /usr/local/bin/ollama
sudo rm -rf /usr/share/ollama
sudo userdel ollama
```

이렇게 하면 모든 Ollama 파일과 다운로드한 모델이 제거됩니다.

## 10단계. 다음 단계

**설명**: 작동하는 Ollama 설정으로 추가 기능 및 통합 옵션을 탐색합니다.

[Ollama 라이브러리](https://ollama.com/library)에서 다양한 모델을 테스트하십시오:
```bash
ollama pull llama3.1:8b
ollama pull codellama:13b
ollama pull phi3.5:3.8b
```

NVIDIA Sync를 통해 사용 가능한 DGX Dashboard를 사용하여 추론 중 GPU 및 시스템 사용량을 모니터링하십시오.

선호하는 프로그래밍 언어의 HTTP 클라이언트 라이브러리와 통합하여 Ollama API를 사용하는 애플리케이션을 구축하십시오.

## 문제 해결

| 증상 | 원인 | 해결 방법 |
|---------|--------|-----|
| localhost:11434에서 "Connection refused" | SSH 터널이 활성화되지 않음 | NVIDIA Sync 사용자 정의 앱에서 Ollama Server를 시작합니다 |
| 디스크 공간 오류로 모델 다운로드 실패 | Spark의 저장 공간 부족 | 공간을 확보하거나 더 작은 모델을 선택하십시오(예: qwen2.5:7b) |
| 설치 후 Ollama 명령을 찾을 수 없음 | 설치 경로가 PATH에 없음 | 터미널 세션을 다시 시작하거나 `source ~/.bashrc`를 실행하십시오 |
| API가 "model not found" 오류 반환 | 모델이 가져오지 않았거나 이름이 잘못됨 | `ollama list`를 실행하여 사용 가능한 모델을 확인하십시오 |
| Spark에서 추론이 느림 | GPU 메모리에 비해 모델이 너무 큼 | 더 작은 모델을 시도하거나 `nvidia-smi`로 GPU 메모리를 확인하십시오 |
