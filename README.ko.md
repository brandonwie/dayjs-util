# @brandonwie/dayjs-util

[English](./README.md) | **한국어**

[dayjs](https://day.js.org/) 기반의 타임존 안전 날짜 유틸리티 — 캘린더 애플리케이션용으로 설계되었습니다.

## 왜 dayjs를 직접 사용하지 않는가?

| 문제                      | 순수 dayjs                                       | 이 라이브러리                                              |
| ------------------------- | ------------------------------------------------ | ---------------------------------------------------------- |
| **타임존 모호성**         | `dayjs("2025-01-01")` — UTC? 로컬? 서울?         | `DayjsUtil.tzParse("2025-01-01", "Asia/Seoul")` — 명시적   |
| **종일/시간 이벤트 구분** | 내장 구분 없음                                   | `stripTimezoneToUTC()` vs `convertToUTCDate()`             |
| **플러그인 설정**         | `extend(utc)`, `extend(timezone)` 매번 기억 필요 | import 시 자동 로드                                        |
| **반환 타입 명확성**      | 모든 것이 `Dayjs` 반환                           | `*Date` → JS Date, `*String` → string, 접미사 없음 → Dayjs |

## 설치

```bash
pnpm add @brandonwie/dayjs-util dayjs
# 또는
npm install @brandonwie/dayjs-util dayjs
```

> `dayjs`는 **peer dependency**입니다 — 버전을 직접 관리합니다.

## 빠른 시작

```typescript
import { DayjsUtil } from "@brandonwie/dayjs-util";

// 특정 타임존으로 날짜 문자열 파싱
const seoulMidnight = DayjsUtil.tzParse("2025-01-01 00:00:00", "Asia/Seoul");
// → 2025-01-01 00:00:00 KST를 나타내는 Dayjs (UTC: 2024-12-31 15:00:00)

// DB 저장용 UTC Date로 변환
const utcDate = DayjsUtil.convertToUTCDate("2025-06-15T09:00:00+09:00");
// → Date(2025-06-15T00:00:00.000Z)

// 종일 이벤트: 시간 유지, 타임존 제거
const allDay = DayjsUtil.stripTimezoneToUTC("2025-06-15T00:00:00+09:00");
// → Date(2025-06-15T00:00:00.000Z)  ← 시간 그대로 보존!

// API 응답용 포맷팅
DayjsUtil.formatUTCString(new Date()); // "2025-06-15T00:00:00Z"
DayjsUtil.formatISOString(new Date(), "Asia/Seoul"); // "2025-06-15T09:00:00+09:00"
```

## API 레퍼런스

### 파싱

| 메서드                       | 반환 타입 | 설명                                   |
| ---------------------------- | --------- | -------------------------------------- |
| `utc(date?)`                 | `Dayjs`   | UTC로 생성/변환                        |
| `tz(date?, timezone?)`       | `Dayjs`   | 타임존으로 변환 (같은 순간, 다른 표시) |
| `tzParse(str, timezone?)`    | `Dayjs`   | 해당 타임존으로 파싱 (다른 순간!)      |
| `parseToTz(str?, timezone?)` | `Dayjs`   | 문자열 파싱 후 타임존으로 표시         |

#### `tz()` vs `tzParse()` — 핵심 차이점

```typescript
const str = "2025-01-01 00:00:00";

// tz(): 서버 타임존에서 파싱 후 서울 시간으로 표시 변환
DayjsUtil.tz(str, "Asia/Seoul").toDate();
// → 2025-01-01T00:00:00.000Z (서버가 UTC인 경우)

// tzParse(): 문자열을 서울 시간으로 해석
DayjsUtil.tzParse(str, "Asia/Seoul").toDate();
// → 2024-12-31T15:00:00.000Z (9시간 이전!)
```

사용자 입력을 해당 타임존에서 해석해야 할 때 `tzParse()`를 사용하세요. 이미 알고 있는 UTC 시간을 표시용으로 변환할 때 `tz()`를 사용하세요.

### 변환

| 메서드                     | 반환 타입 | 설명                                               |
| -------------------------- | --------- | -------------------------------------------------- |
| `convertToUTCDate(date?)`  | `Date`    | 타임존 변환 → UTC. **시간 지정 이벤트**용.         |
| `stripTimezoneToUTC(str?)` | `Date`    | 시간 유지, 타임존만 UTC로 변경. **종일 이벤트**용. |
| `epoch()`                  | `Date`    | `1970-01-01T00:00:00Z` 반환. 센티넬 값.            |

### 포맷팅

| 메서드                             | 반환 타입 | 설명                            |
| ---------------------------------- | --------- | ------------------------------- |
| `formatISOString(date?, tz?)`      | `string`  | `2025-01-01T09:00:00+09:00`     |
| `formatUTCString(date?)`           | `string`  | `2025-01-01T00:00:00Z`          |
| `formatDateOnlyString(date?, tz?)` | `string`  | `2025-01-01` (타임존 고려)      |
| `extractDateOnlyString(date?)`     | `string`  | `2025-01-01` (타임존 변환 없음) |

### 비교

| 메서드                    | 반환 타입 | 설명                     |
| ------------------------- | --------- | ------------------------ |
| `isSame(d1?, d2?, unit?)` | `boolean` | 지정 단위로 두 날짜 비교 |
| `diff(d1?, d2?, unit?)`   | `number`  | 지정 단위로 차이 계산    |

### 검증

| 메서드                           | 반환 타입 | 설명                                  |
| -------------------------------- | --------- | ------------------------------------- |
| `isValidDateFormat(str, format)` | `boolean` | `DATE_FORMAT` 패턴에 대한 문자열 검증 |

#### 지원하는 `DATE_FORMAT` 상수

| 상수                 | 패턴                            | 예시                            |
| -------------------- | ------------------------------- | ------------------------------- |
| `DATE`               | `YYYY-MM-DD`                    | `2025-01-01`                    |
| `DATETIME`           | `YYYY-MM-DDTHH:mm:ss`           | `2025-01-01T10:00:00`           |
| `DATETIME_UTC`       | `YYYY-MM-DDTHH:mm:ssZ`          | `2025-01-01T10:00:00Z`          |
| `DATETIME_OFFSET`    | `YYYY-MM-DDTHH:mm:ss±HH:mm`     | `2025-01-01T10:00:00+09:00`     |
| `DATETIME_MS`        | `YYYY-MM-DDTHH:mm:ss.SSS`       | `2025-01-01T10:00:00.000`       |
| `DATETIME_MS_UTC`    | `YYYY-MM-DDTHH:mm:ss.SSSZ`      | `2025-01-01T10:00:00.000Z`      |
| `DATETIME_MS_OFFSET` | `YYYY-MM-DDTHH:mm:ss.SSS±HH:mm` | `2025-01-01T10:00:00.000+09:00` |

## EventDateHandler (캘린더 이벤트)

캘린더 이벤트 날짜 처리를 위한 선택적 import:

```typescript
import { EventDateHandler } from "@brandonwie/dayjs-util";
// 또는: import { EventDateHandler } from '@brandonwie/dayjs-util/event';

// 종일 이벤트: 타임존 제거, 시간 유지
const allDay = EventDateHandler.processAllDayEventDates(
  "2025-06-15T00:00:00+09:00",
  "2025-06-16T00:00:00+09:00",
);
// { startAt: Date(2025-06-15T00:00:00Z), endAt: Date(2025-06-16T00:00:00Z), zone: 'UTC' }

// 시간 지정 이벤트: UTC로 변환
const timed = EventDateHandler.processTimedEventDates(
  "2025-06-15T09:00:00+09:00",
  "2025-06-15T10:00:00+09:00",
  "Asia/Seoul",
);
// { startAt: Date(2025-06-15T00:00:00Z), endAt: Date(2025-06-15T01:00:00Z), zone: 'Asia/Seoul' }

// 통합: isAllDay 플래그에 따라 자동 분기
const [start, end, zone] = EventDateHandler.computeScheduleDates({
  startAt: "2025-06-15T09:00:00+09:00",
  endAt: "2025-06-15T10:00:00+09:00",
  timeZone: "Asia/Seoul",
  isAllDay: false,
});
```

## 마이그레이션 가이드: `new Date()` → DayjsUtil

| 기존 코드                          | 변경 후                                 | 이유                                |
| ---------------------------------- | --------------------------------------- | ----------------------------------- |
| `new Date()`                       | `DayjsUtil.utc().toDate()`              | 명시적 UTC, 로컬 타임존 모호성 제거 |
| `new Date(str)`                    | `DayjsUtil.utc(str).toDate()`           | 일관된 파싱                         |
| `new Date(str).toISOString()`      | `DayjsUtil.formatUTCString(str)`        | 동일 결과, 깔끔한 API               |
| `date.toISOString().split('T')[0]` | `DayjsUtil.extractDateOnlyString(date)` | 모든 입력 타입 처리                 |
| `new Date(0)`                      | `DayjsUtil.epoch()`                     | 자기 설명적 센티넬 값               |
| `d1.getTime() - d2.getTime()`      | `DayjsUtil.diff(d1, d2, 'ms')`          | 가독성, 단위 인식                   |
| 수동 오프셋 계산                   | `DayjsUtil.tz(date, 'Asia/Seoul')`      | IANA 타임존, DST 안전               |
| `dayjs(str).tz(tz)`                | `DayjsUtil.tzParse(str, tz)`            | 정확한 시맨틱 (위 설명 참조)        |

## 설계 결정

- **Static 클래스** — 인스턴스화 불필요, dayjs 인스턴스는 호출마다 생성 (불변, ~0.01ms)
- **플러그인 1회 로드** — utc, timezone, isSameOrAfter는 import 시 등록
- **dayjs peer dependency** — 사용자가 버전 관리, 중복 방지
- **Dual CJS/ESM** — Node.js, 브라우저, 번들러 모두 지원

## 라이선스

MIT
