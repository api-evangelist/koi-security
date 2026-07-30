---
name: Screen a VS Code extension for risk with ExtensionTotal
description: >-
  Look up the ExtensionTotal risk score for one or many Visual Studio Code extensions by marketplace
  identifier, apply Koi's own high-risk threshold, and pace bulk scans so you do not trip the free
  rate limit.
api: openapi/koi-security-extensiontotal-openapi.yml
operations:
  - getExtensionRisk
generated: '2026-07-19'
method: generated
---

# Screen a VS Code extension for risk

Koi's ExtensionTotal service scores Visual Studio Code extensions for supply-chain risk. The public
API exposes one operation, `getExtensionRisk`, which takes a marketplace extension identifier and
returns a numeric risk score plus a label.

## Before you start

- **Identify the extension correctly.** The `q` field takes the marketplace identifier in
  `publisher.name` form — for example `ms-python.python`. This is not the display name.
- **Decide whether you need a key.** Anonymous calls work and are rate limited. Koi issues
  organizational API keys, documented as having no rate limit, at
  <https://app.extensiontotal.com/sponsor>. Send a key in the `X-API-Key` header when you have one.
- There is **no idempotency contract and no versioning** on this API. See
  `conventions/koi-security-conventions.yml`.

## Screen a single extension

Call `getExtensionRisk`:

```
POST https://app.extensiontotal.com/api/getExtensionRisk
Content-Type: application/json
X-API-Key: <your key>        # omit for free anonymous access

{ "q": "ms-python.python" }
```

Read these fields off the response: `display_name`, `version`, `risk`, `riskLabel`, `updated_at`.

## Apply the threshold

Koi's own first-party client treats **`risk >= 7` as a high-risk finding** and raises a blocking
alert. Use the same threshold so your results match Koi's tooling. Report `riskLabel` alongside the
number — the label is what a human reviewer should see.

## Scan an estate in bulk

Iterate your installed-extension inventory and call `getExtensionRisk` once per extension, but:

1. **Sleep 1500 ms between requests.** That is the pacing Koi's own client uses.
2. **Abort on the first `429`.** The free rate limit is exhausted; do not retry in a loop. Get a key.
3. **Abort on the first `403`**, or on a response body equal to the literal string `Invalid API key` —
   both mean the key is bad, and every subsequent call will fail the same way.
4. **Cache by extension id and version.** Re-scan only when the installed version changes; that is how
   Koi's client avoids redundant lookups.

Error semantics are catalogued in `errors/koi-security-problem-types.yml`.

## Attribute findings to a machine (optional)

In organization mode, attach reporting context so a finding can be traced back to where it was found:

```json
{ "q": "ms-python.python", "orgData": { "hostname": "...", "username": "..." } }
```

Only send `orgData` when your organization has agreed to that attribution — it carries the hostname
and OS username of the scanning machine.
