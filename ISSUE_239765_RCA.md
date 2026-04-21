# Root cause analysis — vscode#239765

> Scratch doc. Safe to delete before opening the PR (or lift sections into
> the PR description).

## Symptom

When a chat participant streams a `MarkdownString` that contains inline HTML
with `supportHtml: true`, the chat UI briefly shows the raw markup as
escaped text (e.g. `<span style="color:var(--vscode-editorCodeLens-foreground);">$(info) …`)
before the chunk finishes and the content re-renders correctly.

Repro (reproduces on `1.108.0-insider`):

```ts
const m = new vscode.MarkdownString(
  `<sup><span style="color:var(--vscode-editorCodeLens-foreground);">$(info) Repro</span></sup>\n`,
);
m.supportHtml = true;
m.supportThemeIcons = true;
chatStream.markdown(m);
```

## Call path

1. `chatListRenderer.ts:2988` sets
   `fillInIncompleteTokens = !element.isComplete` for streaming responses.
2. That flag reaches `ChatMarkdownContentPart` and is passed into
   `ChatContentMarkdownRenderer.render(...)`.
3. `ChatContentMarkdownRenderer` wraps the value in `<body>\n\n${value}\n\n</body>`
   (only when `supportHtml: true`) and forwards to
   `IMarkdownRendererService` → `renderMarkdown` in
   `src/vs/base/browser/markdownRenderer.ts`.
4. `renderMarkdown` runs `fillInIncompleteTokens(tokens)` on the lexed
   result, which patches trailing incomplete markdown constructs (tables,
   lists, codespans, `**bold`, links, headings).

## Root cause (two layers)

### Layer 1 — no HTML handler

`fillInIncompleteTokens` has **no branch for unterminated HTML tags**. When
a streamed chunk ends mid-tag (opening `<` without a matching `>`), marked's
inline HTML rule refuses to match because it requires a closing `>`:

```
<[a-zA-Z][\w-]*(?:\s+…)*?\s*\/?>
```

marked falls back and tokenizes the fragment as a `text` subtoken of the
current paragraph. On render, that raw text is escaped (e.g.
`&lt;span style="color:…`) and flashes on screen until the next chunk
delivers the closing `>`.

The analogous markdown-syntax cases (incomplete backticks, `**`, `[`, `|`,
…) don't flash because `fillInIncompleteTokens` already synthesises a
trailing closer during the partial state. HTML has no such handler.

### Layer 2 — the `<body>` wrapper hides the paragraph

`ChatContentMarkdownRenderer` wraps content in `<body>\n\n…\n\n</body>`
before marked sees it (comment in source: the `\n\n` prevents marked from
swallowing the content as a single opaque HTML block). marked's block HTML
rule — type 6 in CommonMark — matches `<body>` as a block tag and consumes
until the first blank line, so the effective token stream is:

```
[html "<body>\n\n", paragraph "<actual content>", space, html "</body>"]
```

The existing patch code keys off `tokens.at(-1)` when deciding which
paragraph/list to fix:

```ts
const lastToken = tokens.at(-1);
if (!newTokens && lastToken?.type === 'paragraph') { … }
```

For chat markdown with `supportHtml: true`, `tokens.at(-1)` is the `</body>`
html block, not the paragraph being streamed. `completeSingleLinePattern`
therefore never runs against the paragraph — which is why even the existing
markdown-token patching can silently miss wrapped chat content. The HTML
fix in this PR has the same dependency.

Verified empirically:

```
input:  '<body>\n\n<sup><span style="color:…;\n\n</body>'
lexer → [html '<body>\n\n', paragraph '<sup><span style="color:…;', space, html '</body>']
```

## Fix

Two coordinated changes in `src/vs/base/browser/markdownRenderer.ts`:

1. **Find the effective last content token.** In
   `fillInIncompleteTokensOnce`, walk back from `tokens.length - 1`
   skipping trailing `html` / `space` tokens to locate the paragraph / list
   / heading to patch. When replacing that token, append the skipped
   trailing tokens so `</body>` (etc.) survives.
2. **Trim unterminated HTML tags.** Add `trimIncompleteHtmlTag(raw)` using
   `/<\/?[a-zA-Z][^<>]*$/`:
   - requires a letter after `<` or `</`, so literal `a < b` / `2 < 3` are
     not mistaken for tags;
   - `[^<>]*$` anchored at end guarantees the tag is genuinely
     unterminated (a later `>` in the same string would break the match).

   `completeSingleLinePattern` calls the helper first. On a hit it re-lexes
   the trimmed prefix and returns the resulting token. When the prefix is
   empty (paragraph was only the incomplete tag), it returns a zero-length
   `space` token so the paragraph collapses — marked's `renderer.space`
   returns `''`.

Both changes are gated on `fillInIncompleteTokens=true`, which is only set
for streaming chat responses (`chatListRenderer.ts:2988`). Hover tooltips,
completed responses, and other `renderMarkdown` callers are untouched.

## Alternatives considered

- **Buffer at the stream layer** (don't hand `<…` to marked until
  balanced). Rejected — requires new state at every `chatStream.markdown`
  call site and duplicates marked's job.
- **Synthesise a closing `>` instead of stripping.** Rejected — closing
  `<span style="color:var(--vscode-editorCodeLens-foreground);` with `>` or
  `">` still leaves a `<span>` wrapping subsequent output until it reaches
  `</span>`, producing mis-styled flashes. Stripping is deterministic and
  keeps render output quiet while the chunk lands.
- **Preprocess the raw string in `ChatContentMarkdownRenderer` before
  wrapping.** Workable, but keeps the "patch incomplete tokens" logic
  split across two files. Putting the walk-back + trim in
  `fillInIncompleteTokens` keeps the concern in one place and means other
  chat renderers that use the same body wrapping automatically benefit.

## Edge cases

| Input (end of stream) | Trimmed? | Resulting last content token |
| --- | --- | --- |
| `hello <sup` | yes | paragraph `hello ` |
| `hello <span style="color:red` | yes | paragraph `hello ` |
| `hello </su` | yes | paragraph `hello ` |
| `<sup` (only content) | yes | dropped (empty `space`) |
| `hello <sup>note</sup>` | no | untouched |
| `hello <sup>not` | no (`>` at `sup>` breaks `[^<>]*$`) | untouched; opener already valid |
| `count is a < b` | no (space after `<`, not a letter) | untouched |
| `<sup><span style="color:…;` | yes | `<sup>` (inner fragment trimmed) |
| `a<b\>b\</b\>c` (`MarkdownString.appendText` output) | no (`\>` → `>` breaks `[^<>]*$`) | untouched — preserves existing test |
| `hello <!-- com` | no (`!` not a letter) | untouched — HTML comments unaffected |
| `<body>\n\n<sup><span …;\n\n</body>` (body-wrapped streaming) | yes | `[…html <body>, <sup>, space, html </body>]` — inner fragment trimmed, wrapper preserved |

Residual: a lone trailing `<` with nothing after is not trimmed (the regex
requires a letter). That is a one-character flash, which is far less
disruptive than the full-tag flash the issue describes. Extending to bare
`<` would also swallow legitimate `a <` at end-of-chunk, so the
conservative match is the intended trade-off.

## Blast radius

- Gated on `fillInIncompleteTokens=true` → only the streaming chat render
  path.
- The walk-back in `fillInIncompleteTokensOnce` keeps the previous
  behaviour whenever the last token is already a paragraph / list / heading
  (trailing html/space skip loop bails on the first iteration), so all
  existing `fillInIncompleteTokens` tests stay green.
- Existing `supportHtml` tests in `markdownRenderer.test.ts` don't use
  `fillInIncompleteTokens=true`, so they're unaffected.

## Tests added

`src/vs/base/test/browser/markdownRenderer.test.ts` → new
`suite('incomplete html tag')` covering every row of the table above,
plus the exact body-wrapped scenario chat actually hits.

## Follow-ups (not in this PR)

- `completeTable` merges `tokens.slice(i)` raw text, which under the
  `<body>` wrapper now also includes trailing `\n\n</body>`. Tables that
  stream mid-separator might therefore be affected by the wrapper too.
  Not observed in the wild yet — flagging for a follow-up if it surfaces.
- Consider trimming a lone trailing `<` when `supportHtml=true` to fully
  eliminate the single-character flash. Left out because the heuristic is
  noisier than the current one.
