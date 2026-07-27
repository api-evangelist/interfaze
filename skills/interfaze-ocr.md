---
name: ocr
description: >
  Use Interfaze AI for OCR and visual text extraction from images, screenshots,
  scans, PDFs, invoices, receipts, IDs, forms, menus, and labels with complex
  layouts. Activate when the user wants to extract text, fields, or structured
  data from an image or scanned document — even if they say "read this", "what
  does this say", or "pull the data from this photo" without mentioning OCR.
---

# OCR — Visual OCR with Interfaze AI

Extract text and structured fields from images and scanned documents using Interfaze AI's vision OCR capability.

## When to use this skill

- The user has an image, screenshot, scan, or photo and wants text extracted from it
- The input is a receipt, invoice, ID card, form, menu, or label captured as an image
- The user wants structured fields (name, address, totals, line items) pulled from a visual source
- The user says "read this image", "extract the text", "what does this receipt say"
- The task involves layout-aware extraction where text position matters

## When not to use this skill

- The input is a native text file, JSON, or CSV — no vision needed
- The user wants to transcribe audio → use `speech-to-text`
- The task is locating objects without reading text → use `object-detection`
- The user wants to convert existing data to a schema → use `structured-output`
- The user wants to pull data from a live URL → use `web-scraping`

## Workflow

1. Confirm the input is an image URL, PDF URL, or base64-encoded file.
2. Decide whether you need plain text or structured fields.
3. For structured fields, define a Zod (TS) or Pydantic (Python) schema.
4. Pass the file in the message `content` array using the SDK's file/image part.
5. Match the SDK and language to the user's project.

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

## Input formats

Model name is always `interfaze-beta`. Supported sources: public HTTPS URLs and base64 data.

| SDK | Image part | PDF / document part |
| --- | --- | --- |
| Vercel AI SDK | `{ type: "image", mediaType: "image/jpeg", image: "<url-or-base64>" }` | `{ type: "file", data: "<url>", mediaType: "application/pdf" }` |
| OpenAI / LangChain (TS & Py) | `{ type: "image_url", image_url: { url: "<url>" } }` | `{ type: "file", file: { filename: "doc.pdf", file_data: "<url-or-base64>" } }` |

## Example: extract fields from an ID card

### TypeScript — OpenAI SDK

```ts
import { z } from "zod";
import { zodResponseFormat } from "openai/helpers/zod";

const IDSchema = z.object({
  first_name: z.string().describe("First name on the ID"),
  last_name: z.string().describe("Last name on the ID"),
  dob: z.string().describe("Date of birth on the ID"),
  driver_licence_number: z.string().describe("Driver licence number on the ID"),
});

const response = await interfaze.chat.completions.create({
  model: "interfaze-beta",
  messages: [
    {
      role: "user",
      content: [
        { type: "text", text: "Extract the details from this ID" },
        { type: "image_url", image_url: { url: "https://example.com/id.jpg" } },
      ],
    },
  ],
  response_format: zodResponseFormat(IDSchema, "id_schema"),
});

console.log(response.choices[0].message.content);
```

### TypeScript — Vercel AI SDK

```ts
import { generateObject } from "ai";
import { z } from "zod";

const IDSchema = z.object({
  first_name: z.string().describe("First name on the ID"),
  last_name: z.string().describe("Last name on the ID"),
  dob: z.string().describe("Date of birth on the ID"),
  driver_licence_number: z.string().describe("Driver licence number on the ID"),
});

const { object } = await generateObject({
  model: interfaze.chat("interfaze-beta"),
  schema: IDSchema,
  messages: [
    {
      role: "user",
      content: [
        { type: "text", text: "Extract the details from this ID" },
        { type: "image", mediaType: "image/jpeg", image: "https://example.com/id.jpg" },
      ],
    },
  ],
});

console.log(object);
```

### TypeScript — LangChain SDK

```ts
import { z } from "zod";

const IDSchema = z.object({
  first_name: z.string().describe("First name on the ID"),
  last_name: z.string().describe("Last name on the ID"),
  dob: z.string().describe("Date of birth on the ID"),
  driver_licence_number: z.string().describe("Driver licence number on the ID"),
});

const structuredModel = interfaze.withStructuredOutput(IDSchema);

const response = await structuredModel.invoke([
  {
    role: "user",
    content: [
      { type: "text", text: "Extract the details from this ID" },
      { type: "image_url", image_url: { url: "https://example.com/id.jpg" } },
    ],
  },
]);

console.log(response);
```

### Python — OpenAI SDK

```python
from pydantic import BaseModel, Field

class IDSchema(BaseModel):
    first_name: str = Field(..., description="First name on the ID")
    last_name: str = Field(..., description="Last name on the ID")
    dob: str = Field(..., description="Date of birth on the ID")
    driver_licence_number: str = Field(..., description="Driver licence number on the ID")

response = interfaze.chat.completions.create(
    model="interfaze-beta",
    messages=[
        {
            "role": "user",
            "content": [
                {"type": "text", "text": "Extract the details from this ID"},
                {"type": "image_url", "image_url": {"url": "https://example.com/id.jpg"}},
            ],
        }
    ],
    response_format={
        "type": "json_schema",
        "json_schema": {"name": "id_schema", "schema": IDSchema.model_json_schema()},
    },
)

print(response.choices[0].message.content)
```

### Python — LangChain SDK

```python
from langchain_core.messages import HumanMessage
from pydantic import BaseModel, Field

class IDSchema(BaseModel):
    first_name: str = Field(..., description="First name on the ID")
    last_name: str = Field(..., description="Last name on the ID")
    dob: str = Field(..., description="Date of birth on the ID")
    driver_licence_number: str = Field(..., description="Driver licence number on the ID")

structured_llm = interfaze.with_structured_output(IDSchema)

response = structured_llm.invoke([
    HumanMessage(content=[
        {"type": "text", "text": "Extract the details from this ID"},
        {"type": "image_url", "image_url": {"url": "https://example.com/id.jpg"}},
    ])
])

print(response)
```

## Example: receipt line items

Same call as the ID card above — only the schema changes. Pass this schema to `response_format` / `generateObject` / `withStructuredOutput`:

```ts
const ReceiptSchema = z.object({
  items: z.array(z.object({ name: z.string(), price: z.string() })),
  total_cost: z.string(),
  tax: z.string(),
});
```
```python
class ReceiptItem(BaseModel):
    name: str
    price: str

class ReceiptSchema(BaseModel):
    items: list[ReceiptItem]
    total_cost: str
    tax: str
```

## Plain text (no schema)

Use `generateText` (Vercel AI SDK), or omit `response_format` (OpenAI / LangChain) when you only need raw text:

```ts
import { generateText } from "ai";

const { text } = await generateText({
  model: interfaze.chat("interfaze-beta"),
  messages: [
    {
      role: "user",
      content: [
        { type: "text", text: "Extract all text from this image" },
        { type: "image", image: "https://example.com/scan.jpg" },
      ],
    },
  ],
});
```

## Tips

- Use `.describe()` / `Field(description=...)` when a field name alone is ambiguous.
- For multilingual text, name the target language in the prompt or field descriptions. 100+ languages, including mixed-language documents, are supported.
- For layout-aware extraction, include coordinate fields in your schema.

## Advanced: raw OCR output & bounding boxes

For raw OCR (no custom schema) with line-level bounding boxes and per-word confidence, set `<task>ocr</task>` in the system message. The response's `precontext` contains `extracted_text` and `sections` with line-level bounding boxes and per-word confidence scores.

### TypeScript — OpenAI SDK

```ts
const response = await interfaze.chat.completions.create({
  model: "interfaze-beta",
  messages: [
    { role: "system", content: "<task>ocr</task>" },
    {
      role: "user",
      content: [
        { type: "text", text: "Extract all text from this image" },
        { type: "image_url", image_url: { url: "<url>" } },
      ],
    },
  ],
  response_format: zodResponseFormat(z.any(), "empty_schema"),
});
```

### TypeScript — Vercel AI SDK

```ts
const { object } = await generateObject({
  model: interfaze.chat("interfaze-beta"),
  system: "<task>ocr</task>",
  schema: z.any(),
  messages: [
    {
      role: "user",
      content: [
        { type: "text", text: "Extract all text from this image" },
        { type: "image", mediaType: "image/jpeg", image: "<url>" },
      ],
    },
  ],
});
```

### Python — OpenAI SDK

```python
response = interfaze.chat.completions.create(
    model="interfaze-beta",
    messages=[
        {"role": "system", "content": "<task>ocr</task>"},
        {
            "role": "user",
            "content": [
                {"type": "text", "text": "Extract all text from this image"},
                {"type": "image_url", "image_url": {"url": "<url>"}},
            ],
        },
    ],
    response_format={
        "type": "json_schema",
        "json_schema": {
            "name": "empty_schema",
            "schema": {"type": "object", "properties": {}, "additionalProperties": True},
        },
    },
)
```

For non-task calls, raw OCR results are also available on `response.precontext` (OpenAI SDK / Python) or `response.body?.precontext` (Vercel AI SDK):

```ts
// @ts-expect-error precontext is not typed
const precontext = response.precontext ?? response.body?.precontext;
console.log("OCR Results:", precontext[0]?.result);
```
