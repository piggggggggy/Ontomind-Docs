
// =============================================================
// Step 4 — Prompt Assembler (system/user/toolSchema + evidence)
// -------------------------------------------------------------
// 목표: IR + generated(특히 prompt-summary.json) + (선택) instances 를 바탕으로
// LLM 호출에 바로 쓸 수 있는 프롬프트 번들(system/user/toolSchema)을 생성.
// =============================================================

/*
📁 추가 파일 트리

packages/
  prompt/
    src/
      index.ts
      assemble.ts
      filters.ts
      templates.ts
      types.ts
    package.json

apps/
  chat/
    app/
      api/
        assemble/route.ts   # Next.js(App Router) API 라우트 예시

tools/
  assemble.ts              # CLI: 프롬프트 조립 미리보기
*/

// =============================================================
// packages/prompt/src/types.ts — 타입
// -------------------------------------------------------------
import type { OMIR } from '@ontomind/core/src/types';

export interface AssembleOptions {
  intentKey: string;           // 예: "intent:diagnose@1.0.0"
  targetIds?: string[];        // 예: ["project:Project/ontomind-core"]
  contextIds?: string[];       // 예: ["core:Context/workspace-admin"]
  maxEvidence?: number;        // 기본 10
}

export interface PromptBundle {
  system: string;
  user: string;
  toolSchema?: Record<string, any>; // 선택: Intent Output schema
  evidence?: any[];                 // 선택: 첨부 데이터 (ABox 서브셋)
}

export interface DataSources {
  promptSummaryPath: string;         // data/generated/prompt-summary.json
  intentsSchemaPath: string;         // data/generated/intents.schema.json
  instancesDir?: string;             // data/instances (선택)
}

export interface PromptSummary {
  version: number;
  namespaces: string[];
  classes: Array<{ id: string; label: string; props: Array<{ key: string; type: string }> }>;
  relations: Array<{ id: string; label: string; domain: string; range: string; card?: string }>;
  intents: Array<{ id: string; label: string; input: Record<string,string>; output: Record<string,string> }>;
  policies: Array<{ id: string; label: string }>;
  contexts: Array<{ id: string; label: string; role?: string }>;
}

// =============================================================
// packages/prompt/src/filters.ts — 컨텍스트/타겟 필터 & 증거 수집(라이트)
// -------------------------------------------------------------
import fs from 'node:fs';
import path from 'node:path';

export function loadPromptSummary(p: string) {
  return JSON.parse(fs.readFileSync(p, 'utf8')) as any;
}

export function loadIntentsSchema(p: string) {
  return JSON.parse(fs.readFileSync(p, 'utf8')) as Record<string, any>;
}

// 아주 단순한 JSONL 로더 (빈 파일/없음 허용)
export function readJsonl(file: string): any[] {
  if (!fs.existsSync(file)) return [];
  const lines = fs.readFileSync(file, 'utf8').split(/
?
/).map(s => s.trim()).filter(Boolean);
  const out: any[] = [];
  for (const l of lines) { try { out.push(JSON.parse(l)); } catch { /* skip */ } }
  return out;
}

export function collectInstances(instancesDir?: string) {
  if (!instancesDir || !fs.existsSync(instancesDir)) return { entities: [], relations: [] };
  const files = fs.readdirSync(instancesDir).filter(f => f.endsWith('.jsonl'));
  const entities: any[] = []; const relations: any[] = [];
  for (const f of files) {
    const arr = readJsonl(path.join(instancesDir, f));
    for (const obj of arr) {
      if (obj && typeof obj === 'object') {
        if ('type' in obj && 'from' in obj && 'to' in obj) relations.push(obj);
        else if ('class' in obj && 'data' in obj) entities.push(obj);
      }
    }
  }
  return { entities, relations };
}

// 타겟 ID로 주어진 엔티티 + 1-hop 이웃을 간단히 수집
export function pickEvidence(targetIds: string[] = [], instances: { entities: any[]; relations: any[] }, max = 10) {
  if (!instances.entities.length) return [];
  const set = new Set<string>(targetIds);
  const idToEntity = new Map(instances.entities.map((e: any) => [e.id, e] as const));
  const out: any[] = [];
  for (const id of targetIds) {
    const e = idToEntity.get(id);
    if (e) out.push(e);
  }
  // 1-hop 관계 따라 확장
  for (const rel of instances.relations) {
    if (out.length >= max) break;
    if (set.has(rel.from) && idToEntity.has(rel.to)) out.push(idToEntity.get(rel.to)!);
    else if (set.has(rel.to) && idToEntity.has(rel.from)) out.push(idToEntity.get(rel.from)!);
  }
  return out.slice(0, max);
}

// =============================================================
// packages/prompt/src/templates.ts — 프롬프트 템플릿
// -------------------------------------------------------------
import type { PromptSummary } from './types';

export function renderSystem(summary: PromptSummary, intentLabel: string, ctxLabels: string[]) {
  const core = `You are OntoMind assistant. Follow ontology strictly.
` +
    `- Use the provided schema vocabulary.
` +
    `- Return JSON exactly matching the tool schema when asked.
` +
    `- Cite entity ids when relevant.
`;
  const classes = summary.classes.map(c => `• ${c.label} (${c.id}) → { ${c.props.map(p => p.key+':'+p.type).join(', ')} }`).join('
');
  const rels = summary.relations.map(r => `• ${r.label} : ${r.domain} → ${r.range} [${r.card || ''}]`).join('
');
  const ctx = ctxLabels.length ? `Active contexts: ${ctxLabels.join(', ')}` : '';
  return [core, `# Classes
${classes}`, `# Relations
${rels}`, ctx, `# Active Intent
${intentLabel}`].filter(Boolean).join('

');
}

export function renderUser(intentLabel: string, targetIds: string[], evidence: any[]) {
  const head = `Intent: ${intentLabel}`;
  const targets = targetIds.length ? `Targets: ${targetIds.join(', ')}` : '';
  const ev = evidence.length ? `Evidence:
${evidence.map(e => `- ${e.id} ${JSON.stringify(e.data)}`).join('
')}` : 'Evidence: (none)';
  return [head, targets, ev].filter(Boolean).join('

');
}

// =============================================================
// packages/prompt/src/assemble.ts — 핵심 조립 로직
// -------------------------------------------------------------
import path from 'node:path';
import { loadIntentsSchema, loadPromptSummary, collectInstances, pickEvidence } from './filters';
import { renderSystem, renderUser } from './templates';
import type { AssembleOptions, DataSources, PromptBundle, PromptSummary } from './types';

export function assemblePrompt(opts: AssembleOptions, src: DataSources): PromptBundle {
  const summary = loadPromptSummary(src.promptSummaryPath) as PromptSummary;
  const intentsSchema = loadIntentsSchema(src.intentsSchemaPath);

  // intent 선택
  const intent = summary.intents.find(i => i.id === opts.intentKey);
  if (!intent) throw new Error(`intent not found: ${opts.intentKey}`);

  // 컨텍스트 라벨 모으기(선택)
  const ctxLabels = (opts.contextIds || [])
    .map(cid => summary.contexts.find(c => c.id === cid)?.label)
    .filter(Boolean) as string[];

  // 증거 수집 (선택: ABox 없으면 비어있음)
  const instances = collectInstances(src.instancesDir);
  const evidence = pickEvidence(opts.targetIds || [], instances, opts.maxEvidence ?? 10);

  // system/user
  const system = renderSystem(summary, intent.label, ctxLabels);
  const user = renderUser(intent.label, opts.targetIds || [], evidence);

  // toolSchema: intent output schema를 제공(있으면 검사 가능)
  const toolSchema = intentsSchema[opts.intentKey]?.output || intentsSchema[opts.intentKey];

  return { system, user, toolSchema, evidence };
}

// =============================================================
// packages/prompt/src/index.ts — Public API
// -------------------------------------------------------------
export { assemblePrompt } from './assemble';
export type { AssembleOptions, DataSources, PromptBundle, PromptSummary } from './types';

// =============================================================
// packages/prompt/package.json
// -------------------------------------------------------------
{
  "name": "@ontomind/prompt",
  "version": "0.1.0",
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "files": ["dist"],
  "scripts": {
    "build": "tsup src/**/*.ts --dts",
    "test": "vitest --run"
  },
  "devDependencies": {
    "tsup": "^7.2.0",
    "typescript": "^5.3.3",
    "vitest": "^1.0.0"
  }
}

// =============================================================
// tools/assemble.ts — CLI: 프롬프트 조립 미리보기
// -------------------------------------------------------------
import path from 'node:path';
import fs from 'node:fs';
import { assemblePrompt } from '@ontomind/prompt/src/assemble';

const intentKey = process.argv[2] || 'intent:diagnose@1.0.0';
const targets = (process.argv[3]?.split(',') || []).filter(Boolean);

const bundle = assemblePrompt(
  { intentKey, targetIds: targets, maxEvidence: 8 },
  {
    promptSummaryPath: path.join(process.cwd(), 'data', 'generated', 'prompt-summary.json'),
    intentsSchemaPath: path.join(process.cwd(), 'data', 'generated', 'intents.schema.json'),
    instancesDir: path.join(process.cwd(), 'data', 'instances')
  }
);

fs.mkdirSync(path.join(process.cwd(), 'tmp'), { recursive: true });
fs.writeFileSync(path.join(process.cwd(), 'tmp', 'prompt.system.txt'), bundle.system, 'utf8');
fs.writeFileSync(path.join(process.cwd(), 'tmp', 'prompt.user.txt'), bundle.user, 'utf8');
fs.writeFileSync(path.join(process.cwd(), 'tmp', 'tool.schema.json'), JSON.stringify(bundle.toolSchema ?? {}, null, 2));
console.log('[OntoMind] assemble done. See tmp/prompt.* & tmp/tool.schema.json');

// =============================================================
// apps/chat/app/api/assemble/route.ts — Next.js API 예시
// -------------------------------------------------------------
import { NextRequest, NextResponse } from 'next/server';
import path from 'node:path';
import { assemblePrompt } from '@ontomind/prompt/src/assemble';

export async function POST(req: NextRequest) {
  const body = await req.json();
  const intentKey = body.intentKey as string;
  const targetIds = (body.targetIds as string[]) || [];
  const contextIds = (body.contextIds as string[]) || [];

  try {
    const bundle = assemblePrompt(
      { intentKey, targetIds, contextIds, maxEvidence: 8 },
      {
        promptSummaryPath: path.join(process.cwd(), 'data', 'generated', 'prompt-summary.json'),
        intentsSchemaPath: path.join(process.cwd(), 'data', 'generated', 'intents.schema.json'),
        instancesDir: path.join(process.cwd(), 'data', 'instances')
      }
    );
    return NextResponse.json(bundle);
  } catch (e: any) {
    return NextResponse.json({ error: e?.message || String(e) }, { status: 400 });
  }
}

// =============================================================
// 루트 package.json 스크립트 추가
// -------------------------------------------------------------
/*
{
  "scripts": {
    "oml:assemble": "tsx tools/assemble.ts intent:diagnose@1.0.0 project:Project/example-1"
  }
}
*/

// =============================================================
// 사용 방법
// -------------------------------------------------------------
/*
1) pnpm oml:codegen        # Step 2 산출물 생성 (필수)
2) (선택) data/instances/*.jsonl 에 간단한 엔티티 몇 줄 추가
3) pnpm oml:assemble "intent:diagnose@1.0.0" "project:Project/sample-1"
   - tmp/prompt.system.txt, tmp/prompt.user.txt, tmp/tool.schema.json 생성
4) apps/chat 프론트에서 /api/assemble 호출 → 반환된 system/user/toolSchema로 LLM 호출
*/
