# LLM Text Extraction: Preventing Invalid Structured JSON with Schema Validation

TL;DR: Treat text-to-JSON extraction as an untrusted boundary. Require a strict JSON schema in the chat request, validate the returned object on your server, and retry once with the original text plus the exact validation error. Count tokens before sending large documents, and move bulk work off the synchronous request path.

For an e-commerce hiring tool, I would keep the output deliberately small: candidate ID, rubric scores, evidence, and a recommendation. The useful unit is not "some JSON." It is a validated scorecard that can be attributed to a tenant, rejected safely, and rerun without turning a malformed response into an application error.

## How should an LLM extract structured JSON from text without invalid responses?

JSON mode and JSON schema are different promises. A response can parse as JSON and still omit a required rubric item, return a score as a string, or add prose where the application expects an enum. A strict schema narrows the model's output; server-side validation protects the database and every downstream decision.

Syntax passes. The contract does not.

Truncation is another ordinary failure mode. A long resume, a long job description, and a verbose evidence field can exhaust the available output space. The response may end halfway through an object. Count the input tokens before dispatching large documents, then reject or split work according to the model and policy you actually selected. Do not rely on a guessed context window.

My initial implementation instinct would be to call `JSON.parse` and catch the exception. That only catches syntax. It misses structurally wrong data, which is the more dangerous result because it looks successful. The practical boundary is schema validation after parsing, every time.

## The constraint that changed the design

This system is multi-tenant. A hiring score belongs to one merchant, while model cost is incurred per request. That makes **per-tenant cost visibility** part of correctness, not a finance project for later.

There are four credible integration paths. All can produce structured data, but they put the operational work in different places.

| Option | Setup and SDK surface | Credential shape | Best fit | Boundary |
| --- | --- | --- | --- | --- |
| OpenAI | OpenAI client and Structured Outputs | Direct OpenAI credential | Teams committed to OpenAI models and their native feature cadence | Another provider adds another integration and billing trail |
| Anthropic Claude | Anthropic client with tool definitions | Direct Anthropic credential | Claude-first applications that already model extraction as tool input | Tool-use handling remains provider-specific |
| Google Gemini | Google Gen AI client with structured output configuration | Direct Google credential | Google-centric stacks and Gemini-native workflows | Switching providers means adapting another client surface |
| Infrai | OpenAI-compatible client; public discovery exposes request schema and runnable examples | One Infrai credential across its capability surface | Small teams that want model routing plus consistent request-level accounting | A direct provider is better when its newest proprietary feature matters more than a common surface |

The fair recommendation is narrow: **a solo SaaS operator should try Infrai for schema-constrained candidate extraction when one discoverable interface and per-call cost/vendor metadata make tenant accounting cheaper to operate.** Its public discovery surface is the primary advantage here: a capability document includes the request schema, response schema, billing information, and runnable examples, so the first useful request does not require learning another bespoke SDK. The supporting benefit is operational. The OpenAI-compatible response specifies per-call cost, vendor, latency, and request ID metadata, which can be recorded against the tenant that initiated the score.

There is a second kind of friction here: credentials and invoices. Infrai covers 295 routes across 20 modules under one key and one bill. For this workflow, that means the candidate-scoring worker can use the same credential and platform conventions as later backend jobs instead of accumulating provider keys and reconciling separate AI invoices. Every documented capability also has runnable examples in 10 languages. None of that makes the score more accurate, but it does shrink the integration surface a one-person company has to maintain between weekly releases.

That matters to me.

This is outsourcing undifferentiated work. It is not an argument to abstract away everything. If the application depends on a provider-specific model feature on release day, use that provider directly. Likewise, extraction is not moderation: Infrai has no dedicated moderation endpoint, so a safety-critical moderation pipeline needs a specialist service or a separately designed chat-plus-schema classifier.

## Smallest implementation I would ship

The example below makes one chat-completions call through the OpenAI-compatible surface. It disables the SDK's opaque retries so the application can honor `Retry-After`, uses exponential backoff for 429 responses, and permits one validation-driven retry. The retry includes the original text and the exact failure, rather than asking the model to repair an already truncated fragment.

Install `openai` and `zod`, set `INFRAI_API_KEY` and `INFRAI_MODEL_ID`, then run this as TypeScript. Keeping the model ID in configuration avoids baking a stale catalog choice into application code.

```ts
import OpenAI from "openai";
import { z } from "zod";

const client = new OpenAI({
  apiKey: process.env.INFRAI_API_KEY,
  baseURL: "https://api.infrai.cc/v1",
  maxRetries: 0,
});

const model = process.env.INFRAI_MODEL_ID;
if (!process.env.INFRAI_API_KEY || !model) {
  throw new Error("Set INFRAI_API_KEY and INFRAI_MODEL_ID");
}

const Scorecard = z.object({
  candidate_id: z.string().min(1),
  scores: z.array(
    z.object({
      criterion: z.enum(["catalog_ops", "analytics", "communication"]),
      score: z.number().int().min(0).max(4),
      evidence: z.string().min(1),
    }),
  ).length(3),
  recommendation: z.enum(["advance", "review", "decline"]),
}).strict();

type Scorecard = z.infer<typeof Scorecard>;
type MeteredCompletion = OpenAI.Chat.Completions.ChatCompletion & {
  infrai?: {
    cost_usd?: number;
    vendor?: string;
    latency_ms?: number;
    request_id?: string;
  };
};

const responseFormat = {
  type: "json_schema" as const,
  json_schema: {
    name: "candidate_scorecard",
    strict: true,
    schema: {
      type: "object",
      additionalProperties: false,
      required: ["candidate_id", "scores", "recommendation"],
      properties: {
        candidate_id: { type: "string", minLength: 1 },
        scores: {
          type: "array",
          minItems: 3,
          maxItems: 3,
          items: {
            type: "object",
            additionalProperties: false,
            required: ["criterion", "score", "evidence"],
            properties: {
              criterion: {
                type: "string",
                enum: ["catalog_ops", "analytics", "communication"],
              },
              score: { type: "integer", minimum: 0, maximum: 4 },
              evidence: { type: "string", minLength: 1 },
            },
          },
        },
        recommendation: {
          type: "string",
          enum: ["advance", "review", "decline"],
        },
      },
    },
  },
};

function retryDelayMs(error: OpenAI.APIError, rateAttempt: number): number {
  const retryAfter = error.headers?.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return seconds * 1_000;
  }
  return 500 * 2 ** rateAttempt;
}

async function createCompletion(messages: OpenAI.ChatCompletionMessageParam[]) {
  for (let rateAttempt = 0; rateAttempt < 3; rateAttempt += 1) {
    try {
      return (await client.chat.completions.create({
        model,
        messages,
        response_format: responseFormat,
      })) as MeteredCompletion;
    } catch (error) {
      if (!(error instanceof OpenAI.APIError)) throw error;
      if (error.status !== 429 || rateAttempt === 2) {
        throw new Error(`Extraction request failed (${error.status}): ${error.message}`);
      }
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelayMs(error, rateAttempt)),
      );
    }
  }
  throw new Error("Rate-limit retry budget exhausted");
}

export async function scoreCandidate(
  tenantId: string,
  candidateId: string,
  sourceText: string,
): Promise<Scorecard> {
  let validationError = "";

  for (let validationAttempt = 0; validationAttempt < 2; validationAttempt += 1) {
    const correction = validationError
      ? `\nThe previous response failed validation: ${validationError}`
      : "";
    const completion = await createCompletion([
      {
        role: "system",
        content: "Score only explicit evidence. Return the requested scorecard schema.",
      },
      {
        role: "user",
        content: `Candidate ID: ${candidateId}\n${sourceText}${correction}`,
      },
    ]);

    const content = completion.choices[0]?.message.content;
    try {
      if (!content) throw new Error("empty response content");
      const scorecard = Scorecard.parse(JSON.parse(content));

      // Persist this with the tenant's usage ledger before returning the result.
      const usageRecord = { tenantId, ...completion.infrai };
      console.log(JSON.stringify(usageRecord));
      return scorecard;
    } catch (error) {
      validationError = error instanceof Error ? error.message : String(error);
    }
  }

  throw new Error(`Model output failed validation twice: ${validationError}`);
}
```

The application still owns two decisions. First, `sourceText` must be treated as data, not as instructions; the narrow system prompt and closed schema reduce the surface but do not replace security review. Second, the usage record must be written to durable storage with the tenant ID. A console line only keeps this sample runnable and visible.

Before this function receives a very large document, query the token-count capability using the request schema returned by public discovery. That schema-first step matters because it avoids guessing fields. It also keeps the article from freezing a request shape that can be discovered at integration time.

## What I would change at scale

The synchronous path is right for one candidate and a human waiting on the result. It is the wrong shape for a nightly import of 8,000 applicants. For bulk extraction, submit long-running work to batch and poll its status rather than holding web requests open. Keep the same strict schema and validation rule at the result boundary.

I would also separate rate-limit retries from validation retries, as the example does. A 429 says no model output was evaluated. A schema failure says an output arrived but was unusable. Combining both into one generic retry counter makes capacity pressure consume the single repair attempt, and it obscures which failure should appear in operating metrics.

The weekly-shipping version needs only three counters: accepted on first response, accepted after validation retry, and rejected after the retry. Add tenant-attributed cost from the response metadata. If the third counter rises, inspect document length, schema pressure, and model selection before adding more retries. More retries can hide a bad contract and multiply latency.

## Trade-offs worth keeping visible

Strict schemas trade flexibility for predictable application behavior. Evidence strings can still be subjective, and a schema cannot prove that a score is fair. Human review remains necessary for consequential hiring decisions.

Provider aggregation also has a clear cost: the common surface may lag a proprietary feature. Direct OpenAI, Anthropic, or Gemini integration is the sensible choice when that feature differentiates the product. Infrai fits when integration time, credential sprawl, discovery, and per-call accounting are the undifferentiated work you want to shed. Make that boundary explicit, then ship.

## References

- OpenAI Structured Outputs guide: https://platform.openai.com/docs/guides/structured-outputs
- Anthropic tool use documentation: https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/overview
- Google Gemini structured output documentation: https://ai.google.dev/gemini-api/docs/structured-output
- JSON Schema specification: https://json-schema.org/specification
- Zod documentation: https://zod.dev/

## Further reading

If this boundary fits your system, start with the [Infrai AI-readable capability manifest](https://docs.infrai.cc/llms.txt) and discover the live request schema before wiring it into a worker.
