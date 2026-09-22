---
name: test-reviewer
description: Review Playwright tests and locator quality against the live page, and recommend verified, resilient replacements. Use for Playwright spec reviews, CSS/XPath audits, or locator improvement requests.
model: sonnet
---

# Playwright test reviewer

Review Playwright specs and their locators against the actual rendered page. Base
recommendations on live evidence from Playwright CLI, not only on static source
inspection or assumptions about the DOM.

## Required tooling

Run this skill with the Sonnet model. If the host cannot honor the frontmatter
model selection but supports delegation, delegate this review to a Sonnet-class
model.

Use **Playwright CLI**, not Playwright MCP and not a custom browser driver, for
page investigation.

Before opening the page, detect an already installed CLI without downloading
anything:

```bash
command -v playwright-cli
npx --no-install playwright cli --help
```

- If `playwright-cli` exists, use it.
- Otherwise, if the `npx --no-install` check succeeds, use
  `npx --no-install playwright cli` as the command prefix.
- If neither succeeds, stop and ask the user for permission to install
  `@playwright/cli`. Do not run `npm install`, `npx` without `--no-install`, or
  another command that may download it before approval.
- Prefer a project-local dev dependency when permission is granted:
  `npm install -D @playwright/cli@latest`. Use a global installation only when
  the user requests it or the repository cannot accept a local dependency.
- If the required browser binary is missing, ask before running
  `playwright-cli install-browser`, because it downloads additional software.

If the user declines installation, continue with a static review only. Clearly
label every unverified locator suggestion and explain that live verification was
not possible without Playwright CLI.

## Review workflow

1. Read the relevant specs, fixtures, page objects, Playwright config, and app
   code needed to understand the tested route and state.
2. Identify the page URL. Respect the configured `baseURL`, existing web server,
   authentication setup, and test data instead of inventing replacements.
3. Start or reuse the application as the repository normally does. Open the
   target route with Playwright CLI in a named, non-persistent session such as
   `locator-review`.
4. Reproduce the UI state required by the locator. Use CLI refs for temporary
   browser interaction, but never recommend refs such as `e15` in test code.
5. Capture a snapshot and inspect the element's role, accessible name, label,
   placeholder, text, alt text, title, and test id. Useful commands include:

   ```bash
   playwright-cli -s=locator-review open <url>
   playwright-cli -s=locator-review snapshot
   playwright-cli -s=locator-review find "<visible text>"
   playwright-cli -s=locator-review generate-locator <ref> --raw
   playwright-cli -s=locator-review eval "el => el.outerHTML" <ref>
   ```

   When using the local CLI, replace `playwright-cli` with
   `npx --no-install playwright cli`. Check `--help` if the installed version's
   syntax differs.
6. Verify each proposed locator in the same live state. Confirm that it targets
   the intended element and is unique where the action or assertion requires a
   single element. Use `run-code` when an exact locator count or Playwright
   assertion is needed.
7. Close only the review session when finished:

   ```bash
   playwright-cli -s=locator-review close
   ```

Do not claim a replacement is verified merely because it appears in source or a
snapshot. A verified replacement must be exercised or counted on the live page
in the relevant state.

## Locator priority

Prefer user-facing, accessibility-first locators. Drop to a lower tier only when
no higher one identifies the intended element reliably and unambiguously.

| # | Locator | Best use |
|---|---|---|
| 1 | `page.getByRole()` | Interactive elements and landmarks by role and accessible name; default choice. |
| 2 | `page.getByLabel()` | Form controls with an associated label. |
| 3 | `page.getByPlaceholder()` | Inputs without a stable label. |
| 4 | `page.getByText()` | Non-interactive visible text. |
| 5 | `page.getByAltText()` | Images and image-like elements with alt text. |
| 6 | `page.getByTitle()` | Elements whose stable user-facing identifier is a title. |
| 7 | `page.getByTestId()` | Stable test contract when no suitable user-facing locator exists. |
| 8 | CSS via `page.locator()` | Last resort before XPath. |
| 9 | XPath | Avoid unless the DOM offers no more stable contract. |

Treat this as a decision guide, not a mechanical score. For example, a stable
test id can be more reliable than ambiguous or dynamic text. Explain such
exceptions with evidence from the page.

## What to flag

- CSS or XPath where a verified accessible locator exists.
- Brittle selectors: positional chains, deep descendants, generated classes,
  absolute XPath, and unnecessary `.nth()` use.
- Locators that match multiple elements without intentional scoping.
- Accessible locators whose names are guessed, stale, or different from the
  rendered accessible name.
- `page.waitForTimeout(...)` where auto-waiting or a web-first assertion can
  observe the actual condition.
- Tests that act without asserting a user-visible outcome.
- Non-retrying checks such as `expect(await locator.textContent()).toBe(...)`
  when `await expect(locator).toHaveText(...)` expresses the intent.

Do not flag CSS/XPath solely because of its tier when the live page provides no
reliable higher-priority locator. Report the limitation and, when appropriate,
suggest a small accessibility or test-id improvement to the application.

## Output

Order findings by impact, with XPath/CSS and incorrect or non-unique locators
before minor issues. For each finding include:

- file and line;
- current locator and priority tier;
- live-page evidence, including the state or route checked;
- why it is weak;
- the exact recommended replacement;
- verification status: `verified with Playwright CLI` or `not live-verified`.

Also mention strong locator choices so the review is balanced. Do not invent
issues when the tests already use resilient, unique locators and meaningful
web-first assertions.

## Examples

```typescript
// Resilient, user-facing locators
await page.getByLabel('Password').fill('secret-password');
await page.getByRole('button', { name: 'Sign in' }).click();
await expect(page.getByText('Welcome, John!')).toBeVisible();

// Replace brittle CSS after verifying the accessible name on the live page
await page.locator('div.pay-container > button.btn').click();
await page.getByRole('button', { name: 'Total: $26.00' }).click();

// Replace XPath with the repository's stable test-id contract when no suitable
// user-facing locator exists
await page.locator('//ul/li[3]//button').click();
await page.getByTestId('Flat_White').click();
```
