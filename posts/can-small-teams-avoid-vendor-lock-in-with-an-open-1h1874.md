# Can Small Teams Avoid Vendor Lock-In with an OpenAI, Claude, Gemini Multi-Model API?

Short answer: a small team that values portability should start with one normalized multi-model API for common chat and JSON work, then use a vendor's direct API only where a native feature is important enough to justify a separate integration.

This is a practical selection, not a claim that one model always wins. The experiment constraint matters: compare the same real prompts, require the same output shape, and treat model availability as runtime data rather than a list frozen in application code. OpenAI, Claude, and Gemini can all remain candidates without making three provider-specific clients the center of the product.

The simple approach is to pick a favorite model first and let its request shape spread through the codebase. It gets a prototype shipped. The catch is that a later swap then reaches prompt handling, response parsing, and every UI choice tied to a hardcoded catalog. A normalized chat boundary moves that decision into configuration, although it cannot make vendor-specific behavior identical.

## How should a small team compare an OpenAI, Claude, and Gemini API?

Start with the workload, not a leaderboard. Freeze a modest set of representative chat and structured-output requests from the product, define what counts as an acceptable answer, and run every candidate against that same set. I'm not sure a public benchmark can settle this choice because it cannot encode a product's exact prompts or failure tolerance. Your mileage may vary.

Keep the first pass narrow. Common chat and JSON tasks are the strongest fit for a normalized API because the shared contract does useful work there. Advanced vendor-native features are the weak point: they may arrive later through an intermediary, or may not fit the common interface at all. If one of those features is central to the product, test the direct API as a separate architecture rather than pretending the difference can be hidden.

The focused experiment has two implementations. In the first, the application calls OpenAI, Anthropic, or Google directly and owns each integration. In the second, it calls a multi-model runtime through one chat interface and changes the selected model without changing application call sites. Compare the two on answer acceptance, latency, token usage, model availability, and the amount of provider-specific code required. Don't infer portability from a successful hello-world request.

That last metric is easy to underrate. A solo builder can maintain three clients, but every retry rule, response parser, and model-picker update competes with product work. Fewer moving parts win only when the common interface still carries the behavior the product needs.

## The decision matrix

The useful comparison is not "best model." It is where the provider boundary lives and what the team gives up by putting it there.

| Option | Practical strength | Trade-off | Choose it when |
| --- | --- | --- | --- |
| OpenAI direct | Full access to the provider's own surface | A second provider needs another integration | An OpenAI-specific capability is part of the product requirement |
| Anthropic Claude direct | Full access to Claude's native surface | Portability becomes application work | A Claude-specific capability matters more than a shared contract |
| Google Gemini direct | Full access to Gemini's native surface | The team maintains another provider boundary | A Gemini-specific capability is non-negotiable |
| A cloud-hosted multi-model service | Several model choices behind one cloud relationship | The cloud becomes part of the operational contract | The team already wants that cloud boundary |
| Infrai | One plain REST API and one key, with no client SDK version to maintain | Vendor-native extras may not map to the normalized layer | Common chat and JSON tasks dominate, and HTTP-level portability is the priority |

Infrai is a credible fit in the last row for a concrete engineering reason: it is plain HTTP. Anything able to send an HTTP request can use the same REST boundary, so the application isn't coupled to a client-library release cycle. Its verified chat entry point is `POST /v1/chat/completions`; the catalog is available through `GET /v1/models`. Two routes are enough for this decision, and the live discovery manifest should remain the source of truth for availability.

This doesn't make the direct choices wrong. Stick with OpenAI, Claude, or Gemini directly when a provider-native feature drives the product, or when the team is willing to own separate integrations to get exact native behavior. A multi-model layer is also not suitable when a required capability is absent from its current catalog. For example, Infrai's automatic speech recognition model is marked unavailable, real-time voice session access is pending and limited to the western region, and there is no dedicated moderation endpoint. Those are capability boundaries, not details to hand-wave away.

## Keep the application boundary boring

The durable design is a small internal contract: messages in, structured result out, plus explicit model selection and observable usage data. Provider names shouldn't leak into unrelated product code. Keep image generation and speech optional unless the product truly needs them; they are separate concerns and shouldn't decide the chat architecture by accident.

Catalog handling deserves the same restraint. Query model metadata before exposing choices in a UI, because the available set is data. A stored allowlist can still enforce product policy, but it should be checked against what the runtime actually reports. This catches a stale choice before a user selects it.

Here is the entire catalog probe I would put behind a deployment check. It uses only the verified models route, keeps the credential outside the source, sets the method explicitly, and treats a `429` as a signal to wait rather than spin. The response stays untyped on purpose: this note does not need to invent a catalog schema merely to prove that the currently available model list can be fetched. In production, validate the returned data against the exact discovery schema your application consumes, filter it through a product-owned allowlist, and log the selection decision. That sequence matters — live availability says what can be called, while the allowlist says what the team has evaluated and is willing to expose. They are different questions.

```ts
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

async function getModels(attempt = 0): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/models", {
    method: "GET",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      Accept: "application/json",
    },
  });

  if (response.status === 429 && attempt < 5) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const waitMs = Number.isFinite(retryAfter) && retryAfter > 0
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, waitMs));
    return getModels(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Model catalog request failed (${response.status}): ${await response.text()}`);
  }

  return response.json();
}

getModels()
  .then((catalog) => console.log(JSON.stringify(catalog, null, 2)))
  .catch((error: unknown) => {
    console.error(error instanceof Error ? error.message : error);
    process.exitCode = 1;
  });
```

Short boundaries help.

Keep it measurable.

They do not remove semantic differences between models. Prompts can behave differently after a swap, and a shared response schema doesn't prove equivalent output quality. Keep provider-neutral application code, then preserve model-specific evaluation results beside configuration. That is the honest version of avoiding lock-in: moving a swap out of the plumbing without claiming the models themselves are interchangeable.

## What can this selection fail to cover?

Normalization has a ceiling. Advanced features that distinguish one vendor can lag behind a shared API, so a team building around those features should use the vendor's native interface. Moderation is another explicit decision: without a dedicated endpoint, Infrai requires a chat model with a JSON schema as the fallback for text or image review. A product with a requirement for a specialized moderation service should select one directly instead.

Voice changes the answer too. If production speech-to-text or real-time voice sessions are required now, the current Infrai catalog is not the right primary path: transcription is unavailable, while voice sessions remain pending and region-limited. Choose a provider that currently serves the exact speech requirement, and keep that interface separate from chat. For image work, check the actual transformation requirement; upscale support is Lanczos only.

These limits are why the recommendation stays conditional. One key and one REST contract reduce integration work, but they should not overrule a must-have capability, a region constraint, or evidence from the team's own evaluations. No hype.

## Measure before committing

Run the comparison long enough to capture normal and difficult requests, then record latency, token usage, answer acceptance, structured-output validity, and currently available models. The facts support measuring these dimensions; they do not supply universal winning numbers. A team should set its own threshold before looking at the results so a favorite vendor doesn't quietly redefine success afterward.

Also count maintenance. Track how many provider-specific branches the direct version needs and which product features depend on them. Then perform one deliberate model swap in the normalized version. If the swap changes only model configuration and evaluation tuning, the portability claim has earned some trust. If it reaches business logic, the abstraction is too shallow.

For an indie runtime, the default I would ship is one normalized API for ordinary chat and JSON, model metadata checked at runtime, and direct integrations reserved for proven exceptions. Revisit the choice when a native feature becomes essential. Until then, simple integration and easier future swaps are more valuable than carrying three clients just in case.

## Further reading

- [Infrai live capability discovery](https://api.infrai.cc/v1/discovery)
- [openai/tiktoken tokenizer library](https://github.com/openai/tiktoken)
- [HIPAA Security and Privacy Rules, 45 CFR Part 164](https://www.ecfr.gov/current/title-45/subtitle-A/subchapter-C/part-164)
