---
name: web-search
description: >
  Use Interfaze AI to search the web and retrieve current information, facts, prices,
  news, or any real-time data. Activate when the user needs up-to-date information
  from the internet, wants to look something up, or asks a question that requires
  current knowledge — even if they say "look this up", "find out about", or "what's
  the latest on" without explicitly mentioning web search.
---

# Web Search with Interfaze AI

Search the web and retrieve current information using Interfaze AI's built-in web search index. No tool config required — just send a prompt that needs current information and the model searches as needed.

## When to use this skill

- The user needs current or real-time information that isn't in the model's training data
- The user asks about prices, availability, news, events, or recent developments
- The user says "search for", "look up", "find out about", "what's the latest"
- The task requires verifying facts against current web sources
- The user provides a topic and needs comprehensive, up-to-date information

## When not to use this skill

- The question can be answered from the model's existing knowledge (e.g. "what is 2 + 2", "explain `let` vs `const`")
- The user wants to extract data from a specific known URL → use `web-scraping`
- The user has an image or audio to process → use `ocr`, `object-detection`, or `speech-to-text`

## Workflow

1. Identify what information the user needs from the web.
2. Formulate a clear, specific search prompt.
3. Use a structured-output call for typed results, or a plain text call for a natural-language summary.
4. The model returns results directly; raw hits are available via `precontext`.

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

## Example: basic web search (text summary)

### TypeScript — OpenAI SDK

```ts
const response = await interfaze.chat.completions.create({
  model: "interfaze-beta",
  messages: [{ role: "user", content: "Latest news on Nvidia" }],
});

console.log(response.choices[0].message.content);
```

### TypeScript — Vercel AI SDK

```ts
import { generateText } from "ai";

const { text } = await generateText({
  model: interfaze.chat("interfaze-beta"),
  prompt: "Latest news on Nvidia",
});

console.log(text);
```

### TypeScript — LangChain SDK

```ts
const response = await interfaze.invoke("Latest news on Nvidia");

console.log(response.content);
```

### Python — OpenAI SDK

```python
response = interfaze.chat.completions.create(
    model="interfaze-beta",
    messages=[{"role": "user", "content": "Latest news on Nvidia"}],
)

print(response.choices[0].message.content)
```

### Python — LangChain SDK

```python
response = interfaze.invoke("Latest news on Nvidia")

print(response.content)
```

## Example: structured web search

### TypeScript — OpenAI SDK

```ts
import { z } from "zod";
import { zodResponseFormat } from "openai/helpers/zod";

const NvidiaNewsSchema = z.object({
  summary: z.string(),
  current_stock_price: z.number(),
  links: z.array(z.string()),
});

const response = await interfaze.chat.completions.create({
  model: "interfaze-beta",
  messages: [{ role: "user", content: "Latest news on Nvidia" }],
  response_format: zodResponseFormat(NvidiaNewsSchema, "nvidia_news_schema"),
});
```

### TypeScript — Vercel AI SDK

```ts
import { generateObject } from "ai";
import { z } from "zod";

const NvidiaNewsSchema = z.object({
  summary: z.string(),
  current_stock_price: z.number(),
  links: z.array(z.string()),
});

const { object } = await generateObject({
  model: interfaze.chat("interfaze-beta"),
  schema: NvidiaNewsSchema,
  prompt: "Latest news on Nvidia",
});
```

### TypeScript — LangChain SDK

```ts
import { z } from "zod";

const NvidiaNewsSchema = z.object({
  summary: z.string(),
  current_stock_price: z.number(),
  links: z.array(z.string()),
});

const structuredModel = interfaze.withStructuredOutput(NvidiaNewsSchema);

const response = await structuredModel.invoke("Latest news on Nvidia");
```

### Python — OpenAI SDK

```python
from typing import List
from pydantic import BaseModel

class NvidiaNewsSchema(BaseModel):
    summary: str
    current_stock_price: float
    links: List[str]

response = interfaze.chat.completions.create(
    model="interfaze-beta",
    messages=[{"role": "user", "content": "Latest news on Nvidia"}],
    response_format={
        "type": "json_schema",
        "json_schema": {"name": "nvidia_news_schema", "schema": NvidiaNewsSchema.model_json_schema()},
    },
)
```

### Python — LangChain SDK

```python
from typing import List
from pydantic import BaseModel

class NvidiaNewsSchema(BaseModel):
    summary: str
    current_stock_price: float
    links: List[str]

structured_llm = interfaze.with_structured_output(NvidiaNewsSchema)

response = structured_llm.invoke("Latest news on Nvidia")
```

For structured queries, Interfaze may also follow up with a web extract pass — that result appears as a separate `precontext` entry with `name: "web_extract"`.

## Example: look up a company

Same call — use a schema shaped for the entity:

```ts
const { object } = await generateObject({
  model: interfaze.chat("interfaze-beta"),
  schema: z.object({
    company_name: z.string(),
    description: z.string(),
    headquarters: z.string().nullable(),
    industry: z.string(),
    key_people: z.array(z.string()),
  }),
  prompt: "Find information about Acme Corp from their website and LinkedIn.",
});
```

## Raw results via precontext

Raw search hits (`title`, `description`, `content`, `snippets`, `url` per hit) are returned on the response's `precontext` array — `response.precontext` (OpenAI SDK / Python) or `response.body?.precontext` (Vercel AI SDK).

```ts
// @ts-expect-error precontext is not typed
const precontext = response.precontext ?? response.body?.precontext;
console.log(precontext[0]?.result); // array of search hits
```

## Coverage

Interfaze's web search covers research indexes (biology, finance, medicine, law, arxiv), social platforms (LinkedIn, X, personal sites), financial data (stocks, forex, crypto, commodities, SEC filings), and general (news, events, products, company info).

## Tips

- Be specific. "Find the price of X on Amazon" works better than "look up X."
- Use structured output when you need machine-readable results.
- A web search is triggered automatically when the prompt requires current information.
- To normalize results further, combine with `structured-output`.
