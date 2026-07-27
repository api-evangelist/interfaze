---
name: web-scraping
description: >
  Use Interfaze AI to scrape and extract structured data from specific web pages and
  URLs. Activate when the user provides a URL and wants to pull products, listings,
  prices, profiles, or any structured content from that page — even if they say
  "get the data from this page", "pull the listings from this URL", or "extract
  products from this site" without explicitly mentioning scraping.
---

# Web Scraping with Interfaze AI

Extract structured data from web pages by including a URL in the prompt and defining a schema for the desired output. Interfaze has built-in residential proxies, auto-scaling browser infrastructure, and bot-block handling — no separate scraping library required.

## When to use this skill

- The user provides a specific URL and wants data extracted from it
- The task involves pulling product listings, prices, profiles, or structured content from a webpage
- The user says "scrape this page", "get the data from this URL", "extract products from this site"
- The task requires pulling data from an e-commerce site, job board, directory, or social profile

## When not to use this skill

- The user wants to search the web without a specific URL → use `web-search`
- The user has an image → use `ocr` or `object-detection`
- The user has audio → use `speech-to-text`
- The user wants to reshape data they already have → use `structured-output`
- The URL points to a downloadable file rather than a web page

## Workflow

1. Confirm the target URL.
2. Define a schema (Zod for TypeScript, Pydantic for Python) matching the data the user wants.
3. Include the URL inline in the prompt; the model fetches and parses the page.
4. Return the structured data.

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

## Example: scrape product listings

The URL goes inline in the prompt — there is no separate URL parameter.

### TypeScript — OpenAI SDK

```ts
import { z } from "zod";
import { zodResponseFormat } from "openai/helpers/zod";

const ProductSchema = z.object({
  price: z.number(),
  listing_name: z.string(),
});

const response = await interfaze.chat.completions.create({
  model: "interfaze-beta",
  messages: [
    {
      role: "user",
      content: "Extract the information from https://www.amazon.com/Nintendo-Switch-Neon-Blue-Joy-Con/dp/B0BFJWCYTL",
    },
  ],
  response_format: zodResponseFormat(ProductSchema, "product_schema"),
});

console.log(response.choices[0].message.content);
```

### TypeScript — Vercel AI SDK

```ts
import { generateObject } from "ai";
import { z } from "zod";

const ProductSchema = z.object({
  price: z.number(),
  listing_name: z.string(),
});

const { object } = await generateObject({
  model: interfaze.chat("interfaze-beta"),
  schema: ProductSchema,
  prompt: "Extract the information from https://www.amazon.com/Nintendo-Switch-Neon-Blue-Joy-Con/dp/B0BFJWCYTL",
});
```

### TypeScript — LangChain SDK

```ts
import { z } from "zod";

const ProductSchema = z.object({
  price: z.number(),
  listing_name: z.string(),
});

const structuredModel = interfaze.withStructuredOutput(ProductSchema);

const response = await structuredModel.invoke(
  "Extract the information from https://www.amazon.com/Nintendo-Switch-Neon-Blue-Joy-Con/dp/B0BFJWCYTL"
);
```

### Python — OpenAI SDK

```python
from pydantic import BaseModel

class ProductSchema(BaseModel):
    price: float
    listing_name: str

response = interfaze.chat.completions.create(
    model="interfaze-beta",
    messages=[
        {"role": "user", "content": "Extract the information from https://www.amazon.com/Nintendo-Switch-Neon-Blue-Joy-Con/dp/B0BFJWCYTL"},
    ],
    response_format={
        "type": "json_schema",
        "json_schema": {"name": "product_schema", "schema": ProductSchema.model_json_schema()},
    },
)

print(response.choices[0].message.content)
```

### Python — LangChain SDK

```python
from pydantic import BaseModel

class ProductSchema(BaseModel):
    price: float
    listing_name: str

structured_llm = interfaze.with_structured_output(ProductSchema)

response = structured_llm.invoke(
    "Extract the information from https://www.amazon.com/Nintendo-Switch-Neon-Blue-Joy-Con/dp/B0BFJWCYTL"
)
```

## Example: scrape a social profile

Same call — swap in the schema:

```ts
const LinkedInProfileSchema = z.object({
  first_name: z.string(),
  last_name: z.string(),
  location: z.string(),
  latest_education: z.string(),
  current_job: z.string(),
  followers: z.number(),
});
// prompt: "Extract information from https://www.linkedin.com/in/example-user/"
```
```python
class LinkedInProfileSchema(BaseModel):
    first_name: str
    last_name: str
    location: str
    latest_education: str
    current_job: str
    followers: int
```

## Example: scrape an array of listings

Use an array at the top level to extract many items:

```ts
const { object } = await generateObject({
  model: interfaze.chat("interfaze-beta"),
  schema: z.array(
    z.object({
      title: z.string(),
      url: z.string(),
      points: z.number().nullable(),
    })
  ),
  prompt: "Extract all article titles, URLs, and points from https://news.ycombinator.com",
});
```

## Tips

- Pass the URL inline in the prompt. No separate URL parameter is needed.
- Be specific: "get all product names and prices" works better than "scrape this page".
- Use `z.array(...)` / `List[...]` at the top level when extracting lists.
- Use `.describe()` / `Field(description=...)` for ambiguous field names.
- Residential proxies, browser infrastructure, and bot-block handling are built in — no extra setup.

## Advanced: raw scraping output

For maximum speed and lowest cost without a custom schema, set `<task>scraper</task>` in the system message — the model fetches the page and returns scraped content in a fixed structure on `precontext`.

### TypeScript — OpenAI SDK

```ts
const response = await interfaze.chat.completions.create({
  model: "interfaze-beta",
  messages: [
    { role: "system", content: "<task>scraper</task>" },
    { role: "user", content: "Extract post titles and points from https://news.ycombinator.com" },
  ],
  response_format: zodResponseFormat(z.any(), "scraper_schema"),
});
```

### TypeScript — Vercel AI SDK

```ts
const { object } = await generateObject({
  model: interfaze.chat("interfaze-beta"),
  system: "<task>scraper</task>",
  schema: z.any(),
  messages: [
    { role: "user", content: "Extract post titles and points from https://news.ycombinator.com" },
  ],
});
```

### TypeScript — LangChain SDK

```ts
const structuredModel = interfaze.withStructuredOutput({});

const response = await structuredModel.invoke([
  { role: "system", content: "<task>scraper</task>" },
  { role: "user", content: "Extract post titles and points from https://news.ycombinator.com" },
]);
```

### Python — OpenAI SDK

```python
response = interfaze.chat.completions.create(
    model="interfaze-beta",
    messages=[
        {"role": "system", "content": "<task>scraper</task>"},
        {"role": "user", "content": "Extract post titles and points from https://news.ycombinator.com"},
    ],
    response_format={
        "type": "json_schema",
        "json_schema": {
            "name": "scraper_schema",
            "schema": {"type": "object", "properties": {}, "additionalProperties": True},
        },
    },
)
```

### Python — LangChain SDK

```python
from langchain_core.messages import SystemMessage, HumanMessage

structured = interfaze.with_structured_output({
    "name": "scraper_schema",
    "schema": {"type": "object", "properties": {}, "additionalProperties": True},
})

response = structured.invoke([
    SystemMessage(content="<task>scraper</task>"),
    HumanMessage(content="Extract post titles and points from https://news.ycombinator.com"),
])
```
