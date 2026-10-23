# 7주차 - 도커 컴포즈

## 이번 주에 한 일
방명록 앱과 Redis(DB)를 서로 다른 컨테이너로 나눠서 `compose.yaml` 하나로 실행하고, 비밀번호 분리, healthcheck, nginx와 scale까지 단계별로 다듬었다.

## 1. 왜 나누고 왜 컴포즈인가
컨테이너 하나에는 역할 하나만 맡긴다. 앱과 DB를 한 컨테이너에 넣으면 앱만 업데이트해도 DB가 재시작되고, 앱만 늘리기 어렵고, 장애를 따로 다루기 어렵다.

docker run으로 컨테이너를 여러 개 띄우면 명령이 길고 오타가 나기 쉽고 팀원에게 전달하기도 어렵다. 구성을 `compose.yaml` 파일로 적어 두면 같은 구성을 누구나 같은 명령으로 실행한다. 컴포즈는 컨테이너를 하나로 합치는 도구가 아니라 독립된 컨테이너들이 함께 동작하도록 구성하고 관리하는 도구다.

## 2. compose.yaml과 네트워크
- `services` 아래의 이름(web, redis)이 서비스 이름이다.
- 컴포즈가 네트워크를 만들고 서비스 이름을 내부 DNS에 등록해서, web은 IP 대신 `REDIS_HOST: "redis"`처럼 이름으로 Redis를 찾는다. IP가 바뀌어도 상관없다.

```bash
docker compose up -d      # 네트워크 1 + 볼륨 1 + 컨테이너 2개를 한 번에
docker compose ps
docker compose logs -f web
```

`docker compose ps` 결과 (healthy 포함):
```
(여기에 실제 docker compose ps 결과 붙여넣기)
```


## 3. 볼륨과 down / down -v
web만 재시작해도 글이 남는 이유는 데이터가 web이 아니라 redis 컨테이너에 있기 때문이다. 컨테이너를 통째로 지워도 별도로 만든 볼륨은 남는다.

| 명령 | 지우는 것 | 방명록 글 |
|---|---|---|
| `docker compose down` 후 `up -d` | 컨테이너, 네트워크 | 남아 있음 |
| `docker compose down -v` 후 `up -d` | 컨테이너, 네트워크, 볼륨 | 사라짐 |

운영 환경에서 `-v`를 잘못 실행하면 중요한 데이터가 지워질 수 있어서 조심해야 한다.


## 4. 네트워크 격리
redis에는 `ports:`를 적지 않아서 호스트에서 `127.0.0.1:6379`로 접속하면 `Connection refused`가 난다. 같은 네트워크의 web 컨테이너 안에서는 `redis`라는 이름이 IP로 풀려서 접속된다. 앱만 외부에 열고 DB는 내부에서만 접근하게 하는 기본적인 보안 설계다.

## 5. 컴포즈 다듬기
| 단계 | 문제 | 해결 |
|---|---|---|
| 3-1 `.env` | 비밀번호가 compose.yaml에 그대로 적혀 있음 | `${REDIS_PASSWORD}`로 바꾸고 값은 `.env`에 둠 |
| 3-2 healthcheck | depends_on은 시작 순서만 보장해서 Redis 준비 전에 web이 연결을 시도할 수 있음 | `condition: service_healthy`로 준비 완료까지 기다림 |
| 3-3 nginx + scale | web을 3개 띄우면 8080 포트 충돌, ports를 빼면 바깥에서 접근 불가 | nginx가 8080을 열고 web들에게 요청을 나눠 줌 |
| 3-4 config, watch | YAML 오류는 띄워 봐야 알고, 고칠 때마다 빌드를 손으로 함 | `docker compose config`로 미리 검사, `watch`로 자동 반영 |

### .env를 Git에 올리지 않는 이유
`.env`에는 비밀번호 같은 비밀 값이 들어 있다. GitHub에 올리면 누구나 볼 수 있고, 나중에 파일에서 지워도 Git 이력에 남을 수 있다. 그래서 `.gitignore`에 `.env`를 넣어 실수로 커밋하는 걸 막고, 이름만 알려 주는 `.env.example`만 올린다. `.dockerignore`에도 넣어서 이미지 안에 복사되지 않게 한다. (`.gitignore`는 암호화가 아니라 올리지 않을 뿐이다)

### scale 결과
```bash
docker compose up -d --scale web=3
```
nginx가 Docker 내부 DNS에서 web의 IP 3개를 받아 돌아가며 요청을 보내서, 새로고침할 때마다 Pod/Host(컨테이너 ID)가 바뀌었다. 글은 어느 web으로 가도 같은데 데이터가 하나의 redis에 있기 때문이다.


## 6. 호기심: web 하나를 멈추면
`docker stop guestbook-web-2`로 하나를 멈춰도 남은 web 2개가 번갈아 응답해서 서비스는 살아 있다. 하지만 멈춘 컨테이너를 다시 살리는 건 없다. 직접 `docker stop`한 컨테이너는 `restart: unless-stopped`로도 자동 재시작되지 않아서, 사람이 `up -d --scale web=3`을 다시 입력해야 3개로 돌아온다.

## 7. 컴포즈의 한계
컴포즈는 컴퓨터 한 대 안에서만 동작한다. 서버가 여러 대이거나, 서버가 고장 났을 때 다른 서버로 옮기거나, 무중단 배포를 하려면 다음 시간의 도구가 필요하다.

## 느낀 점 / 막혔던 점
앱과 DB를 나눠서 띄우니 web만 재시작해도 글이 남는다는 게 직접 보여서 이해가 잘 됐다. 비밀번호를 `.env`로 분리하는 이유도 깃 이력에 남을 수 있다는 걸 알고 나서 납득이 됐다.
