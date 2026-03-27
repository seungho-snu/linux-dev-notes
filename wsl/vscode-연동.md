# WSL2 + VS Code 연동

## VS Code에서 WSL 접속

1. Windows VS Code에 **Remote - WSL** 확장 설치
2. 접속 방법 2가지:
   - VS Code 좌하단 초록 아이콘 → "Connect to WSL"
   - WSL 터미널에서 `code .` 실행 (현재 폴더를 VS Code로 열기)

```bash
# WSL 터미널에서 프로젝트 열기
cd ~/project/ml-sharp
code .
```

3. 하단 상태바에 **WSL: Ubuntu** 표시 확인

## Python 인터프리터 선택

VS Code에서 conda 환경의 Python을 사용하려면:

1. `Ctrl+Shift+P` → "Python: Select Interpreter"
2. `~/miniconda3/envs/sharp/bin/python` 등 원하는 환경 선택

## WSL GPU 관련

```bash
# GPU 인식 확인
nvidia-smi

# WSL에서는 Windows 쪽 NVIDIA 드라이버를 공유함
# → WSL 안에 별도 NVIDIA 드라이버 설치하면 안 됨!
# → Windows에서 드라이버를 업데이트하면 WSL에도 반영됨
```

## 파일 접근

```bash
# WSL에서 Windows 파일 접근
ls /mnt/c/Users/seungho/Desktop/

# Windows에서 WSL 파일 접근 (탐색기 주소창)
\\wsl$\Ubuntu\home\seungho\
```
