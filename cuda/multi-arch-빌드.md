# CUDA Multi-Arch 빌드 (TORCH_CUDA_ARCH_LIST)

슈퍼컴처럼 **GPU를 동적으로 할당**받는 환경, 또는 여러 장비(3090 / A100 / H100)에서 같은 코드를 돌리는 경우에 꼭 필요한 지식.

> 현상 요약: 한 GPU에서 빌드한 커스텀 CUDA extension이 다른 GPU에 올라가면 `RuntimeError: CUDA error: no kernel image is available for execution on the device`.
>
> 원인: 빌드 시점의 GPU **단일 아키텍처**만 바이너리에 포함됨.
>
> 해결: `TORCH_CUDA_ARCH_LIST`로 빌드 시 **여러 아키텍처 동시 컴파일**.

## GPU → Compute Capability 매핑

| GPU | Compute Capability | sm 태그 |
|---|:---:|:---:|
| RTX 2080 Ti | 7.5 | sm_75 |
| **V100** | **7.0** | **sm_70** |
| RTX 30xx (3090 등) | 8.6 | sm_86 |
| **A100** | **8.0** | **sm_80** |
| RTX 40xx (4090 등) | 8.9 | sm_89 |
| **H100** | **9.0** | **sm_90** |

## 왜 필요한가

PyTorch 커스텀 CUDA extension을 빌드하면, `setup.py`/`nvcc`는 **환경변수 `TORCH_CUDA_ARCH_LIST`** (또는 기본값: 빌드 머신의 GPU)를 보고 어떤 arch로 컴파일할지 결정합니다. 설정 안 하면 빌드 시 보이는 GPU만 타겟.

```
빌드 환경: H100 (sm_90) → .so에 sm_90만 포함
      ↓
실행 환경: A100 (sm_80) 할당 → kernel mismatch → "no kernel image..." 에러
```

## 해결법: 처음부터 multi-arch 빌드

```bash
# 커스텀 extension 설치 *직전에* 환경변수 설정
export TORCH_CUDA_ARCH_LIST="7.0;8.0;9.0"

# 다음 중 하나로 설치
./install.sh                      # Forward-Warp 등
pip install .                     # setup.py 있는 프로젝트
pip install -e .                  # editable install
python setup.py develop           # 구식 방식
```

이 한 줄이면 `.so`에 sm_70, sm_80, sm_90 바이너리가 모두 들어가서 **어느 GPU에 올려도 동작**합니다.

### 장단점

- **장점**: 한 번 빌드 → 모든 노드에서 동작. 실행 성능은 단일 arch와 **동일** (실행 시 자기 arch 바이너리 선택).
- **단점**: `.so` 파일 크기 소폭 증가 (보통 2~3배, 수십 MB 수준). 빌드 시간 길어짐 (arch 수만큼).

## 영구화 — conda activate.d

매번 export 까먹지 않게 환경에 묶어둠:

```bash
conda activate myenv

mkdir -p $CONDA_PREFIX/etc/conda/activate.d
cat > $CONDA_PREFIX/etc/conda/activate.d/cuda_arch.sh <<'EOF'
# Multi-arch CUDA build target for custom extensions
# V100=7.0, A100=8.0, H100=9.0
export TORCH_CUDA_ARCH_LIST="7.0;8.0;9.0"
EOF
```

→ conda 관련 일반 패턴은 [[conda/환경-관리]] 참조.

## 영구화 — uv/.venv

`source .venv/bin/activate` 끝에 붙이거나, 프로젝트 루트에 별도 `env.sh`:

```bash
# .venv/bin/activate 끝에
echo '' >> .venv/bin/activate
echo '# Multi-arch CUDA build target' >> .venv/bin/activate
echo 'export TORCH_CUDA_ARCH_LIST="7.0;8.0;9.0"' >> .venv/bin/activate
```

또는 direnv 쓰는 경우 `.envrc`에:
```bash
export TORCH_CUDA_ARCH_LIST="7.0;8.0;9.0"
```

## 검증

빌드한 `.so`가 실제로 multi-arch인지 확인:

```bash
# 1. 설치 경로 찾기
python -c "import Forward_Warp, os; print(os.path.dirname(Forward_Warp.__file__))"

# 2. cuobjdump로 arch 확인
cuobjdump --list-elf /path/to/extension/*.so | grep sm_
# 또는
cuobjdump --list-ptx /path/to/extension/*.so | grep sm_
```

다음처럼 여러 sm이 나오면 성공:
```
compute_70  sm_70
compute_80  sm_80
compute_90  sm_90
```

단 하나만 나오면 `TORCH_CUDA_ARCH_LIST`가 안 먹힌 것.

## 재빌드 (이미 단일 arch로 설치했다면)

```bash
# 0. 환경변수 먼저
export TORCH_CUDA_ARCH_LIST="7.0;8.0;9.0"

# 1. 기존 빌드 아티팩트 완전 제거
cd <extension_dir>
rm -rf build/ *.egg-info/ dist/
find . -name "__pycache__" -type d -exec rm -rf {} + 2>/dev/null
find . -name "*.so" -delete

# 2. 패키지 언인스톨
pip uninstall -y <package_name>

# 3. PyTorch JIT extension 캐시도 클리어 (JIT 로딩 방식이면)
rm -rf ~/.cache/torch_extensions/

# 4. 환경변수 정말 set 되었는지 확인
echo "TORCH_CUDA_ARCH_LIST=$TORCH_CUDA_ARCH_LIST"

# 5. 재설치
./install.sh    # 또는 pip install . 등
```

### 재빌드 시 빌드 로그에서 확인할 것

`nvcc` 호출 로그에 `-gencode` 플래그가 여러 개 나와야 함:
```
... -gencode=arch=compute_70,code=sm_70 \
    -gencode=arch=compute_80,code=sm_80 \
    -gencode=arch=compute_90,code=sm_90 ...
```

한 줄만 보이면 환경변수가 빌드 프로세스에 전파되지 않은 것.

## 디버깅 체크리스트

"재빌드 했는데 여전히 `no kernel image...` 에러나는 경우":

1. **재빌드가 실제로 일어났는지**: `.so` 파일의 수정 시간(`ls -la *.so`)이 방금 전이어야 함.
2. **구 설치가 남아있는지**: `pip list | grep -i <pkg>`로 중복 설치 확인. site-packages에 두 번 설치돼 있으면 어느 쪽이 import되는지 복불복.
   ```bash
   python -c "import <pkg>; print(<pkg>.__file__)"
   ```
3. **`TORCH_CUDA_ARCH_LIST`가 빌드 시점에 set 됐는지**: `install.sh` 안에서 `sudo`나 새 shell이 실행되면 환경변수 전달이 안 될 수 있음. 스크립트 첫줄에 `export TORCH_CUDA_ARCH_LIST` 넣거나, `env TORCH_CUDA_ARCH_LIST="..." ./install.sh`처럼 명시.
4. **Python/PyTorch 환경이 같은지**: `which python`, `python -c "import torch; print(torch.__file__)"`로 재빌드 때와 실행 때가 같은 환경인지 확인.
5. **CUDA 버전 불일치**: PyTorch가 CUDA 11.8로 빌드됐는데 nvcc는 CUDA 12.4면 컴파일은 되지만 런타임 mismatch. `python -c "import torch; print(torch.version.cuda)"` vs `nvcc --version`.
6. **드라이버 compute cap 지원**: 아주 오래된 드라이버는 새 arch (예: sm_90) 못 로드. `nvidia-smi`로 driver version 확인, H100이면 525+ 필요.

## 관련 도구 / 실전 사례

SHARP, Forward-Warp, gsplat, diff-gaussian-rasterization, tiny-cuda-nn, tcnn, pointops, pytorch3d CUDA ext, MMCV CUDA ops 등 **모든** 커스텀 CUDA extension에 동일하게 적용됩니다.

- SHARP (`sharp-stereo-sbs`): gsplat 커스텀 빌드 시 `TORCH_CUDA_ARCH_LIST="7.0;8.0;9.0"` 필요 — SHARP 프로젝트 노트 참조
- Forward-Warp (`stereocrafter`, `stereocrafter-3d-photo`): 동일 패턴
- MoVieS: DINOv2 + custom ops 사용 시 동일
- **xformers**: precompiled wheel이 H100 sm_90 미포함일 때 별도 이슈. 자세한 건 [[cuda/xformers-sm-mismatch]] 참고

## 한 줄 요약

```bash
# 커스텀 CUDA extension 설치 전 반드시
export TORCH_CUDA_ARCH_LIST="7.0;8.0;9.0"
```

→ 시스템 CUDA 버전 관리 자체는 [[cuda/버전-관리]]에 별도.
