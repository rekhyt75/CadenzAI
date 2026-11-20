# 10 · Optional: "AI summary" of a ride with a local language model

> **This feature is OPTIONAL and OFF by default.** Do not build it unless the user says yes. Ask once, during the questions of phase 0 (`00-overview.md`, section 4), with the explanation below, and record the answer in `config/profile.yaml` (`ai_summary.enabled`). The rest of the plan works unchanged without it.

## 1. What it is, and what it is not

An **on-demand** button (robot icon, label "AI summary") on the ride-detail page that asks a **language model running on the user's own computer** to write a short summary of that ride: observations, hypotheses and things to look at. It is a convenience that rephrases numbers the app already computed. It is **not** a coach, it does not decide anything, and it must never be the only place where a number appears.

## 2. Ask the user first (what to tell them)

Explain in plain words, then ask "Do you want the optional AI summary? (default: no)":

- The model runs **locally** (Ollama bound to `127.0.0.1`); the ride data **never leaves the computer**.
- One-time **downloads** from the internet: the Ollama program (about 1.5 GB) and the model (about 4.7 GB for the recommended one). No personal data is sent for that.
- It needs **disk space** (about 6.5 GB) and is **slow on a CPU** (about 1 minute per summary on a 16-thread laptop CPU).
- A small model **can be wrong**: that is why the text is labelled, the input numbers can be inspected, and the page's own numbers remain the reference.
- The user can remove it at any time (delete the Ollama folder and the models folder).

If the user declines, skip this whole file and do not show the button.

## 3. Privacy invariants (non-negotiable)

1. The model URL must be a **loopback address** (`127.0.0.1`, `localhost`, `::1`). The code refuses to run otherwise (so a mistaken setting cannot send data to a remote service).
2. The model receives **only aggregated numbers computed by the app**: no GPS track, no ride name, no identifiers, no dates beyond the month. Show the user exactly what was sent ("Show the data given to the model").
3. The summary is generated **only when the button is pressed**, never in the background, and it is cached locally.
4. Ollama listens on `127.0.0.1` only. Never expose it on `0.0.0.0`.
5. The README and the in-app text must state the exceptions to "nothing leaves your computer" truthfully: Garmin calls, map tiles, and the one-time downloads of Ollama and the model.

## 4. Installation (user space, no `sudo`)

- Download the official Linux archive of Ollama from its GitHub releases page (pin the version), into a **new empty folder**; list its contents before extracting (`tar -t`), extract into `~/.local/opt/ollama`, delete the archive. On macOS and Windows use the official installer instead and keep the same settings.
- Store models under `~/.local/share/cadenzai-models` (`OLLAMA_MODELS`).
- Start with `OLLAMA_HOST=127.0.0.1:11434 ollama serve`, then `ollama pull <model>`.
- Integrate with `start.sh` (`07`): `start` launches Ollama detached **only if it is installed and not already running**, stores its PID in `data/ollama.pid`, and `stop` stops it **only if the script started it** (verify the process command line contains `ollama serve`).

## 5. Model choice and measured performance (reference machine: 16 threads, 31 GB RAM, no usable discrete GPU)

| Model | Size | Time per summary (CPU) | Quality in the test |
|---|---|---|---|
| `qwen2.5:3b` | 1.9 GB | about 40 s | too vague; misreads numbers, invents advice |
| `qwen2.5:7b` (**default**) | 4.7 GB | about 75 s | correct observations; hypotheses and advice still generic |

Lessons: the integrated GPU through Vulkan was **3–4 times slower** than the CPU, so pass `num_gpu: 0` and `num_thread` ≈ 3/4 of the cores by default (overridable by environment variables `CADENZAI_LLM_GPU_LAYERS`, `CADENZAI_LLM_THREADS`); prompt processing is the slow part, so keep the input short (about 700 tokens). Let the user choose another model with `CADENZAI_LLM_MODEL`. Re-measure on the user's machine and tell them the real time.

## 6. Backend

- Endpoints: `GET /api/ai/status` (is the server up, is the model installed), `GET /api/ride/{id}/summary` (cached text or `{ "testo": null }`), `POST /api/ride/{id}/summary?force=true|false` (generate or regenerate). A failure returns HTTP 503 with a readable message (the UI shows it).
- Table `ride_ai(activity_id PRIMARY KEY, model, text, facts, created)`: cache of the text **and of the exact facts that were sent**.
- Request to `POST /api/chat` of the local server with `stream: false`, `keep_alive: "10m"`, `temperature: 0.2`, `num_predict: 320`, `num_ctx: 2048`.

## 7. What goes in the prompt

**Facts** (label → value, `N/D` when missing): ride type, month, distance, climbing, total and moving time, average speed, mean power, NP, IF, TSS, reference FTP, VI, mean HR (or "N/D (strap not recorded)"), cadence, work, temperature, aerobic decoupling (or N/D with the reason), share of pedalling time per torque/cadence quadrant, time per power zone, best efforts with their all-time rank, number of climbs and the longest one (distance **and its own** gain), left/right balance, bike, and context: number of rides, mean km and mean TSS in the previous 28 days.

**Deterministic readings**, written by rules, so that the model rephrases instead of interpreting raw numbers: intensity from IF (≥ 0.95 very high, ≥ 0.85 high, ≥ 0.75 medium-high, ≥ 0.55 moderate, else light); regularity of power from VI (≥ 1.3 very irregular, ≥ 1.1 irregular, else regular); cadence (< 70 low, < 90 normal, else high, with the note that low cadence is common on steep MTB climbs); load from TSS (≥ 300 very high, ≥ 200 high, ≥ 100 medium, else low). Each reading is a "Reading" line in the facts.

**System prompt** (in the user's language; Italian in the reference build), with these rules:

```text
You are the assistant of a cycling app. Write in Italian, clear and sober, to a cyclist who knows their own data.
Use ONLY the numbers provided: do not invent values, do not estimate those marked N/D (say they are not available).
The "Lettura" lines are already verified interpretations: rephrase them, do not contradict them and do not reinterpret the raw numbers.
An N/D value only means that it is missing: say so in one sentence and draw no conclusion from it (neither about fitness nor anything else).
Do not give generic advice about tools or training that is not tied to the numbers.
Do not confuse acronyms (VI is the regularity of power, not of cadence; IF is intensity, not a percentage).
Structure: three short parts with bold titles: Osservazioni (2-4 facts from the numbers), Ipotesi (possible explanations, always introduced by "forse", "potrebbe" or "è possibile"; never as certainties), Da guardare (1-2 prudent suggestions; no medical advice).
Maximum 140 words. Do not repeat the list of data.
```

## 8. User interface

- A button with a fun **robot-with-sparkle icon** and the label "AI summary", placed on the ride header row **right after "Next"** (`06`). Slight hover wiggle only if the user has not asked for reduced motion.
- It opens a **modal**: a status line while generating ("the little robot is reading your numbers… it can take 20 seconds to a minute on your computer"), then the text (bold titles rendered), then a line "Written by `<model>` locally on `<date>` in `<n>` s. It is a small model and **can be wrong**: the real numbers are those on the page. Nothing was sent outside the computer."
- Buttons: "Regenerate" and "Show/Hide the data given to the model" (a two-column table of the stored facts; long values wrap).
- Error state: a readable message that says how to start the model server and pull the model.
- The component is keyed by ride id so that the modal closes when the ride changes.

## 9. Acceptance checks

1. With the model server stopped, the button shows a readable error and the rest of the page is unaffected.
2. Setting the URL to a non-loopback host makes the backend refuse to call it (test it).
3. The facts table shown to the user contains **no** GPS coordinates and no ride name.
4. Percentages in the facts are percentages (the quadrant shares are stored as **seconds**: divide by their sum).
5. A ride without heart rate yields "N/D" and the summary draws no conclusion from it (read three generated texts).
6. A second press returns the cached text instantly; "Regenerate" produces a new one.
7. `./start.sh stop` does not stop an Ollama that the user started by hand.

## 10. Honest limits

A 7-billion-parameter model on a CPU gives plausible, sometimes generic text. Treat it as a readable restatement of the numbers, not as analysis. Larger models or an API would write better but would send data outside the computer; if the user insists, that is a different feature with its own explicit consent and must not be built silently on top of this one.
