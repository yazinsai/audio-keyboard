# Self-hosting the AI server

My Board can talk to any server that speaks the same request shapes as Venice AI. Nothing here
is specific to one implementation: Speaches, llama.cpp's server, or a small proxy of your own all
qualify if they answer the four calls below.

Set it up in **Settings → AI service → My server**. There are two addresses, because dictation and
writing are separate calls to separate endpoints, and they may live on different hosts.

| field | used for | app appends |
|---|---|---|
| **Voice typing server** | dictation (`/v1/audio/transcriptions`) | that path |
| **Chat server** | ✦, waiting replies, chat summaries (`/v1/chat/completions`) | that path |
| **API key** | both | `Authorization: Bearer <key>` |

Both fields take a **bare root** — `https://mybox.example:8080`, not `.../v1/chat/completions`.
The app builds the paths itself. A blank key sends no `Authorization` header at all.

Plain `http://` is allowed, and the settings screen warns when the address is neither `https` nor a
private network. Audio and message text go wherever you point these two fields.

## The four calls

| call | when | timeout |
|---|---|---|
| `POST {chat}/v1/chat/completions` | every ✦, reply draft and chat summary | 3 s connect, 30 s read |
| `GET {chat}/v1/models` | when you open the model picker | 5 s |
| `POST {voice}/v1/audio/transcriptions` | every dictation chunk | 3 s connect, 10 s + 2× the clip length |
| `GET {voice}/v1/models` | when you open the speech-model picker | 5 s |

`/v1/models` only fills the pickers. If it fails, the app falls back to "Automatic" and still works.

## Writing: `POST /v1/chat/completions`

```json
{
  "model": "your-model-id",
  "messages": [
    {"role": "system", "content": "…the project's prompt…"},
    {"role": "user", "content": "…the instruction, the visible screen and style samples…"}
  ],
  "max_tokens": 1024,
  "stream": false,
  "chat_template_kwargs": {"enable_thinking": false},
  "response_format": {
    "type": "json_schema",
    "json_schema": {
      "name": "field_text",
      "strict": true,
      "schema": {
        "type": "object",
        "properties": {"text": {"type": "string"}},
        "required": ["text"],
        "additionalProperties": false
      }
    }
  }
}
```

- `model` is the id chosen in settings, or `""` when it is left on "Automatic" — serve your default.
- `chat_template_kwargs` is llama.cpp's way to turn thinking off; servers that do not know the field
  should ignore it rather than reject the request.
- `response_format` is how the reply is forced into JSON. llama.cpp compiles it to a grammar; a server
  without structured output can ignore it and simply return JSON.

The reply is standard OpenAI-shaped, and only `choices[0].message.content` is read:

```json
{"choices": [{"message": {"role": "assistant", "content": "{\"text\": \"See you at 7.\"}"}}]}
```

`content` must contain a JSON object with a `text` string. Text around the object is tolerated — the
first `{` to the last `}` is parsed — so `"Here you go: {\"text\": …}"` works too. Anything without a
`text` field fails with *"The AI server didn't answer in the JSON format"*.

A `reasoning_content` field is ignored, so models that think first are fine as long as the answer
arrives in `content`.

## Dictation: `POST /v1/audio/transcriptions`

`multipart/form-data`, with a 16 kHz mono WAV in `file`:

| field | value |
|---|---|
| `model` | the speech model chosen in settings, or `""` for the server's default |
| `language` | the language being dictated (`ar`, `en`); omitted when several may be spoken, so the server detects |
| `response_format` | `verbose_json` |
| `temperature` | `0` |
| `prompt` | optional context: the last 200 characters before the cursor, so names and spellings carry over |
| `hotwords` | optional comma-separated words from the personal dictionary |

`verbose_json` is what makes the app's confidence check possible, so prefer it:

```json
{
  "text": "see you at seven",
  "segments": [
    {"text": " see you at seven", "avg_logprob": -0.31, "no_speech_prob": 0.02}
  ]
}
```

A reply with only `text` is accepted. With `segments`, the app drops a chunk when `no_speech_prob`
is high and `avg_logprob` is low, and uses the mean `avg_logprob` to choose between two language
passes — that is the only reason those fields matter.

`408`, `429` and `5xx` are retried later with the audio kept; any other `4xx` is shown to the user with
the server's own message (`error.message`, `detail`, or `error` as a string are all understood).

## What the app does not do

- **No extra headers.** `Authorization: Bearer <key>` is the only one it sends, so a server that needs
  another header (a session id, an API-version, a custom key name) has to sit behind a proxy that adds it.
- **No streaming.** `stream` is always `false`; a streamed reply would be read as one body.
- **No retries on writing.** A failed ✦ is shown with Retry in the orb; there is no background retry.
- **A single key for both services.** Two separate keys mean one proxy in front of both, or the same
  key accepted by both.

## Checking it works

**Settings → AI service** has a **Test connection** row under each address. For writing it makes one
short request and shows the reply; for dictation it transcribes a bundled English and Arabic clip.
Both report the server's own error text, which is the fastest way to find a wrong path or a rejected
key.
