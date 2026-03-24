# Jenkins에 대해 학습한 내용을 공유하는 레포지토리입니다.

> Jenkins? CI (Continous Integration) / CD (Continous Delivery) / CD (Continous Deployment) 를 자동화하는 오픈소스 소프트웨어

Jenkins를 사용하는 이유
- 자동화: 코드가 변경되면 자동으로 빌드/테스트/배포 가능
- 시간 절약: 반복적인 작업을 자동으로 처리하여 시간 절약 가능
- 오류 감소: 수동 작업으로 인한 오류 감소
---

## Jenkins 설치

> Docker를 활용하여 간단하고 빠르게 Jenkins 환경 구성 가능

- **Docker로 Jenkins 컨테이너 실행**

  ```bash
  docker run --name myjenkins -p 8080:8080 jenkins/jenkins:lts-jdk17
  ```

- 설치 후 컨테이너 로그를 통해 초기 비밀번호 확인: 

```bash
docker logs myjenkins -f
```

<img width="1183" height="463" alt="스크린샷 2026-03-24 155349" src="https://github.com/user-attachments/assets/ccf8b135-3f6f-4528-b526-c9136fa2b0e9" />
<img width="1223" height="850" alt="스크린샷 2026-03-24 155623" src="https://github.com/user-attachments/assets/5fff7a43-95f7-42ef-b9f6-f1ca2720aec6" />
<img width="1186" height="813" alt="스크린샷 2026-03-24 155809" src="https://github.com/user-attachments/assets/db6bfa84-5474-4b20-a496-960dafb51f09" />

---

## 외부 연동 설정 (ngrok & GitHub Webhook)

> 로컬 구동 중인 Jenkins를 인터넷망에 노출하여 GitHub 이벤트 알림을 수신하도록 설정

- **ngrok 포트 포워딩**
  - 로컬 80 포트를 외부에서 접근 가능하도록 터널링: `ngrok http 80`

- **GitHub Webhook 등록**
  - 레포지토리 `Settings` > `Webhooks`에서 Payload URL에 ngrok 발급 주소(`https://...ngrok-free.app/github-webhook/`) 등록
  
  - 코드가 레포지토리에 푸시(Push)될 때마다 Jenkins로 자동 빌드 트리거 발송
    
<img width="1070" height="524" alt="스크린샷 2026-03-24 160248" src="https://github.com/user-attachments/assets/beca8b5b-7462-4ba3-b1e7-ec6f3e6701b7" />

```bash
ngrok config add-authtoken your_token
```
<img width="1109" height="517" alt="스크린샷 2026-03-24 160342" src="https://github.com/user-attachments/assets/f2322ccb-30e2-412a-8c05-6a1322dee998" />

<img width="751" height="364" alt="스크린샷 2026-03-24 오후 4 07 33" src="https://github.com/user-attachments/assets/87f62ae4-698d-4fd4-a5c7-9996fd249c7a" />

- 레포지토리 설정에서 웹훅 추가

---

## Jenkins Pipeline 구성

> 체크아웃 -> 빌드 -> 테스트 -> 배포 등의 모든 단계를 정의하고 관리하는 스크립트 도구

- **파이프라인 종류**
  - **선언형(Declarative) 파이프라인**: 가독성이 뛰어나고 구조화되어 있어 가장 널리 권장
  - **스크립트형(Scripted) 파이프라인**: Groovy 기반 문법으로 복잡한 로직 및 조건 제어 시 활용

- **기본 구조 (선언형 예시)**
  ```groovy
  pipeline {
    agent any
    stages {
      stage('Checkout') { ... } // GitHub 코드 가져오기
      stage('Build') { ... }    // 빌드 실행 (Gradle 등)
    }
  }
  ```
---

## 빌드 도구 (Gradle) 활용

> Java 프로젝트의 빌드, 테스트, 의존성 라이브러리를 관리해주는 도구

- **빌드 명령어 실행 (명령어 퍼미션 주의)**
  - `chmod +x gradlew`: 최초 1회 실행 권한 부여
  - `./gradlew build`: 프로젝트 빌드 실행 (Windows는 `.\gradlew build` 활용)

- 빌드가 문제없이 성공하면 `build/libs/` 경로 내에 배포 및 실행 가능한 `app.jar` 파일이 생성됨

---

## Bind Mount를 활용한 CI/CD 파이프라인

> Docker 컨테이너(Jenkins)와 호스트 운영체제의 폴더를 연결(Bind Mount)하여, 내부에서 빌드된 결과물을 호스트로 자동 추출하는 구성

- **마운트 폴더 생성 및 Jenkins 재구동**
  - `-v` 옵션으로 호스트 폴더(`/home/ubuntu/appjardir`)와 컨테이너 폴더(`/var/jenkins_home/appjar`)를 동기화
  
  ```bash
  docker run --name myjenkins2 -p 8888:8080 \
    -v /home/ubuntu/appjardir:/var/jenkins_home/appjar \
    jenkins/jenkins:lts-jdk17
  ```
- **자동 배포 흐름 구축 로직**
  1. 개발자가 변경된 코드를 GitHub에 **Push**
  2. Webhook 기능으로 Jenkins **자동 빌드 트리거** 동작
  3. Pipeline을 거쳐 빌드된 `.jar` 파일을 마운트 타겟 위치로 **자동 복사** (`cp` 명령어 적용)
  4. 복사된 jar 파일을 호스트 컴퓨터 터미널에서 `java -jar app.jar` 명령어로 구동

<img width="738" height="345" alt="스크린샷 2026-03-24 오후 4 10 19" src="https://github.com/user-attachments/assets/6f80d10e-a4bf-4aa4-98b9-90390b47d9d9" />
<img width="752" height="650" alt="스크린샷 2026-03-24 오후 4 10 54" src="https://github.com/user-attachments/assets/23d4cb44-865e-49e7-8eb4-2ed5f25fa875" />
