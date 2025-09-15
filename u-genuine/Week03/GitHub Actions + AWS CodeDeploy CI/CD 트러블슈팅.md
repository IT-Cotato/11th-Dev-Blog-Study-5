최근 코테이토에서 진행한 날씨 관련 미니 프로젝트에서 CI/CD를 구현하기 위해 GitHub Actions와 AWS CodeDeploy를 사용했다.

빌드 → S3 업로드 → CodeDeploy 배포까지는 정상적으로 완료되었는데, EC2에서 애플리케이션이 실행되지 않는 문제가 발생했다.

---

## 문제 상황

처음에는 어디서 문제가 발생했는지 파악하기 어려웠다.

-   GitHub Actions 워크플로우는 성공 상태
-   S3에도 압축 파일도 정상 업로드
-   EC2에도 파일이 복사된 것으로 보였음

그런데 애플리케이션은 실행되지 않았고, 로그도 생성되지 않았다.

---

### CodeDeploy 로그 확인

CodeDeploy 로그를 확인해보니 다음과 같은 메시지가 있었다

```
LifecycleEvent - AfterInstall
Script - scripts/deploy.sh
[stdout]> 환경 변수 설정
[stdout]> 배포 zip 파일이 존재하지 않습니다: /home/ec2-user/app/step2/zip/cotato-11th-weather.zip
```

scripts/deploy.sh 파일은 정상적으로 실행된 것 처럼 보였지만

스크립트 내부에서 필요한 JAR 파일 또는 zip 파일을 찾지 못하면서 실행이 중단되었다.

>  💡 **CodeDeploy 동작 과정**
>
> 1\. S3에서 압축 파일 다운로드  
> 2, EC2의 임시 디렉토리에 압축 해제  
>    (Linux 기준) /opt/codedeploy-agent/deployment-root/\[deployment-id\]/  
> 3\. appspec.yml의 files 항목에 따라 파일들을 목적지로 복사  
> 4\. hooks 에 정의된 스크립트들 (scripts/cleanup.sh, scripts/deploy.sh)을 임시 디렉토리에서 실행

---

## 삽질의 과정

### 당시 배포 설정

**GitHub Actions (deploy.yml)**

```
- name: Prepare deploy package
  run: |
    mkdir -p before-deploy/scripts
    cp scripts/*.sh before-deploy/scripts/
    cp appspec.yml before-deploy/
    cp build/libs/*.jar before-deploy/
```

**CodeDeploy 설정 (appspec.yml)**

```
files:
  - source: .  # 압축 해제된 모든 파일을
    destination: /home/ec2-user/app/step2/zip/  # 이 경로로 복사
    
hooks:
  BeforeInstall:
    - location: scripts/cleanup.sh
  AfterInstall:
    - location: scripts/deploy.sh
```

**압축 구조**

```
before-deploy/
├── scripts/
│   ├── deploy.sh
│   └── cleanup.sh
├── appspec.yml
└── cotato-11th-weather-plain.jar
```

**압축 해제 후 EC2에서의 임시 구조**

```
/opt/codedeploy-agent/deployment-root/[deployment-id]/
├── scripts/
│   ├── deploy.sh
│   └── cleanup.sh
├── appspec.yml
└── app.jar
```

그리고 **appspec.yml**의 **files.destination** 설정에 따라 이 모든 파일은 /home/ec2-user/app/step2/zip/ 에 복사된다

---

### ❌ 잘못된 접근 과정

처음에는 이 문제가 .sh 파일이 scripts/ 폴더 아래에 있었기 때문이라고 생각했다.

그래서 .sh파일을 루트에 복사하고, **appspec.yml** 의 hooks 경로도 scripts/deploy.sh → deploy.sh로 바꿨다

**수정된 appspec.yml**

```
hooks:
  BeforeInstall:
    - location: cleanup.sh
  AfterInstall:
    - location: deploy.sh
```

**변경된 압축 구조**

```
before-deploy/
├── deploy.sh
├── cleanup.sh
├── appspec.yml
└── cotato-11th-weather-plain.jar
```

하지만 문제는 해결되지 않았다..

---

## 실제 원인 - 실행 불가능한 JAR 선택

[**deploy.sh**](http://deploy.sh) 스크립트 \*\*\*\*내부에는 다음 코드가 포함되어 있었다

```
#!/bin/bash

APP_DIR=/home/ec2-user/app/step2
ZIP_DIR=$APP_DIR/zip

...

**JAR_NAME=$(ls -tr $ZIP_DIR/*.jar | tail -n 1)**
```

이 스크립트는 EC2의 /home/ec2-user/app/step2/zip/ 디렉토리에 .jar 파일 목록 중 가장 마지막에 생성된 파일을 선택한다.

하지만 빌드 시 생성된 JAR 파일은 두 가지였다

```
cotato-11th-weather.jar            # 실행 가능한 JAR
cotato-11th-weather-plain.jar      # 실행 불가능한 plain JAR
```

마지막으로 생성된 -plain.jar이 선택되었고,

이 파일은 실행할 수 없는 JAR이기 때문에 서버가 실행되지 않았던 것이다.

---

### 해결 방법

스크립트에서 실행 가능한 JAR만 선택하도록 필터 조건을 추가했다.

```
JAR_NAME=$(ls -tr $ZIP_DIR/*.jar | **grep -v 'plain'** | tail -n 1)
```

이제 -plain.jar 를 제외한 실제 실행 가능한 .jar 파일만 선택되면서

애플리케이션이 정상적으로 실행됐다 !

---

### 정리

-   CI/CD 구성 자체는 문제 없었지만,
-   **스크립트 내부 로직에서 실행 불가능한 JAR을 선택한 것**이 문제였다.
-   .sh 경로 구조를 바꾸는 시도는 잘못된 방향이었고,
-   grep -v 'plain' 조건 하나로 모든 문제가 해결되었다.