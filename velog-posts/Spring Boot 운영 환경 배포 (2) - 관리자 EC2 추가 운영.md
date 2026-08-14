<h2 id="0-들어가며">0. 들어가며</h2>
<p>사실 처음 개발할 때는 사용자 API 구현에 급급했다.
하지만 실제 서비스 관점에서 생각해보면
사용자 서비스를 원활하게 유지하기 위해서는 관리자 기능도 그만큼 중요하다. </p>
<p>주문 처리, 상품 관리, 회원 관리 등 서비스 운영 전반을 담당하는 관리자 기능이 없다면 사용자 서비스도 제대로 돌아갈 수 없기 때문이다.</p>
<p>그래서 사용자 서비스만 구현되어 있던 것에서 관리자 기능을 추가로 구현하게 되었고, 자연스럽게 관리자 배포 구성까지 고려하게 되었다.</p>
<p>이번 포스팅에서는 관리자 배포 구성과 설계 의도, 그리고 그 과정을 담아보고자 한다.</p>
<br />
<br />

<h2 id="1-분리-배포를-선택한-이유">1. 분리 배포를 선택한 이유</h2>
<p>관리자와 사용자 기능을 하나의 EC2에 배포하는 것이 아니라 분리해서 배포하기로 했다. </p>
<p>내가 분리 배포를 선택한 이유는 아래와 같다.</p>
<p><strong>(1) 사용자 트래픽이 몰려도 관리자 기능 영향 없도록</strong></p>
<p>카페 주문 특성상 출근 시간 피크 타임에 주문이 한꺼번에 몰릴 수 있다. </p>
<p>이때 사용자 서버에 부하가 걸려 오류가 발생한다면, 관리자가 빠르게 대응해야 하는 상황임에도 불구하고 같은 서버를 사용하고 있다면 관리자 기능까지 함께 영향을 받게 된다.</p>
<p>이러한 상황을 막기 위해 관리자와 사용자 서버를 분리해서
사용자 서버에 부하가 발생하더라도 관리자 기능은 독립적으로 동작하기 위함이다.</p>
<br />

<p><strong>(2) 보안 분리 (관리자 엔드포인트 독립 관리)</strong></p>
<p>관리자 API와 사용자 API를 같은 서버에서 관리하게 되면
사용자가 URL 패턴을 유추해 관리자 엔드포인트에 접근을 시도할 가능성이 있다. </p>
<p>서버를 분리하면 이러한 접근 자체를 완전히 차단할 수 있고
추가로 보안 그룹에서 특정 IP만 허용하도록 설정할 수 있다. </p>
<p>JWT로 사용자와 관리자 권한을 검증해서 1차로 방어하고
서버 분리와 보안 그룹 설정으로 2차 방어까지 구성할 수 있다.</p>
<br />

<h2 id="2-인프라-구성">2. 인프라 구성</h2>
<p><img alt="" src="https://velog.velcdn.com/images/se0o_129/post/1edcac77-2aaa-4037-9e65-806d8c93338f/image.png" /></p>
<p>같은 VPC 안에서 사용자 EC2와 관리자 EC2를 각각 운영하는 구조로 구성하였다. </p>
<p>관리자 도메인은 Route 53 서브도메인을 활용해서 <code>admin.codepresso.site</code>로 분리했고</p>
<p>RDS는 프라이빗 서브넷에 배치하여 두 서버가 공유하되 보안 그룹으로 접근을 제어하도록 구성했다.</p>
<p><img alt="" src="https://velog.velcdn.com/images/se0o_129/post/bb9039c2-c472-46d5-8b50-49d1c65b0bdf/image.png" /></p>
<br />
<br />

<h2 id="3-cicd-분리">3. CI/CD 분리</h2>
<p>모노레포 구조이지만 GitHub Actions의 paths 필터를 활용해 user-api와 admin-api를 독립적으로 배포할 수 있도록 구성했다. 
<img src="https://velog.velcdn.com/images/se0o_129/post/d8528efd-7cbf-44b4-a343-2bb6a6661774/image.png" width="50%" /></p>
<p><code>user-api/**</code> 변경 시에는 user-api workflow만, <code>admin-api/**</code> 변경 시에는 admin-api workflow만 트리거된다. 
core/** 변경 시에는 공통 모듈이므로 양쪽 workflow가 모두 트리거된다.</p>
<p>그리고 브랜치별로 역할을 분리해 dev 브랜치 push 시에는 CI(빌드)만 수행하고, main 브랜치 push 시에는 CD(빌드 + 배포)까지 수행하도록 구성했다.</p>
<p><img src="https://velog.velcdn.com/images/se0o_129/post/92c4537f-b00c-4103-88f0-e6a350832d02/image.png" width="40%" /><img src="https://velog.velcdn.com/images/se0o_129/post/8319f9c7-1aad-4715-8582-37c588c10c0b/image.png" width="40%" /><img alt="" src="https://velog.velcdn.com/images/se0o_129/post/2158fe6e-c39a-4cb9-8772-ce6bfa14041a/image.png" /></p>
<br />
<br />

<h2 id="4-트러블슈팅">4. 트러블슈팅</h2>
<h4 id="1-vpc-불일치로-rds-접근-불가">(1) VPC 불일치로 RDS 접근 불가</h4>
<p>처음 admin-api EC2를 생성할 때 VPC 설정을 따로 하지 않아서
기본 VPC로 생성되었다. 
user-api EC2와 RDS는 codepresso-vpc에 있었는데 admin-api EC2는 다른 VPC에 생성된 것이다. 
VPC가 다르면 같은 AWS 계정이라도 내부 통신이 불가능하기 때문에 RDS에 접근할 수 없었다. 
EC2를 삭제하고 codepresso-vpc를 직접 선택해서 재생성하여 해결했다. ㅠㅠ
<img alt="" src="https://velog.velcdn.com/images/se0o_129/post/23d990cf-bd0b-423f-abf7-922948f38ab1/image.png" /></p>
<h4 id="2-mysql-유저-권한-문제">(2) MySQL 유저 권한 문제</h4>
<p>VPC 문제를 해결하고 나서도 DB 연결이 안 됐다. 
로그를 확인해보니 Access denied for user 'codepresso'@'{IP}' 에러가 발생했다. 
RDS 보안 그룹은 열려있었지만 MySQL 유저 자체에 새 EC2 IP 접근 권한이 없었던 것이었다. 
user-api EC2에서 MySQL에 접속해 아래 명령어로 유저를 생성하고 권한을 부여해 해결했다.</p>
<pre><code class="language-sql">CREATE USER 'codepresso'@'%' IDENTIFIED BY 'DBPASSWORD';
GRANT ALL PRIVILEGES ON codepresso.* TO 'codepresso'@'%';
FLUSH PRIVILEGES;</code></pre>
<br />
<br />

<h2 id="5-배운점">5. 배운점</h2>
<p>처음에 관리자 배포를 고려했을 때 그냥 사용자랑 함께 배포하면 되는 거 아닌가? 라고 단순하게만 생각했다. 
하지만 실제 서비스 측면에서 고려해보니 그 선택이 얼마나 위험할 수 있는지 깨달았다. </p>
<p>백엔드 기능 구현에만 집중하다 보면 놓치기 쉬운 부분이지만
이번 계기로 서비스 관점에서 한 번 더 생각하는 것의 중요성과 배포 구조에 대해 깊이 생각해볼 수 있는 계기가 되었다.</p>