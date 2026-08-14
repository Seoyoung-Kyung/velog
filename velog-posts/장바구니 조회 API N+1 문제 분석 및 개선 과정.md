<h2 id="0-들어가며">0. 들어가며</h2>
<p>카페 주문 플랫폼 프로젝트를 리팩토링하던 중
장바구니 조회 페이지에서 예상보다 많은 쿼리가 실행되고 있는 것을 로그를 통해 발견했다.</p>
<p>특히 장바구니에 음료를 하나씩 추가할수록
조회 시 실행되는 쿼리 수가 함께 증가하는 현상이 나타났다.</p>
<p>이러한 패턴을 보며 성능 병목이 발생할 수 있는 API라고 생각했고, 
문제의 원인을 명확히 확인해보기로 했다.</p>
<p>이번 포스팅에서는
N+1 문제를 분석하고 개선한 과정까지 작성해보려 한다.</p>
<br />

<hr />
<h2 id="1-문제-상황-및-분석">1. 문제 상황 및 분석</h2>
<h3 id="11-문제-상황-확인">1.1 문제 상황 확인</h3>
<p>장바구니 조회 API를 기준으로
랜덤으로 각 상품에 3개의 옵션을 선택한 뒤 
상품을 하나씩 장바구니에 담아가며 API를 호출해보았다.</p>
<p>그 결과 상품을 하나 추가할 때마다 
쿼리 로그가 상품 수가 증가함에 따라 쿼리 수 역시 함께 증가하는 것을 발견했다.</p>
<p>이를 통해서 연관 관계 조회 과정에서 불필요한 쿼리가 반복적으로 발생하고 있을 가능성을 의심하게 되었다.</p>
<p>먼저 대시보드를 통해 전반적인 흐름을 파악한 뒤,
테이블 구조를 정리하고 테스트 코드를 통해 쿼리 수를 정확히 확인해보겠다.</p>
<br />

<h3 id="12-장바구니-도메인-구조">1.2 장바구니 도메인 구조</h3>
<p><strong>현재 ERD 구조</strong>
<img src="https://velog.velcdn.com/images/se0o_129/post/480c9510-a24b-48c6-aedd-7a7d5469274f/image.png" width="80%" /></p>
<pre><code>Cart
 └─ CartItem
     ├─ Product
     └─ CartOption
         └─ ProductOption
             └─ OptionStyle</code></pre><ul>
<li><p><strong>Cart / CartItem</strong></p>
<ul>
<li><strong>Cart</strong>는 회원당 하나의 장바구니를 가지며, <strong>CartItem</strong>은 장바구니에 담긴 상품 단위이다.</li>
</ul>
</li>
<li><p><strong>CartOption</strong></p>
<ul>
<li><strong>CartOption</strong>은 <strong>CartItem</strong>에 대해 사용자가 실제로 선택한 옵션을 저장하는 테이블이다.</li>
<li>하나의 상품에는 여러 옵션이 선택될 수 있으며, 이로 인해 CartItem ↔ CartOption은 1:N 관계를 가진다. </li>
</ul>
</li>
<li><p><strong>ProductOption / OptionStyle</strong></p>
<ul>
<li>옵션 정보는 <strong>ProductOption과</strong> <strong>OptionStyle로</strong> 분리되어 있다.</li>
<li><strong>ProductOption</strong> : 상품에 어떤 옵션이 존재하는지</li>
<li><strong>OptionStyle</strong> : 옵션명과 추가 가격 등 실제 표시/계산에 필요한 정</li>
</ul>
</li>
</ul>
<p>이러한 구조에서는 CartItem 하나를 조회하더라도
선택된 옵션 수에 따라 연관 엔티티 조회가 반복적으로 발생할 가능성이 있다.</p>
<br />

<h3 id="13-문제가-발생하는-service-코드">1.3 문제가 발생하는 Service 코드</h3>
<p>N+1 문제가 발생하는 API의 서비스 코드를 살펴보자.</p>
<p>우선 멤버 아이디를 기준으로 장바구니 엔티티를 조회하면서
Fetch Join을 통해 장바구니 상품과 상품 엔티티까지 함께 조인하고 있다.<img alt="" src="https://velog.velcdn.com/images/se0o_129/post/93235ba9-7f2a-4917-bdf2-0a1cbfda5ed0/image.png" /><img alt="" src="https://velog.velcdn.com/images/se0o_129/post/a5740572-286e-45f9-96ad-9587dd8e0c71/image.png" /></p>
<p>하지만 상품 조회 페이지에서는 상품 정보뿐만 아니라, 
해당 상품에 대한 옵션 정보도 필요함에도 불구하고
상품 옵션 엔티티에 대해서는 Fetch Join을 사용하지 않고 있었다.</p>
<p><img alt="" src="https://velog.velcdn.com/images/se0o_129/post/f8dde388-e99e-42b0-9758-caab6e5ddbbc/image.png" /></p>
<p><code>Fetch join</code>으로 가져오지 않은 <code>cartOption</code>, <code>optionStyle</code>, <code>member</code> 엔티티의 데이터를 메서드를 통해 연관관계를 따라 체이닝 방식으로 접근하면서,
각 상품마다 그에 해당하는 데이터를 가져오기 위해서 추가 쿼리가 발생하고 있었다.</p>
<p>결과적으로 상품과 선택한 옵션 수만큼 추가 쿼리가 실행되면서 N+1 문제가 발생하고 있음을 확인할 수 있었다.</p>
<br />

<h3 id="14-n1-문제-탐지-결과">1.4 N+1 문제 탐지 결과</h3>
<p>테스트 코드 기반으로 정확한 수치를 확인해보겠다.</p>
<p><strong>테스트 환경</strong></p>
<ul>
<li>데이터베이스 : MySQL (HikariCP 커넥션 풀)</li>
<li>테스트 데이터 : 옵션 3개 이상인 상품 10개 선택, 각 상품당 3개 옵션 선택 (총 30개)</li>
<li>JPA 1차 캐시 : 매 측정마다 <code>EntityManager.clear()</code>로 초기화하여 캐시 영향 제거</li>
</ul>
<p><strong>측정 방법:</strong></p>
<ul>
<li>워밍업 : 3회 실행 (JVM 최적화 및 DB 준비)</li>
<li>실제 측정 : 10회 반복 후 평균값 산출</li>
<li>수집 지표 : 쿼리 실행 횟수, 응답 시간(평균/최대/P95), 테이블별 쿼리 분포</li>
</ul>
<p>테스트 데이터로 옵션이 3개 이상 등록된 상품 10개를 조회하여 사용하였다.
<img alt="" src="https://velog.velcdn.com/images/se0o_129/post/48d94ac1-a4a3-4c19-bb45-2893968e8261/image.png" /></p>
<p>내부적으로는 실행 시간, 쿼리 수을 수집하였고
응답시간은 평균, 최대, P95 기준으로 비교하였다.<img alt="" src="https://velog.velcdn.com/images/se0o_129/post/9b27e9ba-8d81-47cd-a227-76ca86c5b3c9/image.png" /><img src="https://velog.velcdn.com/images/se0o_129/post/4024e743-8333-4cce-8d9a-588b41f7d54a/image.png" width="50%" /> </p>
<p>P95 기준 응답 시간은 약 <strong>475ms</strong> 정도였고,
동일 조건에서 더 다양한 옵션을 선택할 경우
<code>option_style</code>에 대한 쿼리 수가 증가하면서 응답 시간 또한 함께 증가할 것으로 예상된다.</p>
<p>옵션 3개씩 적용된 상품 10개가 담긴 장바구니를 조회한 결과
총 <strong>46개</strong>의 쿼리가 발생한 것을 확인할 수 있었다.</p>
<ul>
<li><strong>product_option</strong> : 3 × 10 = 30회</li>
<li><strong>nutrition_info</strong> : 10회</li>
<li><strong>option_style</strong> : 4회</li>
<li><strong>cart_option, cart</strong> : 각각 1회</li>
</ul>
<br />

<p><strong>그런데 여기서 추가로 발견한 문제점이 있다.</strong></p>
<p><code>product</code>와 1:1 연관관계를 맺고 있는 <code>nutrition_info</code>(영양 정보) 엔티티에서 10개의 추가 쿼리가 발생하고 있었다.
<img src="https://velog.velcdn.com/images/se0o_129/post/9999090b-0ba2-4b41-a587-2a33c75e9448/image.png" width="60%" /> <img src="https://velog.velcdn.com/images/se0o_129/post/782f93a2-5e82-4cc4-9e3b-0f6f3be8fcda/image.png" width="60%" /> </p>
<p>처음에는 단순한 로딩 전략 문제라고 생각했다.
하지만 원인을 찾아보니 <code>@OneToOne</code> 연관관계에서 FK의 위치로 인해 발생한 문제였다.</p>
<p><code>nutrition_info</code>는 product를 참조하고 있으며,
FK는 <code>nutrition_info</code> 테이블에 존재한다.</p>
<p>장바구니 조회에서는 <code>nutrition_info</code> 데이터를 전혀 사용하지 않음에도 불구하고,
<code>product</code> 엔티티를 로딩하는 과정에서
<code>nutrition_info</code>에 대한 추가 조회 쿼리가 발생하고 있었다.</p>
<p>이는 FK가 존재하지 않는 쪽(product)에서
@OneToOne(fetch = LAZY) 연관관계를 사용할 경우 발생하는 JPA의 특성 때문이다.</p>
<p>JPA는 프록시를 생성하기 위해 연관 엔티티의 식별자(ID)를 알아야 하지만,
이 경우 <code>product</code> 엔티티만으로는 <code>nutrition_info</code>의 ID를 알 수 없다.
따라서 <code>nutrition_info</code>가 실제로 존재하는지,
혹은 null인지 판단하기 위해 즉시 조회 쿼리를 실행할 수밖에 없다.</p>
<p>그 결과, <code>fetch = LAZY</code>로 설정했음에도
실제로는 즉시 로딩처럼 동작하게 되며,
이 조회가 상품 수만큼 반복되면서 N+1 문제가 발생하게 된다.</p>
<p>이는 단순한 Fetch 전략 변경으로 해결할 수 있는 문제가 아니라
연관관계 매핑 설계 자체에서 발생한 문제라고 볼 수 있다.</p>
<br />

<hr />
<h2 id="2-n1-문제-해결-전략">2. N+1 문제 해결 전략</h2>
<h3 id="21-대표적인-세-가지-전략">2.1 대표적인 세 가지 전략</h3>
<p>N+1 문제를 해결하는 대표적인 세 가지 전략을 간단하게 살펴보자.</p>
<br />

<p><strong>Fetch Join</strong></p>
<p>패치 조인을 사용할 경우,
연관관계에 해당하는 엔티티를 하나의 쿼리로 한 번에 조회할 수 있다.
이렇게 조회된 엔티티는 이후 접근 시에도 추가 쿼리가 발생하지 않는다.</p>
<p>목록 조회에서 항상 함께 사용하는 연관 엔티티가 명확하고,
데이터 양이 많지 않을 때 가장 확실한 해결 방법이다.</p>
<p><strong>@BatchSize</strong></p>
<p>하나의 쿼리 수행 시,
IN 절을 사용해 여러 연관 엔티티를 묶어서 조회하도록 설정하는 어노테이션이다.
지정한 배치 사이즈만큼 데이터를 한 번에 가져오며,
이를 초과할 경우 다음 쿼리를 통해 동일한 방식으로 조회한다.</p>
<p>Fetch Join을 사용하기 어렵거나,
여러 연관 엔티티가 선택적으로 사용되는 경우에 유용하다.</p>
<p><strong>@EntityGraph</strong></p>
<p>조회 시 함께 가져올 연관 엔티티를 어노테이션으로 명시하여 
필요한 연관 데이터만 즉시 로딩하도록 설정할 수 있다.</p>
<p>JPQL을 수정하지 않고,
메서드 단위로 Fetch 전략을 제어하고 싶을 때 사용한다.</p>
<br />

<h3 id="22-나의-해결-방안">2.2 나의 해결 방안</h3>
<p>우선 하나의 쿼리로 가져오던 데이터를 두 개의 쿼리로 나눠서 가져오기로 했다.</p>
<p>장바구니 조회에서 실제로 필요한 데이터는 <code>Cart</code>가 아니라 <code>CartItem</code>이었고,
<code>Cart</code>를 기준으로 <code>fetch join</code> 을 해서
DISTINCT를 통해 이를 다시 하나의 Cart 엔티티로 합치는 과정이 불필요하다고 생각했다.</p>
<p>그래서 <code>Cart</code>는 식별자 조회로 분리하고,
<code>CartItem</code>을 루트로 필요한 연관 데이터만 <code>fetch join</code> 하는 두 단계 조회 방식을 선택했다.</p>
<p><img alt="" src="https://velog.velcdn.com/images/se0o_129/post/b7e0dfca-bf3e-4fcd-a8d5-4c12e229cae1/image.png" /></p>
<br />

<p>*<em>1. 조회 방식 선택 : Fetch join vs Projection *</em></p>
<p>장바구니 조회 API에서 N+1 문제를 해결하기 위해
<code>Fetch Join</code>과 <code>Projection</code> 중 어떤 방식을 사용할지 고민했다.</p>
<p>먼저 두 방식의 특징을 간단히 정리해보았다.</p>
<br />

<p><strong>Projection</strong></p>
<ul>
<li>필요한 컬럼만 조회하므로 가져오는 데이터 양이 줄어든다.</li>
<li>엔티티를 생성하거나 영속성 컨텍스트에 등록하지 않기 때문에 
엔티티 로딩 및 관리 비용이 발생하지 않는다.</li>
<li>이러한 특성으로 인해 대량 조회 시 메모리 및 CPU 사용량을 줄이는 데 효과적이다.</li>
</ul>
<p><strong>Fetch Join</strong></p>
<ul>
<li>연관된 엔티티를 한 번의 쿼리로 함께 조회할 수 있어 N+1 문제를 방지할 수 있다.</li>
<li>엔티티를 그대로 조회하므로
연관 관계 탐색, 옵션 조합, 금액 계산 등 도메인 로직을 자연스럽게 처리할 수 있다.</li>
<li>컬렉션 구조가 유지된 상태로 로딩되기 때문에
<code>Projection</code>처럼 결과를 다시 그룹핑하거나 가공할 필요가 없다.</li>
</ul>
<br />

<p>두 방식을 비교해보았을 때,</p>
<p>장바구니 조회는 한 사용자 기준의 데이터로 규모가 크지 않고,
옵션과 같은 중첩된 연관 구조를 그대로 활용해야 하는 특성을 가지고 있었다.</p>
<p>이러한 구조를 <code>Projection</code>으로 조회할 경우
서비스 계층에서 결과를 다시 그룹핑하고 가공하는 추가 로직이 필요해져
오히려 코드 복잡도가 증가할 수 있다고 생각했다.</p>
<p>그리고 성능적 측면에서도 대량 데이터 조회에 효과 큰 <code>Projection</code>의 이점을 장바구니 조회 에선 효과가 크지 않을 것 같았다.</p>
<p>반면 <code>Fetch Join</code>을 사용하면
연관 데이터를 한 번에 조회하면서도 엔티티 구조를 그대로 활용할 수 있어
도메인 로직을 단순하게 유지하고, 유지보수성 측면에서도 유리하다고 생각했다.</p>
<p>따라서 이번 장바구니 조회 API에서는
<code>Projection</code> 이 아닌 <code>Fetch Join</code> 방식을 선택했다.</p>
<br />

<p><strong>2. Fetch join 전략 활용</strong></p>
<p><code>Fetch Join</code> 을 활용하여 장바구니 조회에 필요한 연관 데이터를 한 번의 쿼리로 가져오도록 수정하였다.</p>
<p><img alt="" src="https://velog.velcdn.com/images/se0o_129/post/9a491bf6-1a93-4f0c-ab54-63fb4fbbf118/image.png" /></p>
<p>장바구니 도메인에는 다음과 같은 다양한 상태가 존재한다.</p>
<ul>
<li>장바구니는 존재하지만 상품이 없는 경우</li>
<li>상품은 존재하지만 옵션이 선택되지 않은 경우</li>
<li>옵션 조합이 부분적으로만 선택된 상태</li>
</ul>
<p>이러한 특성상 <code>INNER JOIN FETCH</code>를 사용할 경우,
연관 데이터가 존재하지 않는 시점에서는
장바구니 또는 장바구니 아이템 자체가 조회되지 않는 문제가 발생할 수 있다고 생각했다.</p>
<p>그래서 연관 데이터의 존재 여부와 관계없이 
장바구니 정보를 안정적으로 조회하기 위해
모든 연관 관계를 <code>LEFT JOIN FETCH</code>로 조회하는 방식을 선택하였다.</p>
<br />

<p>*<em>3. 비효율적인 OneToOne 연관 관계 설계 개선 *</em></p>
<p>두 번째로 문제였던 부분은
실제로는 필요하지 않은 <code>nutritionInfo</code> 데이터가 조회되고 있다는 점이었다.</p>
<p>일반적으로 1:1 연관관계를 양방향으로 매핑하는 경우는
해당 데이터가 여러 곳에서 사용되거나, 추후 확장 가능성을 고려하는 경우가 많다.</p>
<p>하지만 현재 도메인 구조를 살펴보면 영양정보(<code>nutritionInfo</code>)를 조회하는 API는
상품 상세 조회 API 하나뿐이었다.</p>
<p>이 상태를 그대로 유지할 경우
N+1 문제를 해결하기 위해 장바구니 조회 API에서는 사용하지도 않는 영양정보까지
<code>fetch join</code>으로 함께 조회해야 하는 상황이 발생한다.</p>
<p>실제로 다른 곳에서 영양정보를 조회하는 경우가 없었기 때문에
양방향 연관관계를 제거하고 단방향 구조로 수정하였다.</p>
<p>결과적으로 
상품 상세 조회 API에서는 영양정보를 별도의 쿼리로 조회하도록 분리했고</p>
<p>장바구니 조회 API에서는 영양정보를 <code>fetch join</code> 하지 않고도
필요한 데이터만 조회할 수 있도록 구조를 개선할 수 있었다.
<img src="https://velog.velcdn.com/images/se0o_129/post/b0816337-37af-4253-8f3c-dc2cd0434471/image.png" width="60%" /></p>
<br />

<hr />
<h2 id="3-개선-결과">3. 개선 결과</h2>
<p>개선한 장바구니 조회 API(<code>/users/cart</code>)를 기준으로,
개선 전과 동일한 조건으로 상품을 장바구니에 담아 API를 호출해보았다.</p>
<p>그 결과,
상품 개수와 상관없이 총 2개의 쿼리만 수행되는 것을 확인할 수 있었다.
<img alt="" src="https://velog.velcdn.com/images/se0o_129/post/78e7b586-5d64-4c10-bb01-fa9471c5aa77/image.jpg" /></p>
<pre><code class="language-sql">Hibernate: 
    select
        c1_0.cart_id 
    from
        cart c1_0 
    where
        c1_0.member_id=?
Hibernate: 
    select
        distinct ci1_0.cart_item_id,
        ci1_0.cart_id,
        co1_0.cart_item_id,
        co1_0.cart_option_id,
        co1_0.product_option_id,
        po1_0.product_option_id,
        os1_0.option_style_id,
        os1_0.extra_price,
        os1_0.option_name_id,
        os1_0.option_style,
        po1_0.product_id,
        ci1_0.price,
        ci1_0.product_id,
        p1_0.product_id,
        p1_0.category_id,
        p1_0.favorite_count,
        p1_0.price,
        p1_0.product_content,
        p1_0.product_name,
        p1_0.product_photo,
        p1_0.version,
        ci1_0.quantity 
    from
        cart_item ci1_0 
    left join
        product p1_0 
            on p1_0.product_id=ci1_0.product_id 
    left join
        cart_option co1_0 
            on ci1_0.cart_item_id=co1_0.cart_item_id 
    left join
        product_option po1_0 
            on po1_0.product_option_id=co1_0.product_option_id 
    left join
        option_style os1_0 
            on os1_0.option_style_id=po1_0.option_style_id 
    where
        ci1_0.cart_id=?</code></pre>
<img src="https://velog.velcdn.com/images/se0o_129/post/9cc6efb3-f8ac-4190-a008-b7d79e9ff9de/image.png" width="60%" />

<p>테스트 코드로 확인한 결과 역시 동일했다.
기존에는 상품 수에 따라 쿼리가 증가하던 구조였으나,
개선 이후에는 cart 조회 1회, cart_item 조회 1회로 
항상 총 <strong>2개</strong>의 쿼리만 실행되도록 개선되었다.</p>
<p>또한 성능 측면에서도 눈에 띄는 개선이 있었다.
P95 기준 응답 시간은 <strong>472ms → 42ms</strong>로  감소했다.</p>
<br />

<h3 id="개선-전-후-비교표">개선 전 후 비교표</h3>
<table>
<thead>
<tr>
<th align="left">지표</th>
<th align="center">개선 전 (N+1 발생)</th>
<th align="center">개선 후 (Fetch Join)</th>
<th align="center">개선율</th>
</tr>
</thead>
<tbody><tr>
<td align="left"><strong>총 쿼리 실행 횟수</strong></td>
<td align="center">46회</td>
<td align="center"><strong>2회</strong></td>
<td align="center"><strong>95.6% ↓</strong></td>
</tr>
<tr>
<td align="left"><strong>평균 응답 시간</strong></td>
<td align="center">200ms</td>
<td align="center"><strong>26ms</strong></td>
<td align="center"><strong>87.0% ↓</strong></td>
</tr>
<tr>
<td align="left"><strong>P95 응답 시간</strong></td>
<td align="center">472ms</td>
<td align="center"><strong>42ms</strong></td>
<td align="center"><strong>91.1% ↓</strong></td>
</tr>
</tbody></table>
<br />

<hr />
<h2 id="4-배운점">4. 배운점</h2>
<p><strong>모니터링만으로는 모든 문제를 발견할 수 없다</strong></p>
<p>Grafana 대시보드를 통해
요청 수가 늘어날수록 쿼리 수가 함께 증가하는 현상은 확인할 수 있었지만,
어떤 연관 관계에서, 어떤 코드 지점에서 쿼리가 발생하는지까지는
모니터링만으로 파악하기 어려웠다.</p>
<p>결국 실제 서비스 코드와 쿼리 로그,
테스트 코드를 함께 보면서 문제를 추적해야
N+1이 발생하는 정확한 원인을 이해할 수 있었다.</p>
<br />

<p><strong>N+1 문제는 단순한 로딩 전략 문제가 아닐 수 있다</strong></p>
<p>처음에는 fetch = LAZY / EAGER 설정만의 문제라고 생각했는데 분석해보니,
연관 관계를 어떻게 설계했는지
특히 @OneToOne 관계에서 FK의 위치와 양방향 매핑 여부에 따라서도
의도하지 않은 쿼리가 발생할 수 있다는 것을 알게 되었다.</p>
<p>즉, N+1 문제는
단순히 fetch 전략을 바꾸는 것으로 해결되지 않고,
엔티티 매핑 설계 자체를 다시 고민해야 하는 문제일 수도 있었다.</p>
<br />

<p>N+1 문제는 다양한 원인과 상황에서 발생하고
Fetch Join, @BatchSize, @EntityGraph 등 여러 해결 전략이 존재한다.
각 전략의 특성을 이해하고 상황에 맞게 선택하는 것이 중요함을 배웠다.</p>