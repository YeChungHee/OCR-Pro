# OCR Pro 전시회 엔진 고도화 계획서

**작성일:** 2026-04-25  
**현재 버전:** v9.10  
**기준 학습:** Franchise COEX 추출 파이프라인 보고서 (52개 기업, 2026-04-25)

---

## 1. 현재 상태 요약 (v9.8 완료 기준)

### 완료된 항목
| 항목 | 설명 |
|------|------|
| ✅ BBS 목록 파서 | `parseBbsListPage()` — td.subject + ptype=view 구조 감지 및 파싱 |
| ✅ BBS 상세 파서 | `parseBbsDetailPage()` + `extractBbsField()` — table.bbs_view th+td 구조 |
| ✅ URL 정규화 | `normalizeHomepageHtml()` — href-in-text 비정형 URL 처리 |
| ✅ 자동 감지 | `isBbsPage()` — ptype=view 감지 시 BBS 파서 우선 적용 |
| ✅ 프리셋 추가 | 프랜차이즈 COEX 빠른 선택 버튼 |

### 현재 파싱 가능 포맷
```
URL 입력 → 포맷 감지
  ├─ .pdf          → PDF 구조화 파서 → (실패 시) Gemini Vision
  ├─ 이미지 URL    → Gemini Vision
  ├─ BBS 게시판형  → parseBbsListPage → parseBbsDetailPage (NEW v9.8)
  ├─ JSON API      → tryDiscoverExhibitorAPI
  └─ 일반 HTML     → parseExhibitorTable → (실패 시) Gemini AI
```

---

## 2. 고도화 방안 전체 로드맵

### Tier 1 — 즉시 적용 가능 (고영향, 낮은 공수)

#### 2-1. 병렬 상세 페이지 조회 (mapLimit 방식)
**현황:** 현재 상세 페이지 조회가 순차 처리 (1개씩 fetch → sleep)  
**문제:** 52개 기업 × 400ms = 약 21초 소요  
**개선:** 6개 동시 요청 → 약 4초로 단축 (80% 성능 향상)

```javascript
// 구현 위치: registerCompanies() 함수 내
// 현재: 순차 for 루프
// 개선: mapLimit(allCompanies, 6, async (comp) => {...})

async function mapLimit(items, limit, mapper) {
  const results = [];
  for (let i = 0; i < items.length; i += limit) {
    const batch = items.slice(i, i + limit);
    const batchResults = await Promise.all(batch.map(mapper));
    results.push(...batchResults);
    updateExpoProgress(`📦 배치 ${Math.floor(i/limit)+1} 처리 중...`, i, items.length);
  }
  return results;
}
```

**적용 조건:** BBS 사이트 감지 시에만 활성화 (일반 사이트는 기존 순차 유지)

---

#### 2-2. 사이트 구조 프리셋 라이브러리
**현황:** 매번 URL을 입력하고 구조를 자동 감지해야 함  
**개선:** 주요 전시회 사이트별 파싱 설정을 코드에 내장

```javascript
const EXPO_SITE_PRESETS = {
  'franchisecoex.co.kr': {
    name: '프랜차이즈 COEX',
    type: 'bbs',
    listUrl: 'https://www.franchisecoex.co.kr/visit/join_list.php',
    pageParam: 'page',
    pageParamSuffix: '&code=join_list',
    detailFields: ['부스번호','브랜드명','품목','홈페이지 주소','회차'],
  },
  'simtos.org': {
    name: 'SIMTOS',
    type: 'html_table',
    listUrl: 'https://simtos.org/kor/exhibitors/exhibitor_list.do',
    pageParam: 'pageIndex',
  },
  'kimes.kr': {
    name: 'KIMES',
    type: 'json_api',
    apiHint: '/api/exhibitors',
  },
};
```

**효과:** 구조 감지 시간 0, 파싱 정확도 향상

---

#### 2-3. 구조 분석(🔍) 결과 개선
**현황:** "구조 분석" 버튼 결과가 단순 통계만 표시  
**개선:** BBS 사이트 감지 시 상세 정보 표시

```
✅ BBS 게시판형 구조 감지
📄 페이지 파라미터: page (code=join_list 추가 감지)
📋 총 페이지 수: 3 (목록 22개 기업 × 예상 52개)
🔗 상세 페이지: ✅ table.bbs_view 구조 (부스번호·홈페이지 추출 가능)
📦 예상 추출 필드: 부스번호, 브랜드명, 품목, 홈페이지, 회차
```

---

### Tier 2 — 단기 적용 (중간 영향, 중간 공수)

#### 2-4. 사업자번호 자동 보강 배치
**현황:** 전시회 수집 기업은 사업자번호 없음  
**개선:** 수집 완료 후 홈페이지 URL → 사업자번호 자동 조회 배치 처리

```
[🏛 전시회 수집 완료: 52개]
  ↓
[💼 사업자번호 자동 보강 시작]
  ├─ 홈페이지 있는 기업: 46개 → URL탭 fetchWebpageHTML + seocho API 조회
  └─ 홈페이지 없는 기업: 6개 → 국세청 API 기업명 검색 (선택적)
```

**구현 위치:** `registerCompanies()` 완료 후 별도 배치 함수 `enrichBiznoFromExpo()`

---

#### 2-5. 세일즈맵 엑셀 영문명 컬럼 추가
**현황:** 영문명(GREEK BERRY 등)이 추출되지만 엑셀에 미반영  
**개선:** `exportExcel()` 함수에 영문명 컬럼 추가

```
현재 열 순서: 회사명, 담당자, 부스번호, 업종, 웹주소, ...
개선 열 순서: 회사명, 영문명, 담당자, 부스번호, 업종, 웹주소, ...
```

**구현:** `emptyRec()`에 `영문명` 필드 추가 + BBS 파서에서 영문명 설정 + 엑셀 컬럼 매핑

---

#### 2-6. 전시회 회차/연도 메타 자동 기록
**현황:** 전시회명·회차 정보가 추출되어도 저장 위치 없음  
**개선:** `부스번호` 필드 포맷 변경: `"J117"` → `"84회 J117"` (회차 포함)  
또는 별도 `전시회명` 필드 추가

```
현재: rec.부스번호 = "J117"
개선: rec.부스번호 = "84회 J117"  // 회차+부스번호 통합
rec.업종 = "프랜차이즈 창업박람회 | 제과/제빵/디저트"  // 전시회+카테고리
```

---

### Tier 3 — 중장기 적용 (높은 가치, 높은 공수)

#### 2-7. 새 전시회 사이트 구조 자동 학습
**개념:** 처음 파싱한 전시회 사이트의 구조를 localStorage에 저장 → 다음 번에 재사용

```javascript
// 구조 캐시 저장
function saveExpoStructureCache(url, structureInfo) {
  const cache = JSON.parse(localStorage.getItem('expoCaches') || '{}');
  cache[new URL(url).hostname] = { ...structureInfo, savedAt: Date.now() };
  localStorage.setItem('expoCaches', JSON.stringify(cache));
}
// 다음 방문 시 자동 적용
function loadExpoStructureCache(url) {
  const cache = JSON.parse(localStorage.getItem('expoCaches') || '{}');
  return cache[new URL(url).hostname];
}
```

**효과:** 동일 전시회 반복 방문 시 구조 감지 단계 스킵

---

#### 2-8. 다회차 참가 이력 추적
**현황:** 매 추출 시 레코드가 독립적으로 생성됨  
**개선:** 동일 기업명 + 다른 회차 감지 시 "반복 참가" 표시

```
그릭베리 (84회 J117) → 중복 감지 → 83회에도 참가 이력 존재
  → 신뢰도 점수 +10 (반복 참가 기업)
  → 메모: "83·84회 연속 참가"
```

---

#### 2-9. 다양한 한국 전시회 사이트 지원 확장

현재 OCR Pro가 지원하지 못하거나 제한적으로 지원하는 주요 사이트:

| 사이트 | URL 패턴 | 현재 상태 | 고도화 방안 |
|--------|----------|-----------|-------------|
| 코엑스 (각 전시회) | cdn.ems.coex.co.kr/...pdf | ✅ PDF 파서 | 구조화 파서 정확도 향상 |
| 킨텍스 | kintex.com | ⚠️ 부분 지원 | 킨텍스 전용 BBS 파서 |
| KOTRA 무역관 | okta.or.kr | ❌ 미지원 | JSON API 탐색 필요 |
| 농식품부 전시회 | mafra.go.kr | ❌ 미지원 | 정부 표준 BBS 파서 |
| 소상공인 박람회 | seda.or.kr | ❌ 미지원 | 게시판형 파서 적용 |
| 한국프랜차이즈산업협회 | ikfa.or.kr | ❌ 미지원 | BBS 파서 적용 가능 |

---

## 3. 적용 우선순위 일정

### Phase 1: 즉시 (현 세션)
- [x] v9.8 BBS 파서 구현 완료
- [x] v9.9 병렬 상세 조회 + 프리셋 라이브러리 + 영문명 컬럼
- [x] v9.10 BBS maxPage 정확 감지 (detectBbsMaxPage — 페이지네이션/상세링크 구분)
- [ ] **git push** (Mac에서 git-push.command 더블클릭)

### Phase 2: 단기 (다음 1~2 세션)
- [ ] 2-1: mapLimit 병렬 상세 조회 (BBS 전용)
- [ ] 2-2: 사이트 프리셋 라이브러리 내장
- [ ] 2-3: 구조 분석 결과 고도화

### Phase 3: 중기 (3~5 세션)
- [ ] 2-4: 사업자번호 자동 보강 배치
- [ ] 2-5: 영문명 컬럼 추가
- [ ] 2-6: 회차/전시회명 메타 기록

### Phase 4: 장기 (기능 안정화 후)
- [ ] 2-7: 사이트 구조 자동 학습 (localStorage 캐시)
- [ ] 2-8: 다회차 참가 이력 추적
- [ ] 2-9: 킨텍스 등 추가 사이트 지원

---

## 4. 핵심 구현 파일

| 파일 | 역할 |
|------|------|
| `/Users/appler/Documents/Claude/Projects/OCR_pro/ocr-pro-v51.html` | 메인 앱 (단일 파일) |
| `/Users/appler/Documents/Claude/Projects/OCR_pro/index.html` | GitHub Pages 서빙 |
| `/Users/appler/Documents/Claude/Projects/OCR_pro/git-push.command` | Mac에서 더블클릭 → git push |

---

## 5. Codex 파이프라인 학습 핵심 요약

Franchise COEX 보고서에서 학습한 재사용 가능한 패턴:

1. **2단계 파이프라인이 기본**: 목록(링크 수집) → 상세(필드 보강)
2. **Gemini 없이도 가능**: 순수 정규식 HTML 파싱으로 52개 기업 100% 추출
3. **홈페이지 정규화 필수**: `http://<a href=...>` 비정형 패턴이 실전에서 자주 등장
4. **JSON 먼저 저장**: 중간 산출물 보존으로 엑셀 생성 실패 시 복구 가능
5. **병렬 6개 제한**: 서버 부하와 속도 사이 최적 균형
6. **`th+td` 패턴**: 한국 표준 BBS 상세 페이지의 범용 필드 추출 방법

---

*이 계획서는 OCR Pro v9.8 기준으로 작성되었으며, 버전 업데이트에 따라 자동으로 갱신됩니다.*
