# Step 0 & 1 Scaffold

```typescript

// =============================================================
// OntoMind — Step 0 & 1 Scaffold (IR + DSL Parser)
// -------------------------------------------------------------
// 이 문서는 P0의 0)목표정의 & 1)파서·IR 구현을 한 번에 붙일 수 있는
// 파일 세트를 담고 있습니다. 각 섹션의 파일을 그대로 생성하세요.
// =============================================================

/*
📁 제안 파일 트리 (모노레포 루트 기준)

ontomind/
├─ packages/
│  ├─ core/
│  │  ├─ src/
│  │  │  ├─ types.ts
│  │  │  ├─ id.ts
│  │  │  └─ errors.ts
│  │  └─ package.json
│  └─ dsl/
│     ├─ src/
│     │  ├─ normalize.ts
│     │  ├─ parse.ts
│     │  └─ index.ts
│     └─ package.json
├─ tools/
│  ├─ merge-oml.ts
│  └─ parse-oml.ts
└─ data/
   ├─ schemas/        # 이미 준비됨 (ontomind.oml.yml + tbox/*)
   └─ generated/      # 산출물 저장 위치 (ir.json 등)
*/

// =============================================================
// 0) Step 0 — 목표 & 수용 기준 (코멘트)
// -------------------------------------------------------------
/*
Acceptance (Step 0 완료 기준)
- data/schemas/ 에 OML(TBox) 존재, meta.schema.yml로 1차 문법 검증
- parse-oml 실행 시: OML → IR 객체 생성 & data/generated/ir.json 저장
- IR에는 classes/relationTypes/intents/contexts/policies가 정상 채워짐
- 기본 의미론 검증 통과: 중복 ID 없음, relationType의 domain/range가 존재
*/

// =============================================================
// 1) packages/core/src/types.ts — IR 타입 정의
// -------------------------------------------------------------
export type NS = string;

export interface OMIdentifier {
  ns: NS;
  slug: string;
  version?: string; // semver-ish, optional for "latest"
}

export type OMStatus = 'draft' | 'active' | 'deprecated';
export type OMCardinality = '1-1' | '1-N' | 'N-N';

export type OMPrimitive =
  | 'string' | 'number' | 'boolean' | 'date' | 'datetime'
  | 'url' | 'markdown' | 'id' | 'json';

export interface OMAnnotation {
  label: string;
  description?: string;
  aliases?: string[];
  tags?: string[];
}

export type OMTypeRef = OMPrimitive | { enum: string[] } | { ref: OMIdentifier };

export interface OMProperty extends OMAnnotation {
  id: OMIdentifier;
  type: OMTypeRef;
  required?: boolean;
  default?: any;
  pattern?: string;
  min?: number;
  max?: number;
  unique?: boolean;
}

export interface OMClass extends OMAnnotation {
  id: OMIdentifier;
  extends?: OMIdentifier[];
  properties: OMProperty[];
  status?: OMStatus;
}

export interface OMRelationType extends OMAnnotation {
  id: OMIdentifier;
  domain: OMIdentifier; // Class id
  range: OMIdentifier;  // Class id
  cardinality?: OMCardinality;
  inverseOf?: OMIdentifier;
  symmetric?: boolean;
  transitive?: boolean;
  status?: OMStatus;
}

export interface OMIntent extends OMAnnotation {
  id: OMIdentifier;
  input: Record<string, string>;  // light schema hints
  output: Record<string, string>; // JSON schema will be generated later
}

export type OMContextRole = 'me' | 'lead' | 'coach' | 'analyst';

export interface OMContext extends OMAnnotation {
  id: OMIdentifier;
  role?: OMContextRole;
  filters?: Record<string, any>;
}

export interface OMPolicy extends OMAnnotation {
  id: OMIdentifier;
  forbid?: string[];
  prefer?: string[];
  when?: Record<string, any>;
}

export interface OMIR {
  version: number; // OML version
  namespaces: string[];
  classes: OMClass[];
  relationTypes: OMRelationType[];
  intents: OMIntent[];
  contexts: OMContext[];
  policies: OMPolicy[];
}

// =============================================================
// 1-1) packages/core/src/id.ts — 식별자 유틸
// -------------------------------------------------------------
import type { OMIdentifier } from './types';

export function idToString(id: OMIdentifier): string {
  const v = id.version ? `@${id.version}` : '';
  return `${id.ns}:${id.slug}${v}`;
}

export function sameId(a: OMIdentifier, b: OMIdentifier): boolean {
  return a.ns === b.ns && a.slug === b.slug && a.version === b.version;
}

export function keyById<T extends { id: OMIdentifier }>(arr: T[]): Map<string, T> {
  const m = new Map<string, T>();
  for (const it of arr) {
    const k = idToString(it.id);
    if (m.has(k)) throw new Error(`Duplicate id: ${k}`);
    m.set(k, it);
  }
  return m;
}

// =============================================================
// 1-2) packages/core/src/errors.ts — 에러 헬퍼
// -------------------------------------------------------------
export class OntoMindError extends Error {
  constructor(message: string, public issues: string[] = []) { super(message); }
}

export class ValidationError extends OntoMindError {}

// =============================================================
// 1-3) packages/dsl/src/normalize.ts — OML → IR 정규화 보조
// -------------------------------------------------------------
import type { OMClass, OMIR, OMIntent, OMPolicy, OMRelationType, OMContext } from '@ontomind/core/src/types';
import { ValidationError } from '@ontomind/core/src/errors';
import { idToString, keyById } from '@ontomind/core/src/id';

export interface RawOML {
  version: number;
  namespaces: string[];
  classes?: OMClass[];
  relationTypes?: OMRelationType[];
  intents?: OMIntent[];
  contexts?: OMContext[];
  policies?: OMPolicy[];
}

export function toIR(doc: RawOML): OMIR {
  if (!doc || typeof doc !== 'object') throw new ValidationError('OML root must be an object');
  if (typeof doc.version !== 'number') throw new ValidationError('OML.version must be a number');
  if (!Array.isArray(doc.namespaces)) throw new ValidationError('OML.namespaces must be an array');

  const classes = (doc.classes ?? []) as OMClass[];
  const relationTypes = (doc.relationTypes ?? []) as OMRelationType[];
  const intents = (doc.intents ?? []) as OMIntent[];
  const contexts = (doc.contexts ?? []) as OMContext[];
  const policies = (doc.policies ?? []) as OMPolicy[];

  const classById = keyById(classes);

  // 의미론 검증: relationType의 domain/range 존재 여부
  const issues: string[] = [];
  for (const rel of relationTypes) {
    const d = idToString(rel.domain);
    const r = idToString(rel.range);
    if (!classById.has(d)) issues.push(`relationType ${idToString(rel.id)}: domain not found: ${d}`);
    if (!classById.has(r)) issues.push(`relationType ${idToString(rel.id)}: range not found: ${r}`);
  }
  if (issues.length) throw new ValidationError('Semantic validation failed', issues);

  return {
    version: doc.version,
    namespaces: doc.namespaces,
    classes,
    relationTypes,
    intents,
    contexts,
    policies,
  };
}

// =============================================================
// 1-4) packages/dsl/src/parse.ts — 파일 머지 + 파싱 + IR 변환
// -------------------------------------------------------------
import fs from 'node:fs';
import path from 'node:path';
import yaml from 'yaml';
import type { OMIR } from '@ontomind/core/src/types';
import { toIR, RawOML } from './normalize';

function readY(file: string) { return yaml.parse(fs.readFileSync(file, 'utf8')); }

function mergeDeep<T extends Record<string, any>>(base: T, extra: T): T {
  const out: any = { ...base };
  for (const k of Object.keys(extra)) {
    const a = (base as any)[k];
    const b = (extra as any)[k];
    if (Array.isArray(a) && Array.isArray(b)) out[k] = [...a, ...b];
    else if (a && typeof a === 'object' && b && typeof b === 'object') out[k] = mergeDeep(a, b);
    else out[k] = b;
  }
  return out;
}

export interface ParseOptions {
  rootDir?: string; // default: process.cwd()
  schemasDir?: string; // default: data/schemas
  tboxDirName?: string; // default: tbox
}

export function parseOML(opts: ParseOptions = {}): OMIR {
  const rootDir = opts.rootDir ?? process.cwd();
  const schemasDir = opts.schemasDir ?? path.join(rootDir, 'data', 'schemas');
  const tboxDir = path.join(schemasDir, opts.tboxDirName ?? 'tbox');

  const rootFile = path.join(schemasDir, 'ontomind.oml.yml');
  if (!fs.existsSync(rootFile)) throw new Error(`Not found: ${rootFile}`);

  let merged: RawOML = readY(rootFile);
  if (fs.existsSync(tboxDir)) {
    const parts = fs.readdirSync(tboxDir)
      .filter(f => f.endsWith('.yml') || f.endsWith('.yaml'))
      .map(f => path.join(tboxDir, f))
      .sort(); // deterministic merge
    for (const f of parts) {
      const doc = readY(f);
      merged = mergeDeep(merged, doc);
    }
  }

  // 기본 정규화 & 의미론 검증 → IR
  const ir = toIR(merged);
  return ir;
}

// =============================================================
// 1-5) packages/dsl/src/index.ts — 공개 API
// -------------------------------------------------------------
export { parseOML } from './parse';
export { toIR, type RawOML } from './normalize';

// =============================================================
// 1-6) tools/merge-oml.ts — (선택) 병합 결과 확인용 스크립트
// -------------------------------------------------------------
import fs from 'node:fs';
import path from 'node:path';
import yaml from 'yaml';
import { parseOML } from '@ontomind/dsl/src/parse';

const rootDir = process.cwd();
const schemasDir = path.join(rootDir, 'data', 'schemas');
const outFile = path.join(schemasDir, 'ontomind.merged.oml.yml');

// parseOML 내부에서 병합하지만, 사람이 보게끔 병합 결과를 저장하고 싶을 때 사용
const irDoc: any = (() => {
  // parseOML은 IR을 반환하므로, 병합 원문이 궁금하면 별도 merge 로직을 두거나
  // 아래처럼 간단히 루트+조각을 다시 병합해서 저장할 수 있습니다.
  const root = yaml.parse(fs.readFileSync(path.join(schemasDir, 'ontomind.oml.yml'), 'utf8'));
  const tboxDir = path.join(schemasDir, 'tbox');
  const files = fs.existsSync(tboxDir)
    ? fs.readdirSync(tboxDir).filter(f => f.endsWith('.yml') || f.endsWith('.yaml')).sort()
    : [];
  const mergeDeep = (a: any, b: any) => {
    const out: any = { ...a };
    for (const k of Object.keys(b)) {
      const A = a?.[k]; const B = b[k];
      if (Array.isArray(A) && Array.isArray(B)) out[k] = [...A, ...B];
      else if (A && typeof A === 'object' && B && typeof B === 'object') out[k] = mergeDeep(A, B);
      else out[k] = B;
    }
    return out;
  };
  return files.reduce((acc, f) => mergeDeep(acc, yaml.parse(fs.readFileSync(path.join(tboxDir, f), 'utf8'))), root);
})();

fs.writeFileSync(outFile, yaml.stringify(irDoc), 'utf8');
console.log(`Merged OML written to ${outFile}`);

// =============================================================
// 1-7) tools/parse-oml.ts — CLI: OML → IR 출력 (ir.json 저장)
// -------------------------------------------------------------
import fs from 'node:fs';
import path from 'node:path';
import { parseOML } from '@ontomind/dsl/src/parse';

const ir = parseOML();
const out = path.join(process.cwd(), 'data', 'generated', 'ir.json');
fs.mkdirSync(path.dirname(out), { recursive: true });
fs.writeFileSync(out, JSON.stringify(ir, null, 2), 'utf8');
console.log(`[OntoMind] IR saved: ${out}`);
console.log(`- classes: ${ir.classes.length}`);
console.log(`- relationTypes: ${ir.relationTypes.length}`);
console.log(`- intents: ${ir.intents.length}`);
console.log(`- contexts: ${ir.contexts.length}`);
console.log(`- policies: ${ir.policies.length}`);

// =============================================================
// 1-8) packages/dsl/test/parse.spec.ts — Vitest 테스트 (옵션)
// -------------------------------------------------------------
import { describe, it, expect } from 'vitest';
import { toIR } from '@ontomind/dsl/src/normalize';

describe('toIR basic semantics', () => {
  it('fails when relation domain/range missing', () => {
    const doc: any = {
      version: 0.1,
      namespaces: ['core'],
      classes: [],
      relationTypes: [
        {
          id: { ns: 'core', slug: 'depends-on' },
          label: 'depends_on',
          domain: { ns: 'core', slug: 'thing' },
          range: { ns: 'core', slug: 'thing' }
        }
      ]
    };
    expect(() => toIR(doc)).toThrowError();
  });

  it('passes with valid class references', () => {
    const doc: any = {
      version: 0.1,
      namespaces: ['core'],
      classes: [
        { id: { ns: 'core', slug: 'thing' }, label: 'Thing', properties: [] }
      ],
      relationTypes: [
        {
          id: { ns: 'core', slug: 'depends-on' },
          label: 'depends_on',
          domain: { ns: 'core', slug: 'thing' },
          range: { ns: 'core', slug: 'thing' }
        }
      ]
    };
    const ir = toIR(doc);
    expect(ir.classes.length).toBe(1);
    expect(ir.relationTypes.length).toBe(1);
  });
});

// =============================================================
// 1-9) 패키지 설정 힌트 (패키지별 package.json)
// -------------------------------------------------------------

/*
packages/core/package.json
{
  "name": "@ontomind/core",
  "version": "0.1.0",
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "files": ["dist"],
  "scripts": {
    "build": "tsup src/**/*.ts --dts",
    "test": "vitest --run"
  },
  "devDependencies": {"tsup": "^7.2.0", "typescript": "^5.3.3", "vitest": "^1.0.0"}
}

packages/dsl/package.json
{
  "name": "@ontomind/dsl",
  "version": "0.1.0",
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "files": ["dist"],
  "scripts": {
    "build": "tsup src/**/*.ts --dts",
    "test": "vitest --run"
  },
  "dependencies": {"yaml": "^2.3.5"},
  "devDependencies": {"tsup": "^7.2.0", "typescript": "^5.3.3", "vitest": "^1.0.0"}
}

루트 package.json
{
  "scripts": {
    "oml:parse": "tsx tools/parse-oml.ts",
    "oml:merge": "tsx tools/merge-oml.ts",
    "build": "pnpm -r build",
    "test": "pnpm -r test"
  },
  "devDependencies": {"tsx": "^4.7.0"}
}
*/

// =============================================================
// 사용 방법 (요약)
// -------------------------------------------------------------
/*
1) data/schemas/ 아래 OML(TBox) 준비(완료)
2) pnpm i → pnpm -r build
3) pnpm oml:parse  →  data/generated/ir.json 생성 + 개수 출력
4) (선택) pnpm test → Vitest로 파서 의미론 테스트

이제 Step 2(codegen)에서 이 IR을 받아 JSON Schema/Prompt Summary를
생성하면, apps/chat에서 즉시 사용할 수 있습니다.
*/
```
