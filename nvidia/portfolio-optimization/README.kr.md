# 포트폴리오 최적화

> cuOpt 및 cuML을 사용한 GPU 가속 포트폴리오 최적화

## 목차

- [개요](#개요)
- [지침](#지침)
- [문제 해결](#문제-해결)

---

## 개요

## 기본 아이디어

이 플레이북은 NVIDIA cuOpt 및 NVIDIA cuML을 사용하여 Mean-CVaR(조건부 위험 가치) 모델을 사용하여 대규모 포트폴리오 최적화 문제를 거의 실시간으로 해결하는 엔드투엔드 GPU 가속 워크플로우를 보여줍니다.

포트폴리오 최적화(PO)는 위험과 수익의 균형을 맞추기 위해 고차원, 비선형 수치 최적화 문제를 해결하는 것을 포함합니다. 현대 포트폴리오는 종종 수천 개의 자산을 포함하므로 기존 CPU 기반 솔버는 고급 워크플로우에 너무 느립니다. 계산 부담을 GPU로 이동함으로써 이 솔루션은 계산 시간을 극적으로 줄입니다.

## 달성할 내용

성능 평가, 전략 백테스팅, 벤치마킹 및 시각화를 위한 도구를 제공하는 파이프라인을 구현합니다. 워크플로우에는 다음이 포함됩니다:
- **GPU 가속 최적화:** NVIDIA cuOpt LP/MILP 솔버 활용
- **데이터 기반 위험 모델링:** 자산 수익 분포에 대한 가정 없이 테일 리스크를 모델링하는 시나리오 기반 위험 측정으로 CVaR 구현
- **시나리오 생성:** NVIDIA cuML을 통해 GPU 가속 커널 밀도 추정(KDE)을 사용하여 수익 분포 모델링
- **실제 제약 조건 관리:** 집중 제한, 레버리지 제약, 회전율 제한 및 카디널리티 제약을 포함한 제약 조건 구현
- **포괄적인 백테스팅:** 리밸런싱 전략 테스트를 위한 특정 도구로 포트폴리오 성능 평가


## 시작하기 전에 알아야 할 사항

- **필수 기술(익힐 수 있음):**
  - 터미널 및 Linux 명령줄에 대한 기본 지식
  - Docker 컨테이너에 대한 기본 이해
  - Jupyter Notebook 및 Jupyter Lab 사용에 대한 기본 지식
  - Python에 대한 기본 지식
  - 데이터 과학 및 머신 러닝 개념에 대한 기본 지식
  - 주식 시장과 주식이 무엇인지에 대한 기본 지식

- **선택적 기술(즐길 수 있음):**
  - 특히 정량적 금융 및 포트폴리오 관리 분야의 금융 서비스 배경
  - 머신 러닝 개념을 사용하여 Python으로 알고리즘 및 전략을 프로그래밍하는 중급 지식

- **알아야 할 용어:**
  - **CVaR vs. Mean-Variance:** 기존 평균-분산 모델과 달리 이 워크플로우는 위험의 뉘앙스, 특히 테일 리스크 또는 시나리오별 스트레스를 포착하기 위해 조건부 위험 가치(CVaR)를 사용합니다.
  - **선형 프로그래밍:** CVaR은 위험-수익 절충을 시나리오 수에 따라 문제 크기가 확장되는 시나리오 기반 선형 프로그램으로 재구성하므로 GPU 가속이 중요합니다.
  - **벤치마킹:** 파이프라인에는 성능 향상을 검증하기 위해 표준 CPU 기반 라이브러리에 대한 벤치마킹 프로세스를 간소화하는 기본 제공 도구가 포함되어 있습니다.

## 전제 조건

**하드웨어 요구 사항:**
- NVIDIA Grace Blackwell GB10 Superchip System (DGX Spark)
- docker 컨테이너 및 GPU 가속 데이터 처리를 위한 최소 40GB 통합 메모리 여유
- docker 컨테이너 및 데이터 파일을 위한 최소 30GB 사용 가능한 저장 공간
- 고속 인터넷 연결 권장

**소프트웨어 요구 사항:**
- 작동하는 NVIDIA 및 CUDA 드라이버가 있는 NVIDIA DGX OS
- Docker
- Git

## 부속 파일

필요한 모든 자산은 [Portfolio Optimization 리포지토리에서](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main/nvidia/portfolio-optimization/assets/) 찾을 수 있습니다. 실행 중인 플레이북에서는 모두 `playbook` 폴더 아래에 있습니다.

- `cvar_basic.ipynb` - 메인 플레이북 노트북
- `/setup/README.md` - 플레이북 환경에 대한 빠른 시작 가이드
- `/setup/start_playbook.sh` - Docker 컨테이너에서 플레이북 설치를 시작하는 스크립트
- `/setup/setup_playbook.sh` - 사용자가 jupyterlab 환경에 들어가기 전에 Docker 컨테이너를 구성
- `/setup/pyproject.toml` - setup_playbook의 명령이 플레이북 환경에 설치할 라이브러리 목록으로 사용됨
- `cuDF, cuML 및 cuGraph 폴더` - GPU 가속 데이터 과학 여정을 계속하기 위한 추가 예제 노트북. Docker 컨테이너를 시작하면 이들이 포함됩니다.

## 시간 및 위험

* **예상 시간** 첫 실행 시 약 20분
  - 총 노트북 처리 시간: 전체 파이프라인에 약 7분

- **위험:**
  - Docker 컨테이너에서 실행되므로 최소화됨

* **롤백:** Docker 컨테이너를 중지하고 클론된 리포지토리를 제거하여 설치를 완전히 제거

* **마지막 업데이트:** 01/02/2026
  * 최초 게시

## 지침

## 1단계. 환경 확인

먼저 작동하는 GPU, git 및 Docker가 있는지 확인하겠습니다. 터미널을 열고 아래 명령을 복사하여 붙여넣으세요:

```bash
nvidia-smi
git --version
docker --version
```

- `nvidia-smi`는 GPU에 대한 정보를 출력합니다. 그렇지 않으면 GPU가 올바르게 구성되지 않은 것입니다.
- `git --version`은 `git version 2.43.0`과 같이 출력됩니다. git이 설치되지 않았다는 오류가 발생하면 다시 설치하세요.
- `docker --version`은 `Docker version 28.3.3, build 980b856`과 같이 출력됩니다. Docker가 설치되지 않았다는 오류가 발생하면 다시 설치하세요.

## 2단계. 설치
터미널을 열고 아래 명령을 복사하여 붙여넣으세요:

```bash
git clone https://github.com/NVIDIA/dgx-spark-playbooks/nvidia/portfolio-optimization
cd dgx-spark-playbooks/nvidia/portfolio-optimization/assets
bash ./setup/start_playbook.sh
```

start_playbook.sh는 다음을 수행합니다:

1. RAPIDS 25.10 Notebooks Docker 컨테이너 가져오기
2. `setup_playbook.sh`를 사용하여 컨테이너에서 플레이북에 필요한 모든 환경 빌드
3. Jupyterlab 시작

플레이북을 사용하는 동안 터미널 창을 열어 두세요.

세 가지 방법으로 Jupyterlab 서버에 액세스할 수 있습니다
1. DGX Spark에서 로컬로 실행하는 경우 `http://127.0.0.1:8888`에서
2. 네트워크를 통해 DGX Spark를 헤드리스로 사용하는 경우 `http://<SPARK_IP>:8888`에서
3. 터미널에서 `ssh -L 8888:localhost:8888 username@spark-IP`를 사용하여 SSH 터널을 생성한 다음 호스트 머신의 브라우저에서 `http://127.0.0.1:8888`로 이동

Jupyterlab에 들어가면 `cvar_basic.ipynb`와 `cudf`, `cuml` 및 `cugraph` 폴더가 포함된 디렉토리가 표시됩니다.

- `cvar_basic.ipynb`는 플레이북 노트북입니다. 파일을 더블 클릭하여 열 수 있습니다.
- `cudf`, `cuml`, `cugraph` 폴더에는 계속 탐색하는 데 도움이 되는 표준 RAPIDS 라이브러리 예제 노트북이 포함되어 있습니다.
- `playbook`에는 플레이북 파일이 포함되어 있습니다. 이 폴더의 내용은 루트리스 Docker 컨테이너 내에서 읽기 전용입니다.

자체 시스템에 플레이북 노트북을 설치하려면 노트북과 함께 제공되는 폴더 내의 readme를 확인하세요

## 3단계. 노트북 실행

jupyterlab에 들어가면 `cvar_basic.ipynb`를 실행하기만 하면 됩니다.

노트북에서 셀 실행을 시작하기 전에 **노트북의 지침에 따라 커널을 "Portfolio Optimization"으로 변경하세요.** 그렇지 않으면 두 번째 코드 셀에서 오류가 발생합니다. 이미 시작한 경우 올바른 커널로 설정한 다음 커널을 다시 시작하고 다시 시도해야 합니다.

`Shift + Enter`를 사용하여 각 셀을 수동으로 자신의 속도로 실행하거나 `Run > Run All`을 사용하여 모든 셀을 실행할 수 있습니다.

`cvar_basic` 노트북 탐색을 마친 후 폴더로 이동하여 다른 노트북을 선택하고 동일한 작업을 수행하여 다른 RAPIDS 노트북을 탐색할 수 있습니다.

## 4단계. 작업 다운로드

docker 컨테이너는 권한이 없고 호스트 시스템에 다시 쓸 수 없으므로 Jupyterlab을 사용하여 docker 컨테이너가 종료된 후 보관하려는 파일을 다운로드할 수 있습니다.

브라우저에서 원하는 파일을 마우스 오른쪽 버튼으로 클릭하고 드롭다운에서 `Download`를 클릭하기만 하면 됩니다.

## 5단계. 정리

모든 작업을 다운로드한 후 플레이북 실행을 시작한 터미널 창으로 돌아갑니다.

터미널 창에서:
1. `Ctrl + C`를 입력
2. 프롬프트에서 `y`를 입력하고 `Enter`를 누르거나 `Ctrl + C`를 다시 빠르게 누름
3. Docker 컨테이너가 종료됩니다

> [!WARNING]
> 이렇게 하면 Docker 컨테이너에서 이미 다운로드하지 않은 모든 데이터가 삭제됩니다. 브라우저 창이 여전히 열려 있으면 캐시된 파일이 계속 표시될 수 있습니다.

## 6단계. 다음 단계

이 기본 워크플로우에 익숙해지면 **[NVIDIA AI Blueprints](https://github.com/NVIDIA-AI-Blueprints/quantitative-portfolio-optimization/)**에서 이러한 고급 포트폴리오 최적화 주제를 순서에 관계없이 탐색하세요:

* **[`efficient_frontier.ipynb`](https://github.com/NVIDIA-AI-Blueprints/quantitative-portfolio-optimization/tree/main/notebooks/efficient_frontier.ipynb)** - 효율적 프론티어 분석

  이 노트북은 다음을 보여줍니다:
  - 여러 최적화 문제를 해결하여 효율적 프론티어 생성
  - 다양한 포트폴리오 구성에 걸친 위험-수익 절충 시각화
  - 효율적 프론티어를 따라 포트폴리오 비교
  - GPU 가속을 활용하여 여러 최적 포트폴리오를 빠르게 계산

* **[`rebalancing_strategies.ipynb`](https://github.com/NVIDIA-AI-Blueprints/quantitative-portfolio-optimization/tree/main/notebooks/rebalancing_strategies.ipynb)** - 동적 포트폴리오 리밸런싱

  이 노트북은 동적 포트폴리오 관리 기법을 소개합니다:
  - 시계열 백테스팅 프레임워크
  - 다양한 리밸런싱 전략 테스트(주기적, 임계값 기반 등)
  - 포트폴리오 성능에 대한 거래 비용의 영향 평가
  - 다양한 시장 상황에서 전략 성능 분석
  - 여러 리밸런싱 접근 방식 비교

* 유사한 위험-수익 프레임워크를 사용하여 포트폴리오 최적화 문제를 공식화하는 방법을 더 배우려면 **[DLI 과정: 포트폴리오 최적화 가속화](https://learn.nvidia.com/courses/course-detail?course_id=course-v1:DLI+S-DS-09+V1)**를 확인하세요

## 7단계. 추가 지원

질문이나 문제가 있는 경우 다음을 방문하세요:
- [GitHub Issues](https://github.com/NVIDIA-AI-Blueprints/quantitative-portfolio-optimization/issues)

## 문제 해결

| 증상 | 원인 | 해결 방법 |
|---------|-------|-----|
| Docker를 찾을 수 없음 | DGX Spark에 사전 설치되어 있으므로 Docker가 제거되었을 수 있음 | 여기에서 편의 스크립트를 사용하여 Docker를 설치하세요: `curl -fsSL https://get.docker.com -o get-docker.sh && sudo sh get-docker.sh`. 비밀번호를 입력하라는 메시지가 표시됩니다. |
| Docker 명령이 "permissions" 오류로 예기치 않게 종료됨 | 사용자가 `docker` 그룹에 속하지 않음 | 터미널을 열고 다음 명령을 실행하세요: `sudo groupadd docker $$ sudo usermod -aG docker $USER`. 비밀번호를 입력하라는 메시지가 표시됩니다. 그런 다음 터미널을 닫고 새로 열고 다시 시도하세요 |
| Docker 컨테이너 다운로드, 환경 빌드 또는 데이터 다운로드 실패 | 연결 문제가 있거나 리소스를 일시적으로 사용할 수 없음 | 나중에 다시 시도해야 할 수 있습니다. 이것이 계속되면 저희에게 연락하세요! |


> [!NOTE]
> DGX Spark는 GPU와 CPU 간에 동적 메모리 공유를 가능하게 하는 통합 메모리 아키텍처(UMA)를 사용합니다.
> 많은 애플리케이션이 여전히 UMA를 활용하도록 업데이트 중이므로 DGX Spark의 메모리 용량 내에 있더라도
> 메모리 문제가 발생할 수 있습니다. 이런 경우 다음과 같이 버퍼 캐시를 수동으로 플러시하세요:
```bash
sudo sh -c 'sync; echo 3 > /proc/sys/vm/drop_caches'
```

최신 알려진 문제에 대해서는 [DGX Spark 사용자 가이드](https://docs.nvidia.com/dgx/dgx-spark/known-issues.html)를 참조하세요.
