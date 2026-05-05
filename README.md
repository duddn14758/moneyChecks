# 도토리창고 가계부 구현 정리

## 현재 구조

이 프로젝트는 GitHub Pages의 `index.html`을 프론트엔드로 사용하고, Google Apps Script의 `Code.gs`를 백엔드 API로 사용한다.

- 프론트엔드: `index.html`
- 백엔드: `Code.gs`
- 데이터 저장소: Google Spreadsheet
- 원본 서식 참고 파일: `🐿️ 도토리창고 가계부🐿️.xlsx`

프론트엔드는 정적 페이지이므로 GitHub Pages에서 빠르게 로딩된다. 실제 데이터 저장, 삭제, 히스토리 조회, 통계 계산은 Apps Script API를 통해 수행한다.

## 화면 구성

로그인 후 앱은 세 개의 탭으로 동작한다.

1. 입력
2. 히스토리
3. 통계

### 입력 탭

초기 진입 시 기본으로 보이는 화면이다.

입력 탭에서는 다음 항목을 입력한다.

- 날짜
- 카테고리
- 태그
- 내용
- 금액
- 입금계좌
- 출금계좌

저장 버튼을 누르면 Apps Script API에 `POST` 요청을 보내고, `Code.gs`의 `addRecord()`가 `가계부 기록` 시트에 새 행을 추가한다.

### 히스토리 탭

히스토리 탭은 `action=records` GET API를 사용한다.

현재 동작 방식:

- 최초 조회 시 최근 기록 10개를 가져온다.
- 아래쪽 `더보기` 버튼을 누르면 다음 10개를 추가로 가져온다.
- 삭제 버튼을 누르면 `POST` 요청으로 해당 rowIndex를 전달하고, Apps Script에서 행을 삭제한다.

관련 프론트 함수:

- `loadRecords(reset, silent, force)`
- `loadMoreRecords()`
- `renderList()`

관련 백엔드 함수:

- `getRecordsPage(offset, limit)`
- `getAllRecordRows()`
- `deleteRecord(rowIndex)`

### 통계 탭

통계 탭은 `action=monthlyStats` GET API를 사용한다.

현재 동작 방식:

- 기본값은 현재 월이다.
- 선택한 월의 지출만 집계한다.
- 지출 판단 기준은 `가계부 기록` 시트의 `태그` 값이 `지출`인 경우다.
- 지출 카테고리별 합계를 원형 그래프로 보여준다.

관련 프론트 함수:

- `loadStats(silent, force)`
- `renderStats(data)`
- `drawPieChart(items, total)`
- `renderStatsLegend(items, total)`

관련 백엔드 함수:

- `getMonthlyExpenseStats(year, month)`

## 초기 로딩 최적화

초기 로딩 시간을 줄이기 위해 입력 탭에서 필요한 카테고리와 계좌 목록은 현재 `index.html`에 하드코딩되어 있다.

관련 상수:

- `HARDCODED_CATEGORIES`
- `HARDCODED_ACCOUNTS`

따라서 로그인 후 입력 화면을 표시할 때는 Apps Script GET API를 호출하지 않는다.

현재 `loadData()`는 다음처럼 동작한다.

```js
function loadData() {
  categories = HARDCODED_CATEGORIES;
  accounts = HARDCODED_ACCOUNTS;

  renderCategory();
  renderAccounts();
}
```

이 구조 덕분에 메인 입력 화면은 Apps Script 응답을 기다리지 않고 바로 렌더링된다.

## 사일런트 프리로드

입력 화면이 표시된 뒤, 히스토리와 통계 탭으로 이동할 때의 체감 속도를 줄이기 위해 백그라운드 프리로드를 수행한다.

관련 함수:

```js
function preloadSecondaryData(force) {
  setTimeout(() => {
    loadRecords(true, true, force);
    loadStats(true, force);
  }, 200);
}
```

동작 방식:

- 입력 화면은 즉시 표시한다.
- 0.2초 뒤 히스토리 첫 페이지와 현재 월 통계를 조용히 가져온다.
- 사용자가 히스토리 또는 통계 탭을 누를 때 이미 데이터가 있으면 API를 다시 호출하지 않고 즉시 표시한다.

## 저장/삭제 중 레이스 방지

사일런트 로딩 중 저장 또는 삭제가 발생하면 오래된 응답이 늦게 도착할 수 있다.

이를 막기 위해 `dataVersion`을 사용한다.

관련 변수:

```js
let dataVersion = 0;
```

동작 방식:

- 히스토리/통계 요청을 보낼 때 현재 `dataVersion`을 함께 기억한다.
- 저장 또는 삭제가 완료되면 `dataVersion`을 증가시킨다.
- 이전 버전의 요청 응답이 늦게 도착하면 무시한다.
- 입력 탭에 머물러 있으면 최신 데이터를 다시 사일런트 프리로드한다.

이 덕분에 저장 직후 히스토리나 통계에 예전 데이터가 덮이는 문제를 방지한다.

## Apps Script API

현재 프론트에서 실제로 사용하는 API는 다음과 같다.

### 기록 조회

```txt
GET {API}?action=records&offset=0&limit=10
```

응답 예시:

```json
{
  "records": [],
  "nextOffset": 10,
  "hasMore": true
}
```

### 월별 지출 통계

```txt
GET {API}?action=monthlyStats&year=2026&month=5
```

응답 예시:

```json
{
  "year": 2026,
  "month": 5,
  "total": 0,
  "items": []
}
```

### 기록 추가

```txt
POST {API}
```

요청 body:

```json
{
  "action": "add",
  "secret": "secret123",
  "date": "2026/05/06",
  "type": "지출  ›  🍔 식비  ›  외식·배달🛵",
  "tag": "지출",
  "content": "내용",
  "amount": 10000,
  "inAccount": "",
  "outAccount": "💳 삼성신용카드(공금)"
}
```

### 기록 삭제

```txt
POST {API}
```

요청 body:

```json
{
  "action": "delete",
  "secret": "secret123",
  "rowIndex": 356
}
```

## Deprecated 초기 데이터 API

현재 `Code.gs`에는 카테고리와 계좌 목록을 스프레드시트에서 읽어오는 GET API가 남아 있다.

프론트에서는 현재 사용하지 않지만, 나중에 하드코딩을 제거하고 실제 스프레드시트 값을 다시 읽고 싶을 때 사용할 수 있다.

관련 백엔드 함수:

- `getInitialData()`
- `getCategories()`
- `getAccounts()`
- `getCachedJson(key, factory)`

`action` 없이 Apps Script API를 호출하면 초기 데이터가 반환된다.

```txt
GET {API}
```

응답 구조:

```json
{
  "categories": [
    { "value": "지출  ›  🍔 식비  ›  식료품🛒" }
  ],
  "accounts": [
    "💵 기업은행 허브"
  ]
}
```

이 API는 `CacheService`를 사용해 카테고리와 계좌 목록을 일정 시간 캐시한다.

## 하드코딩 해제 방법

나중에 카테고리/계좌 목록을 다시 Apps Script에서 받아오고 싶다면 `index.html`의 `loadData()`를 API 호출 방식으로 되돌리면 된다.

현재 코드:

```js
function loadData() {
  categories = HARDCODED_CATEGORIES;
  accounts = HARDCODED_ACCOUNTS;

  renderCategory();
  renderAccounts();
}
```

API 기반으로 변경할 코드:

```js
function loadData() {
  fetch(API)
    .then(res => res.json())
    .then(data => {
      categories = data.categories || [];
      accounts = data.accounts || [];

      renderCategory();
      renderAccounts();
    })
    .catch(() => alert("API 실패"));
}
```

`renderCategory()`는 현재 하드코딩 배열과 API 응답 배열을 모두 처리할 수 있게 되어 있다.

```js
function renderCategory() {
  const select = document.getElementById("category");
  select.innerHTML = `<option value="">카테고리 선택</option>`;
  categories.forEach(c => {
    const value = c.value || c;
    select.innerHTML += `<option value="${value}">${value}</option>`;
  });
}
```

따라서 `loadData()`만 바꾸면 된다.

## 시트명 처리

`Code.gs`에서는 시트명을 한글 그대로 사용한다.

현재 지원하는 시트명:

```js
const RECORD_SHEET_NAMES = ["가계부 기록 시트", "가계부 기록 ✍️"];
const SETTING_SHEET_NAMES = ["가계부 설정 시트", "가계부 설정 ⚙️"];
```

이렇게 두 이름을 모두 지원하는 이유는 실제 스프레드시트 시트명이 이모지를 포함할 수 있고, 기존 코드 또는 배포 환경에서는 이모지가 없는 이름을 사용할 수도 있기 때문이다.

## 배포 시 주의사항

`index.html`만 수정한 경우:

- GitHub Pages에 반영하면 된다.

`Code.gs`를 수정한 경우:

- Apps Script 편집기에 코드를 반영한다.
- 웹앱을 새 버전으로 배포한다.
- `index.html`의 `API` URL이 새 배포 URL과 일치하는지 확인한다.

## 현재 성능 전략

현재 성능 최적화 방향은 다음과 같다.

- 입력 화면 필수 옵션은 하드코딩해 즉시 렌더링한다.
- 히스토리는 10개씩 페이지 단위로 가져온다.
- 통계는 월별로 필요한 시점에만 계산한다.
- 입력 화면 진입 후 히스토리와 통계를 사일런트 프리로드한다.
- 저장/삭제 후 오래된 프리로드 응답은 `dataVersion`으로 무시한다.

이 구조는 개인 가계부처럼 설정값은 거의 바뀌지 않고 기록만 계속 추가되는 앱에 맞춰 초기 체감 속도를 우선한 방식이다.
