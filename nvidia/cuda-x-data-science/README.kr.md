# CUDA-X Data Science

> 코드 변경 없이 UMAP, HDBSCAN, pandas 등을 가속화하기 위해 NVIDIA cuML 및 NVIDIA cuDF 설치 및 사용


## 목차

- [개요](#개요)
- [지침](#지침)

---

## 개요

## 기본 아이디어
이 플레이북에는 CUDA-X Data Science 라이브러리를 사용하여 핵심 머신러닝 알고리즘과 핵심 pandas 작업의 가속화를 보여주는 두 가지 예제 노트북이 포함되어 있습니다:

- **NVIDIA cuDF:** 코드 변경 없이 8GB의 문자열 데이터에 대한 데이터 준비 및 핵심 데이터 처리 작업을 가속화합니다.
- **NVIDIA cuML:** 코드 변경 없이 sci-kit learn(LinearSVC), UMAP 및 HDBSCAN에서 인기 있는 계산 집약적인 머신러닝 알고리즘을 가속화합니다.

CUDA-X Data Science(공식적으로 RAPIDS)는 데이터 과학 및 데이터 처리 에코시스템을 가속화하는 오픈 소스 라이브러리 컬렉션입니다. 이러한 라이브러리는 코드 변경 없이 scikit-learn 및 pandas와 같은 인기 있는 Python 도구를 가속화합니다. DGX Spark에서 이러한 라이브러리는 기존 코드로 책상에서 성능을 극대화합니다.

## 달성할 내용
GPU에서 인기 있는 머신러닝 알고리즘 및 데이터 분석 작업을 가속화합니다. 인기 있는 Python 도구를 가속화하는 방법과 DGX Spark에서 데이터 과학 워크플로를 실행하는 가치를 이해하게 됩니다.

## 전제 조건
- pandas, scikit-learn, 지원 벡터 머신, 클러스터링 및 차원 축소 알고리즘과 같은 머신러닝 알고리즘에 대한 익숙함.
- conda 설치
- Kaggle API 키 생성

## 시간 및 위험
* **소요 시간:** 설정 시간 20-30분 및 각 노트북 실행에 2-3분.
* **위험:**
  * 네트워크 문제로 인한 데이터 다운로드 느림 또는 실패
  * 재시도가 필요한 Kaggle API 생성 실패
* **롤백:** 정상적인 사용 중에는 영구적인 시스템 변경이 이루어지지 않습니다.
* **최종 업데이트:** 11/07/2025
  * 사소한 수정

## 지침

## 1단계. 시스템 요구 사항 확인
- `nvcc --version` 또는 `nvidia-smi`를 사용하여 시스템에 CUDA 13이 설치되어 있는지 확인
- [이 지침](https://docs.anaconda.com/miniconda/install/)을 사용하여 conda 설치
- [이 지침](https://www.kaggle.com/discussions/general/74235)을 사용하여 Kaggle API 키를 생성하고 노트북과 동일한 폴더에 **kaggle.json** 파일을 배치

## 2단계. Data Science 라이브러리 설치
다음 명령을 사용하여 CUDA-X 라이브러리를 설치하십시오(새 conda 환경이 생성됨)
  ```bash
    conda create -n rapids-test -c rapidsai-nightly -c conda-forge -c nvidia  \
    rapids=25.10 python=3.12 'cuda-version=13.0' \
    jupyter hdbscan umap-learn
  ```
## 3단계. conda 환경 활성화
  ```bash
    conda activate rapids-test
  ```
## 4단계. 플레이북 저장소 복제
- github 저장소를 복제하고 **cuda-x-data-science** 폴더의 assets 폴더로 이동
  ```bash
    git clone https://github.com/NVIDIA/dgx-spark-playbooks
  ```
- 1단계에서 생성한 **kaggle.json**을 assets 폴더에 배치

## 5단계. 노트북 실행
GitHub 저장소에는 두 개의 노트북이 있습니다.
하나는 GPU에서 pandas 코드로 대형 문자열 데이터 처리 워크플로의 예를 실행합니다.
- **cudf_pandas_demo.ipynb** 노트북을 실행하고 브라우저에서 `localhost:8888`을 사용하여 노트북에 액세스
  ```bash
    jupyter notebook cudf_pandas_demo.ipynb
  ```
다른 하나는 UMAP 및 HDBSCAN을 포함한 머신러닝 알고리즘의 예를 다룹니다.
- **cuml_sklearn_demo.ipynb** 노트북을 실행하고 브라우저에서 `localhost:8888`을 사용하여 노트북에 액세스
  ```bash
    jupyter notebook cuml_sklearn_demo.ipynb
  ```
DGX-Spark에 원격으로 액세스하는 경우 로컬 브라우저에서 노트북에 액세스하기 위해 필요한 포트를 포워딩해야 합니다. 포트 포워딩을 위해 아래 지침을 사용하십시오
```bash
  ssh -N -L YYYY:localhost:XXXX username@remote_host
```
- `YYYY`: 사용하려는 로컬 포트 (예: 8888)
- `XXXX`: 원격 시스템에서 Jupyter Notebook을 시작할 때 지정한 포트 (예: 8888)
- `-N`: SSH가 원격 명령을 실행하지 않도록 방지
- `-L`: 로컬 포트 포워딩 지정
