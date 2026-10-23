# 6주차 - 도커파일

## 이번 주에 한 일
Dockerfile로 방명록 앱(guestbook)을 이미지로 만들고, ghcr.io에 올려서 친구의 이미지를 받아 실행했다.

## 1. 내 이미지 주소
```
ghcr.io/ericlair/guestbook:v1
```
주소는 `레지스트리 / 계정 / 이미지 이름 : 태그`로 읽는다. 레지스트리를 생략하면 Docker Hub, 태그를 생략하면 `latest`를 요청해서 그 태그가 없으면 실패한다.

## 2. Dockerfile 한 줄씩
| 명령어 | 내가 이해한 역할 |
|---|---|
| `FROM python:3.12-slim` | 기반이 되는 이미지. 내 이미지는 이 위에 쌓인다 |
| `WORKDIR /app` | 이미지 안의 작업 폴더 지정(없으면 생성) |
| `COPY requirements.txt .` | 라이브러리 목록만 먼저 복사 |
| `RUN pip install ...` | 이미지를 만드는 동안 한 번 실행해서 라이브러리 설치 |
| `COPY . .` | 나머지 코드를 복사 |
| `ENV APP_TITLE=...` | 환경 변수 기본값을 이미지에 넣음 |
| `EXPOSE 5000` | 쓰는 포트를 알리는 메모일 뿐 실제로 열지는 않음 |
| `USER appuser` | root가 아닌 사용자로 실행해서 위험을 줄임 |
| `CMD ["python", "app.py"]` | 컨테이너가 시작될 때마다 실행할 명령 |

RUN은 이미지를 만들 때 한 번, CMD는 컨테이너를 켤 때마다 실행된다. 포트를 실제로 열려면 `docker run -p 8080:5000`처럼 따로 지정해야 한다.

## 3. 빌드와 캐시
```bash
docker build -t guestbook:v1 .
```
도커는 단계마다 레이어를 만들고, 바뀌지 않은 단계는 캐시를 쓴다. 한 레이어가 바뀌면 그 아래 단계는 전부 다시 만들어진다. 그래서 자주 바뀌는 코드 복사는 뒤에, 잘 안 바뀌는 `requirements.txt` 복사와 `pip install`은 앞에 둔다.

- 아무것도 안 고치고 다시 빌드하면 대부분 `CACHED`로 표시됐다.
- `templates/index.html`만 고치면 `COPY requirements.txt`와 `pip install`은 CACHED, `COPY . .`부터 다시 실행됐다.
- 반대로 `COPY . .`를 먼저 둔 Dockerfile.bad는 코드 한 줄만 고쳐도 `pip install`이 다시 실행됐다.


## 4. 설정을 넣는 세 가지 방법
우선순위는 `코드 기본값(app.py) < Dockerfile의 ENV < docker run -e` 순서다.

| 방법 | 확정 시점 | 바꾸려면 |
|---|---|---|
| app.py 기본값 | 코드 | 코드 수정 후 재빌드 |
| Dockerfile ENV | 이미지 빌드 시점 | 재빌드 |
| docker run -e | 컨테이너 실행 시점 | 컨테이너만 다시 실행 |

앱 코드의 기본값은 빨간색인데 화면이 파란색으로 나왔던 이유가 이 순서 때문이다. ENV로 구워 둔 값이 있어도 `-e`로 주면 그 값이 이긴다.


## 5. 이미지 공유
패키지를 Public으로 바꿔야 친구가 받을 수 있다. 친구 이미지는 `docker run -d -p 9090:5000 --name friend ghcr.io/친구아이디/guestbook:v1`로 실행한다. 친구의 소스 코드나 Flask 설치 없이도 내 컴퓨터에서 돌아간다. 다만 이미지는 프로그램만 공유하고 방명록에 쓴 데이터는 공유하지 않는다.


## 6. 혼자 해보기: V2 만들기
- 제목·색(Dockerfile ENV)과 문구·레이아웃(templates/index.html)을 3곳 이상 바꿔 `guestbook:v2`를 빌드하고 push했다.


## 참고
- 맥(arm64)에서 만든 이미지를 amd64 PC에서 실행하면 문제가 생길 수 있어서 `docker buildx build --platform linux/amd64,linux/arm64`로 멀티 아키텍처 이미지를 만든다.
- 방명록에 쓴 글은 프로세스 메모리에만 있어서 컨테이너를 재시작하면 사라진다. 이 문제는 다음 주 컴포즈와 볼륨에서 다룬다.

## 느낀 점 / 막혔던 점
내가 만든 이미지를 올려서 친구가 설치 없이 바로 실행할 수 있다는 공유 방식이 신기했다. 방명록에 쓴 데이터는 같이 공유되지 않는다는 점은 실습 해보고 알게 되었다.
