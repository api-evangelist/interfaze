---
name: speech-to-text
description: >
  Use Interfaze AI for speech-to-text transcription from audio files, voice notes,
  recordings, podcasts, and meeting audio. Activate when the user wants to transcribe
  audio to text, identify speakers, detect language, or extract timestamps — even if
  they say "what did they say in this recording" or "convert this audio to text"
  without explicitly mentioning transcription.
---

# STT — Speech-to-Text with Interfaze AI

Transcribe audio files into text with speaker diarization, timestamps, translation, and audio analysis using Interfaze AI.

## When to use this skill

- The user has an audio file and wants it transcribed to text
- The input is a voice note, meeting recording, podcast, interview, or phone call
- The user wants speaker identification or diarization from an audio source
- The user asks for timestamps alongside the transcript
- The user says "transcribe this", "what does this audio say", "convert this recording to text"
- The user wants to translate audio between languages

## When not to use this skill

- The input is an image or screenshot → use `ocr`
- The user wants to generate speech from text (text-to-speech) — not supported
- The input is already text that needs reformatting — no transcription needed
- The user wants a video's visual content → use `ocr` or `object-detection`

## Workflow

1. Confirm the input is an audio file URL or file reference.
2. Determine what the user needs: plain transcript, speaker-labeled transcript, timestamps, or translation.
3. Build a schema matching the desired output (Zod for TypeScript, Pydantic for Python).
4. Pass the audio in the message `content` array. A URL can also be passed inline in the prompt text (slightly faster).

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

Model name is always `interfaze-beta`. Sources: public HTTPS URLs and base64 data.

| SDK | Audio part |
| --- | --- |
| Vercel AI SDK | `{ type: "file", data: "<url>", mediaType: "audio/mpeg" }` |
| OpenAI / LangChain (TS & Py) | `{ type: "file", file: { filename: "audio.mp3", file_data: "<url-or-base64>" } }` |

You can also pass the URL inline in the prompt across all SDKs — `{ role: "user", content: "Transcribe https://example.com/audio.mp3" }` — which is slightly faster than a file part.

MIME types: MP3 → `audio/mpeg`, WAV → `audio/wav`, M4A → `audio/mp4`. Other formats use their canonical IANA MIME type.

## Example: basic transcription

### TypeScript — OpenAI SDK

```ts
import { z } from "zod";
import { zodResponseFormat } from "openai/helpers/zod";

const STTSchema = z.object({ text: z.string() });

const response = await interfaze.chat.completions.create({
  model: "interfaze-beta",
  messages: [
    {
      role: "user",
      content: [
        { type: "text", text: "Transcribe the audio file" },
        { type: "file", file: { filename: "voice-note.mp3", file_data: "https://example.com/voice-note.mp3" } },
      ],
    },
  ],
  response_format: zodResponseFormat(STTSchema, "stt_schema"),
});

console.log(response.choices[0].message.content);
```

### TypeScript — Vercel AI SDK

```ts
import { generateObject } from "ai";
import { z } from "zod";

const STTSchema = z.object({ text: z.string() });

const { object } = await generateObject({
  model: interfaze.chat("interfaze-beta"),
  schema: STTSchema,
  messages: [
    {
      role: "user",
      content: [
        { type: "text", text: "Transcribe the audio file" },
        { type: "file", data: "https://example.com/voice-note.mp3", mediaType: "audio/mpeg" },
      ],
    },
  ],
});

console.log(object);
```

### TypeScript — LangChain SDK

```ts
import { z } from "zod";

const STTSchema = z.object({ text: z.string() });

const structuredModel = interfaze.withStructuredOutput(STTSchema);

const response = await structuredModel.invoke([
  {
    role: "user",
    content: [
      { type: "text", text: "Transcribe the audio file" },
      { type: "file", file: { filename: "voice-note.mp3", file_data: "https://example.com/voice-note.mp3" } },
    ],
  },
]);

console.log(response);
```

### Python — OpenAI SDK

```python
from pydantic import BaseModel

class STTSchema(BaseModel):
    text: str

response = interfaze.chat.completions.create(
    model="interfaze-beta",
    messages=[
        {
            "role": "user",
            "content": [
                {"type": "text", "text": "Transcribe the audio file"},
                {"type": "file", "file": {"filename": "voice-note.mp3", "file_data": "https://example.com/voice-note.mp3"}},
            ],
        }
    ],
    response_format={
        "type": "json_schema",
        "json_schema": {"name": "stt_schema", "schema": STTSchema.model_json_schema()},
    },
)

print(response.choices[0].message.content)
```

### Python — LangChain SDK

```python
from langchain_core.messages import HumanMessage
from pydantic import BaseModel

class STTSchema(BaseModel):
    text: str

structured_llm = interfaze.with_structured_output(STTSchema)

response = structured_llm.invoke([
    HumanMessage(content=[
        {"type": "text", "text": "Transcribe the audio file"},
        {"type": "file", "file": {"filename": "voice-note.mp3", "file_data": "https://example.com/voice-note.mp3"}},
    ])
])

print(response)
```

## Example: speaker diarization

Same call as basic transcription — swap in this schema and prompt `"Transcribe and identify the speakers in the audio file"`. `speaker_id` is a string; up to 50 speakers are supported. Add analysis fields per chunk if needed, e.g. `sentiment: z.enum(["positive", "negative", "neutral"])`.

```ts
const DiarizationSchema = z.object({
  full_text: z.string(),
  chunks: z.array(
    z.object({
      speaker_id: z.string(),
      text: z.string(),
      start_time: z.number(),
      end_time: z.number(),
    })
  ),
  number_of_speakers: z.number(),
});
```
```python
class Chunk(BaseModel):
    speaker_id: str
    text: str
    start_time: float
    end_time: float

class DiarizationSchema(BaseModel):
    full_text: str
    chunks: list[Chunk]
    number_of_speakers: int
```

## Example: audio translation

Include the target language in the prompt (e.g. "translate to Chinese") and use this schema:

```ts
const TranslationSchema = z.object({
  translated_text: z.string().describe("translated text"),
  original_language_code: z.string(),
  translated_language_code: z.string(),
});
```
```python
class TranslationSchema(BaseModel):
    translated_text: str = Field(..., description="translated text")
    original_language_code: str
    translated_language_code: str
```

## Plain transcription (no schema)

Use `generateText` (Vercel AI SDK) or omit `response_format` (OpenAI / LangChain) when you only need raw text.

## Tips

- Use canonical MIME types: `audio/mpeg` (MP3), `audio/wav` (WAV), `audio/mp4` (M4A).
- Include "identify the speakers" in the prompt to trigger diarization.
- Specify the target language in the prompt for translation.

## Advanced: long audio & word-level timestamps

For long audio (1hr+) or lowest cost/latency, run as a single task with `<task>speech_to_text</task>` in the system message. The response contains `text` and `chunks: [{ timestamp: [start, end], text }]`.

### TypeScript — OpenAI SDK

```ts
const response = await interfaze.chat.completions.create({
  model: "interfaze-beta",
  messages: [
    { role: "system", content: "<task>speech_to_text</task>" },
    { role: "user", content: [{ type: "text", text: "Transcribe https://example.com/audio.mp3" }] },
  ],
  response_format: zodResponseFormat(z.any(), "empty_schema"),
});
```

### TypeScript — Vercel AI SDK

```ts
const { object } = await generateObject({
  model: interfaze.chat("interfaze-beta"),
  system: "<task>speech_to_text</task>",
  schema: z.any(),
  messages: [
    { role: "user", content: [{ type: "text", text: "Transcribe https://example.com/audio.mp3" }] },
  ],
});
```

### Python — OpenAI SDK

```python
response = interfaze.chat.completions.create(
    model="interfaze-beta",
    messages=[
        {"role": "system", "content": "<task>speech_to_text</task>"},
        {"role": "user", "content": [{"type": "text", "text": "Transcribe https://example.com/audio.mp3"}]},
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

**Precontext:** `precontext` on the response (`response.precontext` for OpenAI SDK / Python, `response.body?.precontext` for Vercel AI SDK) contains `name: "stt"` (raw transcript with `chunks[].timestamp`) and, when translation is requested, `name: "translate"` (`source_language`, `target_language`, `translated_text`).
