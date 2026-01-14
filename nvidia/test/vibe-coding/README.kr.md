# VS Code에서 Vibe 코딩

> Ollama와 Continue를 사용하여 DGX Spark를 로컬 또는 원격 Vibe 코딩 어시스턴트로 활용하기


## 목차

- [개요](#개요)
  - [달성할 목표](#달성할-목표)
  - [사전 요구사항](#사전-요구사항)
  - [소요 시간 및 위험도](#소요-시간-및-위험도)
- [사용 방법](#사용-방법)
- [문제 해결](#문제-해결)

---

## 개요

## 기본 아이디어

이 플레이북은 DGX Spark를 **Vibe 코딩 어시스턴트**로 설정하는 과정을 안내합니다 — Continue.dev를 사용하여 VSCode의 로컬 또는 원격 코딩 도우미로 활용할 수 있습니다.
이 가이드는 **Ollama**와 **GPT-OSS 120B**를 사용하여 VSCode에 코딩 어시스턴트를 쉽게 배포할 수 있도록 합니다. 또한 DGX Spark와 Ollama가 로컬 네트워크를 통해 코딩 어시스턴트를 제공할 수 있도록 하는 고급 지침도 포함되어 있습니다. 이 가이드는 **새로 설치된** OS를 기준으로 작성되었습니다. OS가 새로 설치되지 않은 상태에서 문제가 발생하면 문제 해결 탭을 참조하세요.

### 달성할 목표

다음과 같은 기능을 갖춘 완전히 구성된 DGX Spark 시스템을 구축하게 됩니다:
- Ollama를 통한 로컬 코드 지원 실행
- Continue 및 VSCode 통합을 위한 원격 모델 서빙
- 통합 메모리를 사용하여 GPT-OSS 120B와 같은 대형 LLM 호스팅

### 사전 요구사항

- DGX Spark (128GB 통합 메모리 권장)
- **Ollama** 및 원하는 LLM (예: `gpt-oss:120b`)
- **VSCode**
- **Continue** VSCode 확장 프로그램
- 모델 다운로드를 위한 인터넷 연결
- Linux 터미널 열기, 명령어 복사 및 붙여넣기에 대한 기본 숙지
- sudo 접근 권한 보유
- 선택 사항: 원격 접근 구성을 위한 방화벽 제어

### 소요 시간 및 위험도
* **소요 시간:** 약 30분
* **위험도:** 네트워크 문제로 인한 데이터 다운로드 지연 또는 실패
* **롤백:** 정상 사용 중 영구적인 시스템 변경 사항 없음
* **최종 업데이트:** 2025년 10월 21일
  * 최초 게시

## 사용 방법

## 1단계. Ollama 설치

다음 명령어를 사용하여 최신 버전의 Ollama를 설치합니다:

```bash
curl -fsSL https://ollama.com/install.sh | sh
```
서비스가 실행되면 원하는 모델을 다운로드합니다:

```bash
ollama pull gpt-oss:120b
```

## 2단계. (선택 사항) 원격 접근 활성화

원격 연결을 허용하려면 (예: VSCode와 Continue를 사용하는 워크스테이션에서), Ollama systemd 서비스를 수정합니다:

```bash
sudo systemctl edit ollama
```

주석 처리된 섹션 아래에 다음 줄을 추가합니다:

```ini
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"
Environment="OLLAMA_ORIGINS=*"
```

서비스를 다시 로드하고 재시작합니다:

```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

방화벽을 사용하는 경우 11434 포트를 엽니다:

```bash
sudo ufw allow 11434/tcp
```

워크스테이션이 DGX Spark의 Ollama 서버에 연결할 수 있는지 확인합니다:

  ```bash
  curl -v http://YOUR_SPARK_IP:11434/api/version
  ```
 **YOUR_SPARK_IP**를 DGX Spark의 IP 주소로 교체하세요.
 연결이 실패하면 문제 해결 탭을 참조하세요.

## 3단계. VSCode 설치

DGX Spark (ARM 기반)의 경우, VSCode를 다운로드하고 설치합니다:
  https://code.visualstudio.com/download 로 이동하여 Linux ARM64 버전의 VSCode를 다운로드합니다.
  다운로드가 완료되면 다운로드한 패키지 이름을 확인합니다. 다음 명령어에서 DOWNLOADED_PACKAGE_NAME을 해당 이름으로 교체하여 사용합니다.
```bash
sudo dpkg -i DOWNLOADED_PACKAGE_NAME
```

원격 워크스테이션을 사용하는 경우, **시스템 아키텍처에 적합한 VSCode를 설치**하세요.

## 4단계. Continue.dev 확장 프로그램 설치

VSCode를 열고 Marketplace에서 **Continue.dev**를 설치합니다:
- VSCode에서 **확장 프로그램 보기**로 이동
- [Continue.dev](https://www.continue.dev/)에서 게시한 **Continue**를 검색하여 확장 프로그램을 설치합니다.
설치 후 오른쪽 바에서 Continue 아이콘을 클릭합니다.

## 5단계. 로컬 추론 설정
- `Or, configure your own models` 클릭
- `Click here to view more providers` 클릭
- Provider로 `Ollama` 선택
- Model은 `Autodetect` 선택
- 테스트 프롬프트를 전송하여 추론 테스트

다운로드한 모델이 이제 추론의 기본 모델이 됩니다 (예: `gpt-oss:120b`).

## 6단계. DGX Spark의 Ollama 서버에 연결하도록 워크스테이션 설정

VSCode를 실행하는 워크스테이션을 원격 DGX Spark 인스턴스에 연결하려면 해당 워크스테이션에서 다음을 완료해야 합니다:
  - 4단계의 지침에 따라 Continue 설치
  - 왼쪽 창에서 `Continue` 아이콘 클릭
  - `Or, configure your own models` 클릭
  - `Click here to view more providers` 클릭
  - Provider로 `Ollama` 선택
  - Model로 `Autodetect` 선택

Continue는 로컬로 호스팅된 Ollama 서버에 연결을 시도하므로 모델 감지에 **실패할 것입니다**.
  - Continue 창의 오른쪽 상단에서 `톱니바퀴` 아이콘을 찾아 클릭합니다.
  - 왼쪽 창에서 **Models** 클릭
  - **Chat** 아래 첫 번째 드롭다운 메뉴 옆의 톱니바퀴 아이콘 클릭
  - Continue의 `config.yaml`이 열립니다. DGX Spark의 IP 주소를 확인합니다.
  - 구성을 다음으로 교체합니다. **YOUR_SPARK_IP**는 DGX Spark의 IP로 교체해야 합니다.


```yaml
name: Config
version: 1.0.0
schema: v1

assistants:
  - name: default
    model: OllamaSpark

models:
  - name: OllamaSpark
    provider: ollama
    model: gpt-oss:120b
    apiBase: http://YOUR_SPARK_IP:11434
    title: gpt-oss:120b
    roles:
      - chat
      - edit
      - autocomplete
```

`YOUR_SPARK_IP`를 DGX Spark의 IP 주소로 교체하세요.
원격으로 호스팅하려는 다른 Ollama 모델에 대해서도 추가 모델 항목을 추가할 수 있습니다.

## 문제 해결

| 증상 | 원인 | 해결 방법 |
|---------|-------|-----|
|Ollama가 시작되지 않음|GPU 드라이버가 올바르게 설치되지 않았을 수 있음|터미널에서 `nvidia-smi`를 실행합니다. 명령어가 실패하면 DGX Dashboard에서 DGX Spark 업데이트를 확인하세요.|
|Continue가 네트워크를 통해 연결할 수 없음|11434 포트가 열려있지 않거나 접근할 수 없을 수 있음|`ss -tuln \| grep 11434` 명령어를 실행합니다. 출력에 ` tcp   LISTEN 0      4096               *:11434            *:*  `가 표시되지 않으면 2단계로 돌아가 ufw 명령어를 실행하세요.|
|Continue가 로컬에서 실행 중인 Ollama 모델을 감지할 수 없음|구성이 제대로 설정되지 않았거나 감지되지 않음|`/etc/systemd/system/ollama.service.d/override.conf` 파일에서 `OLLAMA_HOST` 및 `OLLAMA_ORIGINS`를 확인합니다. `OLLAMA_HOST`와 `OLLAMA_ORIGINS`가 올바르게 설정되어 있으면 이 줄들을 `~/.bashrc` 파일에 추가하세요.|
|높은 메모리 사용량|모델 크기가 너무 큼|`nvidia-smi`로 다른 대형 모델이나 컨테이너가 실행 중이지 않은지 확인합니다. 가벼운 사용을 위해 `gpt-oss:20b`와 같은 작은 모델을 사용하세요.|

> [!NOTE]
> DGX Spark는 통합 메모리 아키텍처(UMA)를 사용하여 GPU와 CPU 간의 동적 메모리 공유를 가능하게 합니다.
> 많은 애플리케이션이 아직 UMA를 활용하도록 업데이트 중이기 때문에, DGX Spark의 메모리 용량 내에 있어도
> 메모리 문제가 발생할 수 있습니다. 이런 경우 다음 명령어로 버퍼 캐시를 수동으로 플러시하세요:
```bash
sudo sh -c 'sync; echo 3 > /proc/sys/vm/drop_caches'
```
