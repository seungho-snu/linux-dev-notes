# decord / OpenCV mp4v 호환성 이슈

딥러닝 비디오 파이프라인 (StereoCrafter, DepthCrafter, SVD 계열)에서 자주 만나는 에러와 해결 패턴.

## 핵심 에러

### 1) decord가 OpenCV mp4v 파일을 못 읽음

```
decord._ffi.base.DECORDError: ... Check failed: st_nb >= 0 (-1381258232 vs. 0)
ERROR cannot find video stream with wanted index: -1
```

### 2) 4K 이상 해상도에서 특히 자주 발생

1080p 입력이라도 2×2 grid 만들면 이미 4K 초과 (3840×2160) → OpenCV mp4v가 불안해짐.

## 원인

OpenCV는 기본적으로 비디오를 이렇게 씁니다:

```python
out = cv2.VideoWriter(
    path,
    cv2.VideoWriter_fourcc(*"mp4v"),   # MPEG-4 Part 2
    fps,
    (W, H),
)
```

`mp4v` fourcc(MPEG-4 Part 2)는 오래된 코덱이고 OpenCV 내장 encoder는 특정 조건에서 **헤더가 깨진 mp4 파일**을 만듭니다:

- 매우 큰 해상도 (>= 4K)
- 비정상적인 aspect ratio
- 짧은 길이
- pixel format 불일치

파일 자체는 "성공적으로 저장된 것처럼" 보이지만, `ffprobe`로 보면 `nb_frames=N/A`, `duration=N/A`, 또는 codec_tag가 이상하게 나옴. **decord의 `av_find_best_stream()`이 video stream을 찾지 못하고 `-1` 반환** → 위 에러.

참고: OpenCV의 `mp4v` encoder는 원래부터 약한 encoder로 유명. 프로덕션에선 ffmpeg 계열 사용이 원칙.

## 진단 — 3단계

### 1) 파일 자체가 정상인지 (ffprobe)

```bash
ffprobe -v error -select_streams v:0 \
    -show_entries stream=codec_name,codec_tag_string,width,height,nb_frames,duration \
    problem.mp4
```

정상 파일:
```
codec_name=h264
codec_tag_string=avc1
width=3840
height=2112
nb_frames=72
duration=3.000000
```

**깨진 mp4v 파일 예시**:
```
codec_name=mpeg4
codec_tag_string=mp4v
width=7680
height=4224
nb_frames=N/A         # ← 주의: 프레임 수 없음
duration=N/A          # ← 또는 duration 없음
```

`nb_frames=N/A`가 나오면 decord가 못 읽을 가능성 매우 높음.

### 2) decord로 직접 열어보기

```bash
python -c "
from decord import VideoReader, cpu
vr = VideoReader('problem.mp4', ctx=cpu(0))
print('OK', len(vr), vr[0].shape)
"
```

에러나면 파일이 decord-incompatible 확정.

### 3) OpenCV로는 읽히는지 (대조군)

```bash
python -c "
import cv2
cap = cv2.VideoCapture('problem.mp4')
print('frames:', int(cap.get(cv2.CAP_PROP_FRAME_COUNT)))
ret, f = cap.read()
print('first frame:', ret, f.shape if ret else None)
"
```

OpenCV로는 열리는데 decord만 실패 → **OpenCV가 쓴 파일을 OpenCV만 다시 읽을 수 있다**는 전형적인 mp4v 병리.

## 해결 — 3가지 전략

### A. ffmpeg로 재인코딩 (가장 확실)

```bash
ffmpeg -i input.mp4 -c:v libx264 -pix_fmt yuv420p -crf 16 -an output.mp4
```

- `libx264`: H.264, 가장 널리 지원
- `yuv420p`: 표준 pixel format (decord + 모든 player 호환)
- `-crf 16`: 시각적으로 lossless에 가까움 (0=무손실, 23=기본, 51=최악)
- `-an`: 오디오 제거 (우리 용도에선 불필요)

### B. 파이프라인 코드에서 OpenCV 대신 ffmpeg 사용

OpenCV VideoWriter 대신, ffmpeg subprocess로 직접 쓰거나, `imageio[ffmpeg]` / `pyav` 같은 라이브러리 사용.

```python
import subprocess
import numpy as np

def write_video_ffmpeg(path, frames, fps):
    """frames: (N, H, W, 3) uint8 RGB"""
    N, H, W, _ = frames.shape
    cmd = [
        "ffmpeg", "-y", "-loglevel", "error",
        "-f", "rawvideo", "-pix_fmt", "rgb24",
        "-s", f"{W}x{H}", "-r", str(fps),
        "-i", "-",
        "-c:v", "libx264", "-pix_fmt", "yuv420p", "-crf", "16",
        "-an",
        path,
    ]
    proc = subprocess.Popen(cmd, stdin=subprocess.PIPE)
    for f in frames:
        proc.stdin.write(f.tobytes())
    proc.stdin.close()
    proc.wait()
```

### C. 하이브리드: OpenCV로 일단 쓰고 ffmpeg 재인코딩

코드 수정을 최소화하면서 호환성 확보. 실전에서 가장 자주 쓰는 패턴:

```python
# OpenCV로 임시 파일 쓰기
tmp_path = output_path + ".tmp_mp4v.mp4"
out = cv2.VideoWriter(tmp_path, cv2.VideoWriter_fourcc(*"mp4v"), fps, (W, H))
for frame in frames:
    out.write(frame)
out.release()

# ffmpeg로 재인코딩
import subprocess, shutil, os
if shutil.which("ffmpeg"):
    subprocess.check_call([
        "ffmpeg", "-y", "-loglevel", "error",
        "-i", tmp_path,
        "-c:v", "libx264", "-pix_fmt", "yuv420p", "-crf", "16",
        "-an", output_path,
    ])
    os.remove(tmp_path)
else:
    os.replace(tmp_path, output_path)  # ffmpeg 없으면 mp4v 그대로
```

실제 `stereocrafter-3d-photo@5142f2c`에서 Stage A에 적용한 패치가 이 방식.

## 비상 우회 (코드 수정 불가 시)

이미 만들어진 깨진 파일을 재인코딩:

```bash
# 단일 파일
ffmpeg -i broken.mp4 -c:v libx264 -pix_fmt yuv420p -crf 16 -an fixed.mp4
mv fixed.mp4 broken.mp4

# 폴더 전체 (모든 mp4)
for f in ./outputs/*.mp4; do
    ffmpeg -y -i "$f" -c:v libx264 -pix_fmt yuv420p -crf 16 -an "${f%.mp4}.fixed.mp4"
    mv "${f%.mp4}.fixed.mp4" "$f"
done
```

## 왜 decord는 OpenCV보다 까다로운가

- **OpenCV VideoCapture**: 자체 여러 백엔드 (ffmpeg, GStreamer, MediaFoundation 등). 하나 실패하면 다른 것 시도. 깨진 header에도 관대.
- **decord**: ffmpeg 하나에만 의존. `avformat_open_input` + `avformat_find_stream_info` + `av_find_best_stream`을 엄격하게 호출. stream metadata가 완벽해야 통과.

즉 decord는 "표준 준수 검사"가 엄격해서, OpenCV가 대충 쓴 파일을 잘 걸러냅니다. 품질 측면에선 오히려 decord가 올바른 것.

## 관련 도구가 decord를 쓰는 경우

딥러닝 비디오 파이프라인 중 decord 의존:
- **StereoCrafter** `inpainting_inference.py`, `depth_splatting_inference.py`
- **DepthCrafter**
- **MMAction2**, 기타 video understanding 프레임워크
- **일부 SVD/video diffusion 구현**

이 도구들 연결할 때는 **모든 중간 mp4를 libx264 yuv420p로 통일**해두면 안전.

## 실전 사례

### Case: stereocrafter-3d-photo에서 4K 입력 시 Stage B 크래시

- 증상: 1080p/720p 입력은 정상, 4K에서만 "cannot find video stream"
- 원인: Stage A의 OpenCV mp4v writer가 7680×4224 grid에서 깨진 mp4 생성
- 해결: Stage A 끝에 ffmpeg libx264 yuv420p 재인코딩 subprocess 추가
- 커밋: `seungho-snu/stereocrafter-3d-photo@5142f2c`

### Case: StereoCrafter 원본이 왜 평소엔 돌아가나

원본 레포의 `inpainting_inference.py`는 자기가 읽는 파일(`depth_splatting_inference.py` 출력)을 자기가 씁니다. 둘 다 OpenCV mp4v. 해상도가 보통 1080p 이하라 운좋게 돌아가지만, **4K 지원이 공식적으로 된다고 주장하긴 어려움**. 우리 파이프라인에서 4K까지 돌리려면 이 fix 필수.

## 한 줄 요약

```
OpenCV mp4v로 쓴 mp4를 decord가 못 읽으면
-> ffmpeg로 libx264 yuv420p 재인코딩
```

---

→ 관련: [[ffmpeg/ffmpeg-명령어-레퍼런스]] (ffmpeg 기본 옵션),
  [[ffmpeg/ffprobe-명령어-레퍼런스]] (진단 명령)
