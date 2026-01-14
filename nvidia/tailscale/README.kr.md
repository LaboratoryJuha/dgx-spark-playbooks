# Spark에 Tailscale 설정

> Tailscale을 사용하여 어디에 있든 홈 네트워크의 Spark에 연결

## 목차

- [개요](#개요)
- [지침](#지침)
  - [1단계. 시스템 요구사항 확인](#1단계-시스템-요구사항-확인)
  - [2단계. SSH 서버 설치 (필요한 경우)](#2단계-ssh-서버-설치-필요한-경우)
  - [3단계. NVIDIA DGX Spark에 Tailscale 설치](#3단계-nvidia-dgx-spark에-tailscale-설치)
  - [4단계. Tailscale 설치 확인](#4단계-tailscale-설치-확인)
  - [5단계. DGX Spark를 Tailscale 네트워크에 연결](#5단계-dgx-spark를-tailscale-네트워크에-연결)
  - [6단계. 클라이언트 장치에 Tailscale 설치](#6단계-클라이언트-장치에-tailscale-설치)
  - [7단계. 클라이언트 장치를 tailnet에 연결](#7단계-클라이언트-장치를-tailnet에-연결)
  - [8단계. 네트워크 연결 확인](#8단계-네트워크-연결-확인)
  - [9단계. SSH 인증 구성](#9단계-ssh-인증-구성)
  - [10단계. SSH 연결 테스트](#10단계-ssh-연결-테스트)
  - [11단계. 설치 검증](#11단계-설치-검증)
  - [13단계. 정리 및 롤백](#13단계-정리-및-롤백)
  - [14단계. 다음 단계](#14단계-다음-단계)
- [문제 해결](#문제-해결)

---

## 개요

## 기본 개념

Tailscale은 복잡한 방화벽 구성이나 포트 포워딩 없이 어디서나 NVIDIA DGX Spark 장치에 안전하게 액세스할 수 있는 암호화된 피어-투-피어 메시 네트워크를 만듭니다. DGX Spark와 클라이언트 장치 모두에 Tailscale을 설치하면 각 장치가 안정적인 개인 IP 주소와 호스트 이름을 받는 개인 "tailnet"을 설정하여 집, 직장 또는 커피숍에 있든 원활한 SSH 액세스를 가능하게 합니다.

## 달성할 목표

DGX Spark 장치 및 클라이언트 머신에서 Tailscale을 설정하여 안전한 원격 액세스를 생성합니다. 완료 후 `ssh user@spark-hostname`과 같은 간단한 명령을 사용하여 어디서나 DGX Spark에 SSH로 접속할 수 있으며 모든 트래픽이 자동으로 암호화되고 NAT 순회가 투명하게 처리됩니다.

## 시작하기 전에 알아야 할 사항

- 터미널/명령줄 인터페이스 작업
- 기본 SSH 개념 및 사용법
- Ubuntu에서 `apt`를 사용한 패키지 설치
- 사용자 계정 및 인증에 대한 이해
- systemd 서비스 관리에 대한 익숙함

## 필수 사항

**하드웨어 요구사항:**
-  NVIDIA Grace Blackwell GB10 Superchip 시스템

**소프트웨어 요구사항:**
- NVIDIA DGX OS
- 원격 액세스를 위한 클라이언트 장치 (Mac, Windows 또는 Linux)
- 연결 테스트 시 클라이언트 장치와 DGX Spark가 동일한 네트워크에 있지 않아야 함
- 두 장치 모두에서 인터넷 연결
- Tailscale 인증을 위한 유효한 이메일 계정 (Google, GitHub, Microsoft)
- SSH 서버 가용성 확인: `systemctl status ssh`
- 패키지 관리자 작동: `sudo apt update`
- DGX Spark 장치에 대한 sudo 권한이 있는 사용자 계정

## 시간 및 위험도

* **기간**: 초기 설정에 15-30분, 추가 장치당 5분
* **위험:** 중간
  * 잠재적인 SSH 서비스 구성 충돌
  * 초기 설정 중 네트워크 연결 문제
  * 인증 공급자 서비스 종속성
* **롤백**: Tailscale은 `sudo apt remove tailscale`로 완전히 제거할 수 있으며 모든 네트워크 라우팅이 자동으로 기본 설정으로 되돌아갑니다.
* **마지막 업데이트:** 11/07/2025
  * 소규모 편집

## 지침

### 1단계. 시스템 요구사항 확인

DGX Spark 장치가 지원되는 Ubuntu 버전을 실행하고 있으며 인터넷 연결이 있는지 확인합니다. 이 단계는 DGX Spark 장치에서 실행되어 필수 사항을 확인합니다.

```bash
## Ubuntu 버전 확인 (20.04 이상이어야 함)
lsb_release -a

## 인터넷 연결 테스트
ping -c 3 google.com

## sudo 액세스 확인
sudo whoami
```

### 2단계. SSH 서버 설치 (필요한 경우)

Tailscale은 네트워크 연결을 제공하지만 원격 액세스를 위해서는 SSH가 필요하므로 DGX Spark 장치에서 SSH 서버가 실행되고 있는지 확인합니다. 이 단계는 DGX Spark 장치에서 실행됩니다.

```bash
## SSH가 실행 중인지 확인
systemctl status ssh --no-pager
```

**SSH가 설치되어 있지 않거나 실행 중이 아닌 경우:**

```bash
## OpenSSH 서버 설치
sudo apt update
sudo apt install -y openssh-server

## SSH 서비스 활성화 및 시작
sudo systemctl enable ssh --now --no-pager

## SSH가 실행 중인지 확인
systemctl status ssh --no-pager
```

### 3단계. NVIDIA DGX Spark에 Tailscale 설치

공식 Ubuntu 리포지토리를 사용하여 DGX Spark에 Tailscale을 설치합니다. 이 단계는 Tailscale 패키지 리포지토리를 추가하고 클라이언트를 설치합니다.

```bash
## 패키지 목록 업데이트
sudo apt update

## 외부 리포지토리 추가에 필요한 도구 설치
sudo apt install -y curl gnupg

## Tailscale 서명 키 추가
curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/noble.noarmor.gpg | \
  sudo tee /usr/share/keyrings/tailscale-archive-keyring.gpg > /dev/null

## Tailscale 리포지토리 추가
curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/noble.tailscale-keyring.list | \
  sudo tee /etc/apt/sources.list.d/tailscale.list

## 새 리포지토리로 패키지 목록 업데이트
sudo apt update

## Tailscale 설치
sudo apt install -y tailscale
```

### 4단계. Tailscale 설치 확인

인증을 진행하기 전에 DGX Spark 장치에 Tailscale이 올바르게 설치되었는지 확인합니다.

```bash
## Tailscale 버전 확인
tailscale version

## Tailscale 서비스 상태 확인
sudo systemctl status tailscaled --no-pager
```

### 5단계. DGX Spark를 Tailscale 네트워크에 연결

선택한 ID 공급자를 사용하여 DGX Spark 장치를 Tailscale로 인증합니다. 이렇게 하면 개인 tailnet이 생성되고 안정적인 IP 주소가 할당됩니다.

```bash
## Tailscale 시작 및 인증 시작
sudo tailscale up

## 표시된 URL에 따라 브라우저에서 로그인을 완료합니다
## Google, GitHub, Microsoft 또는 기타 지원되는 공급자 중에서 선택
```

### 6단계. 클라이언트 장치에 Tailscale 설치

DGX Spark에 원격으로 연결하는 데 사용할 장치에 Tailscale을 설치합니다.

클라이언트 운영 체제에 적합한 방법을 선택합니다:

**macOS의 경우:**
- 옵션 1: Mac App Store에서 "Tailscale"을 검색하고 Get → Install을 클릭하여 설치
- 옵션 2: [Tailscale 웹사이트](https://tailscale.com/download)에서 .pkg 설치 프로그램 다운로드


**Windows의 경우:**
- [Tailscale 웹사이트](https://tailscale.com/download)에서 설치 프로그램 다운로드
- .msi 파일을 실행하고 설치 프롬프트를 따릅니다
- 시작 메뉴 또는 시스템 트레이에서 Tailscale 시작


**Linux의 경우:**

DGX Spark 설치에 사용된 것과 동일한 지침을 따릅니다.

```bash
## 패키지 목록 업데이트
sudo apt update

## 외부 리포지토리 추가에 필요한 도구 설치
sudo apt install -y curl gnupg

## Tailscale 서명 키 추가
curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/noble.noarmor.gpg | \
  sudo tee /usr/share/keyrings/tailscale-archive-keyring.gpg > /dev/null

## Tailscale 리포지토리 추가
curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/noble.tailscale-keyring.list | \
  sudo tee /etc/apt/sources.list.d/tailscale.list

## 새 리포지토리로 패키지 목록 업데이트
sudo apt update

## Tailscale 설치
sudo apt install -y tailscale
```

### 7단계. 클라이언트 장치를 tailnet에 연결

DGX Spark에 사용한 것과 동일한 ID 공급자 계정을 사용하여 각 클라이언트 장치에서 Tailscale에 로그인합니다.

**macOS/Windows (GUI)의 경우:**
- Tailscale 앱 시작
- "Log in" 버튼 클릭
- DGX Spark에서 사용한 것과 동일한 계정으로 로그인

**Linux (CLI)의 경우:**

```bash
## 클라이언트에서 Tailscale 시작
sudo tailscale up

## 동일한 계정을 사용하여 브라우저에서 인증 완료
```

### 8단계. 네트워크 연결 확인

SSH 연결을 시도하기 전에 장치가 Tailscale 네트워크를 통해 통신할 수 있는지 테스트합니다.

```bash
## 모든 장치에서 tailnet 상태 확인
tailscale status

## Spark 장치에 대한 ping 테스트 (상태 출력의 호스트 이름 또는 IP 사용)
tailscale ping <SPARK_HOSTNAME>

## 예상 출력은 성공적인 ping을 보여야 함
```

### 9단계. SSH 인증 구성

DGX Spark에 대한 안전한 액세스를 위해 SSH 키 인증을 설정합니다. 이 단계는 클라이언트 장치 및 DGX Spark 장치에서 실행됩니다.

**클라이언트에서 SSH 키 생성 (아직 완료하지 않은 경우):**

```bash
## 새 SSH 키 쌍 생성
ssh-keygen -t ed25519 -f ~/.ssh/tailscale_spark

## 복사할 공개 키 표시
cat ~/.ssh/tailscale_spark.pub
```

**DGX Spark에 공개 키 추가:**

```bash
## Spark 장치에서 클라이언트의 공개 키 추가
echo "<YOUR_PUBLIC_KEY>" >> ~/.ssh/authorized_keys

## 올바른 권한 설정
chmod 600 ~/.ssh/authorized_keys
chmod 700 ~/.ssh
```

### 10단계. SSH 연결 테스트

Tailscale 네트워크를 통해 SSH를 사용하여 DGX Spark에 연결하여 전체 설정이 작동하는지 확인합니다.

```bash
## Tailscale 호스트 이름을 사용하여 연결 (권장)
ssh -i ~/.ssh/tailscale_spark <USERNAME>@<SPARK_HOSTNAME>

## 또는 Tailscale IP 주소를 사용하여 연결
ssh -i ~/.ssh/tailscale_spark <USERNAME>@<TAILSCALE_IP>

## 예제:
## ssh -i ~/.ssh/tailscale_spark nvidia@my-spark-device
```

### 11단계. 설치 검증

Tailscale이 올바르게 작동하고 SSH 연결이 안정적인지 확인합니다.

```bash
## 클라이언트 장치에서 연결 상태 확인
tailscale status

## 클라이언트 장치에서 테스트 파일 생성
echo "test file for the spark" > test.txt

## SSH를 통한 파일 전송 테스트
scp -i ~/.ssh/tailscale_spark test.txt <USERNAME>@<SPARK_HOSTNAME>:~/

## 원격으로 명령을 실행할 수 있는지 확인
ssh -i ~/.ssh/tailscale_spark <USERNAME>@<SPARK_HOSTNAME> 'nvidia-smi'
```

예상 출력:
- Tailscale 상태가 두 장치를 모두 "active"로 표시
- 파일 전송 성공
- 원격 명령 실행 작동

### 13단계. 정리 및 롤백

필요한 경우 Tailscale을 완전히 제거합니다. 이렇게 하면 장치가 tailnet에서 연결 해제되고 모든 네트워크 구성이 제거됩니다.

> [!WARNING]
> 이렇게 하면 Tailscale 네트워크에서 장치가 영구적으로 제거되며 다시 가입하려면 재인증이 필요합니다.

```bash
## Tailscale 서비스 중지
sudo tailscale down

## Tailscale 패키지 제거
sudo apt remove --purge tailscale

## 리포지토리 및 키 제거 (선택 사항)
sudo rm /etc/apt/sources.list.d/tailscale.list
sudo rm /usr/share/keyrings/tailscale-archive-keyring.gpg

## 패키지 목록 업데이트
sudo apt update
```

복원하려면: 설치 단계 3-5를 다시 실행합니다.

### 14단계. 다음 단계

Tailscale 설정이 완료되었습니다. 이제 다음을 수행할 수 있습니다:

- 모든 네트워크에서 DGX Spark 장치에 액세스: `ssh <USERNAME>@<SPARK_HOSTNAME>`
- 파일을 안전하게 전송: `scp file.txt <USERNAME>@<SPARK_HOSTNAME>:~/`
- DGX 대시보드를 열고 JupyterLab을 시작한 다음 연결:
  `ssh -L 8888:localhost:1102 <USERNAME>@<SPARK_HOSTNAME>`

## 문제 해결

| 증상 | 원인 | 해결 방법 |
|---------|-------|-----|
| `tailscale up` 인증 실패 | 네트워크 문제 | 인터넷 확인, `curl -I login.tailscale.com` 시도 |
| SSH 연결 거부 | SSH가 실행되지 않음 | Spark에서 `sudo systemctl start ssh --no-pager` 실행 |
| SSH 인증 실패 | 잘못된 SSH 키 | `~/.ssh/authorized_keys`에서 공개 키 확인 |
| 호스트 이름에 ping을 보낼 수 없음 | DNS 문제 | 대신 `tailscale status`의 IP 사용 |
| 장치 누락 | 다른 계정 | 모든 장치에 동일한 ID 공급자 사용 |


최신 알려진 문제는 [DGX Spark 사용자 가이드](https://docs.nvidia.com/dgx/dgx-spark/known-issues.html)를 검토하세요.
