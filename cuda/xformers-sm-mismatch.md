# xformers sm 아키텍처 불일치

H100/A100 에서 `stable-video-diffusion` / `StereoCrafter` / 기타 diffusers 파이프라인 실행 시 자주 만나는 에러.

## 증상

```
FATAL: kernel 'fmha_cutlassF_f32_aligned_64x64_rf_sm80' is for sm80-sm100,
but was built for sm50
```

또는

```
NotImplementedError: No operator found for `memory_efficient_attention_forward`
with inputs: ... (H100 / sm_90)
```

GPU는 H100 (sm_90) 또는 A100 (sm_80)인데, pip로 설치된 `xformers`가 **sm_50 같은 옛날 아키텍처만 포함**된 precompiled wheel이라서 발생.

## 원인

`xformers`의 PyPI 공식 precompiled wheel은 빌드 당시 타겟으로 했던 sm arch만 포함합니다. 배포된 wheel이 H100/A100 를 지원 안 하거나, uv/pip가 현재 torch/CUDA에 맞지 않는 옛 버전을 가져왔을 때 이 에러가 납니다.

특히 **uv로 관리되는 `.venv` 환경에서 더 자주 발생**합니다 — uv의 resolver가 호환 wheel을 더 공격적으로 선택해서 오래된 버전을 끌어오는 경향 때문.

## 해결 — 3가지 전략

### 전략 A (권장): xformers 완전 제거

```bash
# conda 환경
conda activate myenv
pip uninstall -y xformers

# uv / .venv 환경
source .venv/bin/activate
uv pip uninstall xformers   # 또는 pip uninstall -y xformers
```

xformers가 import되지 않으면 **diffusers 계열 대부분은 torch 2.0+ 내장 SDPA로 자동 fallback** 합니다. 성능은 약간 떨어지지만 기능상 차이 없음.

**`XFORMERS_DISABLED=1` 환경변수만 설정하는 건 안 먹히는 경우가 많습니다** — 어떤 diffusers 버전은 이 플래그를 무시하고 무조건 xformers를 import 시도합니다. 완전 제거가 가장 확실.

### 전략 B: torch 버전과 맞춘 xformers 재설치

성능이 중요하면 H100 지원하는 wheel 설치:

```bash
# 먼저 현재 torch/cuda 버전 확인
python -c "import torch; print(torch.__version__, torch.version.cuda)"

# torch 2.0.1+cu118 기준 예시
pip uninstall -y xformers
pip install xformers==0.0.22 --index-url https://download.pytorch.org/whl/cu118
```

### xformers 버전 매트릭스 (대략)

| torch | CUDA | 권장 xformers |
|---|---|---|
| 2.0.1 | cu118 | `0.0.22` |
| 2.1.0 | cu118 | `0.0.23` |
| 2.2.0 | cu118 | `0.0.24` |
| 2.3.0 | cu118 | `0.0.26.post1` |
| 2.4.0 | cu121/cu124 | `0.0.28.post1` |
| 2.5.0 | cu121/cu124 | `0.0.28.post2` |

**주의**:
- torch 버전이 바뀌면 다른 패키지 depenency 깨질 수 있음
- CUDA 11.8용 precompiled wheel들은 H100 지원 없을 가능성 높음 (cu121+ 권장)

### 전략 C (최후 수단): 소스 빌드

특정 torch 버전을 유지하면서 H100 xformers 지원이 꼭 필요한 경우:

```bash
# multi-arch 명시 (H100 + A100)
export TORCH_CUDA_ARCH_LIST="8.0;9.0"

# 기존 제거
pip uninstall -y xformers

# 소스에서 빌드 (30분 ~ 1시간 소요)
pip install -v -U git+https://github.com/facebookresearch/xformers.git@main#egg=xformers
```

빌드 성공 후 검증:
```bash
python -c "import xformers; print(xformers.__version__)"
python -c "
import torch
import xformers.ops as xops
q = k = v = torch.randn(1, 256, 8, 64, device='cuda', dtype=torch.float16)
out = xops.memory_efficient_attention(q, k, v)
print('xformers OK on', torch.cuda.get_device_name(0))
"
```

---

## 진단 — 내 환경 상태 확인

```bash
# 1) xformers 설치 여부 & 버전
python -c "
try:
    import xformers
    print('xformers:', xformers.__version__, 'at', xformers.__file__)
except ImportError:
    print('xformers: NOT installed')
"

# 2) torch / cuda / gpu
python -c "
import torch
print('torch:', torch.__version__, 'cuda:', torch.version.cuda)
print('gpu:', torch.cuda.get_device_name(0))
print('capability:', torch.cuda.get_device_capability(0))
"

# 3) XFORMERS_DISABLED 환경변수 효과 체크
XFORMERS_DISABLED=1 python -c "
import os
print('env:', os.environ.get('XFORMERS_DISABLED'))
import xformers   # 여전히 import 됨 — 환경변수는 단지 '힌트'
print('loaded anyway:', xformers.__file__)
"

# 4) diffusers / 사용 중인 파이프라인 코드에서 xformers 직접 호출 여부
grep -rn "enable_xformers\|memory_efficient_attention\|xformers" \
    .venv/lib/python*/site-packages/diffusers/ 2>/dev/null | head -5
```

---

## 실전 사례

### stereocrafter-3d-photo (2026-04-21, 슈퍼컴 H100)

증상:
```
trajA_splatting_results, trajA_inpainting_results_sbs, left_eye 까지 생성 후
Stage C 진입 직전에:
FATAL: kernel 'fmha_cutlassF_f32_aligned_64x64_rf_sm80' is for sm80-sm100,
but was built for sm50
```

해결: `pip uninstall -y xformers` (전략 A) → Stage C 정상 진행. 속도 손실은 체감 안 될 정도.

`XFORMERS_DISABLED=1`은 효과 없었음 — StereoCrafter / diffusers pipeline 일부가 이 플래그를 무시하고 xformers를 import.

### 로컬 WSL2 / conda 환경 (2026-04-19)

이전에는 H100 로컬 서버에서 동일 증상을 `XFORMERS_DISABLED=1`로 해결했으나, 슈퍼컴에서는 uv가 다른 xformers 버전을 설치해서 동일 플래그가 통하지 않음. **환경별로 동작이 다를 수 있다는 점** 유의.

---

## 한 줄 요약

```bash
# 99% 케이스의 해결책
pip uninstall -y xformers
```

속도 최적화 없어도 파이프라인은 잘 돌아감. 성능이 필요한 프로덕션에서만 전략 B/C 고려.

---

→ 관련: [[cuda/multi-arch-빌드]] (커스텀 CUDA extension 다중 arch 빌드 — xformers 소스 빌드에도 적용), [[pip/빌드-격리]] (빌드 시 `--no-build-isolation` 필요한 상황)
