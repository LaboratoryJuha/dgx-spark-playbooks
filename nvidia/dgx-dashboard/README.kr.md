# DGX Dashboard

> DGX 시스템을 모니터링하고 JupyterLab 시작하기

## 목차

- [개요](#개요)
- [지침](#지침)
- [문제 해결](#문제-해결)

---

## 개요

## 기본 아이디어

DGX Dashboard는 DGX Spark 장치에서 로컬로 실행되는 웹 애플리케이션으로, 시스템 업데이트, 리소스 모니터링 및 통합 JupyterLab 환경을 위한 그래픽 인터페이스를 제공합니다. 사용자는 앱 런처에서 로컬로 대시보드에 액세스하거나 NVIDIA Sync 또는 SSH 터널링을 통해 원격으로 액세스할 수 있습니다. 대시보드는 원격으로 작업할 때 시스템 패키지 및 펌웨어를 업데이트하는 가장 쉬운 방법입니다.

## 달성할 내용

DGX Spark 장치에서 DGX Dashboard에 액세스하고 사용하는 방법을 배웁니다. 이 연습이 끝나면 사전 구성된 Python 환경으로 JupyterLab 인스턴스를 시작하고, GPU 성능을 모니터링하고, 시스템 업데이트를 관리하고, Stable Diffusion을 사용하여 샘플 AI 워크로드를 실행할 수 있습니다. 데스크톱 바로 가기, NVIDIA Sync 및 수동 SSH 터널링을 포함한 여러 액세스 방법을 이해하게 됩니다.

## 시작하기 전에 알아야 할 사항

- SSH 연결 및 포트 포워딩을 위한 기본 터미널 사용
- Python 환경 및 Jupyter 노트북에 대한 이해

## 전제 조건

**하드웨어 요구 사항:**
-  NVIDIA Grace Blackwell GB10 Superchip 시스템

**소프트웨어 요구 사항:**
- NVIDIA DGX OS
- NVIDIA Sync 설치됨 (원격 액세스 방법용) 또는 SSH 클라이언트 구성됨

## 부속 파일

- SDXL용 Python 코드 스니펫은 [여기 GitHub에서](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main/nvidia/dgx-dashboard/assets/jupyter-cell.py) 찾을 수 있습니다


## 시간 및 위험

* **소요 시간:** 샘플 AI 워크로드를 포함한 전체 연습에 15-30분
* **위험 수준:** 낮음 - 최소한의 시스템 영향이 있는 웹 인터페이스 작업
* **롤백:** 대시보드 인터페이스를 통해 JupyterLab 인스턴스를 중지합니다. 정상적인 사용 중에는 영구적인 시스템 변경이 이루어지지 않습니다.
* **최종 업데이트:** 11/21/2025
  * 사소한 수정

## 지침

## 1단계. DGX Dashboard 액세스

DGX Dashboard 웹 인터페이스에 액세스하려면 다음 방법 중 하나를 선택하십시오:

**옵션 A: 데스크톱 바로 가기 (로컬 액세스)**

DGX Spark 장치에 로컬 액세스 권한이 있는 경우:

1. DGX Spark 장치의 Ubuntu Desktop 환경에 로그인
2. 화면 왼쪽 하단 모서리를 클릭하여 Ubuntu 앱 런처를 엽니다
3. 앱 런처에서 DGX Dashboard 바로 가기를 클릭합니다
4. 대시보드가 기본 웹 브라우저에서 `http://localhost:11000`으로 열립니다

**옵션 B: NVIDIA Sync (원격 액세스 권장)**

로컬 시스템에 NVIDIA Sync가 설치되어 있는 경우:

1. 시스템 트레이에서 NVIDIA Sync 아이콘을 클릭합니다
2. 장치 목록에서 DGX Spark 장치를 선택합니다
3. "Connect"를 클릭합니다
4. "DGX Dashboard"를 클릭하여 대시보드를 시작합니다
5. 대시보드가 자동 SSH 터널을 사용하여 기본 웹 브라우저에서 `http://localhost:11000`으로 열립니다

NVIDIA Sync가 없습니까? [여기에서 설치하십시오](/spark/connect-to-your-spark/sync)

**옵션 C: 수동 SSH 터널**

NVIDIA Sync 없이 수동 원격 액세스를 하려면 먼저 [수동으로 SSH 터널을 구성](/spark/connect-to-your-spark/manual-ssh)해야 합니다.

Dashboard 서버(포트 11000)와 원격으로 액세스하려는 경우 JupyterLab에 대한 터널을 열어야 합니다. 각 사용자 계정에는 JupyterLab에 대해 서로 다른 할당된 포트 번호가 있습니다.

1. DGX Spark에 SSH로 접속하고 다음 명령을 실행하여 할당된 JupyterLab 포트를 확인하십시오:

```bash
cat /opt/nvidia/dgx-dashboard-service/jupyterlab_ports.yaml
```

2. 사용자 이름을 찾아 할당된 포트 번호를 기록해 두십시오.
3. 두 포트를 모두 포함하는 새 SSH 터널을 만드십시오:

```bash
ssh -L 11000:localhost:11000 -L <ASSIGNED_PORT>:localhost:<ASSIGNED_PORT> <USERNAME>@<SPARK_DEVICE_IP>
```
`<USERNAME>`을 DGX Spark 장치 사용자 이름으로, `<SPARK_DEVICE_IP>`를 장치의 IP 주소로 바꾸십시오.

`<ASSIGNED_PORT>`를 YAML 파일의 포트 번호로 바꾸십시오.

웹 브라우저를 열고 `http://localhost:11000`으로 이동하십시오.


## 2단계. DGX Dashboard 로그인

브라우저에서 대시보드가 로드되면:

1. 사용자 이름 필드에 DGX Spark 시스템 사용자 이름을 입력합니다
2. 비밀번호 필드에 시스템 비밀번호를 입력합니다
3. "Login"을 클릭하여 대시보드 인터페이스에 액세스합니다

JupyterLab 관리, 시스템 모니터링 및 설정을 위한 패널이 있는 메인 대시보드가 표시되어야 합니다.

## 3단계. JupyterLab 인스턴스 시작

JupyterLab 환경을 만들고 시작하십시오:

1. 오른쪽 패널에서 "Start" 버튼을 클릭합니다
2. 상태가 다음과 같이 전환되는 것을 모니터링합니다: Starting → Preparing → Running
3. 상태가 "Running"으로 표시될 때까지 기다립니다 (첫 번째 시작 시 몇 분이 걸릴 수 있음)
4. "Running"이 되면 JupyterLab이 브라우저에서 자동으로 열리지 않는 경우 (팝업이 차단됨), "Open In Browser" 버튼을 클릭할 수 있습니다

시작할 때 기본 작업 디렉토리(/home/<USERNAME>/jupyterlab)가 생성되고 가상 환경이 자동으로 설정됩니다. 작업 디렉토리에 생성된 `requirements.txt` 파일을 보면 설치된 패키지를 확인할 수 있습니다.

향후 "Stop" 버튼을 클릭하고 새 작업 디렉토리의 경로를 변경한 다음 "Start" 버튼을 다시 클릭하여 새로운 격리된 환경을 만들어 작업 디렉토리를 변경할 수 있습니다.

## 4단계. 샘플 AI 워크로드로 테스트

간단한 Stable Diffusion XL 이미지 생성 예제를 실행하여 설정을 확인하십시오:

1. JupyterLab에서 새 노트북을 만듭니다: File → New → Notebook
2. "Python 3 (ipykernel)"을 클릭하여 노트북을 만듭니다
3. 새 셀을 추가하고 다음 코드를 붙여넣습니다:

```python
import warnings
warnings.filterwarnings('ignore', message='.*cuda capability.*')
import tqdm.auto
tqdm.auto.tqdm = tqdm.std.tqdm

from diffusers import DiffusionPipeline
import torch
from PIL import Image
from datetime import datetime
from IPython.display import display

## --- Model setup ---
MODEL_ID = "stabilityai/stable-diffusion-xl-base-1.0"
dtype = torch.float16 if torch.cuda.is_available() else torch.float32

pipe = DiffusionPipeline.from_pretrained(
    MODEL_ID,
    torch_dtype=dtype,
    variant="fp16" if dtype==torch.float16 else None,
)
pipe = pipe.to("cuda" if torch.cuda.is_available() else "cpu")

## --- Prompt setup ---
prompt = "a cozy modern reading nook with a big window, soft natural light, photorealistic"
negative_prompt = "low quality, blurry, distorted, text, watermark"

## --- Generation settings ---
height = 1024
width = 1024
steps = 30
guidance = 7.0

## --- Generate ---
result = pipe(
    prompt=prompt,
    negative_prompt=negative_prompt,
    num_inference_steps=steps,
    guidance_scale=guidance,
    height=height,
    width=width,
)

## --- Save to file ---
image: Image.Image = result.images[0]
display(image)
image.save(f"sdxl_output.png")
print(f"Saved image as sdxl_output.png")
```

4. 셀을 실행합니다 (Shift+Enter 또는 Run 버튼 클릭)
5. 노트북이 모델을 다운로드하고 이미지를 생성합니다 (첫 실행 시 몇 분이 걸릴 수 있음)

## 5단계. GPU 사용률 모니터링

이미지 생성이 실행되는 동안:

1. 브라우저에서 DGX Dashboard 탭으로 다시 전환합니다
2. 모니터링 패널에서 GPU 텔레메트리 데이터를 관찰합니다

## 6단계. JupyterLab 인스턴스 중지

세션이 끝나면:

1. 메인 DGX Dashboard 탭으로 돌아갑니다
2. JupyterLab 패널에서 "Stop" 버튼을 클릭합니다
3. 상태가 "Running"에서 "Stopped"로 변경되는지 확인합니다

## 6단계. 시스템 업데이트 관리

시스템 업데이트를 사용할 수 있는 경우 배너 또는 설정 페이지에 표시됩니다.

설정 페이지의 "Updates" 탭에서:

1. "Update"를 클릭하여 확인 대화 상자를 엽니다
2. "Update Now"를 클릭하여 업데이트 프로세스를 시작합니다
3. 업데이트가 완료되고 장치가 재부팅될 때까지 기다립니다

> [!WARNING]
> 시스템 업데이트는 패키지, 펌웨어(사용 가능한 경우)를 업그레이드하고 재부팅을 트리거합니다. 계속하기 전에 작업을 저장하십시오.

## 7단계. 정리 및 롤백

리소스를 정리하고 시스템을 원래 상태로 되돌리려면:

1. 대시보드를 통해 실행 중인 JupyterLab 인스턴스를 중지합니다
2. JupyterLab 작업 디렉토리를 삭제합니다

> [!WARNING]
> 시스템 업데이트를 실행한 경우 유일한 롤백은 시스템 백업 또는 복구 미디어에서 복원하는 것입니다.

정상적인 대시보드 사용 중에는 시스템에 영구적인 변경이 이루어지지 않습니다.

## 8단계. 다음 단계

이제 DGX Dashboard가 구성되었으므로 다음을 수행할 수 있습니다:

- 다양한 프로젝트를 위한 추가 JupyterLab 환경 만들기
- 대시보드를 사용하여 시스템 유지 관리 및 업데이트 관리

## 문제 해결

| 증상 | 원인 | 해결 방법 |
|---------|-------|-----|
| 사용자가 업데이트를 실행할 수 없음 | 사용자가 sudo 그룹에 없음 | sudo 그룹에 사용자 추가: `sudo usermod -aG sudo <USERNAME>`; 그런 다음 `newgrp docker` 실행|
| JupyterLab이 시작되지 않음 | 현재 가상 환경에 문제가 있음 | JupyterLab 패널에서 작업 디렉토리를 변경하고 새 인스턴스를 시작합니다 |
| SSH 터널 연결 거부됨 | IP 또는 포트가 잘못됨 | Spark 장치 IP를 확인하고 SSH 서비스가 실행 중인지 확인합니다 |
| 모니터링에서 GPU가 보이지 않음 | 드라이버 문제 | `nvidia-smi`로 GPU 상태를 확인합니다 |


최신 알려진 문제에 대해서는 [DGX Spark 사용자 가이드](https://docs.nvidia.com/dgx/dgx-spark/known-issues.html)를 참조하십시오.
