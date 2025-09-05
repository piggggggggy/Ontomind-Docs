# OntoMind — Architecture & Usage Guide v0.1

본 문서는 OntoMind P0(나만의 LLM 챗 인터페이스)를 위해 설계한 **Ontology→IR→Codegen→Validator→Prompt Assembler→LLM Port** 파이프라인을 **처음 보는 사람도 이해하고 바로 사용할 수 있게** 상세히 설명합니다.

---

## 0. 목표와 철학

* **목표**: 내가 정의한 온톨로지(TBox/ABox) 기반의 DSL로 정보를 구조화하고, 이를 프롬프트에 녹여 **일관된 LLM 상호작용**을 만든다.
* **핵심 원칙**

  1. **선언적 정의(TBox)** → **구현 자동화**(IR, Codegen, Prompt)
  2. **검증 가능한 구조**(JSON Schema, Ajv, 경량 의미론 체크)
  3. **플러깅 가능한 포트**(provider adapter, streaming, mock)
  4. **버전 가능/확장 가능**(ns\:slug\@version, tbox 분할/병합)

---

## 1. 용어 및 데이터 구조

### 1.1 TBox / ABox / OML / IR

* **TBox**: 클래스·관계·인텐트 등 **스키마(개념)** 정의.
* **ABox**: 실제 **사실 데이터(인스턴스)**. `instances/*.jsonl`에 엔티티/관계를 기록.
* **OML(OntoMind Markup Language)**: YAML DSL. `data/schemas/*`에 보관.
* **IR(Intermediate Representation)**: OML을 파싱한 **중간표현 JSON**. `data/generated/ir.json`.

### 1.2 네임스페이스/슬러그/버전

* **id 포맷**: `ns:slug@version` (예: `core:project@1.0.0`)
* **ABox 엔티티 id 포맷 권장**: `ns:Class/slug#version` (예: `project:Project/ontomind-core#1.0.0`)
* **버전 생략** 시: 최신 버전으로 해석(파서/검증기 구현 의도에 따름).

### 1.3 핵심 섹션

* **classes**: 타입(Thing, Person, Project …)과 속성 정의.
* **relationTypes**: 클래스 간 관계(예: `part_of`, `depends_on`).
* **intents**: 모델에게 수행시킬 **역할/작업** 계약(입력/출력 구조).
* **contexts**: 프롬프트 조립 시 활성화할 **역할/필터**.
* **policies**: 프롬프트 정책(금지/선호 규칙 등).

---

## 2. 저장소 레이아웃

```
ontomind/
├─ packages/
│  ├─ core/        # 타입/ID/에러 등 코어 유틸
│  ├─ dsl/         # OML 파서, IR 변환
│  ├─ validator/   # Ajv + 경량 의미론 검증
│  ├─ prompt/      # system/user/toolSchema 조립
│  └─ port/        # LLM provider adapters (openai/mock 등)
├─ tools/          # CLI 스크립트 모음
├─ apps/
│  └─ chat/        # (선택) Next API 라우트 예시
└─ data/
   ├─ schemas/     # OML 소스(TBox/ABox 스키마)
   ├─ instances/   # ABox 인스턴스(JSONL)
   └─ generated/   # IR, schema, prompt-summary 등 산출물
```

---

## 3. `data/` 디렉토리 규약

### 3.1 `data/schemas`

* `ontomind.oml.yml`: 루트 OML(TBox). 분리 운영 시 `tbox/*.yml`과 병합.
* `tbox/*.yml`: 도메인별 분리(예: `core.yml`, `intents.yml`).
* `meta.schema.yml`: OML 문법 메타-스키마(JSON Schema). PR 시 1차 문법 검증.
* `abox.entity.schema.yml`, `abox.relation.schema.yml`: ABox 기록 시 형식 검증용.

### 3.2 `data/instances`

* JSON Lines 포맷: 한 줄 = 하나의 엔티티/관계.
* **엔티티 레코드 예**

  ```json
  {"id":"project:Project/ontomind-core#1.0.0","class":{"ns":"core","slug":"project"},"data":{"name":"OntoMind Core","status":"active"}}
  ```
* **관계 레코드 예**

  ```json
  {"type":{"ns":"core","slug":"depends-on"},"from":"project:Project/ontomind-core#1.0.0","to":"project:Project/prompt-kit#1.0.0"}
  ```

### 3.3 `data/generated`

* `ir.json`: OML→IR 변환 결과.
* `intents.schema.json`: Intent I/O JSON Schema 번들(키: `intent:diagnose@1.0.0`).
* `prompt-summary.json`: 프롬프트 조립용 요약(클래스·관계·인텐트·컨텍스트 등).
* (선택) `classes.schema.json`, `types.d.ts`: 클래스별 JSON Schema/타입 선언.

---

## 4. 패키지 별 역할

### 4.1 `@ontomind/core`

* **types.ts**: IR 타입(OMClass/OMRelationType/OMIntent/OMIR …), 프리미티브, 카드inality 등.
* **id.ts**: `idToString`, `keyById`(중복 체크) 등 식별자 유틸.
* **errors.ts**: `ValidationError` 등 공통 에러 타입.

### 4.2 `@ontomind/dsl`

* **parse.ts**: `parseOML()`

  * 루트 OML + `tbox/*.yml` 병합(결정적 정렬).
  * `meta.schema.yml` 1차 검증(선택) 후 `toIR` 호출.
* **normalize.ts**: `toIR()`

  * 필수 키 확인, `relationTypes`의 domain/range가 클래스에 존재하는지 **경량 의미론 검증**.
  * 성공 시 IR 생성.

### 4.3 `@ontomind/validator`

* **ajv.ts**

  * Ajv 인스턴스 생성. *(draft-2020-12 사용 시 `Ajv2020` 권장)*
* **entity.ts / relation.ts**

  * ABox 단건 라이트 검증(필수 키, required 속성, enum, 문자열/숫자 등).
  * 필요 시 `validateEntityWithSchema()`로 codegen 산출 JSON Schema 2차 검증.
* **intent.ts**

  * `validateIntentIO(intentKey, payload, which, intentsSchemaPath)`
  * `intents.schema.json`에서 해당 인텐트의 **input/output 스키마**로 유효성 검사.
* **instances.ts**

  * `validateJsonlFile`, `validateInstancesDir`: JSONL 스트림을 행 단위로 검증/리포트.

### 4.4 `@ontomind/prompt`

* **assemble.ts**: `assemblePrompt(opts, src)` → **PromptBundle**

  * `system`: 온톨로지/관계/컨텍스트/인텐트 라벨 포함.
  * `user`: 인텐트명, 타겟 id, (선택) evidence.
  * `toolSchema`: 해당 인텐트의 **출력 스키마**(모델의 JSON 출력 가이드로 사용).
  * instances 디렉토리에서 타겟 id 기준 원홉 증거 수집.
* **templates.ts**: system/user 렌더러(필요 시 커스터마이즈).
* **filters.ts**: prompt-summary, intents.schema 로드 + instances 수집.

### 4.5 `@ontomind/port`

* **providers/openai.ts**: OpenAI 호환 어댑터

  * `response_format: json_schema` 지원 모델과 연동.
  * 429/5xx **재시도(백오프)**, 에러 메시지 수집, 키 마스킹 유틸.
* **providers/mock.ts**: **모의(Mock) 프로바이더)**

  * `toolSchema`를 바탕으로 **스키마에 맞는 더미 응답** 생성 → 검증 라인 전체 실험 가능.
* **llm.ts**: `callLLM(bundle, opts)`

  * `bundle(system/user/toolSchema)`를 메시지로 구성해 호출.
  * 비스트리밍: 응답 JSON 파싱 → `validateIntentOutput()`로 검사 후 반환.
  * 스트리밍: `ReadableStream` 그대로 반환(프론트에서 처리).
* **validate.ts**: `validateIntentOutput()` 헬퍼.
* **streaming.ts**: SSE 텍스트 스트림 파서(간단형).

---

## 5. 도구(`tools/`)와 스크립트

### 5.1 파서·병합·코드젠·조립

* `parse-oml.ts` → `pnpm oml:parse`: OML→IR 생성(`data/generated/ir.json`).
* `merge-oml.ts` → `pnpm oml:merge`: 사람이 확인용 병합 문서 덤프.
* `codegen.ts` → `pnpm oml:codegen`: `intents.schema.json`, `prompt-summary.json` 등 산출.
* `assemble.ts` → `pnpm oml:assemble <intentKey> <targets>`: `tmp/prompt.*`/`tmp/tool.schema.json` 생성.

### 5.2 검증·호출

* `validate-instances.ts` → `pnpm oml:validate:instances`: `data/instances/*.jsonl` 일괄 검증.
* `validate-intent.ts` → `pnpm oml:validate:intent <intentKey> <input|output> <file.json>`: 인텐트 I/O 검증.
* `smoke-openai.ts` → `pnpm oml:smoke`: OpenAI 통신 스모크 테스트.
* `chat-call.ts` → `pnpm oml:chat:cli` / `pnpm oml:chat:cli:mock`: **앱 없이** 조립→호출→검증.

---

## 6. 엔드투엔드 실행 흐름(E2E)

1. **스키마 준비**: `data/schemas/ontomind.oml.yml` + `tbox/*.yml`
2. **IR 생성**: `pnpm -r build` → `pnpm oml:parse`
3. **코드젠**: `pnpm oml:codegen` → `data/generated` 생성
4. (선택) **ABox 채우기**: `data/instances/*.jsonl`
5. **프롬프트 조립**: `pnpm oml:assemble "intent:diagnose@1.0.0" "project:Project/sample-1"`
6. **모의 호출(추천)**: `pnpm oml:chat:cli:mock` → 스키마 일치 응답으로 전체 라인 검증
7. **실제 호출**: `OPENAI_API_KEY` 설정 후 `pnpm oml:chat:cli`

   * 429(Quota) 발생 시 mock로 개발 계속 → 키/모델 준비 후 전환

---

## 7. 베스트 프랙티스 & 운영 팁

* **버전 전략**: 스키마 변경 시 `@<ver>` 상승. ABox는 `#<ver>`로 마이그레이션 가이드 남기기.
* **네임스페이스 설계**: `core`(범용) + 도메인별(ns). 공통 클래스/관계는 `core`에 배치.
* **검증 단계화**: (1) meta.schema.yml 문법 → (2) IR 의미론 → (3) Ajv(JSON Schema).
* **스키마 주도 개발**: Intent output을 먼저 정하고, Prompt 템플릿/LLM 포트를 그에 맞춤.
* **키 보안**: 콘솔/로그에 API 키 **마스킹**(prefix…suffix). `.env`로 주입.
* **429/Quota**: 재시도 백오프 + mock 프로바이더로 개발 지속.
* **인스턴스 관리**: JSONL은 Git에 커밋 가능(소량). 대량/민감 데이터는 DB/스토리지로 분리 고려.

---

## 8. 자주 묻는 질문(FAQ)

**Q1. ABox 없이도 동작하나요?**
네. 프롬프트 조립은 되며, evidence가 비어 더 “일반적” 답을 내놓을 수 있습니다.

**Q2. 왜 Intent가 중요하죠?**
LLM의 **역할·입출력 계약**을 명시합니다. `response_format: json_schema`와 결합해 **정형 출력**을 유도합니다.

**Q3. Draft-2020-12 스키마 에러(메타 스키마 없음)가 나요.**
Ajv 기본 대신 `Ajv2020`을 사용하거나, draft-07로 내리세요(권장: Ajv2020).

**Q4. 429 Too Many Requests와 Quota exceeded의 차이?**
둘 다 429이나 메시지가 다릅니다. rate-limit은 잠시 후 재시도하면 해결, **quota exceeded**는 결제/한도 증설 필요.

**Q5. 왜 mock 프로바이더가 필요하죠?**
키/한도/네트워크가 막혀도 **스키마→프롬프트→검증** 전 과정을 개발/테스트할 수 있게 합니다.

---

## 9. 새로운 도메인 추가 가이드(샘플)

1. `data/schemas/tbox/<domain>.yml` 생성:

```yml
classes:
  - id: { ns: sales, slug: opportunity, version: "1.0.0" }
    label: Opportunity
    extends: [{ ns: core, slug: thing, version: "1.0.0" }]
    properties:
      - { id: { ns: sales, slug: amount, version: "1.0.0" }, label: Amount, type: number, required: true }
      - { id: { ns: sales, slug: stage, version: "1.0.0" }, label: Stage, type: { enum: [new, won, lost] } }

relationTypes:
  - id: { ns: sales, slug: owned-by, version: "1.0.0" }
    label: owned_by
    domain: { ns: sales, slug: opportunity }
    range:  { ns: core,  slug: person }
    cardinality: "N-N"
```

2. (선택) `tbox/intents.yml`에 인텐트 추가.
3. `pnpm oml:parse && pnpm oml:codegen` → 오류 없으면 성공.

---

## 10. Next.js API 연동(선택)

* `/api/assemble` → 프론트에서 프롬프트 번들 조회.
* `/api/chat` → 번들을 받아 실제 호출(스트리밍/논-스트리밍) 후 결과 반환.

프론트엔드는 Step 6에서 **채팅 패널 + 온톨로지 뷰 + 검증 배지**로 구성 예정.

---

## 11. 체크리스트(설치부터 호출까지)

* [ ] `data/schemas/`에 TBox/Intent 정의 존재, `namespaces`에 `intent` 포함
* [ ] `pnpm -r build` 성공
* [ ] `pnpm oml:parse` → `data/generated/ir.json` 생성
* [ ] `pnpm oml:codegen` → `intents.schema.json`, `prompt-summary.json` 생성
* [ ] (선택) `data/instances/*.jsonl` 추가 → `pnpm oml:validate:instances`
* [ ] `pnpm oml:assemble "intent:diagnose@1.0.0" "project:Project/sample-1"`
* [ ] **Mock**: `pnpm oml:chat:cli:mock` 성공
* [ ] **OpenAI**: `OPENAI_API_KEY` 설정 후 `pnpm oml:chat:cli` (Quota 가능성 체크)

---

## 12. 다음 단계(로드맵)

* **Step 6 UI**: 채팅/증거 하이라이트/스키마 검증 배지/컨텍스트 토글.
* **그래프 검증**: 카디널리티/역관계 글로벌 검사(전체 인덱싱).
* **프로바이더 확장**: Anthropic, vLLM, serverless 프록시.
* **데이터 소스**: DB/검색 인덱스/문서 파이프라인과 ABox 자동 동기화.
* **권한/정책**: policy 기반 redaction/role별 프롬프트 가드.

---

### 부록 A — 샘플 인텐트 Output(JSON)

```json
{
  "symptoms": ["프로젝트 일정 지연", "리소스 부족"],
  "causes": ["요구사항 변경", "의사결정 지연"],
  "remedies": ["변경관리 강화", "대체 인력 확보"]
}
```

### 부록 B — 문제 해결 가이드(요약)

* **메타 스키마 에러**: Ajv2020 사용.
* **domain/range not found**: `classes`에 해당 클래스 id가 존재하는지 확인.
* **intent not found**: `intents.yml` 병합 경로/네임스페이스 포함 확인.
* **429**: rate-limit → 백오프, quota → 키/플랜 점검 or mock 사용.
