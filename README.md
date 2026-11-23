# Ask Rishika: a RAG chatbot over my own documents

A small retrieval-augmented generation (RAG) chatbot that answers questions about my work, projects, education and skills, using only the documents in `docs/`. Every answer shows how it was found: the chunks retrieved, their similarity scores, the tokens used, the estimated cost and the time taken.

## How it works
1. `docs/*.md` are split into 35 chunks (`scripts/build-chunks.mjs` → `data/chunks.json`).
2. On the first request, all chunks are embedded with **gemini-embedding-001** (768 dimensions) and kept in memory.
3. The question is embedded the same way, and the top 4 chunks are found by **cosine similarity** (vector search).
4. **Guardrail:** if the best match is below `MIN_SCORE` (default 0.55), the bot refuses *without* calling the language model. That means no made-up answers and no wasted tokens.
5. Otherwise the top chunks go into the prompt within a **~2,500-token context budget**. **Gemini 2.5 Flash-Lite** answers, with **Gemini 2.5 Flash** as an automatic fallback if the first model fails.
6. The system prompt only allows facts from the context, asks for sources, and blocks phone numbers or email.

## Files
- `index.html`: chat page with the "How this answer was found" panel
- `api/ask.js`: Vercel serverless function (POST `/api/ask`)
- `lib/rag.js`: embeddings, vector search, context budget, guardrail, fallback, cost
- `lib/gemini.js`: Gemini REST calls (no SDK)
- `tests/rag.test.mjs`: offline tests with a fake embedder (`npm test`)
- `scripts/eval.mjs` + `tests/eval-set.json`: live evaluation on 20 questions (15 answerable, 5 off-topic)

## Deploy on Vercel
1. Create a new GitHub repo (for example `ask-rishika`) and upload all these files.
2. In Vercel: **Add New → Project → import the repo → Deploy**. No build settings are needed.
3. In **Project → Settings → Environment Variables**, add `GEMINI_API_KEY` (the same key you use for the voice bot), then **Redeploy**.
4. Open the site and try the suggested questions.

## Tune the guardrail (do this once after deploying)
On your laptop, with Node 18+:
```
GEMINI_API_KEY=your_key node scripts/eval.mjs
```
It prints the retrieval hit rate and how many off-topic questions were correctly refused. If good questions get refused, lower `MIN_SCORE` (for example to 0.5). If off-topic questions get answered, raise it. Then set `MIN_SCORE` as an environment variable in Vercel.

## Updating the content
Edit the files in `docs/`, run `node scripts/build-chunks.mjs`, and push. Everything in `docs/` is public, so keep phone numbers and email out of it.
