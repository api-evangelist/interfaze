---
name: object-detection
description: >
  Use Interfaze AI to detect, locate, and identify objects in images and PDFs with
  bounding box coordinates. Activate when the user wants to find objects, get their
  positions, count items, or locate specific things in an image — even if they say
  "find the car in this photo", "where is the logo", or "what objects are in this
  image" without explicitly mentioning object detection.
---

# Object Detection with Interfaze AI

Detect and locate objects in images with bounding box coordinates using Interfaze AI.

## When to use this skill

- The user wants to find or locate objects in an image
- The user needs bounding box coordinates for detected objects
- The user wants to count specific items in a photo
- The user says "find the X in this image", "where is the Y", "what objects are here"
- The task involves spatial understanding — positions, regions, or layout of objects
- The user wants both object positions and any visible text with coordinates

## When not to use this skill

- The user only wants to read text from an image → use `ocr`
- The user wants to transcribe audio → use `speech-to-text`
- The input is not an image
- The user wants image classification without location data
- The user wants to scrape a URL → use `web-scraping`

## Workflow

1. Confirm the input is an image URL or base64-encoded image.
2. Determine what objects the user wants to detect, or if they want all objects.
3. Build a schema with object names and bounding box fields (`top_left_x`, `top_left_y`, `bottom_right_x`, `bottom_right_y`).
4. Call the chosen SDK with the image and a prompt describing what to detect.
5. Return the detected objects with their positions.

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

Model name is always `interfaze-beta`. Images pass in the message `content` array (same as OCR). Coordinates are returned in pixels relative to the original image dimensions.

| SDK | Image part |
| --- | --- |
| Vercel AI SDK | `{ type: "image", mediaType: "image/png", image: "<url>" }` |
| OpenAI / LangChain (TS & Py) | `{ type: "image_url", image_url: { url: "<url>" } }` |

## Example: detect objects and text with positions

### TypeScript — OpenAI SDK

```ts
import { z } from "zod";
import { zodResponseFormat } from "openai/helpers/zod";

const DetectionSchema = z.object({
  objects: z.array(
    z.object({
      name: z.string().describe("describe the object in the image"),
      top_left_x: z.number(),
      top_left_y: z.number(),
      bottom_right_x: z.number(),
      bottom_right_y: z.number(),
    })
  ),
  texts: z
    .array(
      z.object({
        text: z.string(),
        top_left_x: z.number(),
        top_left_y: z.number(),
        bottom_right_x: z.number(),
        bottom_right_y: z.number(),
      })
    )
    .describe("any alphabetic characters text in the image"),
});

const response = await interfaze.chat.completions.create({
  model: "interfaze-beta",
  messages: [
    {
      role: "user",
      content: [
        { type: "text", text: "Get the position of the crane in the image and any text" },
        { type: "image_url", image_url: { url: "https://example.com/construction.png" } },
      ],
    },
  ],
  response_format: zodResponseFormat(DetectionSchema, "detection_schema"),
});

console.log(response.choices[0].message.content);
```

### TypeScript — Vercel AI SDK

```ts
import { generateObject } from "ai";
import { z } from "zod";

const DetectionSchema = z.object({
  objects: z.array(
    z.object({
      name: z.string().describe("describe the object in the image"),
      top_left_x: z.number(),
      top_left_y: z.number(),
      bottom_right_x: z.number(),
      bottom_right_y: z.number(),
    })
  ),
  texts: z
    .array(
      z.object({
        text: z.string(),
        top_left_x: z.number(),
        top_left_y: z.number(),
        bottom_right_x: z.number(),
        bottom_right_y: z.number(),
      })
    )
    .describe("any alphabetic characters text in the image"),
});

const { object } = await generateObject({
  model: interfaze.chat("interfaze-beta"),
  schema: DetectionSchema,
  messages: [
    {
      role: "user",
      content: [
        { type: "text", text: "Get the position of the crane in the image and any text" },
        { type: "image", mediaType: "image/png", image: "https://example.com/construction.png" },
      ],
    },
  ],
});

console.log(object);
```

### TypeScript — LangChain SDK

```ts
import { z } from "zod";

const DetectionSchema = z.object({
  objects: z.array(
    z.object({
      name: z.string().describe("describe the object in the image"),
      top_left_x: z.number(),
      top_left_y: z.number(),
      bottom_right_x: z.number(),
      bottom_right_y: z.number(),
    })
  ),
  texts: z
    .array(
      z.object({
        text: z.string(),
        top_left_x: z.number(),
        top_left_y: z.number(),
        bottom_right_x: z.number(),
        bottom_right_y: z.number(),
      })
    )
    .describe("any alphabetic characters text in the image"),
});

const structuredModel = interfaze.withStructuredOutput(DetectionSchema);

const response = await structuredModel.invoke([
  {
    role: "user",
    content: [
      { type: "text", text: "Get the position of the crane in the image and any text" },
      { type: "image_url", image_url: { url: "https://example.com/construction.png" } },
    ],
  },
]);

console.log(response);
```

### Python — OpenAI SDK

```python
from typing import List
from pydantic import BaseModel, Field

class DetectedObject(BaseModel):
    name: str = Field(..., description="describe the object in the image")
    top_left_x: float
    top_left_y: float
    bottom_right_x: float
    bottom_right_y: float

class DetectedText(BaseModel):
    text: str
    top_left_x: float
    top_left_y: float
    bottom_right_x: float
    bottom_right_y: float

class DetectionSchema(BaseModel):
    objects: List[DetectedObject]
    texts: List[DetectedText] = Field(..., description="any alphabetic characters text in the image")

response = interfaze.chat.completions.create(
    model="interfaze-beta",
    messages=[
        {
            "role": "user",
            "content": [
                {"type": "text", "text": "Get the position of the crane in the image and any text"},
                {"type": "image_url", "image_url": {"url": "https://example.com/construction.png"}},
            ],
        }
    ],
    response_format={
        "type": "json_schema",
        "json_schema": {"name": "detection_schema", "schema": DetectionSchema.model_json_schema()},
    },
)

print(response.choices[0].message.content)
```

### Python — LangChain SDK

```python
from typing import List
from langchain_core.messages import HumanMessage
from pydantic import BaseModel, Field

class DetectedObject(BaseModel):
    name: str = Field(..., description="describe the object in the image")
    top_left_x: float
    top_left_y: float
    bottom_right_x: float
    bottom_right_y: float

class DetectedText(BaseModel):
    text: str
    top_left_x: float
    top_left_y: float
    bottom_right_x: float
    bottom_right_y: float

class DetectionSchema(BaseModel):
    objects: List[DetectedObject]
    texts: List[DetectedText] = Field(..., description="any alphabetic characters text in the image")

structured_llm = interfaze.with_structured_output(DetectionSchema)

response = structured_llm.invoke([
    HumanMessage(content=[
        {"type": "text", "text": "Get the position of the crane in the image and any text"},
        {"type": "image_url", "image_url": {"url": "https://example.com/construction.png"}},
    ])
])

print(response)
```

## Example: detect specific object categories

Same call — swap in a schema with a `category` field:

```ts
const TrafficSchema = z.object({
  objects: z.array(
    z.object({
      name: z.string(),
      category: z.string().describe("vehicle or pedestrian"),
      top_left_x: z.number(),
      top_left_y: z.number(),
      bottom_right_x: z.number(),
      bottom_right_y: z.number(),
    })
  ),
});
```
```python
class TrafficObject(BaseModel):
    name: str
    category: str = Field(..., description="vehicle or pedestrian")
    top_left_x: float
    top_left_y: float
    bottom_right_x: float
    bottom_right_y: float

class TrafficSchema(BaseModel):
    objects: list[TrafficObject]
```

## Tips

- Be specific in the prompt about what to detect. "Find all vehicles" works better than "detect objects."
- Use `.describe()` / `Field(description=...)` on the name field to guide labelling.
- Add a `category` field to group objects by type.
- Coordinates are returned in pixels relative to the original image dimensions.

## Advanced: raw detection output

For maximum speed and lowest cost without a custom schema, set `<task>object_detection</task>` in the system message — returns a fixed structure with `detected_objects` (each with `bounds` and `label`) and `gui_elements`.

### TypeScript — OpenAI SDK

```ts
const response = await interfaze.chat.completions.create({
  model: "interfaze-beta",
  messages: [
    { role: "system", content: "<task>object_detection</task>" },
    {
      role: "user",
      content: [
        { type: "text", text: "Detect objects in this image" },
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
  system: "<task>object_detection</task>",
  schema: z.any(),
  messages: [
    {
      role: "user",
      content: [
        { type: "text", text: "Detect objects in this image" },
        { type: "image", mediaType: "image/png", image: "<url>" },
      ],
    },
  ],
});
```

### TypeScript — LangChain SDK

```ts
const structuredModel = interfaze.withStructuredOutput({});

const response = await structuredModel.invoke([
  { role: "system", content: "<task>object_detection</task>" },
  {
    role: "user",
    content: [
      { type: "text", text: "Detect objects in this image" },
      { type: "image_url", image_url: { url: "<url>" } },
    ],
  },
]);
```

### Python — OpenAI SDK

```python
response = interfaze.chat.completions.create(
    model="interfaze-beta",
    messages=[
        {"role": "system", "content": "<task>object_detection</task>"},
        {
            "role": "user",
            "content": [
                {"type": "text", "text": "Detect objects in this image"},
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

### Python — LangChain SDK

```python
from langchain_core.messages import SystemMessage, HumanMessage

structured = interfaze.with_structured_output({
    "name": "empty_schema",
    "schema": {"type": "object", "properties": {}, "additionalProperties": True},
})

response = structured.invoke([
    SystemMessage(content="<task>object_detection</task>"),
    HumanMessage(content=[
        {"type": "text", "text": "Detect objects in this image"},
        {"type": "image_url", "image_url": {"url": "<url>"}},
    ]),
])
```
