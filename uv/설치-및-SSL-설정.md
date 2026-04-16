# uv 설치 및 SSL 인증서 설정

`uv`는 Astral에서 만든 Python 패키지 매니저로, pip/venv를 대체하며 속도가 빠르다.
DepthCrafter 등 일부 프로젝트에서 `uv venv`, `uv sync`를 사용한다.

---

## 1. 설치 방법

### Linux

```bash
# 방법 1: 공식 스크립트 (권장)
curl -LsSf https://astral.sh/uv/install.sh | sh
source $HOME/.local/bin/env

# 방법 2: pip
pip install --user uv
export PATH="$HOME/.local/bin:$PATH"

# 방법 3: snap (Ubuntu)
sudo snap install astral-uv --classic
# --classic 필수: uv가 시스템 파일에 접근해야 하므로
```

> PATH 영구 등록:
> ```bash
> echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
> source ~/.bashrc
> ```

### Windows

```powershell
# 방법 1: PowerShell 공식 스크립트
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"

# 방법 2: winget
winget install --id=astral-sh.uv -e

# 방법 3: pip / pipx
pip install uv
```

> ⚠️ WSL2 내부는 리눅스이므로 리눅스용 설치를 별도로 해야 함.
> 윈도우에 설치한 uv는 WSL에서 보이지 않음.

### 설치 확인

```bash
uv --version
```

---

## 2. 기본 사용법

```bash
# 가상환경 생성
uv venv

# 활성화
source .venv/bin/activate

# pyproject.toml 기반 의존성 설치 (= pip install .)
uv sync

# 패키지 목록 확인
uv pip list

# 개별 패키지 설치
uv pip install <패키지명>
```

---

## 3. 흔한 에러 및 해결

### 3.1 `uv venv` 시 Python 다운로드 실패

```
error: Request failed after 3 retries
Failed to download https://github.com/astral-sh/python-build-standalone/...
```

**원인:** uv가 자체 Python을 다운받으려는데 외부망이 차단됨.

**해결:** 시스템에 이미 설치된 Python을 지정.

```bash
uv venv --python $(which python3)
```

### 3.2 `uv sync` 시 SSL 인증서 에러 (UnknownIssuer)

```
error: Failed to download 'packaging==24.1'
invalid peer certificate: UnknownIssuer
```

**원인:** uv는 기본적으로 자체 TLS 라이브러리(rustls)를 사용하는데,
사내 SSL 프록시/방화벽의 CA 인증서를 인식하지 못함.
curl이나 pip는 시스템 OpenSSL을 사용하므로 문제없지만 uv만 실패.

**확인:** curl은 정상인지 체크.

```bash
curl -v https://pypi.org 2>&1 | grep -i "ssl\|certificate"
# "SSL certificate verify ok." 나오면 시스템 인증서는 정상
```

**해결: `--native-tls` 옵션 사용**

시스템 인증서 번들(OpenSSL)을 쓰도록 강제:

```bash
uv sync --native-tls
```

**영구 적용 (환경변수):**

```bash
export UV_NATIVE_TLS=true
uv sync
```

`~/.bashrc`에 추가해두면 매번 안 붙여도 됨:

```bash
echo 'export UV_NATIVE_TLS=true' >> ~/.bashrc
source ~/.bashrc
```

### 3.3 Python 버전 호환 에러

```
error: Python requirement '>=3.13' incompatible with Python 3.10.12
```

**원인:** `pyproject.toml`의 `requires-python`이 높게 설정됨.

**해결 1:** pyproject.toml 수정.

```bash
sed -i 's/requires-python = ">=3.13"/requires-python = ">=3.10"/' pyproject.toml
```

**해결 2:** conda로 해당 Python 버전 설치 후 사용.

```bash
conda create -n myenv python=3.13 -y
conda activate myenv
uv venv --python $(which python)
```

---

## 4. uv를 못 쓸 때 pip로 대체

uv가 외부망/인증서 문제로 안 되면 pip로 동일하게 진행 가능.

| uv 명령 | pip 대체 |
|---------|---------|
| `uv venv` | `python3 -m venv .venv` |
| `uv sync` | `pip install .` |
| `uv pip install 패키지` | `pip install 패키지` |
| `uv pip list` | `pip list` |

> `uv sync`는 `uv.lock` 파일을 읽어 정확한 버전으로 설치.
> `pip install .`은 `pyproject.toml`의 버전 범위 내 최신으로 설치.
> 대부분의 경우 결과 동일.

pip에서도 SSL 에러 나면:

```bash
pip install . --trusted-host pypi.org --trusted-host files.pythonhosted.org --trusted-host download.pytorch.org
```

---

## 5. 참고

- uv 공식 문서: https://docs.astral.sh/uv/
- `uv.lock`은 uv 전용 잠금 파일. pip은 이 파일을 무시함.
- venv 삭제는 별도 명령 없이 폴더 삭제로 충분: `rm -rf .venv`
