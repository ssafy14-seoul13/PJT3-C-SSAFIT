# SSAFIT

---

## 팀원

* **채연 솔하**

---

## 목차

- [SSAFIT](#ssafit)
  - [팀원](#팀원)
  - [목차](#목차)
  - [프로젝트 개요](#프로젝트-개요)
  - [목표](#목표)
  - [자료구조 설계](#자료구조-설계)
    - [회원](#회원)
    - [영상](#영상)
    - [리뷰](#리뷰)
    - [찜](#찜)
    - [운동 계획](#운동-계획)
    - [헬스장](#헬스장)
  - [알고리즘 설계](#알고리즘-설계)
    - [Algo1: 영상 검색 \& 정렬](#algo1-영상-검색--정렬)
    - [Algo2: 리뷰 조회](#algo2-리뷰-조회)
    - [Algo3: 내 주변 헬스장 검색](#algo3-내-주변-헬스장-검색)
    - [Algo4: 개인화 추천](#algo4-개인화-추천)
    - [Algo5: 운동 계획 최적화](#algo5-운동-계획-최적화)
  - [성능 분석 (Big-O)](#성능-분석-big-o)
  - [기대 효과](#기대-효과)
  - [이슈 및 해결 방안](#이슈-및-해결-방안)

---

## 프로젝트 개요

**SSAFIT**은 운동 영상 데이터를 기반으로 사용자에게 **부위별/인기순 추천**과 **리뷰** 기능을 제공하는 **운동 루틴 추천 & 커뮤니티 서비스**입니다.
본 프로젝트는 **JSP/Servlet 기반 MVC 아키텍처**로 구현되며, 서비스의 주요 기능을 적절한 **자료구조와 알고리즘**으로 설계하여 **성능/확장성**을 확보하는 것을 목표로 합니다.

---

## 목표

* 서비스 기능에 맞는 **자료구조 선택** 및 **알고리즘 적용**
* **Big-O** 표기법을 통한 **성능 분석**
* **검색 속도**, **정렬 응답성**, **추천 정확도** 향상
* 회원·영상 데이터 증가에 대비한 **확장성 최적화**

---

## 자료구조 설계

### 회원

* **자료구조:** `List<User>` (초기), **RDB 테이블**(`users`)
* **이유:** CRUD 중심. `email`/`id` 인덱스로 단건 조회 최적.
* **장점:** 단순한 회원 CRUD에 적합, **DB 인덱스**를 통한 효율적 검색.
* **단점:** 팔로우/팔로잉 확장 시 **N\:M** 관계 테이블(`follows`) 필요.

---

### 영상

* **자료구조:** `ArrayList<Video>`
* **이유:** **조회/정렬 빈번** → 배열 기반 리스트가 효율적.
* **장점:** **O(1)** 인덱스 접근, **DB 인덱스 + 캐시** 활용 가능.
* **단점:** 중간 **삽입/삭제**가 잦으면 성능 저하.

---

### 리뷰

* **자료구조:** `HashMap<videoId, List<Review>>`
* **이유:** 특정 **영상의 리뷰**를 빠르게 불러오기 위함.
* **성능:** 평균 **O(1)** 조회(HashMap), 리뷰 정렬 시 **O(n log n)**.
* **장점:** **영상별 리뷰 집계** 용이.
* **단점:** 해시 충돌 시 성능 저하, **DB에서는 JOIN/페이징** 처리 필요.

---

### 찜

* **자료구조:** 사용자별 `Set<videoId>`
* **이유:** **중복 없는 저장** 필요.
* **장점:** 삽입/삭제/포함 여부 **O(1)**.
* **단점:** **순서 없음** → “최근 찜”은 `LinkedHashSet` 또는 **타임스탬프 컬럼** 필요.

---

### 운동 계획

* **자료구조:** `List<Plan>`
* **이유:** **시간순 정렬** 중요.
* **성능:** 순회 **O(n)**, DB에서는 `ORDER BY plan_date`.
* **장점:** **캘린더 UI** 연동 용이.

---

### 헬스장

* **자료구조:** `ArrayList<Gym>`
* **이유:** CSV 로드 후 **좌표 기반 거리 계산**.
* **성능:** 선형 검색 **O(n)** → **k-d Tree / R-Tree / SPATIAL INDEX** 적용 시 **O(log n)** 수준으로 개선.

---

## 알고리즘 설계

### Algo1: 영상 검색 & 정렬

* **알고리즘:** 퀵정렬(Quick Sort), 병합정렬(Merge Sort), **DB `ORDER BY`**
* **적용 기능:** 인기순, 최신순, 리뷰 많은 순, 영상 길이순
* **복잡도:** **O(n log n)**
* **장점:** 빠른 정렬 성능, **DB 인덱스 최적화**와 결합 시 실무 성능 우수
* **단점:** 데이터셋이 매우 작을 때는 **삽입정렬** 등 단순 알고리즘이 더 효율적일 수 있음
  
```java

@WebServlet("/video/search")
public class VideoSearchServlet extends HttpServlet {
    private VideoDao videoDao = new VideoDao();

@Override
protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws IOException, ServletException {
    String keyword = req.getParameter("q");
    String sort = req.getParameter("sort"); // popular, latest, review, duration

    List<Video> list = videoDao.findAll(keyword);

    switch (sort) {
        case "popular" -> list.sort(Comparator.comparingInt(Video::getViews).reversed());
        case "latest" -> list.sort(Comparator.comparing(Video::getCreatedAt).reversed());
        case "review" -> list.sort(Comparator.comparingInt(Video::getReviewCount).reversed());
        case "duration" -> list.sort(Comparator.comparingInt(Video::getDurationSec));
    }

    req.setAttribute("videos", list);
    req.getRequestDispatcher("/videoList.jsp").forward(req, resp);
}
}

---

### Algo2: 리뷰 조회

* **알고리즘:** **해시 탐색(Hash Lookup)**
* **적용 기능:** 특정 영상의 리뷰 **즉시 조회**
* **복잡도:** 평균 **O(1)**, 최악 **O(n)**
* **장점:** 상세 모달 오픈 시 **즉시 렌더** UX 제공
* **단점:** 대규모 리뷰에서 **해시 충돌/메모리 관리** 필요 (캐시 TTL/무효화 전략 병행)

```java
@WebServlet("/review/list")
public class ReviewListServlet extends HttpServlet {
    private ReviewDao reviewDao = new ReviewDao();

@Override
protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws IOException, ServletException {
    long videoId = Long.parseLong(req.getParameter("videoId"));
    // HashMap<videoId, List<Review>> 캐싱 가능
    List<Review> reviews = reviewDao.findByVideoId(videoId);

    req.setAttribute("reviews", reviews);
    req.getRequestDispatcher("/reviewList.jsp").forward(req, resp);
}
}

---

### Algo3: 내 주변 헬스장 검색

* **알고리즘:** 피타고라스 정리(근사) 또는 **Haversine 공식**(구면 거리)
* **적용 기능:** 사용자 위치 `(lat, lng)`와 DB 좌표 비교 후 **가까운 헬스장** 반환
* **복잡도:** 선형 **O(n)** → 공간 분할(k-d Tree, R-Tree)·**SPATIAL INDEX** 적용 시 후보 축소로 성능 향상
* **장점:** **위치 기반 맞춤 서비스** 구현
* **단점:** **GPS 오차**, **권한 허용 UX** 이슈

```java
@WebServlet("/gym/nearby")
public class GymSearchServlet extends HttpServlet {
    private GymDao gymDao = new GymDao();

@Override
protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws IOException, ServletException {
    double myLat = Double.parseDouble(req.getParameter("lat"));
    double myLng = Double.parseDouble(req.getParameter("lng"));

    List<Gym> gyms = gymDao.findAll();

    gyms.sort(Comparator.comparingDouble(g -> haversine(myLat, myLng, g.getLat(), g.getLng())));

    req.setAttribute("gyms", gyms);
    req.getRequestDispatcher("/gymList.jsp").forward(req, resp);
}

private double haversine(double lat1, double lon1, double lat2, double lon2) {
    double R = 6371e3; // 지구 반지름 (m)
    double dLat = Math.toRadians(lat2 - lat1);
    double dLon = Math.toRadians(lon2 - lon1);
    double a = Math.sin(dLat/2) * Math.sin(dLat/2) +
            Math.cos(Math.toRadians(lat1)) * Math.cos(Math.toRadians(lat2)) *
            Math.sin(dLon/2) * Math.sin(dLon/2);
    double c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a));
    return R * c;
}
}

---

### Algo4: 개인화 추천

* **알고리즘:** **협업 필터링(CF)**, **콘텐츠 기반 추천**, (확장) **AI 모델**
* **적용 기능:** 사용자 **시청·리뷰 기록 기반** 추천
* **복잡도:** 기본 CF는 **O(mn)** (m=사용자 수, n=아이템 수)
* **장점:** **개인화**로 만족도/체류시간/재방문율 상승
* **단점:** **콜드스타트**(신규 사용자/아이템) 발생 → 인기순과 **하이브리드**로 완화

```java
@WebServlet("/recommend")
public class RecommendServlet extends HttpServlet {
    private RecommendService recommendService = new RecommendService();

@Override
protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws IOException, ServletException {
    long userId = Long.parseLong(req.getParameter("userId"));

    List<Video> recs = recommendService.recommend(userId);

    req.setAttribute("recommendations", recs);
    req.getRequestDispatcher("/recommend.jsp").forward(req, resp);
}
}

---

### Algo5: 운동 계획 최적화

* **알고리즘:** **동적 계획법(DP)**
* **적용 기능:** 제한된 시간(예: 하루 60분) 안에 **효과 최대** 조합 추천
* **복잡도:** **O(n·W)** (n=운동 개수, W=총 시간)
* **장점:** **제약 최적화**로 “최적 루틴” 자동 제안
* **단점:** 입력 규모 커지면 계산량 증가 → **그리디/휴리스틱** 병행 고려

```java
@WebServlet("/plan/optimize")
public class PlanOptimizeServlet extends HttpServlet {
    private PlanDao planDao = new PlanDao();

@Override
protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws IOException, ServletException {
    int totalTime = Integer.parseInt(req.getParameter("time")); // ex) 하루 60분
    List<Exercise> exercises = planDao.findAll();

    // DP: 0-1 Knapsack 유사
    int n = exercises.size();
    int[][] dp = new int[n+1][totalTime+1];

    for (int i=1; i<=n; i++) {
        Exercise ex = exercises.get(i-1);
        for (int t=0; t<=totalTime; t++) {
            if (ex.getTime() <= t) {
                dp[i][t] = Math.max(dp[i-1][t], dp[i-1][t-ex.getTime()] + ex.getEffect());
            } else {
                dp[i][t] = dp[i-1][t];
            }
        }
    }

    int maxEffect = dp[n][totalTime];
    req.setAttribute("maxEffect", maxEffect);
    req.getRequestDispatcher("/planResult.jsp").forward(req, resp);
}
}

---

## 성능 분석 (Big-O)

| 기능        | 자료구조             | 알고리즘        | 시간 복잡도     |
| --------- | ---------------- | ----------- | ---------- |
| 영상 정렬     | ArrayList        | Quick Sort  | O(n log n) |
| 리뷰 조회     | HashMap          | Hash Lookup | O(1)       |
| 헬스장 검색    | ArrayList        | Haversine   | O(n)       |
| 추천 시스템    | User-Item Matrix | CF          | O(mn)      |
| 운동 계획 최적화 | List             | DP          | O(n·W)     |

---

## 기대 효과

* **검색/정렬 응답 속도 향상** → 사용자 만족도 상승
* **리뷰 관리 효율성** → 평균 **O(1)** 접근
* **위치 기반 기능** → 차별화된 서비스 제공
* **개인화 추천** → 서비스 **지속 사용성** 증가

---

## 이슈 및 해결 방안

* **콜드스타트(추천)** → **인기순 + 콘텐츠 기반** 혼합으로 완화
* **대규모 데이터 성능 저하** → **DB 인덱스**, **비정규화(review\_count)**, **캐싱(Redis)**, **키셋 페이징**
* **보안** → 비밀번호 **해시(bcrypt/argon2)**, **세션/토큰** 기반 인증, 입력 검증
