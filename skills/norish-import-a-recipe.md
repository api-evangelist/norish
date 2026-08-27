---
name: norish-import-a-recipe
description: Import a recipe into Norish from a web URL or from pasted text, and understand what can go wrong in the render-and-parse pipeline behind it.
api: Norish Recipe API
base: https://{norish-instance-host}/api/v1
operations:
- health (GET /api/v1/health)
- recipeImportUrl (POST /api/v1/recipes/import/url)
- recipeImportPaste (POST /api/v1/recipes/import/paste)
- recipeGet (GET /api/v1/recipes/{id})
- recipeSearch (POST /api/v1/recipes/search)
generated: '2026-08-27'
method: generated
source: https://docs.norish.dev/reference/api
---

# Import a recipe into Norish

## Check health first — this one really matters here

`GET /api/v1/health` returns 200 only when the API **and** the internal parser service are
both healthy. Import is the operation that depends on the parser, so a failing health check
predicts a failing import. Check it before you import, not after.

## URL import

`POST /api/v1/recipes/import/url`.

What happens behind the call, per <https://docs.norish.dev/configuration/parser>: the page is
loaded in Obscura, a stealth headless browser, because many recipe sites build the page with
JavaScript and the recipe is not in the raw HTML. The rendered HTML then goes to the Python
parser service, which uses `recipe-scrapers` and reads `application/ld+json` schema.org
Recipe markup, with an AI fallback behind it if the instance has an AI provider configured.

Consequences an agent should hold on to:

- If Obscura cannot reach or render the page, the import fails outright. There is no
  plain-HTTP fallback and no second browser. Do not retry the same URL expecting a different
  path through the system.
- Obscura refuses loopback, private-network and link-local addresses. Never try to import
  from `localhost`, `127.0.0.1`, `10.x`, `192.168.x` or `169.254.x` — it is a deliberate
  block, not a bug.
- A source that needs a login works only if the household has already saved site
  authentication tokens for it. You cannot supply credentials through this call.

## Paste import

`POST /api/v1/recipes/import/paste` takes recipe text directly. Use it when the URL import
fails, when the source is behind a wall, or when the user pasted the recipe to you rather
than linking it.

## After the import

Read the result back with `GET /api/v1/recipes/{id}`, or find it via
`POST /api/v1/recipes/search`. Verify the ingredients and steps actually landed before
telling the user the recipe is in — an AI-fallback parse is a best effort, not a guarantee.

## Idempotency and the missing undo

Send `x-operation-id`; a replayed import returns the stored response rather than creating a
duplicate recipe. This matters more than usual because **there is no delete on this API** —
no `DELETE /api/v1/recipes/{id}` is documented, so a recipe imported twice has to be cleaned
up by a human in the web app.
