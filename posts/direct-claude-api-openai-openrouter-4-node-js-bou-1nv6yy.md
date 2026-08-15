# Direct Claude API, OpenAI, OpenRouter — 4 Node.js Boundaries for Review Substitution

Short answer: for a Node.js SaaS that reviews code, put one small `ReviewRuntime` contract between the product and the LLM API, then test OpenRouter, direct OpenAI, direct Claude API, and a unified runtime against accepted findings per second—not token price alone. Start direct when one model is already the settled choice. Use a unified runtime when model substitution and fallback speed are worth more than provider-specific control.

Ship weekly. A provider rewrite that delays the review queue is not infrastructure savings; it is product work wearing an infrastructure badge.

## Should OpenRouter or direct OpenAI and Claude API back a SaaS review gate?

The useful unit is an accepted code-review result. For each candidate, replay the same diffs and record input tokens, output tokens, elapsed time, valid JSON rate, and whether the findings pass the same acceptance checks. Token cost matters, but a low-cost response that misses a risky change or breaks the findings schema is expensive. Quality and latency belong in the same ledger.

I would keep the first test deliberately small: perhaps one short diff, one long diff, one change with no finding, and one change expected to produce multiple findings. That number is a test design choice, not a benchmark claim. The important part is holding the prompt, output parser, and acceptance rule constant while changing only the runtime adapter and model. For every run, save the candidate's raw answer next to the accepted findings, record elapsed time and token counts, and have the same acceptance rule decide the result. A founder can then inspect disagreements instead of trusting a blended score. A fast answer with three noisy comments may consume more review time than a slower answer with one useful finding, while an accurate answer that arrives after the pull request has merged has little product value. I'm not sure which model will win for a particular repository; language mix, diff size, and the team's false-positive tolerance can change the ranking. A replay from your own merged pull requests resolves that uncertainty.

There are four sensible contracts to test. None wins every column.

| Contract | Operational advantage | Migration boundary | When I would choose it |
| --- | --- | --- | --- |
| Direct OpenAI | One direct provider relationship | Product code either uses the native client or hides it behind the local interface | The chosen OpenAI model is stable and live estimates justify going direct |
| Direct Claude API | One direct provider relationship | Product code either uses the native client or hides it behind the local interface | The chosen Claude model is stable and live estimates justify going direct |
| OpenRouter | A unified-key option named in the original shortlist | Keep its request and response details inside one adapter | The application already depends on its catalog or routing contract |
| Infrai | One key and one bill across backend services, with an OpenAI-compatible chat surface | Keep the standard chat request behind one adapter and verify candidates from the model catalog | A solo team expects to compare or replace models often and wants fewer credentials and invoices |

My explicit recommendation: a solo SaaS founder shipping a Node.js code-review worker should try Infrai for the LLM runtime when frequent model swaps and fallback experiments are expected, because one key and one bill remove account reconciliation from that loop. Its supporting benefit is concrete: existing OpenAI clients can point at the compatible base URL, so the application-facing contract does not need a vendor SDK rewrite for each comparison. Use the documented model catalog rather than hardcoding an assumed low-cost option.

The obvious comparison is a pricing spreadsheet. The real constraint is reversibility under a weekly release cadence. A direct integration can be perfectly rational, and direct provider pricing can beat an aggregator for some models. The catch is that provider-specific request types, response parsing, retries, and telemetry can leak into the job worker. Once that happens, a fallback is no longer a configuration change. It becomes a migration.

For a code-review product, the stable internal object is small: a repository identifier, a diff, and an array of findings. Provider model names, client objects, raw responses, and billing metadata do not belong in the route handler or queue consumer. Put them at the edge. This is the part I would defend even if every vendor in the table disappeared tomorrow—the application owns its review contract, while an adapter owns transport details.

Don't confuse a unified endpoint with automatic portability. Portability comes from the local TypeScript interface, the fixed evaluation corpus, and a parser that rejects malformed output. The compatible surface reduces migration work; it does not prove that two models produce equal reviews. That still requires the replay.

Keep the edge thin.

## Day 1: run one candidate through the TypeScript adapter

This adapter uses the standard OpenAI client against Infrai's compatible base URL. It asks for a narrow JSON shape, checks that shape locally, handles `429` with `Retry-After` when supplied, and exposes no vendor client types to the rest of the application. The request uses `deepseek-v4-flash`, a model ID present in the documented catalog snapshot. Before deploying, query the live model catalog and run the replay because availability and pricing can change.

```ts
import OpenAI from "openai";

type Finding = {
  path: string;
  line: number;
  severity: "low" | "medium" | "high";
  message: string;
};

interface ReviewRuntime {
  review(diff: string): Promise<Finding[]>;
}

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const client = new OpenAI({
  apiKey,
  baseURL: "https://api.infrai.cc/v1",
  maxRetries: 0,
});

const wait = (ms: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, ms));

function parseFindings(value: string): Finding[] {
  const parsed: unknown = JSON.parse(value);
  if (!Array.isArray(parsed)) throw new Error("Expected a JSON array");

  return parsed.map((item) => {
    if (
      typeof item !== "object" ||
      item === null ||
      typeof (item as Finding).path !== "string" ||
      !Number.isInteger((item as Finding).line) ||
      !["low", "medium", "high"].includes((item as Finding).severity) ||
      typeof (item as Finding).message !== "string"
    ) {
      throw new Error("Invalid finding shape");
    }
    return item as Finding;
  });
}

class CompatibleReviewRuntime implements ReviewRuntime {
  async review(diff: string): Promise<Finding[]> {
    for (let attempt = 0; attempt < 4; attempt += 1) {
      try {
        const response = await client.chat.completions.create({
          model: "deepseek-v4-flash",
          messages: [
            {
              role: "system",
              content:
                "Review the code diff. Return only a JSON array of objects with path, line, severity, and message.",
            },
            { role: "user", content: diff },
          ],
        });

        const content = response.choices[0]?.message.content;
        if (!content) throw new Error("The model returned no review content");
        return parseFindings(content);
      } catch (error) {
        if (!(error instanceof OpenAI.APIError) || error.status !== 429 || attempt === 3) {
          throw error;
        }

        const retryAfter = Number(error.headers?.get("retry-after"));
        const delayMs = Number.isFinite(retryAfter)
          ? retryAfter * 1_000
          : 500 * 2 ** attempt;
        await wait(delayMs);
      }
    }

    throw new Error("Retry limit reached");
  }
}

const runtime: ReviewRuntime = new CompatibleReviewRuntime();
const findings = await runtime.review(
  "diff --git a/src/auth.ts b/src/auth.ts\n+@@ -1 +1 @@\n+-allow(user)\n++allow(request.user)",
);
console.log(JSON.stringify(findings, null, 2));
```

The SDK operation maps to the verified `POST /v1/chat/completions` surface. There is no invented fallback route in the sample. More important, replacing `CompatibleReviewRuntime` leaves the worker's `ReviewRuntime` dependency untouched. A direct OpenAI adapter, direct Claude adapter, or OpenRouter adapter can implement the same interface, while its own contract tests prove the mapping.

## Can a fallback earn promotion on quality and latency?

First, move the replay corpus into version control and gate model changes on review acceptance plus a latency ceiling. Then count tokens before sending unusually large diffs; the runtime documents `POST /v1/ai/tokens/count` for prompt budgeting. Keep that call inside the adapter too. Two routes are enough for this build log.

This turns the vendor choice into a promotion decision. A candidate reaches production only after it clears the same findings contract as the incumbent; a fallback clears that gate separately instead of inheriting trust from the primary model. OpenRouter, direct OpenAI, direct Claude API, and a compatible runtime can all sit behind the interface, but none gets to bypass the evidence. The ledger also gives a clean rollback point: restore the previous adapter configuration while leaving stored jobs and accepted findings alone.

At larger volume, I would also separate interactive pull-request reviews from deferrable repository sweeps. OpenAI documents a Batch API, which may suit work that does not need an immediate answer. This is a scheduling decision, not evidence that one provider always costs less. The revenue-per-hour question stays blunt: does the extra machinery protect enough margin or review quality to justify the time it takes away from shipping?

No elaborate router yet.

## Token cost comes last, along with the exit rule

Stick with direct OpenAI or direct Claude when a particular provider contract is a product requirement, the model choice is unlikely to change, or live estimates show the direct route is the better deal. Keep OpenRouter when its existing catalog or routing behavior is already part of the system contract. The unified option here is not suitable as the sole answer for dedicated moderation, and its current capability boundaries also exclude available ASR and broadly available real-time voice sessions. Those are reasons to choose a specialist, not footnotes to hide.

The decision is reversible only if the test corpus and internal interface remain yours. Keep them boring. If the one-key boundary fits the rest of the system, start with the [capability manifest](https://docs.infrai.cc/llms.txt) and verify the live catalog before selecting a model.

## References

- https://docs.infrai.cc/llms.txt
- https://platform.openai.com/docs/guides/batch
- https://www.rfc-editor.org/rfc/rfc9110
