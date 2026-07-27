---
name: structured-output
description: >
  Use Interfaze AI to produce strict JSON or schema-constrained structured objects
  from any prompt. Activate when the user needs typed, machine-readable output —
  JSON objects, arrays, validated fields, or Zod/Pydantic-style extraction. Use this
  when the task requires a specific output shape, data normalization, or converting
  unstructured text into structured records, even if the user just says "give me this
  as JSON" or "parse this into fields."
---

# Structured Output with Interfaze AI

Generate outputs into strict, schema-validated JSON using Zod (TypeScript) or Pydantic (Python) schemas.

## When to use this skill

- The user needs output in a specific JSON shape or typed format
- The task requires extracting structured fields from unstructured text
- The user says "give me JSON", "parse this into an object", "extract these fields"
- You need to validate AI output against a schema before using it downstream
- The output will be consumed by code, not displayed to a human

## When not to use this skill

- The user wants free-form text, summaries, or conversational responses
- The input is an image that needs OCR first → use `ocr`, then apply structured output
- The input is audio → use `speech-to-text` first, then apply structured output if needed

## Workflow

1. Identify the target output shape from the user's request.
2. Define a Zod (TypeScript) or Pydantic (Python) schema matching the expected structure.
3. Pass the schema via the SDK's structured-output mechanism:
   - Vercel AI SDK: `generateObject({ schema })`
   - OpenAI SDK: `response_format` with `zodResponseFormat` (TS) or `json_schema` (Python)
   - LangChain SDK: `withStructuredOutput` / `with_structured_output`

## Setup

### TypeScript — OpenAI SDK

```ts
import OpenAI from "openai";

const interfaze = new OpenAI({
  baseURL: "https://api.interfaze.ai/v1",
  apiKey: process.env.INTERFAZE_API_KEY,
});
```

### TypeScript — Vercel AI SDK

```ts
import { createOpenAI } from "@ai-sdk/openai";

const interfaze = createOpenAI({
  baseURL: "https://api.interfaze.ai/v1",
  apiKey: process.env.INTERFAZE_API_KEY,
});
```

### TypeScript — LangChain SDK

```ts
import { ChatOpenAI } from "@langchain/openai";

const interfaze = new ChatOpenAI({
  configuration: { baseURL: "https://api.interfaze.ai/v1" },
  apiKey: process.env.INTERFAZE_API_KEY,
  model: "interfaze-beta",
});
```

### Python — OpenAI SDK

```python
from openai import OpenAI

interfaze = OpenAI(base_url="https://api.interfaze.ai/v1", api_key="<your-api-key>")
```

### Python — LangChain SDK

```python
from langchain_openai import ChatOpenAI

interfaze = ChatOpenAI(base_url="https://api.interfaze.ai/v1", api_key="<your-api-key>", model="interfaze-beta")
```

## Example: generate a structured object from a prompt

### TypeScript — OpenAI SDK

```ts
import { z } from "zod";
import { zodResponseFormat } from "openai/helpers/zod";

const CodeSchema = z.object({
  code: z.string(),
  sample_input: z.string(),
  sample_output: z.string(),
});

const response = await interfaze.chat.completions.create({
  model: "interfaze-beta",
  messages: [
    { role: "user", content: "Write a Python script to check CPU type using subprocess." },
  ],
  response_format: zodResponseFormat(CodeSchema, "code_schema"),
});

console.log(response.choices[0].message.content);
```

### TypeScript — Vercel AI SDK

```ts
import { generateObject } from "ai";
import { z } from "zod";

const CodeSchema = z.object({
  code: z.string(),
  sample_input: z.string(),
  sample_output: z.string(),
});

const { object } = await generateObject({
  model: interfaze.chat("interfaze-beta"),
  schema: CodeSchema,
  prompt: "Write a Python script to check CPU type using subprocess.",
});

console.log(object);
```

### TypeScript — LangChain SDK

```ts
import { z } from "zod";

const CodeSchema = z.object({
  code: z.string(),
  sample_input: z.string(),
  sample_output: z.string(),
});

const structuredModel = interfaze.withStructuredOutput(CodeSchema);

const response = await structuredModel.invoke([
  { role: "user", content: "Write a Python script to check CPU type using subprocess." },
]);

console.log(response);
```

### Python — OpenAI SDK

```python
from pydantic import BaseModel

class CodeSchema(BaseModel):
    code: str
    sample_input: str
    sample_output: str

response = interfaze.chat.completions.create(
    model="interfaze-beta",
    messages=[
        {"role": "user", "content": "Write a Python script to check CPU type using subprocess."},
    ],
    response_format={
        "type": "json_schema",
        "json_schema": {"name": "code_schema", "schema": CodeSchema.model_json_schema()},
    },
)

print(response.choices[0].message.content)
```

### Python — LangChain SDK

```python
from pydantic import BaseModel

class CodeSchema(BaseModel):
    code: str
    sample_input: str
    sample_output: str

structured_llm = interfaze.with_structured_output(CodeSchema)

response = structured_llm.invoke("Write a Python script to check CPU type using subprocess.")

print(response)
```

## Example: array of structured objects

Use an array at the top level (or a wrapper object with a list field):

```ts
const { object } = await generateObject({
  model: interfaze.chat("interfaze-beta"),
  schema: z.array(
    z.object({
      name: z.string(),
      rank: z.number(),
      primary_use_case: z.string(),
    })
  ),
  prompt: "List the top 5 programming languages by popularity.",
});
```
```python
class Language(BaseModel):
    name: str
    rank: int
    primary_use_case: str

class LanguagesSchema(BaseModel):
    languages: list[Language]
```

## Schema design tips

- Use `.nullable()` (Zod) or `Optional[T] = None` (Pydantic) for fields that may not be present.
- Use `.describe()` (Zod) or `Field(description=...)` (Pydantic) to clarify ambiguous fields — the model reads these descriptions.
- Use `z.array(...)` / `List[...]` at the top level when extracting lists.
- Keep schemas focused. Prefer multiple targeted calls over one massive schema.

## Schema patterns

Basic objects and arrays are shown above. Other common shapes:

### Nullable fields

Use when a field may not exist in the source data.

```ts
z.object({
  name: z.string(),
  email: z.string().nullable(),
  phone: z.string().nullable(),
})
```
```python
from typing import Optional

class Contact(BaseModel):
    name: str
    email: Optional[str] = None
    phone: Optional[str] = None
```

### Described fields

Use `.describe()` (Zod) or `Field(description=...)` (Pydantic) when the field name alone is ambiguous — the model reads these.

```ts
z.object({
  items: z.array(z.object({ name: z.string(), price: z.string() })),
  highlighted_items: z.array(
    z.object({ name: z.string(), price: z.string() })
  ).describe("items that are visually highlighted or marked in the source"),
})
```
```python
from pydantic import Field

class Item(BaseModel):
    name: str
    price: str

class ReceiptSchema(BaseModel):
    items: list[Item]
    highlighted_items: list[Item] = Field(
        ..., description="items that are visually highlighted or marked in the source"
    )
```

### Nested coordinates (spatial extraction)

```ts
z.object({
  text_in_original_language: z.string(),
  bounds: z.array(
    z.object({
      top_left_x: z.number(),
      top_left_y: z.number(),
      bottom_right_x: z.number(),
      bottom_right_y: z.number(),
    })
  ).describe("bounding box for each line"),
})
```
```python
from pydantic import Field

class Bound(BaseModel):
    top_left_x: float
    top_left_y: float
    bottom_right_x: float
    bottom_right_y: float

class SpatialSchema(BaseModel):
    text_in_original_language: str
    bounds: list[Bound] = Field(..., description="bounding box for each line")
```

## Response handling

- All SDKs return an object that matches your schema exactly.
- Vercel AI SDK: `const { object } = await generateObject({ schema, ... })`; invalid responses are retried automatically.
- OpenAI SDK: parse `response.choices[0].message.content` as JSON.
- LangChain SDK: the call returns the typed object directly.
