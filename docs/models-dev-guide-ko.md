# models.dev 전수조사 & 활용 가이드 (한국어 정리)

> 카리나랑 같이 분석한 대화 내용을 한 파일로 정리한 문서예요. ✨

## 🔗 깃허브 / 관련 주소

| 구분 | 주소 |
| --- | --- |
| 내 포크(이 저장소) | https://github.com/bmshin94/models.dev |
| 원본(업스트림) 저장소 | https://github.com/anomalyco/models.dev |
| 공식 웹사이트 | https://models.dev |
| 전체 데이터 API | https://models.dev/api.json |
| 모델 전용 메타데이터 | https://models.dev/models.json |
| 둘 다 합친 카탈로그 | https://models.dev/catalog.json |
| 공급사 로고 | `https://models.dev/logos/{provider}.svg` |
| 공식 npm SDK | `@opencode-ai/models` (https://www.npmjs.com/package/@opencode-ai/models) |
| 이 DB를 쓰는 대표 서비스 | https://opencode.ai |

---

## 1. 이게 뭐하는 저장소야? (전수조사 결과)

**한 줄 요약:** 세상에 나와 있는 AI 모델들의 **스펙 · 가격 · 기능**을 한곳에 모아둔 **오픈소스 "AI 모델 백과사전 + 가격표" 데이터베이스**예요.

- 제작: SST 팀(현재 Anomaly, 코딩 에이전트 **opencode** 만든 팀)
- 라이선스: **MIT** (상업적 이용 가능, 저작권 표시만 유지)
- 조사 시점 규모: 공급사(provider) 폴더 **222개**, 공급사별 모델 TOML **약 8,100개**, 모델 원본 메타데이터 TOML **약 400개**, 연구소(lab) **23곳**

### 폴더 구조와 역할

| 경로 | 역할 | 비유 |
| --- | --- | --- |
| `providers/` | 모델을 **파는 곳(API 공급사)** 별 폴더. `provider.toml`(이름, 필요한 API 키 환경변수명, 문서 링크, API 주소) + `logo.svg` + `models/*.toml`(가격·한도) | 가게별 메뉴판 |
| `models/` | 모델을 **만든 곳** 기준의 "순수 스펙"(출시일, 컨텍스트 길이, 멀티모달 여부, 벤치마크 등) | 제품 설명서 |
| `labs/` | 모델 제작사(Anthropic, OpenAI, Google, DeepSeek…) 소개문 + 로고 | 제조사 소개 |
| `packages/core` | TOML 검증 스키마(Zod), JSON 생성기, 공급사 API에서 모델 목록 자동 동기화(sync) 스크립트 | 데이터 공장 |
| `packages/web` | models.dev 웹사이트(Bun + JSX 서버 렌더링) 빌드 | 쇼룸 |
| `packages/function` | Cloudflare Worker. 정적 JSON/웹 제공, 사용 통계 수집 | 매장 입구 |
| `packages/sdk` | 공식 TypeScript 클라이언트 `@opencode-ai/models` (+ 오프라인 스냅샷, Effect 버전) | 배달 앱 |
| `.github/workflows` | 검증, 배포, 모델 자동 동기화, AI 기반 PR 리뷰/이슈 자동 수정 | 자동화 직원 |
| `.opencode/` | 이 저장소 **관리용** opencode AI 에이전트(PR 리뷰어, CI 수정봇, 이슈 수정봇) + 스킬 | 사내 AI 직원 |
| `AGENTS.md`, `sync.md` | 기여 규칙, 동기화 규칙 문서 | 업무 매뉴얼 |
| `models.json` (루트) | OpenRouter 원본 카탈로그 스냅샷(동기화 입력 데이터) | 원자재 |
| `sst.config.ts` | Cloudflare 배포 설정 | 배송 설정 |

### 동작 흐름

```
사람/봇이 TOML 수정 → bun validate(스키마 검증) → 빌드(TOML → JSON 합치기)
→ Cloudflare 배포 → models.dev/api.json 으로 누구나 조회
```

- **상속 구조(`base_model`)**: `models/anthropic/claude-opus-4-6.toml`에 모델 공통 스펙을 두고, 각 공급사 파일(예: OpenRouter, Bedrock)은 `base_model = "anthropic/claude-opus-4-6"` + **자기 가격/차이점만** 적어요. 중복을 최소화하는 설계.
- **자동 동기화**: OpenRouter, Kilo, DeepInfra 등 공급사 API를 주기적으로 긁어서 새 모델/가격 변경을 자동 PR로 올림.

### 모델 한 개에 담긴 정보 (예시)

- 이름, 계열(family), 출시일, 지식 컷오프
- 추론(reasoning) / 도구호출(tool_call) / 구조화 출력 / 파일첨부 지원 여부
- 컨텍스트·입력·출력 토큰 한도
- 입력/출력 모달리티(텍스트, 이미지, PDF, 오디오, 비디오)
- 가격(100만 토큰당 USD): 입력, 출력, 캐시 읽기/쓰기, 오디오 등
- 오픈 웨이트 여부, 라이선스, 가중치 링크, 벤치마크 점수
- 추론 강도 옵션(`reasoning_options`: low/medium/high 등)

### 언제 쓰나?

- "이 모델 컨텍스트 몇 토큰이지?", "어디가 제일 싸지?" 확인할 때
- 내 서비스/에이전트에 **모델 선택 드롭다운**, **요금 계산기**, **토큰 한도 체크**를 넣을 때
- 여러 공급사 모델을 한 앱에서 바꿔 쓰는 **멀티 프로바이더** 앱을 만들 때 (AI SDK 모델 ID와 호환)

### 나(우리)한테 도움 되는 점

1. 모델 스펙/가격을 일일이 찾아다닐 필요 없음 → **API 한 번 호출**로 해결
2. 비용 계산기, 모델 비교, 자동 모델 선택 로직을 **공짜 데이터로** 구현 가능
3. 로컬 에이전트/웹서비스에서 "한도 초과 전 자동 요약", "싼 모델로 폴백" 같은 기능 구현 가능
4. MIT라 이 데이터를 기반으로 **상업 서비스**도 만들 수 있음

---

## 2. 더 쉽게 설명하면 🍔

- models.dev = **AI 모델계의 "다나와 / 네이버 가격비교"**
- `labs/` = 제조사 (삼성, 애플 같은 곳 = Anthropic, OpenAI)
- `models/` = 제품 설명서 (이 폰은 램 8GB, 카메라 몇 화소 = 이 모델은 컨텍스트 20만 토큰, 이미지 입력 가능)
- `providers/` = 판매처 (쿠팡, 11번가 = OpenRouter, AWS Bedrock). 같은 제품도 판매처마다 가격이 다름!
- `packages/` = 이 정보를 웹사이트/API로 보여주는 **프로그램 코드**
- `.github/` + `.opencode/` = 가격 바뀌면 알아서 업데이트하는 **자동화 로봇 직원들**

**비유로 정리:** 우리가 AI 앱을 만들 때 "어떤 모델 쓸까? 얼마 나올까? 이미지 넣어도 되나?"를 고민하는데, 이 저장소가 그 답을 **표 한 장(JSON)** 으로 줌.

**이런 순간에 딱!**
- "GPT랑 Claude 중 뭐가 더 싸?" → 가격표 비교
- "이 문서 30만 토큰인데 넣을 수 있나?" → `limit.context` 확인
- "사진 분석되는 모델만 보여줘" → `modalities.input`에 `image` 있는 모델 필터

---

## 3. Q&A 모음

### 3-1. 설치 및 사용법

**A. 설치 없이 바로 쓰기 (제일 쉬움)**

```bash
curl https://models.dev/api.json      # 공급사별 전체 데이터
curl https://models.dev/models.json   # 모델 자체 스펙만
curl https://models.dev/catalog.json  # 둘 다
```

**B. JavaScript/TypeScript SDK**

```bash
npm install @opencode-ai/models
```

```ts
import { Models } from "@opencode-ai/models"
const client = Models.make()
const providers = await client.providers()
providers["anthropic"]?.models["claude-opus-4-6"]?.cost?.input // 100만 토큰당 USD

// 인터넷 없이 쓰는 내장 스냅샷
import { providers as offline } from "@opencode-ai/models/snapshot"
```

**C. 저장소를 로컬에서 직접 돌리기** (Bun 필요: https://bun.sh)

```bash
git clone https://github.com/bmshin94/models.dev
cd models.dev
bun install
bun validate                 # 전체 TOML 검증 (조사 시점 통과 확인 ✅)
cd packages/web && bun run dev   # http://localhost:3000 에서 사이트 실행
```

**D. 모델/공급사 추가 기여**: `providers/<id>/provider.toml` + `logo.svg` + `models/<모델id>.toml` 작성 → `bun validate` → PR

### 3-2. 플러그인이야? 스킬이야? MCP야?

**셋 다 아님!** 이건 **오픈 데이터베이스(데이터 + 웹사이트 + 공개 JSON API + npm SDK)** 예요.

- `.opencode/` 안의 agent/skill은 이 저장소 **관리자들이 쓰는 내부 자동화용**이지, 설치해서 쓰는 플러그인이 아님
- 대신 **우리가 이걸 감싸서 MCP 서버나 Claude 스킬로 만들 수는 있음** (예: `get_model_price`, `find_cheapest_model` 툴을 제공하는 MCP)

### 3-3. API 토큰이 필요해?

**models.dev 자체는 필요 없음.** 로그인/키 없이 누구나 무료로 JSON 조회 가능.

- `provider.toml`의 `env = ["ANTHROPIC_API_KEY"]` 같은 건 "이 공급사 모델을 **실제로 호출할 때** 필요한 환경변수 이름"을 알려주는 정보일 뿐
- 실제 AI 모델을 부를 땐 당연히 각 공급사 키가 필요 (models.dev와는 별개)
- 일부 자동 동기화 스크립트는 공급사 API 접근이 필요할 수 있지만, 일반 사용자는 신경 안 써도 됨
- 참고: `opencode`/`bun` User-Agent 요청은 사용 통계(PostHog 등)로 집계됨

### 3-4. 왜 깃허브에서 유명할까?

1. **빈자리를 채움** — 모든 AI 모델 정보를 모은 단일 DB가 없었음
2. **opencode(인기 오픈소스 코딩 에이전트)가 실제로 사용** → 실사용 검증 + 노출
3. **SST 팀 브랜드** (유명 인프라 도구 제작팀)
4. **무료 + 인증 없는 API**, MIT 라이선스
5. **AI SDK(Vercel) 모델 ID와 호환** → JS 생태계에서 바로 쓰기 좋음
6. **규모와 신선도** — 200개 넘는 공급사, 수천 개 모델, 봇이 매일 자동 동기화
7. **커뮤니티 기여 구조** — TOML 파일 하나만 고치면 누구나 기여 가능 + AI 봇이 PR 리뷰

### 3-5. 로컬 에이전트 구축에 도움이 될까?

**도움 됨 (단, 모델을 돌려주는 도구는 아님!)** 모델 실행은 Ollama/LM Studio가 하고, models.dev는 **"모델 정보 두뇌"** 역할.

- `providers/lmstudio` → `api = "http://127.0.0.1:1234/v1"` (로컬 LM Studio 연결 정보 포함)
- `providers/ollama-cloud`, 오픈웨이트 모델(`open_weights = true`)의 가중치 링크/라이선스 확인 가능
- 활용 예:
  - 컨텍스트 한도 보고 자동으로 대화 요약/잘라내기
  - 작업 난이도에 따라 로컬 모델 ↔ 클라우드 모델 **자동 라우팅**
  - 도구호출(`tool_call`) 지원 모델만 에이전트 후보로 필터
  - 비용 추적 대시보드
- 오프라인 환경: SDK 스냅샷(`@opencode-ai/models/snapshot`) 또는 `api.json`을 로컬에 저장
- opencode에 로컬 빌드 데이터 연결: `OPENCODE_MODELS_PATH="dist/_api.json" opencode`

### 3-6. React나 PHP로 만들 수 있어?

**당연히 가능!** 데이터가 그냥 JSON이라 언어 상관없음.

**React (모델 비교표)**

```tsx
import { useEffect, useState } from "react"

export default function ModelTable() {
  const [rows, setRows] = useState<any[]>([])
  useEffect(() => {
    fetch("https://models.dev/api.json")
      .then((r) => r.json())
      .then((data) => {
        const list = Object.entries<any>(data).flatMap(([pid, p]) =>
          Object.values<any>(p.models).map((m) => ({ provider: pid, ...m })),
        )
        setRows(list.filter((m) => m.cost).sort((a, b) => a.cost.input - b.cost.input))
      })
  }, [])
  return (
    <table>
      <tbody>
        {rows.slice(0, 50).map((m) => (
          <tr key={m.provider + m.id}>
            <td>{m.provider}</td><td>{m.name}</td>
            <td>${m.cost.input}</td><td>${m.cost.output}</td><td>{m.limit?.context}</td>
          </tr>
        ))}
      </tbody>
    </table>
  )
}
```

**PHP (캐시해서 비용 계산)**

```php
<?php
$cache = __DIR__ . '/api.json';
if (!file_exists($cache) || time() - filemtime($cache) > 86400) {
    file_put_contents($cache, file_get_contents('https://models.dev/api.json'));
}
$data  = json_decode(file_get_contents($cache), true);
$model = $data['anthropic']['models']['claude-opus-4-6'];
$cost  = (100000 / 1e6) * $model['cost']['input'] + (20000 / 1e6) * $model['cost']['output'];
echo "{$model['name']}: 입력 10만 + 출력 2만 토큰 = \$" . round($cost, 4);
```

- 권장: 매 요청마다 부르지 말고 **하루 1번 캐시** (파일/DB/Redis)
- 저장소 자체(TOML → JSON 빌드)를 PHP로 재구현도 가능하지만, 공식 JSON을 **가져다 쓰는 게 훨씬 효율적**

---

## 4. 수익화 아이디어 💰

> 전제: 데이터는 MIT라 상업 이용 OK. 출처(models.dev) 표기 권장. 데이터 자체는 무료로 누구나 받을 수 있으니, **돈은 "편의·가공·현지화·알림·자동화"에서** 나온다는 점이 핵심!

| # | 아이디어 | 타깃 | 수익 모델 | 난이도 | 기술 |
| --- | --- | --- | --- | --- | --- |
| 1 | **한국어 AI 모델 가격비교/요금계산기 사이트** (원화 환산, 부가세 포함, "월 예상 비용" 계산) | 국내 개발자·기획자·스타트업 | 애드센스, 공급사 제휴(레퍼럴) 링크, 프리미엄 | ⭐ 쉬움 | React/Next.js 또는 PHP + SEO |
| 2 | **가격 변동·신모델 알림 서비스** (저장소 커밋/`api.json` diff 감지 → 카톡/슬랙/이메일) | 비용 민감 팀, CTO | 월 구독 (예: 4,900~19,900원) | ⭐⭐ | 크론 + diff + 알림 API |
| 3 | **LLM 비용 최적화 SaaS** — 사용 로그 업로드 → "이 작업은 더 싼 모델로 바꾸면 월 ○○만원 절약" 리포트 | AI 쓰는 중소기업 | 절감액 %·구독, 컨설팅 | ⭐⭐⭐ | 백엔드 + 분석 |
| 4 | **스마트 모델 라우터/게이트웨이** — 요청 난이도·컨텍스트 길이·예산 보고 자동으로 모델 선택 | 에이전트/챗봇 개발사 | 사용량 과금, 셀프호스팅 라이선스 | ⭐⭐⭐⭐ | 프록시 서버 |
| 5 | **models.dev MCP 서버 / Claude 스킬** ("가장 싼 비전 모델 찾아줘") | AI 에이전트 사용자 | 오픈소스 + 호스팅 유료판, 브랜딩 → 외주 유입 | ⭐⭐ | MCP SDK |
| 6 | **React 컴포넌트 키트** (모델 선택 드롭다운, 비용 미터, 토큰 한도 게이지) | AI 앱 개발자 | 유료 템플릿/라이선스, 깃허브 스폰서 | ⭐⭐ | React/npm |
| 7 | **워드프레스/PHP 플러그인** (블로그 AI 글쓰기 시 모델 선택 + 비용 표시) | 워드프레스 운영자 | 프리미엄 플러그인 | ⭐⭐ | PHP |
| 8 | **월간 "AI 모델 시장 리포트" 뉴스레터/전자책** (가격 추이, 신모델 스펙 비교, 한국어 해설) | 기획자·투자자·교육기관 | 유료 구독·기업 판매 | ⭐ | 데이터 분석 + 글쓰기 |
| 9 | **기업용 AI 도입 견적 컨설팅** (요구사항 → 모델 조합 + 월 비용 견적서 자동 생성) | 국내 기업 | 건당 컨설팅/견적 수수료 | ⭐⭐ | 문서 자동화 |
| 10 | **교육 콘텐츠** ("멀티 모델 AI 앱 만들기" 강의, 이 저장소 실습) | 입문 개발자 | 강의 판매 (인프런/클래스101) | ⭐ | 강의 제작 |

### 추천 실행 순서 (작게 시작 → 키우기)

1. **1주차 — 한국어 가격비교/계산기 사이트(#1)** 오픈: 트래픽·SEO 자산 확보 (React 정적 사이트 or PHP, 하루 1회 캐시)
2. **2~4주차 — 알림 기능(#2)** 추가: 이메일 수집 → 유료 전환 퍼널
3. **1~2개월 — MCP 서버/React 키트(#5, #6)** 오픈소스 공개: 깃허브 스타·브랜딩 → 외주/컨설팅(#9) 유입
4. **3개월~ — 비용 최적화 SaaS/라우터(#3, #4)**: 쌓인 사용자 기반으로 본격 B2B 수익

### 주의할 점

- 데이터는 누구나 무료로 받을 수 있음 → **데이터 재판매만으로는 경쟁력 약함**, 가공/현지화/자동화에 가치를 두기
- 가격 정보는 커뮤니티/봇 기반이라 **오류 가능성** → "참고용, 공식 문서 확인" 안내 필수
- 공급사 레퍼럴 프로그램은 공급사마다 조건이 다르니 개별 확인
- MIT 라이선스 고지(LICENSE) 유지, 출처 표기

---

## 5. 작업 기록

- 이 문서: `docs/models-dev-guide-ko.md`
- 작업 브랜치: `claude/serene-goldberg-gkwn1y` → `dev` 브랜치로 PR 후 머지
- 조사 시 `bun install && bun validate` 실행 → 검증 통과 확인 ✅
