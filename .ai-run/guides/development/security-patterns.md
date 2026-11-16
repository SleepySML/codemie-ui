# Secure frontend patterns

Index: [`../security/README.md`](../security/README.md). This file owns the application-level rules
— HTML sinks, navigation from untrusted input, cross-window messages, what is and is not secret in a
browser build, secrets in the repo, and suppression hygiene.

This is about writing secure code. Remediating a reported finding starts at the index.

## Overview

Untrusted input reaches the DOM through HTML sinks, URLs and redirects. The code sanitizes HTML with DOMPurify, allows images only from an allow list, normalizes agent-supplied URLs before rendering A2UI media, and strips protocol-relative paths from stored and hash redirects.

| Part | What it holds | Start here |
|---|---|---|
| `src/utils/postLoginRedirect.ts` | return-URL storage with open-redirect checks | `src/utils/postLoginRedirect.ts::sanitizePostLoginRedirect` |
| `src/utils/redirectHashRoutes.ts` | rewrite of legacy `#/` URLs | `src/utils/redirectHashRoutes.ts::redirectHashRoutes` |
| `src/utils/imageAllowList.ts` | which image sources may load | `src/utils/imageAllowList.ts::isImageAllowed` |
| `src/a2ui/utils.ts` | agent URL sanitizing | `src/a2ui/utils.ts::sanitizeAgentUrl` |

## HTML sinks

```bash
grep -rn "dangerouslySetInnerHTML\|\.innerHTML" src --include='*.ts' --include='*.tsx'
grep -rn "DOMPurify" src --include='*.ts' --include='*.tsx'
```

Three shapes exist in `src/` today, and the difference between them is the whole rule:

| Shape | Example | Safe because |
|---|---|---|
| Sanitize inline at the sink | `src/pages/chat/components/ChatHistory/ChatUserMessage/ChatUserMessage.tsx::ChatUserMessage` | The `DOMPurify.sanitize` call is on the same line as the render |
| Sanitize at the producer, pass HTML down | `src/components/markdown/MarkdownTokens.tsx` → `tokens/TableBlock.tsx` | `MarkdownTokens` sanitizes, `TableBlock` renders its `html` prop untouched |
| Render a constant | the `gradientSvg` sinks under `src/pages/workflows/`, the close button in `src/utils/toaster.ts` | The value is an imported SVG asset or a literal in the same file, never request data |

**`TableBlock` is a trusting sink.** It takes `html: string` and renders it as-is; its safety lives
entirely in its caller. Any new caller must sanitize first. The same holds for any component you
add with an HTML-typed prop — if the component cannot sanitize, its type is a contract that the
caller does, and that contract must be stated where the prop is declared.

Never introduce a fourth shape. A sink whose input is neither sanitized on the line, nor sanitized
by a named producer, nor a constant defined in the same file, is a finding — regardless of how
trusted the source looks today.

`DOMPurify` is a direct dependency; use it rather than hand-rolling an escape. Where a call passes
options — `{ ADD_ATTR: ['target'] }` in the markdown renderer, an SVG-scoped config in
`MermaidDiagram.tsx` — the options widen what is allowed, so copying a call site means copying its
threat model too. Read the existing options before reusing them.

## Navigation built from untrusted input

`src/utils/redirectHashRoutes.ts` is the repository's open-redirect precedent (CWE-601,
`EPMCDME-12556`, `git show 7956f6765`). It strips protocol-relative prefixes — `//host`, `/\host` —
from a pathname before handing it to `window.location.replace`.

The rule that fix encodes:

- **A string that came from `window.location`, a query parameter, or an API response is not a
  path.** Strip leading `/` and `\` runs before concatenating it into a URL.
- **Never pass such a string to `location.href` / `assign` / `replace`, `window.open`, or an
  `<a href>` without that treatment.** A leading `//` turns a "relative" redirect into a fully
  qualified one to somebody else's host.
- **The regression test is part of the fix.**
  `src/utils/__tests__/redirectHashRoutes.test.ts` is the shape to copy — the reason this class of
  bug recurs is that the fix is one line and nothing pins it down.

## Agent-authored A2UI media

- **Draw agent-supplied media through the project renderers.** `Image`, `Video` and `AudioPlayer` are registered in `src/a2ui/registry.tsx::OWN_COMPONENTS` through `src/a2ui/factory.tsx::createMediaImplementation` instead of the SDK catalog; a new A2UI component that loads a URL from the agent is registered the same way.
- **Normalize an agent-supplied URL with `sanitizeAgentUrl` before it reaches an attribute.** It returns `null` for a non-string, an empty or relative reference, or any protocol other than `http:` and `https:` (`src/a2ui/utils.ts::sanitizeAgentUrl`).

## Cross-window messages

```bash
grep -rn "addEventListener('message'\|postMessage(" src --include='*.ts' --include='*.tsx'
```

`src/hooks/useAuthCallbackListener.ts` is the pattern. Three properties make it safe, and all
three have to be copied together:

- **The origin is compared to a configured value**, not to a substring or a regex. `getApiOrigin`
  builds it from `appInfoStore.getMcpAuthOrigin()` or `api.BASE_URL` via `new URL(...).origin`, and
  returns `null` on anything it cannot parse — so an unparseable config denies rather than allows.
- **The comparison runs before any state change.** The shape check runs first only so that
  unrelated cross-origin traffic is never logged; `event.origin !== apiOrigin` still gates
  everything that follows. The file says so in a comment — keep it there.
- **The payload is shape-checked** by a type guard, not trusted because the origin matched.

The sender side, `src/pages/login-success/LoginSuccessPage.tsx`, passes
`window.location.origin` as the `targetOrigin` argument. **Never pass `'*'`** — that broadcasts
the message to whatever page happens to hold the opener reference.

### Read first
| When you are | Exemplar | Why this one | Tests |
|---|---|---|---|
| handling a message a hosted application posts through its container frame | `src/utils/hostedApplicationWindows.ts::createHostedWindowOpener` | acts only on payloads passing `isOpenRequest` / `isCloseRequest` and ignores the rest; opens a window only under an active user click and within `HOSTED_OPENS_PER_MINUTE` | `src/utils/__tests__/hostedApplicationWindows.test.ts::createHostedWindowOpener` |

## Browser configuration is public

Configuration reaches the app by two paths, and **both are public**:

| Path | Read via | Set where |
|---|---|---|
| Build time | `import.meta.env.VITE_*` | `ENV VITE_*` lines in `multistage.Dockerfile`, baked into the bundle |
| Run time | `window._env_.*` | `config.js`, copied into the image and served with `Cache-Control: no-store` |

```bash
grep -rn "import.meta.env" src --include='*.ts' --include='*.tsx'
grep -rn "window\._env_" src config.js
cat config.js
```

`src/utils/api.ts` reads `window._env_?.VITE_API_URL` and falls back to the build-time value — that
fallback is why a value can appear at both layers.

A build-time `VITE_*` variable is string-substituted into the shipped JavaScript, and the runtime
`config.js` is fetched by any browser that loads the page. **An API key, a client secret, or a
token placed in either is published.** If a value must stay secret, it belongs behind the backend,
reached through `src/utils/api.ts` — not in the frontend's configuration at all.

## Secrets in the repository

`npm run secrets:check` runs gitleaks against the working tree before every commit. It does not
read git history, and a container-registry failure prints the same "Secrets detected" message a
real finding does — [`../security/README.md`](../security/README.md) § Exit codes that mislead.

A credential that reached a commit is compromised and must be rotated by a human; deleting the line
is not the fix. Rotation-first remediation is owned by `secops:secret-fix`.

`.gitleaks.toml` carries a path allowlist. Adding to it suppresses scanning for everyone — it needs
a reason in the commit message, and it is never the way to get a blocked commit through. gitleaks
also runs server-side as the pipeline's `gitleaks-scan` task, so the allowlist suppresses both
surfaces at once.

### The tracked `.env`

```bash
git ls-files .env && grep -n '^\.env' .gitignore
git show HEAD:.env | sed 's/=.*/=<redacted>/'      # keys only, never the values
```

`.env` is committed and is **not** in `.gitignore`. Among its keys is `KC_ENTRA_CLIENT_SECRET`,
used by `npm run start:keycloak` to render `.keycloakify/realm-kc-26.json` from its template.

That combination is the risk: the file a developer edits to run Keycloak locally is a tracked
file, so the next `git commit -a` publishes whatever they typed into it. Two rules follow:

- **Put a real Entra secret in `.env.local`, never in `.env`.** `.env.local` is the path
  `.gitleaks.toml` already allowlists.
- **`.keycloakify/realm-kc-26.json` is allowlisted too** and is untracked. It holds substituted
  secrets. Never `git add -f` it — gitleaks is configured not to look at it.

`npm run secrets:check` passing on the current tree means gitleaks does not classify today's `.env`
values as credentials. It is not a guarantee about the value you are about to paste in.

The MR-side Sonar report carries a **Security Hotspots** count. Treat a new hotspot the way you
would a failing test: read it before merging, do not accept it silently.

## Sonar suppressions

```bash
grep -rn "nosonar" src --include='*.ts' --include='*.tsx'
grep -n "sonar.issue.ignore" sonar-project.properties
```

`sonar-project.properties` carries an explicit `multicriteria` ignore list keyed by rule and file,
and `src/` carries inline `// nosonar` markers. Each one is a decision somebody made about a
specific rule at a specific place.

- **Do not add a suppression to clear a warning you have not read.** Fix the code first; suppress
  only when the rule genuinely does not apply, and say why in the same commit.
- **Do not widen an existing entry's `resourceKey` to `**/*`** to cover a new file. Add a new
  numbered criterion instead — a widened key silently disables the rule everywhere.
- **Do not delete one as cleanup.** Every entry in the `multicriteria` list is a decision somebody
  recorded on purpose.

## Response headers

```bash
grep -nE "add_header|server_tokens" nginx.conf
```

`nginx.conf` sets `server_tokens off` and per-location `Cache-Control`. It sets no
`Content-Security-Policy`, `X-Frame-Options`, `X-Content-Type-Options` or `Referrer-Policy`, and
nothing in the repository supplies them elsewhere.

That is the current state. Changing it does not belong inside an unrelated fix: `nginx.conf` ships inside the image and no gate validates it, so a header
change is its own ticket with its own rebuild and its own manual verification.

## Examples

### Sanitizing a stored return URL

Source: `src/utils/postLoginRedirect.ts::sanitizePostLoginRedirect`

```ts
export function sanitizePostLoginRedirect(redirect: string | null | undefined): string | null {
  if (!redirect) return null
  // Reject protocol-relative and backslash-relative paths (CWE-601 open-redirect defence).
  if (!redirect.startsWith('/') || redirect.startsWith('//') || redirect.startsWith('/\\')) {
    return null
  }
  return redirect
}
```

### Normalizing an agent-supplied URL

Source: `src/a2ui/utils.ts::sanitizeAgentUrl`

```ts
export function sanitizeAgentUrl(rawUrl: unknown): string | null {
  if (typeof rawUrl !== 'string') return null

  const trimmed = rawUrl.trim()
  if (!trimmed) return null

  let parsed: URL
  try {
    // No base URL on purpose: relative references must fail here.
    parsed = new URL(trimmed)
  } catch {
    return null
  }

  if (!ALLOWED_PROTOCOLS.has(parsed.protocol)) return null

  return parsed.href
}
```

## Common pitfalls

| Symptom | Cause | Fix | Evidence |
|---|---|---|---|
| An injected markdown image fetches an attacker URL | image sources were not checked | route every rendered image through `isImageAllowed` | `src/utils/imageAllowList.ts::isImageAllowed` |
| Sign-in returns the user to another site | a `//host` or `/\host` return path | read it through `consumePostLoginRedirect` | `src/utils/postLoginRedirect.ts::consumePostLoginRedirect` |

## Quick reference

| Task | Do | Where |
|---|---|---|
| Configure the image allow list | see the README section | `README.md` § Image Allow-List |