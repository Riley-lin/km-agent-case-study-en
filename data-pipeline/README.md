# Data Pipeline Engineering

An agent is only as good as what actually makes it into the knowledge base — and real consumer electronics product knowledge is scattered across dynamic web pages, PDFs, PPTs, Excel sheets, and even internal emails, each in a different format that standard knowledge-base crawlers simply can't ingest. This is the engineering work that turns those messy sources into clean, usable knowledge.

## Working around dynamic web page limitations

When the public website renders content with JavaScript (for example, spec content that only loads after a model is selected from a dropdown), a standard "website crawler" knowledge source can't see the real content, and the agent naturally can't answer correctly. The solution is a custom scraper: parse the HTML, route content by type (plain text / text-with-images / tables / table-as-image / video links), run OCR on images, and normalize everything into a format the knowledge base can ingest.

**Technology choices:** on Node.js (≥20, ES Modules), requests use native fetch with session/cookie management and request throttling (roughly 700ms between requests, to avoid being flagged as abnormal traffic and blocked); HTML parsing uses cheerio; HTML/table-to-clean-Markdown conversion uses turndown (with the turndown-plugin-gfm plugin to preserve table structure); OCR uses tesseract.js (dual Traditional Chinese + English models, with a timeout guard so a single stuck image can't block an entire batch); since the target system doesn't ingest Markdown, the final step assembles a docx structure and converts it to a Word file for Copilot Studio to consume. The scraper supports **incremental updates**, so it doesn't need to re-crawl the entire site every run.

## Content that "can't be copied as text" needs visual extraction, not forced text conversion

Wiring diagrams, accessory compatibility charts, and installation-angle illustrations aren't really about the text — forcing them into text just loses fidelity. The solution is to export these pages as images and hand them to a multimodal LLM to generate captions, under a fixed prompt rule: **describe only what's visibly there, and flag anything uncertain for manual review** — so the model doesn't hallucinate content it can't actually make out. Generated captions are cached in `vision-cache.json`, so the same image is never sent to the model twice, saving both time and cost.

## PPT and Excel office formats take a separate conversion path

Product decks (PPT) and spec sheets (Excel) aren't web pages, so the scraper pipeline doesn't apply — they go through a different pipeline: Python (python-docx, openpyxl) handles structured Word/Excel content, with `lxml` + `zipfile` unpacking OOXML directly when needed (PPT/DOCX/XLSX are, under the hood, a bundle of XML files inside a zip, and sometimes it's necessary to read that underlying structure directly rather than work around a library's limitations). PPT decks additionally go through PowerShell with `pywin32`, using COM automation to drive the actual local PowerPoint application and export each slide as an image, which then flows into the same visual-extraction pipeline described above. The converted content is finally batch-cleaned through a Power Automate AI Builder prompt node and normalized before entering the knowledge base.

## Complex tables need explicit conversion rules, not left for the AI to guess

A cross-reference table (models across the top, features down the side, checkmarks in between) loses its meaning if converted to text naively — the AI can't reliably tell which checkmark belongs to which model. The rules have to be defined explicitly up front: how to handle blank cells, how to handle special markers, whether merged-cell values should be propagated down each row, and finally expanding each cell into a complete sentence (e.g. "Model X supports Feature Y"). This translates the tacit understanding a human applies when reading a table into explicit, mechanically executable rules, rather than assuming the AI can infer the table's implicit logic on its own.

## Sensitive data gets automatic masking plus a manual confirmation gate

Extracting knowledge from unstructured sources like internal emails often surfaces personal information (names, contact details, etc.) mixed in. The solution is to automatically mask this kind of information, paired with a **preview step before generation** so a human can confirm the personal information is actually scrubbed and the converted content is correct, before the final file is produced — when automating the handling of sensitive data, the manual confirmation gate can't be skipped.

## Every automated run produces an audit report

After each automated extraction/conversion run, a report is generated: how many records were updated, which ones need manual follow-up, and which images need further analysis. This keeps "what the automation actually did" visible and reviewable, rather than a black box that runs and leaves no one knowing whether the result is correct.

## The pipeline tooling itself is also tested

The scraping and conversion tools built for this pipeline are tested using Node.js's built-in test framework (`node --test`) — they weren't shipped on the assumption that they'd just keep working. This is a separate concern from "the knowledge-base agent itself has no automated test suite": the quality of the agent's answers can't be verified with assertions, but **the tools that produce the knowledge can be**, so the parts of the system that can be tested, are.
