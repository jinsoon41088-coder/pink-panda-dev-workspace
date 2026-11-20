# 개발자용 작업실 구축

## 1. 미션 목표

Windows 로컬 환경에서 PowerShell, Docker, Git을 사용해 개발 환경을 구성하고, Dockerfile 기반 웹 서버의 빌드·실행·포트 매핑·바인드 마운트·볼륨 영속성을 검증한다.

## 2. 실행 환경

| 항목 | 확인 결과 |
|---|---|
| OS | Windows 10 Home |
| 쉘 | Windows PowerShell 5.1 |
| Docker CLI | 29.9.2 |
| Docker Server | 29.9.2 (`linux`) |
| Git | 2.56.0.windows.4 |
| Docker Desktop | Engine running 확인 |

## 3. 수행 체크리스트

- [x] PowerShell 위치·목록·파일 생성·읽기·복사·이름 변경·이동·삭제
- [x] 파일 및 디렉터리 ACL 확인·변경·복원
- [x] Docker 버전·엔진 상태 확인
- [x] `hello-world`, Ubuntu 컨테이너 실행 및 기본 운영 명령
- [x] NGINX 기반 커스텀 이미지 빌드
- [x] 포트 매핑 접속 확인
- [x] 바인드 마운트 변경 반영 확인
- [x] Docker 볼륨 데이터 영속성 확인
- [x] Git 저장소 초기화 및 기본 브랜치 `main` 설정
- [ ] GitHub 저장소 연결 및 VS Code 로그인 증거
- [ ] README에 스크린샷 증거 추가

## 4. PowerShell 기본 조작 로그

개인 계정 경로는 공개하지 않도록 가렸다.

```powershell
Get-Location
# C:\Users\REDACTED\codyssey-workstation

Get-ChildItem -Force
# 숨김 항목과 일반 항목 목록 확인

New-Item -ItemType Directory -Name cli-practice
Set-Location .\cli-practice
# 작업 폴더 생성 및 이동

Set-Content -Path .\note.txt -Value "CLI practice"
Get-Content -Path .\note.txt
# CLI practice

New-Item -ItemType File -Path .\empty.txt
Get-Item .\empty.txt | Select-Object Name,Length
# empty.txt, Length 0

Copy-Item -Path note.txt -Destination note-copy.txt
Rename-Item -Path note-copy.txt -NewName note-renamed.txt
New-Item -ItemType Directory -Name archive
Move-Item -Path note-renamed.txt -Destination archive
Get-ChildItem -Recurse -Name
# archive\note-renamed.txt 확인

Remove-Item -Path empty.txt
# 직접 만든 연습용 빈 파일 삭제
```

## 5. 권한 실습

Windows에서는 `chmod 755/644` 대신 ACL을 `icacls`로 확인했다. 기존 권한을 확인한 뒤 연습 파일과 폴더에만 임시 권한을 추가하고 제거했다.

```powershell
icacls note.txt
# SYSTEM, Administrators, 현재 사용자: Full Control

icacls note.txt /grant "*S-1-5-32-545:(R)"
# BUILTIN\Users:(R) 추가 확인

icacls note.txt /remove "*S-1-5-32-545"
# 임시 권한 제거 확인

icacls archive /grant "*S-1-5-32-545:(RX)"
# BUILTIN\Users:(RX) 추가 확인

icacls archive /remove "*S-1-5-32-545"
# 임시 권한 제거 확인
```

## 6. Docker 설치 및 기본 운영 로그

```powershell
docker --version
# Docker version 29.9.2, build a82e528

docker info --format 'Server Version: {{.ServerVersion}} | OSType: {{.OSType}}'
# Server Version: 29.9.2 | OSType: linux

docker run --rm hello-world
# Hello from Docker!

docker run -it --name ubuntu-lab ubuntu bash
# 컨테이너 내부에서 실행
pwd
# /
ls
# bin, boot, dev, etc, home, root, tmp, usr 등 표시
echo "Hello from Ubuntu"
# Hello from Ubuntu
exit

docker ps -a --filter "name=ubuntu-lab"
# ubuntu-lab: Exited (0)

docker run -d --name ubuntu-keep ubuntu sleep infinity
docker exec ubuntu-keep bash -lc "echo exec works"
# exec works

docker ps --filter "name=ubuntu-keep"
# ubuntu-keep: Up 상태 확인

docker stats ubuntu-keep
# CPU 0.00%, 메모리 약 7.199 MiB 사용 확인
# Ctrl+C는 통계 표시만 종료하며 컨테이너를 중지하지 않음

docker stop ubuntu-keep
docker ps -a --filter "name=ubuntu-keep"
# 중지 상태 확인

docker run --name log-demo ubuntu bash -lc "echo log-check; echo container-finished"
docker logs log-demo
# log-check
# container-finished

docker images
# hello-world, ubuntu 이미지 확인
```

## 7. Dockerfile 기반 웹 서버

베이스 이미지로 `nginx:alpine`을 선택했다. `site/`의 정적 페이지를 NGINX 웹 루트로 복사하고, 컨테이너 포트 80을 문서화했다.

`web-demo/Dockerfile`:

```dockerfile
FROM nginx:alpine
LABEL org.opencontainers.image.title="workstation-web"
COPY site/ /usr/share/nginx/html/
EXPOSE 80
```

빌드 및 실행:

```powershell
cd web-demo
docker build -t workstation-web:2.0 .
docker run -d --name web-final -p 8082:80 workstation-web:2.0
Invoke-WebRequest -UseBasicParsing http://localhost:8082 |
  Select-Object StatusCode,Content
# StatusCode: 200
# 페이지 내용에 HOST VERSION B 표시
```

`-p 8082:80`은 호스트의 8082 포트를 컨테이너의 80 포트에 연결한다.

## 8. 바인드 마운트 검증

호스트의 `site` 폴더를 컨테이너의 NGINX 웹 루트에 읽기 전용으로 연결했다. 호스트 파일을 변경한 뒤 이미지를 다시 빌드하지 않고도 변경 내용이 응답에 반영됐다.

```powershell
docker run -d --name web-bind -p 8081:80 `
  -v "${PWD}\site:/usr/share/nginx/html:ro" nginx:alpine

# 변경 전: localhost:8081에서 HTTP 200 및 기존 페이지 확인
Set-Content -Path site\index.html `
  -Value '<!doctype html><html><body><h1>HOST VERSION B</h1></body></html>' `
  -Encoding ASCII

Invoke-WebRequest -UseBasicParsing http://localhost:8081 |
  Select-Object StatusCode,Content
# StatusCode: 200
# HOST VERSION B 확인
```

## 9. Docker 볼륨 영속성 검증

컨테이너를 삭제한 뒤에도 같은 볼륨을 연결한 새 컨테이너에서 파일을 읽을 수 있었다.

```powershell
docker volume create workstation-data
docker run -d --name volume-test -v workstation-data:/data ubuntu sleep infinity
docker exec volume-test bash -lc "echo persistent-data > /data/persist.txt; cat /data/persist.txt"
# persistent-data

docker rm -f volume-test
docker run --rm -v workstation-data:/data ubuntu cat /data/persist.txt
# persistent-data
```

## 10. Git 설정 및 GitHub 연동

```powershell
git --version
# git version 2.56.0.windows.4

git config --global --get init.defaultBranch
# main

git init
# codyssey-workstation 저장소 초기화

git status --short --branch
# ## No commits yet on main
```

`git config --list` 결과를 기록할 때는 개인 이름·이메일 등 식별 정보를 가린다. GitHub 저장소 URL과 VS Code 연동이 완료되면 이 섹션에 저장소 링크와 증거를 추가한다. 토큰·비밀번호·개인키는 문서나 스크린샷에 넣지 않는다.

## 11. 트러블슈팅

1. **Docker 엔진에 연결할 수 없음**
   - 증상: `docker info`에서 `dockerDesktopLinuxEngine` 연결 실패.
   - 원인: Docker Desktop 엔진이 실행되지 않음.
   - 확인: `docker desktop status`에서 Docker Desktop이 실행 중이 아니라고 표시됨.
   - 해결: Docker Desktop을 시작하고 `Engine running` 상태를 확인한 뒤 `docker info` 재검증.

2. **Windows ACL에서 Users 그룹 이름을 찾지 못함**
   - 증상: `BUILTIN\Users` 이름 매핑 오류.
   - 해결: SID `*S-1-5-32-545`로 임시 권한을 부여하고, 확인 후 같은 SID 권한을 제거.

3. **Docker 이미지 빌드 컨텍스트 누락**
   - 증상: `docker buildx build requires 1 argument`.
   - 원인: 빌드 경로 인수 누락.
   - 해결: 현재 폴더를 뜻하는 `.`을 포함해 `docker build -t workstation-web:2.0 .` 실행.

## 12. 증거 자료

아래 스크린샷을 `screenshots/`에 저장한 뒤 링크를 확인한다.

- [ ] Docker Desktop 엔진 상태: `screenshots/docker-engine.png`
- [ ] 브라우저 주소창과 웹 페이지 응답(`localhost:8082`): `screenshots/web-8082.png`
- [ ] VS Code의 GitHub 로그인 및 저장소 연결: `screenshots/vscode-github.png`
