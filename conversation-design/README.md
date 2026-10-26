# Conversation Design

This layer decides whether the agent should answer, what it should look up, and when it should honestly admit it doesn't know — the layer between a user's raw message and a well-sourced answer.

## Role and scope boundaries

The agent is positioned as an internal knowledge assistant dedicated to the consumer electronics customer service team. It only answers within a defined scope: model specifications, FAQs, configuration, warranty and repair, firmware updates, accessory compatibility, and troubleshooting — it is not an open-ended Q&A assistant. Sales-related information such as inventory, pricing, or lead time isn't in the knowledge base at all, so the agent clearly directs those questions to sales or the PM contact; fully unrelated questions get a scope explanation instead. When a single message mixes in-scope and out-of-scope content, the in-scope part is still answered normally rather than rejecting the whole message.

## Intent detection before retrieval

Internal staff frequently paste a customer's exact wording — colloquial, with typos, sometimes several questions in one message. The agent first restates its understanding in one line ("I understand you're asking about ___"), letting the user immediately correct it, before converting the colloquial phrasing into a standardized query for retrieval. When a single message contains multiple questions, each is handled separately rather than only answering the first one detected.

## Model-number matching — the core safeguard

This is the system's single highest-priority rule: different models are always treated as entirely different products, and data is never cross-applied just because models look similar, belong to the same series, or share similar features. A real example of the confusion this guards against: a model with an "R" prefix and its non-prefixed counterpart look like the same series but are actually different product lines (a rear-view-mirror form factor vs. a standard unit); two models differing by a single letter prefix can also have completely different specs. This is enforced through four layers of protection:

1. **A master model list**: the authoritative list of every valid model number — matching is based on this list, not pattern-guessing.
2. **Validate before use**: before using any retrieved data, first confirm that the model number explicitly labeled on the source actually matches what the user asked about.
3. **List candidates when not found, but never jump to an answer**: when a model number might be mistyped or incomplete (the string the user entered doesn't exactly match anything in the master list), the system lists the closest candidate models in the same product line and asks the user to confirm — and those candidates can never be used as the source for an answer before the user confirms.
4. **A second verification gate before output**: right before the answer is sent, every cited passage is re-checked against the model number it claims to be about; any passage that doesn't match is simply removed, even if that leaves the answer incomplete — no other model's data is substituted in to fill the gap.

**This is distinct from the case where the model number is correct but the knowledge base simply has no data for it.** Rule 3 handles the case where the model number itself is in question (mistyped or incomplete) — the agent must ask first and is barred from citing any candidate data. But if the model number the user entered is a genuinely valid entry in the authoritative master list, and the knowledge base simply hasn't been populated with data for that specific model yet, that's a different situation — the agent can explicitly say "no data found for [this model]; the following is reference data for the similar model [Model B]," but it cannot repackage that reference data as an official answer for the original model. The distinction: the first case is "the model identity itself is uncertain, so no candidate data may be used"; the second is "the model identity is certain, the knowledge base has a gap, and clearly-labeled reference data counts as partially correct at best — never fully correct."

The same discipline extends to list-type questions (e.g. "which models support WiFi") — a model can only be listed if the retrieved data is itself an explicit model-to-feature mapping table; the system never treats "this model happens to appear in a document that mentions this feature" as evidence that the model actually supports it.

## Progressive slot-filling

When a question is ambiguous, the agent asks one clarifying question at a time, and only queries the knowledge base once enough information has been gathered. Two question types are distinguished: model-specific questions (specs, configuration, troubleshooting, purchase recommendations) require both a valid model number and a specific need before being considered complete; general questions unrelated to a specific model (membership registration, warranty policy, repair process) don't require a model number first. Information already provided is never asked for again, and some follow-up questions branch based on the answer received (e.g. during an installation-environment assessment, the on-site conditions reported by the user determine which installation guidance to surface).

## Anti-hallucination and confidence thresholds

"Insufficient data" is a deliberately designed, fixed output state — not something left to the model's discretion. The rules explicitly prohibit inferring an answer from similarity, prohibit providing numbers or test results not present in the source knowledge, and require that quoted text match the source verbatim. The system would rather say "no data found in the knowledge base" than give an answer that sounds plausible but isn't grounded in anything.

## Query rewrite and retry

When the first retrieval attempt comes back empty, or the answer contains phrases like "insufficient data" or "not found" with no clickable source attached, the system automatically rewrites the colloquial query into the phrasing used in the documents (for example, completing a model number's formal prefix) and retries once — invisibly to the user, but with a meaningful improvement in hit rate. This retry only happens once, to avoid an endless loop of rewriting a query that simply has no match.

## Output discipline across multi-step calls

The pipeline chains together several independent retrieval/verification steps, each responsible for exactly one thing: a step that has nothing to output must be able to output nothing at all — no filler explanation, no restating the question, no "let me look that up for you" transition text. This is essentially the single-responsibility principle applied to prompt design, keeping intermediate-step noise out of the final answer.

## Tiered source labeling

Content cited from internally-restricted documents (e.g. an unannounced product intro, an internal spec sheet) automatically gets a disclaimer reminding staff that "this is internal information — confirm disclosure scope before sharing externally"; content sourced from the public website or published FAQs doesn't need this. This bakes a data-governance rule directly into the output logic, rather than relying on manual spot checks after the fact.

## Language consistency

The user's language is detected once per conversation and stored in a shared variable; every subsequent message — including button labels, disclaimers, and follow-up questions — reads from that same variable, so there's no "the main answer is in the right language but a detail sentence slips into the other one."

## Three failure modes, tracked separately

"Couldn't answer" is deliberately split into three tracked categories: question unclear (ask a follow-up), out of scope (explain the service boundary), and genuinely no data available (be honest about it and suggest manual confirmation). Lumping these together makes it hard to diagnose problems later — a spike in "unclear" cases and a spike in "no data found" cases point to completely different fixes.
