# Repro & verify — vscode#239765

> Scratch doc for local verification. Safe to delete before opening the PR.

## What's fixed

When a chat participant streams a `MarkdownString` with `supportHtml: true`, a
chunk could cut in the middle of an HTML tag (e.g. `<span style="color:`).
`marked` tokenizes that trailing fragment as text, so the chat UI briefly
shows the escaped raw markup until the next chunk arrives and the tag closes.

This branch extends `fillInIncompleteTokens` in
`src/vs/base/browser/markdownRenderer.ts` so that when the last paragraph's
raw ends with an unterminated HTML tag (`<tag…` with no `>`), we strip that
fragment and re-lex. The fix only kicks in when `fillInIncompleteTokens=true`,
which is only set for streaming chat responses — finalised content paths are
untouched.

## Repro (extension)

1. Build and launch this branch.
2. Create a minimal chat participant extension following
   <https://code.visualstudio.com/api/extension-guides/ai/chat#create-a-chat-participant>.
3. In the handler, stream markdown that contains HTML:

   ```ts
   const message = new vscode.MarkdownString(
     `<sup><span style="color:var(--vscode-editorCodeLens-foreground);">$(info) Repro</span></sup>\n`,
   );
   message.supportHtml = true;
   message.supportThemeIcons = true;
   chatStream.markdown(message);
   ```

4. Mention the participant in the Chat view via its `@` handle.

### Before the fix

A raw string (e.g. `<span style="color:var(--vscode-editorCodeLens-foreground);">$(info) Repro`)
flashes on screen, then gets replaced by the formatted HTML. See the GIF on
the original issue.

### After the fix

Content appears formatted (or invisible while buffering); the raw
`<span …>` fragment no longer flashes.

## Repro (unit tests)

Without leaving the repo:

```bash
# one-time
npm install

# run the added tests
npm run test-browser-no-install -- --run src/vs/base/test/browser/markdownRenderer.test.ts --grep "incomplete html tag"
```

The new `incomplete html tag` suite in
`src/vs/base/test/browser/markdownRenderer.test.ts` covers:

- trailing `<sup` / `</su` / `<span style="color:red` gets stripped
- paragraph that's **only** an incomplete tag collapses to empty output
- complete tags, partially-open tags with body, and literal `a < b` are left
  untouched
- the exact issue's nested case (`<sup><span style="color:…;`) collapses to
  just `<sup>` until more arrives

## Files touched

- `src/vs/base/browser/markdownRenderer.ts` — fix + helper
- `src/vs/base/test/browser/markdownRenderer.test.ts` — new test suite
- `ISSUE_239765_REPRO.md` — this file (delete before PR)

## Notes for the PR description

- Reference `Fixes #239765` in the body.
- Label `help wanted` was already on the issue; assignee is `@mjbvz`.
- Consider leaving a short comment on the issue first mentioning the planned
  approach so the maintainer can redirect if they had something else in mind.
