# Path 확장 & 글로빙

`~`, `./`, `../`, `*`, `**`, `?`, `[abc]`, `{a,b,c}` 등 경로/파일명 확장 문법. 매일 쓰지만 의외로 헷갈리는 부분들.

---

## 1. 특수 경로 기호

| 기호 | 의미 |
|:---:|---|
| `~` | 현재 사용자 홈 디렉토리 (`$HOME`) |
| `~user` | 다른 사용자의 홈 |
| `.` | 현재 디렉토리 |
| `..` | 상위 디렉토리 |
| `-` | **이전** 디렉토리 (cd 전용) |
| `/` | 루트 (절대 경로 시작) |

### `~` — 홈 디렉토리

```bash
cd ~                 # /home/seungho
cd ~/project          # /home/seungho/project
cd ~/.cache           # /home/seungho/.cache

echo ~                # 확인: /home/seungho

# 다른 사용자 홈 (권한 있으면)
ls ~otheruser/

# 이걸 실행 전에 미리 알고 싶으면 echo로
echo ~/Downloads/*.pt
# → 실제 확장된 결과 보여줌
```

**주의**: `~`는 **쉘이 확장**합니다. 큰따옴표 안에서는 확장되긴 하지만 주의:

```bash
echo "~"              # 문자 ~ 출력 (확장 안 됨)
echo ~                # /home/seungho (확장)
echo "$HOME"          # /home/seungho (환경변수 사용)
```

**스크립트에서는 `$HOME` 사용이 더 안전**:
```bash
DIR="$HOME/project"   # ✅
DIR="~/project"       # ❌ 따옴표 안에서는 ~ 확장 안 됨
```

### `.` 과 `..`

```bash
cd ./subdir           # 현재 폴더의 subdir (./ 생략 가능)
cd ..                 # 상위 폴더
cd ../..              # 두 단계 위
cd ../other_proj      # 상위로 갔다가 다른 폴더로
```

### `-` — 이전 디렉토리로 점프

```bash
cd /home/seungho/project
cd /tmp
cd -                  # /home/seungho/project 로 복귀
# 출력도 나옴: /home/seungho/project
```

두 폴더 사이 왔다갔다 할 때 유용. `cd -`는 토글처럼 동작.

---

## 2. 절대 경로 vs 상대 경로

### 절대 경로 (`/`로 시작)

```bash
/home/seungho/project/file.txt     # 루트부터 시작, 어디서든 같은 위치
```

- 스크립트에서 안정적 (실행 위치 무관)
- 서버 간 공유할 때 경로 의존

### 상대 경로 (`.`, `..`, 또는 이름으로 시작)

```bash
./data/image.jpg      # 현재 폴더 기준
../shared/model.pt    # 상위 폴더의 shared 기준
project/file.txt      # 현재 폴더 안의 project 폴더 기준 (./와 동일)
```

- 현재 작업 위치(`pwd`)에 따라 다르게 해석
- 프로젝트 내부에서 유연함

### 스크립트에서 현재 스크립트의 경로

```bash
# run.sh 파일 안에서...
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
echo "내 위치: $SCRIPT_DIR"
# 어디서 실행되든 run.sh 가 있는 폴더의 절대경로
```

관용구. 스크립트가 자기 폴더의 다른 파일을 참조할 때 씀:
```bash
source "$SCRIPT_DIR/env.sh"
python "$SCRIPT_DIR/main.py"
```

---

## 3. 글로빙 (`*`, `?`, `[]`) — 와일드카드

**글로빙 = 파일명 패턴 매칭**. 쉘이 명령 실행 전에 **미리 확장**해서 매칭되는 파일들로 대체.

### `*` — 0개 이상의 문자

```bash
ls *.mp4                 # 모든 mp4 파일
ls img_*.jpg             # img_ 로 시작하는 jpg
ls *_result.mp4          # _result.mp4 로 끝나는
ls IMG_*_edit.png        # 중간 와일드카드

# 주의: . 으로 시작하는 파일은 기본적으로 매칭 안 됨
ls *                     # .bashrc, .git 등 숨김파일 빠짐
ls .*                    # 숨김파일만
shopt -s dotglob         # 이 옵션 켜면 *도 숨김파일 포함
```

### `?` — 정확히 1개 문자

```bash
ls img?.jpg              # img1.jpg, imgA.jpg 는 매칭, img10.jpg 는 안 됨
ls log_????.txt          # log_0001.txt ~ log_ZZZZ.txt
```

### `[...]` — 문자 중 하나

```bash
ls img[0-9].jpg          # img0.jpg ~ img9.jpg
ls img[0-9][0-9].jpg     # img00.jpg ~ img99.jpg
ls [ABC]*.txt            # A, B, C 로 시작
ls [!abc]*               # a, b, c 로 시작 '안 하는' 것 (negation)
```

### `**` — 재귀 매칭 (`globstar` 옵션)

```bash
# 기본 bash에서는 **은 * 와 같음
# globstar 옵션 켜면 재귀
shopt -s globstar

ls ./src/**/*.py         # src 아래 모든 하위 폴더의 .py 파일
ls ~/**/config.yml       # 홈 아래 어디든 config.yml
```

**주의**: `**`는 시간 오래 걸릴 수 있음. 거대한 폴더에선 `find` 가 나음:
```bash
find ./src -name "*.py"          # 전통적, 항상 됨
```

### `{a,b,c}` — Brace expansion

글로빙 아님 (실제 파일 체크 안 함). 단순 **문자열 확장**.

```bash
# 여러 확장자 매칭
ls *.{jpg,jpeg,png,webp}
# = ls *.jpg *.jpeg *.png *.webp

# 연속 숫자 / 문자
echo {1..5}              # 1 2 3 4 5
echo {01..10}            # 01 02 ... 10
echo {a..e}              # a b c d e

# step
echo {0..100..10}        # 0 10 20 ... 100

# 조합 - 백업 파일명
cp file.txt{,.bak}       # = cp file.txt file.txt.bak

# 여러 값을 각각 생성
mkdir -p data/{train,val,test}/{images,labels}
# data/train/images, data/train/labels, data/val/images, ...
```

### 매칭 실패 시 동작

```bash
ls *.xyz
# 매칭 없으면 → "ls: cannot access '*.xyz': No such file or directory"
# 또는 패턴 그대로 "*.xyz" 출력

for f in *.xyz; do echo "$f"; done
# 매칭 없으면 → literal "*.xyz" 한 번 출력 (위험!)
```

방어 옵션:
```bash
shopt -s nullglob        # 매칭 없으면 빈 리스트
for f in *.xyz; do echo "$f"; done
# 매칭 없으면 loop 한 번도 안 돔
```

---

## 4. 변수 확장 — `$var`, `${var}`

### 기본

```bash
name="seungho"
echo $name               # seungho
echo ${name}             # seungho (동일)
echo "$name/project"     # seungho/project
```

### `${}` 중괄호가 필요한 경우

```bash
file="report"
echo $file_2026.txt      # ❌ $file_2026 을 변수로 해석 → 빈 문자열
echo ${file}_2026.txt    # ✅ report_2026.txt
```

변수 이름 뒤에 **영문/숫자/언더스코어가 이어지면** `${}` 필수.

### 큰따옴표 vs 작은따옴표

```bash
s="hello"

echo "$s world"          # hello world    ($ 확장)
echo '$s world'          # $s world       (리터럴)
echo "\$s world"         # $s world       ($ 이스케이프)
```

**규칙**:
- 변수 확장해야 → 큰따옴표
- 문자 그대로 → 작은따옴표
- 큰따옴표 안에서 `$` 자체 쓰려면 `\$`

### 경로에서 자주 쓰는 변수

```bash
$HOME        # /home/seungho
$USER        # seungho
$PWD         # 현재 디렉토리
$OLDPWD      # 이전 디렉토리 ($ cd - 로 가는 곳)
$PATH        # 명령어 검색 경로
$SHELL       # /bin/bash
$HOSTNAME    # 서버 이름

echo "집은 $HOME 사용자는 $USER 위치는 $PWD"
```

---

## 5. 명령 치환 — `$(...)` 과 백틱

명령의 출력을 문자열로 변환.

```bash
today=$(date +%Y-%m-%d)
echo $today              # 2026-04-21

# 명령 결과를 다른 명령에
cd $(dirname $(which python))   # python 실행파일이 있는 폴더로
ls $(pwd)/*.mp4                  # 절대경로로 ls

# nesting
backup="backup_$(date +%Y%m%d)_$(hostname).tar.gz"
```

### 구 문법 백틱 vs 권장 `$()`

```bash
x=`date`          # ❌ 옛날 문법. nesting 불가, 가독성 낮음
x=$(date)         # ✅ 권장. nesting 가능

# nesting 예
result=$(echo $(date)-backup)   # $()만 됨
```

백틱은 레거시 스크립트에서만 보임. 새 스크립트에는 `$()`.

---

## 6. `readlink` / `realpath` — 실제 경로 해석

심볼릭 링크 따라가서 진짜 경로 구하기.

```bash
readlink /usr/local/cuda           # 심볼릭 링크의 타겟만
# → /usr/local/cuda-12.4

realpath /usr/local/cuda           # 절대 경로로 완전 해석 (recursive)
# → /usr/local/cuda-12.4

realpath ./my_file.txt             # 상대 → 절대
# → /home/seungho/project/my_file.txt

realpath ../../config              # 여러 단계 상위도
# → /home/seungho/config

# readlink -f 는 realpath와 유사
readlink -f ./path                  # GNU 환경에서 realpath 대체
```

**활용**:
```bash
# 심볼릭 링크인지 확인
if [ -L "/usr/local/cuda" ]; then
    echo "link -> $(readlink /usr/local/cuda)"
fi

# 스크립트 자기 위치
SCRIPT_DIR=$(realpath $(dirname $0))
```

---

## 7. `dirname` / `basename` — 경로 분리

```bash
p="/home/seungho/project/file.txt"

dirname "$p"          # /home/seungho/project
basename "$p"         # file.txt
basename "$p" .txt    # file   (확장자 제거)
```

파라미터 확장 대안:
```bash
dir="${p%/*}"         # /home/seungho/project (dirname 과 같음)
file="${p##*/}"       # file.txt (basename 과 같음)
stem="${file%.*}"     # file
ext="${file##*.}"     # txt
```

**파라미터 확장이 외부 명령보다 빠름** (fork 없음). 스크립트 안에선 `${}` 권장.

---

## 8. 실전 패턴

### 패턴 A: 스크립트가 자기 위치 기준으로 작동

```bash
#!/bin/bash
# run.sh
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

# 스크립트 있는 폴더의 env.sh source
source "$SCRIPT_DIR/env.sh"

# 스크립트 있는 폴더의 python 실행
python "$SCRIPT_DIR/main.py" --config "$SCRIPT_DIR/config.yml"
```

### 패턴 B: 여러 확장자 한 번에

```bash
# 이미지 폴더의 모든 지원 포맷
for img in ./images/*.{jpg,jpeg,png,webp,bmp}; do
    [ -f "$img" ] || continue       # 매칭 실패한 패턴 스킵
    process "$img"
done

# 또는 nullglob
shopt -s nullglob
for img in ./images/*.{jpg,jpeg,png,webp,bmp}; do
    process "$img"
done
shopt -u nullglob    # 원상복구
```

### 패턴 C: 날짜 기반 파일명

```bash
now=$(date +%Y%m%d_%H%M%S)
output="./results/output_${now}.mp4"

# 매일 다른 로그
log="./logs/$(date +%Y-%m-%d).log"
echo "[$(date +%H:%M:%S)] start" >> "$log"
```

### 패턴 D: 백업 파일 만들기

```bash
# 단일 파일 백업
cp file.txt{,.bak}        # file.txt → file.txt.bak

# 타임스탬프 백업
cp file.txt "file_$(date +%Y%m%d).txt"

# 여러 파일 백업 (conf → conf.bak)
for f in *.conf; do cp "$f" "$f.bak"; done
```

### 패턴 E: 중첩 경로 한번에 만들기

```bash
mkdir -p project/{src,tests,docs}/{__pycache__,output}
# project/src/__pycache__, project/src/output,
# project/tests/__pycache__, project/tests/output, ...

mkdir -p outputs/{h100,a100,v100}/{fp16,fp32}
# outputs/h100/fp16, outputs/h100/fp32, outputs/a100/fp16, ...
```

### 패턴 F: 확장자만 바꾸기 (배치)

```bash
for f in *.JPG; do
    mv "$f" "${f%.JPG}.jpg"
done

# 한 줄 (rename 명령 있으면)
rename 's/\.JPG$/.jpg/' *.JPG       # Perl-style (debian/ubuntu)
```

### 패턴 G: 최근 수정된 파일만

```bash
ls -t                     # 수정 시간 순 (최신 위)
ls -t | head -5           # 최근 5개
ls -1t *.mp4 | head -5    # 최근 5개 mp4 파일명만

# find + newer
find . -newer reference.txt      # reference.txt보다 새로 수정된 것
find . -mtime -1                 # 24시간 이내 수정된 것
find . -mmin -60                 # 60분 이내
```

---

## 9. 실수하기 쉬운 지점

### 실수 1: `~` 따옴표 안에서 확장 안 됨

```bash
DIR="~/project"          # ❌ DIR = 리터럴 ~/project
ls $DIR                  # ❌ 파일 못 찾음

DIR="$HOME/project"      # ✅
```

`$HOME`을 쓰거나, 따옴표 밖에 두기.

### 실수 2: 글로빙 패턴을 큰따옴표로 감쌈

```bash
rm "*.tmp"               # ❌ 리터럴 "*.tmp" 파일 하나를 찾음
rm *.tmp                 # ✅ 와일드카드 확장
```

**와일드카드는 쉘이 확장하므로, 확장 원하면 따옴표 밖에** 두기.

단 변수 안에 패턴이 있으면 주의:
```bash
pat="*.mp4"
ls "$pat"                # "*.mp4" 리터럴 파일 찾음 (보통 실패)
ls $pat                  # 확장됨 (단어 분리 위험)
```

### 실수 3: 경로에 공백 있는데 따옴표 안 씀

```bash
DIR=/home/seungho/My\ Folder        # 스페이스 이스케이프
cd $DIR                              # ❌ $DIR → "/home/seungho/My Folder" 단어 분리
cd "$DIR"                            # ✅
```

**변수 확장할 때 항상 `"$var"`**. 습관적으로.

### 실수 4: `cd` 명령 실패를 무시

```bash
cd ./outputs              # 폴더 없어도 계속 진행
rm *                      # 원래 폴더에서 삭제됨 (위험!)

# 방어
cd ./outputs || { echo "no outputs"; exit 1; }
rm *
```

### 실수 5: 상대경로로 절대경로 필요한 위치에 사용

```bash
# conda activate.d 스크립트 안
export PATH="./bin:$PATH"     # ❌ activate 시점의 pwd에 따라 의미 달라짐
export PATH="$CONDA_PREFIX/bin:$PATH"    # ✅
```

시스템 설정/환경 파일은 **절대경로가 원칙**.

### 실수 6: `$HOME` 과 `~/` 혼용

```bash
echo $HOME            # /home/seungho
echo ~/file           # /home/seungho/file

# 문제: 스크립트를 sudo로 실행하면 $HOME이 root로 바뀜
sudo bash my_script.sh
# my_script.sh 안에서 $HOME = /root (sudo user의 홈)
```

**사용자 지정이 필요하면 `/home/username`으로 명시**하거나, `getent passwd username` 으로 조회.

### 실수 7: `basename`으로 잘못 확장자 자르기

```bash
file="archive.tar.gz"
basename "$file" .gz        # archive.tar  (마지막 .gz만)
basename "$file" .tar.gz    # archive      (지정 가능)

# 더 편한 방법 (파라미터 확장)
echo "${file%%.*}"          # archive (첫 .부터 전부)
echo "${file%.*}"           # archive.tar (마지막 .만)
```

---

## 10. 한 줄 요약 치트시트

```bash
# 홈/현재/상위
cd ~                  # 홈으로
cd -                  # 이전 위치
cd ../..              # 두 단계 위

# 경로 분리
dirname "$path"       # 폴더
basename "$path"      # 파일명
"${path%/*}"          # 폴더 (faster)
"${path##*/}"         # 파일명 (faster)
"${name%.*}"          # 확장자 제거
"${name##*.}"         # 확장자만

# 글로빙
*.mp4                 # 현재 폴더 mp4
**/*.py               # 재귀 (shopt -s globstar 후)
img[0-9].jpg          # img0 ~ img9
*.{jpg,png}           # 여러 확장자

# Brace expansion
{1..10}               # 숫자 범위
{a,b,c}               # 나열
file.txt{,.bak}       # = file.txt file.txt.bak

# 절대경로
$(pwd)                # 현재 절대경로
$(realpath path)      # 상대 → 절대
"$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"   # 스크립트 폴더

# 환경변수
$HOME $USER $PWD $PATH
"$var"                # 공백 보존
```

---

→ 관련: [[bash/기본-명령어]], [[bash/문자열-처리]], [[bash/조건-반복]], [[bash/리다이렉트-heredoc]]
