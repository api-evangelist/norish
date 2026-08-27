---
name: norish-plan-the-week
description: Plan a household's meals for the coming week on a Norish instance — find recipes, place them on the shared calendar, and read back what is planned.
api: Norish Recipe API
base: https://{norish-instance-host}/api/v1
operations:
- recipeSearch (POST /api/v1/recipes/search)
- plannedRecipeCreate (POST /api/v1/planned-recipes)
- plannedRecipesWeek (GET /api/v1/planned-recipes/week)
- plannedRecipesToday (GET /api/v1/planned-recipes/today)
- plannedRecipeDelete (DELETE /api/v1/planned-recipes/{itemId})
generated: '2026-08-27'
method: generated
source: https://docs.norish.dev/reference/api
---

# Plan the week in Norish

Norish is self-hosted. There is no vendor endpoint: the base URL is the household's own
instance, `https://<their-host>/api/v1`, and the credential is an API key issued by that
instance.

## Before you start

- Authenticate every call with `x-api-key: <key>` (or `Authorization: Bearer <key>`).
- `GET /api/v1/health` is the only unauthenticated operation. Call it first; it returns 200
  only when the API *and* the parser service are healthy, so a non-200 means imports will
  fail even if reads work.
- Put an `x-operation-id` header on every write. Norish claims that id in Redis for 7 days
  and returns the stored response on a replay instead of re-running the handler, so a retry
  after a dropped connection will not double-book a meal.

## Steps

1. **Check the instance is up.** `GET /api/v1/health`.
2. **See what is already planned.** `GET /api/v1/planned-recipes/week` — the week runs
   Monday through Sunday in the *server's* timezone, not the caller's. Do not compute the
   week boundary yourself.
3. **Find candidate recipes.** `POST /api/v1/recipes/search`. Useful filters: `search`,
   `searchFields`, `tags`, `categories`, `filterMode`, `sortMode`, `minRating`,
   `maxCookingTime`. Paging is cursor-based — `cursor` defaults to `0`, `limit` to `50`.
   Ask for a page at a time; do not try to pull the whole library.
4. **Read a candidate in full** before committing to it: `GET /api/v1/recipes/{id}`.
5. **Place it on the calendar.** `POST /api/v1/planned-recipes` with a fresh
   `x-operation-id`.
6. **Confirm.** Re-read `GET /api/v1/planned-recipes/week` (or `/today`, `/month`) rather
   than trusting your own bookkeeping.

## Undoing

Every plan you create is reversible: `DELETE /api/v1/planned-recipes/{itemId}`. Norish
states no time limit on this, so treat it as available but do not promise the household a
window the docs do not give.

## What this API cannot do

There is no `DELETE /api/v1/recipes/{id}`. A recipe you create or import through the API
cannot be removed through the API — removal is a web-app action. Say so rather than
attempting it.
