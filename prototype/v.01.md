# OntoMind v0.1 — Personal Ontology Framework (LLM × Me)

> 목적: **“나와 LLM이 같은 세계관에서 생각하도록”** 내 삶/업무 도메인을 온톨로지로 구조화하고, 이를 프롬프트·RAG·에이전트 흐름에 일관되게 연결하는 최소한의 규격을 정의한다.

---

## 0) 디자인 원칙

* **Pragmatic first**: RDF/OWL은 참고하되, v0.1은 가벼운 스키마/DSL로 시작.
* **Human-in-the-loop**: 자동 추론보다 *내 의사결정 보조*에 초점.
* **Composable**: 도메인별 모듈(Work/People/Project/Knowledge)을 조립가능하게.
* **Traceable**: 근거(Source)와 확신도(Confidence)를 남김.
* **LLM-ready**: 스키마 → JSON I/O → 프롬프트 템플릿을 표준화.

---

## 1) 핵심 개념 (Information Model)

* **Entity**: 세계를 구성하는 대상 (예: Project, Person, Widget, Document)
* **Attribute**: 엔티티의 속성 (타입/제약 포함)
* **Relation**: 엔티티 간 의미있는 연결 (유형/방향/카디널리티)
* **Event**: 시간 기반 변화/행위 (예: 미팅, 릴리스, 결정)
* **Intent**: 내가 LLM에게 시키는 목적/요청의 분류 (예: Summarize, Plan, Diagnose)
* **Context**: 질의 시 유효 범위/상황 (시간/스코프/역할/권한)
* **Policy**: 제약, 금지, 우선순위 규칙 (예: 개인정보, 톤&스타일)
* **Source**: 근거 문서/링크/저자/시간
* **Confidence**: 지식의 신뢰도(0\~1)

---

## 2) 네이밍 & 네임스페이스

* 전역 prefix: `om:` (OntoMind)
* 도메인 prefix 예: `work:`, `people:`, `project:`, `fe:`(frontend), `kb:`(knowledge)
* ID 규칙: `prefix:Type/slug#version` (예: `project:Project/spaceone#2024-09`)

---

## 3) 스키마 (TypeScript 표현)

```ts
// ontomind.schema.ts (핵심 타입)
export type EntityID = string; // e.g., "project:Project/spaceone#2024-09"

export interface AttributeSpec {
  name: string;
  type: 'string'|'number'|'boolean'|'date'|'enum'|'url'|'markdown'|'id'|'json';
  required?: boolean;
  enumValues?: string[];
}

export interface EntityType {
  type: string;              // e.g., 'Project'
  attributes: AttributeSpec[];
}

export interface RelationType {
  name: string;              // e.g., 'owns', 'depends_on', 'reports_to'
  from: string;              // EntityType.type
  to: string;                // EntityType.type
  cardinality?: '1-1'|'1-N'|'N-N';
  directed?: boolean;        // default true
}

export interface EventType {
  type: string;              // e.g., 'Decision', 'Release', 'Meeting'
  attributes: AttributeSpec[];
}

export interface PolicyRule {
  name: string;
  description: string;
  when?: Record<string, any>;    // 조건
  forbid?: string[];              // 금지 행동
  prefer?: string[];              // 선호 규칙
}

export interface IntentSpec {
  name: string;   // e.g., 'Plan', 'Summarize', 'Diagnose'
  input: Record<string, string>;  // 입력 파라미터 스키마
  output: Record<string, string>; // LLM이 반환해야 할 JSON 스키마(라이트)
}

export interface ContextSpec {
  name: string;  // e.g., 'Workspace:Admin', 'Week:2025-W33'
  filters?: Record<string, any>;  // 엔티티/관계 필터
  role?: 'me'|'lead'|'coach'|'analyst';
}
```

---

## 4) DSL (OML: OntoMind Markup Language, YAML)

```yaml
# ontomind.oml.yml
version: 0.1
namespaces: [work, people, project, fe, kb]

entities:
  - type: Project
    attributes:
      - { name: name, type: string, required: true }
      - { name: goal, type: markdown }
      - { name: status, type: enum, enumValues: [draft, active, paused, done] }
      - { name: start_date, type: date }
      - { name: end_date, type: date }

  - type: Person
    attributes:
      - { name: display_name, type: string, required: true }
      - { name: role, type: enum, enumValues: [fe, be, pm, designer, leader] }
      - { name: strengths, type: json }
      - { name: cautions, type: json }

  - type: Widget
    attributes:
      - { name: name, type: string, required: true }
      - { name: options, type: json }
      - { name: version, type: string }

relations:
  - { name: owns, from: Person, to: Project, cardinality: '1-N' }
  - { name: depends_on, from: Project, to: Project, cardinality: 'N-N' }
  - { name: uses_widget, from: Project, to: Widget, cardinality: 'N-N' }
  - { name: collaborates_with, from: Person, to: Person, cardinality: 'N-N', directed: false }

intents:
  - name: Plan
    input:  { target: 'id:Project', horizon: 'string' }
    output: { milestones: 'json', risks: 'json', next_actions: 'json' }
  - name: Diagnose
    input:  { target: 'id:Project' }
    output: { symptoms: 'json', causes: 'json', remedies: 'json' }
  - name: Summarize
    input:  { scope: 'json' }
    output: { summary: 'markdown', highlights: 'json' }

contexts:
  - name: Workspace:Admin
    role: me
    filters: { visibility: 'admin' }

policies:
  - name: Privacy.Person
    description: '개인 민감정보 노출 금지, 필요 시 익명화'
    forbid: ['leak_sensitive_personal_info']

views: # 선택적: LLM 출력 포맷/테이블 정의
  - name: ProjectHealth
    of: Project
    columns: [name, status, start_date, end_date]
```

---

## 5) 지식 인스턴스(팩트) 예시 (JSON Lines)

`/ontomind/instances/projects.jsonl`

```json
{"id":"project:Project/spaceone#2024-09","type":"Project","name":"SpaceONE Multi-Tenancy","status":"active","goal":"Admin/Workspace 스코프 분리 및 권한 전환 안정화","start_date":"2023-09-01","source":"notion:doc/123","confidence":0.9}
{"id":"project:Project/console-vq-devtools#2025-05","type":"Project","name":"Console Vue Query Devtools","status":"done","goal":"Vue2.7용 커스텀 Devtools 출시","start_date":"2025-04-10","end_date":"2025-05-05","source":"github:repo/console-vq-devtools","confidence":0.95}
```

`/ontomind/instances/people.jsonl`

```json
{"id":"people:Person/yongtae","type":"Person","display_name":"박용태","role":"fe","strengths":["system thinking","query cache design"],"cautions":["context narrowing"],"confidence":1}
```

`/ontomind/instances/relations.jsonl`

```json
{"type":"owns","from":"people:Person/yongtae","to":"project:Project/console-vq-devtools#2025-05","source":"slack:msg/abc","confidence":0.9}
{"type":"depends_on","from":"project:Project/spaceone#2024-09","to":"project:Project/console-vq-devtools#2025-05","confidence":0.6}
```

---

## 6) LLM 연동 — 표준 프롬프트 템플릿

### 6.1 System Prompt (고정)

```
You are OntoMind Agent. Always reason within the OntoMind ontology.
- Use the provided schema (entities/relations/intents/contexts/policies).
- Validate inputs/outputs against the schema.
- Cite sources and include confidence scores when possible.
- Prefer structured JSON per IntentSpec.output.
```

### 6.2 Tool/Function Spec (예: Diagnose)

```json
{
  "name": "om_diagnose",
  "description": "Diagnose a project's health within OntoMind ontology.",
  "parameters": {
    "type": "object",
    "properties": {
      "target": {"type":"string", "description":"Project EntityID"}
    },
    "required": ["target"]
  }
}
```

**LLM 응답 JSON 스펙 (IntentSpec.output에 맞춤)**

```json
{
  "symptoms": ["..."],
  "causes": ["..."],
  "remedies": ["..."],
  "_meta": {"sources": ["..."], "confidence": 0.8}
}
```

### 6.3 Grounding 프롬프트 스니펫

```
<SCHEMA>
{...OML parsed summary...}
</SCHEMA>
<CONTEXT name="Workspace:Admin">{...}
</CONTEXT>
<OBJECTS>
- load: /instances/projects.jsonl (filter id in ...)
- load: /instances/relations.jsonl (join ...)
</OBJECTS>
<TASK intent="Diagnose" target="project:Project/spaceone#2024-09" />
```

---

## 7) RAG 파이프라인 (하이브리드)

1. **Symbolic filter**: Context/Policy로 스코프 축소 (엔티티 타입, 상태 등)
2. **Vector retrieve**: 관련 문서/메모를 임베딩 검색
3. **Graph join**: relations.jsonl 로 이웃 엔티티 합류
4. **Prompt assemble**: 스키마 + 인스턴스 + 증거(Source) 묶어 LLM 호출
5. **Validator**: IntentSpec.output JSON 스키마 검사 → 실패 시 재질의

---

## 8) 워크플로우 레시피

* **Daily Review (Summarize)**

  1. 오늘 이벤트/노트 인입 → instances/events.jsonl append
  2. Context = Week\:YYYY-WW
  3. Intent=Summarize → `summary.md` 생성 (근거/다음 액션 포함)

* **Project Plan 갱신 (Plan)**

  1. Project ID 선택 → 관련 위젯/의존 프로젝트 조인
  2. 위험/가정 추출 → Remedial actions 제안
  3. Notion/Jira 동기화

* **Coaching Guide (Diagnose + Policy)**

  1. 사람-프로젝트-피드백 그래프 불러오기
  2. Privacy.Policy 적용
  3. 코칭 대화 스크립트 생성

---

## 9) 저장소 구조 (Git 관리)

```
ontomind/
  schemas/
    ontomind.schema.ts
    ontomind.oml.yml
  instances/
    projects.jsonl
    people.jsonl
    relations.jsonl
    events.jsonl
    notes.jsonl
  prompts/
    system.txt
    intents/
      diagnose.json
      plan.json
      summarize.json
  pipelines/
    rag.hybrid.md
  policies/
    privacy.yml
```

---

## 10) 검증 & 거버넌스

* **Schema Lint**: OML → JSON Schema 변환 후 lint
* **ID Uniqueness**: 프리커밋 훅으로 중복 탐지
* **Confidence Rule**: 0.7 미만은 ‘가설’ 태그로 표기
* **Change Log**: OML 변경 시 `CHANGELOG.md` 자동 생성

---

## 11) 최소 CLI 설계 (Node/TS)

* `om validate` : OML/instances 검증
* `om assemble --intent Diagnose --target <EntityID>`: 프롬프트 빌드 출력
* `om export --view ProjectHealth`: 표/CSV 생성

---

## 12) v0.1 → v0.2 로드맵

* v0.1: 스키마/DSL/샘플 인스턴스/프롬프트 템플릿/수동 RAG
* v0.2: 간단 CLI, JSON Schema 기반 유효성 검사, 노션/지라 싱크 어댑터
* v0.3: 임베딩 인덱스 + 하이브리드 검색, 그래프 뷰어(Sankey/Force)
* v0.4: 코칭/피드백 온톨로지 확장(리더십 메타층)

---

## 13) 적용 예시 — FE 아키텍처 온톨로지 (요약)

* **Entity**: Service, Resource, QueryKeyPolicy, Widget, DataSource
* **Relation**: service-uses-resource, widget-binds-datasource, querykey-applies-to-resource
* **Intent**: Diagnose (캐시 미스/중복 요청), Plan (마이그레이션 단계), Summarize (릴리즈 노트)

---

## 14) 시작 체크리스트

* [ ] ontomind 저장소 생성
* [ ] `ontomind.oml.yml` 베이스 커밋
* [ ] people/projects/relations 3개 인스턴스 파일 작성(최소 5개 팩트)
* [ ] system 프롬프트 고정 텍스트 확정
* [ ] Daily/Summarize 파이프라인 수동 실행해 1주 검증

---

OntoMind는 \*“내 삶/일의 의미 좌표계”\*를 LLM이 공유하도록 만드는 얇고 실용적인 온톨로지 프레임워크다. v0.1은 파일·CLI·프롬프트만으로도 충분히 효과를 발휘한다.

