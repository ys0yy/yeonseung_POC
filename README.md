1. 혼잡도 계산 기준

빈틈에서는 최근 제보가 현재 카페 상황을 보여준다고 보고, 최근 60분 이내의 제보를 기준으로 현재 혼잡도를 계산한다.

제보가 너무 적은 상태에서 혼잡도를 판단하면 실제 상황과 다를 수 있기 때문에 최소 2개의 유효 제보가 있어야 혼잡도를 표시하도록 정했다.

항목	기준
제보 조회 범위	현재 시간 기준 최근 60분
최소 제보 수	2건
사용 가능한 상태	여유 / 보통 / 혼잡
중복 제보	같은 사용자가 같은 카페에 60분 이내 여러 번 제보한 경우 최신 제보만 사용
혼잡도 결정	가장 많이 제보된 상태
최다 상태가 같은 경우	정보 부족
유효 제보가 2건 미만	정보 부족
60분보다 오래된 제보	현재 혼잡도 계산에서 제외
잘못된 상태값	계산에서 제외
혼잡도 계산 예시

최근 60분 동안 유효한 제보가 다음과 같이 들어왔다고 가정한다.

제보	혼잡도
1	보통
2	보통
3	혼잡
4	여유

→ 보통 2건 / 혼잡 1건 / 여유 1건

따라서 현재 혼잡도는 보통으로 표시한다.

반대로

제보	혼잡도
1	여유
2	혼잡

처럼 2개의 상태가 같은 경우에는 어느 한쪽을 임의로 선택하지 않고 정보 부족으로 표시한다.

2. 혼잡도 계산 흐름

내가 보기엔 네 과제 제출물에는 이 흐름 하나 넣어주면 충분해.

혼잡도 제보 조회
       ↓
해당 카페의 제보만 확인
       ↓
최근 60분 이내인지 확인
       ↓
유효하지 않은 제보 제외
       ↓
동일 사용자의 중복 제보 확인
       ↓
가장 최근 제보만 사용
       ↓
유효 제보가 2건 이상인가?
    ↓              ↓
   아니오           예
    ↓              ↓
정보 부족       상태별 제보 수 계산
                       ↓
                 가장 많은 상태 확인
                       ↓
                 최다 상태가 1개인가?
                    ↓          ↓
                   아니오       예
                    ↓          ↓
                 정보 부족   현재 혼잡도 결정
                                  ↓
                         화면에 결과 표시

이렇게 하면 네가 앞에서 작성한 "제보 부족 → 정보 부족", "중복 제보 제한", "최근 제보 기준"이 코드에 실제로 연결돼.

3. Java 코드

너희 기술 스택이 Spring Boot + Java니까 Java로 구현하는 게 제일 자연스러워.

CongestionStatus.java
public enum CongestionStatus {
    RELAXED,   // 여유
    NORMAL,    // 보통
    CROWDED,   // 혼잡
    UNKNOWN    // 정보 부족
}
CongestionReport.java
import java.time.LocalDateTime;

public class CongestionReport {

    private Long cafeId;
    private String userId;
    private CongestionStatus status;
    private LocalDateTime reportedAt;

    public CongestionReport(Long cafeId, String userId,
                            CongestionStatus status,
                            LocalDateTime reportedAt) {
        this.cafeId = cafeId;
        this.userId = userId;
        this.status = status;
        this.reportedAt = reportedAt;
    }

    public Long getCafeId() {
        return cafeId;
    }

    public String getUserId() {
        return userId;
    }

    public CongestionStatus getStatus() {
        return status;
    }

    public LocalDateTime getReportedAt() {
        return reportedAt;
    }
}
4. 혼잡도 계산 클래스

여기가 핵심이야.

import java.time.LocalDateTime;
import java.util.*;

public class CongestionCalculator {

    private static final int MIN_REPORT_COUNT = 2;

    public CongestionStatus calculate(
            Long cafeId,
            List<CongestionReport> reports,
            LocalDateTime now) {

        LocalDateTime startTime = now.minusMinutes(60);

        // 1. 해당 카페 + 최근 60분 이내 제보만 확인
        List<CongestionReport> validReports = reports.stream()
                .filter(report -> report.getCafeId().equals(cafeId))
                .filter(report -> !report.getReportedAt().isBefore(startTime))
                .filter(report -> !report.getReportedAt().isAfter(now))
                .filter(report -> report.getStatus() != null)
                .toList();

        // 2. 같은 사용자의 중복 제보는 가장 최근 것만 사용
        Map<String, CongestionReport> latestReports = new HashMap<>();

        for (CongestionReport report : validReports) {
            String userId = report.getUserId();

            if (!latestReports.containsKey(userId)
                    || report.getReportedAt()
                    .isAfter(latestReports.get(userId).getReportedAt())) {

                latestReports.put(userId, report);
            }
        }

        List<CongestionReport> finalReports =
                new ArrayList<>(latestReports.values());

        // 3. 최소 제보 수 확인
        if (finalReports.size() < MIN_REPORT_COUNT) {
            return CongestionStatus.UNKNOWN;
        }

        // 4. 혼잡도별 개수 계산
        int relaxed = 0;
        int normal = 0;
        int crowded = 0;

        for (CongestionReport report : finalReports) {

            switch (report.getStatus()) {
                case RELAXED -> relaxed++;
                case NORMAL -> normal++;
                case CROWDED -> crowded++;
            }
        }

        // 5. 가장 많이 나온 상태 확인
        int max = Math.max(relaxed, Math.max(normal, crowded));

        int maxCount = 0;

        if (relaxed == max) maxCount++;
        if (normal == max) maxCount++;
        if (crowded == max) maxCount++;

        // 가장 많은 상태가 2개 이상이면 판단하지 않음
        if (maxCount > 1) {
            return CongestionStatus.UNKNOWN;
        }

        if (relaxed == max) {
            return CongestionStatus.RELAXED;
        }

        if (normal == max) {
            return CongestionStatus.NORMAL;
        }

        return CongestionStatus.CROWDED;
    }
}
이 코드에서 중요한 부분

너무 어렵게 설명할 필요 없이 과제에는 이렇게 적으면 돼.

최근 60분 내 제보를 먼저 확인하고, 같은 사용자가 여러 번 제보한 경우 가장 최근 제보만 사용한다. 이후 유효 제보가 2건 이상일 때 혼잡도별 개수를 비교하여 가장 많이 나온 상태를 현재 혼잡도로 결정한다. 가장 많이 나온 상태가 2개 이상이면 특정 상태를 판단하지 않고 정보 부족으로 처리한다.

이 정도면 학생이 직접 설계한 로직 설명처럼 보여.

5. 테스트 코드

JUnit으로 테스트하면 과제 제출할 때 훨씬 좋아 보여.

CongestionCalculatorTest.java
import org.junit.jupiter.api.Test;

import java.time.LocalDateTime;
import java.util.List;

import static org.junit.jupiter.api.Assertions.assertEquals;

class CongestionCalculatorTest {

    private final CongestionCalculator calculator =
            new CongestionCalculator();

    private final LocalDateTime now =
            LocalDateTime.of(2026, 9, 17, 14, 0);

    @Test
    void 보통이_가장_많으면_보통으로_나온다() {

        List<CongestionReport> reports = List.of(
                new CongestionReport(
                        1L, "user1",
                        CongestionStatus.NORMAL,
                        now.minusMinutes(10)
                ),
                new CongestionReport(
                        1L, "user2",
                        CongestionStatus.NORMAL,
                        now.minusMinutes(20)
                ),
                new CongestionReport(
                        1L, "user3",
                        CongestionStatus.CROWDED,
                        now.minusMinutes(30)
                )
        );

        assertEquals(
                CongestionStatus.NORMAL,
                calculator.calculate(1L, reports, now)
        );
    }

    @Test
    void 제보가_한개면_정보부족이다() {

        List<CongestionReport> reports = List.of(
                new CongestionReport(
                        1L, "user1",
                        CongestionStatus.RELAXED,
                        now.minusMinutes(10)
                )
        );

        assertEquals(
                CongestionStatus.UNKNOWN,
                calculator.calculate(1L, reports, now)
        );
    }

    @Test
    void 최근_60분이_지난_제보는_제외한다() {

        List<CongestionReport> reports = List.of(
                new CongestionReport(
                        1L, "user1",
                        CongestionStatus.RELAXED,
                        now.minusMinutes(10)
                ),
                new CongestionReport(
                        1L, "user2",
                        CongestionStatus.CROWDED,
                        now.minusMinutes(80)
                )
        );

        assertEquals(
                CongestionStatus.UNKNOWN,
                calculator.calculate(1L, reports, now)
        );
    }

    @Test
    void 같은_사용자의_중복제보는_최근것만_사용한다() {

        List<CongestionReport> reports = List.of(
                new CongestionReport(
                        1L, "user1",
                        CongestionStatus.CROWDED,
                        now.minusMinutes(30)
                ),
                new CongestionReport(
                        1L, "user1",
                        CongestionStatus.RELAXED,
                        now.minusMinutes(5)
                ),
                new CongestionReport(
                        1L, "user2",
                        CongestionStatus.RELAXED,
                        now.minusMinutes(10)
                )
        );

        assertEquals(
                CongestionStatus.RELAXED,
                calculator.calculate(1L, reports, now)
        );
    }

    @Test
    void 여유와_혼잡이_같으면_정보부족이다() {

        List<CongestionReport> reports = List.of(
                new CongestionReport(
                        1L, "user1",
                        CongestionStatus.RELAXED,
                        now.minusMinutes(10)
                ),
                new CongestionReport(
                        1L, "user2",
                        CongestionStatus.CROWDED,
                        now.minusMinutes(20)
                )
        );

        assertEquals(
                CongestionStatus.UNKNOWN,
                calculator.calculate(1L, reports, now)
        );
    }
}
6. SQL로 계산

여기서는 PostgreSQL 기준으로 하면 돼.

SQL에서는 Java에서 한 일을 순서대로 나눠서 생각하면 어렵지 않아.

① 최근 60분 제보 조회
SELECT
    cafe_id,
    user_id,
    status,
    reported_at
FROM congestion_report
WHERE cafe_id = 1
  AND reported_at >= CURRENT_TIMESTAMP - INTERVAL '60 minutes'
  AND reported_at <= CURRENT_TIMESTAMP;
② 같은 사용자의 중복 제보 제거

PostgreSQL에서는 ROW_NUMBER()를 사용하면 편해.

WITH recent_reports AS (
    SELECT
        cafe_id,
        user_id,
        status,
        reported_at,
        ROW_NUMBER() OVER (
            PARTITION BY user_id
            ORDER BY reported_at DESC
        ) AS rn
    FROM congestion_report
    WHERE cafe_id = 1
      AND reported_at >= CURRENT_TIMESTAMP - INTERVAL '60 minutes'
      AND reported_at <= CURRENT_TIMESTAMP
)

SELECT
    cafe_id,
    user_id,
    status,
    reported_at
FROM recent_reports
WHERE rn = 1;

여기서 rn = 1인 것만 가져오니까 같은 사람이 여러 번 제보해도 가장 최근 제보 하나만 남게 돼.

7. 최종 혼잡도 SQL

그다음 상태별 개수를 세면 돼.

WITH recent_reports AS (
    SELECT
        cafe_id,
        user_id,
        status,
        reported_at,
        ROW_NUMBER() OVER (
            PARTITION BY user_id
            ORDER BY reported_at DESC
        ) AS rn
    FROM congestion_report
    WHERE cafe_id = 1
      AND reported_at >= CURRENT_TIMESTAMP - INTERVAL '60 minutes'
      AND reported_at <= CURRENT_TIMESTAMP
),

valid_reports AS (
    SELECT
        status
    FROM recent_reports
    WHERE rn = 1
),

status_count AS (
    SELECT
        status,
        COUNT(*) AS report_count
    FROM valid_reports
    GROUP BY status
),

total_count AS (
    SELECT COUNT(*) AS total
    FROM valid_reports
),

max_count AS (
    SELECT MAX(report_count) AS max_report_count
    FROM status_count
),

top_status AS (
    SELECT status
    FROM status_count
    WHERE report_count = (SELECT max_report_count FROM max_count)
)

SELECT
    CASE
        WHEN (SELECT total FROM total_count) < 2
            THEN '정보 부족'

        WHEN (SELECT COUNT(*) FROM top_status) > 1
            THEN '정보 부족'

        ELSE (SELECT status FROM top_status)
    END AS congestion_status,

    (SELECT total FROM total_count) AS valid_report_count;

이 SQL의 결과는 예를 들어

congestion_status	valid_report_count
보통	4

이런 식으로 나와.

8. 테스트 결과

과제 제출할 때는 테스트 결과를 표로 하나 만들어주는 게 좋아.

실제로 네가 테스트하고 나온 결과에 맞춰서 숫자만 수정하면 돼.

테스트	입력 상황	예상 결과
1	보통 3건, 여유 1건	보통
2	여유 1건	정보 부족
3	최근 제보 1건 + 70분 전 제보 3건	정보 부족
4	여유 2건, 혼잡 1건	여유
5	여유 1건, 혼잡 1건	정보 부족
6	같은 사용자 제보 2건 + 다른 사용자 제보 1건	최신 제보 기준으로 계산
7	해당 카페의 제보가 아닌 다른 카페 제보	계산에서 제외
