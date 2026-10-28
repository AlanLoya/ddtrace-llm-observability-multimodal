[---
name: ddtrace-llm-observability-multimodal
description: "Implement and troubleshoot Datadog LLM Observability with ddtrace for multimodal LLM calls. Use when adding LLMObs spans, annotating image or audio inputs, preventing truncated payload DATA_ISSUE evaluations, sanitizing URLs and secrets, enforcing structured output confidence, or migrating this pattern to another Python project."
argument-hint: "Describe the LLM call, modality, provider, and output schema to instrument."
user-invocable: true
---

# ddtrace LLM Observability for Multimodal Calls

Instrument multimodal LLM calls so Datadog can evaluate the actual source content without
leaking credentials, duplicating large payloads, or producing unsupported structured output.
This workflow applies to Python services using `ddtrace.llmobs.LLMObs`.

## Core rule

Send visual or audio bytes as typed multimodal parts, not as text.

For an image, the Datadog annotation must use:

```python
LLMObs.annotate(
    span=llm_span,
    input_data={
        "role": "user",
        "content": sanitized_prompt,
        "image_parts": [
            {
                "mime_type": "image/png",
                "content": base64_content,
            }
        ],
    },
    output_data={
        "role": "assistant",
        "content": json.dumps(result, ensure_ascii=False),
    },
)
```

Never append a `data:image/...;base64,...` value to `content`. Datadog truncates long
textual span fields and evaluators then report issues such as:

- `inline base64 content is explicitly truncated`;
- `complete image content is unavailable`;
- `the source is not fully available for assessment`.

`image_parts` is the multimodal field. Keep the source there only once.

## Procedure

### 1. Identify the actual LLM boundary

1. Locate the function that makes the provider request.
2. Wrap that request in `LLMObs.llm(...)` when LLM Observability is enabled.
3. Use a `contextlib.nullcontext()` fallback when the application also runs without
   LLM Observability.
4. Annotate the same span after receiving the response.
5. Keep the provider request payload and the Datadog annotation as separate concerns:
   the provider may need a data URL, while Datadog should receive `image_parts`.

```python
llm_context = (
    LLMObs.llm(
        model_name=os.getenv("LLM_DEPLOYMENT", "unknown"),
        name="service.operation",
        model_provider="provider",
    )
    if LLMObs.enabled
    else contextlib.nullcontext()
)

with llm_context as llm_span:
    response = client.post(...)
    result = parse_response(response)
    if LLMObs.enabled:
        LLMObs.annotate(span=llm_span, input_data=..., output_data=...)
```

### 2. Build valid multimodal parts

For every source image:

1. Download or read the bytes using the existing application path.
2. Verify that the payload is a valid `data:image/...;base64,...` value before annotating.
3. Split the header and payload.
4. Decode with strict base64 validation.
5. Set the actual MIME type (`image/png`, `image/jpeg`, `image/gif`, or `image/webp`).
6. Put only the base64 payload, without the `data:` prefix, in `image_parts[].content`.
7. If the payload is too large, resize and recompress it while preserving legibility.
8. If preparation fails, omit the invalid part and log a warning; do not emit a fake
   successful image.

Use a bounded budget. The exact value depends on deployment limits, but a practical
starting point is no more than 1.5 MB decoded per image and a maximum dimension near
1600 px. For documents containing small text, do not reduce to thumbnail-sized images:
excessive compression can make the visual source unverifiable.

Do not put the same image in both `content` and `image_parts`. Duplication increases the
chance that the span exceeds the event limit and gets truncated.

### 3. Sanitize prompts and references

Prompt text may contain URLs with SAS tokens, query credentials, or local paths. Before
annotation:

1. Remove query strings and fragments from HTTP(S) URLs.
2. Replace local paths with their basename.
3. Preserve the actual image in `image_parts`; sanitization applies only to text.
4. Never log or annotate API keys, SAS tokens, cookies, authorization headers, or raw
   filesystem paths outside the image part.

The provider request can still use the original URL or data URL. Only the observability
copy must be sanitized.

### 4. Enforce structured output at the parser boundary

Do not trust that a prompt alone will produce a safe response. Parse and validate before
annotating output or returning a value to the application.

For visual name extraction:

```python
{
    "full_name": "VISIBLE NAME",
    "confidence": 1.0,
}
```

Reject the name and return `{"full_name": None}` when any of these conditions holds:

- the response is not valid JSON;
- `confidence` is absent, boolean, non-numeric, outside `0.0..1.0`, or below `0.95`;
- the name is empty, null-like, or not a string;
- the model did not provide the required evidence field.

If the application only needs `full_name`, it may consume that field, but the Datadog
output annotation should preserve the validated `confidence` field so the observed output
matches the declared contract.

For every extraction prompt, explicitly require:

- no guessing, completion, correction, or inference;
- `null` when any character is uncertain;
- no inference from filenames, URLs, signatures, context, or external knowledge;
- valid JSON only;
- the confidence field and its threshold.

### 5. Annotate errors explicitly

If the provider request or parser fails:

1. Log the exception with the operation name.
2. Annotate an error-shaped output with the operation error type.
3. Return the repository-standard failure value.
4. Never return a plausible name, score, or success-shaped fallback.

Keep exception handling narrow around the provider boundary and parser. Do not silently
swallow invalid image or response data.

### 6. Add focused tests

At minimum, test:

1. `LLMObs.annotate` receives `image_parts` with the expected MIME type and base64
   payload.
2. `input_data["content"]` contains no `base64` or data URL.
3. URL query strings and fragments are removed from annotated prompt text.
4. Oversized images are resized/recompressed and remain valid.
5. Missing or invalid images are not represented as valid image parts.
6. Valid structured output preserves the confidence field.
7. Missing, low, boolean, or invalid confidence returns the null result.
8. Provider exceptions annotate/log an explicit failure and do not fabricate output.

Use mocks for the provider and `LLMObs`. Do not require live Datadog or provider
credentials in unit tests.

## Datadog DATA_ISSUE troubleshooting

When an evaluator reports a truncated or unavailable source:

1. Inspect the span input and find whether base64 appears inside textual `content`.
2. Remove every inline data URL from textual fields.
3. Confirm the complete image is present in `image_parts`.
4. Confirm the MIME type is renderable and the content is valid base64.
5. Reduce only oversized image parts, not the prompt text.
6. Verify that the output contains every required schema field.
7. Re-run focused tests, then inspect a newly emitted span.

Common incorrect implementations:

- **Base64 in prompt text:** gets truncated and causes `DATA_ISSUE`.
- **Only `mime_type` and an attachment reference:** evaluators may not have visual content
  in their evaluation context.
- **Tiny thumbnail fallback:** the image exists but document text is no longer legible.
- **Prompt-only confidence requirement:** the model can omit it; parser validation is
  mandatory.
- **Sanitizing the provider payload instead of only the annotation:** can break the LLM
  request or remove the source image.
- **Returning a guessed name on parse failure:** creates hallucination findings.

## Verification checklist

- [ ] LLM span name and provider are stable and meaningful.
- [ ] `content` is text only; it contains no base64 data URL.
- [ ] Images are represented in `image_parts`.
- [ ] Image MIME types are renderable and payloads are strict base64.
- [ ] Image size and dimension limits are enforced.
- [ ] Annotation text has no secrets or SAS parameters.
- [ ] Output is parsed and schema-validated before annotation.
- [ ] Unverifiable visual extraction returns `null`.
- [ ] Error paths are logged and explicitly annotated.
- [ ] Focused tests cover payload shape, truncation prevention, and parser rejection cases.
](https://dev.azure.com/ab-inbev-maz/MAZ_BTA/_git/maz-bta-spare-finder-crewai/pullrequest/97708)
