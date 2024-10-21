# 구독추천 Manifest 관리

이 가이드는 구독추천 서비스의 CI/CD 파이프라인을 만드는 방법에 대해 설명합니다.  

## 사전준비
- Tool chains: 실습에서는 Jenkins, SonarQube, ArgoCD만 설치 필요  
  - Jenkins: CI/CD Pipeline script를 작성  
  - SonarQube: Code품질 검사와 테스트 커버리지를 검사하는 툴  
  - ArgoCD: Git Manifest의 변화를 감지하여 배포를 수행하는 툴  
  - Image registry: 컨테이너 이미지 저장소. DockerHub를 사용  
  - Git: 소스 형상 관리 툴. GitHub 사용  
  - Trivy: 컨테이너 이미지 보안 취약성 검사 툴. 파이프라인 수행 Pod내 컨테이너로 자동 생성  
  - kubectl: k8s CLI. 파이프라인 수행 Pod내 컨테이너로 자동 생성  
  - node: React 라이브러리 설치 및 실행 파일 빌드 툴. 파이프라인 수행 Pod내 컨테이너로 자동 생성  
  - gradle: Spring Boot 애플리케이션 실행 파일 빌드 툴. 파이프라인 수행 Pod내 컨테이너로 자동 생성  
  - sonar-scanner: React 소스의 코드 품질 검사를 하여 SonarQube에 전송. 파이프라인 수행 Pod내 컨테이너로 자동 생성  
  - podman: 컨테이너 이미지 빌드와 푸시 툴. 파이프라인 수행 Pod내 컨테이너로 자동 생성  
  - envsubst: 환경변수로 manifest파일 내용을 치환하는 툴. 파이프라인 수행 Pod내 컨테이너로 자동 생성  

- CI/CD Toolchain 주소: 도메인은 현재는 msa.edutdc.com이며 변경될 수 있음  
  - Jenkins: http://jenkins.{Domain}
  - SonarQube: http://sonar.{Domain}
  - ArgoCD: http://argo.{Domain}

- Jenkins 환경설정: Admin이 수행  
  - 플러그인 설치: Kubernetes, Pipeline Utility Steps, Docker Pipeline, GitHub, Slack Notification, Blue Ocean, SonarQube Scanner, Matrix Authorization Strategy  
  - Kubernetes 연결 설정: Jenkins관리 > Clouds에 k8s cluster 연결 설정   
  - Jenkins Agent 포트 설정: Jenkins관리 > Security에서 'Agent'섹션의 TCP Port를 50000번으로 고정. 이 포트는 파이프라인 수행을 하는 파드에서 Jenkins서버를 호출할 때 사용  
  - Credential 작성  
    - SonarQube Access: Secret text 유형 'sonarqube_access_token'라는 이름으로 SonarQube 접근 인증 토큰 작성   
    - 파일 서버 Access:  SSH Username with private key 유형 'jenkins-nfs-ssh'라는 이름으로 NFS서버 접근 인증 토큰 작성  
  - SonarQube 서버 정보 설정: Jenkins관리 > System의 'SonarQube servers'섹션에서 작성  
    - Name: SonarQube
    - Server URL: http://sonar-sonarqube.sonarqube.svc.cluster.local
    - Server authentication token: sonarqube_access_token 
  - 글로벌 환경변수 작성: Jenkins관리 > System의 'Global properties > Environment variables'를 체크하고 작성  
    - GRADLE_CACHE_DIR: gradle
    - NFS_CREDENTIAL: jenkins-nfs-ssh
    - NFS_DIR: data
    - NFS_HOST: NFS서버 IP나 호스트. 예) 10.110.50.111
    - SONAR_SERVER_ID: SonarQube
    - TRIVY_CACHE_DIR: trivy_cache
  - 공유 라이브러리 설정: Jenkins 관리 > System을 클릭하고 'Global Trusted Pipeline Libraries' 섹션으로 이동하여 작성  
    - Name: pipeline_shared_library
    - Default version: main
    - Git Project Repository: https://github.com/cna-bootcamp/pipeline_shared_library.git
  
- SonarQube 환경 설정: https://happycloud-lee.tistory.com/49 참조    
  - Webhook 등록: Administration > Configuration에서 Webhooks선택하여 등록  
  - Quality Gate 작성: Quality Fates 메뉴에서 'Subride way' 등록 

- 사용자 준비 작업
  - Jenkins 가입: jenkins 접근하여 회원 등록  
  - Jenkins Credential 작성   
    - GitHub Access: 'Username with password'유형으로 'github_access_token_{userid}'라는 이름으로 작성. 암호는 Access Token을 넣어야 함   
    - Image Access: 'Username with password'유형으로 'image_reg_credential_{userid}'라는 이름으로 작성. 암호는 로그인 암호를 넣으면 됨  
  - Image Pull secret 작성: 본인 namespace에 'dockerhub'라는 이름으로 작성. 없으면 아래 예제 참조하여 작성  
    ```
    kubectl create secret docker-registry dockerhub \
    --docker-server=docker.io \
    --docker-username=hiondal \
    --docker-password=11111 -n ott
    ``` 


## 프론트엔드 애플리케이션
- 로컬 작업 디렉토리로 이동  
  ```
  cd ~/home/workspace
  ```
- subride-front의 Git repo를 로컬에 Clone 또는 Pull하고 vscode에서 오픈    
  기존에 local git repo가 없으면 clone하고 이동  
  ```
  git clone {본인 Git Repo}
  cd subride-front
  ```
  기존에 이미 local git repo가 있으면 이동 후 pull
  ```
  cd subride-front
  git pull
  ```


  만약, 기존 작성된 코드가 없다면 아래 예제 코드를 클론하고 본인 Git Repository를 작성하여 푸시  
  - Github에 'subride-front'라는 이름으로 Git repo 작성  
  - 로컬에서 sample 클론
    ```
    git clone https://github.com/hiondal/subride-front.git
    cd subride-front
    ```
  - 본인 Git repo로 푸시  
    ```
    git checkout -B main
    git remote set-url origin {본인 Git 주소}
    echo " " >> README.md
    git add . && git commit -m "to my git" && git push -u origin main
    ```
- SonarQube에 본인 프로젝트 작성   
  - 로그인: ID는 본인 계정(예:user15)이고 암호는 '{본인계정}P@ssw0rd$'임. 암호예) user15P@ssw0rd$  
  - Projects 누르고 [Create local project] 클릭하여 생성
    - Project display name: subride-front-{본인계정}  
    - Project key:  subride-front-{본인계정}  
    - Main branch name: main  
    - 'Next'누르고 다음 페이지에서 'Use the global setting' 선택  
  
- SonarQube 정보 파일 작성: 최상위 디렉토리에 'sonar-project.properties'파일 작성  
  ```
  sonar.projectKey=subride-front-user15
  sonar.sources=src
  sonar.tests=src
  sonar.test.inclusions=src/**/*.test.js,src/**/*.test.jsx
  sonar.javascript.lcov.reportPaths=coverage/lcov.info
  ```

- Image build 파일 확인
  - buildfile 디렉토리 하위에 'Dockerfile_react_express'와 'server.js' 있는지 확인  
  - 없으면, https://github.com/hiondal/subride-front.git 의 내용 확인하여 작성  

- CI/CD 파이프라인과 관련 파일 작성 
  - 최상위 디렉토리 하위에 deployment 디렉토리 추가  
  - 하위에 Jenkinsfile, deploy.yaml.template, deploy_env_vars 파일 작성  
  - 파일 내용은 https://github.com/hiondal/subride-front.git 의 내용으로 일단 작성하고 저장  
  - deploy_env_vars의 아래 항목을 본인것으로 변경  
    'MANIFEST_REPO'는 아직 작성하지 않았지만 일단 본인 Organization으로 변경  
    ```
    IMAGE_REG_CREDENTIAL=image_reg_credential_user15
    API_GATEWAY_FQDN=http://user15.scg.msa.edutdc.com
    USE_ARGOCD=false
    GIT_ACCESS_CREDENTIAL=github_access_token_user15
    MANIFEST_REPO=github.com/hiondal/subrecommend-manifest.git
    NAMESPACE=user15
    SERVICE_ACCOUNT=sa-user15
    IMAGE_REG_ORG=hiondal
    EUREKA_URL=http://user15.eureka.msa.edutdc.com/eureka/apps/subrecommend-service
    INGRESS_HOST=user15.subride-front.msa.edutdc.com  
    ```
  - 파일을 모두 저장하고, 터미널을 vscode에서 열고 Git push 함   
    ```
    git add . && git commit -m "add cicd" && git push -u origin main
    ```
- Jenkins 파이프라인 작성  
  - Jenkins에 본인 ID/PW로 로그인  
  - 좌측에서 '새로운 Item' 클릭  
  - 이름을 'subride-front-{본인계정}'으로 하고, item type은 'Pipeline'으로 선택
  - 파이프라인 구성  
    - Enable project-based security: 본인만 파이프라인 보게 하는 권한 설정   
      - 체크 하고 'Do not inherit permission grants from other ACLs' 선택
      - 'Add user'누르고, 본인계정 추가. 본인 계정 맨 오론쪽에 '모두체크' 아이콘 클릭하여 모든 권한 부여  
    - Build Triggers: GitHub hook trigger for GITScm polling' 체크. Git 소스 변경 시 자동으로 파이프라인 수행하는 옵션임  
    - Pipeline: 아래 항목만 변경하고 나머지는 기본값 사용  
      - Definition: Pipeline script from SCM
      - SCM : Git
      - Repository URL: 본인의 Subride-front Git repo 주소
      - Credentials: 본인의 git 접근 credential 선택 
      - Branch Specifier: */main
      - Script Path: ./deployment/Jenkinsfile    
  - 저장 후 좌측메뉴에서 '지금 빌드' 클릭  
  - 좌측 메뉴에서 '블루오션 열기'를 클릭하여 진행상황 모니터링  

- Git repo에 Webhook 추가
  - github에서 subride-front repo 열기
  - 'Settings'선택 후 좌측에서 'Webhooks' 클릭
  - 'Add webhook' 클릭
    - Payload URL: http://jenkins.msa.edutdc.com/github-webhook/
    - Content type: application/json
    - SSL verfification: Disable
  - 테스트: 'Edit'버튼 클릭하고, 'Recent Deliveries'탭에서 맨 앞에 체크 아이콘 되어 있는지 확인  

> 참고: 파이프라인의 공통 수행은 [pipeline_shared_library](https://github.com/cna-bootcamp/pipeline_shared_library.git)의  'CommonFunctions.groovy'파일을 참고하십시오.  
 
## 백엔드 애플리케이션
- 로컬 작업 디렉토리로 이동  
  ```
  cd ~/home/workspace
  ```
- subrecommend의 Git repo를 로컬에 Clone 또는 Pull하고 IntelliJ에서 오픈    
  기존에 local git repo가 없으면 clone하고 이동  
  ```
  git clone {본인 Git Repo}
  cd subrecommend
  ```
  기존에 이미 local git repo가 있으면 이동 후 pull
  ```
  cd subrecommend
  git pull
  ```


  만약, 기존 작성된 코드가 없다면 아래 예제 코드를 클론하고 본인 Git Repository를 작성하여 푸시  
  - Github에 'subrecommend'라는 이름으로 Git repo 작성  
  - 로컬에서 sample 클론
    ```
    git clone https://github.com/hiondal/subrecommend.git
    cd subrecommend
    ```
  - 본인 Git repo로 푸시  
    ```
    git checkout -B main
    git remote set-url origin {본인 Git 주소}
    echo " " >> README.md
    git add . && git commit -m "to my git" && git push -u origin main
    ```
- SonarQube에 본인 프로젝트 작성   
  - 로그인: ID는 본인 계정(예:user15)이고 암호는 'P@ssw0rd$'임  
  - Projects 누르고 [Create local project] 클릭하여 생성
    - Project display name: subrecommend-{본인계정}  
    - Project key:  subrecommend-{본인계정}  
    - Main branch name: main  
    - 'Next'누르고 다음 페이지에서 'Use the global setting' 선택  
  
- SonarQube 정보 파일 작성: 최상위 디렉토리에 'sonar-project.properties'파일 작성  
  최상위 build.gradle에 SonarQube 설정
  ```
  plugin 'org.sonarqube' 추가

  plugins {
    id 'java'
    id 'org.springframework.boot' version '3.2.6'
    id "org.sonarqube" version "5.0.0.4638" apply false		//apply false 해야 서브 프로젝트에 제대로 적용됨
  }
  ```
  subprojects에 'sonarqube' 항목 추가
  ```
  subprojects {
	apply plugin: 'org.springframework.boot'
	apply plugin: 'org.sonarqube'
    ...
  }
  ```

- Image build 파일 확인
  - buildfile 디렉토리 하위에 'Dockerfile_java' 있는지 확인  
  - 없으면, https://github.com/hiondal/subrecommend.git 의 내용 확인하여 작성  

- CI/CD 파이프라인과 관련 파일 작성 
  - 최상위 디렉토리 하위에 deployment 디렉토리 추가  
  - 하위에 Jenkinsfile, deploy.yaml.template, deploy_env_vars 파일 작성  
  - 파일 내용은 https://github.com/hiondal/subrecommend.git 의 내용으로 일단 작성하고 저장  
  - deploy_env_vars의 아래 항목을 본인것으로 변경  
    'MANIFEST_REPO'는 아직 작성하지 않았지만 일단 본인 Organization으로 변경  
    ```
    IMAGE_REG_CREDENTIAL=image_reg_credential_user15
    SONAR_PROJECT_KEY=subrecommend-user15
    USE_ARGOCD=false
    GIT_ACCESS_CREDENTIAL=github_access_token_user15
    MANIFEST_REPO=github.com/hiondal/subrecommend-manifest.git
    NAMESPACE=user15
    SERVICE_ACCOUNT=sa-user15
    IMAGE_REG_ORG=hiondal
    FRONT_HOST=http://user15.subride-front.msa.edutdc.com
    INGRESS_HOST=user15.subride-front.msa.edutdc.com  
    ```
    > Tip: DB_URL은 값을 큰따옴표로 감싸야 함. 값에 '='이 있기 때무임  

  - 파일을 모두 저장하고, 터미널을 IntelliJ에서 열고 Git push 함   
    ```
    git add . && git commit -m "add cicd" && git push -u origin main
    ```
- Jenkins 파이프라인 작성  
  - Jenkins에 본인 ID/PW로 로그인  
  - 좌측에서 '새로운 Item' 클릭  
  - 이름을 'subrecommend-{본인계정}'으로 하고, item type은 'Pipeline'으로 선택
  - 파이프라인 구성  
    - Enable project-based security: 본인만 파이프라인 보게 하는 권한 설정   
      - 체크 하고 'Do not inherit permission grants from other ACLs' 선택
      - 'Add user'누르고, 본인계정 추가. 본인 계정 맨 오론쪽에 '모두체크' 아이콘 클릭하여 모든 권한 부여  
    - Build Triggers: GitHub hook trigger for GITScm polling' 체크. Git 소스 변경 시 자동으로 파이프라인 수행하는 옵션임  
    - Pipeline: 아래 항목만 변경하고 나머지는 기본값 사용  
      - Definition: Pipeline script from SCM
      - SCM : Git
      - Repository URL: 본인의 subrecommend Git repo 주소
      - Credentials: 본인의 git 접근 credential 선택 
      - Branch Specifier: */main
      - Script Path: ./deployment/Jenkinsfile    
  - 저장 후 좌측메뉴에서 '지금 빌드' 클릭  
  - 좌측 메뉴에서 '블루오션 열기'를 클릭하여 진행상황 모니터링  

- Git repo에 Webhook 추가
  - github에서 subrecommend repo 열기
  - 'Settings'선택 후 좌측에서 'Webhooks' 클릭
  - 'Add webhook' 클릭
    - Payload URL: http://jenkins.msa.edutdc.com/github-webhook/
    - Content type: application/json
    - SSL verfification: Disable
  - 테스트: 'Edit'버튼 클릭하고, 'Recent Deliveries'탭에서 맨 앞에 체크 아이콘 되어 있는지 확인  

## ArgoCD 연동  
지금까지는 deploy_env_vars에서 USE_ARGOCD를 'false'로 하였기 때문에 Jenkins에서 Kubectl로 바로 배포하였습니다.  
이제 ArgoCD를 연동하여 배포를 해보겠습니다.   

- manifest repo 작성: ArgoCD가 동기화하는 manifest가 있는 Git repo   
  sample을 이용하여 manifest repo를 작성합니다.  
  
  - 로컬에서 sample 클론
    ```
    cd ~/home/workspace
    git clone https://github.com/hiondal/subrecommend-manifest.git
    cd subrecommend-manifest
    ```
  - vscdoe에서 열어 config.yaml, eureka.yaml, scg.yaml에서 'user15'를 본인계정으로 변경  
  - image는 'hiondal'에 있는 기존 이미지를 사용하므로 변경 불필요. 본인 이미지가 있다면 변경해도 됨  
  - 본인 Git repo로 푸시  
    ```
    git checkout -B main
    git remote set-url origin {본인 Git 주소}
    echo " " >> README.md
    git add . && git commit -m "to my git" && git push -u origin main
    ```  
- ArgoCD에 본인 프로젝트와 Application 작성  
  - ArgoCD에 로그인: admin / P@ssw0rd$  
  - Project 작성: 'Settings > Projects' 선택하고, 'New Project' 클릭  
    - Project Name: 본인 계정. 예) user15
    - SOURCE REPOSITORIES: 본인 manifest repo 추가. 예) https://github.com/hiondal/subrecommend-manifest.git
    - DESTINATIONS: 
      - Server: https://kubernetes.default.svc
      - Name: in-cluster
      - Namespace: 본인 k8s namespace. 예) user15
  - Application 작성: 좌측메뉴에서 'Application' 선택하고, 'New App' 클릭  
    - Application Name: subrecommend-{본인계정}   
    - Project Name: 본인계정
    - Sync Policy: Automatic
    - SOURCE > Repository URL: 본인 manifest repo 주소  
    - SOURCE > Path: manifest
    - DESTINATION > Cluster URL: https://kubernetes.default.svc
    - DESTINATION > Namespace: 본인 k8s namespace 

- 프론트엔드 CI/CD
  - 로컬에서 subride-front의 deployment/deploy_env_vars에서 USE_ARGOCD의 값을 'true'로 변경
  - Git add, commit, push한 후 자동으로 파이프라인 시작 확인  
  - 파이프라인 정상 종료 후 manifest repo의 manifest디렉토리에 subride-front.yaml이 생성되었는지 확인  
  - ArgoCD의 Application 눌러서 배포 상태 확인
  
- 백엔드 CI/CD 
  - 로컬에서 subrecommend의 deployment/deploy_env_vars에서 USE_ARGOCD의 값을 'true'로 변경
  - Git add, commit, push한 후 자동으로 파이프라인 시작 확인  
  - 파이프라인 정상 종료 후 manifest repo의 manifest디렉토리에 subrecommend.yaml이 생성되었는지 확인  
  - ArgoCD의 Application 눌러서 배포 상태 확인

- 최종 확인
  - 웹브라우저에서 subride-front의 ingress host로 접근  
  - 모든 기능이 정상 동작하는 지 확인  
  
         
 
