

// =============================================================
// Step 3 — Validator (ABox + Intent I/O)
// -------------------------------------------------------------
// 목표: (1) 인스턴스(Entity/Relation) JSONL 검증, (2) Intent 입·출력 검증,
// (3) 의미론 규칙(클래스/속성 존재, enum 일치, 카디널리티/역관계) 체크
// =============================================================

/*
📁 추가 파일 트리

packages/
  validator/
    src/
      index.ts
      ajv.ts
      entity.ts
      relation.ts
      intent.ts
      instances.ts
    package.json

tools/
  validate-instances.ts   # CLI: instances/*.jsonl 전체 검증
  validate-intent.ts      # CLI: 임의 Intent I/O 샘플 검증
*/

// =============================================================
// packages/validator/src/ajv.ts — Ajv 인스턴스/유틸
// -------------------------------------------------------------
import Ajv, { ErrorObject } from 'ajv';
import addFormats from 'ajv-formats';

export function createAjv() {
  const ajv = new Ajv({ allErrors: true, strict: false });
  addFormats(ajv);
  return ajv;
}

export function formatErrors(errors: ErrorObject[] = []) {
  return errors.map(e => `${e.instancePath || '/'} ${e.message}`).join('
');
}

// =============================================================
// packages/validator/src/entity.ts — 엔티티 검증
// -------------------------------------------------------------
import type { OMIR, OMClass, OMProperty } from '@ontomind/core/src/types';
import { createAjv, formatErrors } from './ajv';
import fs from 'node:fs';
import path from 'node:path';

export interface ValidateResult { ok: boolean; errors?: string[] }

function propertyIndex(c: OMClass) {
  const m = new Map<string, OMProperty>();
  for (const p of c.properties ?? []) m.set(p.id.slug, p);
  return m;
}

export function validateEntityRecord(ir: OMIR, entity: any): ValidateResult {
  const errors: string[] = [];
  // 필수 상단 키 체크
  for (const k of ['id','class','data']) if (!(k in entity)) errors.push(`missing key: ${k}`);
  if (errors.length) return { ok: false, errors };

  // 클래스 찾기(ns:slug[@ver] 혹은 ns:slug)
  const clsKey = entity.class.version ? `${entity.class.ns}:${entity.class.slug}@${entity.class.version}` : `${entity.class.ns}:${entity.class.slug}`;
  const classesByFull = new Map(ir.classes.map(c => [`${c.id.ns}:${c.id.slug}${c.id.version ? '@'+c.id.version : ''}`, c] as const));
  const classesByBase = new Map(ir.classes.map(c => [`${c.id.ns}:${c.id.slug}`, c] as const));
  const cls = classesByFull.get(clsKey) ?? classesByBase.get(`${entity.class.ns}:${entity.class.slug}`);
  if (!cls) return { ok: false, errors: [`class not found: ${clsKey}`] };

  const pIndex = propertyIndex(cls);
  const data = entity.data ?? {};

  // required 속성 체크
  for (const p of cls.properties ?? []) {
    if (p.required && !(p.id.slug in data)) errors.push(`required property missing: ${p.id.slug}`);
  }

  // 타입/enum 기초 검증 (라이트, Ajv는 generated schema에서 2차 검증)
  for (const [k, v] of Object.entries(data)) {
    const spec = pIndex.get(k);
    if (!spec) continue; // 추가 속성 허용
    if (typeof spec.type === 'string') {
      if (spec.type === 'number' && typeof v !== 'number') errors.push(`property ${k}: expected number`);
      if (spec.type === 'string' && typeof v !== 'string') errors.push(`property ${k}: expected string`);
      if (spec.type === 'boolean' && typeof v !== 'boolean') errors.push(`property ${k}: expected boolean`);
    } else if ('enum' in spec.type) {
      if (typeof v !== 'string' || !spec.type.enum.includes(v)) errors.push(`property ${k}: not in enum`);
    }
  }

  return errors.length ? { ok: false, errors } : { ok: true };
}

export function validateEntityWithSchema(entity: any, classesSchemaPath: string): ValidateResult {
  // 선택적: codegen 산출물(JSON Schema)을 사용한 2차 검증
  const ajv = createAjv();
  const bundle = JSON.parse(fs.readFileSync(classesSchemaPath, 'utf8')) as Record<string, any>;
  const classSlug = entity.class?.slug as string;
  const key = Object.keys(bundle).find(k => k.startsWith(`${entity.class.ns}:${classSlug}`));
  if (!key) return { ok: false, errors: [`schema for class not found: ${entity.class.ns}:${classSlug}`] };
  const schema = bundle[key];
  const validate = ajv.compile(schema);
  const ok = validate(entity.data ?? {});
  return ok ? { ok: true } : { ok: false, errors: [formatErrors(validate.errors || [])] };
}

// =============================================================
// packages/validator/src/relation.ts — 관계 검증
// -------------------------------------------------------------
import type { OMIR, OMRelationType } from '@ontomind/core/src/types';

export function validateRelationRecord(ir: OMIR, rel: any): ValidateResult {
  const errors: string[] = [];
  for (const k of ['type','from','to']) if (!(k in rel)) errors.push(`missing key: ${k}`);
  if (errors.length) return { ok: false, errors };

  const typeKey = rel.type.version ? `${rel.type.ns}:${rel.type.slug}@${rel.type.version}` : `${rel.type.ns}:${rel.type.slug}`;
  const relByFull = new Map(ir.relationTypes.map(r => [`${r.id.ns}:${r.id.slug}${r.id.version ? '@'+r.id.version : ''}`, r] as const));
  const relByBase = new Map(ir.relationTypes.map(r => [`${r.id.ns}:${r.id.slug}`, r] as const));
  const rSpec = relByFull.get(typeKey) ?? relByBase.get(`${rel.type.ns}:${rel.type.slug}`);
  if (!rSpec) return { ok: false, errors: [`relationType not found: ${typeKey}`] };

  // 카디널리티 체크는 전체 그래프 단위에서 수행(여기선 단건 구조만 체크)
  if (typeof rel.from !== 'string' || typeof rel.to !== 'string') errors.push('from/to must be id strings');

  return errors.length ? { ok: false, errors } : { ok: true };
}

// =============================================================
// packages/validator/src/intent.ts — Intent I/O 검증
// -------------------------------------------------------------
import fs from 'node:fs';
import path from 'node:path';
import { createAjv, formatErrors } from './ajv';

export function validateIntentIO(intentKey: string, payload: any, which: 'input'|'output', intentsSchemaPath: string): ValidateResult {
  const bundle = JSON.parse(fs.readFileSync(intentsSchemaPath, 'utf8')) as Record<string, any>;
  const schemaPair = bundle[intentKey];
  if (!schemaPair) return { ok: false, errors: [`intent schema not found: ${intentKey}`] };
  const schema = schemaPair[which] ?? schemaPair; // 구조에 따라 input/output 또는 단일
  const ajv = createAjv();
  const validate = ajv.compile(schema);
  const ok = validate(payload);
  return ok ? { ok: true } : { ok: false, errors: [formatErrors(validate.errors || [])] };
}

// =============================================================
// packages/validator/src/instances.ts — JSONL 스트림 검증
// -------------------------------------------------------------
import fs from 'node:fs';
import readline from 'node:readline';
import path from 'node:path';
import type { OMIR } from '@ontomind/core/src/types';
import { validateEntityRecord } from './entity';
import { validateRelationRecord } from './relation';

export interface FileReport { file: string; ok: boolean; count: number; errors: Array<{ line: number; message: string }>; }

export async function validateJsonlFile(ir: OMIR, filePath: string): Promise<FileReport> {
  const rl = readline.createInterface({ input: fs.createReadStream(filePath), crlfDelay: Infinity });
  let lineNo = 0; let okCount = 0; const errors: FileReport['errors'] = [];
  for await (const line of rl) {
    lineNo++;
    const t = line.trim();
    if (!t) continue;
    let obj: any;
    try { obj = JSON.parse(t); } catch (e) { errors.push({ line: lineNo, message: 'invalid JSON' }); continue; }
    const kind = obj.type ? 'relation' : 'entity'; // 단순 판별: relation은 {type, from, to}
    const res = kind === 'relation' ? validateRelationRecord(ir, obj) : validateEntityRecord(ir, obj);
    if (!res.ok) errors.push({ line: lineNo, message: (res.errors || []).join('; ') }); else okCount++;
  }
  return { file: filePath, ok: errors.length === 0, count: okCount, errors };
}

export async function validateInstancesDir(ir: OMIR, dir: string): Promise<FileReport[]> {
  const files = fs.readdirSync(dir).filter(f => f.endsWith('.jsonl'));
  const out: FileReport[] = [];
  for (const f of files) out.push(await validateJsonlFile(ir, path.join(dir, f)));
  return out;
}

// =============================================================
// packages/validator/src/index.ts — Public API
// -------------------------------------------------------------
export type { ValidateResult } from './entity';
export { validateEntityRecord, validateEntityWithSchema } from './entity';
export { validateRelationRecord } from './relation';
export { validateIntentIO } from './intent';
export { validateJsonlFile, validateInstancesDir, type FileReport } from './instances';

// =============================================================
// packages/validator/package.json
// -------------------------------------------------------------
{
  "name": "@ontomind/validator",
  "version": "0.1.0",
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "files": ["dist"],
  "scripts": {
    "build": "tsup src/**/*.ts --dts",
    "test": "vitest --run"
  },
  "dependencies": {
    "ajv": "^8.12.0",
    "ajv-formats": "^2.1.1"
  },
  "devDependencies": {
    "tsup": "^7.2.0",
    "typescript": "^5.3.3",
    "vitest": "^1.0.0"
  }
}

// =============================================================
// tools/validate-instances.ts — CLI: instances/*.jsonl 일괄 검증
// -------------------------------------------------------------
import path from 'node:path';
import fs from 'node:fs';
import { parseOML } from '@ontomind/dsl/src/parse';
import { validateInstancesDir } from '@ontomind/validator/src/instances';

const ir = parseOML();
const dir = path.join(process.cwd(), 'data', 'instances');
(async () => {
  const reports = await validateInstancesDir(ir, dir);
  for (const r of reports) {
    console.log(`
[${r.ok ? 'OK' : 'FAIL'}] ${r.file} — records: ${r.count}`);
    for (const e of r.errors) console.log(`  line ${e.line}: ${e.message}`);
  }
  const allOk = reports.every(r => r.ok);
  if (!allOk) process.exit(1);
})();

// =============================================================
// tools/validate-intent.ts — CLI: Intent I/O 샘플 검증
// -------------------------------------------------------------
import path from 'node:path';
import fs from 'node:fs';
import { parseOML } from '@ontomind/dsl/src/parse';
import { validateIntentIO } from '@ontomind/validator/src/intent';

const intentKey = process.argv[2]; // 예: intent:diagnose@1.0.0
const which = (process.argv[3] as 'input'|'output') || 'output';
const payloadPath = process.argv[4];
if (!intentKey || !payloadPath) {
  console.error('Usage: tsx tools/validate-intent.ts <intentKey> <input|output> <payload.json>');
  process.exit(1);
}

const intentsSchema = path.join(process.cwd(), 'data', 'generated', 'intents.schema.json');
const payload = JSON.parse(fs.readFileSync(payloadPath, 'utf8'));
const res = validateIntentIO(intentKey, payload, which, intentsSchema);
if (res.ok) console.log('OK'); else { console.error(res.errors?.join('
')); process.exit(1); }

// =============================================================
// 루트 package.json 스크립트 추가
// -------------------------------------------------------------
/*
{
  "scripts": {
    "oml:parse": "tsx tools/parse-oml.ts",
    "oml:codegen": "tsx tools/codegen.ts",
    "oml:validate:instances": "tsx tools/validate-instances.ts",
    "oml:validate:intent": "tsx tools/validate-intent.ts"
  }
}
*/

// =============================================================
// 사용 순서
// -------------------------------------------------------------
/*
1) pnpm oml:codegen   # Step 2 산출물 생성
2) pnpm oml:validate:instances
   - data/instances/*.jsonl 모두 라이트 의미론 + 형식 검증
3) pnpm oml:validate:intent "intent:diagnose@1.0.0" output tmp/diagnose.output.sample.json
   - LLM 응답 샘플을 intents.schema.json으로 검증

추가 아이디어(다음 단계):
- 카디널리티/역관계 글로벌 검사(그래프 스캔)
- enum 확장/상속 규칙 검사
- strict 모드(버전 충돌 시 명시 버전 강제)
*/
