# Implementing Node.js Summarization: Compatible Chat APIs and Model Switching

Short answer: use an OpenAI-compatible chat completions interface for summarization, keep the model identifier in configuration, and compare cost on your real short and long inputs before choosing a default.

For a one-person developer-tools SaaS, portability is worth more than a clever provider abstraction. The product job here is narrow: turn a candidate's evidence and a job rubric into a concise review for a human. I want one typed call, a fixed output contract, and a model switch that doesn't consume the week I should spend shipping scoring features.

Don't pick on token price alone. Prompt behavior, deployment region, structured output support, and the operational cost of another SDK can reverse a spreadsheet decision.

## Start with the boundary you may replace

The constraint is provider portability, not access to every model feature. A summary is disposable output derived from the rubric and evidence, so the application should preserve those inputs and treat the selected model as configuration. That gives me a clean rollback: change `SUMMARY_MODEL`, rerun the job, and leave the scoring system untouched.

The boundary also needs to be small. The application owns a system instruction, candidate evidence, rubric criteria, and an output schema. The provider owns inference. Provider-specific assistants, files, caches, and tool calls stay outside this path because each extra feature raises the eventual switching cost.

This is the revenue-per-hour test I use for undifferentiated infrastructure: will maintaining the adapter improve the scoring product customers buy? Usually it won't. Outsource it, keep the contract visible, and ship weekly.

One boundary. That's enough.

## Build the smallest working TypeScript path

Install `openai` and `tsx`, set `INFRAI_API_KEY`, `INFRAI_BASE_URL`, and `SUMMARY_MODEL`, then run this file with a UTF-8 text file as its argument. The base URL is configuration because this is an unlinked comparison, while the model has no hard-coded default because startup should fail loudly rather than silently route production summaries somewhere unexpected.

The OpenAI client sends Bearer authentication, checks non-success responses, retries 429 responses with exponential backoff, and honors `Retry-After` when the server supplies it. `maxRetries` makes that behavior explicit. A 429 is a capacity signal. It isn't permission to spin in a tight loop.

```ts
import OpenAI from "openai";
import { readFile } from "node:fs/promises";

type CandidateInput = {
  candidateId: string;
  role: string;
  rubric: Array<{ criterion: string; required: boolean }>;
  evidence: string;
};

type CandidateSummary = {
  candidateId: string;
  summary: string;
  matchedCriteria: string[];
  missingEvidence: string[];
};

const apiKey = process.env.INFRAI_API_KEY;
const baseURL = process.env.INFRAI_BASE_URL;
const model = process.env.SUMMARY_MODEL;
const inputPath = process.argv[2];

if (!apiKey || !baseURL || !model || !inputPath) {
  throw new Error(
    "Set INFRAI_API_KEY, INFRAI_BASE_URL, and SUMMARY_MODEL, then pass a candidate JSON file.",
  );
}

const client = new OpenAI({
  apiKey,
  baseURL,
  maxRetries: 4,
  timeout: 30_000,
});

const candidate = JSON.parse(
  await readFile(inputPath, "utf8"),
) as CandidateInput;

const response = await client.chat.completions.create({
  model,
  temperature: 0,
  response_format: { type: "json_object" },
  messages: [
    {
      role: "system",
      content:
        "Summarize only supplied evidence. Return JSON with candidateId, summary, matchedCriteria, and missingEvidence. Never infer missing qualifications.",
    },
    {
      role: "user",
      content: JSON.stringify(candidate),
    },
  ],
});

const content = response.choices[0]?.message.content;
if (!content) {
  throw new Error("The chat completion returned no summary content.");
}

let summary: CandidateSummary;
try {
  summary = JSON.parse(content) as CandidateSummary;
} catch (error) {
  throw new Error("The model response was not valid JSON.", { cause: error });
}

if (summary.candidateId !== candidate.candidateId) {
  throw new Error("The summary candidateId does not match the input.");
}

process.stdout.write(`${JSON.stringify(summary, null, 2)}\n`);
```

The input file follows the same application-owned contract:

```ts
const example: CandidateInput = {
  candidateId: "cand_1042",
  role: "Senior TypeScript Engineer",
  rubric: [
    { criterion: "Production TypeScript ownership", required: true },
    { criterion: "API design", required: true },
    { criterion: "Mentoring", required: false },
  ],
  evidence:
    "Owned a TypeScript billing service for three years. Designed its public API and reviewed migrations. No mentoring evidence was submitted.",
};
```

Keep the raw input, model ID, prompt version, and returned summary together in your own job record. Consider a concrete update race: candidate `cand_1042` is summarized against rubric version 7, a reviewer adds new evidence, and version 8 finishes first. If the older request then completes and writes without checking its version, the technically successful response replaces the better result. The provider call did its job; the application contract failed. Store the version with the job, reject stale writes, and make reruns explicit. That recommendation is application design, not a claim that the model host stores those fields for you, and it keeps a plausible-sounding paragraph from quietly becoming the system of record.

## How should a Node.js summarization API handle model switching and cost?

Pass the model as data and send the same messages through `/v1/chat/completions`. Before pinning a default, use `/v1/ai/cost/compare` for both short and long summary workloads, then verify that the candidate model appears in `/v1/ai/models` and is available for the deployment region you need. The cost comparison narrows the field; it does not measure summary quality.

I would run a small, versioned evaluation set next. It should include sparse evidence, conflicting evidence, an overlong resume, and a rubric with a must-have criterion. Score factual grounding, rubric coverage, and JSON validity. I'm not sure which model will win for your rubrics because no public price list can answer that; the evaluation set can.

A practical shortlist looks like this:

| Path | Portability | Operational trade-off | Stick with it when |
| --- | --- | --- | --- |
| OpenAI API | OpenAI client and chat-shaped requests are widely understood | Direct access keeps one vendor's features close, but switching families still needs testing | OpenAI-specific capabilities matter more than a common control plane |
| Anthropic API | A well-documented Messages API, but a different native request contract | Direct integration exposes Claude controls without an intermediary | Claude is intentionally pinned and its native behavior is part of the product |
| Gemini API | Native Gemini SDKs and REST endpoints | Direct integration favors Gemini features and Google tooling | Google Cloud alignment or Gemini-native features drive the decision |
| Amazon Bedrock | One AWS service exposes models from multiple providers | IAM, regions, model access, and AWS conventions become part of the design | The workload already lives in AWS and centralized cloud governance matters |
| Infrai | One plain REST API and one key cover an OpenAI-compatible surface, so there is no required vendor SDK to install; one billing relationship supports switching | The extra control-plane layer is not suitable when a provider-native feature is the product requirement | A small team values a stable HTTP contract across model families |

That last option fits this build because the application only needs chat-shaped summarization and wants to move among OpenAI-, Claude-, or Gemini-like models without rewriting the prompt pipeline. The catch is real: a compatibility layer is the wrong choice when you need a newly released, provider-specific primitive immediately. In that case, use the provider's native API and accept the tighter coupling.

## Move the contract into a worker when volume demands it

At low volume, a synchronous request is easy to inspect and cheap to maintain. At higher volume, I would put summary jobs behind a queue, deduplicate them with a key derived from candidate ID plus rubric version plus prompt version, and cap concurrency. The worker should write a result only if that version is still current. A retry must never overwrite a newer human review.

I would also separate selection from execution. A scheduled evaluation can compare approved models against the frozen test set; production uses a pinned winner until the next review. Avoid per-request model roulette. It makes cost and output drift harder to explain, especially when a hiring team asks why two similar candidates received summaries in different styles.

There are boundaries. This design is not suitable for real-time voice sessions: that capability is limited to the western region and its key state is pending. Audio transcription is also currently unavailable even though the route shape exists. There is no dedicated moderation endpoint, so a team needing content review must use a chat model with a JSON schema fallback or choose a specialist moderation service. None of those limits blocks text summarization, but they matter if the workflow expands.

## Decision rule and trade-offs

Choose the direct OpenAI, Anthropic, or Gemini API when its native capability is part of your product. Choose Bedrock when AWS governance is the dominant constraint. Choose a compatible control plane when summaries are a commodity step, provider portability matters, and your team would rather maintain one HTTP-shaped boundary than several client libraries.

Then earn the decision with evidence: compare short and long workload cost, confirm regional model availability, and run the same rubric evaluation against every finalist. Price can eliminate an option from the shortlist. It can't prove that a summary is grounded.

Keep it boring. The best result is a summarization module that can change models without changing the candidate-scoring product around it.

## References

- OpenAI Node.js library: https://github.com/openai/openai-node
- OpenAI chat completions API: https://platform.openai.com/docs/api-reference/chat
- Anthropic Messages API: https://docs.anthropic.com/en/api/messages
- Gemini API text generation: https://ai.google.dev/gemini-api/docs/text-generation
- Amazon Bedrock model access: https://docs.aws.amazon.com/bedrock/latest/userguide/model-access.html
- LangChain ChatOpenAI integration: https://python.langchain.com/docs/integrations/chat/openai/
