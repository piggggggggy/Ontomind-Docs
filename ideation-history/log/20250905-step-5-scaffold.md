// =============================================================
// OntoMind — Step 5: LLM Port (Adapters + Validation + Streaming)
// -------------------------------------------------------------
// 목표: Step 4의 프롬프트 번들(system/user/toolSchema)을 받아 실제 LLM을 호출하고,
//       (가능하면) 구조화 출력(response_format)으로 받으며, validator로 결과를 검증.
//       스트리밍/논스트리밍 모두 지원.
// =============================================================

/*
📁 추가 파일 트리

packages/
  port/
    src/
      index.ts
      llm.ts
      validate.ts
      streaming.ts
      providers/
        openai.ts
        openai-types.ts
    package.json

apps/
  chat/
    app/
      api/
        chat/route.ts     # Next.js(App Router) — 모델 호출 API

.env.local (예시)
  OPENAI_API_KEY=sk-...
  OPENAI_BASE_URL=https://api.openai.com/v1
  OPENAI_MODEL=gpt-4o-mini
*/

// =============================================================
// packages/port/src/providers/openai-types.ts — 최소 타입
// -------------------------------------------------------------
export interface OpenAIChatMessage { role: 'system'|'user'|'assistant'|'tool'; content: string }
export interface OpenAIChatRequest {
  model: string
  messages: OpenAIChatMessage[]
  temperature?: number
  response_format?: { type: 'json_schema', json_schema: any } | { type: 'text' }
  stream?: boolean
}

export interface OpenAIChatChunkChoiceDelta { role?: string; content?: string }
export interface OpenAIChatChunkChoice { delta?: OpenAIChatChunkChoiceDelta; finish_reason?: string }
export interface OpenAIStreamChunk { choices?: OpenAIChatChunkChoice[] }

// =============================================================
// packages/port/src/providers/openai.ts — OpenAI 어댑터
// -------------------------------------------------------------
import type { OpenAIChatRequest } from './openai-types'

export interface OpenAIConfig {
  apiKey: string
  baseURL?: string // 기본: https://api.openai.com/v1
  model: string
}

export async function openaiChat(req: OpenAIChatRequest, cfg: OpenAIConfig) {
  const base = cfg.baseURL || process.env.OPENAI_BASE_URL || 'https://api.openai.com/v1'
  const res = await fetch(`${base}/chat/completions`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${cfg.apiKey}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(req)
  })
  if (!res.ok) throw new Error(`OpenAI HTTP ${res.status}`)
  return res
}

// =============================================================
// packages/port/src/streaming.ts — SSE-like 텍스트 스트림 파서
// -------------------------------------------------------------
export async function* readServerSentEvents(res: Response) {
  const reader = res.body!.getReader()
  const decoder = new TextDecoder()
  let buf = ''
  while (true) {
    const { done, value } = await reader.read()
    if (done) break
    buf += decoder.decode(value, { stream: true })
    const chunks = buf.split('\n\n')
    buf = chunks.pop() || ''
    for (const block of chunks) {
      for (const line of block.split('\n')) {
        if (!line.startsWith('data:')) continue
        const data = line.slice(5).trim()
        if (data === '[DONE]') return
        yield data
      }
    }
  }
}

// =============================================================
// packages/port/src/validate.ts — 응답 검증 헬퍼
// -------------------------------------------------------------
import path from 'node:path'
import { validateIntentIO } from '@ontomind/validator/src/intent'

export function validateIntentOutput(intentKey: string, payload: any): { ok: boolean; errors?: string[] } {
  const intentsSchemaPath = path.join(process.cwd(), 'data', 'generated', 'intents.schema.json')
  return validateIntentIO(intentKey, payload, 'output', intentsSchemaPath)
}

// =============================================================
// packages/port/src/llm.ts — 프롬프트 번들 → 모델 호출
// -------------------------------------------------------------
import type { PromptBundle } from '@ontomind/prompt/src/types'
import { openaiChat, type OpenAIConfig } from './providers/openai'
import type { OpenAIChatRequest } from './providers/openai-types'
import { validateIntentOutput } from './validate'

export interface CallOptions {
  provider: 'openai'
  apiKey?: string
  baseURL?: string
  model?: string
  temperature?: number
  stream?: boolean
  intentKey: string
}

export async function callLLM(bundle: PromptBundle, opts: CallOptions) {
  const { system, user, toolSchema } = bundle

  if (opts.provider !== 'openai') throw new Error('Only openai provider is wired for now')
  const cfg: OpenAIConfig = {
    apiKey: opts.apiKey || process.env.OPENAI_API_KEY || '',
    baseURL: opts.baseURL || process.env.OPENAI_BASE_URL || undefined,
    model: opts.model || process.env.OPENAI_MODEL || 'gpt-4o-mini'
  }
  if (!cfg.apiKey) throw new Error('OPENAI_API_KEY missing')

  const req: OpenAIChatRequest = {
    model: cfg.model,
    messages: [
      { role: 'system', content: system },
      { role: 'user', content: user }
    ],
    temperature: opts.temperature ?? 0.2,
    response_format: toolSchema
      ? { type: 'json_schema', json_schema: { name: 'intent_output', schema: toolSchema, strict: true } }
      : { type: 'text' },
    stream: !!opts.stream
  }

  const res = await openaiChat(req, cfg)

  // 스트리밍이 아닌 경우: JSON 파싱 → 검증 후 반환
  if (!opts.stream) {
    const data = await res.json() as any
    const content = data?.choices?.[0]?.message?.content
    let parsed: any
    try { parsed = typeof content === 'string' ? JSON.parse(content) : content } catch { parsed = content }
    const check = validateIntentOutput(opts.intentKey, parsed)
    if (!check.ok) return { ok: false, error: 'schema_validation_failed', details: check.errors }
    return { ok: true, output: parsed, raw: data }
  }

  // 스트리밍인 경우: caller에게 Response 그대로 반환(Next API에서 중계)
  return { ok: true, stream: res.body }
}

// =============================================================
// packages/port/src/index.ts — Public API
// -------------------------------------------------------------
export { callLLM, type CallOptions } from './llm'
export { validateIntentOutput } from './validate'

// =============================================================
// packages/port/package.json
// -------------------------------------------------------------
{
  "name": "@ontomind/port",
  "version": "0.1.0",
  "type": "module",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "files": ["dist"],
  "scripts": {
    "build": "tsup src/**/*.ts --dts",
    "test": "vitest --run"
  },
  "dependencies": {},
  "devDependencies": {
    "tsup": "^7.2.0",
    "typescript": "^5.3.3",
    "vitest": "^1.0.0"
  }
}

// =============================================================
// apps/chat/app/api/chat/route.ts — Next.js API (모델 호출)
// -------------------------------------------------------------
import { NextRequest } from 'next/server'
import path from 'node:path'
import { assemblePrompt } from '@ontomind/prompt/src/assemble'
import { callLLM } from '@ontomind/port/src/llm'

export const runtime = 'nodejs'

export async function POST(req: NextRequest) {
  const body = await req.json()
  const intentKey: string = body.intentKey
  const targetIds: string[] = body.targetIds || []
  const contextIds: string[] = body.contextIds || []
  const stream: boolean = !!body.stream

  try {
    const bundle = assemblePrompt(
      { intentKey, targetIds, contextIds, maxEvidence: 8 },
      {
        promptSummaryPath: path.join(process.cwd(), 'data', 'generated', 'prompt-summary.json'),
        intentsSchemaPath: path.join(process.cwd(), 'data', 'generated', 'intents.schema.json'),
        instancesDir: path.join(process.cwd(), 'data', 'instances')
      }
    )

    const result = await callLLM(bundle, { provider: 'openai', stream, intentKey })

    if (stream && result.stream) {
      // 스트리밍: 원본 바디를 그대로 반환(프론트에서 ReadableStream 처리)
      return new Response(result.stream as any, {
        headers: { 'Content-Type': 'text/event-stream' }
      })
    }

    return new Response(JSON.stringify(result), { headers: { 'Content-Type': 'application/json' } })
  } catch (e: any) {
    return new Response(JSON.stringify({ ok: false, error: e?.message || String(e) }), { status: 400 })
  }
}

// =============================================================
// 루트 package.json 스크립트 추가
// -------------------------------------------------------------
/*
{
  "scripts": {
    "oml:chat": "curl -s -X POST localhost:3000/api/chat -H 'Content-Type: application/json' -d '{\n  \"intentKey\": \"intent:diagnose@1.0.0\",\n  \"targetIds\": [\"project:Project/sample-1\"]\n}' | jq ."
  }
}
*/

// =============================================================
// 사용 방법
// -------------------------------------------------------------
/*
1) Step 2/4 완료 상태에서 Next dev 서버 실행
2) POST /api/chat { intentKey, targetIds, contextIds?, stream? }
3) 논스트리밍이면 JSON, 스트리밍이면 text/event-stream으로 수신
4) 서버에서 응답을 파싱 후 validateIntentOutput으로 검증(논스트리밍 경로)

주의
- OpenAI 호환형 모델 중 response_format: json_schema를 지원하는 모델을 선택하세요.
- 모델이 JSON을 텍스트로 반환할 때를 대비하여 JSON.parse 시도 → 실패 시 원문 전달 + 검증 실패 리포트.
*/
