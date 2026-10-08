# cognee 전수조사 분석 및 수익화 가이드

> 이 문서는 `bmshin94/cognee` 저장소를 전수조사하고, 활용법·기술 질의응답·수익화 전략까지
> 정리한 종합 분석 보고서입니다.
>
> - **대상 저장소(포크)**: https://github.com/bmshin94/cognee
> - **원본 저장소**: https://github.com/topoteretes/cognee
> - **분석 기준 버전**: cognee **v1.5.4**
> - **작성일**: 2026-10-08
> - **라이선스**: Apache-2.0 (상업적 이용·수정·재배포 가능)
> - **논문**: [Optimizing the Interface Between Knowledge Graphs and LLMs for Complex Reasoning](https://arxiv.org/abs/2505.24478) (Markovic et al., 2025)

---

## 목차

1. [저장소 정체 및 규모](#1-저장소-정체-및-규모)
2. [무엇을 해결하는 프로젝트인가](#2-무엇을-해결하는-프로젝트인가)
3. [핵심 API — 4개 함수](#3-핵심-api--4개-함수)
4. [2층 기억 구조 (세션 + 영구)](#4-2층-기억-구조-세션--영구)
5. [폴더별 전수 분석](#5-폴더별-전수-분석)
6. [입력 — 지원 데이터 형식](#6-입력--지원-데이터-형식)
7. [출력 — 검색 전략 21종](#7-출력--검색-전략-21종)
8. [고급 기능](#8-고급-기능)
9. [멀티테넌시 & 권한](#9-멀티테넌시--권한)
10. [설치 및 사용법](#10-설치-및-사용법)
11. [플러그인 / 스킬 / MCP — 정체 구분](#11-플러그인--스킬--mcp--정체-구분)
12. [API 토큰 및 비용](#12-api-토큰-및-비용)
13. [AI 에이전트 구축 활용](#13-ai-에이전트-구축-활용)
14. [React / PHP 연동](#14-react--php-연동)
15. [유튜브 강의 제작 가능성](#15-유튜브-강의-제작-가능성)
16. [수익화 아이디어 10선](#16-수익화-아이디어-10선)
17. [실행 로드맵](#17-실행-로드맵)
18. [리스크 및 주의사항](#18-리스크-및-주의사항)
19. [참고 링크](#19-참고-링크)

---

## 1. 저장소 정체 및 규모

**`topoteretes/cognee`** 를 `bmshin94` 계정으로 포크한 저장소.

> **cognee = AI 에이전트를 위한 오픈소스 장기기억(Long-term Memory) 플랫폼**

| 항목 | 값 |
|---|---|
| 원본 저장소 | https://github.com/topoteretes/cognee |
| 포크 저장소 | https://github.com/bmshin94/cognee |
| 버전 | 1.5.4 |
| 라이선스 | Apache-2.0 |
| 저작권자 | Topoteretes UG (독일 베를린 법인) |
| 전체 파일 | 3,593개 |
| Python 코드 | 2,446개 파일 / 약 353,000줄 |
| 프론트엔드(TS/TSX) | 522개 파일 / 약 47,000줄 |
| Python 요구사항 | 3.10 ~ 3.14 |
| CI 워크플로우 | 58개 |
| 테스트 파일 | 823개 (전체 Python 파일의 약 33%) |
| DB 마이그레이션 | 48개 버전 (alembic) |

**규모 평가**: 약 40만 줄 코드베이스. 토이 프로젝트가 아니라 법인이 운영하는 상업적
오픈소스(Open-core) 제품. Product Hunt 1위, Trendshift 등재 이력.

---

## 2. 무엇을 해결하는 프로젝트인가

### 문제: LLM은 대화창을 닫으면 모든 걸 잊는다

LLM은 컨텍스트 창이라는 한정된 "단기 기억"만 있고 영구 기억이 없다.

### 기존 해결책(RAG)의 한계

```
문서 → 청크로 쪼갬 → 벡터 변환 → DB 저장
질문 → 벡터 변환 → "비슷한 조각" 검색 → LLM에 전달
```

문제는 **"비슷한 것"만 찾고 "관계"를 모른다**는 점.

> 질문: "김 과장이 반대했던 그 결정, 결국 누가 구현했어?"
>
> RAG: "김 과장" 문서 조각과 "구현" 문서 조각을 각각 반환 → 연결 실패 → 엉뚱한 답

### cognee의 해결책: ECL 파이프라인

RAG를 **ECL(Extract → Cognify → Load)** 로 대체.

```
[원본 데이터]
     ↓ Extract  : 문서를 청크로 분할
     ↓ Cognify  : LLM이 개체(Entity)와 관계(Relationship)를 추출  ★핵심★
     ↓ Load     : 그래프DB + 벡터DB + 관계형DB 에 동시 저장
[지식 그래프]
```

결과물은 관계망:

```
 (김과장) ──[반대함]──→ (DB마이그레이션 결정) ──[결정됨]──→ (3월 회의)
                              │                                │
                       [구현됨]│                         [참석함]│
                              ▼                                ▼
 (이대리) ──[작성함]──→ (migration.py) ◄──────────────────────┘
                              │
                       [원인이됨]
                              ▼
                        (장애 #231)
```

**핵심 차이**: RAG는 "비슷한 것"을 찾고, cognee는 "연결된 것"을 따라간다.

### DB를 3개 쓰는 이유

| DB | 역할 | 비유 |
|---|---|---|
| 벡터 DB | "비슷한 의미" 검색 | 비슷한 분위기 책을 추천하는 사서 |
| 그래프 DB | "연결 관계" 순회 | 작가의 스승·제자 계보를 아는 사서 |
| 관계형 DB | 권한·메타데이터 | 대출 기록과 회원을 관리하는 사서 |

RAG는 벡터 DB 하나만 쓴다. cognee는 세 개를 동시에 쓴다.

---

## 3. 핵심 API — 4개 함수

v1.0부터 모든 기능이 4개 함수로 정리됨. 모두 async.

```python
import cognee, asyncio

async def main():
    await cognee.remember("우리 팀은 매주 화요일 2시에 회의한다")   # ① 기억
    results = await cognee.recall("회의가 언제야?")                  # ② 조회
    await cognee.improve()                                           # ③ 보강
    await cognee.forget(dataset="main_dataset")                      # ④ 삭제

asyncio.run(main())
```

| 함수 | 역할 | 비유 |
|---|---|---|
| `remember()` | 데이터 저장 → 그래프 구축 | 기억을 새김 |
| `recall()` | 질문 → 최적 검색전략 자동선택 → 답변 | 기억을 떠올림 |
| `improve()` | 그래프 보강, 피드백 반영, 세션 지식 승격 | 자면서 기억을 정리 |
| `forget()` | 선택적/전체 삭제 | 잊어버림 |

### 하위 레벨 API (세밀한 제어용)

위 4개는 내부적으로 아래를 호출한다.

```python
await cognee.add("데이터")      # 적재 (스테이징)
await cognee.cognify()          # 그래프 구축 (LLM이 개체·관계 추출)
await cognee.search("질문")     # 검색
await cognee.memify()           # 그래프 보강
```

하위 레벨을 써야 하는 경우:

- `functional_relationships=` 로 단일 타겟 관계를 제약할 때 → `cognify()` 만 가능
- 여러 데이터셋을 동시에 처리할 때 (`remember()` 는 항상 데이터셋 1개)
- DB에 이미 있는 데이터로 그래프만 재구축할 때 (`forget(memory_only=True)` 후)
- `prune.prune_system(metadata=True)` 로 관계형 DB까지 완전 초기화할 때

### recall() vs search()

`recall()` 은 `search()` 를 감싸며 3가지를 추가한다.

1. **규칙 기반 쿼리 라우팅** (정규식 점수 방식, LLM 호출 없음 → 비용 0)
2. **세션 메모리를 검색 소스로** (`scope` = graph / session / trace / session_context)
3. **`_source` 키로 태깅된 정규화 결과**

`search()` 를 써야 하는 경우: `skills`/`tools`/`max_iter`/`code_query`/`node_type` 를
1급 파라미터로 쓸 때, 원시 `SearchResult` 객체가 필요할 때, 라우터 없이
`query_type` 을 고정할 때.

---

## 4. 2층 기억 구조 (세션 + 영구)

사람 뇌의 해마 → 대뇌피질 구조를 모방한 설계. cognee의 핵심 차별점.

```
대화 중: remember(..., session_id="chat_1")
              │
              ▼
   ┌──────────────────────────────┐
   │  세션 캐시 (단기기억)         │  ← 즉시 저장. 빠름. 현재 대화에 사용
   │  SQLite / Redis / Postgres    │
   └──────────────────────────────┘
              │
     백그라운드 improve() 가 승격(bridging)
              │
              ▼
   ┌──────────────────────────────┐
   │  지식 그래프 (장기기억)       │  ← 영구 보존. 모든 세션이 공유
   │  Graph DB + Vector DB         │
   └──────────────────────────────┘
```

```python
# 빠른 쪽에 저장 (즉시)
await cognee.remember("사용자는 짧은 답변을 선호함", session_id="chat_1")

# 세션 먼저 보고, 없으면 영구 그래프로 내려감
await cognee.recall("사용자 취향이 뭐야?", session_id="chat_1")
```

### 자동 피드백 (self-improving memory)

`AUTO_FEEDBACK=true` (기본값) 이면 매 턴마다 구조화 출력 LLM 호출 1회로
"사용자가 만족했나? 어떤 암묵적 피드백이 있나?"를 분석해 기억 가중치를 조정하고
교훈을 저장한다. `improve()` 가 그 교훈을 영구 그래프로 승격한다.

→ **쓸수록 똑똑해지는 기억**

### 세션 검색 모드

`SESSION_SEARCH_MODE` 로 배포 단위 설정 (요청별 오버라이드 없음).

| 모드 | 동작 | 차이 |
|---|---|---|
| `concurrent` (기본) | 분석이 검색·응답과 **동시** 실행. 2개 레인(원문 질문 + LLM 없는 결정론적 재작성)으로 보상 | 이번 턴 지침은 다음 턴부터 반영 |
| `sequential` | 분석이 **먼저** 실행, 재작성된 쿼리로 단일 검색 | 이번 턴 지침이 이번 턴 답변에 반영 |

---

## 5. 폴더별 전수 분석

```
🏢 cognee 저장소
│
├─ 🚪 cognee/api/              (159개 파일) REST API + SDK 엔드포인트
│    ├─ v1/remember recall improve forget   ★ 핵심 4개
│    ├─ v1/add cognify search memify delete 하위 레벨
│    ├─ v1/datasets users permissions       멀티테넌시
│    ├─ v1/skills proposals agents tools    에이전트 기능
│    ├─ v1/session sessions                 세션 관리
│    ├─ v1/slack integrations sync push     외부 연동
│    └─ v1/visualize ui export report       시각화·내보내기
│
├─ 🧠 cognee/modules/          (629개 파일, 최대 부서) 도메인 로직
│    ├─ retrieval/             검색기 30종
│    ├─ graph/                 그래프 조작
│    ├─ chunking/              문서 분할 + 증분 업데이트 diff
│    ├─ ontology/              온톨로지(OWL) 기반 추출
│    ├─ search/                검색 라우팅
│    ├─ session_distillation/  세션 → 영구기억 증류
│    ├─ user_preferences/      개인화
│    ├─ provenance/            출처 추적 5종
│    ├─ truth_subspace/        진실성 검증
│    ├─ agent_memory/          에이전트 메모리 데코레이터
│    └─ integrations/          github / linear / slack / OAuth
│
├─ 🔧 cognee/tasks/            (149개 파일) 파이프라인 작업 단위
│    ├─ graph/                 개체·관계 추출
│    ├─ code_graph/            코드 전용 그래프 (LLM 없이 결정론적)
│    ├─ chunks/ documents/ summarization/
│    ├─ temporal_graph/        시간 인식 그래프
│    ├─ web_scraper/           웹 스크래핑
│    └─ translation/ provenance/ schema/
│
├─ 🔌 cognee/infrastructure/   (311개 파일) 외부 시스템 어댑터
│    ├─ databases/graph/       Ladybug(기본)·Neo4j·Neptune·Postgres·Turso
│    ├─ databases/vector/      LanceDB(기본)·PGVector·Neptune·Turso
│    ├─ databases/relational/  SQLite(기본)·PostgreSQL
│    ├─ llm/                   OpenAI·Anthropic·Gemini·Ollama·Mistral·Bedrock
│    ├─ loaders/               파일 로더 (텍스트·코드·CSV·이미지·오디오·비디오)
│    ├─ files/ storage/        로컬 + S3
│    └─ session/ locks/        세션 캐시, 분산 락
│
├─ 💻 cognee/cli/              (38개 파일) cognee-cli 터미널 도구 (명령 22개)
├─ 🧪 cognee/tests/            (823개 파일) unit/integration/e2e/cli/performance
├─ 📊 cognee/eval_framework/   (85개 파일) BEAM 등 벤치마크
├─ ⚙️ cognee/memify_pipelines/ (11개) 그래프 보강 파이프라인
├─ 🗄 cognee/alembic/          DB 스키마 마이그레이션 (48개 버전)
│
├─ 🖥 cognee-frontend/         Next.js 16 + React 19 웹 대시보드
│    └─ D3 / d3-force-3d / react-force-graph-2d 로 그래프 시각화
│
├─ 🔗 cognee-mcp/              MCP 서버 (remember/recall/forget 3개 도구)
├─ 📦 cognee-starter-kit/      신규 프로젝트 템플릿
│
└─ 📚 자료실
     ├─ examples/              실행 가능한 예제 약 40개
     ├─ notebooks/             Jupyter 튜토리얼 6개
     ├─ distributed/deploy/    Fly / Railway / Render / Modal / Daytona 배포 템플릿
     ├─ catalog/               통합·유즈케이스 메타데이터
     ├─ .claude/skills/        Claude Code용 스킬 7개
     ├─ docs/                  문서
     └─ docker-compose.yml     API(8000) + UI(3000) + MCP(8001) + Postgres + Neo4j
```

### 성숙도 신호

| 지표 | 토이 프로젝트 | cognee |
|---|---|---|
| 테스트 비중 | 0~5% | **33%** (823개 파일) |
| CI 파이프라인 | 없음 | **58개** |
| DB 마이그레이션 | 없음 | **48개 버전** |
| 하위호환 테스트 | 없음 | 전용 폴더 존재 |

---

## 6. 입력 — 지원 데이터 형식

`cognee/infrastructure/loaders/` 전수조사 결과.

| 로더 | 지원 확장자 |
|---|---|
| 텍스트 | `txt` `md` `json` `xml` `yaml` `yml` `log` |
| 코드 | `py` `go` `ts` `java` `rs` 등 (★ LLM 비용 0원) |
| CSV | `csv` (dlt 연동 가능) |
| 이미지 | `png` `jpg` `jpeg` `gif` `webp` `apng` `dwg` `xcf` `jpx` 등 (OCR/비전) |
| 오디오 | `mp3` `m4a` `wav` `flac` `ogg` `aac` `mid` `amr` `aiff` (전사) |
| 비디오 | `mp4` `m4v` `mov` `webm` `mkv` `avi` |
| 문서 | PDF, DOCX, PPTX, HTML, 이메일, LaTeX (docling / unstructured) |
| 웹 | URL 스크래핑 (Tavily, BeautifulSoup, Playwright, Keenable) |
| 저장소 | GitHub/GitLab repo URL 통째로 → 단일 코드 그래프 |
| DB | 관계형 DB에서 대량 수집 (dlt) |
| 타 메모리 | Mem0, Letta, Zep, Graphiti → COGX 포맷으로 이전 |

### 코드는 특별 취급 (중요)

코드 파일은 LLM을 전혀 쓰지 않고 **`enola`** 결정론적 파서가 처리한다.

- 정확함 (LLM 추측 아님) + **비용 0원** + 매번 같은 결과
- `CodeSymbol` / `CodeModule` 노드 + `calls` / `imports` / `has_method` 엣지
- 버전은 플랫폼별 SHA-256 과 함께 `cognee/tasks/code_graph/install_enola.py` 에 고정,
  최초 사용 시 `~/.cognee/bin` 에 자동 설치 (`ENOLA_AUTO_INSTALL=false` 로 해제)

```bash
# 아키텍처 다이어그램 자동 생성
cognee-cli search "" -t CODE \
  --code-query '{"operation":"architecture"}' \
  --diagram-out 아키텍처.html
```

| 연산 | 답하는 질문 |
|---|---|
| `impact_analysis` | 이 함수 고치면 뭐가 깨지나 |
| `find_path` | A에서 B까지 호출 경로 |
| `architecture` | 전체 구조도 |
| `delta` | 지난번과 뭐가 달라졌나 |
| `explore` `traverse` `query_facts` `insights` | 탐색·순회·사실조회·통찰 |

출력 형식: `.html`(Mermaid 브라우저 렌더), `.svg/.png/.pdf`(Graphviz DOT), 그 외(원본 소스).

---

## 7. 출력 — 검색 전략 21종

`cognee/modules/search/types/SearchType.py` 전수 확인.

| 타입 | 설명 |
|---|---|
| `HYBRID_COMPLETION` | **기본값**. 문서 구절 + 개체 주변망 + LLM 답변 |
| `GRAPH_COMPLETION` | 그래프 순회 + LLM |
| `GRAPH_COMPLETION_COT` | Chain-of-Thought 추론 |
| `GRAPH_COMPLETION_DECOMPOSITION` | 질문 분해 |
| `GRAPH_COMPLETION_CONTEXT_EXTENSION` | 확장 컨텍스트 |
| `GRAPH_SUMMARY_COMPLETION` | 사전 요약 + 그래프 |
| `TRIPLET_COMPLETION` | 삼중항(주어-술어-목적어) |
| `RAG_COMPLETION` | 전통 RAG |
| `CHUNKS` / `CHUNKS_LEXICAL` | 청크 벡터검색 / 키워드검색 |
| `SUMMARIES` | 요약 검색 |
| `CYPHER` | 그래프 쿼리 직접 실행 (`ALLOW_CYPHER_QUERY=True` 필요) |
| `NATURAL_LANGUAGE` | 자연어 → 구조화 쿼리 |
| `TEMPORAL` | 시간 인식 검색 |
| `CODE` | 코드 그래프 전용 |
| `AGENTIC_COMPLETION` | 에이전트 루프 (도구 사용) |
| `SKILLS` | 스킬 플레이북 탐색 (메타데이터만, LLM 없음) |
| `CODING_RULES` | 코딩 규칙 |
| `GRAPH_REPORT` | 그래프 리포트 |
| `FEELING_LUCKY` | 자동 선택 |

`recall()` 은 정규식 기반 라우터로 자동 선택한다(LLM 호출 없음 → 비용 0).
CLI는 더 좁다: `cognee-cli recall --query-type` 은 `cognee/cli/config.py`의
`SEARCH_TYPE_CHOICES` 만 받고 기본값은 `HYBRID_COMPLETION`. 나머지는 SDK 전용.

---

## 8. 고급 기능

### 8.1 증분 업데이트 (`update()`)

문서 일부만 수정했을 때 전체 재처리를 피한다.

```
문서 1,000페이지 중 한 문단만 수정

❌ 일반 구현: 1,000페이지 전체 재처리 → LLM 비용 폭발
✅ cognee:    바뀐 영역만 찾아 해당 청크만 재처리 → 비용 1/1000
```

- 문단 기준 다중 영역 diff (반복 라인 많은 콘텐츠에도 near-linear)
- 변경 구간을 청크 경계로 확장 후 표준 TextChunker로 재분할
- 모든 청크가 `max_chunk_tokens` 를 저장 → 설정 변경에도 문서 내 일관성 유지
- 안 바뀐 청크는 노드 ID·개체·요약·임베딩 유지, `chunk_index` 는 연속성 보장
- 청크 ID는 콘텐츠 기반 (`uuid5(doc : sha256(text) : occurrence)`)
- 데이터셋별 락 + 데이터셋 스코프 DB 컨텍스트에서 실행 (멀티테넌트 안전)
- 전제조건 실패 시 전체 재처리로 폴백 (최초 적재, 비텍스트, pre-v2 청크, 미검증 그래프 어댑터)
- 검증된 어댑터: Kuzu/Ladybug, Neo4j, Postgres 데모. Neptune은 폴백.
- 해제: `update(..., chunk_level_diff=False)` 또는 `PATCH /update` 쿼리 파라미터

### 8.2 모순 탐지 (기본 꺼짐)

```bash
CONTRADICTION_DETECTION=true
CONTRADICTION_CONFIDENCE_THRESHOLD=0.5   # 플래그 기준 신뢰도
CONTRADICTION_MAX_FACTS=500              # LLM 호출당 사실 상한
```

`cognify()` 의 마지막 태스크로 실행. 이번 적재가 건드린 개체의 1홉 이웃 사실을 모아
LLM에게 "어느 쌍이 양립 불가능한가"를 묻고, 확신 있는 충돌마다 `contradicts` 엣지를
기록한다 (양쪽 사실 텍스트 + 이유 + 신뢰도 포함).

- 엣지만 추가. 재작성/삭제 안 함. 자체 에러를 삼켜서 적재를 깨뜨리지 않음
- `remember()` 와 `improve()` 가 승격한 세션 메모리에도 적용
- 예외: `remember(content_type="code")` (별도 코드 그래프 파이프라인)
- 제외: 구조적 엣지(`contains` `is_part_of` `made_from` `exists_in` `contradicts`),
  끝점 이름이 없는 엣지, 시간축 cognify 경로

### 8.3 출처 추적(Provenance) 5종

| # | 메커니즘 | 답하는 질문 | 저장 위치 | 플래그(기본값) |
|---|---|---|---|---|
| 1 | Source stamping | 누가/어느 실행이 이 노드를 썼나 | 노드의 `source_*` 필드 | `COGNEE_PROVENANCE_MODE` (`lightweight`) |
| 2 | Graph source-refs | 어느 문서가 이 노드/엣지를 소유하나 | 노드/엣지의 source-ref 키 | 항상 켜짐 |
| 3 | **Audit ledger** | 감사용 위조 탐지 이력 | `provenance_entries` 테이블 (해시체인) | `PROVENANCE_TRACKING` (**false**) |
| 4 | Memory-provenance | 누가 무엇에 접근 가능한가 | 요청 시 관계형 DB에서 계산 | — |
| 5 | **Edge evidence** | 이 엣지를 뒷받침하는 문서 청크 | `provenance_edge_evidence` 테이블 | `EDGE_EVIDENCE_ENABLED` (`true`) |

Edge evidence는 `add_data_points` 중 메모리에 모아 데이터 항목당 1회 벌크 기록
(`EDGE_EVIDENCE_FLUSH_THRESHOLD`, 기본 10000). `search(include_references=True)` 로
구조화된 `EvidenceReference` 객체로 조회.

### 8.4 스킬 (절차적 기억)

데이터셋 스코프 `SKILL.md` 플레이북. 에이전트가 발견 → 필요 시 로드 → 실행 → 개선.

- **적재**: `remember(content_type="skills", dataset_name=...)` — 명시적 데이터셋 필수,
  재적재는 upsert (결정론적 ID). HTTP: `POST /skills`
- **발견**: `SearchType.SKILLS` — `Skill_search_text` 컬렉션 벡터검색 1회, LLM 없음,
  **메타데이터만 반환** (본문 미포함 = progressive disclosure). 데이터셋 정확히 1개 필요
- **스킬 게이트**: `recall()` 이 결정론적 정규식 게이트 실행
  (`cognee/api/v1/recall/skill_gate.py`). 절차적 질의면 SKILLS 조회를 동시 실행해
  `source="skills"` 태그로 추가. `SKILL_GATE_ENABLED=false` 로 해제
- **실행**: `SearchType.AGENTIC_COMPLETION` + `skills=[...]` — LLM이 이름+설명을 보고
  `load_skill` 도구로 본문 로드 (12k자 상한)
- **개선**: `SkillRun` 기록 → LLM이 `SkillImprovementProposal` 초안 → 미리보기 후
  제안 ID로 적용 (`/proposals` 라우터)

### 8.5 온톨로지

```bash
ONTOLOGY_RESOLVER=rdflib
MATCHING_STRATEGY=fuzzy                   # 80% 유사도 퍼지 매칭
ONTOLOGY_FILE_PATH=/path/to/ontology.owl
ONTOLOGY_MODE=annotate                    # 또는 strict
```

`ONTOLOGY_MODE=strict` 는 타입이 온톨로지 클래스와 일치하거나 이름이 개체와 일치하는
개체만 남기고 나머지(및 그 엣지)를 버린다. 그래프만 정리하고 청크 텍스트는 그대로
저장·임베딩되므로 CHUNKS/RAG_COMPLETION 으로는 버려진 개체도 조회 가능.

> ⚠️ 작은 온톨로지는 대부분의 개체를 버린다. 빈/없는 온톨로지 파일 + strict = 하드 에러.

호출별 설정도 가능: `config={"ontology_config": {"ontology_mode": "strict", ...}}`

### 8.6 성능 튜닝 플래그 4개

| 플래그(기본값) | 끄면 사라지는 것 | 비용 |
|---|---|---|
| `PERSONALIZATION_ENABLED=false` | 사용자별 `UserPreference` 노드, `prefers` 가중치 엣지, 랭킹 보정, 선호 텍스트 프롬프트 주입, 턴당 1-5 평점, `improve()` 평점 반영 | 기본 꺼짐이라 손실 없음. 켜면 `PERSONALIZATION_INFLUENCE`(기본 0.3, 범위 [0,1])로 강도 조절. 사용자 + 단일 데이터셋 필요 |
| `CACHING=true` | 세션 메모리 전체: `remember(session_id=)` 예외 발생, `recall()` 세션 이력·숏서킷 상실, `agent_memory` 세션 옵션 에러, `AUTO_FEEDBACK` 무의미 | 빠른 세션 쓰기 경로 + 자기개선 메모리 상실. **이 상태로 벤치마크하면 안 됨** |
| `AUTO_FEEDBACK=true` | 턴당 구조화 출력 LLM 호출 1회 (암묵 피드백 감지, 이후 검색 가이드, `improve()` 교훈) | 대화 신호로부터의 자기조정 중단. 세션 저장/조회는 유지 → **저지연 읽기용으로 끌 플래그** |
| `DATASET_QUEUE_ENABLED=true` | 프로세스당 동시 데이터셋 상한(`DATASET_QUEUE_MAX_CONCURRENT`, 기본 6), 서브프로세스 엔진 teardown, 사용 중 엔진 eviction 방지 | 임베디드 엔진이 무제한이 됨 → 파일락 누수, 사용 중 엔진 eviction. 단일 데이터셋 스크립트만 안전 |

`AUTO_FEEDBACK` 은 `CACHING=true` 일 때만 참조된다.
**읽기가 느리면 `AUTO_FEEDBACK=false` + `CACHING=true` 유지가 정답.**

---

## 9. 멀티테넌시 & 권한

```
Tenant → User → Role → Dataset → Data  + ACL(읽기/쓰기/삭제/공유)
```

`ENABLE_BACKEND_ACCESS_CONTROL=True` (기본값) 이면 사용자+데이터셋 조합마다 DB가 격리된다.
권한 없는 검색은 에러가 아니라 **빈 배열**을 반환한다 (정보 누출 방지).

### 지원 매트릭스

출처: `cognee/infrastructure/databases/dataset_database_handler/supported_dataset_database_handlers.py`

| 계층 | 백엔드 | 사용자+데이터셋별 격리 | 비고 |
|---|---|---|---|
| 그래프 | Ladybug/Kuzu (기본) | ✅ | 임베디드, 데이터셋당 DB 1개 |
| 그래프 | Neo4j | ✅ | DBMS 내 데이터셋당 DB 1개 — 멀티DB 지원 에디션 필요(Enterprise/Aura). `neo4j_aura_dev` 핸들러는 데이터셋당 Aura 인스턴스 전체를 프로비저닝 (dev/PoC 전용) |
| 그래프 | Postgres | ✅ | graph-on-Postgres 자체가 데모 기능 |
| 그래프 | Turso | ✅ | |
| 그래프 | Neptune, ladybug-remote | ❌ | `ENABLE_BACKEND_ACCESS_CONTROL=false` 필요 |
| 벡터 | LanceDB (기본) | ✅ | |
| 벡터 | PGVector | ✅ | |
| 벡터 | Turso | ✅ | |
| 벡터 | Neptune Analytics | ❌ | `ENABLE_BACKEND_ACCESS_CONTROL=false` 필요 |
| 벡터 | 커뮤니티 어댑터 (ChromaDB, Qdrant…) | ❌ | `use_dataset_database_handler()` 로 핸들러 등록 시 가능 |
| 관계형 | SQLite / Postgres | 해당 없음 — 항상 공유 | 사용자·ACL·데이터셋DB 레지스트리를 담는 단일 DB |

- 핸들러는 설정된 provider 로부터 **자동 선택**된다 (수동 지정 불필요)
- **그래프와 벡터 둘 다** 격리를 지원해야 한다. 하나라도 미지원이면 `EnvironmentError`
  (플래그가 켜진 기본 상태에서는 **하드 에러**, 공유 DB로 조용히 폴백하지 않음)
- 해결: 백엔드 변경 또는 `ENABLE_BACKEND_ACCESS_CONTROL=false`

---

## 10. 설치 및 사용법

### 레벨 0 — API 키 없이 1분 체험

```bash
pip install cognee
cognee-cli demo
```

### 레벨 1 — Python SDK

```bash
pip install uv
uv venv && source .venv/bin/activate      # Windows: .venv\Scripts\activate
uv pip install cognee
export LLM_API_KEY="sk-..."
```

```python
import cognee, asyncio, os
os.environ["LLM_API_KEY"] = "sk-..."

async def main():
    await cognee.remember("cognee는 문서를 AI 기억으로 바꾼다")
    for r in await cognee.recall("cognee가 뭐 하는 거야?"):
        print(r)

asyncio.run(main())
```

기본 DB는 모두 로컬 파일로 `.venv` 안에 자동 생성된다 (별도 DB 서버 불필요).

- 관계형: **SQLite**
- 벡터: **LanceDB**
- 그래프: **Ladybug(Kuzu)**

위치 변경: `DATA_ROOT_DIRECTORY`, `SYSTEM_ROOT_DIRECTORY`

### 레벨 2 — CLI (명령 22개)

```bash
# 메모리 API (주력)
cognee-cli remember "기억할 내용"          # 파일경로·URL도 가능
cognee-cli remember ./내문서폴더
cognee-cli recall "질문"
cognee-cli recall "질문" --query-type HYBRID_COMPLETION
cognee-cli improve -d my_project
cognee-cli forget --all                    # ⚠️ 확인 프롬프트 없음

# 하위 레벨
cognee-cli add "내용" && cognee-cli cognify
cognee-cli search "질문"
cognee-cli delete --all                    # 확인 프롬프트 있음

# 관리
cognee-cli datasets list
cognee-cli config set LLM_MODEL openai/gpt-5-mini
cognee-cli doctor                          # 설정 진단
cognee-cli sessions list
cognee-cli migrate                         # DB 마이그레이션
cognee-cli eval                            # 벤치마크
cognee-cli serve                           # API 서버
cognee-cli -ui                             # 전체 스택 + 웹UI
```

> ⚠️ `forget --all` 은 확인 없이 즉시 삭제. `delete --all` 은 물어본다.
> `cognee.delete` 는 0.3.9부터 deprecated (`datasets.delete_data` 로 대체).
> `forget()` 이 기존 delete/prune/empty_dataset 경로를 통합한 v1 대체물.

### 레벨 3 — Docker

```bash
docker compose up                                    # API만
cp .env.template .env                                # LLM_API_KEY 입력
docker compose --profile ui --profile mcp up         # 전체 스택
```

| 서비스 | 포트 |
|---|---|
| API | 8000 (`/docs` 에 Swagger) |
| 웹 UI | 3000 |
| MCP | 8001 |

### 레벨 4 — extras

```bash
uv pip install -e ".[postgres,neo4j,docs,anthropic]"
```

```
DB:     postgres, postgres-binary, neptune, turso, redis
문서:   docs(unstructured), docling, docling-full, rapidocr
LLM:    anthropic, ollama, mistral, groq, azure, llama-cpp, huggingface
프레임: langchain, llama-index, graphiti, baml, dlt
기타:   scraping, aws(S3), codegraph, evals, deepeval,
        posthog, tracing, gmail, notebook, fastembed, api
개발:   dev, debug
```

> ⚠️ `docling-full` 과 `codegraph` 는 tree-sitter 버전 핀 충돌로 동시 설치 불가.
> `postgres` extra 는 Postgres 세션 캐시 백엔드(`CACHE_BACKEND=postgres`)도 활성화.

### 레벨 5 — 개발환경

```bash
git clone https://github.com/bmshin94/cognee.git
cd cognee
uv venv && source .venv/bin/activate
uv pip install -e ".[dev]"
pre-commit install

ruff check . && ruff format . && ty check .
pre-commit run --all-files     # ★ 커밋 전 필수
pytest
```

> ⚠️ 원본 저장소는 `main` 이 아니라 **`dev` 에서 분기**한다.
> 코어팀 PR은 Linear 이슈 키(`COG-123`)를 제목이나 브랜치명에 포함해야 한다
> (`linear-issue-check` 필수 체크). 포크/외부 기여자 PR은 면제.

### 완전 무료 구성 (Ollama)

```bash
ollama pull llama3.1:8b
ollama pull nomic-embed-text
```

```bash
LLM_PROVIDER="ollama"
LLM_MODEL="llama3.1:8b"
LLM_ENDPOINT="http://localhost:11434/v1"
LLM_API_KEY="ollama"
EMBEDDING_PROVIDER="ollama"
EMBEDDING_MODEL="nomic-embed-text:latest"
EMBEDDING_ENDPOINT="http://localhost:11434/api/embed"
HUGGINGFACE_TOKENIZER="nomic-ai/nomic-embed-text-v1.5"
```

> ⚠️ **가장 흔한 실수**: LLM만 Ollama로 바꾸고 임베딩을 안 바꾸면 임베딩이 OpenAI로
> 폴백되어 `NoDataError` 가 발생한다. **반드시 둘 다** 설정해야 한다.

### 추천 학습 순서

```
1. cognee-cli demo                                              (1분)
2. examples/guides/simple_cognee_example.py                     (5분)
3. notebooks/cognee_simple_demo.ipynb                           (15분)
4. examples/advanced_guides/remember_recall_improve_example.py   (30분)
5. examples/demos/company_brain_demo.py                         (1시간) ★실전
6. examples/guides/code_graph_example.py                         (코드그래프)
7. examples/guides/graph_visualization.py                        (시각화)
```

---

## 11. 플러그인 / 스킬 / MCP — 정체 구분

**결론: 전부 다다.** 본질은 "Python 라이브러리"고 나머지는 포장지다.

```
        ┌──────────────────────────────────────┐
        │   ⭐ 본질: Python 라이브러리/엔진    │
        │      pip install cognee              │
        │      (353,000줄의 실제 두뇌)         │
        └──────────────────────────────────────┘
                        │
    ┌───────────┬───────┼───────┬───────────┬──────────┐
    ▼           ▼       ▼       ▼           ▼          ▼
 ① SDK     ② CLI   ③ REST  ④ MCP     ⑤ 플러그인  ⑥ 스킬
 (코드)   (터미널) (HTTP)  (AI도구)   (Claude)   (문서)
```

| # | 형태 | 쓰는 사람 | 설치 | 비고 |
|---|---|---|---|---|
| ① | Python SDK | 개발자 | `pip install cognee` | ⭐ 본체 |
| ② | CLI | 누구나 | 위와 동일 | `cognee-cli` |
| ③ | REST API | 모든 언어 | Docker / `cognee-cli serve` | React·PHP·Java… |
| ④ | MCP 서버 | AI 도구 | `cognee-mcp/` | Claude Desktop·Cursor·Cline |
| ⑤ | 플러그인 | Claude Code / Codex | `claude plugin install` | 별도 저장소 |
| ⑥ | 스킬 | Claude | `.claude/skills/` 7개 | 사용법 지식 |

### MCP 서버

노출 도구는 **`remember` / `recall` / `forget` 3개** (의도적 최소화).

| 전송 방식 | 명령 | 용도 |
|---|---|---|
| stdio | (기본값) | Claude Desktop, Cursor 로컬 |
| SSE | `--transport sse` | 실시간 스트리밍 |
| Streamable HTTP | `--transport http` | ⭐ 웹 배포 권장 |

연결 모드: 로컬 / API(기존 FastAPI 서버) / 클라우드(`--serve-url` 또는 `COGNEE_SERVICE_URL`)

```bash
cd cognee-mcp
uv sync --dev --all-extras --reinstall
source .venv/bin/activate
python src/server.py --transport http
```

### 플러그인

```bash
# Claude Code
claude plugin marketplace add topoteretes/cognee-integrations
claude plugin install cognee-memory@cognee

# Codex (먼저 ~/.codex/config.toml 에 [features] hooks = true)
codex plugin marketplace add topoteretes/cognee-integrations --ref main
codex plugin add cognee@cognee

# OpenClaw
npm install @cognee/cognee-openclaw
```

플러그인은 별도 저장소(`topoteretes/cognee-integrations`)에 있다. 이 저장소에는 없다.

### 스킬 (이 저장소에 실제로 존재)

```
.claude/skills/cognee-install/SKILL.md       설치 안내
.claude/skills/cognee-cli/SKILL.md           CLI 사용법
.claude/skills/cognee-docker/SKILL.md        Docker
.claude/skills/cognee-server/SKILL.md        서버 운영
.claude/skills/cognee-permissions/SKILL.md   권한 시스템
.claude/skills/cognee-integrations/SKILL.md  외부 연동
.claude/skills/cognee-community/SKILL.md     커뮤니티 어댑터
cognee/skill.md                              Python API 전반
```

> 💡 **혼동 주의**: 위 스킬은 "cognee 쓰는 법을 Claude에게 가르치는 문서"다.
> 그런데 cognee **안에도 "Skills" 기능**이 따로 있다 (`SearchType.SKILLS`).
> 이건 에이전트의 **절차적 기억** 기능이다. **이름만 같고 전혀 다른 것.**

### 상황별 권장

```
"그냥 체험만"               → cognee-cli demo
"파이썬으로 뭐 만들래"       → ① SDK
"Claude Code가 기억했으면"   → ⑤ 플러그인
"Cursor/Cline에 붙이고파"    → ④ MCP
"React/PHP 앱에 붙이고파"    → ③ REST API
"팀 전체가 쓸 서버"          → Docker + ③
"포크해서 내 제품으로"       → ① + ③ + 소스 개조
```

---

## 12. API 토큰 및 비용

### 결론

```
cognee 가입? → ❌ 없음 (오픈소스)
cognee 토큰? → ❌ 없음
LLM API 키?  → ⭕ 필요 (단, Ollama로 우회 가능)
```

### 필요한 키 3종류

#### ① LLM API 키 — 사실상 필수

| 제공사 | 설정 | 비고 |
|---|---|---|
| **OpenAI** | `LLM_PROVIDER=openai` | 기본값, 가장 안정 |
| Azure OpenAI | `azure` + `LLM_ENDPOINT` + `LLM_API_VERSION` | 기업용 |
| Google Gemini | `gemini` | extra 불필요 |
| Anthropic | `anthropic` | `[anthropic]` extra |
| AWS Bedrock | `bedrock` + AWS 키 | `[aws]` extra |
| **Ollama** | `ollama` | 🆓 로컬, 무료 |
| LM Studio | custom | `LLM_INSTRUCTOR_MODE="json_schema_mode"` 필요 |
| OpenRouter/vLLM | `custom` | 모델 슬러그 수시 변경 주의 |

#### ② 임베딩 키 — ⚠️ 함정

LLM만 바꾸면 임베딩이 몰래 OpenAI로 폴백된다. **반드시 `EMBEDDING_PROVIDER` 도 설정.**

#### ③ 선택적 외부 서비스

AWS(S3/Bedrock/Neptune), Tavily(스크래핑), Neo4j, Postgres, PostHog/OTel,
Slack/GitHub/Linear OAuth (AES-256-GCM 암호화 저장).

### 키 없이 쓰는 3가지 방법

```bash
cognee-cli demo    # ① 데모 (키 0개)
                   # ② 코드 그래프만 (결정론적 파서, LLM 미사용)
                   # ③ Ollama 완전 로컬 (키 0개, 비용 0원, 외부 유출 0)
```

### cognee가 발급하는 토큰 (반대 방향)

cognee를 서버로 띄우면 cognee가 토큰을 발급한다.

```
POST /api/v1/auth/register     회원가입
POST /api/v1/auth/jwt/login    로그인 → JWT
     /api/v1/auth/...          API 키 관리 라우터
```

```bash
ENABLE_BACKEND_ACCESS_CONTROL=True   # 기본값. 인증 필수 + DB 격리
REQUIRE_AUTHENTICATION               # 명시적 오버라이드

# 1인 사용 (인증 끔)
ENABLE_BACKEND_ACCESS_CONTROL=false
REQUIRE_AUTHENTICATION=false
```

> ⚠️ `ENABLE_BACKEND_ACCESS_CONTROL=true` 일 때 `REQUIRE_AUTHENTICATION=false` 는 무시된다.

### 비용 구조

| 작업 | LLM 호출 |
|---|---|
| `cognify()` 개체·관계 추출 | 💰 문서량 비례 (가장 비쌈) |
| `recall()` 답변 생성 | 💰 질문당 1회 |
| 자동 피드백 분석 | 💰 턴당 1회 (`AUTO_FEEDBACK=false` 로 해제) |
| **코드 그래프** | **무료** (결정론적 파서) |
| **`recall()` 자동 라우팅** | **무료** (정규식) |
| **SKILLS 탐색** | **무료** (벡터검색만) |
| 모든 기본 DB | 무료 (로컬 파일) |

**대략적 참고치** (OpenAI gpt-5-mini 기준, 실제 요금은 제공사 가격표 확인 필수):

| 작업 | 규모 | 대략 |
|---|---|---|
| 텍스트 1문장 | — | ~$0.0001 |
| PDF 10페이지 | 1만 토큰 | ~$0.01~0.05 |
| 문서 1,000페이지 | 100만 토큰 | ~$1~5 |
| `recall()` 1회 | — | ~$0.001~0.01 |
| 코드 저장소 통째로 | — | **$0** |

**비용 통제 수단**:

```bash
AUTO_FEEDBACK=false              # 턴당 호출 제거
LLM_RATE_LIMIT_ENABLED=true      # 폭주 방지
LLM_RATE_LIMIT_REQUESTS=60
LLM_RATE_LIMIT_INTERVAL=60
LLM_MODEL="openai/gpt-5-mini"    # 저렴한 모델
# update() 로 증분 처리, 코드는 CODE 경로로
```

### 보안 관련 환경변수

| 변수 | 기본값 | 의미 |
|---|---|---|
| `ACCEPT_LOCAL_FILE_PATH` | True | 로컬 파일 경로 허용 |
| `COGNEE_ALLOWED_LOCAL_FILE_ROOTS` | 미설정 | 로컬 경로 허용 디렉터리 allowlist. 미설정 시 아무 경로나 허용 → **외부 호출자에게 노출되는 서버에서는 반드시 설정** |
| `ALLOW_HTTP_REQUESTS` | True | HTTP 요청 허용 |
| `ALLOW_CYPHER_QUERY` | True | 원시 Cypher 쿼리 허용 |
| `ENABLE_BACKEND_ACCESS_CONTROL` | True | 멀티테넌트 격리 |
| `REQUIRE_AUTHENTICATION` | 미설정 | 인증 명시적 오버라이드 |
| `TELEMETRY_DISABLED` | — | `1` 로 텔레메트리 해제 |

---

## 13. AI 에이전트 구축 활용

### 에이전트에 필요한 5가지

```
① 두뇌 (추론)      → LLM이 해줌 ✅
② 손발 (도구 사용) → 프레임워크가 해줌 ✅
③ 입   (대화)      → 쉬움 ✅
④ 기억 (영구 기억) → ★★★ 여기가 어려움 ★★★
⑤ 학습 (경험→개선) → ★★★ 여기도 어려움 ★★★
```

**cognee가 정확히 ④⑤를 담당한다.**

### 기억이 어려운 8가지 이유와 cognee의 해법

| 문제 | cognee의 해법 |
|---|---|
| 1. 컨텍스트 폭발 | 2층 기억 + 필요한 것만 검색 |
| 2. 요약 시 디테일 유실 | 청크 원문 + 요약 + 그래프 **3중 보존** |
| 3. 벡터DB는 관계를 모름 | **지식 그래프** |
| 4. 뭘 기억하고 뭘 버릴지 | `improve()` + 세션 증류(distillation) |
| 5. 모순된 정보 유입 | `CONTRADICTION_DETECTION` → `contradicts` 엣지 |
| 6. 세션 간 공유 / 동시성 | 세션 캐시 + 분산 락 (`infrastructure/locks/`) |
| 7. 멀티 사용자 격리 | 멀티테넌시 + ACL |
| 8. 왜 그렇게 답했는지 증명 | Provenance 5종 (감사 해시체인 포함) |

### 바로 쓰는 기능 7가지

1. **`agent_memory` 데코레이터** — 함수에 붙이면 기억 생김
   (`cognee/modules/agent_memory/decorator.py`, `examples/guides/agent_memory_quickstart.py`)
2. **세션 메모리** — 멀티턴 대화 (`session_id`)
3. **`AGENTIC_COMPLETION`** — 도구 쓰는 에이전트 루프 (`skills` / `tools` / `max_iter`)
4. **Skills** — 절차적 기억. progressive disclosure + 자기개선 제안
5. **자동 피드백** — 경험에서 학습
6. **개인화** — `PERSONALIZATION_ENABLED=true`, `PERSONALIZATION_INFLUENCE=0.3`
7. **코드 에이전트 기능** — `CODE` 검색, `CODING_RULES`, `tasks/codingagents/`

### 기존 프레임워크 연동

```bash
uv pip install -e ".[langchain]"      # LangChain
uv pip install -e ".[llama-index]"    # LlamaIndex
uv pip install -e ".[graphiti]"       # Graphiti
```

```python
# LangGraph 메모리 노드 예시
async def memory_node(state):
    state["context"] = await cognee.recall(state["질문"], session_id=state["세션"])
    return state

async def save_node(state):
    await cognee.remember(state["대화내용"], session_id=state["세션"])
    return state
```

### 평가

| 항목 | 점수 | 근거 |
|---|---|---|
| 기억 기능 완성도 | ⭐⭐⭐⭐⭐ | 그래프+벡터+세션+피드백+출처 |
| 프로덕션 준비도 | ⭐⭐⭐⭐ | 테스트 823파일, CI 58개, 마이그레이션 48버전, 멀티테넌시 |
| 학습 곡선 | ⭐⭐⭐ | 4개 함수는 쉬움. 설정 플래그 수십 개 (`.env.template` 45KB) |
| 비용 효율 | ⭐⭐⭐ | cognify 비용 큼. 증분업데이트·Ollama·코드경로로 완화 |
| 문서/예제 | ⭐⭐⭐⭐ | 예제 40개, 노트북 6개, docs.cognee.ai, 논문 |
| 커뮤니티 | ⭐⭐⭐⭐ | Discord, r/AIMemory, 커뮤니티 어댑터 저장소 |

---

## 14. React / PHP 연동

### 결론

```
❌ cognee 엔진 자체를 React/PHP로 재작성 → 353,000줄. 사실상 불가능
⭕ cognee를 백엔드로 두고 React/PHP에서 호출 → REST API 완비. 이게 정답
```

### 권장 아키텍처

```
┌─────────────────────────────────────────────────────┐
│  프론트엔드 (React / Next.js / Vue / PHP / Laravel) │
│  ← 당신이 만드는 부분 (UI, 비즈니스 로직, 결제)     │
└─────────────────────────────────────────────────────┘
                      │  HTTP (REST)
                      ▼
┌─────────────────────────────────────────────────────┐
│  cognee API 서버 (Docker, 포트 8000)                │
│  ← 그냥 띄워두면 됨. 건드릴 필요 거의 없음           │
└─────────────────────────────────────────────────────┘
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
      그래프DB     벡터DB     관계형DB
```

### REST 엔드포인트 (`cognee/api/client.py` 확인)

```
POST /api/v1/auth/register          회원가입
POST /api/v1/auth/jwt/login         로그인 → JWT
     /api/v1/auth/...               API 키 관리

POST /api/v1/remember               ⭐ 기억
POST /api/v1/recall                 ⭐ 조회
POST /api/v1/improve                보강
POST /api/v1/forget                 삭제

POST /api/v1/add /cognify /search /memify
     /api/v1/delete /update /validate
     /api/v1/datasets /permissions /users /sessions
     /api/v1/skills /proposals /agents /tools /activity
     /api/v1/ontologies /schema
     /api/v1/settings /configuration /llm
     /api/v1/visualize /export /report /sync /push
     /api/v1/slack /integrations
     /api/v1/responses /checks /health
```

- Swagger 자동 생성: `http://localhost:8000/docs`
- 요청 본문은 **snake_case 와 camelCase 둘 다** 수용
  (`cognee/api/DTO.py` 가 `alias_generator=to_camel`, `populate_by_name=True`)
- `/feedback` 라우트는 **없다**. 피드백은 CLI·SDK 전용.

### React (Next.js) 예시

```ts
// lib/cognee.ts
const BASE = process.env.NEXT_PUBLIC_COGNEE_URL ?? "http://localhost:8000";

async function auth(): Promise<string> {
  const r = await fetch(`${BASE}/api/v1/auth/jwt/login`, {
    method: "POST",
    headers: { "Content-Type": "application/x-www-form-urlencoded" },
    body: new URLSearchParams({ username: "me@x.com", password: "***" }),
  });
  const { access_token } = await r.json();
  return access_token;
}

export async function remember(text: string, sessionId?: string) {
  const token = await auth();
  return fetch(`${BASE}/api/v1/remember`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${token}`,
    },
    body: JSON.stringify({ data: text, datasetName: "main_dataset", sessionId }),
  }).then((r) => r.json());
}

export async function recall(query: string, sessionId?: string) {
  const token = await auth();
  return fetch(`${BASE}/api/v1/recall`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${token}`,
    },
    body: JSON.stringify({ queryText: query, sessionId, topK: 15 }),
  }).then((r) => r.json());
}
```

> ⚠️ **실무 주의**: 토큰 발급을 클라이언트에서 하지 말 것.
> Next.js API Route / Server Action 에서 프록시하고 cognee 서버는 외부에 노출하지 않는다.
>
> ```
> 브라우저 → Next.js 서버 (토큰 보관) → cognee (내부망)  ✅
> 브라우저 → cognee 직접                                   ❌
> ```

### PHP (Laravel) 예시

```php
<?php
namespace App\Services;

use Illuminate\Support\Facades\Http;
use Illuminate\Support\Facades\Cache;

class CogneeService
{
    private string $base;

    public function __construct()
    {
        $this->base = config('services.cognee.url', 'http://localhost:8000');
    }

    private function token(): string
    {
        return Cache::remember('cognee_token', 3000, function () {
            return Http::asForm()
                ->post("{$this->base}/api/v1/auth/jwt/login", [
                    'username' => config('services.cognee.user'),
                    'password' => config('services.cognee.pass'),
                ])
                ->json('access_token');
        });
    }

    public function remember(string $text, ?string $sessionId = null): array
    {
        return Http::withToken($this->token())
            ->post("{$this->base}/api/v1/remember", array_filter([
                'data'        => $text,
                'datasetName' => 'main_dataset',
                'sessionId'   => $sessionId,
            ]))
            ->json();
    }

    public function recall(string $query, ?string $sessionId = null): array
    {
        return Http::withToken($this->token())
            ->post("{$this->base}/api/v1/recall", array_filter([
                'queryText' => $query,
                'sessionId' => $sessionId,
                'topK'      => 15,
            ]))
            ->json();
    }
}
```

순수 PHP도 `curl_init()` 또는 `file_get_contents()` + `stream_context_create()` 로 가능.

### 보너스: 프론트엔드 코드가 이미 있다

`cognee-frontend/` 에 Next.js 16 + React 19 로 만든 완성된 대시보드가 있다.

```json
{
  "next": "^16.3.4",
  "react": "^19.1.2",
  "@mantine/core": "^8.3.13",
  "d3": "^7.9.0",
  "d3-force-3d": "^3.0.6",
  "react-force-graph-2d": "^1.29.0",
  "@tanstack/react-query": "^5.101.2",
  "valibot": "^1.4.2"
}
```

> 이걸 포크해서 제품 UI로 개조하는 것이 가장 빠르다. API 호출·인증·그래프 시각화가
> 이미 구현되어 있다 (47,000줄 참고 구현).

### 공식 SDK

| 언어 | 상태 |
|---|---|
| Python | ⭐ 코어. 1급 지원 |
| TypeScript | 공식 SDK (docs.cognee.ai/typescript) |
| Rust | `topoteretes/cognee-rs` |
| PHP / Java / Go / C# / Ruby | ❌ 없음 → REST 직접 호출 |

---

## 15. 유튜브 강의 제작 가능성

### 법적 검토 — 통과

| 항목 | 상태 |
|---|---|
| 라이선스 | Apache-2.0 |
| 소스코드 화면 노출 | ✅ 허용 |
| 강의로 수익 창출 | ✅ 허용 |
| 코드 수정해서 보여주기 | ✅ 허용 |
| 유료 강의 판매 | ✅ 허용 |
| 로고/에셋 사용 | ⚠️ 상표는 별개 — "비공식" 명시 권장 |
| 필수 의무 | 저작권 고지 + 출처 링크 |

**영상 설명란 권장 문구**:

```
※ cognee는 Topoteretes UG의 Apache-2.0 오픈소스 프로젝트입니다.
   본 강의는 비공식 커뮤니티 콘텐츠입니다.
   원본: https://github.com/topoteretes/cognee
   논문: https://arxiv.org/abs/2505.24478
```

### 시장성

**유리한 점**

1. 주제 타이밍 완벽 — "AI 에이전트 메모리"는 2025~2026 최대 화두. LangChain 강의는 포화
2. 한국어 콘텐츠 사실상 0 → 선점 가능
3. 시각적으로 화려함 — 지식 그래프 시각화는 썸네일 소재로 최상
4. 즉시 결과 — `cognee-cli demo` 로 1분 성과 → 이탈률 낮음
5. 코드가 짧음 — `remember`/`recall` 4줄
6. 저장소 자체가 교재 — 예제 40개, 노트북 6개
7. 수익 연결 자연스러움 — 강의 → 템플릿 → 컨설팅

**불리한 점 & 대응**

| 리스크 | 대응 |
|---|---|
| 타겟이 좁음 (AI 개발자) | 조회수보다 **전환율**로 승부 |
| LLM 비용 발생 | 1~2화에서 Ollama 무료 구성을 먼저 |
| 버전 변화 빠름 (v1.5.4) | 제목에 버전 명시 + 고정 댓글 업데이트 |
| 설정 복잡 (.env 45KB) | 완성된 `.env` 템플릿 배포 |
| 설치 실패 이탈 | `cognee-cli demo` + Docker 원클릭 먼저 |

### 추천 커리큘럼 — 12화

#### 시즌 1: 입문 (조회수 담당)

| 화 | 제목 | 길이 | 내용 |
|---|---|---|---|
| 1 | ChatGPT가 어제 대화를 기억하게 만들었습니다 | 10분 | 문제정의 → `cognee-cli demo` → 그래프 시각화 임팩트 |
| 2 | AI 메모리, 월 0원으로 돌리는 법 (Ollama) | 15분 | 완전 로컬 무료 구성 ★ 이탈 방지 핵심 |
| 3 | RAG는 왜 틀리나 — 지식 그래프 5분 이해 | 12분 | 종이뭉치 vs 노선도 비유 |
| 4 | 내 문서 1,000개를 AI 두뇌로 (실습) | 20분 | PDF·메모·CSV 넣고 `recall()` |

#### 시즌 2: 실전 (전환 담당)

| 화 | 제목 | 길이 | 내용 |
|---|---|---|---|
| 5 | 내 깃허브 레포 전체를 AI에게 먹였습니다 | 18분 | CODE 그래프, 아키텍처 다이어그램 자동생성 ★ 바이럴 |
| 6 | Claude Code / Cursor에 영구기억 달기 (MCP) | 15분 | MCP + 플러그인 설치 |
| 7 | React 앱에 AI 기억 붙이기 (REST API) | 25분 | Next.js + cognee |
| 8 | 세션 메모리 — 대화형 에이전트 만들기 | 20분 | 2층 기억 + 자동 피드백 |

#### 시즌 3: 고급 (전문성 담당)

| 화 | 제목 | 길이 | 내용 |
|---|---|---|---|
| 9 | 온톨로지로 업종 전문 AI 만들기 | 25분 | OWL, `ONTOLOGY_MODE=strict` |
| 10 | AI가 거짓말하는지 증명하기 (Provenance) | 20분 | 출처추적 5종 + 감사 해시체인 |
| 11 | 멀티테넌시 — 고객별 기억 격리 SaaS | 30분 | ACL, 데이터셋 격리 ★ 창업 지향 시청자 |
| 12 | 프로덕션 배포 (Docker + Postgres + Neo4j) | 30분 | 운영, 비용 최적화, 증분 업데이트 |

### 제목·썸네일 전략

```
❌ "cognee 튜토리얼 1편 - 설치하기"        (아무도 cognee를 모른다)
✅ "ChatGPT가 어제 대화를 기억하게 만드는 방법"
✅ "내 깃허브 레포를 AI가 1분에 다 이해했습니다"
✅ "RAG 버리세요. 지식 그래프 쓰세요"
✅ "월 0원으로 돌리는 AI 영구 기억"
```

> **핵심 원칙: 제목에 "cognee"를 넣지 말고 "문제"를 넣어라.**
> 사람들은 cognee를 검색하지 않는다. "AI가 기억 못함"을 검색한다.

### 제작 팁

1. 매 화 첫 15초에 완성 결과(그래프 생성 장면) 먼저
2. 실패 장면도 넣기 — 특히 Ollama 임베딩 미설정 `NoDataError`
3. 강의용 깃허브 저장소 운영 (화별 브랜치/태그, 완성된 `.env` 템플릿) ★ 리드 수집
4. 버전 명시 — "cognee v1.5.4 기준 (2026.10)" + 고정 댓글 업데이트
5. 비용 투명 공개 — "이 영상 실습 비용: $0.37" → 신뢰도 급상승
6. 한국어 + 영어 자막 — 영어권도 cognee 콘텐츠 적음

### 수익 연결

```
유튜브 (무료, 유입)
  ├─▶ 애드센스                    월 10~100만원 (작음)
  ├─▶ 유료 강의 (인프런/Udemy)    5~15만원 × 수백명
  ├─▶ 템플릿/스타터킷             3~30만원
  ├─▶ 전자책/노션                 2~5만원
  ├─▶ ⭐ 기업 컨설팅/구축          수천만원  ← 진짜 돈
  └─▶ 멤버십/커뮤니티             월 1~3만원
```

> **유튜브로 돈을 벌려 하지 말고, 유튜브를 "전문가 증명 장치"로 쓸 것.**
> 영상 12편 = 포트폴리오 = 영업자료.

---

## 16. 수익화 아이디어 10선

### 법적 근거

```
Apache License 2.0
  ✅ 상업적 이용 / 수정 / 배포
  ✅ 특허 사용권 명시적 부여 (MIT에 없는 강점)
  ✅ 클로즈드소스 제품에 포함해 판매
  ✅ SaaS로 서비스 판매
  📋 의무: 라이선스 사본 포함, 저작권 고지 유지(NOTICE.md),
          변경 파일 표시, 상표는 별도 → 자체 브랜딩
```

> ⚠️ README에 *"production ready feature is available as a licenced product"* 문구가 있다.
> **Postgres 그래프 백엔드의 프로덕션 버전은 상용 라이선스**이고 OSS 버전은 데모다.
> → **프로덕션은 Kuzu/Neo4j 를 쓰면 라이선스 문제 없음.**

### 수익화 지도

```
              난이도 낮음 ◄─────────────────► 난이도 높음
         ┌──────────────────┬──────────────────┐
  수익   │  ⑤ 템플릿 판매   │  ① 온프레미스    │
  높음   │  ⑥ 콘텐츠/강의   │  ② 버티컬 SaaS   │
         │                  │  ③ 데이터 정비    │
         ├──────────────────┼──────────────────┤
  수익   │  ⑦ 플러그인/MCP  │  ④ 임베디드 OEM  │
  낮음   │  ⑧ 커뮤니티      │  ⑨ 오디트 도구   │
         │  ⑩ 프리랜싱      │                  │
         └──────────────────┴──────────────────┘
```

---

### 🥇 ① 온프레미스 기업 메모리 구축 — 최고 수익성

**피치**: "귀사 데이터가 OpenAI 서버로 단 한 바이트도 나가지 않는 사내 AI 두뇌를 구축해 드립니다."

**왜 먹히나**: 금융·의료·공공·법무·방산 기업은 "ChatGPT 쓰고 싶은데 사내 문서를 외부
API에 보낼 수 없어서 아무것도 못 하는" 상황이다. cognee + Ollama 가 이걸 뚫는다.

```
┌──── 고객사 내부망 (인터넷 차단) ────────────────┐
│  [사내 문서 10만건]                              │
│         ↓                                        │
│  cognee (온프레미스)                             │
│         ↓                                        │
│  Ollama (로컬 LLM — Llama/Qwen)                  │
│         ↓                                        │
│  Neo4j + PGVector + Postgres (사내 서버)         │
│         ↓                                        │
│  사내 웹 포털 (자체 React UI)                    │
│  ★ 외부 네트워크 호출: 0건                       │
└──────────────────────────────────────────────────┘
```

| 기업 요구사항 | cognee에 이미 있음 |
|---|---|
| 데이터 외부 유출 금지 | ✅ Ollama 완전 로컬 |
| 부서별 접근 권한 분리 | ✅ 멀티테넌시 + ACL |
| "AI가 왜 그렇게 답했나" 감사 | ✅ Provenance 5종 + 해시체인 감사원장 |
| 온프레미스 설치 | ✅ Docker Compose |
| 기존 DB 연동 | ✅ Postgres/Neo4j/dlt |
| 모순 데이터 탐지 | ✅ `CONTRADICTION_DETECTION` |
| 문서 수정 시 저비용 갱신 | ✅ 증분 업데이트 |

**가격 (참고치)**

| 항목 | 금액 |
|---|---|
| PoC (2~4주, 샘플 1천건) | 500만 ~ 1,500만원 |
| 본 구축 (2~4개월) | 3,000만 ~ 1억원 |
| 연 유지보수 (구축비 15~20%) | 500만 ~ 2,000만원/년 |
| 추가 데이터소스 연동 (건당) | 300만 ~ 1,000만원 |
| 사내 교육 (2일) | 300만 ~ 500만원 |

**타겟 산업 우선순위**: 금융 → 의료 → 법무 → 공공 → 제조 R&D

**리스크 대응**

| 리스크 | 대응 |
|---|---|
| 영업 사이클 6~12개월 | PoC를 작게 쪼개 빠른 성과 |
| 로컬 LLM 품질 < GPT | Qwen2.5-72B 등 대형 모델 + 온톨로지 보강 |
| GPU 인프라 비용 | 고객사 부담 명시 + 사양 가이드 제공 |
| 레퍼런스 없으면 수주 불가 | 무료/저가 1호 고객 확보 후 사례화 |

---

### 🥈 ② 업종 특화 SaaS (버티컬) — 확장성 최고

**전략**: cognee는 범용이고, 범용은 아무에게도 안 팔린다. **업종 하나를 골라 그 업종
말을 하는 제품**으로 만든다.

**핵심 무기는 온톨로지**:

```python
config = {
  "ontology_config": {
    "ontology_file_path": "./부동산_온톨로지.owl",
    "ontology_mode": "strict",   # 온톨로지에 없는 개체는 버림 → 잡음 제거
  }
}
```

**버티컬 후보 5개**

| 업종 | 입력 | 핵심 기능 | 타겟 | 가격 |
|---|---|---|---|---|
| 🏢 부동산 | 등기부등본, 임대차계약서, 중개대상물확인서, 판례 | "이 계약서에 불리한 조항?", "이 물건 분쟁 이력?" | 중개법인, 자산운용사 | 월 10~50만원/지점 |
| ⚖️ 법무 | 사건 서면, 판례, 법령, 상담 녹취 | "유사 과거 사건?", "이 쟁점에서 이긴 논리?" | 중소 로펌 | 월 30~200만원 |
| 🏥 의료 | 임상 가이드라인, 논문, 사내 프로토콜 | "이 조건에 맞는 최신 가이드라인?" | 병원, 제약사 MSL | 월 100~500만원 |
| 💼 세무/회계 | 세법, 예규, 판례, 과거 신고서 | "이 거래 구조의 세무 리스크?" | 세무·회계법인 | 월 20~100만원 |
| 🏭 제조 R&D | 특허, 설계도, 시험성적서, 실패 보고서 | "충돌하는 특허?", "과거 유사 실패 사례?" | 제조 대기업 R&D | 연 수천만원 |

> ⚠️ 의료는 의료기기 인증 이슈 → "진단"이 아니라 "문헌 검색"으로 포지셔닝.
> ⚖️ 법무는 "관계 추론"의 가치가 가장 커서 cognee 적합도가 가장 높다.

**가격 구조 예시**

```
Free        문서 100건, 질의 50회/월           0원        → 리드 수집
Pro         문서 5,000건, 질의 1,000회/월      월 9만원
Team        문서 5만건, 사용자 10명            월 49만원
Enterprise  무제한 + 온프레미스 + SLA          연 2,000만원~ (①과 연결)
```

**구현 스택**

```
프론트: Next.js (cognee-frontend/ 포크 개조)
백엔드: cognee API (Docker) + 얇은 BFF
DB:     Postgres (관계형+PGVector) + Neo4j (그래프)
인증:   cognee 내장 JWT 또는 Clerk/Supabase
결제:   토스페이먼츠 / Stripe
배포:   distributed/deploy/ 템플릿 (Fly/Railway/Render)
```

> ⭐ 멀티테넌시가 이미 구현되어 있는 것이 결정적이다. SaaS에서 "고객 A 데이터가
> 고객 B에게 보이면" 사업이 끝난다. 직접 만들면 3~6개월.

---

### 🥉 ③ 데이터 정비 + 그래프 구축 서비스 — 진입 가장 빠름

**피치**: "귀사의 흩어진 10만 개 문서를 AI가 바로 쓸 수 있는 지식 그래프로 정리해 드립니다."

기업의 진짜 문제는 "AI가 없다"가 아니라 "데이터가 쓰레기장"이다.
노션 + 슬랙 + 구글드라이브 + 지라 + 깃허브 + 이메일이 서로 연결되지 않아 아무도
전체를 모른다. cognee가 이걸 그래프 하나로 합친다.

| 패키지 | 내용 | 가격 |
|---|---|---|
| 진단 (1주) | 데이터 현황 감사 + 그래프 설계안 + 비용 산정 | 200~500만원 |
| 구축 (4~8주) | 전체 수집 + 온톨로지 설계 + 그래프 구축 + 시각화 | 1,500~5,000만원 |
| 인수인계 | 운영 매뉴얼 + 교육 + 증분 갱신 체계 | 300~800만원 |
| 구독 운영 | 신규 문서 자동 수집·갱신 | 월 100~500만원 |

**세일즈 무기 2개**

1. **그래프 시각화 = 즉각적 "와우"**
   회의실에서 "귀사 노션 문서 500개를 넣었습니다. 보세요" → D3 포스 그래프 전개
   → "이 노드가 귀사 결제 모듈이고 이 선들이 의존 관계입니다" → 임원: "이걸 우리가 몰랐네"

2. **모순 탐지 = 숨은 문제 발굴** (`CONTRADICTION_DETECTION=true`)
   ```
   "귀사 문서에서 서로 모순되는 사실 47쌍을 발견했습니다.
    예) 2024-03 문서: '배포는 수동 승인 필수'
        2025-01 문서: '배포는 자동화됨'
    → 어느 게 맞습니까?"
   ```
   → "AI 도입"이 아니라 **"리스크 발굴"** 로 팔린다. 예산 승인이 훨씬 쉽다.

---

### ④ 임베디드 / OEM

이미 고객이 있는 SaaS에 "AI 기억 기능"을 붙여주는 사업.

```
타겟: 그룹웨어, CRM, ERP, LMS, 상담 솔루션, 사내 위키 벤더
제안: "귀사 제품에 AI 검색/기억 모듈을 3개월 내 탑재.
       Apache-2.0이라 라이선스 비용 0원"
수익: 개발비 3,000만~1억 + RevShare 또는 라이선스 fee
```

벤더 입장에서는 "AI 인력 채용 1년 + 개발 1년 = 2년" 대신 "3개월, 확정 비용, 종속 없음".

**가장 쉬운 타겟: 고객상담(CS) 솔루션** — 세션메모리 + 그래프로
"이 고객은 3개월 전 같은 문제로 문의, 그때 해결책은 X, 이후 Y 불만 제기" →
상담 시간 단축 = 직접적 ROI 수치 제시 가능.

---

### ⑤ 템플릿 / 보일러플레이트 판매 — 리스크 0, 즉시 시작

```
📦 "cognee 업종별 AI 메모리 스타터킷"
  ├─ Docker Compose 원클릭 구성
  ├─ 완성된 .env 템플릿 (45KB 지옥 해결)
  ├─ Next.js 프론트엔드 (채팅 + 그래프 시각화)
  ├─ 업종 온톨로지 OWL 파일
  ├─ 샘플 데이터 + 시드 스크립트
  ├─ 인증 + 멀티테넌시 설정 완료
  ├─ 배포 가이드 (Fly/Railway/자체서버)
  └─ 한글 문서 + 영상 설명
```

| 상품 | 가격 | 채널 |
|---|---|---|
| 기본 스타터킷 | 5~15만원 | Gumroad, 레몬스퀴지, 자체몰 |
| 업종 특화 (법무/의료/부동산/세무) | 20~50만원 | 동일 |
| 전체 번들 + 1시간 상담 | 80~150만원 | 동일 |
| 화이트라벨 재판매권 | 300~500만원 | 직접 영업 |

**왜 팔리나** — 개발자가 cognee를 처음 켰을 때 겪는 지옥:

1. `.env.template` 45KB → 뭘 채울지 모름
2. 플래그 수십 개 → 뭘 켜고 끌지 모름
3. extras 조합 → `docling-full` + `codegraph` 충돌
4. Ollama 설정 → 임베딩 미설정으로 `NoDataError`
5. 멀티테넌시 → 백엔드 미지원이면 하드 에러
6. 프론트 연동 → 처음부터 만들어야 함

→ "주말 이틀 + 삽질"을 10만원에 사는 셈. 충분히 팔린다.

**장점**: 초기 투자 거의 0, 무한 복제, ⑥ 콘텐츠와 완벽 결합, ①②③ 리드 수집 채널.

---

### ⑥ 콘텐츠 / 교육 — 권위 확보 엔진

```
유튜브 (무료)            → 권위 + 리드
  ↓ 유료 강의             5~15만원 × 수백명
  ↓ 전자책/노션           2~5만원
  ↓ 템플릿 (⑤)           5~50만원
  ↓ 멤버십/커뮤니티       월 1~3만원
  ↓ ★ 기업 구축/컨설팅    수천만원  ← 진짜 돈
```

> 콘텐츠는 수익이 아니라 **신뢰 자산**이다. 영상 12편 = 포트폴리오 = 영업자료.
> 애드센스 월 50만원 < 그 영상 보고 들어온 구축 1건 5,000만원.

---

### ⑦ 플러그인 / MCP 생태계

```
- 한국형 데이터소스 커넥터 (카카오워크, 네이버웍스, 잔디, 플로우)
- 업종 온톨로지 마켓플레이스
- 특화 리트리버 (판례검색, 의료문헌)
- cognee-community 저장소 기여 → "cognee 컨트리뷰터" 타이틀

수익: 유료 커넥터 (월 1~10만원), 스폰서십, ①②③ 유입
```

---

### ⑧ 커뮤니티 / 미디어 — 장기 포석

```
- 한국 AI 메모리 커뮤니티 운영 (디스코드/카톡)
- 뉴스레터: "AI 메모리 위클리"
- 밋업/세미나 주최 (스폰서 유치)
- 번역 기여 (README_ko.md 이미 존재 → 문서 전체 한글화)

수익: 직접 수익은 작음. ①②③의 리드 파이프라인.
```

---

### ⑨ AI 거버넌스 / 오디트 도구 — 틈새 고가

cognee의 **Provenance 5종 + 감사 해시체인**을 활용한 AI 감사 도구.

```
제품: "AI 답변 감사 시스템"
  - AI 답변 근거를 문서 청크 단위로 추적 (Edge Evidence)
  - 해시체인으로 위조 불가 이력 (Audit Ledger)
  - 접근 권한 시각화 (Memory Provenance)
  - 모순 데이터 리포트 (Contradiction)

타겟: 금융감독 대응팀, 내부감사, DPO
가격: 연 2,000만~1억원
```

**타이밍**: 2025~2026 EU AI Act 발효, 각국 AI 규제 본격화 → "AI가 왜 그렇게 판단했는지
설명하라"는 법적 요구 증가 → 설명가능성이 규제 요건이 됨.

> cognee는 이미 5종 추적 + 해시체인을 갖고 있다. "감사 제품"으로 포장만 하면 된다.
> **경쟁자가 거의 없다** — 대부분의 RAG 솔루션은 출처 추적이 허술하다.

---

### ⑩ 프리랜싱 / 긱 — 즉시 현금화

```
플랫폼: Upwork, Toptal, 위시켓, 크몽, 프리모아

"LangChain RAG 챗봇 → 지식 그래프로 업그레이드"      300~1,000만원
"cognee 온프레미스 설치 + 세팅"                      200~500만원
"우리 문서를 AI가 검색하게 해주세요"                 500~2,000만원
"Claude Code / Cursor에 팀 메모리 붙이기"            100~300만원

장점: 오늘 시작 가능, 레퍼런스 확보
단점: 시간 판매 모델 = 확장 안 됨 → ①②로 전환 필요
```

---

### 종합 비교표

| # | 아이디어 | 초기투자 | 진입속도 | 수익규모 | 확장성 | 리스크 | 추천도 |
|---|---|---|---|---|---|---|---|
| ① | 온프레미스 구축 | 중 | 느림 | 매우 큼 | 중 | 중 | ⭐⭐⭐⭐⭐ |
| ② | 버티컬 SaaS | 큼 | 느림 | 큼 | 매우 큼 | 큼 | ⭐⭐⭐⭐⭐ |
| ③ | 데이터 정비 | 작음 | 빠름 | 큼 | 중 | 작음 | ⭐⭐⭐⭐⭐ |
| ④ | OEM/임베디드 | 중 | 중 | 큼 | 중 | 중 | ⭐⭐⭐ |
| ⑤ | 템플릿 판매 | 매우 작음 | 매우 빠름 | 작음~중 | 큼 | 매우 작음 | ⭐⭐⭐⭐ |
| ⑥ | 콘텐츠/교육 | 작음 | 빠름 | 작음~중 | 큼 | 작음 | ⭐⭐⭐⭐⭐ |
| ⑦ | 플러그인 | 작음 | 중 | 작음 | 중 | 작음 | ⭐⭐⭐ |
| ⑧ | 커뮤니티 | 작음 | 느림 | 작음 | 중 | 작음 | ⭐⭐⭐ |
| ⑨ | 감사/거버넌스 | 중 | 느림 | 매우 큼 | 중 | 큼 | ⭐⭐⭐⭐ |
| ⑩ | 프리랜싱 | 0 | 즉시 | 중 | 없음 | 작음 | ⭐⭐⭐ |

---

## 17. 실행 로드맵

### Phase 1 (0~2개월) — 자산 만들기

```
1. cognee 완전 숙달
   - examples 40개 + notebooks 6개 전부 실행
   - Ollama 완전 로컬 구성 성공 (★ 핵심 무기)

2. 데모 3개 제작
   - "내 문서 1000개 → 그래프"           (범용)
   - "깃허브 레포 → 아키텍처 다이어그램"  (개발자용)
   - "모순 탐지 리포트"                   (기업용 ★ 가장 강력)

3. 포크 개조
   - cognee-frontend/ 포크 → 자체 브랜딩 UI
   - Docker Compose 원클릭 패키지
```

**Phase 1 산출물 = 모든 수익 모델의 공통 자산**

### Phase 2 (2~4개월) — 신뢰 쌓기 + 첫 현금

```
4. ⑥ 유튜브 시작 (주 1편, 12편 목표) — 1~4화 먼저
5. ⑤ 템플릿 판매 (Gumroad/크몽) — 월 50~300만원 목표
6. ⑩ 프리랜싱 1~2건 수주 — 레퍼런스 + 현금흐름
```

**목표: 월 200~500만원 + 레퍼런스 2건 + 구독자 1,000명**

### Phase 3 (4~8개월) — 고단가 전환

```
7. ③ 데이터 정비 PoC 수주 — "모순 탐지 리포트" 데모로 영업, 진단 200~500만원부터
8. ① 온프레미스 첫 구축 — PoC 500만원 → 본구축 3,000만원
```

**목표: 구축 1~2건 = 3,000만~1억원**

### Phase 4 (8~18개월) — 확장

```
9.  ② 버티컬 SaaS 1개 출시 — ①③에서 수요 많았던 업종 선택
10. ④ OEM 또는 ⑨ 감사도구로 확장
```

### 최종 권고 3줄

```
1. Ollama 완전 로컬 구성을 숙달하라.
   "데이터가 밖으로 안 나갑니다"가 한국 시장 최강의 무기다.

2. 유튜브 + 템플릿으로 신뢰와 현금흐름을 먼저 만들라.
   영상 12편 = 영업자료. 템플릿 = 생활비.

3. 그 신뢰로 ③ 데이터 정비 → ① 온프레미스 구축으로 올라가라.
   "모순 탐지 리포트" 데모 하나가 수천만원 수주를 만든다.
```

---

## 18. 리스크 및 주의사항

### 경쟁 — cognee 본사가 직접 한다

```
Cognee Cloud (관리형 SaaS)를 본사가 이미 운영 중.

→ 범용 "cognee 호스팅"으로는 본사와 경쟁 = 승산 없음
→ 차별점 필수:
   ✅ 온프레미스 + 한국어 + 업종 특화 + 로컬 지원
   ❌ 그냥 cognee를 클라우드에 띄워서 팔기
```

### 라이선스 경계선

```
✅ Apache-2.0 범위 내: 자유롭게 상업화
⚠️ "Postgres 그래프 프로덕션"은 상용 라이선스 → OSS는 데모
   → 프로덕션은 Kuzu/Neo4j 사용 (문제 없음)
⚠️ 상표(cognee 이름/로고) → 자체 브랜딩 필수
✅ NOTICE.md 고지 + 변경사항 표시 의무 준수
```

### 기술 리스크

| 리스크 | 대응 |
|---|---|
| 버전 변화 빠름 (v1.5.4) | 특정 버전 pin + 업그레이드 계획 |
| 로컬 LLM 품질 < GPT | 온톨로지로 보강, 기대치 사전 조율 |
| LLM 비용 예측 어려움 | 반드시 소량 측정 후 견적 |
| 백엔드별 멀티테넌시 제약 | Kuzu/Neo4j + LanceDB/PGVector 조합 사용 |
| 설정 복잡도 (.env 45KB) | 기본값으로 시작, 템플릿화 |
| `docling-full` ↔ `codegraph` 충돌 | 둘 중 하나만 설치 |
| Postgres 그래프는 데모 | 프로덕션은 Kuzu/Neo4j |

### 시장 리스크

```
- 기업 영업 사이클 6~12개월 → ⑤⑥⑩으로 현금흐름 메우기
- "AI 피로감" → 기술이 아니라 "비용 절감 / 리스크 발굴"로 팔기
- 레퍼런스 없으면 수주 불가 → 1호 고객은 저가/무료로라도 확보
```

### 벤치마크 해석 주의

BEAM 평가 결과는 아래와 같지만 **서로 다른 대화·모델·절차**로 측정되었다.

| BEAM 컨텍스트 | 점수 (0–1) | 범위 |
|---|---|---|
| 100K 토큰 | **0.79** | 고정 하이브리드 검색, 1개 held-out 대화의 20문항 4라운드 |
| 10M 토큰 | **0.67** | 탐색적 결과, 동일 문항셋에서 질문유형 라우팅 선택·채점, 5라운드 평균 |

README가 직접 "다른 시스템과 비교하기 전에 방법론·모델·한계·재현 방법을 읽으라"고
경고한다. 출처: `cognee/eval_framework/beam/REPORT.md`

### 흔한 함정 체크리스트

| 증상 | 원인 | 해결 |
|---|---|---|
| `NoDataError` (Ollama) | 임베딩이 OpenAI로 폴백 | `EMBEDDING_PROVIDER` 도 설정, `HUGGINGFACE_TOKENIZER` 확인 |
| 검색/recall 이 느림 | 턴당 자동 피드백 LLM 호출 | `AUTO_FEEDBACK=false` (단 `CACHING=true` 유지) |
| LM Studio 구조화 출력 실패 | instructor 모드 미지정 | `LLM_INSTRUCTOR_MODE="json_schema_mode"` |
| 예상 못한 OpenAI 호출 | LLM/임베딩 중 하나만 설정 | 둘 다 설정 |
| 검색 결과가 빈 배열 | 권한 없음 (정보 누출 방지 설계) | 데이터셋 권한·사용자 접근권 확인 |
| `EnvironmentError` (핸들러) | 백엔드가 데이터셋 격리 미지원 | 백엔드 변경 또는 `ENABLE_BACKEND_ACCESS_CONTROL=false` |
| Docker에서 로컬 DB 접속 실패 | 호스트명 | `DB_HOST=host.docker.internal` |
| Rate limit 에러 | 요청 폭주 | `LLM_RATE_LIMIT_ENABLED=true` + 한도 조정 |
| `forget --all` 로 데이터 날림 | 확인 프롬프트 없음 | `delete --all` 사용 또는 백업 선행 |

**디버깅 설정**

```bash
LITELLM_LOG="DEBUG"      # LLM 상세 로그 (기본 ERROR)
ENV="development"        # 또는 "debug"
TELEMETRY_DISABLED=1
pip install cognee[debug]  # debugpy
```

---

## 19. 참고 링크

### 저장소

- **이 포크**: https://github.com/bmshin94/cognee
- **원본**: https://github.com/topoteretes/cognee
- 커뮤니티 어댑터/애드온: https://github.com/topoteretes/cognee-community
- 통합·플러그인: https://github.com/topoteretes/cognee-integrations
- Claude Code 플러그인 설정: https://github.com/topoteretes/cognee-integrations/tree/main/integrations/claude-code
- Rust SDK: https://github.com/topoteretes/cognee-rs
- OpenClaw 플러그인: https://www.npmjs.com/package/@cognee/cognee-openclaw

### 문서

- 공식 문서: https://docs.cognee.ai/
- 아키텍처: https://docs.cognee.ai/core-concepts/architecture
- 세션 라이프사이클: https://docs.cognee.ai/core-concepts/sessions-and-caching
- `remember`: https://docs.cognee.ai/core-concepts/main-operations/remember
- `recall`: https://docs.cognee.ai/core-concepts/main-operations/recall
- `improve`: https://docs.cognee.ai/core-concepts/main-operations/improve
- `forget`: https://docs.cognee.ai/core-concepts/main-operations/forget
- 설치: https://docs.cognee.ai/getting-started/installation
- LLM 제공사: https://docs.cognee.ai/setup-configuration/llm-providers
- 로컬 Ollama: https://docs.cognee.ai/guides/local-ollama
- 온톨로지: https://docs.cognee.ai/guides/ontology-support
- 권한: https://docs.cognee.ai/setup-configuration/permissions
- 그래프 시각화: https://docs.cognee.ai/guides/graph-visualization
- MCP: https://docs.cognee.ai/cognee-mcp/mcp-overview
- CLI: https://docs.cognee.ai/cognee-cli/overview
- Python API: https://docs.cognee.ai/python-api
- TypeScript SDK: https://docs.cognee.ai/typescript/getting-started
- REST API: https://docs.cognee.ai/api-reference/introduction
- 메모리 시스템 이전: https://docs.cognee.ai/examples/migrate-memory-systems
- Cognee Cloud: https://docs.cognee.ai/cognee-cloud/overview

### 연구

- 논문: https://arxiv.org/abs/2505.24478
- BEAM 리포트: `cognee/eval_framework/beam/REPORT.md`

### 커뮤니티

- Discord: https://discord.gg/NQPKmU5CCg
- Reddit r/AIMemory: https://www.reddit.com/r/AIMemory/
- 데모 영상: https://www.youtube.com/watch?v=8hmqS2Y5RVQ
- MCP 데모 영상: https://www.youtube.com/watch?v=1bezuvLwJmw
- 공식 사이트: https://cognee.ai
- Company Brain: https://www.cognee.ai/company-brain

### 저장소 내부 참고 경로

| 경로 | 내용 |
|---|---|
| `README.md` / `README_ko.md` | 프로젝트 개요 (한국어판 존재) |
| `CLAUDE.md` (51KB) | AI 에이전트용 상세 작업 지침 |
| `AGENTS.md` | 에이전트 지침 |
| `.env.template` (45KB) | 전체 환경변수 정본 템플릿 |
| `docker-compose.yml` | API + UI + MCP + Postgres + Neo4j |
| `examples/README.md` | 예제 색인 |
| `examples/demos/company_brain_demo.py` | ★ 실전 데모 |
| `examples/guides/local_ollama_example.py` | 완전 무료 구성 |
| `examples/guides/code_graph_example.py` | 코드 그래프 |
| `examples/advanced_guides/remember_recall_improve_example.py` | 메모리 API 전반 |
| `notebooks/tutorial.ipynb` | 튜토리얼 노트북 |
| `cognee-starter-kit/` | 신규 프로젝트 템플릿 |
| `cognee-mcp/README.md` | MCP 서버 가이드 |
| `distributed/deploy/README.md` | 배포 템플릿 |
| `docs/minimal-docker-compose.md` | 최소 Docker 구성 |
| `docs/ollama_models.md` | Ollama 모델 가이드 |
| `docs/recall-vs-search.md` | recall vs search 상세 |
| `.claude/skills/` | Claude Code용 스킬 7개 |
| `cognee/eval_framework/beam/REPORT.md` | 벤치마크 방법론 |

---

## 부록: 한 장 요약

```
┌──────────────────────────────────────────────────────────────┐
│  cognee = AI 에이전트용 오픈소스 장기기억 플랫폼               │
├──────────────────────────────────────────────────────────────┤
│  문제    │ LLM은 대화창 닫으면 다 잊는다. RAG는 "비슷한 것"만 │
│          │ 찾고 "관계"를 모른다.                              │
│  해법    │ 문서 → LLM이 개체·관계 추출 → 지식그래프+벡터DB   │
│  API     │ remember / recall / improve / forget  (4개)        │
│  구조    │ 세션캐시(단기) + 지식그래프(장기), 백그라운드 승격 │
│  입력    │ 텍스트·코드·PDF·CSV·이미지·오디오·비디오·URL·repo  │
│  검색    │ 21종 전략 (그래프/벡터/키워드/시간/코드/에이전트)   │
│  백엔드  │ 그래프 5종 · 벡터 4종+커뮤니티 · 관계형 2종         │
│          │ LLM 6사 (OpenAI/Anthropic/Gemini/Ollama/…)         │
│  부가    │ 멀티테넌시·ACL·출처추적5종·모순탐지·증분업데이트   │
│          │ 스킬(절차적기억)·온톨로지·자동피드백·개인화        │
│  규모    │ 40만줄, 테스트 823파일, CI 58개, v1.5.4            │
│  라이선스│ Apache-2.0 → 상업적 이용·수정·재배포 전부 가능 ✅  │
│  활용    │ ① 내 전용 AI 비서  ② 회사 브레인                  │
│          │ ③ Claude Code 기억력  ④ AI 서비스 창업 기반 엔진  │
│  수익화  │ 🥇 온프레미스 구축  🥈 버티컬 SaaS  🥉 데이터 정비 │
│          │ + 템플릿/강의/OEM/감사도구/프리랜싱               │
└──────────────────────────────────────────────────────────────┘
```

---

*작성: Claude Code · 2026-10-08 · cognee v1.5.4 기준*
*저장소: https://github.com/bmshin94/cognee*
