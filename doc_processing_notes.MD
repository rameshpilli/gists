Doc Type Router first — before anything goes into S3, you route by file type. A PDF, PPTX, and image need completely different downstream handling. This is the fork point.
Storage Adapter pattern — S3 is primary today, Box.com is dashed because it's a future plug-in. A senior engineer always separates the interface from the implementation.
Redis Cache Check in INDEX — before re-processing a doc that hasn't changed, bail early. This alone saves significant compute cost at scale.
Table Extractor is its own lane — you specifically said tabular extraction is 90% of your real-world needs. It deserved its own box, not bundled with "text."
Cap IQ as an external service — shown with bidirectional dashed arrows because it's a side-effect call inside PARSE, not part of the linear pipeline. It breaks out of the stage flow intentionally.
Data contracts on every arrow — file_id · path · type → text[] · chart[] · img[] → embeds[] · data[] · src[]. A senior engineer always asks what format crosses the boundary.
LLM-as-Judge Validator in GENERATE — with the explicit target of 90%. This makes the accuracy requirement a first-class architectural concern, not an afterthought.
Infrastructure layer maps to stages — Nemo feeds PARSE, Postgres holds metadata from INDEX, Redis is shared, Llama Parse is the external API that powers PARSE. Each infra component has a reason to exist.


Step 1 — File Comes In (Input)
A user uploads a PDF, a PowerPoint, maybe a Word doc. The very first thing that happens is the Doc Type Router looks at it and says — okay, is this a PDF? A PPTX? An image? — because the way you handle each one downstream is completely different. At the same time, we check Redis: have we processed this exact file before? If yes, skip everything and return the cached result. Saves a ton of compute.

Step 2 — INDEX (Store it, tag it)
Once we know what type it is, we push the raw file into S3. That's your permanent home for the file. Then we write a record into Elasticsearch — not the content yet, just the metadata: file ID, path, type, who uploaded it, when. Think of this like the library catalogue entry before you've even opened the book.


Step 3 — EXTRACT (Pull out the pieces)
Now we crack the document open. This is where four parallel things happen:

Text Extractor pulls out all paragraphs, headers, footnotes
Table Extractor pulls out rows and columns — this is the most important one, 90% of your real use cases live here
Chart Extractor identifies chart types, axes, series labels — it knows a chart exists but doesn't understand the data inside it yet
Image Extractor grabs embedded visuals and diagrams separately

At the end of EXTRACT you have a clean separation: text is text, tables are tables, charts are charts, images are images. No more mixed blob.


Step 4 — PARSE (Understand what each piece means)
This is the brain of the whole system, and it's powered by Llama Parse.

The text goes through Nvidia Nemo to generate embeddings, which get stored in Elasticsearch or KDBAI as vectors — this powers semantic search later
The tables get parsed into structured row/column arrays with their source tagged
The charts — this is the clever part — Llama Parse looks at the chart, converts it into an array of values, then reads the footnote to figure out where this data came from. If it says "Source: Cap IQ as of 2024," it fires a Cap IQ MCP call to fetch the live current data, and updates the chart with fresh numbers. That's huge for financial docs
Images get described and classified separately

Everything parsed gets its metadata written to PostgreSQL — lineage, state, what source the data came from, when it was last refreshed.


Step 5 — GENERATE (Build the actual output)
Now we have clean, understood, structured content. The generate layer takes all of that and:

Runs it through the Summarization Engine — the same one you already have, reused
Uses Open Code + Plot Skills to re-render charts dynamically if the data was refreshed from Cap IQ
The Doc Renderer assembles it all back into a proper output — PowerPoint, PDF, or Word, depending on what was asked for

Before anything goes out, the LLM-as-Judge Validator checks the output and scores it. Target is 90% accuracy or above. If it doesn't hit that, it flags it.


A user in the pitchbook creator uploads or references something like:

An earnings PDF from a company
A Cap IQ exported report
A research document with charts and tables
What the "cached result" would actually be:
It's NOT the final PowerPoint or document. It's the parsed, structured intermediate output — the expensive part. Specifically:

The extracted text chunks with their embeddings already computed
The tables already converted to structured row/column arrays
The charts already identified, values extracted, source tagged
The Cap IQ data already fetched and attached to those charts


So in Redis you'd store something like:
key:  hash(file_id + last_modified_timestamp)

value: {
  text_chunks: [...embeddings already in Elastic...],
  tables: [{ rows: [...], cols: [...], source: "Cap IQ 2024" }],
  charts: [{ type: "bar", values: [...], capiq_refreshed_at: "2025-01-10" }],
  images: [{ id: "img_1", description: "..." }]
}
The real-world scenario in your context:
Imagine a banker opens pitchbook creator and says "build me a pitch for Company X." You have already processed Company X's earnings PDF last week for a different banker. The Llama Parse run, the Cap IQ MCP call, the Nemo embedding generation — all of that already happened.
Without cache → you run all of it again. Cap IQ call, Llama Parse, Nemo embeddings. That's time and money.
With cache → you check: same file, same version? Hit. You skip straight to GENERATE with the pre-parsed chunks already in hand.

But here's the honest caveat for your system:
The tricky part is Cap IQ data. That data is live and changes. So your cache key needs to be smart:

Static content (text, tables, images) → cache aggressively, long TTL
Cap IQ-powered charts → either skip caching entirely, or cache with a short TTL like 24 hours, so you're not serving stale financial data to a banker
Llama Parse says →  "this chart has these values: [12, 45, 67, 23]
                     it's a bar chart, x-axis is quarters"

OpenCode says    →  "let me write a matplotlib / plotly script,
                     execute it, and produce an actual rendered chart
                     that goes into your PowerPoint slide"


The specific skills it runs in your context:

Plot Code Skill — takes chart value arrays, generates rendered charts
Slide Builder Skill — takes content blocks, assembles into PowerPoint slides
Document Formatter — takes text + tables + charts, composes into Word or PDF
