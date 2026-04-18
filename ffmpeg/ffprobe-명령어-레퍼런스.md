# ffprobe 명령어 레퍼런스

`ffprobe`는 영상/음성 파일의 메타데이터를 확인하는 도구.
딥러닝 전처리에서 해상도, 프레임 수, FPS 등을 확인할 때 필수.

---

## 1. 기본 사용법

```bash
# 전체 정보 (가장 상세)
ffprobe input.mp4

# 에러/경고 숨기고 깔끔하게
ffprobe -v error input.mp4

# 스트림 정보만
ffprobe -v error -show_streams input.mp4

# 포맷(컨테이너) 정보만
ffprobe -v error -show_format input.mp4

# 스트림 + 포맷 둘 다
ffprobe -v error -show_format -show_streams input.mp4
```

---

## 2. 영상 해상도 (가로 × 세로)

```bash
# 기본
ffprobe -v error -select_streams v:0 -show_entries stream=width,height input.mp4
# [STREAM]
# width=1920
# height=1080
# [/STREAM]

# 값만 깔끔하게 (쉼표 구분, 스크립트용)
ffprobe -v error -select_streams v:0 -show_entries stream=width,height \
    -of csv=p=0 input.mp4
# 출력: 1920,1080

# 줄 단위 출력
ffprobe -v error -select_streams v:0 -show_entries stream=width,height \
    -of default=noprint_wrappers=1:nokey=1 input.mp4
# 1920
# 1080
```

---

## 3. 프레임 수

```bash
# 컨테이너에 기록된 프레임 수
ffprobe -v error -select_streams v:0 -show_entries stream=nb_frames input.mp4

# 값만
ffprobe -v error -select_streams v:0 -show_entries stream=nb_frames \
    -of default=noprint_wrappers=1:nokey=1 input.mp4

# nb_frames가 N/A일 때 (일부 코덱/컨테이너)
# → 직접 세기 (느리지만 정확)
ffprobe -v error -count_frames -select_streams v:0 \
    -show_entries stream=nb_read_frames input.mp4
```

---

## 4. FPS (프레임율)

```bash
# r_frame_rate (기본 프레임율)
ffprobe -v error -select_streams v:0 -show_entries stream=r_frame_rate input.mp4
# 30/1 (= 30fps), 24000/1001 (= 23.976fps)

# avg_frame_rate (평균 프레임율, VFR 영상에서 유용)
ffprobe -v error -select_streams v:0 -show_entries stream=avg_frame_rate input.mp4

# 값만
ffprobe -v error -select_streams v:0 -show_entries stream=r_frame_rate \
    -of default=noprint_wrappers=1:nokey=1 input.mp4

# 소수점으로 변환 (bash에서 분수→실수)
FPS=$(ffprobe -v error -select_streams v:0 -show_entries stream=r_frame_rate \
    -of default=noprint_wrappers=1:nokey=1 input.mp4 | bc -l)
echo $FPS
```

### r_frame_rate vs avg_frame_rate

| 항목 | 의미 | 언제 쓰나 |
|------|------|----------|
| `r_frame_rate` | 코덱 기준 프레임율 | CFR (고정 프레임율) 영상 |
| `avg_frame_rate` | 실제 평균 프레임율 | VFR (가변 프레임율) 영상 |

---

## 5. 영상 길이 (Duration)

```bash
# 전체 길이 (초)
ffprobe -v error -show_entries format=duration \
    -of default=noprint_wrappers=1:nokey=1 input.mp4
# 12.500000

# 시:분:초 형식
ffprobe -v error -show_entries format=duration \
    -of default=noprint_wrappers=1 -sexagesimal input.mp4
# duration=0:00:12.500000

# 스트림 기준 길이
ffprobe -v error -select_streams v:0 -show_entries stream=duration \
    -of default=noprint_wrappers=1:nokey=1 input.mp4
```

---

## 6. 코덱 정보

```bash
# 비디오 코덱
ffprobe -v error -select_streams v:0 -show_entries stream=codec_name input.mp4
# h264, hevc, vp9, av1 등

# 코덱 + 프로파일 + 레벨
ffprobe -v error -select_streams v:0 \
    -show_entries stream=codec_name,profile,level input.mp4

# 오디오 코덱
ffprobe -v error -select_streams a:0 -show_entries stream=codec_name input.mp4
# aac, mp3, opus 등

# 픽셀 포맷 (색공간)
ffprobe -v error -select_streams v:0 -show_entries stream=pix_fmt input.mp4
# yuv420p, yuv444p, rgb24 등
```

---

## 7. 비트레이트

```bash
# 전체 비트레이트
ffprobe -v error -show_entries format=bit_rate \
    -of default=noprint_wrappers=1:nokey=1 input.mp4
# 5000000 (= 5Mbps)

# 비디오 스트림 비트레이트
ffprobe -v error -select_streams v:0 -show_entries stream=bit_rate \
    -of default=noprint_wrappers=1:nokey=1 input.mp4

# 오디오 비트레이트
ffprobe -v error -select_streams a:0 -show_entries stream=bit_rate \
    -of default=noprint_wrappers=1:nokey=1 input.mp4
```

---

## 8. 파일 크기

```bash
# 파일 크기 (바이트)
ffprobe -v error -show_entries format=size \
    -of default=noprint_wrappers=1:nokey=1 input.mp4

# 사람이 읽기 쉽게
ls -lh input.mp4
```

---

## 9. 오디오 정보

```bash
# 샘플레이트
ffprobe -v error -select_streams a:0 -show_entries stream=sample_rate input.mp4
# 44100, 48000 등

# 채널 수 (모노=1, 스테레오=2, 5.1=6)
ffprobe -v error -select_streams a:0 -show_entries stream=channels input.mp4

# 오디오 전체 정보
ffprobe -v error -select_streams a:0 \
    -show_entries stream=codec_name,sample_rate,channels,bit_rate input.mp4
```

---

## 10. 스트림 목록 확인

```bash
# 몇 개의 스트림이 있는지 (비디오, 오디오, 자막 등)
ffprobe -v error -show_entries stream=index,codec_type,codec_name input.mp4
# [STREAM]
# index=0
# codec_type=video
# codec_name=h264
# [/STREAM]
# [STREAM]
# index=1
# codec_type=audio
# codec_name=aac
# [/STREAM]
```

---

## 11. 한 번에 여러 정보 확인

```bash
# 해상도 + 프레임 수 + FPS + 코덱 + 픽셀포맷 + 길이
ffprobe -v error -select_streams v:0 \
    -show_entries stream=width,height,nb_frames,r_frame_rate,codec_name,duration,pix_fmt \
    input.mp4

# csv 출력 (한 줄로)
ffprobe -v error -select_streams v:0 \
    -show_entries stream=width,height,nb_frames,r_frame_rate,codec_name \
    -of csv=p=0 input.mp4
# h264,1920,1080,30/1,750

# JSON 출력 (파이썬 파싱용)
ffprobe -v error -select_streams v:0 \
    -show_entries stream=width,height,nb_frames,r_frame_rate,codec_name \
    -of json input.mp4
```

---

## 12. 출력 포맷 옵션 (-of)

| 옵션 | 출력 형태 | 용도 |
|------|----------|------|
| `default` | 태그=값 형태 | 사람이 읽기 |
| `csv=p=0` | 쉼표 구분, 헤더 없음 | 스크립트 파싱 |
| `json` | JSON | Python 파싱 |
| `flat` | streams.stream.0.key=value | grep 용이 |
| `ini` | INI 형식 | 설정 파일 스타일 |

```bash
# flat 예시
ffprobe -v error -select_streams v:0 -show_entries stream=width,height \
    -of flat input.mp4
# streams.stream.0.width=1920
# streams.stream.0.height=1080
```

---

## 13. 딥러닝 프로젝트 체크 스크립트

여러 영상의 정보를 한 번에 비교:

```bash
#!/bin/bash
# video_info.sh - 영상 파일 정보 요약
# 사용법: bash video_info.sh file1.mp4 file2.mp4 ...

printf "%-40s %10s %8s %10s %8s %10s\n" "파일" "해상도" "프레임" "FPS" "코덱" "길이(초)"
printf "%-40s %10s %8s %10s %8s %10s\n" "----" "------" "-----" "---" "----" "-------"

for f in "$@"; do
    W=$(ffprobe -v error -select_streams v:0 -show_entries stream=width -of default=noprint_wrappers=1:nokey=1 "$f")
    H=$(ffprobe -v error -select_streams v:0 -show_entries stream=height -of default=noprint_wrappers=1:nokey=1 "$f")
    NB=$(ffprobe -v error -select_streams v:0 -show_entries stream=nb_frames -of default=noprint_wrappers=1:nokey=1 "$f")
    FPS=$(ffprobe -v error -select_streams v:0 -show_entries stream=r_frame_rate -of default=noprint_wrappers=1:nokey=1 "$f")
    CODEC=$(ffprobe -v error -select_streams v:0 -show_entries stream=codec_name -of default=noprint_wrappers=1:nokey=1 "$f")
    DUR=$(ffprobe -v error -show_entries format=duration -of default=noprint_wrappers=1:nokey=1 "$f")
    printf "%-40s %5sx%-4s %8s %10s %8s %10s\n" "$(basename $f)" "$W" "$H" "$NB" "$FPS" "$CODEC" "$DUR"
done
```

사용:
```bash
chmod +x video_info.sh
bash video_info.sh demo/*.mp4 outputs/reprojected/*.mp4
```

출력 예시:
```
파일                                         해상도    프레임        FPS     코덱     길이(초)
----                                         ------   -----        ---     ----     -------
input.mp4                                  1024x 576       90      30/1    h264   3.000000
input_25f.mp4                              1024x 576       25      30/1    h264   0.833333
input_reprojected.mp4                      1024x 576       25      30/1    h264   0.833333
```

---

## 14. Python에서 ffprobe 사용

```python
import subprocess
import json

def get_video_info(path):
    """ffprobe로 영상 정보를 dict로 반환"""
    cmd = [
        'ffprobe', '-v', 'error',
        '-select_streams', 'v:0',
        '-show_entries', 'stream=width,height,nb_frames,r_frame_rate,codec_name,pix_fmt',
        '-show_entries', 'format=duration,size',
        '-of', 'json',
        path
    ]
    result = subprocess.run(cmd, capture_output=True, text=True)
    return json.loads(result.stdout)

info = get_video_info('input.mp4')
stream = info['streams'][0]
print(f"해상도: {stream['width']}x{stream['height']}")
print(f"프레임 수: {stream['nb_frames']}")
print(f"FPS: {stream['r_frame_rate']}")
print(f"코덱: {stream['codec_name']}")
print(f"길이: {info['format']['duration']}초")
```

---

## 15. 자주 쓰는 옵션 요약

| 옵션 | 의미 |
|------|------|
| `-v error` | 에러만 표시 (경고/정보 숨김) |
| `-select_streams v:0` | 첫 번째 비디오 스트림 선택 |
| `-select_streams a:0` | 첫 번째 오디오 스트림 선택 |
| `-show_entries stream=...` | 특정 스트림 필드만 출력 |
| `-show_entries format=...` | 특정 포맷 필드만 출력 |
| `-of csv=p=0` | CSV 출력 (헤더 없음) |
| `-of json` | JSON 출력 |
| `-of default=noprint_wrappers=1:nokey=1` | 값만 출력 |
| `-count_frames` | 프레임 직접 세기 (느림) |
| `-sexagesimal` | 시:분:초 형식 |
