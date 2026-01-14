# 로컬 네트워크 액세스 설정

> NVIDIA Sync가 SSH 액세스 설정 및 구성을 지원합니다

## 목차

- [개요](#개요)
- [NVIDIA Sync로 연결](#nvidia-sync로-연결)
- [수동 SSH로 연결](#수동-ssh로-연결)
- [문제 해결](#문제-해결)

---

## 개요

## 기본 아이디어

노트북과 같은 다른 시스템에서 주로 작업하고 DGX Spark를 원격 리소스로 사용하려는 경우 이 플레이북은 SSH를 통해 연결하고 작업하는 방법을 보여줍니다. SSH를 사용하면 로컬 시스템에서 DGX Spark의 웹 앱 및 API에 액세스하기 위해 터미널 세션을 안전하게 열거나 포트를 터널링할 수 있습니다.

두 가지 접근 방식이 있습니다: 간소화된 장치 관리를 위한 **NVIDIA Sync(권장)** 또는 직접 명령줄 제어를 위한 **수동 SSH**.

시작하기 전에 이해해야 할 몇 가지 중요한 개념이 있습니다:

**SSH(Secure Shell)**는 신뢰할 수 없는 네트워크를 통해 원격 컴퓨터에 안전하게 연결하기 위한 암호화 프로토콜입니다. 마치 앞에 앉아 있는 것처럼 DGX Spark에서 터미널을 열고, 명령을 실행하고, 파일을 전송하고, 서비스를 관리할 수 있으며 모두 종단간 암호화됩니다.

**SSH 터널링**(포트 포워딩이라고도 함)은 노트북의 포트(예: localhost:8888)를 앱이 수신 대기 중인 DGX Spark의 포트(예: 포트 8888의 JupyterLab)에 안전하게 매핑합니다. 브라우저가 localhost에 연결하고 SSH가 더 넓은 네트워크에 해당 포트를 노출하지 않고 암호화된 연결을 통해 트래픽을 원격 서비스로 전달합니다.

**mDNS(Multicast DNS)**를 사용하면 중앙 DNS 서버 없이 로컬 네트워크에서 장치가 이름으로 서로를 검색할 수 있습니다. DGX Spark는 mDNS를 통해 호스트 이름을 알리므로 IP 주소를 조회하는 대신 `spark-abcd.local`(`.local` 접미사 참고)과 같은 이름을 사용하여 연결할 수 있습니다.

## 달성할 내용

NVIDIA Sync 또는 수동 SSH 구성을 사용하여 DGX Spark 장치에 대한 안전한 SSH 액세스를 설정합니다. NVIDIA Sync는 통합 앱 시작 기능이 있는 장치 관리를 위한 그래픽 인터페이스를 제공하는 반면, 수동 SSH는 포트 포워딩 기능이 있는 직접 명령줄 제어를 제공합니다. 두 접근 방식 모두 터미널 명령을 실행하고, 웹 애플리케이션에 액세스하고, 노트북에서 DGX Spark를 원격으로 관리할 수 있습니다.


## 시작하기 전에 알아야 할 사항

- 기본 터미널/명령줄 사용
- SSH 개념 및 키 기반 인증에 대한 이해
- 호스트 이름, IP 주소 및 포트 포워딩과 같은 네트워크 개념에 대한 익숙함

## 전제 조건

- DGX Spark [장치가 설정](https://docs.nvidia.com/dgx/dgx-spark/first-boot.html)되어 있고 로컬 사용자 계정을 만들었습니다
- 노트북과 DGX Spark가 동일한 네트워크에 있습니다
- DGX Spark 사용자 이름과 비밀번호가 있습니다
- 장치의 mDNS 호스트 이름(빠른 시작 가이드에 인쇄됨) 또는 IP 주소가 있습니다

## 시간 및 위험

- **예상 시간:** 5-10분
- **위험 수준:** 낮음 - SSH 설정에는 자격 증명 구성이 포함되지만 DGX Spark 장치에 대한 시스템 수준 변경은 없습니다
- **롤백:** DGX Spark의 `~/.ssh/authorized_keys`를 편집하여 SSH 키 제거를 수행할 수 있습니다.
- **최종 업데이트:** 10/28/2025
  * 사소한 수정

## NVIDIA Sync로 연결

## 1단계. NVIDIA Sync 설치

NVIDIA Sync는 로컬 네트워크를 통해 컴퓨터를 DGX Spark에 연결하는 데스크톱 앱입니다.
DGX Spark에서 SSH 액세스를 관리하고 개발 도구를 시작할 수 있는 단일 인터페이스를 제공합니다.

시작하려면 컴퓨터에 NVIDIA Sync를 다운로드하고 설치하십시오.

::spark-download

**macOS용**

- 다운로드 후 `nvidia-sync.dmg`를 엽니다
- 앱을 Applications 폴더로 드래그 앤 드롭합니다
- Applications 폴더에서 `NVIDIA Sync`를 엽니다

**Windows용**

- 다운로드 후 설치 프로그램 .exe를 실행합니다
- 설치가 완료되면 NVIDIA Sync가 자동으로 시작됩니다


**Debian/Ubuntu용**

* 패키지 저장소 구성:

  ```
  curl -fsSL  https://workbench.download.nvidia.com/stable/linux/gpgkey  |  sudo tee -a /etc/apt/trusted.gpg.d/ai-workbench-desktop-key.asc
  echo "deb https://workbench.download.nvidia.com/stable/linux/debian default proprietary" | sudo tee -a /etc/apt/sources.list
  ```
* 패키지 목록 업데이트:

  ```
  sudo apt update
  ```
* NVIDIA Sync 설치:

  ```
  sudo apt install nvidia-sync
  ```

## 2단계. 앱 구성

앱은 NVIDIA Sync가 Spark에 자동 연결을 구성하고 시작할 수 있는 노트북에 설치된 데스크톱 프로그램입니다.

설정 창에서 언제든지 앱 선택을 변경할 수 있습니다. "사용할 수 없음"으로 표시된 앱은 사용하기 전에 설치해야 합니다.

**기본 앱:**
- **DGX Dashboard**: 시스템 관리 및 통합 JupyterLab 액세스를 위해 DGX Spark에 사전 설치된 웹 애플리케이션
- **Terminal**: 자동 SSH 연결이 있는 시스템의 내장 터미널

**선택적 앱(별도 설치 필요):**
- **VS Code**: https://code.visualstudio.com/download 에서 다운로드
- **Cursor**: https://cursor.com/downloads 에서 다운로드
- **NVIDIA AI Workbench**: https://www.nvidia.com/workbench 에서 다운로드

## 3단계. DGX Spark 장치 추가

> [!NOTE]
> 연결하려면 호스트 이름 또는 IP 주소를 알아야 합니다.
>
> - 기본 호스트 이름은 상자에 포함된 빠른 시작 가이드에서 찾을 수 있습니다. 예: `spark-abcd.local`
> - 장치에 디스플레이가 연결되어 있는 경우 [DGX Dashboard](http://localhost:11000)의 설정 페이지에서 호스트 이름을 찾을 수 있습니다.
> - `.local`(mDNS) 호스트 이름이 네트워크에서 작동하지 않는 경우 IP 주소를 사용해야 합니다. 이는 Ubuntu의 네트워크 설정 또는 라우터의 관리 콘솔에 로그인하여 찾을 수 있습니다.

마지막으로 양식을 작성하여 DGX Spark를 연결하십시오:

- **Name**: 설명이 포함된 이름 (예: "My DGX Spark")
- **Hostname or IP**: mDNS 호스트 이름 (예: `spark-abcd.local`) 또는 Spark의 IP 주소
- **Username**: DGX Spark 사용자 계정 이름
- **Password**: DGX Spark 사용자 계정 비밀번호

> [!NOTE]
> 비밀번호는 SSH 키 기반 인증을 구성하기 위한 이 초기 설정 중에만 사용됩니다. 설정 완료 후에는 저장되거나 전송되지 않습니다. NVIDIA Sync는 장치에 SSH로 접속하여
> 로컬로 프로비저닝된 SSH 키 쌍을 구성합니다.

"Add" 버튼을 클릭하면 NVIDIA Sync가 자동으로:

1. 노트북에서 SSH 키 쌍을 생성합니다
2. 제공된 사용자 이름과 비밀번호를 사용하여 DGX Spark에 연결합니다
3. 장치의 `~/.ssh/authorized_keys`에 공개 키를 추가합니다
4. 향후 연결을 위해 로컬로 SSH 별칭을 만듭니다
5. 사용자 이름 및 비밀번호 정보를 삭제합니다

> [!IMPORTANT]
> 처음으로 시스템 설정을 완료한 후 장치가 업데이트되고 네트워크에서 사용 가능해지기까지 몇 분이 걸릴 수 있습니다. NVIDIA Sync가 연결에 실패하면 3-4분 기다린 후 다시 시도하십시오.

## 4단계. DGX Spark 액세스

연결되면 NVIDIA Sync가 시스템 트레이/작업 표시줄 애플리케이션으로 나타납니다. NVIDIA Sync
아이콘을 클릭하여 장치 관리 인터페이스를 엽니다.

- **SSH 연결**: 큰 "Connect" 및 "Disconnect" 버튼을 클릭하면 장치에 대한 전체 SSH 연결이 제어됩니다.
- **작업 디렉토리 설정** (선택 사항): NVIDIA Sync를 통해 시작할 때 앱이 열리는
기본 디렉토리를 선택합니다. 이것은 원격 장치의 홈 디렉토리로 기본 설정됩니다.
- **애플리케이션 시작**: 구성된 앱을 클릭하여 DGX Spark에 자동 SSH
연결과 함께 엽니다.
- **포트 사용자 정의** (선택 사항): "Custom Ports"는 장치에서 실행 중인 사용자 정의 웹 앱 또는 API에 대한 액세스를 제공하기 위해 설정 화면에서 구성됩니다.

## 5단계. SSH 설정 확인

NVIDIA Sync는 수동으로 또는 다른 SSH 지원 앱에서 쉽게 액세스할 수 있도록 장치에 대한 SSH 별칭을 만듭니다.

SSH 별칭을 사용하여 로컬 SSH 구성이 올바른지 확인하십시오. 별칭을 사용할 때 비밀번호를 입력하라는 메시지가 표시되지 않아야 합니다:

```bash
## mDNS 호스트 이름을 사용하는 경우 구성됨
ssh <SPARK_HOSTNAME>.local
```

또는

```bash
## IP 주소를 사용하는 경우 구성됨
ssh <IP>
```

DGX Spark에서 연결되었는지 확인하십시오:

```bash
hostname
whoami
```

SSH 세션을 종료하십시오:

```bash
exit
```

## 6단계. 다음 단계

개발 도구를 시작하여 설정을 테스트하십시오:
- NVIDIA Sync 시스템 트레이 아이콘을 클릭합니다.
- "Terminal"을 선택하여 DGX Spark에서 터미널 세션을 엽니다.
- "DGX Dashboard"를 선택하여 JupyterLab을 사용하고 업데이트를 관리합니다.
- [Open WebUI를 사용한 사용자 정의 포트 예제](/spark/open-webui/sync)를 시도해 보십시오.

## 수동 SSH로 연결

## 1단계. SSH 클라이언트 사용 가능 여부 확인

시스템에 SSH 클라이언트가 설치되어 있는지 확인하십시오. 대부분의 최신 운영 체제에는
SSH가 기본적으로 포함되어 있습니다. 터미널에서 다음을 실행하십시오:

```bash
## SSH 클라이언트 버전 확인
ssh -V
```

예상 출력은 OpenSSH 버전 정보를 표시해야 합니다.

## 2단계. 연결 정보 수집

DGX Spark에 필요한 연결 세부 정보를 수집하십시오:

- **Username**: DGX Spark 사용자 계정 이름
- **Password**: DGX Spark 계정 비밀번호
- **Hostname**: 장치의 mDNS 호스트 이름 (빠른 시작 가이드에서 예: `spark-abcd.local`)
- **IP Address**: 아래 설명된 대로 mDNS가 네트워크에서 작동하지 않는 경우에만 필요한 대안

복잡한 기업 환경과 같은 일부 네트워크 구성에서는 mDNS가 예상대로 작동하지 않으며
연결하려면 장치의 IP 주소를 직접 사용해야 합니다. SSH를 시도할 때 명령이 무기한 중단되거나 다음과 같은 오류가 발생하는 상황을 알 수 있습니다:

```
ssh: Could not resolve hostname spark-abcd.local: Name or service not known
```

**mDNS 해석 테스트**

mDNS가 작동하는지 테스트하려면 `ping` 유틸리티를 사용하십시오:

```bash
ping spark-abcd.local
```

mDNS가 작동하고 호스트 이름을 사용하여 SSH를 할 수 있는 경우 다음과 같이 표시됩니다:

```
$ ping -c 3 spark-abcd.local
PING spark-abcd.local (10.9.1.9): 56 data bytes
64 bytes from 10.9.1.9: icmp_seq=0 ttl=64 time=6.902 ms
64 bytes from 10.9.1.9: icmp_seq=1 ttl=64 time=116.335 ms
64 bytes from 10.9.1.9: icmp_seq=2 ttl=64 time=33.301 ms
```

mDNS가 **작동하지 않는** 경우 IP를 직접 사용해야 함을 나타내며 다음과 같이 표시됩니다:

```
$ ping -c 3 spark-abcd.local
ping: cannot resolve spark-abcd.local: Unknown host
```

이 중 어느 것도 작동하지 않는 경우 다음을 수행해야 합니다:
- 라우터의 관리 패널에 로그인하여 IP 주소 찾기
- 디스플레이, 키보드 및 마우스를 연결하여 Ubuntu 데스크톱에서 확인

## 3단계. 초기 연결 테스트

처음으로 DGX Spark에 연결하여 기본 연결을 확인하십시오:

```bash
## mDNS 호스트 이름을 사용하여 연결 (권장)
ssh <YOUR_USERNAME>@<SPARK_HOSTNAME>.local
```

또는

```bash
## 대안: IP 주소를 사용하여 연결
ssh <YOUR_USERNAME>@<DEVICE_IP_ADDRESS>
```

자리 표시자를 실제 값으로 바꾸십시오:
- `<YOUR_USERNAME>`: DGX Spark 계정 이름
- `<SPARK_HOSTNAME>`: `.local` 접미사가 없는 장치 호스트 이름
- `<DEVICE_IP_ADDRESS>`: 장치의 IP 주소

첫 번째 연결에서 호스트 지문 경고가 표시됩니다. `yes`를 입력하고 Enter를 누른 다음
메시지가 표시되면 비밀번호를 입력하십시오.

## 4단계. 원격 연결 확인

연결되면 DGX Spark 장치에 있는지 확인하십시오:

```bash
## 호스트 이름 확인
hostname
## 시스템 정보 확인
uname -a
## 세션 종료
exit
```

## 5단계. 웹 애플리케이션을 위한 SSH 터널링 사용

DGX Spark에서 실행 중인 웹 애플리케이션에 액세스하려면 SSH 포트
포워딩을 사용하십시오. 이 예제에서는 DGX Dashboard 웹 애플리케이션에 액세스합니다.

> [!NOTE]
> DGX Dashboard는 localhost, 포트 11000에서 실행됩니다.

터널을 엽니다:

```bash
## 로컬 포트 11000 → 원격 포트 11000
ssh -L 11000:localhost:11000 <YOUR_USERNAME>@<SPARK_HOSTNAME>.local
```

터널을 설정한 후 브라우저에서 포워딩된 웹 앱에 액세스하십시오: [http://localhost:11000](http://localhost:11000)

## 6단계. 다음 단계

SSH 액세스가 구성되면 다음을 수행할 수 있습니다:
- 영구 터미널 세션 열기: `ssh <YOUR_USERNAME>@<SPARK_HOSTNAME>.local`.
- 웹 애플리케이션 포트 포워딩: `ssh -L <local_port>:localhost:<remote_port> <YOUR_USERNAME>@<SPARK_HOSTNAME>.local`.

## 문제 해결

## NVIDIA Sync를 통한 연결 시 가능한 문제

| 증상 | 원인 | 해결 방법 |
|---------|--------|-----|
| 장치 이름이 해석되지 않음 | 네트워크에서 mDNS 차단됨 | hostname.local 대신 IP 주소 사용 |
| 연결 거부/시간 초과 | DGX Spark가 부팅되지 않았거나 SSH가 준비되지 않음 | 장치 부팅 완료 대기; 업데이트 완료 후 SSH 사용 가능 |
| 인증 실패 | SSH 키 설정 미완료 | NVIDIA Sync에서 장치 설정 재실행; 자격 증명 확인 |

## 수동 SSH를 통한 연결 시 가능한 문제

| 증상 | 원인 | 해결 방법 |
|---------|--------|-----|
| 장치 이름이 해석되지 않음 | 네트워크에서 mDNS 차단됨 | hostname.local 대신 IP 주소 사용 |
| 연결 거부/시간 초과 | DGX Spark가 부팅되지 않았거나 SSH가 준비되지 않음 | 장치 부팅 완료 대기; 업데이트 완료 후 SSH 사용 가능 |
| 포트 포워딩 실패 | 서비스가 실행 중이지 않거나 포트 충돌 | 원격 서비스가 활성화되어 있는지 확인; 다른 로컬 포트 시도 |
