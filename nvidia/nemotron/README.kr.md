# llama.cpp를 사용한 Nemotron-3-Nano

> DGX Spark에서 llama.cpp를 사용하여 Nemotron-3-Nano-30B 모델 실행

## 목차

- [개요](#개요)
- [지침](#지침)
- [문제 해결](#문제-해결)

---

## 개요

## 기본 아이디어

Nemotron-3-Nano-30B-A3B는 300억 개의 파라미터 Mixture of Experts (MoE) 아키텍처와 30억 개의 활성 파라미터만을 특징으로 하는 NVIDIA의 강력한 언어 모델입니다. 이 효율적인 설계는 낮은 계산 요구 사항으로 고품질 추론을 가능하게 하여 DGX Spark의 GB10 GPU에 이상적입니다.

이 플레이북은 GPU 아키텍처에 맞게 빌드 시 CUDA 커널을 컴파일하는 llama.cpp를 사용하여 Nemotron-3-Nano를 실행하는 방법을 보여줍니다. 모델에는 채팅 템플릿을 통해 내장된 추론(사고 모드) 및 도구 호출 지원이 포함되어 있습니다.

## 달성할 내용

DGX Spark에서 실행되는 완전히 작동하는 Nemotron-3-Nano-30B-A3B 추론 서버를 갖추게 되며, OpenAI 호환 API를 통해 액세스할 수 있습니다. 이 설정을 통해 다음을 수행할 수 있습니다:

- 로컬 LLM 추론
- 기존 도구와의 쉬운 통합을 위한 OpenAI 호환 API 엔드포인트
- 내장된 추론 및 도구 호출 기능

## 시작하기 전에 알아야 할 사항

- Linux 명령줄 및 터미널 명령에 대한 기본 친숙함
- git 및 브랜치 작업에 대한 이해
- CMake를 사용한 소스에서 소프트웨어 빌드 경험
- 테스트를 위한 REST API 및 cURL에 대한 기본 지식
- 모델 다운로드를 위한 Hugging Face Hub에 대한 친숙함

## 전제 조건

**하드웨어 요구 사항:**
- GB10 GPU가 있는 NVIDIA DGX Spark
- 최소 40GB 사용 가능한 GPU 메모리(모델은 약 38GB VRAM 사용)
- 모델 다운로드 및 빌드 아티팩트를 위한 최소 50GB 사용 가능한 저장 공간

**소프트웨어 요구 사항:**
- NVIDIA DGX OS
- Git: `git --version`
- CMake (3.14+): `cmake --version`
- CUDA Toolkit: `nvcc --version`
- GitHub 및 Hugging Face에 대한 네트워크 액세스

## 시간 및 위험

* **예상 시간:** 30분(약 38GB의 모델 다운로드 포함)
* **위험 수준:** 낮음
  * 빌드 프로세스는 소스에서 컴파일하지만 시스템 파일을 수정하지 않음
  * 중단된 경우 모델 다운로드를 재개할 수 있음
* **롤백:** 클론된 `llama.cpp` 디렉토리 및 다운로드한 모델 파일을 삭제하여 설치를 완전히 제거
* **마지막 업데이트:** 12/17/2025
  * 최초 게시

## 지침

## 1단계. 전제 조건 확인

계속하기 전에 DGX Spark에 필요한 도구가 설치되어 있는지 확인합니다.

```bash
git --version
cmake --version
nvcc --version
```

모든 명령은 버전 정보를 반환해야 합니다. 누락된 것이 있으면 계속하기 전에 설치하세요.

Hugging Face CLI를 설치합니다:

```bash
python3 -m venv nemotron-venv
source nemotron-venv/bin/activate
pip install -U "huggingface_hub[cli]"
```

설치 확인:

```bash
hf version
```

## 2단계. llama.cpp 리포지토리 클론

Nemotron 모델을 실행하기 위한 추론 프레임워크를 제공하는 llama.cpp 리포지토리를 클론합니다.

```bash
git clone https://github.com/ggml-org/llama.cpp
cd llama.cpp
```

## 3단계. CUDA 지원으로 llama.cpp 빌드

CUDA를 활성화하고 GB10의 sm_121 컴퓨팅 아키텍처를 대상으로 llama.cpp를 빌드합니다. 이는 DGX Spark GPU에 특별히 최적화된 CUDA 커널을 컴파일합니다.

```bash
mkdir build && cd build
cmake .. -DGGML_CUDA=ON -DCMAKE_CUDA_ARCHITECTURES="121" -DLLAMA_CURL=OFF
make -j8
```

빌드 프로세스는 약 5-10분 소요됩니다. 컴파일 진행률과 최종적으로 성공적인 빌드 메시지가 표시되어야 합니다.

## 4단계. Nemotron GGUF 모델 다운로드

Hugging Face에서 Q8 양자화 GGUF 모델을 다운로드합니다. 이 모델은 GB10의 메모리 용량 내에 맞으면서 우수한 품질을 제공합니다.

```bash
hf download unsloth/Nemotron-3-Nano-30B-A3B-GGUF \
  Nemotron-3-Nano-30B-A3B-UD-Q8_K_XL.gguf \
  --local-dir ~/models/nemotron3-gguf
```

이는 약 38GB를 다운로드합니다. 중단된 경우 다운로드를 재개할 수 있습니다.

## 5단계. llama.cpp 서버 시작

Nemotron 모델로 추론 서버를 시작합니다. 서버는 OpenAI 호환 API 엔드포인트를 제공합니다.

```bash
./bin/llama-server \
  --model ~/models/nemotron3-gguf/Nemotron-3-Nano-30B-A3B-UD-Q8_K_XL.gguf \
  --host 0.0.0.0 \
  --port 30000 \
  --n-gpu-layers 99 \
  --ctx-size 8192 \
  --threads 8
```

**파라미터 설명:**
- `--host 0.0.0.0`: 모든 네트워크 인터페이스에서 수신
- `--port 30000`: API 서버 포트
- `--n-gpu-layers 99`: 모든 레이어를 GPU로 오프로드
- `--ctx-size 8192`: 컨텍스트 창 크기(최대 1M까지 증가 가능)
- `--threads 8`: GPU가 아닌 작업을 위한 CPU 스레드

모델이 로드되고 준비되었음을 나타내는 서버 시작 메시지가 표시되어야 합니다:
```
llama_new_context_with_model: n_ctx = 8192
...
main: server is listening on 0.0.0.0:30000
```

## 6단계. API 테스트

새 터미널을 열고 OpenAI 호환 채팅 완성 엔드포인트를 사용하여 추론 서버를 테스트합니다.

```bash
curl http://localhost:30000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "nemotron",
    "messages": [{"role": "user", "content": "New York is a great city because..."}],
    "max_tokens": 100
  }'
```

예상 응답 형식:
```json
{
  "choices": [
    {
      "finish_reason": "length",
      "index": 0,
      "message": {
        "role": "assistant",
        "reasoning_content": "We need to respond to user statement: \"New York is a great city because...\". Probably they want continuation, maybe a discussion. It's a simple open-ended prompt. Provide reasons why New York is great. No policy issues. Just respond creatively.",
        "content": "New York is a great city because it's a living, breathing collage of cultures, ideas, and possibilities—all stacked into one vibrant, never‑sleeping metropolis. Here are just a few reasons that many people ("
      }
    }
  ],
  "created": 1765916539,
  "model": "Nemotron-3-Nano-30B-A3B-UD-Q8_K_XL.gguf",
  "object": "chat.completion",
  "usage": {
    "completion_tokens": 100,
    "prompt_tokens": 25,
    "total_tokens": 125
  },
  "id": "chatcmpl-...",
  "timings": {
    ...
  }
}
```

## 7단계. 추론 기능 테스트

Nemotron-3-Nano에는 내장된 추론 기능이 포함되어 있습니다. 더 복잡한 프롬프트로 테스트합니다:

```bash
curl http://localhost:30000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "nemotron",
    "messages": [{"role": "user", "content": "Solve this step by step: If a train travels 120 miles in 2 hours, what is its average speed?"}],
    "max_tokens": 500
  }'
```

모델은 최종 답변을 제공하기 전에 상세한 추론 체인을 제공합니다.

## 8단계. 정리

서버를 중지하려면 실행 중인 터미널에서 `Ctrl+C`를 누릅니다.

설치를 완전히 제거하려면:

```bash
## llama.cpp 빌드 제거
rm -rf ~/llama.cpp

## 다운로드한 모델 제거
rm -rf ~/models/nemotron3-gguf
```

## 9단계. 다음 단계

1. **컨텍스트 크기 증가**: 더 긴 대화를 위해 `--ctx-size`를 최대 1048576(1M 토큰)까지 증가시킵니다. 단, 더 많은 메모리를 사용합니다
3. **애플리케이션과 통합**: Open WebUI, Continue.dev 또는 사용자 정의 애플리케이션과 같은 도구로 OpenAI 호환 API 사용

서버는 스트리밍 응답, 함수 호출 및 멀티턴 대화를 포함한 전체 OpenAI API 사양을 지원합니다.

## 문제 해결

| 증상 | 원인 | 해결 방법 |
|---------|-------|-----|
| `cmake`가 "CUDA not found"로 실패 | CUDA 툴킷이 PATH에 없음 | `export PATH=/usr/local/cuda/bin:$PATH`를 실행하고 재시도 |
| 모델 다운로드 실패 또는 중단됨 | 네트워크 문제 | `hf download` 명령을 다시 실행 - 중단된 곳에서 재개됩니다 |
| 서버 시작 시 "CUDA out of memory" | GPU 메모리 부족 | `--ctx-size`를 4096으로 줄이거나 더 작은 양자화(Q4_K_M) 사용 |
| 서버가 시작되지만 추론이 느림 | 모델이 GPU에 완전히 로드되지 않음 | `--n-gpu-layers 99`가 설정되어 있는지 확인하고 `nvidia-smi`로 GPU 사용량 확인 |
| 포트 30000에서 "Connection refused" | 서버가 실행되지 않거나 잘못된 포트 | 서버가 실행 중인지 확인하고 `--port` 파라미터 확인 |
| API 응답에서 "model not found" | 잘못된 모델 경로 | `--model` 파라미터의 모델 경로가 다운로드한 파일 위치와 일치하는지 확인 |


> [!NOTE]
> DGX Spark는 GPU와 CPU 간에 동적 메모리 공유를 가능하게 하는 통합 메모리 아키텍처(UMA)를 사용합니다.
> 많은 애플리케이션이 여전히 UMA를 활용하도록 업데이트 중이므로 DGX Spark의 메모리 용량 내에 있더라도
> 메모리 문제가 발생할 수 있습니다. 이런 경우 다음과 같이 버퍼 캐시를 수동으로 플러시하세요:
```bash
sudo sh -c 'sync; echo 3 > /proc/sys/vm/drop_caches'
```

최신 알려진 문제에 대해서는 [DGX Spark 사용자 가이드](https://docs.nvidia.com/dgx/dgx-spark/known-issues.html)를 참조하세요.
