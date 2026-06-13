# [claude] 전시회 적응형 추출 사다리 · 자가 업그레이드 엔진 고도화계획서 v1.0

- 작성일: 2026-06-13
- 대상 서비스: OCR Pro
- 대상 파이프라인: 전시회 입력 파이프라인 (`ExpoLeadLearner` 계열)
- 연계 계획서:
  - `[codex]전시회_웹페이지_자동분석_전체업체추적_고도화계획서_v1.0.md`
  - `[codex]전시회_이메일중심_추출고도화_추가계획서_v1.0.md` (v9.41·v9.60 구현 완료분)
  - `[codex]입력파이프라인별_독립학습엔진_고도화계획서_v1.0.md`
- 핵심 변경: 사이트별 하드코딩 어댑터 의존을 줄이고, **추출 실패를 스스로 감지 → 전략을 단계적으로 격상 → 통한 방법을 학습 프리셋으로 영속화**하는 적응형 엔진(`ExpoSiteLearner`)을 신설한다.

---

## 1. 핵심 결론

전시회 사이트는 사이트마다 렌더링 구조(정적 HTML / AJAX / SPA / 세션 기반)와 데이터 위치가 모두 다르다. 지금까지는 호스트마다 `EXPO_SITE_PRESETS`에 사람이 어댑터를 손으로 추가해 왔다. 이 방식은 새 사이트가 나올 때마다 개발자 개입이 필요하고, 사이트가 개편되면 조용히 실패한다.

목표는 **"코드를 다시 짜는 자가 업그레이드"가 아니다.** 정적 HTML 기반 웹앱에서는 런타임 코드 재작성이 불가능하고 위험하다. 현실적으로 달성 가능하고 충분히 강력한 형태는 다음이다.

> **시스템이 추출 전략을 스스로 선택하고(사다리), 통한 전략의 설정값(선택자·API URL·필드 경로)을 학습으로 기억한다.**

즉 "전략 선택 + 설정 학습"을 자동화하면, 사람이 보기에는 "처음 보는 사이트도 알아서 추출하고, 한 번 뚫은 사이트는 다음부터 즉시 처리한다"는 자가 업그레이드 체감이 나온다.

## 2. 현재 구조와 한계

| 자산 | 위치 | 역할 | 한계 |
|---|---|---|---|
| `EXPO_SITE_PRESETS` | index.html | 호스트별 하드코딩 어댑터 (bbs/html_table/json_api) | 새 사이트마다 수작업, 개편 시 무력화 |
| `tryDiscoverExhibitorAPI()` | index.html | inline JSON·공통 API 패턴 자동 탐색 | 패턴 라이브러리가 고정, 실패 시 폴백 없음 |
| `PipelineLearningRegistry` | index.html | 입력 source별 학습엔진 태깅 그릇 | 추출 *전략* 자체는 학습하지 않음 |
| `ExpoEmailHunter`(v9.60) | index.html | 이메일 확보·점수·등급·확보율 | 목록/상세 *수집*은 책임 밖 |

**빠진 두 가지:**
1. **추출 성공/실패 자동 판정** — "0건인데 완료" 같은 조용한 실패를 막는 정량 기준이 없다. (자가 업그레이드의 *트리거*)
2. **전략 격상 + 학습** — 실패 시 다음 전략으로 올라가고, 성공 설정을 저장하는 오케스트레이션이 없다.

## 3. 최종 목표

1. 미지의 전시회 사이트가 들어와도 사람 개입 없이 **사다리를 타고 내려가며** 추출을 시도한다.
2. 각 단계는 **성공/실패를 스스로 판정**하고, 실패 사유를 구조화해 다음 단계로 넘긴다.
3. 통한 단계의 설정을 **학습 프리셋**으로 localStorage에 저장하고, 다음 방문 시 0단계에서 즉시 재사용한다.
4. 클라이언트로는 도달 불가능한 사이트(세션·CSRF·서버 측 봇차단)는 **우아하게 "수동 캡처 필요" 상태로 떨어진다.** (억지로 뚫지 않는다)
5. 수집 결과는 기존 v9.60 이메일 파이프라인으로 그대로 흘려보내 **이메일 엔진을 한 줄도 새로 짜지 않는다.**
6. 다른 입력 파이프라인(명함·QR·문서·URL)에는 영향이 없다. (회귀 격리 원칙 유지)

## 4. 설계의 3대 기둥

```
① 실패를 감지한다   → ExtractionVerdict (정량 판정)
② 전략을 격상한다   → ExpoSiteLadder (6칸 폴백 사다리)
③ 통한 걸 기억한다  → ExpoSiteLearner (학습 프리셋 영속화)
```

세 기둥은 독립적으로 가치가 있으나, ①이 없으면 ②·③가 트리거되지 않으므로 **①을 최우선으로 구현한다.**

## 5. 추출 성공/실패 자동 판정 — `ExtractionVerdict` (기둥 ①)

추출 시도 결과를 다음 신호로 정량 평가한다.

| 신호 | 설명 | 기본 임계 |
|---|---|---|
| `companyCount` | 추출된 업체 수 | 1건 미만 → 실패 |
| `expectedCount` | 페이지가 노출한 총 업체 수(있으면) | 추출/기대 < 70% → 부분실패 |
| `coreFieldRate` | 핵심필드(회사명) 확보율 | < 90% → 부분실패 |
| `enrichFieldRate` | 보강필드(웹주소/이메일/카테고리) 확보율 | 참고용(실패 판정엔 미반영) |
| `duplicateRate` | 동일 회사명 중복 비율 | > 40% → 파서 오작동 의심 |
| `garbageRate` | 회사명에 슬로건·메뉴·날짜 등 비업체 토큰 비율 | > 30% → 셀렉터 오인식 의심 |

판정 등급:

| 등급 | 조건 | 동작 |
|---|---|---|
| `ok` | companyCount≥1 && coreFieldRate≥0.9 && garbageRate≤0.3 | 사다리 종료, 결과 채택 |
| `partial` | 1건 이상이나 임계 미달 | 다음 칸 시도, 더 나은 결과면 교체 |
| `fail` | 0건 또는 garbage/dup 과다 | 다음 칸으로 격상 |

**산출물:** `{ verdict, companyCount, expectedCount, coreFieldRate, reasons[] }`
**적용 지점:** `registerCompanies()` 진입 직전 + 각 사다리 칸 종료 시.
**조용한 실패 차단:** 완료 토스트에 항상 `ExtractionVerdict` 요약을 노출하고, `fail`이면 "수집 실패 — 자동 격상" 경고를 띄운다.

## 6. 적응형 추출 사다리 — `ExpoSiteLadder` (기둥 ②)

호스트 진입 시 위에서부터 시도, `ExtractionVerdict`가 `ok`가 될 때까지 한 칸씩 내려간다.

| 칸 | 전략 | 상태 | 비용 | 동적 대응 |
|---|---|---|---|---|
| 0 | **학습 프리셋** — 이 호스트의 과거 성공 설정 재사용 | 신규(③) | 0 | ○ |
| 1 | **하드코딩 프리셋** `EXPO_SITE_PRESETS` | 기존 | 0 | ○ |
| 2 | **API 자동탐색** `tryDiscoverExhibitorAPI` 확장 | 기존+확장 | 낮음 | ◎ |
| 3 | **DOM 패턴 마이닝** — 반복 카드/행 구조 자동 인식 | 신규 | 0 | △(정적만) |
| 4 | **AI 추출** — Gemini로 구조·셀렉터·API 추론 | 신규 | 높음(1회) | △ |
| 5 | **수동 캡처 요청** — DevTools XHR 붙여넣기 UI | 신규(반자동) | 사람 | ◎ |

각 칸은 동일 인터페이스를 따른다:

```
async function rung(url, html, ctx) -> {
  companies: [...],         // 추출 결과
  strategyConfig: {...},    // 통했을 때 ③이 저장할 설정 (selector/apiUrl/fieldMap)
  verdict: ExtractionVerdict
}
```

성공(`ok`) 시 `strategyConfig`를 ③에 넘겨 학습 프리셋으로 저장한다.

### 6.1 칸 2 — API 자동탐색 확장

현행 `tryDiscoverExhibitorAPI`에 추가:
- `.do`/`.asp` 계열: `selAction=single_page→list/ajax_list`, `index→index_list/ajax` 등 **action 치환 후보 자동 생성**
- 응답이 HTML 조각(부분 렌더)인 경우도 수용 → 칸 3(DOM 마이닝)으로 재투입
- 흔한 페이지네이션 파라미터(`page/start/offset/pageNo/GotoPage`) 자동 증가 탐색

### 6.2 칸 3 — DOM 패턴 마이닝 (신규)

정적 HTML에서 **반복되는 형제 노드 군집**을 통계적으로 찾는다.
1. 동일 태그+클래스 시그니처를 가진 형제가 N개(기본 ≥8) 이상인 컨테이너 후보 수집
2. 각 후보 안에서 회사명 후보(앵커 텍스트·heading), 링크(href→웹주소), 이미지(로고) 추출
3. 후보별 `ExtractionVerdict` 산정 → 가장 점수 높은 컨테이너 채택
4. 채택된 컨테이너의 셀렉터 경로를 `strategyConfig.listSelector`로 기록

### 6.3 칸 4 — AI 추출 (Gemini) — 진짜 자가 업그레이드

```
1. HTML 골격 압축
   - 텍스트 노드 제거, 반복 태그/클래스/속성 키만 보존
   - <script>의 inline JSON 후보는 별도 발췌 (data 배열 탐지)
   - 토큰 상한(예: 30k) 내로 트림, 초과 시 본문 영역만 잘라 전달
2. Gemini 프롬프트 (구조화 출력 강제)
   "이 전시회 페이지에서 참가업체 목록의 반복 단위 CSS 셀렉터와,
    각 단위 안의 회사명/홈페이지/이메일/부스/카테고리 필드 경로를 JSON으로.
    목록이 AJAX로 로드되면 호출 API URL 패턴과 파라미터도."
3. 응답 schema:
   { listSelector, fields:{회사명, 웹주소, 이메일, 부스번호, 카테고리}, apiHint }
4. 받은 설정으로 칸 3 추출기를 재실행 → ExtractionVerdict 검증
5. ok면 strategyConfig로 저장. fail이면 reasons를 다음 프롬프트에 피드백해 1회 재시도(자기수정 루프, 최대 2회)
```

**비용 통제:** AI는 칸 0~3이 모두 실패한 *처음 한 번*만 호출. 성공 설정은 ③에 캐시되어 재방문 시 칸 0에서 무료로 재사용.

### 6.4 칸 5 — 수동 캡처 (반자동 폴백)

세션/CSRF/봇차단으로 칸 0~4가 모두 실패하면, 사용자에게 **DevTools Network에서 목록 XHR의 URL·응답을 붙여넣는 UI**를 제시한다. 붙여넣은 응답을 정규화기로 흘리고, 추출 성공 시 그 API 패턴을 학습 프리셋으로 저장 → 다음부터는 자동.

## 7. 학습 프리셋 저장소 — `ExpoSiteLearner` (기둥 ③)

`PipelineLearningRegistry`의 학습 철학을 호스트 단위 *전략 설정*으로 확장.

```
namespace: "ocrpro.learn.expoSite.v1"
key: 호스트명 (예: "biz.smarttechkorea.com")
value: {
  strategy: "json_api" | "dom_mining" | "ai_selector" | "manual_api",
  listSelector: "div.exhibitor-card",
  apiUrl: "/biz/ajax_list.asp?page={page}",
  pageParam: "page",
  fieldMap: { 회사명:"...", 웹주소:"...", 이메일:"...", 부스번호:"...", 카테고리:"..." },
  source: "ai" | "discovery" | "manual" | "preset",
  verifiedAt: "2026-06-13",
  successCount: 3,        // 누적 성공 횟수 (신뢰도)
  lastVerdict: { ... }
}
```

정책:
- 칸 0 적중 시에도 `ExtractionVerdict`로 재검증 → 사이트 개편으로 `fail`나면 **학습 프리셋을 폐기하고 사다리를 다시 탄다**(자가 치유).
- `successCount`로 신뢰도 가중. 연속 실패 2회 시 자동 무효화.
- 사용자가 셀렉터를 수동 교정하면(향후 UI) 그 값을 우선 학습 — `OCRLearner` 교정 학습과 동일 철학.

## 8. 동적 · 세션 사이트 처리 정책

- Worker 프록시가 받는 HTML은 **JS 실행 전** 상태 → AJAX로 채워지는 목록은 칸 3·4가 봐도 비어 있다. → 동적 사이트는 **칸 2(API)·칸 5(수동)가 주력**, 칸 3·4는 보조.
- `jsessionid`·CSRF 토큰 사이트(예: Cosmobeauty)는 칸 2에서 토큰 전달이 필요. Worker가 Set-Cookie를 보존·재사용하도록 프록시 옵션 검토(별도 과제).
- 도달 불가 판정 시 억지로 추측 API를 난사하지 않고 칸 5로 떨어진다. (계획서 §11 "억지로 만들지 않는다" 원칙 계승)

## 9. 이메일 파이프라인(v9.60) 연계

사다리는 **목록+상세 수집까지만** 책임진다. 수집된 각 레코드는 기존 경로로 흘러간다.

```
ExpoSiteLadder (목록/상세 수집)
   → registerCompanies()
      → huntEmailForCompany() (v9.60)  // 무수정 재사용
         · scoreExpoEmailCandidate / applyExpoEmailResult
         · formatExpoEmailStats (확보율·도메인 일치율·접점 확보율)
```

신규 코드는 "수집 방식" 한 층뿐이며, 이메일 확보·점수·등급·리포트는 그대로다.

## 10. fixture · 회귀 테스트

`fixtures/expo_site_cases/` 신설. 사다리 각 칸을 검증하는 고정 fixture:

| ID | 상황 | 기대 사다리 칸 |
|---|---|---|
| S01 | 정적 목록 테이블 (NextRise형) | 칸 1 또는 칸 3 |
| S02 | inline JSON 내장 | 칸 2 |
| S03 | `.asp` AJAX 목록 (SmartTech형) | 칸 2 |
| S04 | `.do` SPA 해시라우팅 (Cosmobeauty형) | 칸 2 실패→칸 5 |
| S05 | 반복 카드 div, 클래스 불규칙 | 칸 3 (DOM 마이닝) |
| S06 | 셀렉터 추론 필요한 비정형 | 칸 4 (AI, mock 응답) |
| S07 | 0건 (빈 목록) | ExtractionVerdict=fail 정확 판정 |
| S08 | garbage 과다(메뉴를 회사명으로) | verdict=fail, 격상 트리거 |
| S09 | 학습 프리셋 적중 후 사이트 개편 | 칸 0 재검증 실패→폐기→재탐색 |
| S10 | 동적 목록인데 정적 HTML 비어있음 | 칸 3 skip, 칸 2/5 |

`verify-expo-site-ladder.html` 회귀 실행 도구를 `verify-expo-email-fixtures.html`과 동일 패턴으로 제작. AI 칸(S06)은 mock 응답으로 결정론적 테스트.

## 11. 리스크와 한계 (정직한 한계)

1. **①(판정 기준)이 전부다.** 임계값이 느슨하면 자가 업그레이드가 안 켜지고, 빡빡하면 정상 추출을 실패로 오인해 무한 격상한다. → fixture S07·S08로 임계 보정 필수.
2. **AI 환각·비용·지연.** 잘못된 셀렉터를 줄 수 있어 반드시 `ExtractionVerdict`로 검증 후 채택. 비용은 칸 0 캐시로 통제.
3. **JS-전 HTML 한계.** 진정한 클라이언트 렌더 SPA는 Worker가 헤드리스 브라우저가 아닌 한 목록을 못 본다 → 칸 2/5 의존. (헤드리스 렌더링은 본 계획 범위 밖, 향후 과제)
4. **세션·봇차단은 지능으로 못 뚫는다.** 칸 5로 떨어지는 것이 정답이며 실패가 아니다.
5. **회귀 격리.** 모든 신규 코드는 전시회 경로(`registerCompanies` 상류)에만 연결. 명함·QR·문서·URL 진입 함수·공통 `emptyRec`/`pushRecord`는 불변.

## 12. 구현 단계

### Phase A1. 추출 판정 엔진 (기둥 ①) — 최우선
- `computeExtractionVerdict(companies, ctx)` 구현
- `registerCompanies()`에 진입/완료 판정 연결
- 완료 토스트·결과 패널에 verdict 요약 노출, `fail` 경고
- **완료 기준:** S07·S08 fixture 통과, 0건 완료의 조용한 실패 제거

### Phase A2. 사다리 골격 + 기존 칸 흡수 (기둥 ②)
- `ExpoSiteLadder.run(url)` 오케스트레이터 (칸 0~5 폴백 루프)
- 칸 1(`EXPO_SITE_PRESETS`)·칸 2(`tryDiscoverExhibitorAPI`)를 사다리 인터페이스로 래핑
- 칸 2 action 치환·페이지 파라미터 자동탐색 확장
- **완료 기준:** S02·S03 통과, 기존 KIMES/SIMTOS/COEX 회귀 무손상

### Phase A3. DOM 패턴 마이닝 (칸 3)
- 반복 형제 군집 탐지 + 컨테이너 점수화
- **완료 기준:** S01·S05 통과 (정적 사이트 무코딩 추출)

### Phase A4. 학습 프리셋 영속화 (기둥 ③)
- `ExpoSiteLearner` localStorage 스키마 + load/save/invalidate
- 칸 0(학습 적중) + 재검증·자가치유(S09)
- **완료 기준:** S09 통과, 2회 방문 시 2번째가 칸 0에서 즉시 성공

### Phase A5. AI 추출 (칸 4) + 수동 캡처 (칸 5)
- Gemini 골격 압축·구조화 프롬프트·자기수정 루프(최대 2회)
- 수동 XHR 붙여넣기 UI → 정규화 → 학습 저장
- **완료 기준:** S06(mock) 통과, 미지 정적 사이트 1건 실사이트 추출 성공

## 13. 성공 기준

1. 처음 보는 전시회 사이트(정적)를 손코딩 없이 추출한다. (칸 3/4)
2. 한 번 뚫은 사이트는 다음 방문 시 학습 프리셋(칸 0)으로 즉시 처리된다.
3. 0건·garbage 등 조용한 실패가 모두 `ExtractionVerdict`로 드러난다.
4. 사이트 개편으로 학습 프리셋이 깨지면 자동으로 폐기·재탐색한다.
5. 도달 불가 사이트는 억지 추측 없이 수동 캡처로 우아하게 떨어진다.
6. 이메일 파이프라인(v9.60)과 타 입력 파이프라인은 무수정·무영향.

## 14. 최종 권고

- **A1(판정 엔진)을 단독으로 먼저 배포**해도 즉시 가치가 있다. 지금의 "0건인데 완료" 류 조용한 실패를 없애기 때문이다.
- 지금 손으로 진행 중인 **NextRise/SmartTech/Cosmobeauty 어댑터 작업이 그대로 칸 1의 데이터이자 fixture S01·S03·S04의 근거**가 된다. 이 3개를 처리하며 "어떤 신호로 성공/실패를 판정했는지"를 A1 임계값으로 역산하면 엔진의 씨앗이 된다.
- AI 칸(A5)은 가장 강력하지만 비용·환각 리스크가 있으므로 **판정 엔진(A1)과 학습 캐시(A4)가 선행된 뒤** 도입한다.

권장 구현 순서: **A1 → A2 → A3 → A4 → A5** (기둥 ① → ② → ③ → AI 순).
