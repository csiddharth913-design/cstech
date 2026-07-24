# Swapping Providers Later

This app was built with Supabase (database) and Groq (AI). Neither is locked in — here's roughly what's involved if you (or whoever you hand this off to) want to change any of them, kept brief on purpose.

---

## Swapping the backend (Supabase → something else)

**Why it's contained:** every database read/write in `index.html` goes through one small block of named functions — `dbLoad`, `dbUpsertContact`, `dbUpdateContactFields`, `dbDeleteContact`, `dbInsertLog`, `dbUpdateLog`, `dbDeleteLog`, `dbInsertComment`, `dbDeleteComment`, `dbUpsertTemplate`, `dbDeleteTemplate`, `dbSaveSettings`. Nothing else in the app talks to the database directly.

**What's involved:**
1. Recreate the schema in the new backend (see `SETUP.md` Step 2 for the current full schema — table/column names, not the SQL syntax, are what matter).
2. Rewrite the body of each `db*` function to call the new backend's API instead of `db.from(...)`. The function names, inputs, and outputs should stay the same so nothing calling them needs to change.
3. Recreate the equivalent of Row Level Security in whatever access-control model the new backend uses.
4. Update `functionUrl()` (or remove it) if the AI Edge Functions move too — see below.

Realistic effort: a day or so for someone who knows both platforms; more if the new backend's data model is very different from a relational database (e.g. a document store like Firebase).

---

## Swapping the LLM (Groq → something else)

**Why it's contained:** the frontend never talks to an LLM directly — it only calls two of your own function URLs (`summarize-contact`, `ai-enrich-contact`). All the provider-specific code (endpoint, model name, request/response shape) lives inside those two files and nowhere else.

**What's involved:**
1. Open `supabase/functions/summarize-contact/index.ts` and/or `supabase/functions/ai-enrich-contact/index.ts`.
2. Swap the `fetch(...)` call's URL, headers, and body to match the new provider's API, and update how the response is parsed to pull out the generated text.
3. Update the secret name/value (e.g. `supabase secrets set OPENAI_API_KEY=...` instead of `GROQ_API_KEY`).

If the new provider also uses an OpenAI-compatible chat completions shape (many do), this can be as small as changing a URL, a key, and a model name. Moving to a provider with a different shape (e.g. Google's or Anthropic's own SDKs) means rewriting the request-building and response-parsing in that one file — still fully contained there.

**Note for `ai-enrich-contact` specifically:** it depends on the model having *built-in web search* (that's how it finds current news/company info). If the new provider doesn't have an equivalent, you'd need to either find one that does, or wire in a separate search API call inside that function yourself.

---

## Adding Clay (or another enrichment provider) back in

This is purely additive — nothing currently in the app needs to change, you'd just add to it.

**What's involved:**
1. A new Edge Function that fires Clay's inbound webhook with the contact's info.
2. A second new Edge Function that Clay calls back once its enrichment waterfall finishes, which writes the result onto the contact.
3. A small DB migration for whatever fields Clay returns.

One thing worth knowing going in: sending results *back out* from a Clay table (the second function above) requires Clay's paid **Growth plan** — their free tier doesn't support it. Worth checking current Clay pricing before committing to that path. The `ai-enrich-contact` function already in this app was actually built as a free alternative to this exact Clay setup, for that reason.

If you'd rather add a different enrichment vendor (Clearbit, Apollo, PDL, etc.) instead of Clay, the same two-function pattern (or a single synchronous call, if the vendor's API responds immediately rather than via webhook) applies — check whether the vendor's API is synchronous (request → immediate response, one function needed) or asynchronous (request → webhook callback later, two functions needed) before building it, since that decides the shape of the integration.
