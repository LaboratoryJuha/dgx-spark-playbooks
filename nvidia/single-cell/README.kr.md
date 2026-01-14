# 단일 세포 RNA 시퀀싱

> RAPIDS를 사용한 scRNA-seq를 위한 엔드투엔드 GPU 기반 워크플로우

## 목차

- [개요](#개요)
- [지침](#지침)
- [문제 해결](#문제-해결)

---

## 개요

## 기본 아이디어

단일 세포 RNA 시퀀싱(scRNA-seq)을 통해 연구자들은 각 세포의 유전자 활동을 개별적으로 연구할 수 있으며, 벌크 방법이 숨기는 변이, 세포 유형 및 세포 상태를 노출합니다. 그러나 이러한 대규모 고차원 데이터셋은 처리하는 데 많은 계산이 필요합니다.

이 플레이북은 [scverse® 에코시스템](https://github.com/scverse)의 RAPIDS 기반 라이브러리인 [RAPIDS-singlecell](https://rapids-singlecell.readthedocs.io/en/latest/)을 사용하는 scRNA-seq를 위한 엔드투엔드 GPU 기반 워크플로우를 보여줍니다. 친숙한 [Scanpy API](https://scanpy.readthedocs.io/en/stable/)를 따르며 연구자들이 GPU에서 직접 희소 카운트 행렬로 작업하여 CPU 도구보다 더 빠르게 데이터 전처리, 품질 관리(QC) 및 정리, 시각화 및 조사 단계를 실행할 수 있게 합니다.

## 달성할 내용

1. GPU 가속 데이터 로딩 및 전처리
2. 데이터를 이해하기 위해 QC 세포를 시각적으로 확인
3. 비정상적인 세포 필터링
4. 원치 않는 변이 소스 제거
5. PCA 및 UMAP 데이터 클러스터링 및 시각화
6. Harmony, k-최근접 이웃, UMAP 및 tSNE를 사용한 배치 보정 및 분석
7. 차등 발현 분석 및 궤적 분석으로 데이터의 생물학적 정보 탐색

README는 이러한 단계를 자세히 설명합니다.

## 시작하기 전에 알아야 할 사항

- rapids-singlecell 라이브러리는 scverse의 Scanpy API를 모방하여 표준 CPU 워크플로우에 익숙한 사용자가 cuPy 및 NVIDIA RAPIDS cuML 및 cuGraph를 통해 GPU 가속에 쉽게 적응할 수 있게 합니다.
- 알고리즘 정밀도: 근사 최근접 이웃 검색을 사용하는 Scanpy의 CPU 구현과 달리 이 GPU 구현은 정확한 그래프를 계산합니다. 따라서 결과의 작은 차이는 예상되고 유효합니다.
- 파라미터 민감도: t-SNE를 수행할 때 왜곡을 피하기 위해 최근접 이웃의 수는 최소 3배여야 합니다

## 전제 조건
**하드웨어 요구 사항:**
- NVIDIA Grace Blackwell GB10 Superchip System (DGX Spark)
- docker 컨테이너 및 GPU 가속 데이터 처리를 위한 최소 40GB 통합 메모리 여유
- docker 컨테이너 및 데이터 파일을 위한 최소 30GB 사용 가능한 저장 공간
- 고속 네트워크 연결
- 고속 인터넷 연결 권장

**소프트웨어 요구 사항:**
- NVIDIA DGX OS
- Docker

## 부속 파일

필요한 모든 자산은 [단일 세포 RNA 시퀀싱 리포지토리에서](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main/nvidia/single-cell/) 찾을 수 있습니다. 실행 중인 플레이북에서는 모두 `playbook` 폴더 아래에 있습니다.

- `scRNA_analysis_preprocessing.ipynb` - 메인 플레이북 노트북
- `README.md` - 플레이북 환경에 대한 빠른 시작 가이드. Jupyter Lab의 메인 디렉토리에서도 찾을 수 있습니다. 여기서 시작하세요!
- `/setup/start_playbook.sh` - Docker 컨테이너에서 플레이북 설치를 시작하는 스크립트
- `/setup/setup_playbook.sh` - 사용자가 JupyterLab 환경에 들어가기 전에 Docker 컨테이너를 구성
- `/setup/requirements.txt` - setup_playbook의 명령이 플레이북 환경에 설치할 라이브러리 목록으로 사용됨

## 시간 및 위험
* **예상 시간:** 첫 실행 시 약 15분

  - 총 노트북 처리 시간: 전체 파이프라인에 약 2-3분(데모에서 약 130초 기록).
  - 데이터 로딩: 약 1.7초.
  - 전처리: 약 21초.
  - 후처리(클러스터링/차등 발현): 약 104초.
  - 데이터: docker 컨테이너, 라이브러리 및 데모 데이터셋(dli_census.h5ad)을 다운로드하기 위한 인터넷 액세스.

* **위험**

  - GPU 메모리 제약: 워크플로우는 GPU 메모리를 많이 사용합니다. 대규모 데이터셋은 메모리 부족(OOM) 오류를 발생시킬 수 있습니다.
  - 커널 관리: 워크플로우 단계 간에 GPU 리소스를 확보하기 위해 커널을 종료/재시작해야 할 수 있습니다.
  - 롤백: OOM 오류가 발생하면 모든 커널을 종료하여 GPU 메모리를 확보하고 특정 노트북 또는 전체 플레이북을 다시 시작합니다.

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
- `docker --version`은 `Docker version 28.3.3, build 980b856`과 같이 출력됩니다. Docker가 설치되지 않았다는 오류가 발생하면 다시 설치하세요. 권한 거부 오류가 표시되면 `sudo usermod -aG docker $USER && newgrp docker`를 실행하여 사용자를 docker 그룹에 추가하세요.

## 2단계. 설치
터미널을 열고 아래 명령을 복사하여 붙여넣으세요:

```bash
git clone https://github.com/NVIDIA/dgx-spark-playbooks
cd dgx-spark-playbooks/nvidia/single-cell/assets
bash ./setup/start_playbook.sh
```

start_playbook.sh는 다음을 수행합니다:

1. RAPIDS 25.10 Notebooks Docker 컨테이너 가져오기
2. setup_playbook.sh를 사용하여 컨테이너에서 플레이북에 필요한 모든 환경 빌드
3. JupyterLab 시작

플레이북을 사용하는 동안 터미널 창을 열어 두세요.

두 가지 방법으로 JupyterLab 서버에 액세스할 수 있습니다
1. DGX Spark에서 로컬로 실행하는 경우 `http://127.0.0.1:8888`에서
2. 네트워크를 통해 DGX Spark를 헤드리스로 사용하는 경우 `http://<SPARK_IP>:8888`에서

JupyterLab에 들어가면 scRNA_analysis_preprocessing.ipynb와 `cuDF`, `cuML`, `cuGraph` 및 `playbook` 폴더가 포함된 디렉토리가 표시됩니다.

- `scRNA_analysis_preprocessing.ipynb`는 플레이북 노트북입니다. 파일을 더블 클릭하여 열 수 있습니다.
- `cuDF`, `cuML`, `cuGraph` 폴더에는 계속 탐색하는 데 도움이 되는 표준 RAPIDS 라이브러리 예제 노트북이 포함되어 있습니다.
- `playbook`에는 플레이북 파일이 포함되어 있습니다. 이 폴더의 내용은 루트리스 Docker 컨테이너 내에서 읽기 전용입니다.

자체 시스템에 플레이북 노트북을 설치하려면 노트북과 함께 제공되는 폴더 내의 readme를 확인하세요

## 3단계. 노트북 실행

JupyterLab에 들어가면 `scRNA_analysis_preprocessing.ipynb`를 실행하기만 하면 됩니다. 이 플레이북 노트북과 함께 시작하는 데 도움이 되는 표준 RAPIDS 라이브러리 예제 노트북도 얻을 수 있습니다.

`Shift + Enter`를 사용하여 각 셀을 수동으로 자신의 속도로 실행하거나 `Run > Run All`을 사용하여 모든 셀을 실행할 수 있습니다.

`scRNA_analysis_preprocessing` 노트북 탐색을 마친 후 폴더로 이동하여 다른 노트북을 선택하고 동일한 작업을 수행하여 다른 RAPIDS 노트북을 탐색할 수 있습니다.

## 4단계. 작업 다운로드

docker 컨테이너는 호스트 시스템에 권한 있는 쓰기를 할 수 없으므로 JupyterLab을 사용하여 docker 컨테이너가 종료된 후 보관하려는 파일을 다운로드할 수 있습니다.

브라우저에서 원하는 파일을 마우스 오른쪽 버튼으로 클릭하고 드롭다운에서 `Download`를 클릭하기만 하면 됩니다.

## 5단계. 정리

모든 작업을 다운로드한 후 플레이북 실행을 시작한 터미널 창으로 돌아갑니다.

터미널 창에서,
1. `Ctrl + C`를 입력
2. 프롬프트에서 `y`를 입력하고 `Enter`를 누르거나 `Ctrl + C`를 다시 빠르게 누름
3. Docker 컨테이너가 종료됩니다

> [!WARNING]
> 이렇게 하면 Docker 컨테이너에서 이미 다운로드하지 않은 모든 데이터가 삭제됩니다. 브라우저 창이 여전히 열려 있으면 캐시된 파일이 계속 표시될 수 있습니다.

## 문제 해결

| 증상 | 원인 | 해결 방법 |
|---------|-------|-----|
| Docker를 찾을 수 없음 | DGX Spark에 사전 설치되어 있으므로 Docker가 제거되었을 수 있음 | 여기에서 편의 스크립트를 사용하여 Docker를 설치하세요: `curl -fsSL https://get.docker.com -o get-docker.sh && sudo sh get-docker.sh`. 비밀번호를 입력하라는 메시지가 표시됩니다. |
| Docker 명령이 "permissions" 오류로 예기치 않게 종료됨 | 사용자가 `docker` 그룹에 속하지 않음 | 터미널을 열고 다음 명령을 실행하세요: `sudo groupadd docker && sudo usermod -aG docker $USER`. 비밀번호를 입력하라는 메시지가 표시됩니다. 그런 다음 터미널을 닫고 새로 열고 다시 시도하세요 |
| Docker 컨테이너 다운로드, 환경 빌드 또는 데이터 다운로드 실패 | 연결 문제가 있거나 리소스를 일시적으로 사용할 수 없음 | 나중에 다시 시도해야 할 수 있습니다. 이것이 계속되면 지원을 위해 Spark 사용자 포럼에 게시하세요 |


> [!NOTE]
> DGX Spark는 GPU와 CPU 간에 동적 메모리 공유를 가능하게 하는 통합 메모리 아키텍처(UMA)를 사용합니다.
> 많은 애플리케이션이 여전히 UMA를 활용하도록 업데이트 중이므로 DGX Spark의 메모리 용량 내에 있더라도
> 메모리 문제가 발생할 수 있습니다. 이런 경우 다음과 같이 버퍼 캐시를 수동으로 플러시하세요:
```bash
sudo sh -c 'sync; echo 3 > /proc/sys/vm/drop_caches'
```

최신 알려진 문제에 대해서는 [DGX Spark 사용자 가이드](https://docs.nvidia.com/dgx/dgx-spark/known-issues.html)를 참조하세요.
