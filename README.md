# SPLICE — Greeting Deck Generator

A single-page, client-side tool that takes a character card JSON with
multiple greetings (a primary `first_mes` plus `alternate_greetings`)
and forks it into one **standalone card per greeting** — because
SpicyChat (and several other platforms) only ever reads one greeting
per card.

Everything runs in the browser. Your card and your model endpoint are
only ever talked to directly by your browser — there is no backend
and nothing is uploaded to any third-party server other than the
model endpoint you configure yourself.

## What it does

1. **Load** — parse your uploaded JSON and detect its format
   (Character Card V2 / V3 `{ spec, data: {...} }` wrapper, or a flat
   V1-style object). Name, scenario, personality/description,
   example dialogue, and every greeting are read out and shown with
   their current character counts.
2. **Connect** — point it at a KoboldCpp instance (local or a
   Colab/cloudflared tunnel) or anything exposing an
   OpenAI-compatible `/v1/chat/completions` endpoint
   (text-generation-webui, LM Studio, Ollama, etc).
3. **Run** — personality, scenario, and example dialogue are each
   processed **once** and shared across every output card. Each
   greeting (primary + every alternate) is processed on its own.
   For every field:
   - If it's **over** its character limit → the model is asked to
     **condense** it, preserving voice, tone, and every
     plot-critical fact.
   - If it's **under 50%** of its limit → the model is asked to
     **expand** it with plausible in-character detail.
   - Otherwise it's left untouched.
   - If the model's response still comes back over the limit, it's
     hard-truncated at the nearest sentence/word boundary as a
     safety net (this is logged).
   - If the API call fails for any reason, that field falls back to
     its original text (truncated if needed) and the run continues.
4. **Export** — every greeting variant becomes its own JSON file
   (same name, personality, scenario, example dialogue — only the
   greeting differs, and `alternate_greetings` is cleared to `[]`
   since each file is meant to stand alone). All files are zipped
   together for download.

Character limits used (including spaces and line breaks):

| Field            | Limit |
|------------------|------:|
| Greeting         | 2,000 |
| Personality      | 5,000 |
| Scenario         | 4,000 |
| Example dialogue | 4,000 |

## Deploying to GitHub Pages

1. Create a new repository (or use an existing one) and add
   `index.html` from this folder to its root (or to a `/docs`
   folder — your choice).
2. In the repo, go to **Settings → Pages**, set **Source** to the
   branch/folder you put `index.html` in, and save.
3. GitHub will give you a URL like
   `https://yourname.github.io/your-repo/`. Open it — that's the
   live tool. No build step, no dependencies to install.

## Connecting a model

The tool needs *some* LLM to do the condensing/expanding. It talks to
whatever endpoint you give it directly from your browser, so:

### Local KoboldCpp
Run KoboldCpp as usual (`--port 5001` is the default). Leave **API
type** on "KoboldCpp — native API" and **Endpoint URL** as
`http://localhost:5001`.

### KoboldCpp on Colab
Use the usual Colab notebook to launch KoboldCpp with a Cloudflare
tunnel. Copy the `https://xxxx.trycloudflare.com` URL it prints into
**Endpoint URL**.

### text-generation-webui / LM Studio / Ollama / anything OpenAI-style
Switch **API type** to "OpenAI-compatible — chat completions" and
point **Endpoint URL** at the server's base URL (e.g.
`http://localhost:5000` for text-generation-webui,
`http://localhost:11434` for Ollama). Fill in a model name if your
server expects one.

### CORS
Because every request is made directly from your browser, the
endpoint has to allow it. Local servers you run yourself usually work
out of the box or with a `--cors` / `--api` style flag; check your
server's docs if the **Test connection** button in step 2 fails with
a network error even though the server is reachable in another tab.

## Logging

- **Simple** — one line per field: whether it was condensed,
  expanded, or left alone, and the before/after character count.
- **Detailed** — everything Simple shows, plus the exact prompt sent
  to the model and its raw response for every field, and per-call
  timing.

## Notes on format compatibility

The tool tries to be forgiving about field names:

- Personality is read from `personality`; if that's empty it falls
  back to `description` (and writes back to whichever one it read
  from).
- Example dialogue is read from `mes_example`, `example_dialogue`,
  or `mes_examples` — whichever exists.
- Greetings are read from `first_mes` / `first_message` and
  `alternate_greetings` / `alt_greetings`.
- Everything else in the JSON (name, system prompt, post-history
  instructions, character book, tags, extensions, etc.) is copied
  through untouched on every output file.

If your card uses a structure this doesn't recognize, open an issue
or edit the `pickKey(...)` calls near the top of the script in
`index.html` — they're a short, readable list of the field-name
candidates it checks.
