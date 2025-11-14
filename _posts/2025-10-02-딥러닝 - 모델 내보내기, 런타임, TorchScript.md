---
layout: post
title: 딥러닝 - 모델 내보내기, 런타임, TorchScript
date: 2025-10-02 15:25:23 +0900
category: 딥러닝
---
# 모델 내보내기(ONNX), 런타임(TensorRT 개론), TorchScript 개념

**PyTorch 모델을 배포-가속하기 위한 3가지 경로: ONNX → (ONNXRuntime/TensorRT), TorchScript(Libtorch/모바일)**

## 큰 그림과 선택 가이드

| 경로 | 장점 | 단점/유의 | 언제 쓰나 |
|---|---|---|---|
| **TorchScript** (`torch.jit.script/trace`) | PyTorch 오퍼레이터와 타입을 거의 그대로 지원. **Libtorch(C++)**/모바일(Lite Interpreter)로 손쉬운 이식 | Python 문법 제약(스크립팅), 동적 제어흐름은 `trace`로 불가, 일부 최신 연산/커스텀 확장 이슈 | **C++ 서비스**, **모바일(Pytorch Lite)**, PyTorch 생태계 내에서 끝내고 싶을 때 |
| **ONNX** (`torch.onnx.export`) | 프레임워크 중립 표준(그래프), 다양한 백엔드(ONNXRuntime, TensorRT, OpenVINO 등) | 변환 불가 오퍼/shape 추론 불가 케이스, opset·동적 shape 주의 | 멀티백엔드 배포, **TensorRT/ONNXRuntime** 쓰고 싶을 때 |
| **TensorRT**(ONNX→TRT) | NVIDIA GPU에서 **최대 성능**(연산 fusion, FP16/INT8, tactic 선택, profile) | NVIDIA HW 종속, 미지원 op는 **플러그인** 필요, 빌드 시간이 걸릴 수 있음 | **지연 제한(SLA)·대량 트래픽**·서빙 비용 절감이 핵심일 때 |

**현업 추천 루트**
1) PyTorch 학습 → **ONNX Export** → **ONNXRuntime CPU/GPU**로 기능 검증 → **TensorRT 엔진**으로 성능 극대화.
2) PyTorch 학습 → **TorchScript**로 저장 → **Libtorch(C++)/모바일** 배포(생태계 일관).

---

## ONNX 내보내기 (PyTorch → ONNX)

### 베이스라인 예제 모델

```python
import torch, torch.nn as nn, torch.nn.functional as F

class SmallCNN(nn.Module):
    def __init__(self, num_classes=10):
        super().__init__()
        self.conv1 = nn.Conv2d(3, 32, 3, padding=1, bias=False)
        self.bn1   = nn.BatchNorm2d(32)
        self.conv2 = nn.Conv2d(32, 64, 3, padding=1, bias=False)
        self.bn2   = nn.BatchNorm2d(64)
        self.head  = nn.Linear(64, num_classes)
    def forward(self, x):
        x = self.conv1(x); x = self.bn1(x); x = F.relu(x)
        x = self.conv2(x); x = self.bn2(x); x = F.relu(x)
        x = F.adaptive_avg_pool2d(x, 1).flatten(1)
        return self.head(x)

model = SmallCNN().eval()  # ★ 반드시 eval 모드 (BN/Dropout 고정)
dummy = torch.randn(1,3,224,224)
```

### 가장 단순한 내보내기

```python
torch.onnx.export(
    model, dummy, "smallcnn.onnx",
    input_names=["input"], output_names=["logits"],
    opset_version=17,  # 보통 13~18 권장
    do_constant_folding=True  # 상수 폴딩으로 그래프 단순화
)
```

### **동적 크기**(배치/H/W) 지원 — `dynamic_axes`

```python
torch.onnx.export(
    model, dummy, "smallcnn_dynamic.onnx",
    input_names=["input"], output_names=["logits"],
    dynamic_axes={
        "input":  {0: "batch", 2: "height", 3: "width"},
        "logits": {0: "batch"}
    },
    opset_version=17
)
```
> **의미**: ONNX 그래프에 “0축은 batch, 2/3축은 가변 크기”라는 힌트를 남겨 **런타임에서 다양한 입력 크기**를 받을 수 있게 합니다.
> (TensorRT에선 **Optimization Profile**과 매칭됩니다 — 아래 2.5절)

### ONNX 검증(ONNXRuntime로 기능 확인)

```python
import onnx, onnxruntime as ort, numpy as np

onnx_model = onnx.load("smallcnn_dynamic.onnx")
onnx.checker.check_model(onnx_model)  # 구조/타입 체크

sess = ort.InferenceSession(
    "smallcnn_dynamic.onnx",
    providers=["CPUExecutionProvider"]  # GPU면 "CUDAExecutionProvider"
)
x = np.random.randn(2,3,224,224).astype("float32")
y = sess.run(["logits"], {"input": x})[0]
print(y.shape)  # (2, 10)
```

### 흔한 변환 이슈 & 대응

- **`model.train()` 상태로 export**: BN/Dropout이 학습 모드 → 결과 불안정. **`eval()`** 필수.
- **지원되지 않는 연산**:
  - 대체 경로(공식/커스텀 함수)로 바꾸거나, **TorchScript로 우회**(ONNX 불가면 TorchScript 경로 고려).
  - ONNX opset을 **올리거나/내리기**로 해결되는 경우도 있음.
- **동적 제어 흐름**(데이터 의존 분기/loop) → ONNX로 정적화 어려움.
  - 미리 torch 연산으로 **벡터화**하거나, **TorchScript 경로** 고려.
- **정밀도 차이**: log/exp/softmax 등 **수치 안정** 구현 차이 → 허용 오차 범위(`rtol/atol`) 내 비교.

### 그래프 단순화 & 모양 추론

- **ONNX-Simplifier(onnxsim)** 로 불필요 노드 제거/상수 폴딩 → 변환 성공률/속도↑
- **onnx.shape_inference** 로 텐서 shape 주석 추가 → 디버깅 용이

```python
import onnx
from onnx import shape_inference
m = onnx.load("smallcnn_dynamic.onnx")
m = shape_inference.infer_shapes(m)
onnx.save(m, "smallcnn_dynamic_shaped.onnx")
```

### (심화) 새 API: `torch.onnx.dynamo_export` 개념

- PyTorch 2.x의 **Dynamo 기반 exporter**는 트레이싱 덜 민감, 더 많은 동적 패턴 지원(환경별 가용성 확인).
- 기본 아이디어는 동일: **예시 입력**에서 그래프 캡처 → ONNX 변환.

---

## TensorRT 개론: 빌더–엔진–컨텍스트

### 핵심 객체

- **Network**: 연산 그래프(ONNX를 파싱하여 구성)
- **Builder**: 최적화/전략(tactic) 검색 및 **Engine** 생성
- **Engine**(Serialized Engine): 하드웨어/드라이버에 최적화된 **실행 바이너리**
- **Execution Context**: 엔진의 1회 실행 상태(입출력 바인딩, 스트림 등)

### 가장 빠른 시작 — `trtexec` CLI

```bash
# FP16 엔진 생성 (가능한 하드웨어에서)

trtexec --onnx=smallcnn_dynamic.onnx --saveEngine=smallcnn_fp16.plan --fp16 \
        --minShapes=input:1x3x224x224 \
        --optShapes=input:8x3x224x224 \
        --maxShapes=input:16x3x384x384 \
        --workspace=4096 --verbose
```
- **프로파일(동적 크기)**: 입력 텐서마다 **min/opt/max** 세트가 필요.
- **fp16/int8**: 하드웨어가 지원하면 큰 성능 향상. INT8은 **교정(PTQ)** 또는 **Q/DQ(QAT)** 기반.
- **workspace**: tactic 탐색에 쓰는 GPU 메모리(MB). 부족하면 성능 저하 가능.

테스트:
```bash
trtexec --loadEngine=smallcnn_fp16.plan --shapes=input:8x3x224x224 --separateProfileRun
```

### Python API로 엔진 빌드(ONNX→TRT)

```python
import tensorrt as trt

def build_engine_from_onnx(onnx_path, fp16=True, int8=False, profiles=None, max_workspace=4<<30):
    logger = trt.Logger(trt.Logger.INFO)
    builder = trt.Builder(logger)
    network_flags = 1 << int(trt.NetworkDefinitionCreationFlag.EXPLICIT_BATCH)
    network = builder.create_network(flags=network_flags)
    parser = trt.OnnxParser(network, logger)

    with open(onnx_path, "rb") as f:
        assert parser.parse(f.read()), parser.get_error(0)

    config = builder.create_builder_config()
    config.set_memory_pool_limit(trt.MemoryPoolType.WORKSPACE, max_workspace)
    if fp16 and builder.platform_has_fast_fp16: config.set_flag(trt.BuilderFlag.FP16)
    if int8 and builder.platform_has_fast_int8:
        config.set_flag(trt.BuilderFlag.INT8)
        # int8 calibration을 쓰려면 calibrator 설정 필요(아래 2.4 참조)

    # 동적 shape → Optimization Profile
    if profiles:
        profile = builder.create_optimization_profile()
        for name, (mn, op, mx) in profiles.items():
            profile.set_shape(name, mn, op, mx)
        config.add_optimization_profile(profile)

    engine = builder.build_engine(network, config)
    serialized = engine.serialize()
    with open("smallcnn_fp16.plan", "wb") as f:
        f.write(serialized)
    return "smallcnn_fp16.plan"

profiles = {
  "input": ((1,3,224,224), (8,3,224,224), (16,3,384,384))
}
build_engine_from_onnx("smallcnn_dynamic.onnx", fp16=True, int8=False, profiles=profiles)
```

### 정밀도 옵션( FP32 / FP16 / INT8 )

- **FP16**: 대개 **정확도 손실 미미** + 지연/스루풋 개선.
- **INT8**:
  - **PTQ(교정)**: 대표 데이터로 **활성 분포** 스케일 추정(정확도 민감).
  - **QAT**: 학습 중 **Q/DQ** 노드가 포함된 ONNX면 **교정 없이** INT8 빌드 가능(정확도 가장 좋음).

(PTQ 캘리브레이터 초간단 골격)
```python
# PyCalibrator 스케치(실전은 tensorrt.IInt8EntropyCalibrator2 등 상속 필요)

class MyCalibrator(trt.IInt8EntropyCalibrator2):
    def __init__(self, loader, input_name):
        super().__init__()
        self.loader = iter(loader); self.input_name = input_name
        # allocate device buffers, etc.
    def get_batch_size(self): return 8
    def get_batch(self, names):
        # load next batch → copy to device → return device ptr list
        return [device_ptr_for(self.input_name)]
    def read_calibration_cache(self): return None
    def write_calibration_cache(self, cache): pass
```
> **QAT 경로**: PyTorch QAT → ONNX에 **QuantizeLinear/DequantizeLinear(Q/DQ)** 포함 → TRT가 그대로 해석(교정 불필요).

### 동적 입력 & 최적화 프로파일

- TRT는 입력마다 **min/opt/max shape**를 정의해야 tactic을 고릅니다.
- 런타임에서는 **프로파일 범위 내**의 shape만 허용됩니다(범위 바깥 → 오류/성능저하).
- 실전 팁: **요청의 분포**(예: 224, 256, 384)를 보고 **여러 프로파일**을 준비하면 안정적.

### 실행(엔진 로딩 & 추론)

```python
import numpy as np, pycuda.driver as cuda, pycuda.autoinit
import tensorrt as trt

def infer_trt(engine_path, batch=8, h=224, w=224):
    logger = trt.Logger(trt.Logger.ERROR)
    runtime = trt.Runtime(logger)
    with open(engine_path, "rb") as f:
        engine = runtime.deserialize_cuda_engine(f.read())
    context = engine.create_execution_context()

    # I/O 바인딩(이름은 onnx→trt로 전달됨)
    input_idx  = engine.get_binding_index("input")
    output_idx = engine.get_binding_index("logits")
    context.set_binding_shape(input_idx, (batch,3,h,w))
    out_shape = tuple(context.get_binding_shape(output_idx))

    # GPU 메모리 할당
    def alloc(n_bytes): return cuda.mem_alloc(n_bytes)
    x_host = np.random.randn(* (batch,3,h,w)).astype(np.float32)
    x_dev  = alloc(x_host.nbytes)
    y_dev  = alloc(np.prod(out_shape)*4)
    cuda.memcpy_htod(x_dev, x_host)

    bindings = [None]*engine.num_bindings
    bindings[input_idx]  = int(x_dev)
    bindings[output_idx] = int(y_dev)

    stream = cuda.Stream()
    context.execute_async_v2(bindings, stream.handle)
    y_host = np.empty(out_shape, dtype=np.float32)
    cuda.memcpy_dtoh_async(y_host, y_dev, stream); stream.synchronize()
    return y_host

y = infer_trt("smallcnn_fp16.plan", batch=8, h=224, w=224)
print(y.shape)  # (8, 10)
```

### 디버깅·성능 팁

- **`--verbose`** 로 레이어별 tactic/정밀도 확인.
- **Fusion**: Conv+BN(+ReLU)는 자동 결합. **export 전에 `eval()`** 이 필수(런타임이 BN을 상수로 접기 쉽도록).
- **Plugin**: 미지원 op는 **커스텀 플러그인**으로 구현 가능(고급).
- **Batch=1 지연** 최적화: 프로파일에서 `min=opt=max=(1, …)` 엔진을 별도 만들면 더 빠를 때가 많음.
- **멀티스트림**: 동시 요청 많으면 **컨텍스트 여러 개** 혹은 **CUDA 스트림**으로 동시 실행.

---

## TorchScript 개념: `script` vs `trace`

### TorchScript 란?

- **PyTorch 모델을 Python 런타임 없이 실행**하기 위해, **연산 그래프**와 **런타임**(Libtorch)을 담은 중간 표현.
- **저장**: `model_jit = torch.jit.script(model)` 또는 `torch.jit.trace` → `model_jit.save("m.pt")`
- **로드**: Python(`torch.jit.load`) 또는 **C++(libtorch)** 에서 로드/실행.

### `script`(스크립팅) vs `trace`(트레이싱)

- **`script`**: 함수/모듈을 **해석**하여 TorchScript로 변환 → **데이터 의존 분기/루프** 지원.
  - 제약: **타입 주석**, 일부 Python API 불가(파일 I/O, `.numpy()` 등), Tensor 연산만 허용.
- **`trace`**: **예시 입력** 경로를 **기록** → 단순/고정 흐름 모델은 빠르게 변환.
  - **데이터 의존 제어흐름**은 기록되지 않아 **오류/누락 가능**.

### 스크립팅 예제

```python
import torch, torch.nn as nn, torch.nn.functional as F
from typing import Tuple

class MyBlock(nn.Module):
    def __init__(self, ch: int):
        super().__init__()
        self.conv = nn.Conv2d(ch, ch, 3, padding=1)
        self.bn   = nn.BatchNorm2d(ch)
    def forward(self, x: torch.Tensor) -> torch.Tensor:
        y = self.conv(x); y = self.bn(y); return F.relu(y)

class MyNet(nn.Module):
    def __init__(self, num_classes: int = 10):
        super().__init__()
        self.stem = MyBlock(32)
        self.head = nn.Linear(32, num_classes)
    def forward(self, x: torch.Tensor) -> torch.Tensor:
        x = self.stem(x)
        x = F.adaptive_avg_pool2d(x, 1).flatten(1)
        return self.head(x)

m = MyNet().eval()
m_jit = torch.jit.script(m)         # ★ script
m_jit.save("m_script.pt")
m2 = torch.jit.load("m_script.pt")
```

### 트레이싱 예제(+주의)

```python
m = MyNet().eval()
example = torch.randn(1,32,32,32)
m_trace = torch.jit.trace(m, example)  # 데이터 의존 분기/루프가 없을 때 OK
m_trace.save("m_trace.pt")
```
> **주의**: 입력 크기나 제어 흐름에 따라 경로가 바뀌는 모델은 **`trace` 부적합**.
> `torch.jit.trace(..., check_inputs=[...])` 로 몇 가지 입력을 더 넣어 **검증**을 추가.

### C++ (Libtorch) 추론 예시

```cpp
// infer.cpp (컴파일 시 -ltorch_cpu -lc10 등 링크 필요)
#include <torch/script.h>
#include <iostream>

int main() {
  torch::jit::script::Module mod = torch::jit::load("m_script.pt");
  mod.eval();
  std::vector<torch::jit::IValue> inputs;
  inputs.push_back(torch::randn({1, 32, 32, 32}));
  at::Tensor out = mod.forward(inputs).toTensor();
  std::cout << out.sizes() << std::endl;
  return 0;
}
```

### 모바일(Lite Interpreter) 개관

- TorchScript를 **Lite Interpreter** 포맷으로 줄여 **모바일 앱**에 포함.
- `torch.utils.mobile_optimizer` 로 최적화 후 `torch.jit.save`(lite 모드).
- Android/iOS에서 **libtorch lite**로 로드.

### TorchScript 사용 규칙(체크)

- 텐서 외부의 **Python 오브젝트/사이드이펙트** 금지(파일 I/O, OS 호출 등).
- `.numpy()`, `.item()` 남발 지양(그래프 끊김/비결정).
- 동적 제어흐름은 **스크립팅**으로 가능하지만, **타입 주석**·`if isinstance` 등 제약에 맞춰 작성.
- 커스텀 연산은 **TorchScript ops** 또는 **C++ extension**으로 제공.

---

## 엔드-투-엔드 미니 프로젝트

### 분류 모델: PyTorch → ONNX → ONNXRuntime → TensorRT

```python
# 학습된 모델 로드 & eval

model = SmallCNN(); model.load_state_dict(torch.load("smallcnn_fp32.pth")); model.eval()

# ONNX export (동적 axes)

dummy = torch.randn(1,3,224,224)
torch.onnx.export(model, dummy, "smallcnn.onnx",
    input_names=["input"], output_names=["logits"],
    dynamic_axes={"input":{0:"batch",2:"h",3:"w"}, "logits":{0:"batch"}},
    opset_version=17, do_constant_folding=True)

# ORT 검증

import onnxruntime as ort, numpy as np
sess = ort.InferenceSession("smallcnn.onnx", providers=["CPUExecutionProvider"])
x = np.random.randn(4,3,224,224).astype(np.float32)
pred = sess.run(["logits"], {"input": x})[0]

# TRT 엔진 빌드 (trtexec or Python API)
#    trtexec --onnx=smallcnn.onnx --saveEngine=smallcnn_fp16.plan --fp16 \
#            --minShapes=input:1x3x224x224 --optShapes=input:8x3x224x224 --maxShapes=input:16x3x384x384

# TRT 추론 (2.6 코드 재사용)

y = infer_trt("smallcnn_fp16.plan", batch=8, h=224, w=224)
```

### 동일 모델: TorchScript(C++/모바일 용)

```python
# Python 저장

m = SmallCNN().eval()
m.load_state_dict(torch.load("smallcnn_fp32.pth"))
m_jit = torch.jit.script(m)
m_jit.save("smallcnn_script.pt")

# C++ 서비스에 smallcnn_script.pt 배포 → Libtorch로 로드/추론

```

---

## 운영 체크리스트

### ONNX Export 체크

- [ ] `model.eval()`(BN/Dropout 고정)
- [ ] `opset_version`(보통 17 근처) 지정
- [ ] **동적 축**(batch/H/W) 필요 시 `dynamic_axes` 지정
- [ ] **ONNXRuntime**로 결과/shape/수치 검증
- [ ] 변환 불가 op → 대체/커스텀/버전 조정

### TensorRT 빌드/추론 체크

- [ ] **프로파일(min/opt/max)** 실제 트래픽 범위를 반영
- [ ] FP16 가능 여부 확인(하드웨어) / INT8은 **교정** or **Q/DQ**
- [ ] `--workspace` 충분히 크게(수 GB) 후 성능 비교
- [ ] `--verbose` 로 tactic/정밀도 확인, 병목 레이어 탐색
- [ ] 배치=1 초저지연 엔진 별도 고려

### TorchScript 체크

- [ ] **스크립팅**으로 동적 제어흐름 커버 / `trace`는 고정경로 모델에만
- [ ] 타입 주석 / Python 사이드이펙트 제거
- [ ] `torch.jit.load`(Python) & **Libtorch(C++)** 둘 다 smoke-test
- [ ] 모바일이면 **Lite Interpreter** 변환 + 크기/지연 측정

---

## 자주 묻는 질문 (FAQ)

**Q1. ONNX로 안 나가는 연산이 있어요.**
A. 같은 기능의 **대체 연산 조합**으로 바꾸거나, **TorchScript 경로**를 고려하세요. ONNX opset을 한두 버전 조정해도 풀릴 때가 있습니다.

**Q2. ONNXRuntime와 PyTorch 결과가 조금 달라요.**
A. 수치 안정/epsilon 차이 때문일 수 있습니다. `rtol/atol`을 둔 비교로 확인하고, BN/Softmax/LogSumExp 근처를 의심해 보세요.

**Q3. TensorRT에서 성능이 생각만큼 안 나와요.**
A. 프로파일의 opt/max가 실제 입력과 동떨어졌거나, tactic 탐색 메모리(`workspace`)가 부족했을 수 있습니다. **Batch=1 전용 엔진**, **FP16/INT8** 옵션, **플러그인** 검토도 팁입니다.

**Q4. TorchScript로 trace했더니 동작이 달라요.**
A. 입력값에 따라 제어 흐름이 바뀌는 코드입니다. **`script`** 로 바꾸거나, `check_inputs`를 넉넉히 넣어 검증하세요.

**Q5. INT8 하고 싶지만 정확도가 떨어져요.**
A. **QAT**(학습 시 FakeQuant) 경로가 가장 안전합니다. PTQ면 **대표 캘리브레이션 데이터 다양화**, **히스토그램/percentile observer** 조정, **Outlier 처리**를 시도하세요.

---

## 덤: 실전 성능 측정 스니펫

### ONNXRuntime(배치별) 시간

```python
import time, numpy as np, onnxruntime as ort
sess = ort.InferenceSession("smallcnn.onnx", providers=["CPUExecutionProvider"])
for bs in [1,2,4,8]:
    x = np.random.randn(bs,3,224,224).astype(np.float32)
    for _ in range(5): sess.run(["logits"], {"input": x})  # warmup
    t0=time.time()
    for _ in range(30): sess.run(["logits"], {"input": x})
    print(f"ORT bs={bs}: {(time.time()-t0)/30*1000:.2f} ms")
```

### TensorRT(배치별) 시간

```python
def bench_trt(engine_path, shapes=[(1,224,224),(8,224,224),(16,384,384)], iters=50):
    import time
    for bs,h,w in shapes:
        infer_trt(engine_path, 1, 1, 1)  # warmup call (dummy)
        t0=time.time()
        for _ in range(iters):
            _ = infer_trt(engine_path, batch=bs, h=h, w=w)
        print(f"TRT bs={bs} {h}x{w}: {(time.time()-t0)/iters*1000:.2f} ms")
bench_trt("smallcnn_fp16.plan")
```

---

### 마무리

- **ONNX**는 “**호환성**”과 “**백엔드 선택권**”을, **TensorRT**는 “**NVIDIA GPU에서의 극한 성능**”을, **TorchScript**는 “**PyTorch 생태계 내의 완결형 배포(C++/모바일)**”를 제공합니다.
- 한 줄 요약: **기능 검증은 ONNXRuntime, 성능은 TensorRT, C++/모바일은 TorchScript** — 이 조합으로 대부분의 산업 배포 시나리오를 커버할 수 있습니다.
- 위 템플릿을 바로 붙여 돌려보며, **동적 크기/프로파일/정밀도 옵션**만 바꿔도 체감 성능이 크게 바뀌는 것을 확인해 보세요.
