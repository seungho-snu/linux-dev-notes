# SSL 인증서 / 프록시 문제 해결

사내망 등에서 SSL 인증서 검증 실패나 프록시 때문에 다운로드가 막힐 때.
pip, uv, modelscope, huggingface, git, curl 등 대부분의 도구에 공통 적용.

> 📚 uv 전용 SSL 해결(`--native-tls`)은 → [uv/설치-및-SSL-설정.md](../uv/설치-및-SSL-설정.md)

## 증상

```
SSL: CERTIFICATE_VERIFY_FAILED
certificate verify failed: unable to get local issuer certificate
SSLError / SSLCertVerificationError
```

회사망에서 보안 장비가 HTTPS 트래픽을 가로채(중간자) 자체 인증서로 다시 서명하기 때문에,
표준 인증서 목록에 없는 사내 인증서라 검증이 실패하는 것.

---

## 환경변수로 해결 (도구 공통)

사내 인증서(`.crt`/`.pem`) 경로와 프록시를 환경변수로 지정.
대부분의 Python 도구(requests 기반), curl이 이 변수들을 따름.

```bash
# 사내 인증서 경로 지정
export REQUESTS_CA_BUNDLE=/path/to/your.crt    # requests 라이브러리 (huggingface, modelscope 등)
export SSL_CERT_FILE=/path/to/your.crt         # Python ssl 모듈 전반
export CURL_CA_BUNDLE=/path/to/your.crt        # curl / wget 일부

# 프록시 설정
export HTTP_PROXY=http://proxy주소:포트
export HTTPS_PROXY=http://proxy주소:포트
```

설정 후 바로 다운로드:

```bash
modelscope download --model Wan-AI/Wan2.2-I2V-A14B --local_dir ./models/Wan-AI/Wan2.2-I2V-A14B
```

> 💡 인증서 파일(`.crt`)은 보통 회사 IT에서 받거나, 브라우저에서 사이트 인증서를 내보내기 하거나,
> `/etc/ssl/certs/`에 사내 인증서가 이미 설치되어 있는 경우도 있음.

---

## 매번 안 치고 자동 적용하기 ⭐

### 방법 1: ~/.bashrc에 추가 (전역, 가장 간단)

모든 터미널 세션에 자동 적용.

```bash
# ~/.bashrc 편집
nano ~/.bashrc
```

파일 맨 아래에 추가:

```bash
# ===== 사내망 SSL / 프록시 설정 =====
export REQUESTS_CA_BUNDLE=/path/to/your.crt
export SSL_CERT_FILE=/path/to/your.crt
export CURL_CA_BUNDLE=/path/to/your.crt
export HTTP_PROXY=http://proxy주소:포트
export HTTPS_PROXY=http://proxy주소:포트
```

적용:

```bash
source ~/.bashrc        # 또는 터미널 새로 열기
```

확인:

```bash
echo $REQUESTS_CA_BUNDLE
echo $HTTPS_PROXY
```

### 방법 2: conda 환경에서만 자동 적용

특정 conda 환경 활성화할 때만 적용하고 싶을 때 (환경마다 다른 설정이 필요하거나, 집/회사 환경을 분리할 때 유용).

```bash
# 활성화 시 실행될 스크립트 디렉토리 생성
mkdir -p $CONDA_PREFIX/etc/conda/activate.d

# 환경변수 스크립트 작성
cat > $CONDA_PREFIX/etc/conda/activate.d/ssl_proxy.sh << 'EOF'
export REQUESTS_CA_BUNDLE=/path/to/your.crt
export SSL_CERT_FILE=/path/to/your.crt
export CURL_CA_BUNDLE=/path/to/your.crt
export HTTP_PROXY=http://proxy주소:포트
export HTTPS_PROXY=http://proxy주소:포트
EOF
```

이제 `conda activate 환경명` 할 때마다 자동 적용됨.

> 비활성화 시 해제하려면 `deactivate.d/`에 `unset` 스크립트를 추가하면 됨.

### 방법 3: 별도 스크립트로 분리 (필요할 때만 source)

집/회사를 오가며 가끔만 필요하면, 별도 파일로 만들어 두고 필요할 때만 불러오기.

```bash
# ~/proxy_on.sh 작성
cat > ~/proxy_on.sh << 'EOF'
export REQUESTS_CA_BUNDLE=/path/to/your.crt
export SSL_CERT_FILE=/path/to/your.crt
export CURL_CA_BUNDLE=/path/to/your.crt
export HTTP_PROXY=http://proxy주소:포트
export HTTPS_PROXY=http://proxy주소:포트
echo "사내망 프록시/SSL 설정 적용됨"
EOF

# 필요할 때만 실행
source ~/proxy_on.sh
```

---

## 도구별 개별 옵션 (환경변수 대신)

환경변수를 쓰기 싫거나 일회성이면 각 도구의 옵션을 직접 사용.

```bash
# pip — 인증서 검증 우회
pip install 패키지 --trusted-host pypi.org --trusted-host files.pythonhosted.org

# pip — 사내 인증서 지정
pip install 패키지 --cert /path/to/your.crt

# uv — 시스템 인증서 저장소 사용
uv sync --native-tls

# git — SSL 검증 끄기 (보안 주의, 임시로만)
git config --global http.sslVerify false

# git — 사내 인증서 지정 (권장)
git config --global http.sslCAInfo /path/to/your.crt

# curl / wget — 검증 무시 (보안 주의)
curl -k https://...
wget --no-check-certificate https://...
```

> ⚠️ `sslVerify false`, `-k`, `--no-check-certificate`는 검증 자체를 끄는 거라
> 보안상 위험. 가능하면 인증서 경로를 지정하는 방식을 우선 사용.

---

## 요약

| 상황 | 방법 |
|------|------|
| 매 세션 자동 (전역) | `~/.bashrc`에 export 추가 |
| 특정 conda 환경만 | `$CONDA_PREFIX/etc/conda/activate.d/` |
| 가끔만 필요 | 별도 `.sh` 만들고 `source` |
| 일회성 다운로드 | 환경변수를 명령 앞에 직접 |
