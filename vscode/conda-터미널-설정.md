# VS Code 터미널에서 Conda 인식시키기 (Windows)

VS Code에서 터미널을 열면 `conda`를 찾지 못하는 문제 해결.

> 💡 WSL 환경에서의 VS Code 연동은 → [wsl/vscode-연동.md](../wsl/vscode-연동.md) 참고

## 원인

Windows에서 conda는 별도 초기화(`activate.bat`)가 필요한데, 일반 `cmd.exe`에는 이 초기화가 없어서 `conda`를 찾을 수 없다는 에러가 발생함.

## 방법 1: 기본 터미널을 Conda Prompt로 설정 (추천) ⭐

1. `Ctrl + Shift + P` → **"Preferences: Open Settings (JSON)"** → **User Settings (JSON)** 선택
2. 아래 내용 추가:

```json
{
    "terminal.integrated.profiles.windows": {
        "Conda Prompt": {
            "path": "cmd.exe",
            "args": ["/K", "C:\\Users\\<유저명>\\miniconda3\\Scripts\\activate.bat"]
        }
    },
    "terminal.integrated.defaultProfile.windows": "Conda Prompt"
}
```

> Anaconda 사용 시 경로를 `C:\\Users\\<유저명>\\anaconda3\\Scripts\\activate.bat`으로 변경.

3. 저장 후 터미널 새로 열면(`Ctrl + ~`) conda가 바로 인식됨.

## 방법 2: 현재 터미널에서 즉시 활성화

이미 열린 cmd 터미널에서 직접 실행:

```
C:\Users\<유저명>\miniconda3\Scripts\activate.bat
```

`conda --version`으로 확인. 단, 터미널 닫으면 다시 입력해야 함.

## 방법 3: conda init으로 영구 등록

cmd에서 아래를 한 번 실행하면 이후 자동 초기화:

```
C:\Users\<유저명>\miniconda3\Scripts\activate.bat
conda init cmd.exe
```

실행 후 터미널 재시작.

## Settings 파일 참고

| 항목 | 설명 |
|------|------|
| **User Settings (JSON)** | 전체 VS Code에 적용되는 개인 설정 ← **이것을 선택** |
| Default Settings | 읽기 전용, 수정 불가 |
| Workspace Settings | 현재 프로젝트(폴더)에만 적용 |

파일 실제 경로: `C:\Users\<유저명>\AppData\Roaming\Code\User\settings.json`
