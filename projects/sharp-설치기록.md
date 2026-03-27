# Apple SHARP (ml-sharp) 설치 & 실행 기록

> 날짜: 2026-03-27  
> 환경: WSL2 Ubuntu, RTX 3090 + TITAN Xp, CUDA 12.4

## SHARP란?

단일 사진 → 3D Gaussian Splatting 표현을 1초 이내에 생성하는 Apple의 모델.  
GitHub: https://github.com/apple/ml-sharp

## 설치

```bash
cd ~/project
git clone https://github.com/apple/ml-sharp.git
cd ml-sharp

conda create -n sharp python=3.13
conda activate sharp
pip install -r requirements.txt

# 설치 확인
sharp --help
```

## 환경 설정 (필수)

CUDA 12.4 + RTX 3090만 사용하도록 conda 환경별 설정 적용:  
→ [conda/환경별-설정.md](../conda/환경별-설정.md) 참고

```bash
# 요약
export PATH=/usr/local/cuda-12.4/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/cuda-12.4/lib64:$LD_LIBRARY_PATH
export TORCH_CUDA_ARCH_LIST="8.6"
export CUDA_VISIBLE_DEVICES=0
```

## 실행

```bash
# 1. 이미지 → 3DGS (.ply) 생성
sharp predict -i ./images/0156.png -o ./output/gaussians
# 모델 체크포인트는 최초 실행 시 자동 다운로드 (~2.6GB)
# 저장 위치: ~/.cache/torch/hub/checkpoints/sharp_2572gikvuh.pt

# 2. 3DGS → 비디오 렌더링 (CUDA GPU 필요)
sharp render -i ./output/gaussians -o ./output/renderings
```

## 만난 에러 & 해결

### 에러 1: `cooperative_groups::labeled_partition`

```
error: namespace "cooperative_groups" has no member "labeled_partition"
```

- **원인**: 시스템 기본 CUDA가 11.8이라 gsplat JIT 컴파일이 CUDA 11.8로 빌드됨. `labeled_partition`은 CUDA 12+ 전용 API.
- **해결**: CUDA 12.4 경로를 우선하도록 `PATH`, `LD_LIBRARY_PATH` 설정

### 에러 2: `NVIDIA TITAN Xp sm_61 is not compatible`

```
NVIDIA TITAN Xp with CUDA capability sm_61 is not compatible with the current PyTorch installation.
```

- **원인**: 현재 PyTorch가 sm_70 이상만 지원하는데, TITAN Xp(sm_61)가 보임
- **해결**: `CUDA_VISIBLE_DEVICES=0`으로 RTX 3090만 사용 + `TORCH_CUDA_ARCH_LIST="8.6"`

### 에러 3: JIT 빌드 캐시 문제

한번 실패한 빌드의 중간 파일이 남아서 계속 실패할 수 있음:

```bash
rm -rf ~/.cache/torch_extensions/
```

## 참고사항

- predict (추론)은 CPU/CUDA/MPS 모두 가능하지만, render는 CUDA GPU 필수
- `.ply` 출력은 OpenCV 좌표 (x→오른쪽, y→아래, z→앞)
- 외부 3DGS 뷰어 사용 시 scene center 재조정 필요
