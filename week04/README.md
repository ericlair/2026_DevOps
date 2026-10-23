# 4주차 - 자동화와 협업, 네트워크

## 이번 주에 한 일
셸 스크립트로 백업을 자동화하고, Git 브랜치 → PR → Conflict 해결 → Merge 흐름을 직접 해봤다. 네트워크 기본 개념과 확인 명령어도 배웠다.

## 1. 경로와 검색
- 절대 경로는 `/`부터 시작하고, 상대 경로는 현재 위치 기준이다. `.`은 현재, `..`은 상위, `~`는 홈이다.
- `find . -type f -name "*.sh"`로 파일을 찾고, `grep -R -n "exit 1" week03`으로 내용을 찾는다.
- `|`(파이프)는 앞 명령의 출력을 뒤 명령의 입력으로 넘긴다. 예: `grep ERROR server.log | wc -l`
- `>`는 출력을 파일로 저장한다.

## 2. 셸 스크립트 문법
- 변수는 `NAME="값"`처럼 `=` 앞뒤에 공백이 없어야 하고, 쓸 때는 `$NAME`이다.
- `$#`은 인자 개수, `$1`은 첫 번째 인자, `$?`는 직전 명령의 종료 상태다.
- `[ -f 파일 ]`, `[ -d 폴더 ]`로 존재 여부를 검사하고, `for f in *.sh; do ...; done`으로 반복한다.

## 3. 프로세스와 백업
- 프로그램은 디스크의 실행 코드, 프로세스는 실행 중인 프로그램이다.
- `kill <PID>`는 정상 종료를 요청(SIGTERM)하고, `kill -9`는 강제 종료(SIGKILL)라 최후의 수단이다.
- 백업 스크립트(backup.sh)는 원본 폴더 확인 → 백업 폴더 준비 → `tar -czf`로 압축 → 결과 확인 → 목록 출력 순서로 짰다.
- `tar -tzf`로 내용을 보고, `tar -xzf -C restore-check`로 풀어서 `diff -rq`로 원본과 같은지 비교했다. 종료 코드 0이면 같은 것이다.
- 백업 결과물은 `.gitignore`에 넣어 Git에 올리지 않았다.

[여기에 캡처: backup.sh 실행 결과, diff 결과]

## 4. Git 협업 흐름
```bash
git switch -c feature/backup-log          # 작업용 브랜치
git add week04/backup.sh && git commit -m "feat: 백업 로그 기록 기능 추가"
git push -u origin feature/backup-log
gh pr create --title "..." --body "..." --base main
gh pr merge --squash --delete-branch
git switch main && git pull --ff-only
```
- 브랜치는 main이 망가지지 않게 따로 작업하는 공간이고, PR은 합치기 전에 검토를 받는 요청이다.
- 병합 방식은 세 가지다. `--merge`는 커밋을 유지하고 병합 커밋을 만들고, `--squash`는 여러 커밋을 하나로 합치고, `--rebase`는 커밋을 main 뒤에 일렬로 붙인다.

[여기에 캡처: PR 화면, Merge 후 화면]

## 5. Conflict
같은 줄을 main과 fix/message 브랜치에서 서로 다르게 고쳐서 일부러 충돌을 만들었다. Git이 망가진 게 아니라 어느 쪽을 남길지 사람이 정하라는 신호다.
```
<<<<<<< HEAD
echo "[완료] 로그 기록"
=======
echo "로그 기록"
>>>>>>> origin/main
```
해결은 표시 줄 세 개를 지우고 남길 내용만 편집한 뒤 `add → commit → push → PR 병합` 순서로 한다.

[여기에 캡처: 충돌 표시 화면, 해결 후 Merge 성공 화면]

## 6. main 브랜치 보호
Settings > Rulesets에서 "Require a pull request before merging"을 켜면 main에 직접 push할 때 `GH006: Protected branch update failed` 오류가 난다. (비공개 저장소는 유료 플랜이 필요할 수 있다)

## 7. 네트워크 기초
- IP는 장치의 주소, 포트는 그 안에서 서비스를 구분하는 번호다. `127.0.0.1`은 나 자신(localhost)이다.
- DNS는 도메인 이름을 IP로 바꿔 준다.
- 확인 명령어: `ip addr`(IP), `ping`(응답), `dig`(DNS), `curl -I`(HTTP 헤더), `ss -tlnp`(LISTEN 포트)
- ping이 실패해도 서버가 죽은 게 아닐 수 있다. 방화벽이 ICMP를 막는 경우가 많다.
- `python3 -m http.server 8080`을 두 번 띄우면 `Address already in use`, 즉 포트 충돌이 난다. 다른 포트를 쓰거나 `ss -tlnp | grep 8080`으로 PID를 찾아 `kill`한다.

[여기에 캡처: `ss -tlnp | grep 8080`, 포트 충돌 오류 화면]

## 느낀 점 / 막혔던 점
거의 이론으로 접하던 네트워크 관련 실습이라, 다른 과목 프로젝트에서 서버가 어떻게 응답을 받는지 알아볼 수 있었던 점이 좋았다.
