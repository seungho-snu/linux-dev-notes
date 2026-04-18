# Git Remote 관리 (다중 원격 저장소 활용)

기존에 clone한 저장소에 **다른 저장소의 변경사항을 합쳐서** 사용하고 싶을 때 쓰는 패턴. 예를 들면:

- 원본(TencentARC/StereoCrafter)을 그대로 쓰다가, 내 fork에 추가한 배치 스크립트만 가져오고 싶을 때
- 동료가 같은 프로젝트에 기능을 추가해서 푸시했는데, 그걸 내 작업 디렉토리로 가져오고 싶을 때
- 원본 업데이트는 그대로 받으면서, 내 수정분을 별도 레포로 관리하고 싶을 때

## 언제 이 패턴이 유용한가

기본 전제: **두 저장소가 공통 조상(commit)을 공유**해야 깔끔하게 합쳐짐.
즉, 한 쪽이 다른 쪽을 `git clone`한 뒤 커밋을 얹은 구조일 때 가장 잘 동작한다.

```
원본:     A ─ B ─ C ─ D
내 fork:  A ─ B ─ C ─ D ─ E ─ F    ← E, F만 추가된 상태면 fast-forward 가능
```

## 원격 저장소 여러 개 관리

### 1. 현재 원격 확인

```bash
git remote -v
# origin  https://github.com/TencentARC/StereoCrafter.git (fetch)
# origin  https://github.com/TencentARC/StereoCrafter.git (push)
```

### 2. 새 원격 추가

```bash
# 'my'라는 이름으로 내 fork 저장소 추가
git remote add my https://github.com/seungho-snu/stereocrafter.git

# 추가된 원격에서 커밋 정보만 받아오기 (로컬 파일은 안 건드림)
git fetch my
```

remote 이름은 관례적으로:
- `origin` — clone 시 자동 설정되는 기본 원격
- `upstream` — 내가 fork한 원래 프로젝트를 가리킬 때
- `my` / `fork` / 사람 이름 — 내 fork나 동료의 fork

### 3. Pull 전에 뭐가 들어올지 미리 확인

```bash
# 내 현재 브랜치(HEAD)와 my/main 사이의 차이 커밋 목록
git log HEAD..my/main --oneline
```

예상한 커밋만 있으면 안전하게 진행, 뭔가 이상하면 중단 후 조사.

### 4. 합치기 (Pull)

```bash
# my 원격의 main 브랜치를 현재 브랜치로 병합
git pull my main
```

공통 조상이 있고 내가 추가 커밋을 만들지 않았으면 **fast-forward**로 깔끔하게 합쳐짐.

### 5. 결과 확인

```bash
git log --oneline -5
# 9cf00ea (HEAD -> main, my/main) Add batch processing scripts
# 52c450b (origin/main) Update inpainting_inference.py
# ...
```

`my/main`과 `HEAD`가 같은 커밋을 가리키면 성공.

## 이후 업데이트 받기

`my` 쪽에 새 커밋이 추가되면:

```bash
git fetch my
git pull my main
```

## 대안: 파일 몇 개만 가져오고 싶을 때

전체 브랜치를 합치는 게 아니라 **특정 파일 몇 개만** 가져오고 싶으면 체리픽:

```bash
git remote add my https://github.com/seungho-snu/stereocrafter.git
git fetch my

# my/main에 있는 특정 파일들만 현재 working tree로 가져옴
git checkout my/main -- batch_run.sh batch_inference.py BATCH_USAGE.md

# 본인이 원하면 커밋
git add batch_run.sh batch_inference.py BATCH_USAGE.md
git commit -m "Add batch scripts from my fork"
```

이 방식은 병합 이력이 남지 않아 **깔끔하지만**, 나중에 동일 파일이 원본에서 업데이트되면 충돌 가능성 있음.

## 원격 URL 변경 / 제거

```bash
# URL 변경 (토큰 갱신, 레포 이름 변경 등)
git remote set-url my https://github.com/seungho-snu/stereocrafter-v2.git

# 원격 제거
git remote remove my

# 원격 이름 변경
git remote rename my fork
```

## 자주 만나는 문제

### "Updates were rejected because the remote contains work..."

```
! [rejected]  main -> main (fetch first)
error: failed to push some refs to '...'
hint: Updates were rejected because the remote contains work that you do not
hint: have locally.
```

원인: 원격에 내가 모르는 커밋이 있음. 예를 들어 GitHub UI에서 "Add a README file"을 체크하고 레포를 만들면 원격에 자동 커밋이 하나 생김.

해결:
- **원격 내용이 필요 없고 내 로컬로 덮어쓰고 싶을 때:** `git push --force`
- **원격 내용도 보존해야 할 때:** `git pull --rebase my main` 후 `git push`

### Divergent branches / merge conflict

공통 조상 이후 양쪽 모두 커밋이 쌓이면 fast-forward가 안 됨:

```bash
# 옵션 A: merge (이력 보존, merge commit 생성)
git pull my main

# 옵션 B: rebase (이력 선형 유지, 내 커밋을 my/main 위에 얹음)
git pull --rebase my main
```

rebase 도중 충돌 나면:
```bash
# 충돌 파일 수정 후
git add <충돌-파일>
git rebase --continue

# 포기하고 되돌리기
git rebase --abort
```

### 원본이 force push한 경우

드물지만, upstream 쪽에서 히스토리를 갈아엎었으면 fetch 후 동기화 안 됨. 로컬을 완전히 원격 상태로 맞추려면:

```bash
git fetch my
git reset --hard my/main   # ⚠️ 로컬 커밋 날아감, 미리 백업
```

## 실전 예시: StereoCrafter + 내 배치 스크립트

기존에 `TencentARC/StereoCrafter`를 clone해서 쓰고 있는데, 내 fork(`seungho-snu/stereocrafter`)에 추가한 `batch_run.sh`, `batch_inference.py`를 가져오고 싶은 상황:

```bash
cd /path/to/StereoCrafter

# 현재 상태 깨끗한지 확인
git status

# 내 fork를 my로 추가
git remote add my https://github.com/seungho-snu/stereocrafter.git
git fetch my

# 어떤 커밋이 추가로 들어올지 미리 보기
git log HEAD..my/main --oneline
# → "Add batch processing scripts" 한 줄만 나와야 정상

# 합치기
git pull my main

# 배치 스크립트 실행 권한
chmod +x batch_run.sh

# 확인
ls batch_run.sh batch_inference.py BATCH_USAGE.md
```

이후 배치 스크립트 업데이트 받을 때:
```bash
git fetch my && git pull my main
```

## 참고

- remote 설정은 저장소별로 `.git/config`에 기록됨. `cat .git/config`로 직접 확인 가능.
- 원격 이름은 아무 문자열이나 써도 됨. 관례일 뿐.
- PAT로 원격 URL 설정하면 `.git/config`에 토큰이 평문 저장됨. 노출 위험 있으면 git credential helper 또는 SSH 키 사용 권장.
