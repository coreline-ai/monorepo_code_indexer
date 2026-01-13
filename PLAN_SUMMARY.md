# Monorepo 코드 검색 엔진 아키텍처 정리

회사 monorepo를 대상으로 **코드를 잘 찾아주고, 의미까지 이해하는 검색 엔진**을 만드는 전체 구조 정리입니다.[web:212][web:215]  
각 단계의 역할, 영어 용어, 필요한 기술 스택과 데이터 흐름을 한 문서에 모았습니다.[web:219][web:223]

---

## 0. 전체 구조 개요

### 단계 목록

1. 코드 수집 (Git 동기화)  
2. 인덱서 (코드 블록 단위로 쪼개고 구조화)  
3. 텍스트 검색 인덱스 (키워드/정규식)  
4. 임베딩/벡터 인덱스 (의미 기반 검색)  
5. 검색 API 레이어  
6. LLM/RAG (옵션: 코드 설명·요약·Q&A)  
7. UI / IDE 통합  
8. 운영·모니터링·인프라[web:215][web:219]

### 데이터 플로우(텍스트 다이어그램)

    [ 1. Git Monorepo ]
               |
               v
          [ 2. Indexer ]
               |
               v
       +---------------------+       +----------------------+
       | 3. Text Index       |       | 4. Vector Index      |
       | (keyword / regex)   |       | (embeddings / sem.)  |
       +---------------------+       +----------------------+
               ^                               ^
               |                               |
               +----------[ 5. Search API ]----+
                                   |
                            [ 6. LLM / RAG ]
                                   |
                            [ 7. UI / IDE ]
                                   |
                      [ 8. Infra / Monitoring ]

---

## 1. 코드 수집 (Git 동기화)

**영어 용어**  
- Code ingestion, Repository sync, Git mirroring[web:213][web:218]

**역할**  
- monorepo에서 변경된 코드만 가져와 인덱서에 넘기는 단계.

**작업 흐름**

- Git 서버(GitHub Enterprise, GitLab, Gitea 등)에 Webhook 등록.[web:213][web:218]  
- `push` 이벤트 발생 시:
  - 변경된 리포/브랜치/커밋 정보를 수신.
  - 해당 리포를 read‑only로 `git fetch` / `git pull` 해서 로컬 mirror 갱신.
- 변경된 파일 목록만 계산해서 인덱싱 큐에 넣음.

**기술 스택 / 장비**

- 백엔드: FastAPI, NestJS, Go 등.  
- Git: `git` CLI + 사내 Git 서버.  
- 서버(MVP): 2–4 vCPU, 8GB RAM, SSD.[web:218]

---

## 2. 인덱서 (코드 블록 단위로 쪼개고 구조화)

**영어 용어**  
- Indexer, Code indexing pipeline, Code parsing & indexing[web:219][web:252]

**역할**  
- 파일 단위 텍스트를 **함수/클래스 같은 코드 블록 단위**로 나누고, 메타데이터를 붙여 구조화.

**처리 예시 (TypeScript 파일)**

변경된 파일: `apps/api/payment_service.ts`

1. 언어 판별  
   - 확장자 `.ts` → TypeScript.

2. 파싱 및 블록 추출  
   - Tree‑sitter로 AST 생성.[web:247][web:253]  
   - `function_declaration`, `class_declaration` 노드를 찾아 각각을 코드 블록으로 인식.[web:255]

3. 구조화된 레코드(JSON 형태)

    {
      "id": "codeblock_1",
      "repo": "company-monorepo",
      "file_path": "apps/api/payment_service.ts",
      "start_line": 1,
      "end_line": 4,
      "symbol_name": "cancelPayment",
      "language": "typescript",
      "raw_code": "export async function cancelPayment(orderId: string) {\n  // 결제 취소 로직\n}\n"
    }

    {
      "id": "codeblock_2",
      "repo": "company-monorepo",
      "file_path": "apps/api/payment_service.ts",
      "start_line": 6,
      "end_line": 9,
      "symbol_name": "refundPayment",
      "language": "typescript",
      "raw_code": "export async function refundPayment(orderId: string) {\n  // 환불 처리 로직\n}\n"
    }

이 레코드가 3번(텍스트 인덱스)과 4번(벡터 인덱스)의 공통 입력이 됩니다.[web:219][web:220]

**사용 라이브러리**

- Tree‑sitter + 언어별 grammar[web:247][web:253][web:255]  
- 심볼/참조 확장: LSP, ctags, Sourcegraph LSIF/SCIP 참고.[web:248][web:252][web:256]

**장비**

- 인덱싱 워커: 4–8 vCPU, 16GB RAM 정도.[web:252]

---

## 3. 텍스트 검색 인덱스 (키워드/정규식)

**영어 용어**  
- Full‑text index, Inverted index, Keyword search index[web:215][web:223]

**역할**  
- `cancelPayment`, `"UserNotFoundException"`, 정규식 등 **문자/토큰 기반 검색** 담당.

**인덱싱 방식**

- 문서 필드:
  - 본문: `raw_code`
  - 메타: `file_path`, `symbol_name`, `language` 등.
- 검색 엔진이 역색인(inverted index)을 만들어 토큰 → 문서/위치 매핑을 유지.[web:215][web:223]

**추천 엔진**

- Elasticsearch / OpenSearch / Meilisearch[web:215][web:216][web:223]  
- 로컬 보조: ripgrep(`rg`) 같은 초고속 CLI 검색.[web:218]

**쿼리 예시**

- `cancelPayment lang:ts path:apps/api`  
- `/logger.*error/ lang:java`

**하드웨어**

- 8–16GB RAM + SSD 권장. 리포 크기에 따라 샤딩/수평 확장 가능.[web:215][web:223]

---

## 4. 임베딩/벡터 인덱스 (의미 기반 검색)

**영어 용어**  
- Embedding index, Vector index, Semantic search index[web:229][web:232][web:240]

**역할**  
- “결제 취소 로직”, “JWT 검증하는 코드”처럼 **자연어/의미 기반 쿼리**에 맞는 코드 블록을 찾아주는 역할.

**임베딩 모델이 하는 일**

- 입력: 텍스트 또는 코드 블록 문자열.  
- 출력: 고정 길이 벡터(예: 768차원 float 배열).[web:231][web:235]  
- 비슷한 의미의 텍스트/코드는 벡터 공간에서 서로 가깝게, 다른 의미는 멀리 위치하도록 학습.[web:229][web:232][web:241]

**인덱싱 흐름**

1. 인덱서 레코드의 `raw_code`를 임베딩 모델에 넣어 벡터 생성.[web:229][web:233]

2. 벡터DB에 저장할 구조 예:

    {
      "id": "codeblock_1",
      "embedding": [0.12, -0.03, 0.55, 0.08],
      "metadata": {
        "repo": "company-monorepo",
        "file_path": "apps/api/payment_service.ts",
        "symbol_name": "cancelPayment",
        "language": "typescript",
        "start_line": 1,
        "end_line": 4
      }
    }

3. 검색 시:
   - 사용자 쿼리 `"결제 취소 로직"`을 임베딩 → 쿼리 벡터 생성.[web:219][web:232]
   - 벡터DB에서 쿼리 벡터와 가장 가까운 코드 블록 Top‑K를 조회 → `id` 기준으로 메타데이터/코드 조회.[web:219][web:224]

**스택**

- 임베딩 모델: 코드/텍스트용 임베딩 LLM들.[web:219][web:231][web:233]  
- 라이브러리: Hugging Face Transformers, SentenceTransformers.[web:240]  
- 벡터DB: Qdrant, Weaviate, Milvus, pgvector(Postgres 확장).[web:229][web:232][web:241]

**하드웨어**

- 초기 대량 인덱싱 시 GPU 1장(T4/A10급) 있으면 속도 유리, CPU만으로도 가능.[web:229]  
- 검색 자체는 CPU로 충분.

---

## 5. 검색 API 레이어

**영어 용어**  
- Search API layer, Search service, Query routing & ranking layer[web:216][web:224]

**역할**  
- 텍스트 인덱스(3)와 벡터 인덱스(4)를 묶어 **하나의 `/search` HTTP API**로 제공.

**요청 처리 흐름**

1. 클라이언트 요청 예:

    GET /search?q=결제 취소 로직&lang=ts&path=apps/api

2. Search API 내부:

- 키워드 검색:
  - 텍스트 엔진에 `q` + 필터(lang/path) 전달 → 후보 A.[web:215][web:223]
- 의미 검색:
  - `q`를 임베딩 → 벡터DB에서 Top‑K → 후보 B.[web:219][web:232]
- 랭킹/병합:
  - A/B 결과를 통합해 최종 스코어 계산  
    (BM25 점수 + 코사인 유사도 + 최근 변경 여부 + 파일 위치 가중치 등).[web:214][web:224]

3. 최종 결과를 JSON으로 반환.

**스택**

- 백엔드: FastAPI, NestJS, Go Fiber 등.[web:216][web:223]  
- 인증/권한: 사내 SSO(OIDC/SAML) + repo/폴더 ACL 연동.[web:173][web:222]

---

## 6. LLM/RAG (옵션: 코드 설명·요약·Q&A)

**영어 용어**  
- RAG layer (Retrieval‑Augmented Generation), LLM reasoning layer[web:234][web:237]

**역할**  
- 검색 결과로 얻은 코드/문서들을 LLM에 넣어:
  - 함수 설명
  - 파일/모듈 구조 요약
  - 버그 분석/후보 코드 추천  
  같은 **추가 텍스트 답변**을 생성.[web:219][web:221]

**RAG 흐름(코드 기준)**

1. Search API로 코드 블록 Top‑K 조회.  
2. 코드와 주변 컨텍스트를 프롬프트에 구성:

    [코드 컨텍스트]
    apps/api/payment_service.ts (1-40 lines)
    ...

    [질문]
    "이 코드에서 결제 취소 로직이 어떻게 동작하는지 설명해줘."

3. LLM이 설명/요약/리팩토링 제안 생성 → UI에 표시.[web:219][web:221][web:234]

**스택**

- LLM: 로컬 LLM(Ollama + Llama 계열) 또는 사내 LLM 서버.[web:219]  
- 용도: 코드 설명/요약, diff 요약, 리뷰 코멘트 초안, 리팩토링 제안 등.[web:221]

---

## 7. UI / IDE 통합

**영어 용어**  
- Search UI / Web frontend, IDE extension / plugin, Developer experience (DX) layer[web:212][web:222]

**역할**  
- 개발자가 실제로 쓰는 인터페이스.

**Web UI**

- 스택: Next.js/React + Tailwind 등.[web:212][web:215]  
- 기능:
  - 검색 바, 언어/경로/서비스 필터.
  - 검색 결과 리스트 + 코드 하이라이트.
  - “파일 열기”, “함수 설명(LLM)” 버튼.

**IDE 플러그인**

- VS Code Extension, JetBrains Plugin:  
  - 사이드바 검색 패널.  
  - 현재 파일/커서 기준 검색.  
  - 결과 클릭 시 해당 파일/라인으로 점프.[web:212][web:222]

---

## 8. 운영·모니터링·인프라

**영어 용어**  
- Infrastructure & DevOps, Observability, Platform & operations[web:173][web:219]

**역할**  
- 전체 시스템을 안정적으로 운영/확장/모니터링.

**인프라 구성**

- 컨테이너: Docker, Docker Compose, 필요 시 Kubernetes(k3s 등).[web:219]  
- 서비스 단위:
  - Git 수집
  - 인덱서 워커
  - 검색 API
  - 텍스트 검색 엔진
  - 벡터DB
  - LLM 서버(옵션)

**모니터링/로그**

- 모니터링: Prometheus + Grafana, OpenTelemetry.[web:219]  
- 로그: ELK Stack, Loki 등.  
- 주요 지표:
  - 인덱싱 지연 시간
  - 검색 응답 시간/에러율
  - LLM 호출 실패율 등.

---

## 하드웨어 구성 (MVP 기준)

**서버 1대로 시작하는 예**

- 스펙:
  - 8 vCPU  
  - 32GB RAM  
  - NVMe SSD (1TB)[web:215][web:223][web:229]

- 한 서버에 컨테이너로:
  - Git mirror + 수집 API  
  - 인덱서  
  - Meilisearch/Elasticsearch (텍스트 검색)  
  - Qdrant/pgvector (벡터DB)  
  - 검색 API(FastAPI/NestJS)  
  - LLM 서버(Ollama, 옵션)[web:219][web:229]

**규모 커질 때 분리 우선순위**

1. 텍스트 검색 엔진 (ES/Meilisearch)[web:215][web:223]  
2. 벡터DB + LLM 서버[web:229][web:219]  
3. 인덱서 워커[web:252]

---

## 최종 정리

- 1–2단계: **데이터 준비 파이프라인** (수집 + 코드 블록 인덱싱).  
- 3–4단계: **검색 엔진 코어** (키워드 검색 + 의미 검색).  
- 5단계: **검색 프런트** (두 검색을 통합하는 Search API).  
- 6–7단계: **DX 레이어** (LLM 활용 + Web/IDE 통합).  
- 8단계: **플랫폼 레이어** (인프라/모니터링/운영).[web:215][web:219][web:234]
