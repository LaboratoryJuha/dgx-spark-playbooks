# 최적화된 JAX

> Spark에서 실행하도록 JAX 최적화

## 목차

- [개요](#개요)
- [지침](#지침)
- [문제 해결](#문제-해결)

---

## 개요

## 기본 아이디어

JAX를 사용하면 **NumPy 스타일 Python 코드**를 작성하고 CUDA를 작성하지 않고도 GPU에서 빠르게 실행할 수 있습니다. 다음과 같은 방법으로 수행합니다:

- **가속기의 NumPy**: NumPy처럼 `jax.numpy`를 사용하지만 배열은 GPU에 있습니다.
- **함수 변환**:
  - `jit` → 함수를 빠른 GPU 코드로 컴파일
  - `grad` → 자동 미분 제공
  - `vmap` → 배치 전체에서 함수를 벡터화
  - `pmap` → 여러 GPU에서 병렬로 실행
- **XLA 백엔드**: JAX는 코드를 XLA(Accelerated Linear Algebra 컴파일러)에 전달하여 연산을 융합하고 최적화된 GPU 커널을 생성합니다.

## 달성할 내용

Blackwell 아키텍처가 있는 NVIDIA Spark에서 JAX 개발 환경을 설정하여 익숙한 NumPy와 유사한 추상화를 사용하고 GPU 가속 및 성능 최적화 기능을 갖춘 고성능 머신러닝 프로토타입 제작을 가능하게 합니다.

## 시작하기 전에 알아야 할 사항

- Python 및 NumPy 프로그래밍에 익숙함
- 머신러닝 워크플로 및 기술에 대한 일반적인 이해
- 터미널 작업 경험
- 컨테이너 사용 및 구축 경험
- 다양한 버전의 CUDA에 대한 익숙함
- 선형 대수학에 대한 기본 이해(고등학교 수준의 수학으로 충분)

## 전제 조건

- Blackwell 아키텍처가 있는 NVIDIA Spark 장치
- ARM64(AArch64) 프로세서 아키텍처
- Docker 또는 컨테이너 런타임 설치됨
- NVIDIA Container Toolkit 구성됨
- GPU 액세스 확인: `nvidia-smi`
- marimo 노트북 액세스를 위한 포트 8080 사용 가능

## 부속 파일

필요한 모든 자산은 [여기 GitHub에서](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main) 찾을 수 있습니다

- [**JAX 소개 노트북**](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main/nvidia/jax/assets/jax-intro.py) — NumPy와 JAX 프로그래밍 모델 차이점 및 성능 평가를 다룹니다
- [**NumPy SOM 구현**](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main/nvidia/jax/assets/numpy-som.py) — NumPy에서 자체 조직화 맵 훈련 알고리즘의 참조 구현
- [**JAX SOM 구현**](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main/nvidia/jax/assets/som-jax.py) — JAX에서 SOM 알고리즘의 여러 반복적으로 개선된 구현
- [**환경 구성**](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main/nvidia/jax/assets/Dockerfile) — 패키지 종속성 및 컨테이너 설정 사양


## 시간 및 위험

* **소요 시간:** 설정, 튜토리얼 완료 및 검증을 포함하여 2-3시간
* **위험:**
  * Python 환경의 패키지 종속성 충돌
  * 성능 검증에는 아키텍처별 최적화가 필요할 수 있음
* **롤백:** 컨테이너 환경은 격리를 제공합니다. 컨테이너를 제거하고 재시작하여 상태를 재설정합니다.
* **최종 업데이트:** 11/07/2025
  * 사소한 수정

## 지침

## 1단계. 시스템 전제 조건 확인

NVIDIA Spark 시스템이 요구 사항을 충족하고 GPU 액세스가 구성되어 있는지 확인하십시오.

```bash
## GPU 액세스 확인
nvidia-smi

## ARM64 아키텍처 확인
uname -m

## Docker GPU 지원 확인
docker run --gpus all --rm nvcr.io/nvidia/cuda:13.0.1-runtime-ubuntu24.04 nvidia-smi
```

권한 거부 오류(Docker 데몬 소켓에 연결하는 동안 권한 거부와 같은 것)가 표시되면 sudo로 명령을 실행할 필요가 없도록 docker 그룹에 사용자를 추가하십시오.

```bash
sudo usermod -aG docker $USER
newgrp docker
```

## 2단계. 플레이북 저장소 복제

```bash
git clone https://github.com/NVIDIA/dgx-spark-playbooks
```

## 3단계. Docker 이미지 빌드


> [!WARNING]
> 이 명령은 기본 이미지를 다운로드하고 이 환경을 지원하기 위해 로컬로 컨테이너를 빌드합니다.

```bash
cd dgx-spark-playbooks/nvidia/jax/assets
docker build -t jax-on-spark .
```

## 4단계. Docker 컨테이너 시작

GPU 지원 및 marimo 액세스를 위한 포트 포워딩과 함께 Docker 컨테이너에서 JAX 개발 환경을 실행하십시오.

```bash
docker run --gpus all --rm -it \
    --shm-size=1g --ulimit memlock=-1 --ulimit stack=67108864 \
    -p 8080:8080 \
    jax-on-spark
```

## 5단계. marimo 인터페이스 액세스

JAX 튜토리얼을 시작하려면 marimo 노트북 서버에 연결하십시오.

```bash
## 웹 브라우저를 통해 액세스
## 이동: http://localhost:8080
```

인터페이스는 목차 표시 및 marimo에 대한 간단한 소개를 로드합니다.

## 6단계. JAX 소개 튜토리얼 완료

JAX 프로그래밍 모델이 NumPy와 다른 점을 이해하기 위해 소개 자료를 진행하십시오.

JAX 소개 노트북으로 이동하여 완료하십시오. 다음 내용을 다룹니다:
- JAX 프로그래밍 모델 기초
- NumPy와의 주요 차이점
- 성능 평가 기술

## 7단계. NumPy 기준선 구현

성능 기준선을 설정하기 위해 NumPy 기반 자체 조직화 맵(SOM) 구현을 완료하십시오.

NumPy SOM 노트북을 진행하여:
- SOM 훈련 알고리즘 이해
- 익숙한 NumPy 연산을 사용하여 알고리즘 구현
- 비교를 위한 성능 메트릭 기록

## 8단계. JAX 구현으로 최적화

성능 향상을 보려면 반복적으로 개선된 JAX 구현을 진행하십시오.

JAX SOM 노트북 섹션을 완료하십시오:
- NumPy 구현의 기본 JAX 포트
- 성능 최적화된 JAX 버전
- GPU 가속 병렬 JAX 구현
- 모든 버전의 성능 비교

## 9단계. 성능 향상 검증

노트북은 각 SOM 훈련 구현의 성능을 확인하는 방법을 보여줍니다. JAX 구현이 NumPy 기준선보다 성능 향상을 보여주는 것을 볼 수 있습니다(그리고 일부는 훨씬 더 빠를 것입니다).

무작위 색상 데이터에서 SOM 훈련 출력을 시각적으로 검사하여 알고리즘 정확성을 확인하십시오.

## 10단계. 다음 단계

자체 NumPy 기반 머신러닝 코드에 JAX 최적화 기술을 적용하십시오.

```bash
## 예제: 기존 NumPy 코드 프로파일링
python -m cProfile your_numpy_script.py

## 그런 다음 JAX에 적응하고 성능 비교
```

좋아하는 NumPy 알고리즘을 JAX에 적응시키고 Blackwell GPU 아키텍처에서 성능 향상을 측정해 보십시오.

## 문제 해결

| 증상 | 원인 | 해결 방법 |
|---------|--------|-----|
| `nvidia-smi`를 찾을 수 없음 | NVIDIA 드라이버 누락 | ARM64용 NVIDIA 드라이버 설치 |
| 컨테이너가 GPU에 액세스하지 못함 | NVIDIA Container Toolkit 누락 | `nvidia-container-toolkit` 설치 |
| JAX가 CPU만 사용 | CUDA/JAX 버전 불일치 | CUDA 지원과 함께 JAX를 재설치 |
| 포트 8080을 사용할 수 없음 | 포트가 이미 사용 중 | `-p 8081:8080`을 사용하거나 8080의 프로세스 종료 |
| Docker 빌드의 패키지 충돌 | 오래된 환경 파일 | Blackwell에 대한 환경 파일 업데이트 |

> [!NOTE]
> DGX Spark는 GPU와 CPU 간의 동적 메모리 공유를 가능하게 하는 통합 메모리 아키텍처(UMA)를 사용합니다.
> UMA를 활용하도록 업데이트되는 많은 애플리케이션에서 여전히 DGX Spark의 메모리 용량 내에서도
> 메모리 문제가 발생할 수 있습니다. 그런 경우 다음과 같이 버퍼 캐시를 수동으로 플러시하십시오:
```bash
sudo sh -c 'sync; echo 3 > /proc/sys/vm/drop_caches'
```
