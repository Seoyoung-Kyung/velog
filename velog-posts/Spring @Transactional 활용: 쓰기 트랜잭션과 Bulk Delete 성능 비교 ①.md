<h2 id="0-들어가며">0. 들어가며</h2>
<p>카페 주문 플랫폼 장바구니 비우기 기능을 테스트하던 중, 로그에서 이상한 점을 발견했다.</p>
<p>단 10개의 상품을 삭제하는데 DELETE 쿼리가 11번이나 실행되고 있었다.</p>
<p>처음에는 단순히 비효율적인 구현 때문이라고 생각했다.
하지만 원인을 하나씩 살펴보면서,
JPA에서의 삭제 방식과 트랜잭션 처리에 대해 다시 고민하게 되었다.</p>
<p>이 글에서는 원인 분석과 여러 삭제 방식(Bulk Delete, Cascade 등)을 비교하며
직접 측정해본 과정을 정리해보려고 한다.</p>
<p><img alt="" src="https://velog.velcdn.com/images/se0o_129/post/907b3752-8858-4851-9e56-614d212aa29d/image.png" /></p>
<br />
<br />


<h2 id="1-문제-케이스--트랜잭션-범위-내-반복-쿼리">1. 문제 케이스 : 트랜잭션 범위 내 반복 쿼리</h2>
<blockquote>
<p><strong>문제 상황 분석</strong></p>
</blockquote>
<h4 id="현재-erd-구조">현재 ERD 구조</h4>
<p><img alt="" src="https://velog.velcdn.com/images/se0o_129/post/b0097b8d-2033-4c5d-8322-3e507d0c5388/image.png" /></p>
<p>장바구니 관련 테이블은 다음과 같이 구성되어 있다:</p>
<ul>
<li><strong>cart</strong>: 회원별 장바구니 정보를 저장하는 테이블</li>
<li><strong>cart_item</strong>: 장바구니에 담긴 상품 정보를 저장하는 테이블</li>
<li><strong>cart_option</strong>: 각 상품의 옵션 정보를 저장하는 테이블</li>
</ul>
<hr />
<blockquote>
<h4 id="문제-코드">문제 코드</h4>
</blockquote>
<p><img alt="" src="https://velog.velcdn.com/images/se0o_129/post/dcc15582-ecfe-46b5-8d91-6a4c2cde5968/image.png" /></p>
<p>사용자가 장바구니 &quot;전체 비우기&quot; 버튼을 클릭했을 때 실행되는 메서드이다.</p>
<pre><code class="language-java">@Transactional
public void clearCart(Long memberId, Long cartId) {
    List&lt;CartItem&gt; items = cartItemRepository.findByCartId(cartId);

    for (CartItem ci : items) {
        cartOptionRepository.deleteByCartItemId(ci.getId()); 
        // 반복문 안에서 매번 개별 DELETE 실행
    }

    cartItemRepository.deleteAllInBatch(items);
}</code></pre>
<p><strong>문제점 :</strong> 
장바구니 상품의 옵션을 삭제할 때, 
반복문 안에서 각 CartItem에 대해 매번 개별 DELETE 쿼리가 실행되고 있다.</p>
<hr />
<blockquote>
<h4 id="실행된-쿼리-로그-분석">실행된 쿼리 로그 분석</h4>
</blockquote>
<pre><code class="language-sql">-- 1. 장바구니 조회
SELECT c1_0.cart_id, c1_0.member_id 
FROM cart c1_0 
WHERE c1_0.cart_id=?

-- 2. 장바구니 상품 조회
SELECT ci1_0.cart_item_id, ci1_0.cart_id, ci1_0.price, ci1_0.product_id, ci1_0.quantity 
FROM cart_item ci1_0 
JOIN cart c1_0 ON c1_0.cart_id=ci1_0.cart_id 
WHERE c1_0.cart_id=?

-- 3. 장바구니 옵션 개별 삭제 (10번 반복)
DELETE FROM cart_option WHERE cart_item_id=?
DELETE FROM cart_option WHERE cart_item_id=?
DELETE FROM cart_option WHERE cart_item_id=?
DELETE FROM cart_option WHERE cart_item_id=?
DELETE FROM cart_option WHERE cart_item_id=?
DELETE FROM cart_option WHERE cart_item_id=?
DELETE FROM cart_option WHERE cart_item_id=?
DELETE FROM cart_option WHERE cart_item_id=?
DELETE FROM cart_option WHERE cart_item_id=?
DELETE FROM cart_option WHERE cart_item_id=?

-- 4. 장바구니 상품 일괄 삭제
DELETE FROM cart_item 
WHERE cart_item_id IN (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)</code></pre>
<p><strong>쿼리 실행 결과 :</strong></p>
<ul>
<li>CartOption 개별 삭제: <strong>10번</strong></li>
<li>CartItem 일괄 삭제: <strong>1번</strong></li>
<li>총 DELETE 쿼리: <strong>11번</strong></li>
</ul>
<p>장바구니 상품이 10개일 때, 
DELETE 쿼리가 11번 실행되는 비효율이 발생했다.</p>
<hr />
<blockquote>
<h4 id="트랜잭션-관점의-문제">트랜잭션 관점의 문제</h4>
</blockquote>
<p><strong>불필요한 네트워크 왕복으로 인한 트랜잭션 보유 시간 과다</strong></p>
<ul>
<li>트랜잭션은 DB 커넥션을 점유하는데 트랜잭션 실행 시간이 길어지면 DB 커넥션 점유 시간이 길어진다.<ul>
<li>커넥션 풀의 크기가 제한적이므로, 트랜잭션이 길어지면 다른 트랜잭션들의 대기 시간도 길어지게 된다.</li>
<li>대용량 트래픽이 몰리는 상황에서 1번의 쿼리로 수행할 수 있는 작업을 11번의 쿼리로 수행하는 것은 매우 비효율적이다.</li>
</ul>
</li>
</ul>
<br />
<br />

<h2 id="2-해결-방법">2. 해결 방법</h2>
<h3 id="21-벌크-삭제-bulk-delete">2.1 벌크 삭제 (Bulk Delete)</h3>
<p>반복문으로 개별 DELETE를 수행할 때 발생하는 N+1 삭제 문제는 벌크 삭제를 통해 해결할 수 있다.</p>
<p>벌크 삭제는 여러 개의 데이터를 한 번의 쿼리로 한꺼번에 삭제하는 방식을 말한다.</p>
<p>영속성 컨텍스트를 거치는 단계를 모두 건너뛰고 데이터베이스에 직접 delete sql을 실행하기 때문에 빠르다.</p>
<p>벌크 삭제 방식에는 두 가지 방법이 있다.</p>
<hr />
<blockquote>
<p><strong>1. deleteAllInBatch() 메서드로 삭제 수행</strong></p>
</blockquote>
<p><code>deleteAllInBatch</code>는 Spring Data JPA가 기본 제공하는 메서드이다.</p>
<p>먼저 엔티티를 조회하고 엔티티 컬렉션을 받아서 IN절로 삭제를 수행한다. (SELECT + DELETE, 총 두 번의 쿼리)</p>
<pre><code class="language-sql">SELECT * FROM cart_item WHERE cart_id = ?;  -- 먼저 조회
DELETE FROM cart_item WHERE cart_item_id IN (?, ?, ...);  -- 그 다음 삭제</code></pre>
<hr />
<blockquote>
<p>** 2. Repository에 벌크 삭제 쿼리 추가 (@Query + @Modifying)**</p>
</blockquote>
<p><code>@Query + @Modifying</code> 방식은 <strong>조회 없이 바로 DELETE 쿼리를 실행</strong>하는 특징이 있다.</p>
<p>영속성 컨텍스트를 거치지 않고 <strong>SQL을 직접 실행</strong>하여, 엔티티 로딩 없고 메모리 사용 최소화할 수 있는 방식이다.</p>
<pre><code class="language-java">@Modifying
@Query(&quot;DELETE FROM CartOption co WHERE co.cartItem.cart.id = :cartId&quot;)
void deleteByCartId(@Param(&quot;cartId&quot;) Long cartId);</code></pre>
<p>deleteAllInBatch()와 비교했을 때 조회 없이 바로 삭제가 가능하다는 장점이 있지만, Repository에 쿼리를 직접 작성한 메서드를 추가해야 한다.</p>
<pre><code class="language-java">DELETE FROM cart_item WHERE cart_id = ?;  -- 바로 삭제 (조회 없음)</code></pre>
<br />
<br />

<h3 id="22-cascade-설정">2.2 Cascade 설정</h3>
<p>Cascade는 부모 엔티티의 작업이 자식의 엔티티에게 전파되는 기능을 말한다.</p>
<p>부모를 삭제하면 자식도 자동으로 삭제돼서, 외래키 제약 조건 문제를 자동으로 해결해준다.</p>
<pre><code class="language-java">@OneToMany(mappedBy = &quot;cartItem&quot;, cascade = CascadeType.ALL)  // 모든 작업 전파
private List&lt;CartOption&gt; options = new ArrayList&lt;&gt;();</code></pre>
<br />
<br />

<h2 id="3-성능-테스트">3. 성능 테스트</h2>
<h3 id="31-테스트-시나리오">3.1 테스트 시나리오</h3>
<p>한 명의 사용자가 3개의 옵션이 적용된 10개의 상품이 들어있는 장바구니를 비우는 상황을 가정했다.</p>
<p>개선 전 방식과 3가지 개선 방식(벌크 삭제 2가지, Cascade)을 비교하여 각각 <strong>10회씩 반복 측정</strong>한 후 평균 실행 시간을 비교했다.</p>
<br />

<h3 id="32-테스트-환경">3.2 테스트 환경</h3>
<ul>
<li><strong>테스트 사용자</strong> : memberId = 1</li>
<li><strong>상품 데이터</strong> : 기존 DB의 186개의 상품 중 옵션이 3개 이상인 상품 10개 선택</li>
<li><strong>장바구니 구성</strong><ul>
<li><strong>CartItem</strong> : 10개</li>
<li><strong>CartOption</strong> : 30개 (상품당 3개씩)
<img alt="" src="https://velog.velcdn.com/images/se0o_129/post/5c83aebc-70c1-46cb-97ad-569b0abc7655/image.png" /></li>
</ul>
</li>
<li>*<em>반복 측정 *</em>: 각 방식당 10회</li>
</ul>
<p><img alt="" src="https://velog.velcdn.com/images/se0o_129/post/e6843017-b78c-4e7e-b257-476cfc89a2fb/image.png" />각 측정마다 <code>entityManager.clear()</code>로 캐시를 초기화하여 정확한 성능을 측정했다.</p>
<br />
<br />

<hr />
<h3 id="33-개선-전--반복문--deleteallinbatch">3.3 개선 전 : 반복문 + deleteAllInBatch</h3>
<p><img alt="" src="https://velog.velcdn.com/images/se0o_129/post/215a0f2c-1a5c-4949-a952-3c53ab08cbdb/image.png" /><img alt="" src="https://velog.velcdn.com/images/se0o_129/post/64b78b66-df1c-4d0f-90cb-6bbb332bd65d/image.png" />반복문으로 CartOption을 개별 삭제한 후, CartItem을 일괄 삭제하는 기존 방식에 대한 메서드이다.</p>
<p>총 11번의 DELETE 쿼리 수행으로 매우 비효율적으로 장바구니를 비우고 있다.
<br /></p>
<hr />
<h3 id="34-벌크-삭제-적용--querymodifying">3.4 벌크 삭제 적용 : @Query+@Modifying</h3>
<p><img alt="" src="https://velog.velcdn.com/images/se0o_129/post/3104de72-b002-4df4-9b3e-67bed5f4811e/image.png" /><img alt="" src="https://velog.velcdn.com/images/se0o_129/post/756ee8d4-ab16-4cb2-99e3-acdde900e647/image.png" /><img alt="" src="https://velog.velcdn.com/images/se0o_129/post/97675b59-f9c2-4401-9fb4-c258c10848b8/image.png" /><img alt="" src="https://velog.velcdn.com/images/se0o_129/post/24125da0-f375-4423-bf8a-d8f29c6338cc/image.png" /></p>
<p><code>@Query+@Modifying</code> 방식은 JPQL로 직접 작성한 벌크 삭제 쿼리를 사용한다.</p>
<p>조회 없이 바로 벌크 삭제하고 쿼리 개수가 가장 적게 발생한 방식이다.</p>
<p>개선 전 대비 <strong>367ms</strong> 빠른 방식이라는 것을 알 수 있었다. *<em>(13.5% 향상)
*</em>
<br /></p>
<h3 id="35-벌크-삭제-적용--deleteallinbatch">3.5 벌크 삭제 적용 : deleteAllInBatch</h3>
<p><img alt="" src="https://velog.velcdn.com/images/se0o_129/post/b1d4b2aa-581a-4669-be5f-32d5ca135789/image.png" /><img alt="" src="https://velog.velcdn.com/images/se0o_129/post/98d93035-7e6f-4d95-bf41-52ea320c67c1/image.png" /><code>deleteAllInBatch</code>는 엔티티를 먼저 조회하고 조회한 엔티티들을 메모리에 로딩하여 일괄 삭제하는 방식이다.</p>
<p>CartOption과 CartItem 엔티티를 각각 한번씩 조회하여 두 번의 SELECT문과 두 번의 DELETE문이 발생하였다.</p>
<p>네가지 방식 중 가장 짧은 평균 실행시간을 보였고, 개선 전 대비 <strong>446ms</strong> 빠른 방식이었다. <strong>(16.8%)</strong></p>
<p>@Query + @Modifying 방식보다는 31ms 빠르게 측정되었다.</p>
<br />
<br />

<h3 id="36-cascade--cascadetypeall">3.6 cascade = CascadeType.ALL</h3>
<p><img alt="" src="https://velog.velcdn.com/images/se0o_129/post/703775aa-f3fe-44de-bcca-1c3e6ab73aae/image.png" /><img alt="" src="https://velog.velcdn.com/images/se0o_129/post/e2665840-cd2b-41b0-a5e4-025aa7aa07f3/image.png" /><img alt="" src="https://velog.velcdn.com/images/se0o_129/post/609c6277-274c-4fc7-86ef-589e1cbdc495/image.png" /></p>
<p>CartItem의 options를 <code>cascadeType.ALL</code>을 적용하여 JPA가 연관관계에 따라 자동 삭제하도록 하였다.</p>
<p>N+1 문제로 쿼리 많이 발생하지만, 그럼에도 개선 전과 비교했을 때 <strong>75ms</strong> 더 빨랐다는 걸 알 수 있었다. <strong>(2.8% 향상)</strong></p>
<br />
<br />


<h3 id="37-성능-테스트-종합-분석">3.7 성능 테스트 종합 분석</h3>
<table>
<thead>
<tr>
<th>순위</th>
<th>방식</th>
<th>평균 시간</th>
<th>쿼리 개수</th>
<th>개선율</th>
</tr>
</thead>
<tbody><tr>
<td>1</td>
<td>deleteAllInBatch</td>
<td>227ms</td>
<td>4번</td>
<td>16.8%</td>
</tr>
<tr>
<td>2</td>
<td>@Query + @Modifying</td>
<td>258ms</td>
<td>2번</td>
<td>13.5%</td>
</tr>
<tr>
<td>3</td>
<td>Cascade</td>
<td>298ms</td>
<td>40번+</td>
<td>2.8%</td>
</tr>
<tr>
<td>4</td>
<td>개선 전 (반복문)</td>
<td>303ms</td>
<td>11번</td>
<td>기준</td>
</tr>
</tbody></table>
<p>이론적으로 <code>@Query + @Modifying</code> 방식이 조회없이 바로 삭제하기 때문에 가장 빠를 것으로 예상했다.</p>
<p>하지만 예상과 달리 실제 테스트 결과는 <code>deleteAllInBatch</code> 방식이 가장 빠른 성능을 보였다.</p>
<br />

<blockquote>
<h4 id="왜-예상과-다를까-">왜 예상과 다를까 ?</h4>
</blockquote>
<p>내가 실행한 테스트에서는 소량 데이터(상품 10개, 옵션 30개)가 이러한 결과를 보인 가장 큰 요인으로 예상된다.</p>
<br />

<p><code>deleteAllInBatch</code> 는 먼저 엔티티를 조회한 후 삭제하는 2단계 과정을 거친다.</p>
<pre><code class="language-sql">List options = cartOptionRepository.findByCartId(cartId);
// 30개 엔티티 조회

DELETE FROM cart_option 
WHERE cart_option_id IN (1, 2, 3, 4, 5, ..., 30); // IN절로 삭제</code></pre>
<p>MySQL은 이 IN 절을 처리할 때 매우 효율적으로 동작한다. </p>
<ol>
<li>IN 절의 값들을 정렬</li>
<li>인덱스를 한 번만 순차 스캔하여 해당 값들을 찾음</li>
<li>실제 삭제</li>
</ol>
<p>이러한 과정을 거쳐 삭제 처리가 완료된다.</p>
<br />

<p>반면 <code>@Query + @Modifying</code> 방식은 조회 없이 바로 삭제하지만, 서브쿼리를 실행해야 한다.</p>
<pre><code class="language-sql">DELETE FROM cart_option 
WHERE cart_item_id IN (
    SELECT cart_item_id FROM cart_item WHERE cart_id = 1
);</code></pre>
<p>MySQL이 위 쿼리를 처리하는 방법은 다음과 같다.</p>
<ol>
<li><strong>서브쿼리 실행</strong></li>
</ol>
<pre><code class="language-sql">SELECT cart_item_id FROM cart_item WHERE cart_id = 1
→ 결과: [101, 102, 103, 104, 105, 106, 107, 108, 109, 110]


이 과정에서 아래와 같은 과정이 이루어진다.
- cart_item 테이블 스캔
- WHERE 조건 평가
- 결과 10개 추출
- 임시 메모리 영역에 저장</code></pre>
<ol start="2">
<li><strong>임시 테이블 생성</strong> 
MySQL은 서브쿼리 결과를 임시 테이블(derived table)에 저장하고, 메모리 또는 디스크에 임시 공간 할당한다.</li>
</ol>
<ol start="3">
<li><p><strong>메인 쿼리와 조인</strong>
cart_option 테이블과 임시 테이블을 조인한다.</p>
<pre><code>조인 조건 : cart_option.cart_item_id = 임시테이블.cart_item_id</code></pre></li>
<li><p><strong>삭제 실행 :</strong> 조인 결과로 매칭된 행들을 삭제한다.</p>
</li>
</ol>
<br />

<p>이처럼 <code>@Query + @Modifying</code> 방식은 서브쿼리 실행, 임시 테이블 생성, 조인 처리 등의 <strong>구조적 오버헤드</strong>가 발생한다.</p>
<p>이러한 오버헤드는 데이터 개수와 무관하게 항상 발생하는 준비 작업이다.</p>
<p>소량 데이터에서는 실제 데이터를 삭제하는 시간보다 이런 준비 작업 시간이 더 오래 걸려서 비효율적이다.</p>
<p><code>deleteAllInBatch</code>의 IN 절 방식은 임시 테이블 생성 X, 서브쿼리 실행 X, 단순 인덱스 스캔으로 빠른 처리하기 때문에</p>
<p>30개 정도의 소량 데이터에서는 조회 비용을 감안하더라도,
MySQL의 IN 절 최적화가 매우 효과적으로 작동하여 서브쿼리 방식보다 빠른 성능을 보인다.</p>
<br />

<blockquote>
<h4 id="테스트-결과의-한계">테스트 결과의 한계</h4>
</blockquote>
<p>이번 테스트는 <strong>소량 데이터(상품 10개 + 옵션 30개)</strong>로만 진행했다.</p>
<p><code>deleteAllInBatch</code>가 가장 빨랐지만, 대량 데이터에서는 결과가 달라질 수 있다.</p>
<p><code>deleteAllInBatch</code>는 데이터가 적을 땐 부담이 없지만 1000개가 되면 
1000개 객체를 메모리에 로딩하고, 긴 IN 절 (1000개) 파싱해야 하기 때문에 부담이 크다.</p>
<p>반면 <code>@Query + @Modifying</code> 는 조회 없이 서브쿼리로 바로 삭제하므로
데이터가 많아져도 안정적일 것으로 예상된다.</p>
<br />

<h2 id="4-배운점">4. 배운점</h2>
<p><strong>이론대로만 생각하면 놓치는 것들이 있다</strong></p>
<p>처음에는 @Query + @Modifying 방식이
쿼리를 한 번만 실행하니 당연히 가장 빠를 것이라고 생각했다.</p>
<p>하지만 실제로 테스트해보니
deleteAllInBatch가 더 빠른 결과가 나와 예상과 달라서 꽤 당황했다.</p>
<p>이 경험을 통해
이론에서 배운 내용만으로 성능을 판단하는 데에는 한계가 있다는 것을 느꼈다.
특히 데이터 개수나 실행 상황에 따라 결과가 달라질 수 있다는 점을 직접 확인할 수 있었다.</p>
<p>데이터가 많지 않은 경우에는 deleteAllInBatch 방식이 더 효율적으로 동작했고</p>
<p>데이터가 많아질 경우에는 다른 방식이 더 적합할 수도 있다는 가능성을 알게 되었다</p>
<p>앞으로는 “이 방식이 더 좋다”라고 단정하기보다,
지금 상황에서는 왜 이 방식이 맞는지 고민해봐야겠다고 느꼈다.</p>
<br />

<p><strong>성능은 생각이 아니라 직접 재봐야 알 수 있다</strong></p>
<p>기존 코드에서 DELETE 쿼리가 여러 번 실행되는 것을 보고
“쿼리가 많으니까 무조건 느릴 것 같다”라고 생각했다.</p>
<p>그래서 쿼리 수를 줄이는 데 집중했는데,
막상 측정해보니 쿼리 수가 더 많았던 Cascade 방식이
오히려 전체 실행 시간은 조금 더 빠른 결과를 보였다.</p>
<p>이 결과를 통해
쿼리 개수가 적다고 해서 항상 빠른 것은 아니라는 점을 배웠다.
각 쿼리가 어떤 방식으로 실행되는지,
그 과정에서 어떤 비용이 드는지도 함께 봐야 한다는 것을 알게 되었다.</p>
<p>앞으로는 추측으로 결론을 내리기보다,
반드시 로그와 측정 결과를 먼저 확인하는 습관을 들여야겠다고 느꼈다.</p>