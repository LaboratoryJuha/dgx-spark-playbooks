# Isaac Sim 및 Isaac Lab 설치 및 사용

> Spark용 Isaac Sim 및 Isaac Lab을 소스에서 빌드

## 목차

- [개요](#개요)
- [Isaac Sim 실행](#isaac-sim-실행)
- [Isaac Lab 실행](#isaac-lab-실행)
- [문제 해결](#문제-해결)

---

## 개요

## 기본 아이디어

Isaac Sim은 로봇과 환경의 사실적이고 물리적으로 정확한 시뮬레이션을 가능하게 하는 NVIDIA Omniverse 기반 로봇 시뮬레이션 플랫폼입니다. 물리 시뮬레이션, 센서 시뮬레이션 및 시각화 기능을 포함한 로봇 개발을 위한 포괄적인 툴킷을 제공합니다. Isaac Lab은 Isaac Sim 위에 구축된 강화 학습 프레임워크로 로봇 애플리케이션을 위한 RL 정책 훈련 및 배포를 위해 설계되었습니다.

Isaac Sim은 GPU 가속 물리 시뮬레이션을 사용하여 실시간보다 빠르게 실행할 수 있는 빠르고 현실적인 로봇 시뮬레이션을 가능하게 합니다. Isaac Lab은 이동, 조작 및 내비게이션과 같은 일반적인 로봇 작업을 위한 사전 구축된 RL 환경, 훈련 스크립트 및 평가 도구로 이를 확장합니다. 함께 실제 하드웨어에 배포하기 전에 시뮬레이션에서 로봇 애플리케이션을 개발, 훈련 및 테스트하기 위한 엔드 투 엔드 솔루션을 제공합니다.

## 달성할 내용

NVIDIA DGX Spark 장치에서 Isaac Sim을 소스에서 빌드하고 강화 학습 실험을 위해 Isaac Lab을 설정합니다. 여기에는 Isaac Sim 엔진 컴파일, 개발 환경 구성 및 샘플 RL 훈련 작업을 실행하여 설치를 확인하는 것이 포함됩니다.

## 시작하기 전에 알아야 할 사항

- CMake 및 빌드 시스템을 사용하여 소스에서 소프트웨어 빌드 경험
- Linux 명령줄 작업 및 환경 변수에 대한 익숙함
- 대용량 파일 관리를 위한 Git 버전 제어 및 Git LFS에 대한 이해
- Python 패키지 관리 및 가상 환경에 대한 기본 지식
- 로봇 시뮬레이션 개념에 대한 익숙함(도움이 되지만 필수는 아님)

## 전제 조건

**하드웨어 요구 사항:**
- NVIDIA Grace Blackwell GB10 Superchip 시스템
- Isaac Sim 빌드 아티팩트 및 종속성을 위한 최소 50GB 이상의 사용 가능한 저장 공간

**소프트웨어 요구 사항:**
- NVIDIA DGX OS
- GCC/G++ 11 컴파일러: `gcc --version`이 버전 11.x를 표시
- Git 및 Git LFS 설치됨: `git --version` 및 `git lfs version`이 성공
- GitHub에서 저장소를 복제하고 종속성을 다운로드하기 위한 네트워크 액세스

## 부속 파일

필요한 모든 자산은 GitHub의 Isaac Sim 및 Isaac Lab 저장소에서 찾을 수 있습니다:
- [Isaac Sim 저장소](https://github.com/isaac-sim/IsaacSim) - 메인 Isaac Sim 소스 코드
- [Isaac Lab 저장소](https://github.com/isaac-sim/IsaacLab) - Isaac Lab RL 프레임워크

## 시간 및 위험

* **예상 시간:** 30분(일반적으로 10-15분이 걸리는 빌드 시간 포함)
* **위험 수준:** 중간
  * Git LFS가 있는 대형 저장소 복제는 네트워크 문제로 인해 실패할 수 있음
  * 빌드 프로세스에는 상당한 컴파일 시간이 필요하며 종속성 문제가 발생할 수 있음
  * 빌드 아티팩트는 상당한 디스크 공간을 소비함
* **롤백:** Isaac Sim 빌드 디렉토리를 제거하여 공간을 확보할 수 있습니다. Git 저장소는 필요한 경우 삭제하고 다시 복제할 수 있습니다.
* **최종 업데이트:** 01/02/2026
  * 첫 번째 출판

## Isaac Sim 실행

## 1단계. gcc-11 및 git-lfs 설치

다음 명령을 사용하여 빌드하기 전에 GCC/G++ 11이 사용되고 있는지 확인하십시오:
```bash
sudo apt update && sudo apt install -y gcc-11 g++-11
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-11 200
sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-11 200
sudo apt install git-lfs
gcc --version
g++ --version
```

## 2단계. 작업 공간에 Isaac Sim 저장소 복제

NVIDIA GitHub 저장소에서 Isaac Sim을 복제하고 Git LFS를 설정하여 대용량 파일을 가져오십시오.

> **참고:** Isaac Sim 6.0.0 Early Developer Release의 경우 다음을 사용하십시오:
> ```bash
> git clone --depth=1 --recursive --branch=develop https://github.com/isaac-sim/IsaacSim
> ```

```bash
git clone --depth=1 --recursive https://github.com/isaac-sim/IsaacSim
cd IsaacSim
git lfs install
git lfs pull
```

## 3단계. Isaac Sim 빌드

Isaac Sim을 빌드하고 라이선스 계약에 동의하십시오.

```bash
./build.sh
```

빌드가 성공하면 다음 메시지가 표시됩니다: **BUILD (RELEASE) SUCCEEDED (Took 674.39 seconds)**


## 4단계. 시스템에 Isaac Sim 인식.

다음 명령을 실행할 때 Isaac Sim 디렉토리 안에 있는지 확인하십시오.

```bash
export ISAACSIM_PATH="${PWD}/_build/linux-aarch64/release"
export ISAACSIM_PYTHON_EXE="${ISAACSIM_PATH}/python.sh"
```

## 5단계. Isaac Sim 실행

제공된 Python 실행 파일을 사용하여 Isaac Sim을 시작하십시오.

```bash
export LD_PRELOAD="$LD_PRELOAD:/lib/aarch64-linux-gnu/libgomp.so.1"
${ISAACSIM_PATH}/isaac-sim.sh
```

## Isaac Lab 실행

## 1단계. Isaac Sim 설치
아직 설치하지 않았다면 먼저 [Isaac Sim](build.nvidia.com/spark/isaac/isaac-sim)을 설치하십시오.

## 2단계. 작업 공간에 Isaac Lab 저장소 복제

NVIDIA GitHub 저장소에서 Isaac Lab을 복제하십시오.

```bash
git clone --recursive https://github.com/isaac-sim/IsaacLab
cd IsaacLab
```

## 3단계. Isaac Sim 설치에 대한 심볼릭 링크 생성

다음 명령을 실행하기 전에 [Isaac Sim](build.nvidia.com/spark/isaac/isaac-sim)에서 Isaac Sim을 이미 설치했는지 확인하십시오.

```bash
echo "ISAACSIM_PATH=$ISAACSIM_PATH"
```
Isaac Sim 설치 디렉토리에 대한 심볼릭 링크를 만드십시오.
```bash
ln -sfn "${ISAACSIM_PATH}" "${PWD}/_isaac_sim"
ls -l "${PWD}/_isaac_sim/python.sh"
```

## 4단계. Isaac Lab 설치

```bash
./isaaclab.sh --install
```

## 5단계. Isaac Lab 실행 및 Humanoid 강화 학습 훈련 확인

제공된 Python 실행 파일을 사용하여 Isaac Lab을 시작하십시오. 다음 모드 중 하나에서 훈련을 실행할 수 있습니다:

**옵션 1: 헤드리스 모드(더 빠른 훈련 권장)**

시각화 없이 실행되고 로그를 터미널에 직접 출력합니다.

```bash
export LD_PRELOAD="$LD_PRELOAD:/lib/aarch64-linux-gnu/libgomp.so.1"
./isaaclab.sh -p scripts/reinforcement_learning/rsl_rl/train.py --task=Isaac-Velocity-Rough-H1-v0 --headless
```

**옵션 2: 시각화 활성화**

Isaac Sim에서 실시간 시각화와 함께 실행되어 훈련 프로세스를 대화식으로 모니터링할 수 있습니다.

```bash
export LD_PRELOAD="$LD_PRELOAD:/lib/aarch64-linux-gnu/libgomp.so.1"
./isaaclab.sh -p scripts/reinforcement_learning/rsl_rl/train.py --task=Isaac-Velocity-Rough-H1-v0
```

## 문제 해결

## Isaac Sim의 일반적인 문제

| 증상                     | 원인                    | 해결 방법                               |
|-----------------------------|--------------------------|-----------------------------------|
| Isaac Sim 컴파일 오류 | gcc/g++ 11이 기본값이 아님 | gcc/g++ 11이 기본값인지 확인 |
| Isaac Sim이 실행되지 않음      | libgomp.so.1 오류       | export LD_PRELOAD 추가             |
| 빌드 오류              | 오래된 설치         | .cache 폴더 제거              |

## Isaac Lab의 일반적인 문제
| 증상                          | 원인 | 해결 방법 |
|----------------------------------|--------|-----|
| Isaac Lab이 실행되지 않음           | libgomp.so.1 오류       | export LD_PRELOAD 추가     |
