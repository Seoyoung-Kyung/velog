<h3 id="0-들어가며">0. 들어가며</h3>
<p>바나프레소를 벤치마킹한 카페 주문 플랫폼 프로젝트의 배포 과정을 기록해보려고 한다.</p>
<p>그동안 백엔드 개발 위주로 프로젝트를 진행해왔지만 
배포는 직접 경험해본 적이 없어서 늘 막연하게 느껴졌었다.</p>
<p>이번 프로젝트에서는 직접 서버를 설정하고 배포를 진행해보면서<br />막연하게만 느껴졌던 배포 과정을 하나씩 경험하고 이해해보고자 했다.
<br />
<br /></p>
<h3 id="1-배포-구조">1. 배포 구조</h3>
<p>배포 구조는 아래 아키텍처를 기반으로 구성하였다.
<img alt="" src="https://velog.velcdn.com/images/se0o_129/post/886f784c-d989-4229-84f5-327df02d22aa/image.png" /></p>
<p>VPC 내에 퍼블릭 서브넷과 프라이빗 서브넷을 분리하였다. </p>
<p>퍼블릭 서브넷에는 Docker Compose로 Nginx와 Spring Boot 애플리케이션을 구성하였고, 
프라이빗 서브넷에는 RDS(MySQL)를 배치하여 데이터베이스에 외부에서 직접 접근할 수 없도록 하였다.</p>
<p>Nginx는 리버스 프록시 역할을 담당하며, Certbot으로 발급한 SSL 인증서를 적용하여 HTTPS를 처리한다. </p>
<p>이미지 파일은 EC2 로컬 파일 시스템 대신 S3에 저장하여 서버와 스토리지를 분리하였다.</p>
<p>배포는 main 브랜치로 push하면 
GitHub Actions가 자동으로 Docker 이미지를 빌드하고 
EC2에 배포하는 CI/CD 파이프라인으로 구성하였다.
<br />
<br /></p>
<h3 id="2-핵심-설계-결정">2. 핵심 설계 결정</h3>
<h4 id="vpc-설계-퍼블릭프라이빗-서브넷-분리">VPC 설계 (퍼블릭/프라이빗 서브넷 분리)</h4>
<p>우선 AWS가 기본으로 제공하는 Default VPC는 모든 서브넷이 퍼블릭이라 별도의 VPC를 직접 생성했다.</p>
<p>외부 요청을 받아야 하는 Nginx와 Spring Boot는 퍼블릭 서브넷에, 
외부에 노출될 필요가 없는 RDS는 프라이빗 서브넷에 배치했다. </p>
<p>각 컴포넌트의 역할에 맞게 네트워크 접근 범위를 분리하는 것이 목적이었다.<img alt="" src="https://velog.velcdn.com/images/se0o_129/post/4683dada-7b8c-4456-a69f-b864b54beb40/image.png" /></p>
<br />

<hr />
<h4 id="mysql-컨테이너-대신-rds-분리한-이유">MySQL 컨테이너 대신 RDS 분리한 이유</h4>
<p>컨테이너로 MySQL을 운영하는 방식 대신 RDS를 선택한 이유는 보안 때문이다.</p>
<p>DB에는 사용자 정보, 결제 정보 등 외부에 노출되어서는 안 되는 데이터가 저장된다. </p>
<p>MySQL을 EC2와 같은 퍼블릭 서브넷에서 컨테이너로 관리한다면
보안 그룹 설정 실수 하나로 DB가 외부에 노출될 위험이 있다.</p>
<p>RDS를 프라이빗 서브넷에 배치하면 
보안그룹 설정과 무관하게 네트워크 구조 자체가 외부 접근을 차단한다. </p>
<p>EC2에서만 접근 가능하도록 구성함으로써 이중으로 보안을 강화하였다.</p>
<p><img alt="" src="https://velog.velcdn.com/images/se0o_129/post/4d02e41c-5770-4e4a-b4ac-8f2f2eecafa3/image.png" /><img alt="" src="https://velog.velcdn.com/images/se0o_129/post/8d07ea42-e1f4-420b-bd94-57410b10c33d/image.png" /><img alt="" src="https://velog.velcdn.com/images/se0o_129/post/1a189093-610f-4683-a45d-116845a69be3/image.png" /></p>
<br />

<hr />
<h4 id="iam-role-방식으로-s3-접근한-이유">IAM Role 방식으로 S3 접근한 이유</h4>
<p>Spring에서 S3에 접근할 때 AWS 자격증명을 제공하는 방식은 크게 두 가지다.</p>
<p><strong>Access Key</strong> 방식은 IAM 사용자를 만들고 키를 발급해서 코드나 환경변수에 직접 넣는 방식이다. 
키를 직접 관리해야 하기 때문에 GitHub에 실수로 올릴 위험이 있고, 
주기적으로 교체해야 하는 부담도 있다.</p>
<p>반면에 <strong>IAM Role</strong> 방식은 EC2 자체에 역할을 부여해서 별도의 키 발급 없이 자동으로 인증하는 방식이다. 
키 자체가 존재하지 않아 노출 위험이 없고 AWS에서도 권장하는 방식이다.</p>
<p>키 관리 부담을 없애고 보안 사고 가능성을 구조적으로 차단하기 위해 IAM Role 방식을 선택하게 되었다.</p>
<p><img src="https://velog.velcdn.com/images/se0o_129/post/b9556556-b375-4ce7-ae4f-993a0b3c9ba9/image.png" width="50%" /><img alt="" src="https://velog.velcdn.com/images/se0o_129/post/0ceca32d-42e2-4068-b580-ebeffbf9f7a7/image.png" /></p>
<br />

<hr />
<h4 id="https-적용-방식-선택-이유-certbot-vs-acm">HTTPS 적용 방식 선택 이유 (Certbot vs ACM)</h4>
<p>HTTPS 적용 방식을 찾아보니 크게 세 가지가 있었다.</p>
<p><strong>ACM + ALB 방식</strong>은 AWS에서 인증서 관리와 자동 갱신까지 해주는 실무에서 가장 많이 쓰는 방식이라고 한다. </p>
<p>하지만 ALB 비용이 매월 $15~20 추가로 발생하고, 
ACM 인증서는 ALB 같은 AWS 관리형 서비스에만 붙일 수 있다는 제약이 있어서 지금 구성에는 맞지 않다고 판단했다.</p>
<p><strong>Nginx 컨테이너 안에서 Certbot을 실행하는 방식</strong>은 이식성이 좋지만 컨테이너 간 의존성과 볼륨 설정이 복잡하고 인증서 갱신 자동화도 까다로워 제외했다.</p>
<pre><code>EC2 서버
└── Docker Compose
    ├── Nginx 컨테이너
    ├── Certbot 컨테이너 ← 인증서 발급/갱신 담당
    └── Spring 컨테이너
</code></pre><p>결국 <strong>EC2에 Certbot을 직접 설치하는 방식</strong>을 선택했다. 
인증서를 EC2에 저장하고 Nginx 컨테이너에 볼륨으로 마운트하면 되는 구조라 가장 단순했다.</p>
<pre><code>EC2 서버
├── Certbot 설치됨
├── 인증서 저장: /etc/letsencrypt/  ← EC2에 직접 있음
└── Docker Compose
    ├── Nginx 컨테이너 ← 볼륨으로 EC2 인증서 읽어옴
    └── Spring 컨테이너</code></pre><p>EC2를 교체할 때 재발급이 필요하다는 단점이 있지만 지금 구성에서는 감수할 수 있다고 생각했다.<img alt="" src="https://velog.velcdn.com/images/se0o_129/post/2dee0dba-55c8-4fe1-9850-7a5c2ac72f6c/image.png" /><img alt="" src="https://velog.velcdn.com/images/se0o_129/post/1b78212b-082d-434a-8d47-f1b2d748b24c/image.png" /></p>
<br />

<hr />
<h4 id="github-actions-cicd-구성-방식">GitHub Actions CI/CD 구성 방식</h4>
<p>main 브랜치에 push가 발생하면 GitHub Actions가 자동으로 빌드부터 배포까지 실행된다.</p>
<p>전체 흐름은 다음과 같다.</p>
<p><strong>① 코드 빌드</strong>
GitHub 서버에서 코드를 체크아웃하고 Gradle로 jar 파일을 빌드한다. 
테스트는 -x test 옵션으로 제외했다.
<strong>② Docker 이미지 빌드 &amp; Docker Hub 푸시</strong>
빌드된 jar 파일로 Docker 이미지를 만들고 Docker Hub에 푸시한다.
<strong>③ EC2 배포</strong>
EC2에 SSH로 접속해서 기존 컨테이너를 내리고 새 이미지를 받아서 재시작한다.</p>
<p>민감한 정보(EC2 접속 키, Docker Hub 계정 등)는 GitHub Secrets에 등록해서 코드에 직접 노출되지 않도록 했다.</p>
<pre><code class="language-yml">name: CI/CD Pipeline

on:
  push:
    branches: [ main ]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: 코드 체크아웃
        uses: actions/checkout@v3

      - name: JDK 21 설정
        uses: actions/setup-java@v3
        with:
          java-version: '21'
          distribution: 'temurin'

      - name: Gradle 빌드
        run: |
          chmod +x gradlew
          ./gradlew build -x test

      - name: Docker Hub 로그인
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Docker 이미지 빌드 &amp; 푸시
        run: |
          docker build -t ${{ secrets.DOCKER_USERNAME }}/codepresso:latest .
          docker push ${{ secrets.DOCKER_USERNAME }}/codepresso:latest

      - name: EC2 배포
        uses: appleboy/ssh-action@v0.1.10
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USERNAME }}
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            cd ~/app
            docker-compose down
            docker rmi ${{ secrets.DOCKER_USERNAME }}/codepresso:latest || true
            docker-compose pull
            docker-compose up -d
            docker image prune -f</code></pre>
<br />
<br />

<h3 id="3-트러블슈팅">3. 트러블슈팅</h3>
<h4 id="①-aws-rds-연결-실패---security-group-인바운드-규칙-누락">① AWS RDS 연결 실패 - Security Group 인바운드 규칙 누락</h4>
<p>도커 컨테이너를 실행한 뒤 AWS RDS를 연결하는 과정에서 애플리케이션 실행이 실패하는 문제가 발생했다.</p>
<pre><code class="language-shell">2026-05-09T07:03:00.932Z ERROR 1 --- [codepresso] [           main] o.h.engine.jdbc.spi.SqlExceptionHelper   : Communications link failure
2026-05-09T07:03:00.940Z ERROR 1 --- [codepresso] [           main] j.LocalContainerEntityManagerFactoryBean : Failed to initialize JPA EntityManagerFactory: Unable to build Hibernate SessionFactory
Caused by: java.net.SocketTimeoutException: Connect timed out</code></pre>
<p>처음에는 RDS가 아직 시작 중이거나 DB 설정이 잘못된 것이라고 생각했다.
하지만 여러 설정을 다시 확인해도 해결되지 않아 AWS 설정을 하나씩 확인해보던 중 
RDS의 보안 그룹이 기존에 사용하던 다른 그룹으로 설정되어 있는 것을 발견했다.</p>
<p>EC2에서 RDS로 접근하지 못하고 있었고
인바운드 규칙을 올바르게 수정한 뒤 정상적으로 연결할 수 있었다.</p>
<p>배포를 처음 진행하다 보니 애플리케이션 설정만 계속 확인하고 있었는데
AWS의 보안 그룹과 같은 네트워크 설정도 함께 확인해야 한다는 점을 배울 수 있었다.
<img alt="" src="https://velog.velcdn.com/images/se0o_129/post/c5e6d1b3-b64d-407b-b512-573de572cebf/image.png" /></p>
<br />

<h4 id="②-nginxconf--docker-composeyml-수정-후-반영되지-않는-문제">② nginx.conf / docker-compose.yml 수정 후 반영되지 않는 문제</h4>
<p>nginx.conf와 docker-compose.yml을 수정하고 push했는데 EC2에 반영이 안 됐다.</p>
<p>확인해보니 CI/CD 파이프라인은 Docker 이미지만 교체하는 구조다. 
nginx.conf와 docker-compose.yml은 EC2에 고정된 파일이라 GitHub에 push해도 자동으로 반영되지 않는다.</p>
<p>그래서 scp 명령어로 수동 업로드하는 방식으로 해결했다.</p>
<p>사실 수동 업로드 방식은 설정 파일이 자주 바뀔 경우 불편하다. 
appleboy/scp-action으로 CI/CD에 자동화하는 방법도 있지만, 매 배포마다 변경되지 않는 파일을 복사하는 건 불필요한 작업이라고 생각했다.</p>
<p>현재 구조에서는 EC2에 고정하고 변경이 필요할 때만 수동 업로드하는 방식을 유지했다.
하지만 자주 변경된다고 생각되면 언제든 전환할 의향이 있다.</p>
<pre><code class="language-bash">scp -i &quot;~/Documents/deploy/codepresso-key.pem&quot; docker-compose.yml ubuntu@{EC2_IP}:~/app/
scp -i &quot;~/Documents/deploy/codepresso-key.pem&quot; nginx/nginx.conf ubuntu@{EC2_IP}:~/app/nginx/</code></pre>
<br />
<br />

<h3 id="4-한계-및-개선-방향">4. 한계 및 개선 방향</h3>
<p>최대한 실무적인 방식으로 배포하려고 했지만 
단일 서버 기준 구성이라 실제 서비스 규모로 확장 시 개선이 필요한 부분이 있다.
비용적인 측면도 무시할 수 없어서 현실적인 선택을 한 부분도 있었다.</p>
<p>HTTPS를 적용할 때 실제 서비스였다면 ACM + ALB 방안을 선택했을 것이다. 
이식성 측면에서도 관리 측면에서도 이 방식이 효율적이기 때문이다.</p>
<p>그리고 현재 배포 방식은 docker-compose down 후 docker-compose up으로 컨테이너를 교체하기 때문에 배포 시 짧은 다운타임이 발생한다. 
실제 서비스라면 Blue-Green 배포나 Rolling 배포 방식을 적용해 무중단 배포를 구성해야 한다. </p>
<p>단일 EC2 + Docker Compose 구성에서는 구현이 복잡해지고 추가 인프라 비용도 발생하기 때문에 아쉬운 부분이 있다.</p>
<p>현재는 단일 EC2 기반으로 구성했지만,
향후 관리자 기능이 추가되면 관리자 전용 EC2를 분리하여
사용자 API와 관리자 API를 독립적으로 운영해보고자 한다.
(++ 프론트 배포)</p>