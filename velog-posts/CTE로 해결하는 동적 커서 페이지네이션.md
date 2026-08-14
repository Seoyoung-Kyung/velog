<h2 id="0-들어가며">0. 들어가며</h2>
<p>독크독크 플랫폼에서 개선 포인트를 찾는 중에 
데이터 양이 증가할수록 성능 저하를 유발할 수 있는 비효율적인 로직을 발견했다.</p>
<p>특히 페이지네이션 처리 방식에서 개선할 지점을 포착하게 되었고, 
이에 대한 고민과 해결 과정을 기록해보고자 한다.
<br /></p>
<hr />
<h2 id="1-문제-상황-및-분석">1. 문제 상황 및 분석</h2>
<h3 id="11-페이지-특성">1.1 페이지 특성</h3>
<p>독크독크 플랫폼에서는 사용자가 읽은 책을 기록하고 관리할 수 있는 ‘<strong>내 책장</strong>’ 기능을 제공한다.</p>
<p>사용자가 직접 등록한 도서는 한 권 단위로 정리되어 표시되며, 
해당 페이지에서는 다양한 조건에 따라 목록을 조회할 수 있다.</p>
<ul>
<li>기록 상태별 조회: 기록 중 / 기록 완료</li>
<li>별점 기준 조회: 1점 ~ 5점</li>
<li>모임별 조회</li>
<li>정렬 기준 선택: 최신순 / 오래된 순</li>
</ul>
<p>이 기능을 통해 사용자는 자신의 독서 기록을 목적에 맞게 정리하고 효율적으로 관리할 수 있다.
<img alt="" src="https://velog.velcdn.com/images/se0o_129/post/9caa9e07-e08e-48fc-bea8-134fefcacb92/image.png" /></p>
<br />

<h3 id="12-코드로-살펴보기">1.2 코드로 살펴보기</h3>
<p>코드를 통해 흐름을 조금 더 정리해보자.</p>
<p>우선 내 책장 페이지의 Repository 계층을 보면
해당 쿼리에는 커서 페이지네이션에 필요한 ORDER BY, 커서 조건, LIMIT이 포함되어 있지 않다.</p>
<p>즉, DB 레벨에서는 단순 조회만 수행하고 있다.
<img alt="" src="https://velog.velcdn.com/images/se0o_129/post/075d4ef7-ae08-42b7-b124-2bff300ed599/image.png" /></p>
<p>반면 Service 계층을 살펴보면
정렬 기준(시간순/평점순 × 오름차순/내림차순)이 동적으로 변경된다는 이유로,</p>
<ul>
<li>정렬 처리</li>
<li>커서 조건 필터링</li>
<li>조회 개수 제한(LIMIT)</li>
</ul>
<p>이 모든 로직을 애플리케이션 레벨에서 처리하고 있다.
<img alt="" src="https://velog.velcdn.com/images/se0o_129/post/22c1cf15-d3b9-48ca-859f-2a81ebf8862c/image.png" /><img alt="" src="https://velog.velcdn.com/images/se0o_129/post/3dc6a6de-b97e-4e27-bf14-80c23ae76cf5/image.png" /><img alt="" src="https://velog.velcdn.com/images/se0o_129/post/2c66e16f-ec3d-42b4-9095-890ef30c1bfc/image.png" /></p>
<p>그 결과 실제로는 10건만 필요한 요청임에도 불구하고,
사용자가 보유한 책 전체 데이터를 DB에서 모두 조회한 뒤</p>
<ul>
<li>메모리에서 정렬하고</li>
<li>커서를 기준으로 잘라내고</li>
<li>필요한 개수만 반환하는 방식으로 동작하고 있다.</li>
</ul>
<p>결국 정렬이 동적이라는 이유로 페이지네이션을 애플리케이션 계층으로 올려버리면서,
DB가 가장 잘할 수 있는 작업(정렬 + 제한)을 활용하지 못하고 있는 구조라고 볼 수 있다.</p>
<p>이 부분은 동적 정렬을 SQL 레벨에서 처리하도록 개선하면,
불필요한 전체 조회 없이 필요한 데이터만 효율적으로 가져올 수 있다고 생각했다.</p>
<br />

<hr />
<h2 id="2-해결-과정">2. 해결 과정</h2>
<h3 id="21-애플리케이션-레벨-→-db-레벨">2.1 애플리케이션 레벨 → DB 레벨</h3>
<p>우선 정렬, 필터링, LIMIT과 같은 작업을 애플리케이션이 아니라 
데이터베이스 레벨에서 수행하도록 구조를 변경하는 것을 목표로 했다.</p>
<p>기존 쿼리는 <code>GROUP BY</code>를 통해 책 단위의 집계 결과를 생성하고 있었다.
이 과정에서 다음과 같은 집계 컬럼들이 만들어진다.</p>
<ul>
<li><code>rating (max(br.rating))</code></li>
<li><code>addedAt (max(pb.added_at))</code></li>
<li><code>bookReadingStatus (array_agg)</code></li>
<li><code>gatherings (json_agg)</code></li>
</ul>
<p>문제는 이러한 값들이 <strong><code>GROUP BY</code>로 그룹이 만들어진 뒤, 
집계 함수에 의해 계산되어 생성되는 컬럼</strong>이라는 점이다.</p>
<p>따라서 기존 구조에서는 이 값들을 <code>WHERE</code> 절에서 바로 활용할 수 없어,
페이지네이션이나 추가 필터링을 데이터베이스에서 처리하기 어려웠다.</p>
<p>그래서 집계 결과를 먼저 생성한 뒤,
그 결과를 기준으로 필터링과 페이지네이션을 수행하는 구조로 쿼리를 재구성하기로 했다.</p>
<p>이를 위해 <strong>CTE(Common Table Expression)</strong>를 사용하여 다음과 같이 쿼리를 분리했다.</p>
<ol>
<li><strong>CTE 단계</strong><ul>
<li>GROUP BY를 통해 책 단위의 집계 결과 생성</li>
</ul>
</li>
</ol>
<p><img alt="" src="https://velog.velcdn.com/images/se0o_129/post/18e91cc2-824c-4280-8c0f-425c066f3823/image.png" /></p>
<ol start="2">
<li><strong>외부 SELECT 단계</strong><ul>
<li>집계된 결과를 기준으로 rating 필터링</li>
<li>커서 기반 페이지네이션 적용</li>
<li>ORDER BY 및 LIMIT 처리</li>
</ul>
</li>
</ol>
<p><img alt="" src="https://velog.velcdn.com/images/se0o_129/post/f3fdd497-471e-4a36-9170-748e6b15277f/image.png" /></p>
<p>이렇게 구조를 분리함으로써
<strong>집계 → 필터링 → 정렬 → 페이지네이션</strong>의 흐름을 명확하게 만들 수 있었고,
애플리케이션 레벨이 아니라 DB 레벨에서 데이터 양을 줄일 수 있게 되었다.</p>
<br />


<h3 id="22-테스트-코드">2.2 테스트 코드</h3>
<p><strong>테스트 환경</strong></p>
<ul>
<li><strong>DB</strong> : PostgreSQL</li>
<li><strong>데이터 수</strong>: 200 books</li>
<li><strong>테스트 방식</strong>: warm-up 5회 + 측정 10회 평균</li>
</ul>
<p>우선 내 책장에 200권의 책을 등록한 사용자를 가정하여 테스트를 진행하였다.</p>
<p>초기 실행에서 발생할 수 있는 캐시 미적중이나 JVM 워밍업 등의 영향을 최소화하기 위해 5회의 웜업 실행을 먼저 수행하였다.</p>
<p>이후 동일한 요청을 10회 반복 실행하여 평균 응답 시간을 측정하였다.</p>
<p>또한 쿼리 변경 전후의 차이를 보다 명확히 확인하기 위해 데이터베이스에서 실제로 반환되는 row 수를 함께 출력하도록 구성하였다.</p>
<img src="https://velog.velcdn.com/images/se0o_129/post/4c3b8554-b848-4a3a-affd-75fa1a11542b/image.png" width="60%" />

<p><img alt="" src="https://velog.velcdn.com/images/se0o_129/post/eea89991-624f-415f-8df5-e87ac13d5116/image.png" /></p>
<br />

<hr />
<h2 id="3-개선-결과">3. 개선 결과</h2>
<p>기존에는 데이터베이스에서 200개의 row를 모두 조회한 뒤 애플리케이션 레벨에서 필터링을 수행하고 있었다.
쿼리를 개선한 이후에는 필요한 <strong>11개의 row만 반환</strong>하도록 변경되었다.</p>
<p>그 결과 평균 응답 시간이 <strong>24.66ms → 17.99ms</strong>로 감소하였다.
<img src="https://velog.velcdn.com/images/se0o_129/post/f30d3eb9-b8ae-4b51-b9a5-cccdf84b059a/image.png" width="70%" /></p>
<p>데이터가 많아질수록 전체 조회 비용이 증가하기 때문에, 
이러한 구조 개선의 효과는 더욱 커질 것으로 예상된다.</p>
<br />

<hr />
<h2 id="4-배운점">4. 배운점</h2>
<p>이번 개선의 핵심은 단순한 응답 시간 단축보다
애플리케이션 레벨에서 처리하던 필터링과 페이지네이션을 
데이터베이스 레벨로 이동시켜 구조를 개선했다는 점에 있다.</p>
<p>그동안 성능 개선이라고 하면 
응답 시간과 같은 수치적인 변화에만 집중하는 경향이 있었다.</p>
<p>하지만 이번 작업을 통해 좋은 코드는 단순한 성능 수치뿐 아니라 
코드의 흐름과 구조를 개선하는 과정에서도 만들어질 수 있다는 점을 배울 수 있었다.</p>