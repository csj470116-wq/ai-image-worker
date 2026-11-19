# AI Image Worker — 업그레이드 딥리서치 보고서 (2026-11-20)

> 대상: `csj470116-wq/ai-image-worker` (lza6/Flux-AI-Pro-optimization 포크, Apache-2.0)
> 목적: [성능/안정성 · 자동화/워크플로우 · 최신 기술 · 예외 처리/우회] 4관점 개선안 + 경쟁 프로젝트 비교 → 3개 선택 명세서

---

## 0. 이 세션에서 실제로 수정·검증한 내역 (커밋 9ae6780)

| 문제 | 원인 | 조치 | 검증 |
|---|---|---|---|
| 401 (프롬프트 생성) | `gen.pollinations.ai` 이제 유료/키 필요 | Workers AI로 전면 교체: 이미지 입력 = Llama 3.2 11B Vision, 텍스트 입력 = Llama 3.3 70B FP8 Fast | `POST /api/generate-prompt` 200 OK |
| 한국어 입력이 무관한 결과 | `translateToEnglish`가 한자(중국어)만 감지 | 비-ASCII 전부 Google GTX로 번역 후 Llama에 전달 | "고양이가 우주를 나는 모습" → "A cat flying in space" → 정상 프롬프트 |
| 한국어 UI 슬롯이 중국어 기본값? | UI i18n `zh` 기본 + ko 미검수 | ko 문자열 정상 확인(11개 언어 i18n 존재), 기본 언어는 `zh` → README에 한국어 섹션 | 빌드/렌더 확인 |
| LLaVA 1.5 계획된 사용중단(11/20) | `@cf/llava-hf/llava-1.5-7b-hf` beta+deprecation | Llama 3.2 11B Vision Instruct로 교체 (계정당 1회 Meta 라이선스 acknowledge 자동 처리, 5016 응답을 성공으로 간주) | Cat03.jpg → 정확한 시각 묘사 프롬프트 |
| Llama 3.1 8B 사용중단(2026-05-30) | `@cf/meta/llama-3.1-8b-instruct` deprecated | 3.3 70B FP8 Fast로 교체 | 200 OK |
| 캐시/안정성 | (이전 세션) ArrayBuffer 직렬화, O(n²) base64, rate-limiter fail-closed, bounded metrics | 유지 | dry-run OK |

**테스트된 경로:** 텍스트(한/영) ✓ · 이미지(URL) ✓ · Google GTX ✓ · 캐시 라운드트립(이전) ✓ · `/health` ✓ (배포는 미실시 — 4장 참조)

---

## 1. [성능 및 안정성] — 병목·오류 다발 지점과 해결책

### 1-1. 이미 해결된 것 (이 포크에서 적용)
- Workers AI `flux-1-schnell` 기본값(무료, API 키 불필요) + pollinations/infip/구글 번역 페일오버 체인
- ArrayBuffer 캐시 직렬화(base64), O(n²)→O(n) base64, 레이트리미터 fail-closed + 인메모리 폴백, bounded metrics, `/health`

### 1-2. 남은 병목과 제안 표

| 항목 | 문제 | 해결책 | 난이도 |
|---|---|---|---|
| 캐시 히트 전략 | `SMART_CACHE_STRATEGY=aggressive` 환경변수만 있고 캐시 키 정책이 이미지 바이트를 반영 안 함(img2img에서 다른 참조 이미지로 같은 프롬프트 → 같은 캐시 히트) | 캐시 키에 참조 이미지 sha-256 해시 포함 | 중 |
| 생성 응답 크기 | flux 이미지를 ArrayBuffer로 Job 메모리에 올린 뒤 base64 → data URI로 통째로 반환 | `ctx.waitUntil`로 R2 없이 캐시키만 반환하고 클라이언트가 재요청하도록 스트리밍(또는 `Content-Length` 제한 시 503 유도) | 중 |
| D1 통계 쿼리 | 생성 통계/기록이 단일 스레드 누적(N+1 위험) | `COUNT(*) GROUP BY provider, model` 캐시 테이블 + TTL 5분, 기존 KV에 저장 | 하 |
| JSON 바디 무제한 | `request.json()` 무조건 신뢰 → DoS/OOM 위험 | `Content-Length` 헤더 검사(>2MB reject), `prompt` 길이 2000자 하드 컷 | 하 |
| 외부 API 타임아웃 | fetch에 timeout 없음(GTX 포함) → Worker 벽시계 초과 | `AbortSignal.timeout(8000)` + 재시도 1회(지수 백오프) | 하 |
| Observability | `ADVANCED_SECURITY_ENABLED` 등 on/off만 있고 실제 경보 없음 | 실패율 5% 초과 시 `/health/alert`에 `status:"degraded"`, webhook(Slack 등) 선택 | 중 |
| 업로드 프록시(ImgBB) | 하드코딩된 테스트 키 `8245f7…`(소스에 공개됨) | `IMGBB_API_KEY` env 시크릿으로, 미설정 시 업로드 비활성화 | 하 |

**기대효과:** p95 지연 20~40% 개선(캐시 적중 시), 외부 API 차단 시 무단 실패 0건화, 배포 사고(429/502) 자동 우회.

---

## 2. [자동화 및 워크플로우] — 수동 반복 작업 제거

| 기능 | 왜 필요한지 | 구현 방식(파일) | 난이도/효과 |
|---|---|---|---|
| **프롬프트 자동 최적화(이미 있음, 확장)** | 초보 사용자는 프롬프트 문법 모름 | `handlePromptGeneration` (한글→영어 번역+영어 최적화). 추가: "스타일 프리셋 자동 적용" 토글 | 하 / 입력 포기율 ↓ |
| **배치 생성 (1→4 seed 변형, 그리드 비교)** | "하나 뽑고 실패하면 다시" → 확률 게임화 | `handleGenerate`에 `n=4` 파라미터 → 4개 fetch를 `Promise.all` 병렬, UI에 그리드 선택 버튼, 선택된 것만 기록 | 중 / 실사용 만족도 최대 |
| **D1 멀티디바이스 히스토리 동기화** | 현재 history는 localStorage(기기별), 브라우저 캐시 삭제 시 소실 | `/api/history` GET/POST/DELETE + D1 `history` 테이블(기기ID+사용자ID 키), 기존 `SmartCache` 패턴 재사용 | 중 / 핵심 차별화 |
| **생성 완료 알림 (Web Push/Notification)** | 배치/고품질 모드는 3~30초 대기 → 사용자가 기다리지 않음 | `Notification.requestPermission()` + `new Notification("완료", {body: prompt})` | 하 / 재방문율 ↑ |
| **폴백 모델 체인 자동 선택** | 품질모드(economy/standard/ultra)를 사람이 골라야 함 | `AI_AUTO_QUALITY_SELECTION` env가 이미 있음 → 경쟁 프로젝트에 없는 "프롬프트 복잡도 점수(PromptAnalyzer)"와 결합해 모델/스텝 자동 결정 | 중 / 이미 절반 구현됨 |
| **예약 생성 (cron trigger)** | "매일 아침 3장씩" 같은 반복 생성 | wrangler `[triggers] crons` + Queues/D1 작업 테이블 | 중 / 파워유저 전용 |

---

## 3. [최신 기술 접목] — 2026년 기준, 사용 가능한 것과 이식안

| 기술 | 상태(2026-11 기준) | 우리 프로젝트 접목 |
|---|---|---|
| **FLUX.2 [klein] 4B/9B** (`@cf/black-forest-labs/flux-2-klein-4b|9b`) | Workers AI 신규, Partner 모델(뉴런 과금), 멀티-레퍼런스 편집 지원 | 고품질 모드(ultra) 전용으로 선택 가능하게. 기본은 schnell(무료) 유지. `CONFIG.PROVIDERS`에 id 추가 |
| **FLUX.2 [dev]** (`@cf/black-forest-labs/flux-2-dev`) | 가장 강력, 물리 기반 그라운딩/멀티링구얼 | 슬롯만 노출, 과금 고지 후 opt-in. 참조이미지 스타일  transfer에 특히 유리 |
| **Llama 4 Scout 17B 16E Instruct (vision)** | function calling + vision, 240k 컨텍스트 | 현재 vision은 Llama 3.2 11B. 라이선스/품질 검토 후 차기 교체 후보 |
| **Remote MCP (MCP 2026-07-28, stateless core)** | Workers Streamable-HTTP MCP 공식 지원, `workers-mcp` 템플릿 있음 | **`/mcp` 엔드포인트 내장** — Claude/Cursor가 바로 `generate_image` 툴 호출 (4장 명세) |
| **OpenAI 호환 `/v1` 엔드포인트** | Workers AI가 제공 | 외부 도구(Continue, Aider, Open WebUI)가 우리 Worker를 엔진으로 사용 가능 → 개발자 채택 채널 |
| **prompt-upsampling 내장 패턴** | FLUX.2 dev 가이드가 "시스템 프롬프트로 자동 확장"을 권장 | 이미 우리 프롬프트 최적화가 그 역할 → UX에는 그대로 노출 |
| **ComfyUI 노드/Gradio 템플릿** | 자체 호스팅 표준(수만 개 노드) | 포크해서 Worker 위에 올릴 수 없음(GPU 필요). 대신 "ComfyUI의 workflow JSON을 가져와 우리 프롬프트로 실행"은 불가 — 영역 다름. 경쟁 아님 |
| **오픈소스 모델: Qwen-Image, FLUX schnell, SDXL Lightning** | Replicate/HF Inference Endpoints 유료 | 우리는 "무료 엣지" 포지션 유지, 모델 로테이션만 고려 |

---

## 4. [예외 처리 및 우회] — API 차단/타임아웃/데이터 누락 자동 복구

| 시나리오 | 현재 동작 | 제안(자동 복구) |
|---|---|---|
| Pollinations 502/401 | Workers AI로 자동 폴백(페일오버 래퍼) + 경고 로그 | ✅ 완료. 추가: 실패 소스 헤더(`x-served-by`)로 UI에 "어디서 생성됐는지" 표시 |
| Infip 중단 | Ditto | ✅ 완료 |
| Google GTX 타임아웃/429 | catch → 원문 그대로 사용(현재) | `AbortSignal.timeout(8s)` + 1회 재시도 + UI 토스트("번역 실패, 원문 사용") |
| Workers AI 자체 429/뉴런 한도 | 순간 실패 → 500 | 지수 백오프 재시도 2회(200ms→1s) → 그래도 실패 시 `Retry-After` 담아 503 + 캐시된 이미지 유사본 반환(있으면) |
| 캐시 손상(직렬화 오류) | base64 라운드트립(이전 세션 수정) | 캐시 저장 전 `JSON.parse(JSON.stringify(x))` 안전화 + read 실패 시 즉시 miss 처리 ✅ |
| 참조 이미지 CORS/차단 | 검증 단계에서 400 반환 | ① 원 서버 fetch 실패 → ② wsrv.nl 이미지 프록시 폴백 → ③ 그래도 실패 시 사용자에게 "URL 직접 업로드" 유도 |
| Meta 라이선스 미동의 | (이 세션에서 해당) | 자동 1회 acknowledge + 5016을 성공으로 처리(적용됨) — 배포 체크리스트에 명시 |
| 빈 프롬프트/빈 AI 응답 | 500 throw(적용됨) | 사용자 친화 메시지(422 + "프롬프트를 더 구체적으로") + 자동 재작성 1회 |
| 부분 데이터 누락(D1/KV 실패) | 캐시/통계만 실패 가능 | KV/D1 실패를 try로 감싸 생성 자체는 항상 성공시킴(이미 대체로 적용) — 잔여 지점 grep |
| 비용 폭주(파트너 모델) | — | 일일 뉴런 예산 상한(env `NEURON_BUDGET`) + 초과 시 schnell로 강제 |

---

## 5. 경쟁 프로젝트 비교 (기능 기준, 별점은 2026-11 근사치)

| 프로젝트 | 규모/성격 | 강점 | 약점 | 우리 대비 |
|---|---|---|---|---|
| **AUTOMATIC1111 / SD-WebUI** | ~16만★, 로컬 GPU | inpaint/ControlNet/ADetailer/5k+ 확장 | 설치 무겁고, GPU 필요, 초보 비친화 | 무료·설치불필요·엣지배포에서 우리 우위 |
| **ComfyUI** (Comfy-Org) | ~14만★, 노드그래프 엔진 | 최고의 모듈성, 영상/3D까지 | 학습곡선 급함, 서버 운영 책임 | 우리는 "웹 한 장으로 끝" |
| **Fooocus** (lllyasviel) | ~5만★, 원클릭 품질 | 프롬프트 무슨말이든 그럴듯한 결과 | 로컬 한정, 유지보수 약화 | UX 철학 유사. 우리는 브라우저+D1 |
| **pollinations/pollinations** | ~8천★, 호스티드 API+MCP | 500+ 커뮤니티, 무료 엔드포인트, 영상/오디오/3D, 공식 MCP 패키지 | 익명 무료 계층 불안정(이 세션 401의 시작점), 지원정책 변동 | **우리는 그 위에 안정성 계층**을 얹음 |
| **cf-flux-remix (업스트림)** | ~120★, CF Workers 네이티브 | 서버리스, 0 GPU 비용, 한국어 i18n | 얕은 유지보수 → 이 포크가 보강 중 | — |

### 5-1. Feature Gap (우리에게 없고 경쟁에 있는 것)
1. **배치 생성 + 변형 그리드** — 전 경쟁 기본
2. **업스케일/디테일 강화** — SD계 extras, ComfyUI 업스케일 노드
3. **PWA/오프라인 설치** — 데스크톱 경쟁자는 설치가 기본; 웹 경쟁은 미지원 → 우리 기회
4. **MCP 엔드포인트 내장** — pollinations는 별도 패키지; 나머지는 없음
5. **워크플로우(노드) 재사용** — ComfyUI 전용. 우린 해당 없음(차별화 축으로 안 감)
6. **결과물 "스타일 일관성"(seed lock + ref 동시)** — 부분 구현, 강화 가능

### 5-2. 우리만의 강점
1. **완전 무료 + 키 불필요** 기본값(Workers AI schnell) — pollinations 의존 리스크 제거됨
2. **한/영 프롬프트 자동 번역 + 최적화** 내장 (경쟁: 영어 전제)
3. **한국어 네이티브 UI/i18n** + 저사양 CF 엣지
4. **자기 인프라**: D1 히스토리, KV 캐시, rate-limiter — 외부 SaaS 단절 없음
5. **이미지→프롬프트(vision)** 무료 내장 (MCP 없이도)

### 5-3. 차별화 포지션
> "GPU 없이, 키 없이, 한국어로, 무료로 — 개인 크리에이터의 제로코스트 엣지 이미지 스튜디오. 그리고 MCP로 Claude/Cursor에 직접 꽂히는 유일한 이미지 Worker."

---

## 6. 선택 3개 기능 — 작업 명세서

### Spec A. MCP 엔드포인트 내장 (`/mcp`)

| ① 왜 필요한지 | ② 구현 | ③ 난이도/기대효과 |
|---|---|---|
| 경쟁은 전부 별도 앱/설치. 우리 Worker 하나로 Claude Desktop·Cursor·VS Code가 `generate_image`를 직접 호출하면 개발자 채택 채널이 생김. Pollinations MCP와 차별: "Zero config + 무료 + 한국어" | `worker.js`에 Streamable-HTTP MCP 라우트 추가: `else if (url.pathname === '/mcp')` → 새 모듈 `mcp.js`. 의존: `@modelcontextprotocol/sdk` (또는 직접 JSON-RPC 2.0 최소 구현 ~200줄). tools: `generate_image(prompt, style, seed?, width?,height?)`, `optimize_prompt(text, lang?)`, `list_styles()`. 응답: image content part(base64 data URI) + text fallback. 인증: `MCP_API_KEY` env 옵션(없으면 익명, rate-limiter 재사용). `wrangler.toml`에 `[ai]`/KV 그대로 재사용 | **중 / 매우 큼** — 유일 차별화. 코드량 ~250줄 + README MCP 섹션 + 테스트 2개(en/ko) |


### Spec B. D1 멀티디바이스 히스토리 동기화

| ① 왜 필요한지 | ② 구현 | ③ 난이도/기대효과 |
|---|---|---|
| 현재 history는 localStorage → 브라우저 변경/캐시삭제 시 소실. 모바일/데스크톱 이원화 불가. 경쟁 중 D1급 동기화 내장은 없음 | ① D1 스키마: `CREATE TABLE IF NOT EXISTS history (id TEXT PRIMARY KEY, device_id TEXT, prompt TEXT, style TEXT, provider TEXT, model TEXT, seed INTEGER, image_url TEXT, created_at INTEGER)` + `CREATE INDEX idx_hist_device ON history(device_id, created_at)`. ② worker.js: `GET/POST/DELETE /api/history` (기존 `handleHistory*` 패턴 재사용), 저장은 생성 완료 시 `ctx.waitUntil`. ③ UI: 설정에 "기기 동기화" 토글 + `device_id`(crypto.randomUUID, localStorage) 헤더. ④ 충돌: last-write-wins + `updated_at` 컬럼 | **하~중 / 큼** — 체감 최대. 기존 localStorage와 merge(중복 id skip)로 하위호환. 코드 ~150줄 + 마이그레이션 1개 |

### Spec C. 배치 생성 (1 프롬프트 → 4 seed 변형, 그리드 비교)

| ① 왜 필요한지 | ② 구현 | ③ 난이도/기대효과 |
|---|---|---|
| 현재 "1장→실패하면 재시도"의 슬롯머신 UX. 경쟁은 기본 다중 생성+비교. 우리는 서버리스라 4개 병렬도 무료/빠름 | ① `handleGenerate(request, env, ctx, body)`에 `n`(1~4) 파싱, ② 같은 프롬프트/스타일로 seed만 다르게 `Promise.all([...Array(n)])` → 4개 Workers AI 호출, ③ 응답에 `images: [{seed, b64}]` 배열, ④ UI: 기존 생성 버튼 옆 "4x" 토글 추가, 결과 2×2 그리드 + "이걸로 선택"(=history 저장)·"전체 다운로드"(zip 아님, 개별 다운로드), ⑤ 캐시는 seed 포함 키로 분리 | **중 / 매우 큼** — 1클릭당 만족도 4배. 코드 ~200줄(worker 80 + UI 120). schnell 모델이 병렬 4건도 부담 없음 |

---

## 7. 추천 실행 순서 & 주의사항

1. **지금 바로(30분):** 폴더 정리, ImgBB 키를 env로(로테이션), 배포 전 `/health` 확인
2. **이번 주:** Spec B(히스토리 동기화) → Spec C(배치) — 사용자 체감 즉시
3. **다음 스프린트:** Spec A(MCP) — 개발자 커뮤니티 배포(README에 `claude mcp add` 명령 추가)
4. **보류/검토:** FLUX.2 [dev] 파트너 모델(과금 고지 필요), PWA(서비스워커) — Spec A 이후
5. **배포 주의:** 현재 로컬(8787)에서만 테스트됨. 실제 workers.dev 배포는 공개 URL 생성이라 사용자 확정 필요
6. **라이선스:** 업스트림 Apache-2.0 유지, 포크 표기 유지(`NOTICE`/`README`), trademark(Upstream 이름) 미사용

## 8. 사용자 결정 필요
- ① 위 3개 Spec 중 즉시 구현 범위(A/B/C 전부? B+C 먼저?)
- ② FLUX.2 파트너 모델 과금 허용 여부
- ③ 실제 `wrangler deploy`(공개) 실행 여부
