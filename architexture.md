# LocalDev — Architecture

Local-first AI code assistant. React 19 + Vite + Tailwind v4 frontend, FastAPI
backend, talking to LM Studio's OpenAI-compatible server (`localhost:1234`). The
workspace folder is read **in the browser** via the File System Access API — the
backend never sees files or paths; any file-aware feature assembles context
client-side and sends it in the request body.

_Last updated: 2026-09-12. Update this doc when a store, route, IndexedDB key, or
the request lifecycle changes._

---

## 1. Shell & routing

```
main.tsx → App.tsx
  ThemeProvider              light/dark → class on <html>, localStorage 'localdev-theme'
    BrowserRouter
      Layout                 nav: / · /editor · /settings
        /          Chat.tsx           main chat (one global transcript)
        /editor    EditorPage.tsx     explorer + tabs + Monaco + per-file chat pane
        /settings  SettingsPage.tsx   token knobs, compression/RAG toggles, clear-data, theme
```

Two independent AI surfaces — the **main chat** and the **per-file editor chat** —
that share a single workspace-folder handle.

---

## 2. Frontend state — the module-store pattern

Cross-navigation state lives in **module-scope stores**: plain module-level vars
and `Map`s outside React, with a `Set<listener>`; components subscribe with
`useSyncExternalStore`. A `fetch` started in a store keeps running after you
navigate away from the page that started it — the result lands in the store (and
IndexedDB) and the page picks it up on remount.

| store | file | holds | persisted to |
|---|---|---|---|
| directory access | `lib/editor/directoryAccess.ts` | the one `FileSystemDirectoryHandle`, a `restorable` handle pending re-grant, `nonce` | IDB `rootHandle` |
| editor chat | `lib/editor/chatStore.ts` | per-file threads `Map<"folder::file", msg[]>`, per-file context `Map<"folder::file", ContextItem[]>`, pending / error / review / compact / abort maps | IDB `editor:chats:<folder>`, `editor:context:<folder>` |
| main chat | `lib/mainChatStore.ts` | one global transcript `msg[]`, pending, error, review, compact state, `AbortController` | IDB `chat:messages` |
| main chat context | `lib/mainChatContext.ts` | attached files `Map<folder, ContextItem[]>` | IDB `chat:context:<folder>` |
| RAG index | `lib/rag/store.ts` | one shared `Bm25Index` (built for whichever folder is open) + build/error state | IDB `rag:chunks:<folder>` (raw chunks only, index rebuilt in memory) |

**Not in stores:**

- Editor buffers / open tabs — `useEditorWorkspace` React state → IDB `editor:files:<folder>`.
- UI preferences — `localStorage` (`lib/editor/storage.ts` + a few scattered keys):
  pane widths, chat-open, chat modes, autosave, theme, sidebar position,
  main-chat mode + edit target, the **token knobs**
  (`localdev-context-window` = 8192, `localdev-response-reserve` = 1024,
  `localdev-max-output-tokens` = 0), and the **context-assembly toggles**
  (`localdev-compression`, `localdev-rag`, `localdev-compact-keep-recent`) —
  see §6/§7/§8.

### Folder scoping

`folderKey = rootHandle.name` — the folder's **basename**, not its path. Two
folders with the same name collide on one set of keys.

`persistence.ts::touchFolder` maintains an LRU list (`editor:folders`, cap 5).
Opening a 6th distinct folder **hard-deletes** `editor:{files,chats,context}:<evicted>`
and `chat:context:<evicted>` from IndexedDB. (The RAG index is not folder-LRU'd —
it's a single in-memory slot that just gets rebuilt for whichever folder is
current; see §7.)

### Hydration

- Editor: `useEditorWorkspace` load-effect → `loadEditorState(folderKey)` →
  `hydrateFolderChats` + `hydrateFolderContext` (re-reads each context file from
  disk — only the path is persisted) + (if RAG enabled) `buildRagIndex`.
- Main chat: `Chat.tsx` mount → `hydrateMainChat()` (migrates the old
  localStorage `chat-history` once) + `hydrateMainChatContext()` + (if RAG
  enabled) its own `buildRagIndex` trigger — main chat can open a folder
  without ever visiting the Editor page, so it needs this independently (§7).
- All hydration guards against clobbering live in-memory state (e.g. a reply that
  streamed in while the page was unmounted).

### ContextItem: manual vs retrieved

```ts
ContextItem { path, content, lineRange?, pinned: boolean, source: 'manual' | 'retrieved' }
```

- `source: 'manual'` — user attached it via the picker; always `pinned`, always persisted.
- `source: 'retrieved'` — emitted by `lib/rag/retrieve.ts` on every send, scored
  against the current folder's BM25 index (editor chat *and* main chat both call
  it — see §7); not persisted unless the user pins it.
- `setRetrievedContext` / `setRetrievedMainChatContext` replace all non-pinned
  retrieved items, keeping manual + pinned.
- `setContextItemPinned` / `setMainChatContextPinned` promote a retrieved item to
  sticky.

---

## 3. Persistence map

**IndexedDB** — DB `localdev`, store `kv`, via `lib/idb.ts` (`idbGet/idbSet/idbKeys/idbDelete`):

```
rootHandle                    the FileSystemDirectoryHandle
editor:folders                LRU list of folder names (5-cap)
editor:files:<folder>         { openFiles: OpenFile[], activePath }
editor:chats:<folder>         { [filePath]: EditorChatMessage[] }
editor:context:<folder>       { [filePath]: PersistedContextItem[] }   (path-only)
chat:messages                 EditorChatMessage[]  (main transcript)
chat:context:<folder>         PersistedContextItem[]                   (path-only)
rag:chunks:<folder>           Chunk[]  ({path, lineRange, content}) — index rebuilt from these on load
compaction:log                CompactionLogEntry[]  — one shared array, both chats
```

**localStorage** — `localdev-theme`, `localdev-sidebar-position`,
`localdev-editor-explorer-width`, `localdev-editor-chat-width`,
`localdev-editor-chat-open`, `localdev-editor-chat-mode`,
`localdev-editor-auto-save`, `localdev-main-chat-mode`,
`localdev-main-chat-edit-target`, `localdev-context-window`,
`localdev-response-reserve`, `localdev-max-output-tokens`,
`localdev-compression`, `localdev-rag`, `localdev-compact-keep-recent`.

**Backend disk** — `backend/stats.json`, `backend/calibration.jsonl` (both git-ignored).

---

## 4. Request lifecycle — frontend

```
user hits Send
  │
  ▼
useEditorChat.send()  /  Chat.tsx send()
  │   assembles { history, anchor, mode, contextItems, maxOutputTokens }
  ▼
RAG retrieval (§7) — refreshes contextItems' retrieved slice for this message
  │   editor chat: inside runEditorChat (contextByKey is read internally)
  │   main chat:   inside Chat.tsx::send(), BEFORE building runMainChat's args —
  │                mainChatStore takes contextItems as a plain argument from its
  │                caller rather than reading its own map, so retrieval has to
  │                land here, with a fresh re-read after, or the store gets a
  │                stale pre-retrieval snapshot
  ▼
runEditorChat()  /  runMainChat()          [module store — survives navigation]
  │
  ▼
buildChatRequest(input)                     lib/buildChatRequest.ts — pure, no I/O
  •  compressHistory(history, compression)   §8 — no-op when off, byte-identical request
  •  anchor present → endpoint 'editor-chat'; else 'chat'
  •  filter the anchor out of contextItems
  •  sort context: manual first, then retrieved
  •  planContext(...) → trim history to the token budget
  •  → { endpoint, body: {messages, file_content?, file_path?, mode?, max_tokens, context}, plan }
  │
  ▼
fetch(`${API}/${endpoint}`, body)  →  NDJSON stream
  │   reader loop, one JSON object per line:
  │     {type:'start', mode}
  │     {type:'delta', text}         (repeated — store patch throttled to 40ms)
  │     {type:'done',  mode, code?}
  │     {type:'error', detail}
  ▼
on 'done' with edit code → stage a review
  editor: DiffEditor in EditorSurface        accept → applyEditToBuffer (not disk)
  main:   MainChatReviewModal                 accept → writeFile() straight to disk
```

`anchor` = the file treated as the editable subject:

- editor chat: **always** `{ path: activePath, content: activeFile.content }`
- main chat: only in Edit mode with a target selected, else `null` (plain `/chat`)

---

## 5. Request lifecycle — backend

```
POST /chat  ┐
POST /editor-chat  ┴→  main.py route handler
  │
  ├─ (editor-chat) resolve mode:
  │     request.mode ∈ {chat, edit} ? force it
  │     : mode_detection.detect_mode(latest_user_msg)     clause-by-clause regex; ambiguous → chat
  │
  ├─ build the system prompt:
  │     _format_context(request.context)      "--- path ---" fenced blocks;
  │                                            retrieved ones tagged "(retrieved excerpt, may be partial)"
  │   + template:
  │       /chat                     → CHAT_SYSTEM_PROMPT
  │       /editor-chat  chat mode   → _CHAT_MODE_PROMPT  + _number_lines(file)   ("N | " prefix per line)
  │       /editor-chat  edit mode   → _EDIT_MODE_PROMPT  + raw file
  │
  ├─ messages = [ {system}, *history ]
  │
  ▼
llm.stream_lm(messages, temperature, max_tokens)                       llm.py
  •  POST LM Studio /v1/chat/completions   (stream=True, stream_options.include_usage=true)
  •  yield ("delta", text)  per content chunk
  •  time the decode locally (LM Studio omits its `stats` block while streaming)
  •  yield ("usage", { model, prompt_tokens, completion_tokens, prefill_seconds, decode_seconds, ... })
  •  yield ("done", full_text)
  │
  ▼
route's events() generator re-emits as NDJSON:
  {type:'start', mode}  →  {type:'delta'}...  →  {type:'done', mode, code}
     code = extract_code_block(answer) for edit mode, else null
  on ("usage"):
     stats.record(usage)                     running tok/s avg + token totals → stats.json
     calibration.record(messages, usage)     (predicted chars/3.8+4n, actual prompt_tokens) → calibration.jsonl
  │
  ▼
StreamingResponse(media_type="application/x-ndjson")
```

Other endpoints: `POST /edit` (non-streaming, legacy full-file rewrite),
`POST /summarize` (streaming NDJSON, backs Compact — §8; usage NOT recorded to
`stats`/`calibration`, different kind of traffic), `GET /stats`, `POST /stats/reset`.

---

## 6. Token budgeting

`lib/tokens.ts` — deliberately approximate, no real tokenizer.

- `estimateTokens(text) = ceil(text.length / 3.8)`
- `planContext({ messages, fileContent?, fileLineNumbered?, contextItems?, ... })`:
  - `budget = window − responseReserve − overhead − fileTokens − contextTokens`
  - walk messages newest-first, keep what fits; the latest message is always kept
  - returns `{ window, fileTokens, contextTokens, historyTokens, promptTokens,
    droppedCount, sentMessages, fileTooBig, contextTooBig }`
- `TokenMeter` renders `≈ promptTokens / window · file X · context Y` from the same call.
- Three user knobs (Settings): context window (8192), response reserve (1024),
  max output tokens (0 = uncapped). The window is a **frontend guess** — the app
  does not ask LM Studio for the loaded model's real context length.

### Calibration

`backend/calibration.py` logs `(predicted, actual prompt_tokens)` per request.
`predicted` comes from `chars/3.8 + 4×n_messages` applied to the **exact
assembled prompt** sent to `stream_lm` — system prompt, `_format_context`
blocks, line-numbering where applicable, full history — so `predicted` and
`actual` (LM Studio's real `usage.prompt_tokens`) cover identical content, not a
partial view. `bench/analyze_calibration.py` runs 5-fold CV (rows shuffled),
fits a multiplicative correction two ways (median-of-ratios, least squares), and
reports held-out error — global and split by model / mode. Passive measurement,
not wired into the app.

Findings (n=69 real generations, current sample): chat-mode baseline is already
well-calibrated (~7.4% error, R²=0.994) and barely improves under correction;
edit-mode is not (~12% error, R²=0.908) — a per-mode correction
(median-of-ratios, k≈0.885) brings edit-mode held-out error to ~2.6%, and this
factor has reproduced independently across two separate data collections.
Least-squares gives a tighter fit for chat/blended but is more sensitive to
edit-mode's smaller, higher-variance sample — median-of-ratios wins there
instead. This `k` has not been applied back to `estimateTokens`/`compact.ts`'s
estimates (§8) — it was fit against a different formula (includes system
prompt + `+4×n_messages`) and content mix than those use, so it isn't a
validated correction for them without separate measurement.

---

## 7. Retrieval (RAG)

Lexical (BM25), not embeddings — no dependency, no extra LLM call, fully
offline. **Editor chat and main chat both use it; main chat stays otherwise
workspace-unaware** (no per-file anchor concept). All new code under `lib/rag/`:

| module | job |
|---|---|
| `tokenize.ts` | shared tokenizer for both indexing and querying — lowercase, splits camelCase/snake_case/kebab-case boundaries, drops single-char tokens and a conversational-English stopword list |
| `chunk.ts` | `chunkFile`/`chunkFiles` — whole file ≤120 lines, else 60-line sliding windows w/ 15-line overlap; every chunk always carries `lineRange` (even whole-file ones), matching `ContextItem.lineRange`'s "present = retrieved" contract |
| `bm25.ts` | `buildBm25Index`/`scoreBm25` — standard `k1=1.2, b=0.75`, Lucene-style non-negative idf smoothing. A chunk's **path** is folded into its term-frequency table (repeated `PATH_WEIGHT=3`×) alongside content, so a query naming a file/extension matches even a near-empty target |
| `store.ts` | module-scope store (§2) — `buildRagIndex(folderKey, root)` walks (`walkWorkspaceFiles`) + reads (`readFile`) + chunks + builds the index, persists only the raw chunks to IDB (always rebuilds fresh from disk on open — no staleness tracking beyond the next item); `invalidateFile(folderKey, path, content)` re-chunks one saved file and patches the persisted+in-memory index without a full re-walk, wired into `saveActiveFile`/`saveAllFiles` |
| `retrieve.ts` | `retrieveContext(index, query, opts)` — scores, takes best-first chunks up to a `topK` (8) or token-budget (1000, independent of `planContext`'s budget) cap, excludes the edit anchor/target and already-manually-attached paths (not pinned-retrieved ones — a different chunk of the same file may still help) |

Gated by `lib/context/settings.ts::getRagEnabled()` (`localdev-rag`, default off
= no background walk/read/attach at all). Settings page: toggle + Index Status
(file/chunk counts, or the last build error) + manual **Reindex** button.

**Real bug found via usage, fixed**: asking about "the two java files" scored
those files **zero** and excluded them — chunk *content* rarely contains its own
filename/extension, and the query's own filler words ("can you... to the... in
my...") coincidentally out-scored the one real signal word ("java") against an
unrelated prose file. Fixed by indexing chunk paths (above) and adding the
stopword list to `tokenize.ts`; a residual, honest limitation remains — a
sufficiently prose-heavy unrelated file can still occasionally out-*rank* the
real target on a generic query, even though the target is now correctly
*included*. Not further patched (tried raising `PATH_WEIGHT`, hit BM25's
built-in term-frequency saturation with no real gain) — embeddings would be the
actual fix if this turns out to matter in practice.

---

## 8. Context compression & Compact

Two independent techniques, both gated by `lib/context/settings.ts`, both
no-ops when off (byte-identical request to the pre-feature app — the ablation
baseline).

**Compression** (`lib/context/compress.ts::compressHistory`) — collapses a
resolved edit-mode assistant turn (has `.code`, isn't the newest message) to a
short placeholder; never touches the anchor or the newest message. Runs inside
`buildChatRequest`, before `planContext` (§4/§6). `getCompressionEnabled()` /
`localStorage 'localdev-compression'`.

*(A second technique — skeleton/signature-only reference files — was built,
then deliberately reverted: accurate per-language signature extraction needed
enough special-casing for an ambiguous payoff next to this + Compact.)*

**Compact** (`lib/context/compact.ts`) — on-demand, user-triggered,
`/compact`-style summarization; not automatic (an earlier "automatic rolling
summary" design was dropped — its savings were ambiguous because they'd
interact with `planContext`'s dynamic budget refill). `planCompaction(history,
keepRecent)` keeps the last `keepRecent` messages raw (`getKeepRecentCount()`,
default 2, Settings-configurable) and detects/merges an existing summary
(`EditorChatMessage.isSummary`) so repeated compactions never stack more than
one summary block. `POST /summarize` (streaming NDJSON, reuses `stream_lm`)
generates the new summary, shown live under a "Compacting…" indicator.
`buildCompactedThread` splices the result into the **stored** thread — folded
messages disappear from both what's sent and what's persisted, not just what's
sent. This is destructive and irreversible by design (no archive of the
originals kept) — a deliberate, accepted tradeoff for a single-user local tool
where the action is manually triggered, not automatic. Wired into both
`chatStore.ts::compactFileChat` and `mainChatStore.ts::compactMainChat`, same
pending/abort/error pattern as a regular send.

**Metrics** — `lib/context/compactionLog.ts`, one IDB array (`compaction:log`)
shared across both chats (chosen over a workspace-folder file because main-chat
compaction works with no folder open at all). `logCompactionEvent` **must be
`await`ed**, not fire-and-forget — a `void`-ed call let two back-to-back
compactions race on the read-modify-write and silently drop an entry.
`summarizeCompactionLog` reports `count`, `totalTokensSaved` (Σ before−after,
can be negative — deliberately not paired with raw before/after averages, which
don't divide back into the ratio below), and `meanRatio`/`medianRatio` — each
entry's own before÷after ratio, *then* averaged, **not** total-before ÷
total-after (a different, larger statistic). Since both sides of that ratio use
the identical `chars/3.8` estimate, it cancels out algebraically — the ratio is
independent of that estimate's accuracy; only `totalTokensSaved` (and the
absolute before/after counts) carry it. Settings page: Compaction Stats panel
(Refresh / Clear log), tooltip on every tile stating exactly what it is.

---

## 9. Measurement / logging modules

| module | job | surface |
|---|---|---|
| `backend/stats.py` | tok/s running avg, first-token latency, token totals — global + per model | `GET /stats`, `stats.json` — no UI |
| `backend/calibration.py` | token-estimate accuracy log | `calibration.jsonl` → `bench/analyze_calibration.py` — no UI |
| `backend/mode_detection.py` | CHAT vs EDIT classifier for `/editor-chat` auto mode; clause-by-clause regex, ambiguity → CHAT | return value only — no UI |
| `lib/context/compactionLog.ts` | durable log of real Compact usage | Settings → Compaction Stats |
| `lib/rag/store.ts` | current RAG index's file/chunk counts + build/error state | Settings → Retrieval (RAG) → Index Status |

---

## 10. Extension points

| planned feature | where it hooks in |
|---|---|
| **multi-file-edit workflow** | new backend endpoint (`/multi-edit`) — an internal planning call (sees candidate file *paths* only, cheap) dispatches one focused `call_lm`/`stream_lm` per target file, streaming NDJSON events (`plan`, `file_start`, `file_delta`, `file_done`) over one connection. Deliberately backend-owned rather than orchestrated from the frontend stores, to avoid a second multi-step state machine alongside chat/compact in `chatStore`/`mainChatStore`. Frontend: main chat shows a human-readable plan summary (not the literal per-file dispatch instructions), then an N-file review manager generalizing today's single-file `review`/`reviewByFile` state — independent accept/reject per file, one Monaco diff editor with a file-switcher rather than N live instances, a build-log-style progress list while generating. Known tradeoff, accepted for v1: the whole batch lives only inside one HTTP connection — a drop mid-flight loses the in-flight files, no partial resume |
| **benchmark harness** | `bench/run_benchmark.py` calls `buildChatRequest` directly, POSTs, logs `usage.prompt_tokens` + a per-task pass check — same JSONL + offline-analysis pattern as `calibration.py`. Ablation: baseline / +retrieval / +compression / +both, median of N runs. Frozen `tasks.json`, single-shot/single-file scope only |
| **retrieval eval** | a hand-labeled `{query, relevantPaths}` set + `retrieveContext` (§7) scored against it — hit-rate/precision/recall@K, no LLM call needed (pure retrieval math). Not built; cheap whenever wanted |
| **reasoning / agent loop** | a fixed small set of step types (Reason → Edit → Chat), sequential only (no branching) for v1; a real workflow schema (typed node inputs/outputs) is the design work to do before any UI. Deliberately last on the roadmap |
| **main-chat RAG polish, retry/re-roll, revert-applied-edit, `/v1/models` picker** | rest of the originally-scoped "Phase 5" — main-chat retrieval itself is done (§7); these are the still-open pieces |

---

## 11. Conventions

- **Module stores over React context** for anything that must survive SPA navigation.
- **Path-only persistence** for context items — content is re-read from disk on
  hydrate, so it can't go stale.
- **NDJSON** (`{type, ...}` per line) for all streaming, `media_type
  application/x-ndjson`.
- **Ground-truth tokens** come from LM Studio `usage.prompt_tokens`; the
  `chars/3.8` heuristic is only for the pre-send meter, history-trim decisions,
  and compaction's before/after accounting — never assume a calibration factor
  fit for one of these transfers to another without measuring it there too (§6).
- **Retrieval and compression are pre-processing steps applied by the callers of
  `buildChatRequest`** (the stores/pages assemble `contextItems`/`history`
  before calling it), not branches inside `buildChatRequest` itself — it stays a
  pure function of whatever it's handed, so a benchmark harness can drive it
  with an explicit config instead of real browser storage.
- **A destructive, user-triggered action (Compact) needs an explicit click, not
  an automatic trigger** — the manual step is the minimum viable consent for an
  irreversible operation.
- Backend stays declarative: route handlers in `main.py`, all LM I/O + parsing in
  `llm.py`, classification in `mode_detection.py`.
