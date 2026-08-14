<h3 id="0-시작하며">0. 시작하며</h3>
<p>세션 방식으로 구현되어 있던 로그인 방식을
JWT로 전환하고 Redis 적용하는 과정을 기록해보고자 한다.</p>
<h3 id="1-세션에서-jwt로-전환">1. 세션에서 JWT로 전환</h3>
<p><strong>세션</strong>은 브라우저에 세션 ID를 저장하고 서버 메모리에 사용자 정보를 저장하는 <strong>stateful</strong> 방식이다. 
이 구조는 두 가지 문제가 있다.</p>
<ol>
<li><p>서버를 재시작하면 메모리에 저장된 세션이 모두 사라져 전체 로그아웃이 발생한다. </p>
</li>
<li><p>서버가 여러 대로 확장될 경우 서버마다 세션이 따로 존재하기 때문에 로드밸런서가 다른 서버로 요청을 보내면 로그인 상태가 유지되지 않는다.</p>
</li>
</ol>
<p><strong>JWT</strong>는 서버가 아무것도 저장하지 않는 <strong>stateless</strong> 방식이다.
토큰 자체에 사용자 정보가 담겨 있어 어느 서버로 요청이 가든 토큰 검증만으로 인증이 가능하다. </p>
<p>그리고 서버가 상태를 저장하지 않기 때문에 유저가 늘어나도 서버 메모리 부담이 없다.</p>
<p>현재는 단일 서버 환경이지만 실제 서비스 운영을 고려했을 때 
확장성을 고려한 구조를 선택하고자 JWT로 전환하기로 했다.</p>
<br />
<br />

<h3 id="2-토큰-저장-전략">2. 토큰 저장 전략</h3>
<p>JWT는 <strong>Access Token</strong>과 <strong>Refresh Token</strong> 두 가지로 구성된다.</p>
<ul>
<li><strong>Access Token</strong> : API 요청 시 인증에 사용, 탈취 피해를 최소화하기 위해 만료 시간을 짧게 설정</li>
<li><strong>Refresh Token</strong> : Access Token 만료 시 재로그인 없이 재발급받기 위한 토큰, 만료 시간이 길어 탈취 시 피해가 크다</li>
</ul>
<p>짧은 토큰으로 보안을 챙기고 긴 토큰으로 편의성을 챙기는 역할 분리다.</p>
<hr />
<p>토큰 저장 위치는 두 가지 공격을 기준으로 결정했다.</p>
<ul>
<li><strong>XSS</strong> : 악성 스크립트로 JS에 접근해 토큰을 탈취하는 공격</li>
<li><strong>CSRF</strong> : 브라우저의 쿠키 자동 전송을 악용해 피해자 권한으로 요청을 위조하는 공격</li>
</ul>
<p><strong>Access Token → 클라이언트 메모리</strong>
localStorage는 XSS에 취약하고, 쿠키는 CSRF 위험이 있다. 
클라이언트 메모리도 XSS에 완전히 안전하진 않지만, 
XSS 방어는 토큰 저장 위치가 아닌 입력값 검증이나 이스케이프로 별도 레이어에서 처리해야 할 문제다. 
만약 쿠키에 저장한다면 피해자가 로그인 상태에서 악성 사이트에 접속했을 때, 
악성 사이트에 심어진 코드가 자동으로 서버로 요청을 전송하고,
브라우저가 쿠키를 자동으로 첨부하기 때문에 공격자가 피해자 권한으로 서버에 요청을 실행할 수 있다.
하지만 클라이언트 메모리에 저장하고 
Authorization 헤더로 직접 전송하면 브라우저 자동 전송이 없어 CSRF를 원천 차단할 수 있다는 점에서 이 방식을 선택했다.</p>
<p><strong>Refresh Token → HttpOnly 쿠키 + Redis</strong>
수명이 길어 탈취 시 피해가 크기 때문에 JS 접근을 차단하는 HttpOnly 쿠키에 저장하기로 했다.
다만 쿠키는 브라우저가 자동으로 요청에 첨부하기 때문에 CSRF 위험이 있어, 
SameSite=Lax 설정으로 외부 사이트에서의 요청을 차단했다.
하지만 쿠키는 브라우저가 들고 있어서 서버가 직접 삭제할 수 없다.
로그아웃해도 공격자가 복사해둔 토큰으로 재발급이 가능하다는 문제가 있기 때문에 Redis에 함께 저장하기로 했다.
재발급 요청 시 쿠키 값과 Redis 값을 비교하고, 
로그아웃 시 Redis에서 삭제해서 토큰을 즉시 무효화할 수 있도록 했다.
<br />
<br /></p>
<h3 id="3-로그인--토큰-발급--로그아웃-플로우">3. 로그인 / 토큰 발급 / 로그아웃 플로우</h3>
<p><img alt="" src="https://velog.velcdn.com/images/se0o_129/post/175143ac-8799-4cdb-bf04-6423198e4c57/image.png" /></p>
<br />
<br />

<h3 id="4-구현--핵심-코드-설명">4. 구현 : 핵심 코드 설명</h3>
<h4 id="1-jwtprovider-토큰-생성파싱">(1) JwtProvider (토큰 생성/파싱)</h4>
<p>액세스 토큰과 리프레시 토큰을 생성하는 코드이다.</p>
<p>두 토큰은 <strong>access/refresh</strong> 라는 타입명으로 구별하고 각각의 만료시간을 관리한다.</p>
<p>타입을 구별하는 이유는 Refresh Token으로 API를 호출하거나 
Access Token으로 재발급을 요청하는 것을 방지하기 위해서이다.
<img alt="" src="https://velog.velcdn.com/images/se0o_129/post/74b2f866-7490-4b33-92dd-ade853e9e322/image.png" /></p>
<br />

<h4 id="2-jwtauthenticationfilter-요청마다-검증">(2) JwtAuthenticationFilter (요청마다 검증)</h4>
<p>모든 API 요청이 들어올 때마다 Access Token을 검증하는 필터이다.</p>
<p>우선 Redis 블랙리스트를 확인하여 로그아웃된 토큰인지 확인하고, 
정상 토큰이면 CustomOAuth2User 객체에 사용자 정보를 담아 SecurityContext에 인증 정보를 저장한다.</p>
<p><img alt="" src="https://velog.velcdn.com/images/se0o_129/post/b73cb2bf-03f6-4716-b2fc-f04b59b7a296/image.png" /></p>
<br />

<h4 id="3-refreshtokenrepository-redis-crud">(3) RefreshTokenRepository (Redis CRUD)</h4>
<p>Redis에 Refresh Token을 저장하고 삭제하는 로직이다.</p>
<p>만료시간은 14일로 설정했다. 
독서기록 플랫폼 특성상 기록을 남기거나 모임이 있을 때만 접속하는 서비스라 매일 사용하지 않는 경우가 많다. </p>
<p>그래서 만료시간을 길게 잡아도 무방하다고 생각했고, 금융 서비스처럼 민감한 데이터를 다루지 않기 때문에 14일이 적절하다고 판단했다.</p>
<p>그리고 HttpOnly 쿠키만으로는 서버가 토큰을 직접 무효화할 수 없다. 
쿠키는 클라이언트가 들고 있기 때문에 로그아웃을 해도 공격자가 복사해둔 토큰은 여전히 유효하기 때문이다. 
이를 해결하기 위해 Redis에 함께 저장해 로그아웃 시 삭제함으로써 복사된 토큰도 재발급 요청에서 거부할 수 있도록 했다.
<img alt="" src="https://velog.velcdn.com/images/se0o_129/post/d359cd46-61d0-46ea-a449-34a9abaeac8e/image.png" /><img alt="" src="https://velog.velcdn.com/images/se0o_129/post/b6c940ca-1423-486b-803e-a127ac2a00e7/image.png" /><img alt="" src="https://velog.velcdn.com/images/se0o_129/post/e7388947-b1ba-488c-85a5-eb6b7aad6b43/image.png" /></p>
<br />

<h4 id="4-authservice-토큰-재발급-및-블랙-리스트-등록">(4) AuthService (토큰 재발급 및 블랙 리스트 등록)</h4>
<p>Access Token이 만료되었을 때 Refresh Token을 통해 재발급하고, 
로그아웃 시 해당 Access Token을 블랙리스트에 등록하여 만료 전 탈취된 토큰이 사용되는 것을 방지한다.</p>
<p>Access Token은 stateless 특성상 서버가 직접 무효화할 수 없다. 
그래서 로그아웃 시점에 남은 유효시간만큼 Redis에 블랙리스트로 등록하고, 이후 요청에서 해당 토큰이 감지되면 거부하는 방식으로 구현했다.</p>
<p>재발급 시에는 Refresh Token을 교체하는 Rotation 방식도 고려했지만, 독서 플랫폼 특성상 보안 민감도가 높지 않아 적용하지 않았다. Refresh Token은 HttpOnly 쿠키로 관리되어 탈취 가능성이 낮고, 재발급 요청 시 Redis에 저장된 토큰과 일치 여부를 검증하므로 유효하지 않은 토큰은 차단된다.</p>
<p><img alt="" src="https://velog.velcdn.com/images/se0o_129/post/7e768410-d342-41ca-a3dd-9526e442c764/image.png" /><img alt="" src="https://velog.velcdn.com/images/se0o_129/post/271ee5b6-db75-49e7-a2c6-0c24d08fb82f/image.png" /></p>
<br />
<br />

<h3 id="5-stateless에서-oauth2-state-문제와-해결">5. STATELESS에서 OAuth2 state 문제와 해결</h3>
<p>JWT 기반 stateless 환경에서는 세션이 없기 때문에 
OAuth2 로그인 시 생성되는 state 값을 저장할 공간이 없다. </p>
<p>스프링 시큐리티는 기본적으로 세션에 저장하는데 
세션을 사용하지 않으면 콜백 시점에 state 값을 비교할 수 없어서 CSRF 방어가 불가능해진다.</p>
<p>이를 해결하기 위해 <code>CookieOAuth2AuthorizationRequestRepository</code>를 구현해 state 값을 HttpOnly 쿠키에 저장했다. </p>
<p>로그인 시작할 때 저장하고, 카카오 콜백이 오면 쿠키에서 꺼내 비교한 뒤 즉시 삭제한다. 
TTL은 3분으로 설정해 로그인 완료 전 만료되지 않도록 했다.
<img alt="" src="https://velog.velcdn.com/images/se0o_129/post/b27a08be-531f-4fa2-9390-45a877a19aa7/image.png" /><img alt="" src="https://velog.velcdn.com/images/se0o_129/post/caf904ff-615d-47ec-a800-77b00b84a8db/image.png" /><img alt="" src="https://velog.velcdn.com/images/se0o_129/post/a7a22d12-d371-430c-979e-6c3204714d99/image.png" /></p>
<br />
<br />

<h3 id="6-마무리">6. 마무리</h3>
<p>JWT + OAuth2 + Redis를 조합해 stateless 환경에서 보안을 챙기는 인증 구조를 구현했다. </p>
<p>독서 모임/기록 플랫폼은 금융 서비스처럼 즉각적인 금전 피해가 발생하는 서비스는 아니지만, 
개인의 독서 기록과 감상이 담긴 플랫폼인 만큼 개인정보 보호 측면에서 보안을 소홀히 할 수 없다고 생각한다.</p>