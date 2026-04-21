# 리다이렉트 & Heredoc

명령어의 입출력을 파일로 연결하거나, 여러 줄 텍스트를 명령에 넘기는 기본 문법. 환경 설정 파일 수정 (`.bashrc`, `.venv/bin/activate`, `$CONDA_PREFIX/etc/conda/activate.d/*.sh` 등)에서 자주 쓰임.

---

## 1. 리다이렉트 연산자 (`>`, `>>`, `<`)

명령어의 **입력(stdin)**과 **출력(stdout/stderr)**을 파일에 연결하는 문법.

### stdout 리다이렉트 — 파일로 출력

| 연산자 | 의미 | 예시 |
|:---:|---|---|
| `>` | 파일을 **덮어씀** (기존 내용 삭제) | `ls > files.txt` |
| `>>` | 파일에 **덧붙임** (append, 기존 내용 유지) | `echo "new" >> log.txt` |

```bash
# 현재 디렉토리 목록을 files.txt에 저장 (기존 내용 덮어쓰기)
ls > files.txt

# 기존 log.txt 끝에 한 줄 추가
echo "2026-04-21 작업 완료" >> log.txt

# 여러 명령 결과를 차례로 누적
date >> session.log
nvidia-smi >> session.log
```

**주의**: `>`는 파일을 완전히 날립니다. 기존 내용 지키고 싶으면 반드시 `>>` 사용.

### stderr 리다이렉트

| 연산자 | 의미 |
|:---:|---|
| `2>` | **에러 메시지**를 파일로 덮어쓰기 |
| `2>>` | 에러 메시지 append |
| `2>&1` | stderr를 stdout과 **합침** |
| `&>` | stdout과 stderr를 **모두** 같은 파일로 (bash 전용 단축) |

```bash
# 에러만 따로 파일에 저장
python train.py 2> errors.log

# 출력과 에러를 다 한 파일에 기록
python train.py &> full_log.txt
python train.py > full_log.txt 2>&1     # 동일 효과, POSIX 표준

# 에러를 버리고 출력만 보기
command 2>/dev/null
```

### stdin 리다이렉트 — 파일을 입력으로

| 연산자 | 의미 |
|:---:|---|
| `<` | 파일을 stdin으로 |

```bash
# 파일 내용을 sort에 입력
sort < names.txt

# SQL 파일을 psql에 입력
psql mydb < schema.sql
```

보통 `<`는 드물게 쓰고, 대부분 파이프 `|` 로 대체됨.

### 파이프 `|` — 명령 결과를 다른 명령에

```bash
# ls 결과를 grep에 넘김
ls -la | grep ".py"

# 파일 내용 중 특정 패턴만 보기
cat log.txt | grep ERROR

# 자주 쓰는 조합
du -sh * | sort -h          # 폴더별 크기 정렬
ps aux | grep python        # 실행 중인 python 프로세스 찾기
nvidia-smi | head -20       # 출력 앞부분만
```

---

## 2. Heredoc (`<<`) — 여러 줄 텍스트 입력

명령어에 **여러 줄 텍스트를 한번에 stdin으로 넘기는** 문법.

### 기본 형태

```bash
cat <<EOF
여러
줄의
내용
EOF
```

- `<<EOF` 로 시작
- 종료 마커 (`EOF`)가 등장하는 줄까지를 stdin으로 전달
- 마커는 **임의 이름** 가능 (`END`, `DONE`, `HERE` 등). 관례상 `EOF` 제일 많이 씀

### 리다이렉트와 함께 쓰는 패턴 (가장 흔함)

**파일에 여러 줄 append**:
```bash
cat >> .bashrc <<'EOF'
export HF_HUB_OFFLINE=1
export TRANSFORMERS_OFFLINE=1
export TORCH_CUDA_ARCH_LIST="7.0;8.0;9.0"
EOF
```

동작: `cat`의 stdout을 `.bashrc` 끝에 append + `<<'EOF'` ... `EOF` 사이 내용을 `cat`의 stdin으로.

**파일을 새로 만들기**:
```bash
cat > env.sh <<'EOF'
source .venv/bin/activate
export HF_HUB_OFFLINE=1
EOF
```

기존 파일 덮어씀. 스크립트로 설정 파일 자동 생성할 때 표준.

### ⭐ 따옴표 유무가 중요 — `'EOF'` vs `EOF`

```bash
# 따옴표 O : 안쪽을 literal 그대로
cat >> file <<'EOF'
$PATH 를 그대로 파일에 기록
EOF
#→ 파일 내용: $PATH 를 그대로 파일에 기록

# 따옴표 X : 변수 치환 발생
cat >> file <<EOF
$PATH 를 그대로 파일에 기록
EOF
#→ 파일 내용: /usr/bin:/bin:/home/user/... 를 그대로 파일에 기록
```

**환경설정 파일 (`$PATH`, `$HOME`, `${CONDA_PREFIX}` 등 기록)에는 반드시 `'EOF'` 작은따옴표 필수.**

안 하면 현재 쉘의 값이 치환되어 들어가서 완전 다른 의미가 됨.

### 들여쓰기: `<<-`

탭(tab) 들여쓰기를 허용하는 변형.

```bash
if [ condition ]; then
    cat >> file <<-'EOF'
    	내용이 탭으로 들여쓰기 됨
    	종료 마커도 탭 들여쓰기 가능
    	EOF
fi
```

`<<-`의 `-` 덕에 **탭만** 자동 제거됨 (space는 제거 안 됨, 주의).

### 변수에 담기 — `$(cat <<EOF ...)`

```bash
MY_TEXT=$(cat <<'EOF'
여러 줄
텍스트
EOF
)
echo "$MY_TEXT"
```

---

## 3. 실전 사례

### 사례 A: `.venv/bin/activate` 끝에 환경변수 추가

```bash
cat >> .venv/bin/activate <<'EOF'

# Huggingface offline
export HF_HUB_OFFLINE=1
export TRANSFORMERS_OFFLINE=1
EOF
```

→ venv 활성화 시 자동으로 export. 자세한 건 [[huggingface/캐시-관리]] 방법 B.

### 사례 B: conda activate.d 스크립트 생성

```bash
mkdir -p $CONDA_PREFIX/etc/conda/activate.d
cat > $CONDA_PREFIX/etc/conda/activate.d/cuda_arch.sh <<'EOF'
# Multi-arch CUDA build target
export TORCH_CUDA_ARCH_LIST="7.0;8.0;9.0"
EOF
```

→ conda env 활성화 시 자동 export. [[cuda/multi-arch-빌드]] 참고.

### 사례 C: 프로젝트 `env.sh` 생성

```bash
cat > env.sh <<'EOF'
source .venv/bin/activate
export HF_HUB_OFFLINE=1
export TRANSFORMERS_OFFLINE=1
export TORCH_CUDA_ARCH_LIST="7.0;8.0;9.0"
export XFORMERS_DISABLED=1
EOF
```

이후 `source env.sh` 한 줄로 모든 세팅 적용.

### 사례 D: SSH 키 생성 시 config 자동 작성

```bash
cat >> ~/.ssh/config <<'EOF'
Host supercomputer
    HostName gpu1.example.com
    User seungho
    IdentityFile ~/.ssh/id_ed25519
EOF
```

---

## 4. 실수하기 쉬운 지점

### 실수 1: `>` vs `>>` 혼동

```bash
# 의도: .bashrc에 한 줄 추가하려고
echo 'export FOO=1' > ~/.bashrc    # ❌ bashrc 전체를 덮어씀 → 원래 설정 다 날아감
echo 'export FOO=1' >> ~/.bashrc   # ✅ 끝에 덧붙임
```

**설정 파일 수정 시 무조건 `>>`** 를 떠올리는 게 안전합니다.

### 실수 2: 중복 append

```bash
# 1번 실행
cat >> ~/.bashrc <<'EOF'
export FOO=1
EOF

# 2번 실행 → 같은 export 두 번 들어감 (동작엔 영향 없지만 지저분)
cat >> ~/.bashrc <<'EOF'
export FOO=1
EOF
```

방지: 실행 전에 이미 있는지 확인.
```bash
grep -q "export FOO=1" ~/.bashrc || cat >> ~/.bashrc <<'EOF'
export FOO=1
EOF
```

### 실수 3: `'EOF'` 따옴표 안 써서 변수 치환

```bash
cat >> file <<EOF
export PATH=/new/path:$PATH     # $PATH가 현재 값으로 치환되어 박제됨!
EOF
```

기대와 다르게, 파일 안에는 `$PATH` 리터럴이 아니라 **실행 시점의 현재 PATH 값**이 통째로 박힘. 이후 PATH가 바뀌어도 반영 안 됨.

해결: `'EOF'` 쓰거나, `\$PATH`로 이스케이프.

### 실수 4: EOF 종료 마커 앞에 공백

```bash
cat >> file <<'EOF'
내용
    EOF      # ← 앞에 공백 있음 → 종료 안 됨, heredoc 계속됨
```

종료 마커는 **줄 맨 앞부터** 시작해야 함 (또는 `<<-` 쓸 때만 탭 허용).

### 실수 5: 큰따옴표 `"EOF"` 썼는데 기대와 다름

```bash
cat >> file <<"EOF"       # 큰따옴표도 literal 모드로 동작 (작은따옴표와 동일)
$PATH 그대로
EOF
```

Bash에서 `"EOF"`와 `'EOF'` 둘 다 literal 모드 (변수 치환 안 함). **따옴표 유무만 중요**하지 종류는 상관없음. 관례상 작은따옴표가 가장 많이 쓰임.

---

## 5. 검증 — 내가 쓴 게 파일에 제대로 들어갔는지

```bash
# 파일 끝 보기
tail -10 ~/.bashrc

# 특정 패턴 찾기
grep "HF_HUB_OFFLINE" ~/.bashrc

# 현재 세션에 변수 set 됐는지
source ~/.bashrc
echo "HF_HUB_OFFLINE=$HF_HUB_OFFLINE"
```

---

## 6. 요약 — 가장 자주 쓰는 패턴 3개

```bash
# (1) 한 줄 append
echo 'export FOO=bar' >> ~/.bashrc

# (2) 여러 줄 append (변수 치환 X, literal)
cat >> ~/.bashrc <<'EOF'
export FOO=bar
export BAZ=qux
EOF

# (3) 새 파일 통째로 작성 (덮어쓰기)
cat > env.sh <<'EOF'
source .venv/bin/activate
export FOO=bar
EOF
```

이 셋만 알면 **환경설정 스크립트 자동화는 99% 커버**됩니다.

---

→ 관련: [[bash/기본-명령어]] (cd, ls, mkdir 등), [[huggingface/캐시-관리]] (사례 A), [[cuda/multi-arch-빌드]] (사례 B)
