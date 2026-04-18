# FFmpeg 명령어 레퍼런스

영상/음성 처리에서 자주 쓰는 ffmpeg, ffprobe 명령어 모음.
딥러닝 프로젝트(M2SVid, DepthCrafter 등) 전처리/후처리에서 특히 유용한 명령 위주로 정리.

---

## 1. 영상 정보 확인 (ffprobe)

```bash
# 해상도, 프레임 수 확인
ffprobe -v error -select_streams v:0 -show_entries stream=width,height,nb_frames input.mp4

# FPS 확인
ffprobe -v error -select_streams v:0 -show_entries stream=r_frame_rate input.mp4

# 코덱, 비트레이트, 길이 등 전체 정보
ffprobe -v error -show_format -show_streams input.mp4

# 영상 길이(초) 확인
ffprobe -v error -show_entries format=duration -of default=noprint_wrappers=1:nokey=1 input.mp4

# 픽셀 포맷 확인
ffprobe -v error -select_streams v:0 -show_entries stream=pix_fmt input.mp4
```

---

## 2. 해상도 변환 (리사이즈)

```bash
# 고정 해상도로 변환
ffmpeg -i input.mp4 -vf "scale=1024:576" output.mp4

# 긴 변 기준으로 비율 유지 리사이즈 (-2는 짝수 자동 맞춤)
ffmpeg -i input.mp4 -vf "scale='if(gte(iw,ih),1024,-2)':'if(gte(iw,ih),-2,1024)'" output.mp4

# 긴 변 기준 + 64배수 패딩 (딥러닝 모델 입력용)
ffmpeg -i input.mp4 -vf "scale='if(gte(iw,ih),768,-2)':'if(gte(iw,ih),-2,768)',pad=ceil(iw/64)*64:ceil(ih/64)*64" output.mp4

# 4K 업스케일 (단순 보간, AI 업스케일 아님)
ffmpeg -i input.mp4 -vf "scale=3840:2160:flags=lanczos" output_4k.mp4

# 가로 절반 (SBS 3D용)
ffmpeg -i input.mp4 -vf "scale=iw/2:ih" output_half.mp4
```

### 스케일링 알고리즘

| 플래그 | 용도 |
|--------|------|
| `bilinear` | 빠름, 기본값 |
| `bicubic` | 일반적으로 충분 |
| `lanczos` | 품질 최고, 업스케일 시 권장 |
| `neighbor` | 최근접 보간, 픽셀아트용 |

---

## 3. 프레임 수 / 길이 자르기

```bash
# 앞에서 3초만
ffmpeg -i input.mp4 -t 3 output.mp4

# 10초 지점부터 5초간
ffmpeg -i input.mp4 -ss 10 -t 5 output.mp4

# 정확히 N프레임만 추출
ffmpeg -i input.mp4 -frames:v 25 output.mp4

# 해상도 변환 + 프레임 수 제한 동시에
ffmpeg -i input.mp4 -vf "scale=1024:576" -frames:v 25 output.mp4

# 특정 구간 자르기 (시작~끝)
ffmpeg -i input.mp4 -ss 00:01:00 -to 00:01:30 output.mp4
```

---

## 4. FPS 변환

```bash
# 30fps로 변환
ffmpeg -i input.mp4 -r 30 output.mp4

# 24fps로 변환 (영화용)
ffmpeg -i input.mp4 -r 24 output.mp4

# FPS 변경 + 해상도 변경
ffmpeg -i input.mp4 -vf "fps=24,scale=1024:576" output.mp4
```

---

## 5. 프레임 추출 / 프레임에서 영상 생성

```bash
# 전체 프레임을 이미지로 추출
ffmpeg -i input.mp4 frames/frame_%04d.png

# 1초에 1프레임만 추출
ffmpeg -i input.mp4 -vf "fps=1" frames/frame_%04d.png

# 특정 프레임 1장만 추출 (30번째 프레임)
ffmpeg -i input.mp4 -vf "select=eq(n\,29)" -frames:v 1 frame_30.png

# 이미지 시퀀스 → 영상
ffmpeg -framerate 24 -i frames/frame_%04d.png -c:v libx264 -pix_fmt yuv420p output.mp4

# 이미지 시퀀스 → 영상 (png 품질 유지)
ffmpeg -framerate 24 -i frames/frame_%04d.png -c:v libx264 -crf 0 output_lossless.mp4
```

---

## 6. 코덱 / 품질 설정

```bash
# H.264 (호환성 최고)
ffmpeg -i input.mp4 -c:v libx264 -crf 18 output.mp4

# H.265/HEVC (파일 크기 절반, 느림)
ffmpeg -i input.mp4 -c:v libx265 -crf 18 output.mp4

# NVENC GPU 인코딩 (NVIDIA GPU 필요, 빠름)
ffmpeg -i input.mp4 -c:v h264_nvenc -preset p4 -cq 20 output.mp4
ffmpeg -i input.mp4 -c:v hevc_nvenc -preset p4 -cq 20 output.mp4

# 무손실
ffmpeg -i input.mp4 -c:v libx264 -crf 0 output_lossless.mp4

# 음성 제거 (영상만)
ffmpeg -i input.mp4 -an output_nosound.mp4

# 영상 제거 (음성만)
ffmpeg -i input.mp4 -vn output_audio.aac
```

### CRF 값 가이드

| CRF | 품질 | 용도 |
|-----|------|------|
| 0 | 무손실 | 원본 보존 |
| 15-18 | 고품질 | 최종 출력, 편집 소스 |
| 20-23 | 중간 | 일반 배포 |
| 25-28 | 저품질 | 미리보기, 테스트 |

---

## 7. SBS (Side-by-Side) 3D 합성

```bash
# 좌안 + 우안 → SBS (각각 가로 절반으로 줄여서 합침)
ffmpeg -i left.mp4 -i right.mp4 \
    -filter_complex "[0:v]scale=1920:2160[l];[1:v]scale=1920:2160[r];[l][r]hstack=inputs=2" \
    -c:v libx265 -crf 18 output_sbs_4k.mp4

# Full SBS (가로 해상도 그대로 합침 → 7680×2160)
ffmpeg -i left.mp4 -i right.mp4 \
    -filter_complex "[0:v][1:v]hstack=inputs=2" \
    -c:v libx265 -crf 18 output_full_sbs.mp4

# 상하 합성 (Top-Bottom)
ffmpeg -i left.mp4 -i right.mp4 \
    -filter_complex "[0:v]scale=3840:1080[t];[1:v]scale=3840:1080[b];[t][b]vstack=inputs=2" \
    -c:v libx265 -crf 18 output_tb_4k.mp4

# SBS → 좌안/우안 분리
ffmpeg -i sbs.mp4 -filter_complex "[0:v]crop=iw/2:ih:0:0" left.mp4
ffmpeg -i sbs.mp4 -filter_complex "[0:v]crop=iw/2:ih:iw/2:0" right.mp4
```

---

## 8. 영상 합치기 / 이어붙이기

```bash
# 파일 리스트로 이어붙이기 (같은 코덱일 때, 무손실)
# filelist.txt:
#   file 'clip1.mp4'
#   file 'clip2.mp4'
#   file 'clip3.mp4'
ffmpeg -f concat -safe 0 -i filelist.txt -c copy output.mp4

# 재인코딩하면서 이어붙이기 (코덱 다를 때)
ffmpeg -f concat -safe 0 -i filelist.txt -c:v libx264 -crf 18 output.mp4
```

---

## 9. 크롭 (Crop)

```bash
# 중앙에서 1920×1080 크롭
ffmpeg -i input.mp4 -vf "crop=1920:1080" output.mp4

# 좌상단 기준 크롭 (x=100, y=50에서 1280×720)
ffmpeg -i input.mp4 -vf "crop=1280:720:100:50" output.mp4

# 16:9 비율로 크롭 (상하 레터박스 제거)
ffmpeg -i input.mp4 -vf "crop=iw:iw*9/16" output.mp4
```

---

## 10. 테스트용 더미 영상 생성

```bash
# 검정 배경 (3초, 1024×576, 24fps)
ffmpeg -f lavfi -i color=c=black:s=1024x576:d=3:r=24 -c:v libx264 test.mp4

# 컬러바 패턴
ffmpeg -f lavfi -i smptebars=s=1920x1080:d=5:r=30 -c:v libx264 test_bars.mp4

# 카운터 있는 테스트 영상 (프레임 번호 표시)
ffmpeg -f lavfi -i color=c=black:s=1024x576:d=3:r=24 \
    -vf "drawtext=text='%{frame_num}':x=(w-tw)/2:y=(h-th)/2:fontsize=72:fontcolor=white" \
    -c:v libx264 test_counter.mp4
```

---

## 11. 딥러닝 프로젝트 전처리 패턴

### 입력 영상 준비 (해상도 + 프레임 수 맞추기)

```bash
# M2SVid용: 1024×576, 25프레임
ffmpeg -i input.mp4 -vf "scale=1024:576" -frames:v 25 input_ready.mp4

# DepthCrafter용: 긴 변 768, 64배수 맞춤
ffmpeg -i input.mp4 -vf "scale='if(gte(iw,ih),768,-2)':'if(gte(iw,ih),-2,768)',pad=ceil(iw/64)*64:ceil(ih/64)*64" input_depth.mp4
```

### 윈도우 슬라이딩 (긴 영상을 25프레임씩 분할)

```bash
# 25프레임 단위로 분할 (오버랩 4프레임)
# clip_001: frame 0-24
# clip_002: frame 21-45
# clip_003: frame 42-66
# ...

FPS=$(ffprobe -v error -select_streams v:0 -show_entries stream=r_frame_rate -of default=noprint_wrappers=1:nokey=1 input.mp4 | bc -l | xargs printf "%.0f")
WINDOW=25
OVERLAP=4
STEP=$((WINDOW - OVERLAP))

for i in $(seq 0 $STEP 200); do
    START_TIME=$(echo "scale=3; $i / $FPS" | bc)
    IDX=$(printf "%03d" $((i / STEP + 1)))
    ffmpeg -i input.mp4 -ss $START_TIME -frames:v $WINDOW -c:v libx264 -crf 18 "clips/clip_${IDX}.mp4" -y
done
```

### 결과 합성 (처리된 클립 이어붙이기)

```bash
# 클립 목록 생성
ls clips/clip_*.mp4 | sort | sed "s/^/file '/" | sed "s/$/'/" > filelist.txt

# 이어붙이기
ffmpeg -f concat -safe 0 -i filelist.txt -c:v libx264 -crf 18 output_final.mp4
```

---

## 12. 유용한 필터 조합

```bash
# 리사이즈 + 크롭 + FPS 변환 한 번에
ffmpeg -i input.mp4 -vf "scale=1280:720,crop=1024:576,fps=24" output.mp4

# 밝기/대비 조정
ffmpeg -i input.mp4 -vf "eq=brightness=0.05:contrast=1.2" output.mp4

# 가로/세로 뒤집기
ffmpeg -i input.mp4 -vf "hflip" output_hflip.mp4
ffmpeg -i input.mp4 -vf "vflip" output_vflip.mp4

# 90도 회전
ffmpeg -i input.mp4 -vf "transpose=1" output_rot90.mp4

# 두 영상 나란히 비교 (원본 vs 결과)
ffmpeg -i original.mp4 -i processed.mp4 \
    -filter_complex "[0:v][1:v]hstack=inputs=2" \
    -c:v libx264 -crf 18 comparison.mp4

# GIF 변환 (짧은 클립용)
ffmpeg -i input.mp4 -vf "fps=10,scale=480:-1" -loop 0 output.gif
```

---

## 13. 자주 쓰는 옵션 정리

| 옵션 | 의미 |
|------|------|
| `-y` | 출력 파일 덮어쓰기 (확인 없이) |
| `-an` | 음성 제거 |
| `-vn` | 영상 제거 |
| `-c copy` | 재인코딩 없이 복사 (빠름) |
| `-c:v libx264` | H.264 인코딩 |
| `-c:v libx265` | H.265 인코딩 |
| `-crf 18` | 품질 설정 (낮을수록 고품질) |
| `-frames:v N` | N프레임만 처리 |
| `-t 5` | 5초만 처리 |
| `-ss 10` | 10초 지점부터 시작 |
| `-r 30` | 30fps로 변환 |
| `-pix_fmt yuv420p` | 호환성 높은 픽셀 포맷 |
| `-preset fast` | 인코딩 속도 (ultrafast~veryslow) |

---

## 14. 설치

```bash
# Ubuntu/Debian
sudo apt install ffmpeg

# Conda
conda install -c conda-forge ffmpeg

# 버전 확인
ffmpeg -version
ffprobe -version
```
