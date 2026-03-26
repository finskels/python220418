## Base Specification

```yaml
agent_name: agent_publisher_v8_0_4
purpose: >-
  Build a monetizable blog workflow for SaaS, IT products, software, reviews,
  tutorials, and practical information content with SEO, CTR optimization,
  internal linking, WordPress conversion structure, performance-based rewrites,
  Search Console title optimization, and revenue + CTA optimization.

agent_role:
  responsibilities:
    - discovering monetizable categories and keywords
    - selecting keywords with commercial or strategic informational intent
    - planning articles around realistic SERP expectations
    - generating CTR-oriented titles
    - inserting internal-link structures
    - planning affiliate links, banners, and CTAs
    - producing publication-ready WordPress drafts
    - saving topic memory to reduce duplication
    - monitoring published content performance
    - diagnosing underperforming posts
    - rewriting existing posts based on evidence
    - ingesting Search Console performance data
    - optimizing titles through CTR testing
    - ingesting revenue and conversion data
    - optimizing CTAs and monetization flow
    - reporting workflow usage and status
  optimize_for:
    - search intent alignment
    - usefulness and credibility
    - buyer-decision support
    - scalable internal topical authority
    - non-spammy monetization
    - performance improvement over time
    - revenue growth from existing content
  avoid:
    - thin content
    - unsupported claims
    - fake first-hand experience
    - aggressive or deceptive CTA language
    - over-optimized internal links
    - repetitive AI-sounding phrasing
    - unnecessary full rewrites when targeted edits are enough
    - excessive experimentation without enough data

global_operating_rules:
  - never fabricate facts, prices, statistics, product capabilities, or personal experience
  - prefer verifiable and commercially useful specificity over vague generalities
  - maintain clean variable handoff between steps; every downstream field must come from an explicitly produced upstream output
  - title and meta fields must be distinct and must not collapse into each other
  - do not auto-publish unless the publish step explicitly sets status=publish
  - all final body content sent to WordPress must use the markdown single source of truth path
  - meta_description must be stored separately from title and excerpt and exported as its own file
  - excerpt must be stored separately from body content and exported as its own file
  - if a shell/export step creates required publish artifacts, a verification step must confirm they exist before publish

# Approval / Execution Policy
approval:
  mode: token-file
  token_file: "~/.config/openclaw/allow_run"
  token_ttl_seconds: 900  # token valid duration (seconds)
  required_for_steps:
    - X41_wordpress_publish

# Note: create the token with secure permissions (chmod 600) and rotate regularly.


input:
  site:
    wordpress_base_url: "https://pickrtech.com"
    wordpress_username: "${WORDPRESS_USERNAME}" 
    wordpress_application_password: "${WORDPRESS_APP_PW}"
    openclaw_shared_secret: "${OPENCLAW_SHARED_SECRET}"

  telegram:
    bot_token: "${TELEGRAM_BOT_TOKEN}"
    chat_id: "${TELEGRAM_CHAT_ID}"
  memory:
    posted_topics: []

flow:
  # ── Prefix 범례 ──────────────────────────────────────────
  # P: Plan     — 키워드·방향 결정       (P00–P09)
  # R: Research — 구조·링크·리서치       (R10–R19)
  # W: Write    — 초안 작성·저장         (W20–W29)
  # E: Edit     — 퇴고·링크·SEO·저장     (E30–E39)
  # X: eXport   — 발행·검증             (X40–X49)
  # Z: Zap      — 후처리 (메모리·알림)   (Z50–Z59)
  # ─────────────────────────────────────────────────────────
  - P00_seed_direction
  - P01_keyword_scoring
  - P02_keyword_selection
  - R10_outline_generation
  - R11_internal_link_planning
  - R12_option_research       # 신규: 옵션별 심층 리서치
  - R13_claims_and_axes       # 신규: 비교 축·검증 클레임·질문 생성
  - W20_draft_writing
  - W21_save_raw_draft
  - E30_load_raw_draft
  - E31_editorial_review
  - E32_humanize
  - E33_link_inject           # 신규: internal link 삽입 (humanize 이후)
  - E34_seo_fields
  - E35_save_polished_draft
  - E36_export_publish_files
  - E37_artifact_validation
  - X40_load_publish_artifacts
  - X41_wordpress_publish
  - X42_verify_wordpress_post
  - Z50_memory_save
  - Z51_telegram_report
  - Z51_telegram_fail_report  # E31 reject 시 실패 리포트 (Z51 성공 리포트와 별개)
```

```yaml
## [P00] Seed Direction (revised by Claud:Sonnet 4.6)
id: P00_seed_direction
type: llm
model: gpt-5-mini
max_tokens: 1800  # 변경: 1400 → 1800 (track_reason 포함 10개 항목 출력 여유 확보)
description: Generate two-track keyword directions: primary article candidates and support/adjacent candidates
input:
  site_focus: "OpenClaw-driven monetizable tech blog for SaaS, software, IT products, tutorials, comparisons, reviews, and alternatives"
  monetization_goal: "affiliate revenue + SEO traffic"
  allowed_primary_content_types:
    - comparison
    - review
    - alternatives
    - recommendation
  allowed_support_content_types:
    - tutorial
    - troubleshooting
    - comparison
    - recommendation
  avoid_patterns:
    - generic head terms
    - weak buyer intent
    - celebrity or gossip topics
    - unprovable experience claims
    - news-dependent topics requiring daily freshness
  recent_memory: "{{memory.posted_topics}}"  # 추가: 발행된 글 목록으로 중복 방지

prompt: |
  Generate 10 focused keyword directions for a monetizable tech blog, split into 2 tracks.

  # SITE CONTEXT
  - Site focus: {{site_focus}}
  - Monetization goal: {{monetization_goal}}
  - Already published topics (avoid near-duplicates of these):
    {{recent_memory}}

  # YOUR ROLE
  You are a monetizable tech blog editor. Your job is to find the best next article topics
  by identifying:
  - what SaaS, software, or IT tool categories are currently trending or gaining search interest
  - what buyer decisions are actively being made in the tech market right now
  - which product comparisons, alternatives, or reviews would attract high-intent readers
  - which topics have strong affiliate monetization potential

  Do not limit topics to any pre-set niche. Explore across SaaS, developer tools,
  IT products, software categories, ecommerce platforms, and any adjacent tech area
  where buyer-facing comparison content can generate affiliate revenue.

  # DUPLICATE AVOIDANCE RULE
  - Check each candidate against recent_memory before including it.
  - Do not generate a keyword that closely overlaps an already-published topic
    (same product pair, same framing, or same buyer problem).
  - If recent_memory is empty, skip this check.

  # TRACK DEFINITIONS
  - primary_seed_keywords (exactly 6 items):
    - broad comparison, recommendation, alternatives, or review candidates
    - must naturally support a buyer-decision article with a shortlist, comparison table,
      scenario-based recommendations, and a final recommended pick
    - this is the main pool for choosing the next article
  - support_seed_keywords (exactly 4 items):
    - narrower but still monetizable adjacent candidates
    - may include implementation-led, migration-led, schema-led, reliability-led,
      pricing-math-led, ops-led, or narrower technical comparison topics
    - useful as secondary angles, support sections, or future adjacent articles
    - must NOT dominate the pool

  # HARD CONSTRAINTS — violating these makes the output invalid
  - primary_seed_keywords MUST contain exactly 6 items.
  - support_seed_keywords MUST contain exactly 4 items.
  - The following topic types MUST go into support_seed_keywords, never primary:
    - schema-led
    - reliability-led
    - ops-led
    - migration-led
    - pricing-math-led
    - latency-led or benchmark-led
    - implementation-heavy without a clear buyer shortlist frame
  - Every item MUST include the track_reason field.
    Omitting track_reason on any item makes the entire output invalid.
  - content_type must be exactly one of:
      comparison | review | alternatives | recommendation | tutorial | troubleshooting
  - monetization_strength must be exactly one of: high | medium | low
  - buyer_stage must be exactly one of: problem-aware | solution-aware | decision-ready

  # PRIORITIZE FOR primary_seed_keywords
  - "best X for Y" style topics
  - "X vs Y" when it supports a broad buyer-facing shortlist comparison
  - "X alternatives" topics
  - "X review" topics
  - shortlist-style and category-level recommendation topics
  - topics that naturally support product-by-product comparison, scenario-based
    recommendations, and a final ranked shortlist
  - long-tail keywords with clear buyer or decision intent
  - topics that map to monetizable product categories or affiliate-relevant tools

  # PRIORITIZE FOR support_seed_keywords
  - adjacent but still monetizable or decision-relevant seeds
  - narrower technical comparison candidates
  - implementation, migration, schema, reliability, or ops topics — only when they
    still connect to provider selection, tool choice, shortlist logic, or a
    monetizable adjacent workflow

  # DEPRIORITIZE OVERALL
  - Seeds that would most naturally become a technical explainer instead of a
    decision-support product article
  - Strategy-only, reliability-only, architecture-only, cost-analysis-only, or
    infrastructure-metric-only topics unless they clearly support a product choice
  - Seeds where the reader problem is clear but the product choice is too indirect

  # BALANCE RULES
  - primary_seed_keywords should be clearly stronger as main article candidates
    than support_seed_keywords
  - If both a broad comparison version and a narrow implementation version exist for
    the same reader problem, the broad comparison version goes in primary_seed_keywords
    and the narrow version goes in support_seed_keywords

  # OUTPUT RULES
  - Return ONLY valid JSON. No preamble, no markdown fences, no trailing commentary.
  - Every item in both arrays must include all 5 fields:
    keyword, content_type, monetization_strength, buyer_stage, track_reason.

output_schema:
  primary_seed_keywords:       # MUST be exactly 6 items
    - keyword: string
      content_type: string     # comparison | review | alternatives | recommendation
      monetization_strength: string  # high | medium | low
      buyer_stage: string      # problem-aware | solution-aware | decision-ready
      track_reason: string     # why this belongs in primary (must mention shortlist or comparison fit)
  support_seed_keywords:       # MUST be exactly 4 items
    - keyword: string
      content_type: string     # tutorial | troubleshooting | comparison | recommendation
      monetization_strength: string
      buyer_stage: string
      track_reason: string     # why this belongs in support (must mention narrower or adjacent reason)
retry: 2
timeout: 30
next: P01_keyword_scoring
```

```yaml
## [P01] Keyword Scoring (revised by Claud:Sonnet 4.6)
id: P01_keyword_scoring
type: llm
model: gpt-5-mini
max_tokens: 2800  # 변경: 1800 → 2800 (10개 키워드 전체 채점 + JSON 출력에 충분한 여유)
description: Score primary-track and support-track keywords for main article selection
input:
  primary_seed_keywords: "{{P00_seed_direction.output.primary_seed_keywords}}"
  support_seed_keywords: "{{P00_seed_direction.output.support_seed_keywords}}"
  recent_memory: "{{memory.posted_topics}}"

prompt: |
  Score the keyword candidates for this blog.

  You are given 2 tracks:
  - primary_seed_keywords: main article candidates (track = "primary")
  - support_seed_keywords: narrower adjacent candidates (track = "support")

  # TRACK ASSIGNMENT RULES
  - Every keyword from primary_seed_keywords must have track = "primary" in its scored row.
  - Every keyword from support_seed_keywords must have track = "support" in its scored row.
  - The track field is required on every scored_keywords item. Omitting it makes the output invalid.

  # MAIN GOAL
  - Choose the strongest next article topic from the primary track unless there is an unusually
    strong reason not to.
  - Keep support-track candidates available for secondary keywords, support sections, or future
    follow-up articles.
  - Avoid letting narrower technical topics outrank broader buyer-facing recommendation topics
    as the final main article choice.

  # TRACK HANDLING RULES
  - Primary-track candidates start with an editorial advantage because they are intended as
    main article candidates.
  - Support-track candidates may still score well, but they should usually rank below clearly
    stronger primary-track shortlist candidates.
  - Do not choose a support-track keyword as strongest_recommendation_keyword unless it clearly
    beats the best primary-track candidates in final article usefulness.
  - If a primary-track candidate and a support-track candidate are similarly strong, prefer
    the primary-track candidate for strongest_recommendation_keyword.

  # SCORING DIMENSIONS
  Score each keyword on these 7 dimensions using a strict integer from 1 to 5:
  - monetization_score   : affiliate/revenue potential (5 = highest)
  - intent_score         : clarity and strength of buyer/decision intent (5 = clearest)
  - usefulness_score     : likely article depth and reader usefulness (5 = most useful)
  - differentiation_score: uniqueness vs. existing coverage (5 = most differentiated)
  - evergreen_score      : long-term traffic and expansion value (5 = most evergreen)
  - duplicate_risk_score : risk of overlapping recent memory (5 = no risk, 1 = high risk)
  - internal_link_score  : potential to link to or from existing posts (5 = strongest)

  # TOTAL SCORE FORMULA
  Compute total_score using this exact weighted formula:
    total_score = (monetization_score   * 20)
                + (intent_score         * 20)
                + (usefulness_score     * 15)
                + (differentiation_score * 15)
                + (evergreen_score      * 10)
                + (duplicate_risk_score * 10)
                + (internal_link_score  * 10)
  Maximum possible score: 500.
  Do not use any other formula. Do not invent weights.

  # TOTAL SCORE OUTPUT RULE
  - total_score MUST be stored as a computed integer, not as an expression string.
  - Compute the arithmetic result BEFORE writing the JSON value.
  - Example: scores (5,5,4,3,4,3,4) → (5*20)+(5*20)+(4*15)+(3*15)+(4*10)+(3*10)+(4*10) = 100+100+60+45+40+30+40 = 415
  - INVALID: "total_score": "(5*20+5*20+4*15+3*15+4*10+3*10+4*10)"
  - VALID:   "total_score": 415
  - Outputting a formula string instead of an integer makes the entire output invalid.

  # TOTAL SCORE SELF-CHECK — REQUIRED BEFORE WRITING JSON
  After assigning all 7 dimension scores for a keyword, verify total_score as follows:
  1. Write out each term explicitly:
       (monetization_score × 20) = ___
       (intent_score × 20)       = ___
       (usefulness_score × 15)   = ___
       (differentiation_score × 15) = ___
       (evergreen_score × 10)    = ___
       (duplicate_risk_score × 10) = ___
       (internal_link_score × 10) = ___
  2. Sum all 7 terms.
  3. Confirm the sum equals total_score before writing the JSON field.
  Skipping this check or omitting internal_link_score from the sum makes the output invalid.


  # SCORING RULES
  - Strongly reward keywords that naturally support:
    - a real vendor or product shortlist
    - a meaningful comparison table
    - option-by-option reviews
    - scenario-based recommendations
    - a clear final recommended pick
  - Reserve recommendation = "shortlist" primarily for broad or category-level recommendation
    articles. Do NOT shortlist narrower latency tests, cost analyses, benchmarks, schema guides,
    tutorials, reliability notes, migration notes, or implementation-heavy comparisons.
  - When both are viable, prefer category-level or shortlist-level comparison topics over
    narrower A-vs-B, benchmark-led, migration-led, schema-led, reliability-led, or
    cost-analysis-led topics.
  - If a keyword is mainly a benchmark, latency test, cost analysis, schema guide, tutorial,
    reliability note, architecture note, migration note, or technical explainer, score
    monetization_score and intent_score meaningfully lower even if it is still useful.
  - Weight commercial actionability and category-level recommendation fit more heavily than
    narrower technical usefulness.
  - Do not place a keyword in shortlist just because it contains "vs" or "comparison".

  # STRONGEST RECOMMENDATION OVERRIDE
  After computing total_score for all keywords, apply this override before setting
  strongest_recommendation_keyword:
  - If the highest-scoring keyword is latency-led, benchmark-led, cost-analysis-led,
    schema-led, migration-led, or ops-led, AND a broader category-level comparison keyword
    exists in the shortlist, choose the broader category-level keyword instead.
  - strongest_recommendation_keyword must represent the best final article candidate,
    not merely the highest numeric total.
  - If a broad category-level comparison and a narrower benchmark-led or latency-led
    comparison score similarly, always choose the broader category-level comparison.
  - Document any override in strongest_recommendation_reason.

  # HARD CONSTRAINTS — violating any of these makes the entire output invalid
  - shortlist_keywords MUST contain exactly 2, 3, or 4 items. Never fewer than 2, never more than 4.
  - scored_keywords MUST include at least 2 items with recommendation = "reject".
  - scored_keywords MUST include at least 2 items with recommendation = "consider".
  - An output where all items are "shortlist" is ALWAYS invalid.
  - An output with zero "reject" items is ALWAYS invalid.
  - If you cannot find natural rejects, lower your threshold — not every keyword deserves shortlist.
  - strongest_recommendation_keyword MUST be one of the shortlist_keywords values.
  - Every scored_keywords item MUST include all schema fields including track, content_type,
    duplicate_risk_score, internal_link_score, and evergreen_score.

  # RECOMMENDATION LABEL RULES
  - recommendation must be exactly one of: shortlist | consider | reject
  - shortlist: broad buyer-facing comparison topic with clear shortlist/table/scenario potential
  - consider: useful but narrower, adjacent to the main buyer path, or less direct purchase intent
  - reject: primarily benchmark-led, pricing-math-led, migration-led, schema-led,
    reliability-led, or ops-led without clear buyer-facing shortlist framing
  - Row-level recommendation labels must be consistent with shortlist_keywords.
  - If a keyword appears in shortlist_keywords, its scored row must use recommendation = "shortlist".
  - If a keyword does not appear in shortlist_keywords, its scored row must NOT use recommendation = "shortlist".

  # REASON RULES
  - Do not reuse the same reason sentence across multiple candidates.
  - Reasons must clearly distinguish broad buyer-facing comparison topics from narrower
    technical or adjacent topics.
  - shortlist reasons must explicitly include at least one of:
    broad comparison scope | shortlist potential | category-level recommendation |
    scenario-based recommendation strength
  - consider reasons must explicitly include at least one of:
    useful but narrower | adjacent to the main buyer path | less direct purchase intent
  - reject reasons must explicitly include at least one of:
    benchmark-led scope | pricing-math-led scope | migration-led scope |
    schema-led scope | reliability-led scope | ops-led scope
  - strongest_recommendation_reason must be more specific than the generic row reasons
    and must explain:
    (a) why this candidate is the best final article choice, and
    (b) if an override was applied, why the broader keyword was chosen over the highest scorer.

  # OUTPUT RULES
  - Return ONLY valid JSON. No preamble, no markdown fences, no trailing commentary.
  - Every scored_keywords item must include ALL fields in the output schema.
  - Always include shortlist_keywords, strongest_recommendation_keyword, and
    strongest_recommendation_reason at the top level.

output_schema:
  scored_keywords:
    - keyword: string
      track: string                  # "primary" or "support"
      content_type: string
      monetization_score: integer    # 1-5
      intent_score: integer          # 1-5
      usefulness_score: integer      # 1-5
      differentiation_score: integer # 1-5
      evergreen_score: integer       # 1-5
      duplicate_risk_score: integer  # 1-5 (5 = no risk)
      internal_link_score: integer   # 1-5
      total_score: integer           # computed by weighted formula above, max 500
      recommendation: string         # "shortlist" | "consider" | "reject"
      selection_confidence: integer  # 1-10
      disqualifiers:
        - string
      reason: string
      article_shape_note: string
  shortlist_keywords:                # MUST be 2-4 items only
    - string
  strongest_recommendation_keyword: string
  strongest_recommendation_reason: string
retry: 2
timeout: 30
next: P02_keyword_selection
```

```yaml
## [P02] Keyword Selection (revised by Claud:Sonnet 4.6)
id: P02_keyword_selection
type: llm
model: gpt-5-mini
max_tokens: 1200
description: Select one primary keyword and article direction by AI Agent
input:
  scored_keywords: "{{P01_keyword_scoring.output.scored_keywords}}"
  shortlist_keywords: "{{P01_keyword_scoring.output.shortlist_keywords}}"
  strongest_recommendation_keyword: "{{P01_keyword_scoring.output.strongest_recommendation_keyword}}"
  strongest_recommendation_reason: "{{P01_keyword_scoring.output.strongest_recommendation_reason}}"

prompt: |
  Select one keyword for the next article.

  # SELECTION POOL
  - Use shortlist_keywords as the primary candidate pool.
  - strongest_recommendation_keyword is a strong hint, not an automatic final choice.
  - You may override strongest_recommendation_keyword when a broader primary-track
    shortlist candidate supports a stronger buyer-facing comparison article.
  - If you override, document the reason in selection_rationale.

  # HARD CONSTRAINTS — violating these makes the output invalid
  - primary_keyword MUST come from shortlist_keywords unless all shortlist candidates
    fail the article viability check below.
  - primary_keyword MUST be able to support a useful 2000+ word article with:
      - a meaningful comparison table
      - option-by-option analysis
      - scenario-based recommendations
      - a final recommended pick
      - FAQ
  - chosen_content_type MUST be exactly one of:
      comparison | review | alternatives | recommendation
  - secondary_keywords MUST contain 1 to 4 items, never more than 4.
    Target 2 to 4. If Step 1 scored_keywords contains fewer than 2 viable secondary
    candidates, return as many as are available (minimum 1).
    Never invent or rewrite keywords to meet the minimum.
  - selection_rationale MUST be included and must explain the final choice.
    If strongest_recommendation_keyword was overridden, it must explain why.

  # PRIMARY KEYWORD SELECTION RULES
  - Prefer primary-track candidates over support-track candidates.
  - When both a broad category-level comparison keyword and a narrower technical
    comparison keyword are in the shortlist, always choose the broad category-level one.
  - If a shortlist candidate supports a "best X for Y" or shortlist-style article,
    prefer it over a latency-led, benchmark-led, cost-analysis-led, or
    pilot-checklist-led keyword.
  - If strongest_recommendation_keyword is latency-led, benchmark-led,
    cost-analysis-led, schema-led, migration-led, or ops-led, AND a broader
    primary-track shortlist candidate exists, override it with the broader candidate.
  - Prefer long-tail keywords with strong buyer or decision intent.
  - Avoid near-duplicates of already-covered topics.

  # ARTICLE ANGLE RULES
  - article_angle MUST stay anchored to a recommendation or comparison frame.
  - article_angle MUST NOT center on latency testing, pricing math, pilot checklists,
    or engineering evaluation mechanics when a broader buyer-facing comparison
    candidate is available.
  - article_angle MUST NOT drift into an internal pilot memo, ops note, or
    implementation document without a clear reader-facing choice framework.
  - Valid article_angle examples:
      "vendor comparison + scenario-based recommendations + shortlist"
      "category-level roundup with ranked shortlist and buyer scenarios"
      "alternatives comparison with migration and fit guidance"
  - Invalid article_angle examples:
      "latency benchmark + cost model + pilot checklist"
      "operational metrics comparison + engineering evaluation"

  # SECONDARY KEYWORDS RULES
  - secondary_keywords must strengthen the primary article's comparison logic,
    implementation context, or scenario guidance.
  - Support-track candidates that were not chosen as primary_keyword are good
    candidates for secondary_keywords.
  - Include 2 to 4 items only.
  - secondary_keywords must be selected only from Step 1 scored_keywords
  - do not invent, rewrite, expand, or normalize new secondary keywords that were not present in Step 1 scored_keywords
  - every secondary keyword must exactly match a keyword string from Step 1 scored_keywords
  - if there are not enough good secondary candidates in Step 1 scored_keywords, return fewer secondary keywords rather than generating new ones

  # SECONDARY KEYWORD EXCLUSION RULE
  - secondary_keywords MUST NOT include any keyword where recommendation = "reject" in scored_keywords.
  - Only keywords with recommendation = "shortlist" or "consider" are eligible for secondary_keywords.
  - If filtering out rejects leaves fewer than 2 candidates, return as many as are available rather than including a rejected keyword.

  # AFFILIATE PRODUCTS RULE
  - likely_affiliate_products must reflect the real product shortlist implied by
    primary_keyword.
  - Do not include adjacent tool categories that would not naturally appear in
    the article's comparison or recommendation sections.

  # OUTPUT RULES
  - Return ONLY valid JSON. No preamble, no markdown fences, no trailing commentary.
  - All fields in the output schema are required.

output_schema:
  primary_keyword: string
  secondary_keywords:          # 1–4 items (target 2–4; min 1 if pool is thin)
    - string
  chosen_content_type: string  # comparison | review | alternatives | recommendation
  target_reader: string
  search_intent_summary: string
  article_angle: string        # must be recommendation/comparison-framed, not latency/ops-led
  likely_affiliate_products:
    - string
  selection_rationale: string  # why this keyword was chosen; explain any override of strongest_recommendation_keyword
retry: 2
timeout: 30
next: R10_outline_generation
```

```yaml
## [R10] Outline Generation (revised by Claud:Sonnet 4.6)
id: R10_outline_generation
type: llm
model: gpt-5
max_tokens: 1800
description: Build a practical article outline and metadata for the chosen topic
input:
  primary_keyword: "{{P02_keyword_selection.output.primary_keyword}}"
  secondary_keywords: "{{P02_keyword_selection.output.secondary_keywords}}"
  chosen_content_type: "{{P02_keyword_selection.output.chosen_content_type}}"
  target_reader: "{{P02_keyword_selection.output.target_reader}}"
  search_intent_summary: "{{P02_keyword_selection.output.search_intent_summary}}"
  article_angle: "{{P02_keyword_selection.output.article_angle}}"
  likely_affiliate_products: "{{P02_keyword_selection.output.likely_affiliate_products}}"

prompt: |
  Create a detailed outline and metadata for the article.

  # ARTICLE CONTEXT
  - Primary keyword: {{primary_keyword}}
  - Secondary keywords: {{secondary_keywords}}
  - Content type: {{chosen_content_type}}
  - Target reader: {{target_reader}}
  - Search intent: {{search_intent_summary}}
  - Article angle: {{article_angle}}
  - Affiliate products to cover: {{likely_affiliate_products}}

  # OUTLINE HARD CONSTRAINTS — violating these makes the output invalid
  - The option-by-option review section MUST include exactly the products listed in
    likely_affiliate_products, one H3 subsection per product.
  - Do NOT add "Other notable choices", "Honorable mentions", or any unnamed open slots
    beyond the products in likely_affiliate_products.
  - If a product has limited public information, include it with honest trade-off notes
    rather than omitting it or adding a placeholder.
  - The outline MUST contain 6 to 8 H2 sections. No fewer than 6, no more than 8.
  - Sections that cover the same practical theme must be merged into one H2.
    Examples of sections to merge:
    - "Comparison table" + "Pricing and value" → merge into one comparison H2
    - "Restore testing checklist" + "Implementation checklist" → merge into one
      setup/testing H2 when both cover operational guidance for the same product category
    - "Why X matters" + "How we evaluated" → merge into one criteria/methodology H2
    - "TL;DR" does NOT count as an H2 section — place it as a pre-section note or
      fold it into the opening of the first H2
  - A "Quick answer" section is treated the same as TL;DR — it does NOT count as an H2
    section. Fold it into the opening paragraph of the first H2 instead.
  - The outline must end with a final verdict or conclusion section.
    The last H2 MUST serve as the article's verdict. If the last section is FAQ,
    its heading must be renamed to reflect a final recommendation (e.g. "Final verdict
    and FAQ") and its subpoints must include at least one explicit recommended pick.
  - The option-by-option reviews for ALL products must be grouped under ONE H2 section
    (e.g. "Product reviews"), not split into separate H2s per product

  # H2 COUNT SELF-CHECK — perform this before writing the JSON
  - After drafting the outline, count the number of items in the outline array.
  - If the count is fewer than 6: merge nothing further — you have under-structured.
  - If the count is more than 8: you MUST merge sections before outputting.
    Merge candidates in order: background/intro sections first, then checklist/setup
    sections, then FAQ can absorb edge-case recommendation questions.
  - Do not output an outline array with more than 8 items under any circumstances.
  - A count of 9 or more is always invalid regardless of content quality.
  - "Quick answer" and "TL;DR" sections must NOT appear as outline array items.
    If you drafted one, merge it into the first H2's subpoints before counting.  

  # OUTLINE QUALITY RULES
  - Build the outline as a decision-support article, not a descriptive overview.
  - The outline must help the reader choose, not just understand the category.
  - Prefer a structure that moves from: decision criteria → comparison → option reviews
    → scenario recommendations → implementation/setup → FAQ → final verdict.
  - Avoid sections that spend too much space on generic background or obvious benefits
    unless directly needed for the reader's decision.
  - For comparison-style topics, include:
    - a quick answer or TL;DR section
    - a section explaining what actually matters when evaluating the options
    - at least one meaningful comparison table
    - a detailed option-by-option review section (one H3 per product)
    - a recommendations-by-scenario section
    - an implementation, setup, or testing guidance section when relevant
    - a concise FAQ section for decision-stage questions
    - a final verdict section
  - Each option subsection must support: strengths, limitations, best fit,
    not-ideal-for cases, and meaningful trade-offs in the article's applied context.
  - Do not collapse all options into a single shallow overview.
  - The comparison table must help the reader decide, not just list features.
  - Recommendations by scenario must reflect real reader use cases, budgets, team sizes,
    or operational contexts — not abstract product tiers.
  - FAQ must prioritize questions that help a reader choose, adopt, switch, or validate
    an option — not generic definitional questions.
  - Treat the outline as weak if:
    - options are mentioned but not differentiated
    - there is no obvious place for comparison logic or scenario-based recommendations
    - it would produce a descriptive article that does not help the reader choose

  # TITLE HARD CONSTRAINTS — violating these makes the output invalid
  - Every candidate in title_candidates MUST be 60 characters or fewer. Count carefully.
  - best_title MUST be 60 characters or fewer.
  - best_title MUST be the shortest candidate that preserves the core search intent.
  - If two candidates express the same intent, always choose the shorter one as best_title.
  - backup_title must also be 60 characters or fewer and must be the second-shortest
    valid candidate.

  Title style rules:
  - Put the core keyword near the front.
  - Avoid colons, em dashes, and subtitle-style expansions.
  - Remove secondary modifiers aggressively.
  - Target 42-54 characters as the preferred range.
  - Move descriptive wording into provisional_meta_description instead of the title.
  - Do not stack year + subtitle + qualifier in the same title.

  # TAGS RULES
  - tags MUST contain 3 to 5 items.
  - All tags must be lowercase.
  - Use hyphens instead of spaces (e.g. "woo-commerce" not "woo commerce").
  - No special characters other than hyphens.
  - Tags must reflect the article's primary topic, product category, and platform context.

  # META DESCRIPTION RULES
  - provisional_meta_description must be plain text only. No markdown, no line breaks.
  - Target 135-150 characters.
  - Must summarize the post clearly for search users.
  - Must NOT simply repeat the title.
  - Must NOT begin with "#", "TL;DR", or "-".

  # OUTPUT RULES
  - Return ONLY valid JSON. No preamble, no markdown fences, no trailing commentary.
  - All fields in the output schema are required.

output_schema:
  title_candidates:              # MUST be 5 items, each 60 chars or fewer
    - string
  best_title: string             # shortest valid candidate, 60 chars or fewer
  backup_title: string           # second-shortest valid candidate, 60 chars or fewer
  provisional_slug: string       # lowercase, hyphens only, keyword-relevant
  provisional_meta_description: string  # 135-150 chars, plain text
  category: string
  tags:                          # MUST be 3-5 items, lowercase, hyphens only
    - string
  outline:                       # MUST be 6-8 H2 sections
    - heading: string
      subpoints:
        - string
retry: 2
timeout: 30
next: R11_internal_link_planning
```

```yaml
## [R11] Internal Link Planning (revised by Claud:Sonnet 4.6)
id: R11_internal_link_planning
type: llm
model: gpt-5-mini
max_tokens: 1400
description: Plan only strong internal links using stored post memory
input:
  primary_keyword: "{{P02_keyword_selection.output.primary_keyword}}"
  chosen_content_type: "{{P02_keyword_selection.output.chosen_content_type}}"
  outline: "{{R10_outline_generation.output.outline}}"
  category: "{{R10_outline_generation.output.category}}"
  tags: "{{R10_outline_generation.output.tags}}"
  posted_topics: "{{memory.posted_topics}}"
prompt: |
  Review memory and recommend only the strongest internal links.

  # POSTED TOPICS GUARD
  - If posted_topics is empty or contains no items, return the empty-state response
    immediately. Do not generate, infer, or fabricate any URLs.
  - Only use URLs that appear verbatim in posted_topics.
  - If no verified internal URL exists for a candidate link, omit it entirely.
    Do not return placeholder URLs, unknown post IDs, or unresolved links.

  # LINK SELECTION RULES
  - Recommend 0 to 5 links only.
  - Skip weak or forced matches — fewer high-quality links beat more marginal ones.
  - Prefer links that help the reader make their next practical decision:
    setup, migration, hosting, performance, plugin selection, or related buying context.
  - Every link must map to a specific section in the outline where it will appear.
    Do not assign a link to a section whose heading is identical to the link's topic
    without specifying the exact placement (e.g. "opening sentence of the section",
    "after the comparison table", "in the final recommendation paragraph").

  # ANCHOR TEXT RULES — STRICTLY ENFORCED
  - suggested_anchor_text MUST NOT copy the target_keyword verbatim.
  - suggested_anchor_text MUST NOT copy the target_title verbatim.
  - Anchor text must read naturally as part of a sentence — it describes
    what the reader will learn or do next, not the title of the linked page.
  - Prefer concise, context-fitting phrases that reflect the reader's next question.
  - GOOD examples:
      target_keyword: "wordpress backup strategies"
      → anchor: "how full and incremental backups compare"
      → anchor: "which backup approach fits your store size"

      target_keyword: "restore testing checklist"
      → anchor: "how to verify your backups actually work"
      → anchor: "a step-by-step restore verification process"
  - BAD examples (do not produce these):
      → "wordpress backup strategies"   ← exact keyword match
      → "restore testing checklist"     ← exact keyword match
      → "WordPress Backup Strategies: Full vs Incremental"  ← exact title match

  # PLACEMENT RULES
  - target_section must name the outline section where the link will appear.
  - placement_note must specify WHERE inside that section the link belongs:
    e.g. "opening sentence", "after the comparison table",
    "in the closing recommendation paragraph", "as a parenthetical in bullet 2".
  - Do not assign a link to a section heading that is identical to the link's topic
    without a placement_note that clearly distinguishes where in the section it sits.

  # OUTPUT RULES
  - Return ONLY valid JSON. No preamble, no markdown fences, no trailing commentary.
  - If no strong links qualify, return the empty-state response below.

output_schema:
  no_existing_posts: boolean       # true = posted_topics가 비어 있어 링크 계획 불가
  internal_link_plan:
    - target_title: string
      target_keyword: string
      target_url: string           # must appear verbatim in posted_topics
      target_section: string       # outline section heading where link appears
      placement_note: string       # exact position within the section
      suggested_anchor_text: string  # must NOT be exact-match of target_keyword or target_title
      anchor_rationale: string     # one sentence: why this anchor text fits the reader's context
      placement_reason: string     # why this link strengthens the reader's next decision here

  # Empty-state response when posted_topics is empty:    # CHANGED: 신규 추가
  # {"no_existing_posts": true, "internal_link_plan": []}
retry: 2
timeout: 30
next: R12_option_research
```

```yaml
## [R12] Option Research (revised by Claude:Sonnet 4.6)
id: R12_option_research
type: llm
model: gpt-5
max_tokens: 2500
description: >
  Gather deep per-option research for each affiliate product.
  Focuses exclusively on option-level data: strengths, limitations,
  tradeoffs, and applied-context fit. Claims and axes are handled in R13_claims_and_axes.
input:
  outline: "{{R10_outline_generation.output.outline}}"
  primary_keyword: "{{P02_keyword_selection.output.primary_keyword}}"
  likely_affiliate_products: "{{P02_keyword_selection.output.likely_affiliate_products}}"
  chosen_content_type: "{{P02_keyword_selection.output.chosen_content_type}}"
  target_reader: "{{P02_keyword_selection.output.target_reader}}"
  article_angle: "{{P02_keyword_selection.output.article_angle}}"
prompt: |
  Research each product in likely_affiliate_products for the article on: {{primary_keyword}}
  Article type: {{chosen_content_type}}
  Target reader: {{target_reader}}
  Article angle: {{article_angle}}
  Use the outline as the target structure — every finding should map to a section.

  # AFFILIATE PRODUCT GUARD
  - major_options MUST include every product in likely_affiliate_products.
  - Never drop a product. If data is limited, include it with empty strengths
    and a caution_points entry explaining what could not be confirmed.

  # APPLIED CONTEXT — DERIVE FROM THE ARTICLE, DO NOT ASSUME
  The article is about: {{primary_keyword}} ({{chosen_content_type}})

  Before researching any product, derive the applied context from primary_keyword,
  target_reader, and article_angle. Ask yourself:

  1. DEPLOYMENT ENVIRONMENT — Where does the product run or integrate?
     (e.g. specific platform, OS, hosting type, cloud provider, CMS, framework)
  2. USER WORKFLOW — What is the reader actually trying to accomplish?
     (e.g. migrate data, reduce cost, automate a task, protect against failure)
  3. OPERATIONAL CONSTRAINTS — What real-world limits shape the decision?
     (e.g. team size, budget tier, technical skill level, compliance needs,
      scale of data, frequency of use)
  4. ECOSYSTEM FIT — What adjacent tools or systems must the product work with?
     (e.g. existing stack, integrations, APIs, plugins, third-party services)

  Write these four dimensions into applied_context_summary (output field).
  Every finding for every option must be grounded in these four dimensions.
  Generic infrastructure metrics (raw throughput, API latency, storage speed)
  are low value unless they directly affect a real decision in this context.

  # QUALITY BAR — ALL THREE TESTS MUST PASS FOR EVERY OPTION

  TEST 1 — Limitations must name a specific failure mode or operational friction.
  These descriptions FAIL the test (too generic — do not write these):
    - "Advanced features behind paywall"
    - "Higher cost"
    - "Requires plugin maintenance"
  These descriptions PASS the test (specific and actionable):
    - "WP-Cron dependency means backups silently fail on low-traffic sites
       unless an external cron trigger is configured separately"
    - "Restore UI requires manually selecting each table group; a full-site
       restore on a 5 GB store typically takes 20–40 minutes of active steps"
    - "Free tier caps remote storage at 1 GB, which is insufficient for stores
       with large product image libraries without upgrading to a paid plan"
  The examples above are from a WordPress backup article. For your article,
  the specific failure modes will be different — derive them from applied_context_summary.
  Minimum 2 passing limitation items per option.
  If you cannot find 2 specific limitations, add a caution_point explaining
  what could not be confirmed, and write the best available limitation.

  TEST 2 — operational_tradeoffs must name the contrasting option directly.
  This format FAILS:
    - "Less hands-on control but higher reliability"
    - "Requires cron validation"
  This format PASSES:
    - "Unlike BlogVault, UpdraftPlus requires the site owner to configure and
       validate WP-Cron or an external cron service, adding ongoing maintenance overhead"
    - "Unlike UpdraftPlus, BlogVault handles restore orchestration server-side,
       removing the need for manual table selection during recovery"
  The examples above are from a WordPress backup article. Apply the same
  "Unlike [Other], [This] …" pattern using the actual products in this article.
  Every operational_tradeoffs item must follow this pattern.

  TEST 3 — Applied context fit must appear in at least one of: strengths, limitations,
  best_for, not_ideal_for, or operational_tradeoffs per option.
  Accepted applied-context signals must be derived from applied_context_summary —
  they are not fixed. For each option, ask: does the data address the deployment
  environment, user workflow, operational constraints, or ecosystem fit identified
  in applied_context_summary? If none of these appear in an option's data, the
  option fails TEST 3.

  # SELF-CHECK before writing JSON
  [ ] Did I write applied_context_summary before researching any option?
  [ ] Does every limitation name a specific failure mode, not a generic category?
  [ ] Does every operational_tradeoffs item follow "Unlike [Other], [This] …"?
  [ ] Does every option pass TEST 3 against my own applied_context_summary,
      not against a hardcoded list of WooCommerce signals?

  # OUTPUT RULES
  - Return ONLY valid JSON. No preamble, no markdown fences, no trailing commentary.

output_schema:
  applied_context_summary: string  # 4 dimensions: deployment env · user workflow · operational constraints · ecosystem fit
  major_options:
    - option_name: string
      strengths:
        - string
      limitations:           # min 2 items; each must pass TEST 1
        - string
      best_for:
        - string
      not_ideal_for:
        - string
      operational_tradeoffs: # each item must follow "Unlike [Other], [This] …" pattern
        - string
      likely_cost_positioning: string
      notable_features:
        - string
      caution_points:
        - string
retry: 2
timeout: 45
next: R13_claims_and_axes
```

```yaml
## [R13] Claims and Axes (revised by Claud:Sonnet 4.6)
id: R13_claims_and_axes
type: llm
model: gpt-5
max_tokens: 2000
description: >
  Build comparison axes, verified claims, caution claims, decision-stage questions,
  and risk summary from the option research produced in R12_option_research.
  All outputs feed directly into step6 draft writing.
input:
  primary_keyword: "{{P02_keyword_selection.output.primary_keyword}}"
  chosen_content_type: "{{P02_keyword_selection.output.chosen_content_type}}"
  outline: "{{R10_outline_generation.output.outline}}"
  major_options: "{{R12_option_research.output.major_options}}"
  applied_context_summary: "{{R12_option_research.output.applied_context_summary}}"
prompt: |
  You are given option research for an article on: {{primary_keyword}}
  Article type: {{chosen_content_type}}
  Applied context (derived in R12_option_research): {{applied_context_summary}}
  Major options researched: {{major_options}}

  Your job is to produce the comparison framework and fact layer that step6 will use
  to write the comparison table, scenario recommendations, and FAQ.

  # COMPARISON AXES
  Rules:
  - Produce 5 to 7 axes.
  - Every axis must be usable as a comparison table column header in step6.
  - At least 2 axes must be specific to the article's applied context.
    Derive applied-context axes directly from applied_context_summary — do not use a
    fixed list. For each of the 4 dimensions in applied_context_summary (deployment
    environment, user workflow, operational constraints, ecosystem fit), ask: what
    comparison axis would help the reader make a better decision in that dimension?
    Examples of how this works in practice:
      deployment env "WordPress hosting"  → "Hosting stack compatibility"
      user workflow "restore after hack"  → "Recovery speed and restore reliability"
      operational constraint "solo dev"   → "Setup complexity and ongoing maintenance"
      ecosystem fit "staging required"    → "Staging environment support"
    Generate axes that match your article's actual applied_context_summary, not these examples.
  - Do not repeat the same concept with different wording.
  - Do not use abstract infrastructure metrics (raw storage speed, API latency)
    unless they directly affect a real user decision in this context.

  # VERIFIED CLAIMS vs CAUTION CLAIMS — CLASSIFICATION RULES
  Before writing any claim, classify it using these definitions:

  VERIFIED CLAIM: A specific, factual statement that is directly stated on the vendor's
  official feature page, documentation page, or publicly confirmed product changelog.
  The claim must be stable across plans and regions, or the specific plan/tier must be
  named. If you cannot confirm it on a specific official page, it is NOT verified.

  CAUTION CLAIM: Any claim where one or more of the following is true:
    - The feature or limit differs by plan, tier, region, or partner channel
    - The public documentation is inconsistent or incomplete
    - The claim was true at a past date but may have changed
    - The claim applies to some configurations but not all
    - You cannot identify the specific URL that confirms it

  A claim cannot be both verified and caution. If there is any doubt, classify as caution.
  Do not place a claim in verified_claims if a related caution_claim covers the same
  feature for the same product — this creates a contradiction in the output.    

  # VERIFIED CLAIMS
  Rules:
  - Minimum: (number of major_options) × (number of comparison_axes) claims.
    Example: 2 options × 6 axes = at least 12 verified claims.
  - Every claim must be a specific, factual statement — not a category description.
  - Every claim must include a source reference in parentheses.
    Accepted formats:
      "(updraftplus.com/docs/scheduled-backups)"   ← preferred: specific page
      "(updraftplus.com/features)"                 ← acceptable: feature page
      "(updraftplus.com)"                          ← fallback only: homepage
    Not accepted:
      "(vendor site)"
      "(updraftplus)"
  - If a claim cannot be verified, move it to caution_claims instead.
  - Claims must cover all comparison_axes for all major_options.
    If a claim for a specific option × axis combination cannot be verified,
    add a caution_claim for that combination and note what is missing.

  # CAUTION CLAIMS
  Rules:
  - Minimum: 1 caution claim per major option.
  - Each caution claim must:
      1. State the specific claim that could not be fully verified.
      2. Explain exactly what is uncertain (number, date, tier, version, etc.).
      3. Include a URL to check before publishing.
  This format FAILS:
    "Pricing/retention varies by plan; verify vendor pages"
  This format PASSES:
    "BlogVault advertises 365-day backup retention on higher tiers, but the exact
     day count per plan is not confirmed in public documentation as of this research —
     verify current limits at blogvault.net/pricing before publishing this claim."

  # DECISION STAGE QUESTIONS
  Rules:
  - Minimum 5 questions.
  - Must cover all four reader actions: choose, adopt, switch, validate.
  - Maximum 2 questions per action type.
  - Every question must be specific to the article's applied context
    as defined in applied_context_summary — not generic product-category theory.
  - Label each question with its action type in this format.
    The examples below are from a WordPress backup article; derive yours from
    applied_context_summary for the actual article:
      "[choose] Which option best fits [specific deployment env from applied_context_summary]?"
      "[adopt] How do I set up [product] for [specific user workflow from applied_context_summary]?"
      "[switch] How do I move from my current solution without [key operational risk]?"
      "[validate] How do I confirm [product] works correctly in [specific ecosystem context]?"
      "[validate] What should I check after [key action] to confirm [outcome relevant to reader]?"
  - Do not include the label brackets in the final JSON value —
    use them only to verify coverage before writing output.

  # RISK SUMMARY
  - One paragraph, 2–4 sentences.
  - Must name at least one specific failure mode from the option research.
  - Must give the reader one concrete action to reduce risk.

  # SELF-CHECK — run this before writing JSON
  Before outputting, verify:
  [ ] comparison_axes: at least 2 applied-context axes present?
  [ ] verified_claims count ≥ options × axes?
  [ ] every verified_claim has a source reference with a path (not just a domain)?
  [ ] caution_claims: at least 1 per option, each with a specific URL?
  [ ] no claim appears in both verified_claims and caution_claims for the same product?
  [ ] every caution_claim covers something genuinely uncertain (plan/tier/region/date)?    
  [ ] decision_stage_questions: all four action types (choose/adopt/switch/validate) covered?
  Fix any failures before writing the JSON.

  # OUTPUT RULES
  - Return ONLY valid JSON. No preamble, no markdown fences, no trailing commentary.

output_schema:
  comparison_axes:           # 5–7 items; min 2 applied-context axes
    - string
  verified_claims:           # min: options × axes; each includes source in parentheses
    - string
  caution_claims:            # min 1 per option; each names specific uncertainty + URL
    - string
  decision_stage_questions:  # min 5; covers choose/adopt/switch/validate
    - string
  risk_summary: string       # 2–4 sentences; names a specific failure mode + concrete action
retry: 2
timeout: 45
next: W20_draft_writing
```

```yaml
## [W20] Draft Writing (revised by Claude:Sonnet 4.6 → v8_0_4)
id: W20_draft_writing
type: llm
model: gpt-5
temperature: 0.4
max_tokens: 9000
description: >
  Write a complete long-form draft in markdown focused purely on content quality.
  Fills every section with specific, grounded, opinionated writing.
  Tone/style polish is handled by E32_humanize. Internal links are handled by E33_link_inject.
  SEO fields are handled by E34_seo_fields.
input:
  # ── P02 ──
  primary_keyword:          "{{P02_keyword_selection.output.primary_keyword}}"
  angle:                    "{{P02_keyword_selection.output.article_angle}}"
  audience:                 "{{P02_keyword_selection.output.target_reader}}"
  search_intent:            "{{P02_keyword_selection.output.search_intent_summary}}"
  # ── R10 ──
  selected_title:           "{{R10_outline_generation.output.best_title}}"
  outline:                  "{{R10_outline_generation.output.outline}}"
  # ── R12: option data ──
  major_options:            "{{R12_option_research.output.major_options}}"
  # ── R13: comparison framework ──
  supporting_research:      "{{R13_claims_and_axes.output.verified_claims}}"
  caution_claims:           "{{R13_claims_and_axes.output.caution_claims}}"
  risk_summary:             "{{R13_claims_and_axes.output.risk_summary}}"
  comparison_axes:          "{{R13_claims_and_axes.output.comparison_axes}}"
  decision_stage_questions: "{{R13_claims_and_axes.output.decision_stage_questions}}"

prompt: |
  # ── GATE RULES — CHECK THESE FIRST, BEFORE AND AFTER WRITING ──
  # CHANGED: 집행 게이트를 프롬프트 최상단으로 이동. 분산된 세 곳(HARD REQUIREMENTS /
  #          WORD COUNT GATE / LENGTH AND DEPTH)을 단일 블록으로 통합.

  These rules are checked twice: once before writing (plan) and once after (verify).
  Violating any of them makes the entire output invalid.

  BEFORE WRITING — plan your word budget:
  - Target 2000–2500 words. 2000 is a hard floor.
  - Budget per section (minimums, not targets):
      TL;DR paragraph:        30–60 w
      Evaluation criteria:   150–200 w
      Comparison table:      100–150 w
      Product reviews total: 600–900 w  (≥ 120 w per H3)
      Scenario section:      200–300 w  (≥ 40 w per scenario)
      Setup / guidance:      100–150 w
      FAQ:                   200–300 w  (≥ 60 w per Q&A pair)
      Final verdict:          80–120 w
  - If your plan does not sum to 2000, expand H3 sections or scenario paragraphs
    BEFORE writing — not after.

  AFTER WRITING — verify before outputting JSON:
  1. Count words. If < 2000 → content_markdown: null, set error_reason, stop.
  2. Count "## " lines. Valid: 6–8. Any H2 matching the quick-answer pattern → demote to paragraph.
  3. Count "### " lines in the review section. Must equal number of major_options.
  4. Confirm pipe table exists (lines start with "|").
  5. Read final 3 lines. If any is a process note → delete it.
  If any check fails → fix content_markdown before outputting.

  FORBIDDEN OUTPUT PATTERNS (any of these = invalid output):
  - Process continuation notes anywhere in the article:
      "If you want, I'll now...", "Let me know if...", "Shall I proceed..."
  - Quick-answer as a standalone H2:
      "## TL;DR", "## Quick answer", "## Top picks", "## At a glance",
      "## In brief", "## Summary", "## Bottom line", or any H2 whose sole
      purpose is summarizing conclusions before the body sections begin.
  - H3 sections with no opening prose:
      A section that jumps directly from "### Product name" to a label
      ("Strengths", "Limitations") with no prose paragraph is invalid.
  - Scenario paragraphs that defer the pick:
      "Option A or B depending on budget" — always pick one and explain why.

  # POST STRATEGY
  - Primary keyword: {{primary_keyword}}
  - Angle: {{angle}}
  - Audience: {{audience}}
  - Search intent: {{search_intent}}
  - Exact title to use: {{selected_title}}

  # CONTENT INPUTS
  - Outline: {{outline}}
  - Supporting research (verified claims): {{supporting_research}}
  - Caution claims: {{caution_claims}}
  - Risk summary: {{risk_summary}}
  - Major options: {{major_options}}
  - Comparison axes: {{comparison_axes}}
  - Decision stage questions: {{decision_stage_questions}}

  # WRITING MANDATE
  Your only job is to fill this article with complete, specific, opinionated content.
  Do not think about tone polish, internal links, or SEO fields — those are handled downstream.

  Every paragraph must contain at least one of:
  - A specific product name with a concrete behavior, limitation, or trade-off
  - A real scenario drawn from major_options or verified_claims
  - A concrete recommendation with a named condition
  - A specific caution drawn from caution_claims or risk_summary

  # SECTION DEPTH REQUIREMENTS
  # CHANGED: STRUCTURE SELF-CHECK의 체크 로직과 CONTENT REQUIREMENTS의 작성 지침을
  #          단일 섹션으로 통합. 각 규칙이 "무엇을 써야 하는가"와 "어떻게 검증하는가"를
  #          한 곳에서 함께 서술.

  ## H1 and H2
  - Exactly one H1: {{selected_title}}
  - 6–8 H2 sections. TL;DR/quick-answer block = plain paragraph after H1, not an H2.
  - Primary keyword must appear naturally in at least one H2.

  ## TL;DR paragraph (not an H2)
  - Place immediately after H1 as a plain paragraph.
  - Name the top pick with a one-line reason. 30–60 words.

  ## Evaluation criteria H2
  - What actually matters when choosing — specific to this article's audience.
  - Not a generic list. Each criterion must connect to a real constraint
    the target reader faces.

  ## Comparison table H2
  - Pipe table format only. Columns = comparison_axes. One row per major_options item.
  - Every cell must be specific and differentiated. "Varies" or column-header repeats = invalid.

  ## Product reviews H2 — one H3 per product
  - All H3s under ONE H2. Never split into separate H2s per product.
  - Each H3 MUST contain:
      Opening prose ≥ 40 words: product positioning + one concrete operational
      scenario showing when this product fits (name team size, workflow, or stack).
      Then: strengths, limitations, best_for, not_ideal_for, trade-offs, caution.
      Caution items from caution_claims → write with explicit caveat + verification URL.
      Minimum 120 words per H3.

  ## Scenario recommendations H2
  - One paragraph per scenario. Each ≥ 40 words.
  - Name one pick per scenario. Never "A or B depending on budget."
  - Include at least one concrete reason from major_options data.

  ## Setup or guidance H2
  - Actionable steps specific to this article's context.
  - Reference the actual products where steps differ by tool.

  ## FAQ H2
  - Build from decision_stage_questions. One Q&A per question.
  - Each answer: ≥ 2 full sentences (two terminal punctuation marks, not two clauses
    joined by a semicolon). First sentence = pick or action. Second = concrete reason.
    If caution_claim is relevant, add a third sentence with the verification URL.

  ## Final verdict H2
  - Concrete pick per use case. Not vague ("it depends on your needs").
  - 80–120 words.

  # CAUTION CLAIM HANDLING
  Any claim that appears in caution_claims MUST be written with a caveat.
  FORBIDDEN: "Freddy can cut handle time significantly."
  REQUIRED:  "Freddy can reduce handle time on supported plans — verify the exact
              limits for your tier at freshdesk.com/pricing before relying on this."

  # OUTPUT FORMAT
  Return ONLY valid JSON — no markdown fences, no text outside the object:
  {
    "title": "string",
    "content_markdown": "string or null",
    "estimated_word_count": 0,
    "error_reason": "string or null"
  }
  - "title" must equal {{selected_title}} exactly.
  - "content_markdown": full article if word_count ≥ 2000; null otherwise.
  - "estimated_word_count": your actual counted integer. Differs from actual by > 10% = invalid.
  - "error_reason": null on success; one sentence on failure.

output_schema:
  type: object
  additionalProperties: false
  required:
    - title
    - content_markdown
    - estimated_word_count
    - error_reason
  properties:
    title:
      type: string
    content_markdown:
      type: ["string", "null"]   # null when word count gate fails
    estimated_word_count:
      type: integer
    error_reason:
      type: ["string", "null"]   # null on success; explanation on gate failure

on_error:
  check_field: content_markdown
  if_null:
    action: halt
    next: Z51_telegram_fail_report

retry: 2
timeout: 120
next: W21_save_raw_draft
```


```yaml
## [W21] Save Raw Draft
id: W21_save_raw_draft
type: shell
description: Save the full Step 6 markdown draft and word count to workspace files
input:
  title:                "{{W20_draft_writing.output.title}}"
  content_markdown:     "{{W20_draft_writing.output.content_markdown}}"
  estimated_word_count: "{{W20_draft_writing.output.estimated_word_count}}"
script: |
  set -euo pipefail
 
  WORKDIR="/home/openclaw/.openclaw/workspace"
  mkdir -p "$WORKDIR"
 
  cat > "$WORKDIR/W21_raw_draft.md" <<'EOF'
{{inputs.content_markdown}}
EOF
 
  cat > "$WORKDIR/W21_raw_draft_title.txt" <<'EOF'
{{inputs.title}}
EOF
 
  # estimated_word_count 저장 — step11 텔레그램 리포트에서 활용 가능
  printf '%s' "{{inputs.estimated_word_count}}" > "$WORKDIR/W21_word_count.txt"
 
next: E30_load_raw_draft
```

```yaml
## [E30] Load Raw Draft
id: E30_load_raw_draft
type: shell
description: Load the saved raw Step 6 draft file before editorial polish
output_schema:
  raw_draft_content: string
  raw_draft_title: string
script: |
  set -euo pipefail

  WORKDIR="/home/openclaw/.openclaw/workspace"
  DRAFT_FILE="$WORKDIR/W21_raw_draft.md"
  TITLE_FILE="$WORKDIR/W21_raw_draft_title.txt"

  test -s "$DRAFT_FILE"
  test -s "$TITLE_FILE"

  python3 - <<'PY'
  import json
  from pathlib import Path

  workdir = Path("/home/openclaw/.openclaw/workspace")
  draft = (workdir / "W21_raw_draft.md").read_text(encoding="utf-8")
  title = (workdir / "W21_raw_draft_title.txt").read_text(encoding="utf-8").strip()

  print(json.dumps({
      "raw_draft_content": draft,
      "raw_draft_title": title
  }, ensure_ascii=False))
  PY
next: E31_editorial_review
```

```yaml
## [E31] Editorial Review (revised by Claude:Sonnet 4.6 → v8_0_4)
# CHANGES FROM v8_0_3:
#   1. major_options_count input 추가 — D6에서 H3 수 검증에 사용
#   2. D6: pipe table 존재 여부 + H3 per-product 수 검증 추가
#   3. Hard reject 트리거 4개 → 5개 (pipe table 없음 또는 H3 수 불일치 추가)
#   4. D1: estimated_word_count 복사 금지 강화
#   5. scores 하위 required 필드 명시
#   6. max_tokens: 1200 → 1800
id: E31_editorial_review
type: llm
model: gpt-5-mini
max_tokens: 1800
description: >
  Score the W20 draft on 6 quality dimensions and make a pass/fail decision.
  If any hard reject trigger fires, output verdict=reject with a diagnostic report
  so the pipeline fails fast and W20 can be manually fixed before re-run.
  Does NOT rewrite any content.
input:
  draft_content_markdown:   "{{E30_load_raw_draft.output.raw_draft_content}}"
  raw_draft_title:          "{{E30_load_raw_draft.output.raw_draft_title}}"
  estimated_word_count:     "{{W20_draft_writing.output.estimated_word_count}}"
  primary_keyword:          "{{P02_keyword_selection.output.primary_keyword}}"
  caution_claims:           "{{R13_claims_and_axes.output.caution_claims}}"
  major_options_count:      "{{R12_option_research.output.major_options | length}}"

prompt: |
  You are a senior editorial reviewer. Your only job is to score this draft and
  decide whether it is ready to proceed to humanization and publication.
  Do NOT rewrite anything. Score and judge only.

  # DRAFT METADATA
  - Primary keyword: {{primary_keyword}}
  - Estimated word count from W20 (reference only — do NOT copy this value): {{estimated_word_count}}
  - Expected number of product options: {{major_options_count}}
  - Caution claims (claims R13 flagged as unverified): {{caution_claims}}

  # DRAFT CONTENT
  {{draft_content_markdown}}

  ---

  # SCORING — 6 DIMENSIONS
  Score each dimension 1–5. Record raw counts where required.

  ## D1 — Word count (1–5)
  IMPORTANT: Count words in draft_content_markdown yourself independently.
  Do NOT copy or trust estimated_word_count from W20 — it may be inaccurate.
  Method: count every whitespace-separated token in the full markdown text.
  Record your independent count as actual_word_count.
  - 5: ≥ 2000 words
  - 4: 1800–1999
  - 3: 1600–1799
  - 2: 1500–1599
  - 1: < 1500 words
  HARD REJECT if actual_word_count < 2000.

  ## D2 — H1 integrity (1–5)
  - 5: exactly one H1 present, matches raw_draft_title
  - 3: one H1 present but does not match raw_draft_title
  - 1: zero H1 or more than one H1
  HARD REJECT if h1_count ≠ 1.

  ## D3 — AI pattern density (1–5)
  Count occurrences of these exact phrases (case-insensitive) in the body text:
  "Furthermore", "Additionally", "Moreover", "In conclusion", "It is worth noting",
  "It is important to note", "In summary", "To summarize", "In this article",
  "This article will", "Needless to say", "At the end of the day",
  "In the world of", "It goes without saying".
  - 5: 0–1 occurrences total
  - 4: 2–3
  - 3: 4
  - 2: 5
  - 1: ≥ 6
  HARD REJECT if ai_pattern_count ≥ 5.
  Record the exact count and list the offending phrases found.

  ## D4 — Section intro repetition (1–5)
  A section intro is "repetitive" when the first 1–2 sentences of an H2 section
  simply restate the section heading or paraphrase the prior section's conclusion
  without adding new information.
  Count how many H2 sections have a repetitive intro.
  - 5: 0 repetitive intros
  - 4: 1
  - 3: 2
  - 2: 3
  - 1: ≥ 4
  HARD REJECT if repetitive_intro_count ≥ 3.
  List the specific section headings with repetitive intros.

  ## D5 — Factual claim safety (1–5)
  Cross-reference the draft body against caution_claims (claims R13 already flagged
  as unverified). Check whether the draft presents those claims as established facts
  rather than caveated statements. Also flag any other suspicious statistics,
  pricing figures, or product capability claims not in caution_claims.
  - 5: All caution_claims are written cautiously in the draft; no other suspicious claims
  - 3: 1–2 caution_claims asserted as facts, or 1–2 other suspicious claims
  - 1: 3+ caution_claims asserted as facts, or 3+ suspicious claims
  List each suspicious claim and note whether it came from caution_claims.

  ## D6 — Structural completeness (1–5)
  Check that ALL of the following are present:

  1. TL;DR or quick-answer block (as a paragraph — NOT a standalone H2)
  2. Comparison table in pipe format — lines must start with "|" and include a
     "---" separator row. A bullet list does NOT count as a comparison table.
  3. Option-by-option H3 sections — count H3 headings in the product review area.
     The count must equal major_options_count. ALL H3s must sit under ONE H2,
     not spread across multiple H2s per product.
  4. Scenario recommendations section
  5. FAQ section
  6. Final verdict or conclusion

  Score:
  - 5: All 6 elements present AND pipe table confirmed AND H3 count == major_options_count
  - 4: 5 of 6 elements present
  - 3: 4 of 6
  - 2: 3 of 6
  - 1: ≤ 2 of 6

  HARD REJECT if:
  - pipe table is absent (only bullets found), OR
  - H3 count in the review section does not equal major_options_count

  List any missing or non-conforming elements.

  ---

  # HARD REJECT TRIGGERS — check all five
  REJECT if ANY of the following are true:
  1. actual_word_count < 2000          (D1 ≤ 4)
  2. h1_count ≠ 1                      (D2 = 1)
  3. ai_pattern_count ≥ 5             (D3 ≤ 2)
  4. repetitive_intro_count ≥ 3       (D4 ≤ 2)
  5. comparison table absent or not in pipe format, OR H3 product count ≠ major_options_count  (D6)

  If ANY trigger fires → verdict = "reject"
  If NO trigger fires  → verdict = "pass"

  A "reject" verdict means this pipeline run ends here. The operator must
  fix W20 output and re-run. Do NOT proceed to humanization.

  ---

  # SELF-CHECK before writing JSON
  [ ] Did I count actual words in the markdown myself, NOT copy estimated_word_count?
  [ ] Did I count H1 headings (lines starting with exactly "# ")?
  [ ] Did I check for pipe table syntax (lines starting with "|")?
  [ ] Did I count H3s in the product review section and compare to major_options_count?
  [ ] Did I list every offending AI phrase found, not just the count?
  [ ] Did I list each repetitive section intro heading by name?
  [ ] Is verdict exactly "pass" or "reject" — no other values?
  [ ] Are all required output fields present, including scores with all 6 sub-fields?

  # OUTPUT RULES
  - Return ONLY valid JSON. No preamble, no markdown fences, no trailing commentary.

output_schema:
  type: object
  additionalProperties: false
  required:
    - verdict
    - actual_word_count
    - h1_count
    - ai_pattern_count
    - ai_patterns_found
    - repetitive_intro_count
    - repetitive_intro_sections
    - scores
    - total_score
    - reject_reasons
    - suspicious_claims
    - missing_structural_elements
    - review_summary
  properties:
    verdict:
      type: string           # "pass" | "reject" — no other values
    actual_word_count:
      type: integer          # independently counted; never copied from estimated_word_count
    h1_count:
      type: integer
    ai_pattern_count:
      type: integer
    ai_patterns_found:
      type: array
      items:
        type: string         # e.g. "Furthermore (3×)", "Moreover (1×)"
    repetitive_intro_count:
      type: integer
    repetitive_intro_sections:
      type: array
      items:
        type: string         # heading text of each repetitive-intro section
    scores:
      type: object
      required:
        - d1_word_count
        - d2_h1_integrity
        - d3_ai_pattern_density
        - d4_section_intro_rep
        - d5_factual_safety
        - d6_structural_completeness
      properties:
        d1_word_count:              { type: integer }
        d2_h1_integrity:            { type: integer }
        d3_ai_pattern_density:      { type: integer }
        d4_section_intro_rep:       { type: integer }
        d5_factual_safety:          { type: integer }
        d6_structural_completeness: { type: integer }
    total_score:
      type: integer          # sum of 6 dimension scores (max 30)
    reject_reasons:
      type: array
      items:
        type: string         # one sentence per triggered reject rule; empty array if verdict=pass
    suspicious_claims:
      type: array
      items:
        type: string         # each suspicious claim; empty array if none found
    missing_structural_elements:
      type: array
      items:
        type: string         # e.g. "pipe table", "H3 for Intercom", "FAQ section"
    review_summary:
      type: string           # 2–4 sentences: overall quality + key issues to fix in humanization

on_reject:
  action: continue
  next: Z51_telegram_fail_report

retry: 1
timeout: 60
next: E32_humanize
```


```yaml
## [E32] Humanize
id: E32_humanize
type: llm
model: gpt-5
temperature: 0.65
max_tokens: 9000
description: >
  Rewrite the draft to sound like a skilled human writer:
  natural sentence rhythm, no AI filler phrases, engaging section transitions,
  and a direct second-person tone that speaks to the reader.
  Preserves all factual content, structure, and SEO intent from Step 6.
  Does NOT generate SEO fields (title, meta, slug, excerpt) — those come in Step 7c.
input:
  draft_content_markdown:    "{{E30_load_raw_draft.output.raw_draft_content}}"
  raw_draft_title:           "{{E30_load_raw_draft.output.raw_draft_title}}"
  primary_keyword:           "{{P02_keyword_selection.output.primary_keyword}}"
  chosen_content_type:       "{{P02_keyword_selection.output.chosen_content_type}}"
  target_reader:             "{{P02_keyword_selection.output.target_reader}}"
  review_summary:            "{{E31_editorial_review.output.review_summary}}"
  ai_patterns_found:         "{{E31_editorial_review.output.ai_patterns_found}}"
  repetitive_intro_sections: "{{E31_editorial_review.output.repetitive_intro_sections}}"
  suspicious_claims:         "{{E31_editorial_review.output.suspicious_claims}}"
  verified_claims:           "{{R13_claims_and_axes.output.verified_claims}}"
  caution_claims:            "{{R13_claims_and_axes.output.caution_claims}}"

prompt: |
  You are a skilled human editor with a background in tech journalism.
  Your job is to make this draft read like a person wrote it — not a language model.

  # CONTEXT
  - Primary keyword: {{primary_keyword}}
  - Content type: {{chosen_content_type}}
  - Target reader: {{target_reader}}
  - Step 7a review summary: {{review_summary}}
  - AI patterns to eliminate: {{ai_patterns_found}}
  - Sections with repetitive intros to fix: {{repetitive_intro_sections}}
  - Suspicious claims to soften or remove: {{suspicious_claims}}

  # DRAFT TO HUMANIZE
  {{draft_content_markdown}}

  ---

  # HUMANIZATION MANDATE — THE FOUR PRINCIPLES

  ## PRINCIPLE 1 — Sentence rhythm and breath
  Real writers vary sentence length deliberately. Long sentences build up context
  or explain a nuanced idea. Short sentences land a point. Very short ones punch.
  - Break up any run of 3+ consecutive sentences of similar length.
  - After a long explanatory sentence, follow with a short declarative one.
  - Avoid sentences that start with the same grammatical structure in a row
    (e.g. "This allows...", "This ensures...", "This means...").
  - Avoid noun-heavy pileups: "the process of the configuration of the settings"
    → "how to configure the settings" or just "configuring the settings".
  - Vary paragraph length. A 1-sentence paragraph after a long one creates emphasis.

  ## PRINCIPLE 2 — Eliminate AI filler and transition clichés
  Every phrase in ai_patterns_found must be removed or replaced.
  Additionally, hunt for and remove any of these patterns even if not in the list:
  - "It is worth noting that..." → cut entirely, start with the point
  - "Furthermore, X also Y" → "X also Y" or restructure
  - "In today's digital landscape" / "In the modern era of" → delete, start with the actual claim
  - "It goes without saying" → delete, say the thing
  - "At the end of the day" → delete or replace with "Ultimately" if needed
  - "needless to say" → delete
  - "In this comprehensive guide" / "In this article we will" → delete or rephrase
  - Opening H2 sections with "Now that we've covered X, let's turn to Y" → rewrite the intro
    so it opens with the new section's point, not a reference to the previous section
  - Closing paragraphs that summarize every point just covered → cut or condense to one insight

  ## PRINCIPLE 3 — Fix section intros and outros
  The sections listed in repetitive_intro_sections have intros that restate
  the heading or summarize the prior section. For each one:
  - Replace the intro with the most interesting specific claim or insight from that section.
  - The first sentence of any H2 section should make the reader want to continue,
    not confirm what they already know from the heading.
  - Section outros (last 1–2 sentences before the next H2): do not summarize what was
    just covered. Either end on a concrete recommendation, a useful caveat, or
    a forward-looking sentence that creates natural momentum to the next section.
    If there's nothing useful to add, end the section's last paragraph naturally
    without a dedicated outro sentence.

  ## PRINCIPLE 4 — Second-person, direct tone
  Write as if explaining to a smart colleague, not addressing an abstract audience.
  - Use "you" and "your" freely: "If your WooCommerce store processes over 500 orders
    a day, this matters more than you might think."
  - Give direct recommendations: "Start with UpdraftPlus if you're on a budget.
    Switch to BlogVault when you need guaranteed restore reliability."
  - Avoid corporate hedging: "users may wish to consider" → "consider this if..."
  - Avoid passive voice as the default. Use it only when the actor is genuinely unknown
    or irrelevant.
  - Let opinion show in recommendations: "Honestly, the free tier isn't worth it here."
    This is a review-style article — a clear recommendation voice is appropriate.

  ---

  # WHAT YOU MUST NOT CHANGE
  - Do NOT alter the article's H2/H3 structure or section order.
  - Do NOT remove comparison tables, option reviews, FAQ items, or the final verdict.
  - Do NOT invent new facts, statistics, prices, or product capabilities.
  - Do NOT soften or remove verified_claims — only soften items in suspicious_claims.
  - Do NOT add new sections or new affiliate products not in the original draft.
  - Do NOT change the H1 title — it must stay exactly as {{raw_draft_title}}.
  - Do NOT generate polished_title, meta_description, slug, or excerpt here.
    Those come in Step 7c.
  - The output field is named humanized_content_markdown.
    The H1 must still be "# {{raw_draft_title}}" exactly.

  # CONTENT PRESERVATION — HARD RULES
  These apply regardless of how you judge the original writing quality:

  - Do NOT remove or merge the per-option structural beats under each H3.
    Every option section must keep all of these beats (wording may change):
    Strengths/strength claims, Limitations, Best for, Not ideal for,
    Operational trade-off, Caution. You may rewrite each beat as flowing prose
    rather than a labeled sub-heading, but every beat must remain present
    with its substance intact.
  - Do NOT compress scenario recommendation items. Each scenario must keep
    its condition ("Small remote startup", "Growing product org", etc.)
    AND its explanation of why that tool fits — not just the tool name.
  - Do NOT reduce the Migration section. If the original has numbered steps
    or a checklist, keep every step. You may rewrite the wording, but the
    step count must not decrease.
  - Do NOT shorten FAQ answers. Each answer must remain substantive
    (2–4 sentences minimum). Cutting to a single sentence is not allowed.
  - Do NOT remove the "Why:" or equivalent rationale from any recommendation.
    If the original explains the reason, the humanized version must too.

  ---

  # QUALITY GATE — self-check before writing JSON
  Before outputting, verify every item. If any check fails, fix it before proceeding.

  TONE CHECKS:
  [ ] Every phrase from ai_patterns_found has been removed or replaced.
  [ ] Every section listed in repetitive_intro_sections has a rewritten intro.
  [ ] No section outro just lists what was covered in that section.
  [ ] No 3+ consecutive sentences of the same approximate length remain.
  [ ] "you" / "your" appears at least once per H2 section.
  [ ] Direct recommendations are present in option reviews and final verdict.

  CONTENT PRESERVATION CHECKS:
  [ ] Every H3 option section still contains all six beats: strength, limitation,
      best_for, not_ideal_for, operational trade-off, caution — even if rewritten as prose.
      → If any beat is missing: restore it from the original draft before proceeding.
  [ ] Every scenario recommendation still names the condition AND explains why.
      → If any "Why:" rationale was dropped: restore it.
  [ ] Migration section has the same number of steps as the original draft.
      → If any step was removed: restore it.
  [ ] Every FAQ answer is 2–4 sentences. No one-sentence answers.
      → If any answer was compressed to one sentence: expand it.
  [ ] Word count of humanized_content_markdown is within 10% of the input draft.
      Count words in both. If output is more than 10% shorter than input:
      expand thin recommendations, restore dropped rationale, or add one concrete
      example per compressed section until word count is within range.

  INTEGRITY CHECKS:
  [ ] H1 is exactly "# {{raw_draft_title}}" — no other H1 exists.
  [ ] No new facts, prices, or products have been introduced.
  [ ] No section or H3 was removed entirely.

  # OUTPUT RULES
  - Return ONLY valid JSON. No preamble, no markdown fences, no trailing commentary.
  - humanized_content_markdown MUST contain the FULL rewritten article text as a string value.
  - NEVER put a file path, filename, or workspace path in humanized_content_markdown.
  - NEVER output a field named humanized_content_markdown_path — this field does not exist.
  - The value of humanized_content_markdown must begin with "# {{raw_draft_title}}" and
    contain every section of the article as markdown text, not a reference to a file.

  INVALID example (do not do this):
  {
    "humanized_content_markdown": "/home/openclaw/.openclaw/workspace/W21_raw_draft.md",
    "humanization_notes": "..."
  }

  VALID example (do this):
  {
    "humanized_content_markdown": "# Best AI customer support tools\n\nTL;DR: ...(full article)...",
    "humanization_notes": "..."
  }

output_schema:
  type: object
  additionalProperties: false
  required:
    - humanized_content_markdown
    - humanization_notes
  properties:
    humanized_content_markdown:
      type: string   # FULL markdown article text — NOT a file path. Must start with "# {title}".
    humanization_notes:
      type: string   # 3–5 sentences: what was changed and why; useful for audit

retry: 2
timeout: 180
next: E33_link_inject
```

```yaml
## [E33] Link Inject
id: E33_link_inject
type: llm
model: gpt-5-mini
temperature: 0.2
max_tokens: 9000
description: >
  Insert internal links into the humanized draft using the internal_link_plan from step4.
  Does not rewrite content, adjust tone, or alter structure.
  Only inserts markdown hyperlinks at the positions specified in placement_note.
  Passes the linked draft downstream as linked_content_markdown.
input:
  humanized_content_markdown: "{{E32_humanize.output.humanized_content_markdown}}"
  raw_draft_title:            "{{E30_load_raw_draft.output.raw_draft_title}}"
  internal_link_plan:         "{{R11_internal_link_planning.output.internal_link_plan}}"

prompt: |
  # INTERNAL LINK GUARD
    - If R11_internal_link_planning.output.no_existing_posts is true,
      or if R11_internal_link_planning.output.internal_link_plan is an empty array,
      return the draft unchanged. Do not insert any links.
      Set link_inject_skipped: true in the output.

  You are a precise markdown editor. Your only job is to insert internal hyperlinks
  into the article exactly as specified. Do not change any other content.

  # ARTICLE (humanized)
  {{humanized_content_markdown}}

  # LINK PLAN
  {{internal_link_plan}}

  # INSERTION RULES
  - If internal_link_plan is empty ([]), return the article unchanged.
  - For each entry in internal_link_plan:
    - Find the section named in target_section.
    - Within that section, locate the position described in placement_note.
    - Wrap suggested_anchor_text in a markdown hyperlink: [suggested_anchor_text](target_url)
    - The anchor text must appear verbatim in the article — do not insert a link if the
      exact anchor phrase is not present. Skip that entry rather than forcing it.
    - Insert each link only once. Do not duplicate links.
  - Do not alter any other text, punctuation, heading, or structure.
  - Do not add new sentences or remove existing ones.
  - The H1 must remain exactly "# {{raw_draft_title}}".

  # QUALITY GATE — self-check before writing JSON
  [ ] Every link in internal_link_plan that had a matching anchor phrase has been inserted.
  [ ] No link was inserted more than once.
  [ ] No text outside the linked anchor phrases was changed.
  [ ] H1 is still exactly "# {{raw_draft_title}}".
  [ ] Word count is not meaningfully different from the input article.

  # OUTPUT FORMAT
  Return ONLY valid JSON. No preamble, no markdown fences, no trailing commentary.
  {
    "linked_content_markdown": "string",
    "link_inject_notes": "string"
  }

  - "linked_content_markdown": full article markdown with links inserted.
  - "link_inject_notes": 1–3 sentences — how many links were inserted, which (if any) were
    skipped and why. Used for audit.

output_schema:
  type: object
  additionalProperties: false
  required:
    - linked_content_markdown
    - link_inject_notes
  properties:
    linked_content_markdown:
      type: string
    link_inject_notes:
      type: string
retry: 2
timeout: 60
next: E34_seo_fields
```

```yaml
## [E34] SEO Fields (revised by Claude:Sonnet 4.6 → v8_0_4)
# CHANGES FROM v8_0_3:
#   1. focus_keyword: full primary_keyword 구문 우선, 단축 시 focus_keyword_note 필드에 사유 기재
#   2. meta_description: 135-150자 hard target, 미달/초과 시 expand/trim 의무화
#   3. SELF-CHECK에 글자 수 카운트 항목 추가
#   4. output_schema에 focus_keyword_note 필드 추가
id: E34_seo_fields
type: llm
model: gpt-5-mini
max_tokens: 800
description: >
  Generate the final polished title, meta description, focus keyword, excerpt,
  and slug from the linked draft (humanized + internal links inserted).
  Reads linked_content_markdown as source of truth.
  This is the only step that produces SEO output fields.
input:
  linked_content_markdown:       "{{E33_link_inject.output.linked_content_markdown}}"
  raw_draft_title:               "{{E30_load_raw_draft.output.raw_draft_title}}"
  primary_keyword:               "{{P02_keyword_selection.output.primary_keyword}}"
  provisional_meta_description:  "{{R10_outline_generation.output.provisional_meta_description}}"
  provisional_slug:              "{{R10_outline_generation.output.provisional_slug}}"

prompt: |
  Generate the final SEO fields for this article.
  The article body has already been humanized and finalized.
  Your job is ONLY to produce the metadata fields — do not rewrite the body.

  # INPUTS
  - Raw draft title (H1 in article): {{raw_draft_title}}
  - Primary keyword: {{primary_keyword}}
  - Provisional meta description (reference only): {{provisional_meta_description}}
  - Provisional slug (reference only): {{provisional_slug}}

  # ARTICLE BODY (humanized + links inserted)
  {{linked_content_markdown}}

  ---

  # TITLE RULES (polished_title)
  - Start from raw_draft_title.
  - Keep it if it is already 54 characters or fewer and keyword-led.
  - If it exceeds 54 characters: shorten by removing non-essential qualifiers.
    Preserve the core keyword and search intent. Do not change meaning.
  - Target 42–54 characters.
  - Put the core keyword near the front.
  - No colons, em dashes, or subtitle-style expansions.
  - If two phrasings express the same intent, choose the shorter one.
  - polished_title must match the H1 in linked_content_markdown exactly,
    or be a shortened version. Flag any shortening in title_change_note.

  # META DESCRIPTION RULES (polished_meta_description)
  - Plain text only. No markdown. No line breaks.
  - TARGET LENGTH: 135–150 characters. This is a hard target range, not a guideline.
    Before outputting, count the characters in your draft meta description:
    If under 135 characters: expand with a specific benefit, scenario, or named product.
    If over 150 characters: trim filler words until within range.
    Do not output a meta description outside the 135–150 range.
  - Must summarize the post clearly for a search user scanning results.
  - Must NOT repeat the title verbatim.
  - Must NOT begin with "#", "TL;DR", "-", or a heading fragment.
  - Draw from the article's opening, TL;DR, or final verdict — not from thin air.

  # FOCUS KEYWORD RULES (focus_keyword)
  - Must be the closest natural match to primary_keyword that appears in the article body.
  - Prefer the full primary_keyword phrase over a shortened subset.
    Example: primary_keyword = "best ai helpdesk software for small businesses"
    → CORRECT:   "best ai helpdesk software for small businesses"  (full phrase)
    → INCORRECT: "ai helpdesk software"  (loses differentiating modifiers)
  - Only shorten if the full phrase does not appear naturally in the humanized body.
    If shortened, record the reason in focus_keyword_note.
  - Lowercase. No special characters except hyphens.
  - 2–6 words. Fewer than 4 words is usually a sign the phrase is too generic.

  # EXCERPT RULES (excerpt)
  - Plain text only. No markdown. No line breaks.
  - 1–2 concise sentences.
  - Neutral summary suitable for archive pages or RSS previews.
  - Must NOT be identical to polished_meta_description.
  - Must NOT begin with "In this article" or "This post will".

  # SLUG RULES (slug)
  - Lowercase letters, numbers, hyphens only.
  - Keyword-relevant and concise.
  - No stopword chains. No full-title verbatim copy.
  - Start from provisional_slug; shorten or improve only if clearly needed.

  ---

  # SELF-CHECK before writing JSON
  [ ] polished_title is 42–54 characters? Count each character carefully.
  [ ] polished_meta_description is 135–150 characters? Count each character.
      If under 135: expand. If over 150: trim. Do not output outside this range.
  [ ] polished_meta_description does not start with the title or "#"?
  [ ] focus_keyword is the full primary_keyword or the longest natural match in the body?
  [ ] focus_keyword_note explains "full phrase used" or states reason for shortening?
  [ ] excerpt is different from polished_meta_description?
  [ ] slug contains only lowercase letters, numbers, and hyphens?
  [ ] focus_keyword appears in the humanized article body?

  # OUTPUT RULES
  - Return ONLY valid JSON. No preamble, no markdown fences, no trailing commentary.

output_schema:
  type: object
  additionalProperties: false
  required:
    - polished_title
    - polished_meta_description
    - focus_keyword
    - focus_keyword_note
    - excerpt
    - slug
    - title_change_note
  properties:
    polished_title:
      type: string          # 42–54 chars; keyword-led; no colon/em-dash
    polished_meta_description:
      type: string          # 135–150 chars; plain text; not a repeat of the title
    focus_keyword:
      type: string          # lowercase; 2–6 words; full primary_keyword preferred
    focus_keyword_note:
      type: string          # "full phrase used" if kept; else reason for shortening
    excerpt:
      type: string          # 1–2 sentences; plain text; distinct from meta description
    slug:
      type: string          # lowercase + hyphens only; keyword-relevant
    title_change_note:
      type: string          # "unchanged" if title kept; else what was shortened and why

retry: 2
timeout: 30
next: E35_save_polished_draft
```


```yaml
## [E35] Save Polished Draft
id: E35_save_polished_draft
type: shell
description: >-
  Save the raw linked draft (E33) and polished title (E34) to workspace as audit files.
  H1 sync is intentionally NOT done here — it is applied in E36 to E36_content.md,
  which is the actual publish path.
input:
  polished_title: "{{E34_seo_fields.output.polished_title}}"
  # E33 raw body — E37 will apply the same H1 sync as E36 to build expected content
  polished_content_markdown: "{{E33_link_inject.output.linked_content_markdown}}"
script: |
  set -euo pipefail

  WORKDIR="/home/openclaw/.openclaw/workspace"
  mkdir -p "$WORKDIR"

  cat > "$WORKDIR/E35_audit_draft.md" <<'EOF'
{{inputs.polished_content_markdown}}
EOF

  cat > "$WORKDIR/E35_audit_title.txt" <<'EOF'
{{inputs.polished_title}}
EOF
next: E36_export_publish_files
```

```yaml
## [E36] Export Publish Files
id: E36_export_publish_files
type: shell
description: Export final publish assets to workspace files for publish script
input:
  polished_title: "{{E34_seo_fields.output.polished_title}}"
  polished_slug: "{{E34_seo_fields.output.slug}}"
  polished_content_markdown: "{{E33_link_inject.output.linked_content_markdown}}"
  polished_meta_description: "{{E34_seo_fields.output.polished_meta_description}}"
  focus_keyword: "{{E34_seo_fields.output.focus_keyword}}"
  excerpt: "{{E34_seo_fields.output.excerpt}}"
  category: "{{R10_outline_generation.output.category}}"
  tags: "{{R10_outline_generation.output.tags}}"
script: |
  rm -f /home/openclaw/.openclaw/workspace/E36_title.txt
  rm -f /home/openclaw/.openclaw/workspace/E36_slug.txt
  rm -f /home/openclaw/.openclaw/workspace/E36_content.md
  rm -f /home/openclaw/.openclaw/workspace/E36_meta_description.txt
  rm -f /home/openclaw/.openclaw/workspace/E36_focus_keyword.txt
  rm -f /home/openclaw/.openclaw/workspace/E36_excerpt.txt
  rm -f /home/openclaw/.openclaw/workspace/E36_category.txt
  rm -f /home/openclaw/.openclaw/workspace/E36_tags.json
  rm -f /home/openclaw/.openclaw/workspace/E37_validation_report.txt
  rm -f /home/openclaw/.openclaw/workspace/E36_content.html

  mkdir -p /home/openclaw/.openclaw/workspace

  cat > /home/openclaw/.openclaw/workspace/E36_title.txt <<'EOF'
{{inputs.polished_title}}
EOF

  cat > /home/openclaw/.openclaw/workspace/E36_slug.txt <<'EOF'
{{inputs.polished_slug}}
EOF

  # Write body and replace H1 with polished_title so E36_content.md H1 == title.
  # E33 preserves raw_draft_title in H1; this is the single point where sync is applied.
  # Python is used instead of sed to safely handle /, &, \ in the title.
  cat > /home/openclaw/.openclaw/workspace/E36_content.md <<'EOF'
{{inputs.polished_content_markdown}}
EOF
  TITLE='{{inputs.polished_title}}' \
  TARGET_FILE='/home/openclaw/.openclaw/workspace/E36_content.md' \
  python3 - <<'PY'
import os, pathlib
p = pathlib.Path(os.environ['TARGET_FILE'])
title = os.environ['TITLE']
lines = p.read_text(encoding='utf-8').splitlines(keepends=True)
if lines and lines[0].startswith('# '):
    lines[0] = f'# {title}\n'
p.write_text(''.join(lines), encoding='utf-8')
PY

  cat > /home/openclaw/.openclaw/workspace/E36_meta_description.txt <<'EOF'
{{inputs.polished_meta_description}}
EOF

  cat > /home/openclaw/.openclaw/workspace/E36_focus_keyword.txt <<'EOF'
{{inputs.focus_keyword}}
EOF

  cat > /home/openclaw/.openclaw/workspace/E36_excerpt.txt <<'EOF'
{{inputs.excerpt}}
EOF

  cat > /home/openclaw/.openclaw/workspace/E36_category.txt <<'EOF'
{{inputs.category}}
EOF

  cat > /home/openclaw/.openclaw/workspace/E36_tags.json <<'EOF'
{{inputs.tags}}
EOF
next: E37_artifact_validation
```

```yaml
## [E37] Artifact Validation (revised by Claude:Sonnet 4.6 → v8_0_4)
# CHANGES FROM v8_0_3:
#   1. stub 패턴 감지 추가 — "(Full content for publish)" 등 placeholder 텍스트 error 처리
#   2. pipe table 존재 확인 추가 — 3줄 미만이면 error
#   3. H3 count == 0이면 error
#   4. word_count 최소 기준 1500 → 1800으로 상향
#   5. meta_description 길이 경고 범위 조정 (< 135 경고 추가)
id: E37_artifact_validation
type: shell
description: Validate exported publish artifacts against step outputs and article quality rules before publish
input:
  polished_title:              "{{E34_seo_fields.output.polished_title}}"
  polished_slug:               "{{E34_seo_fields.output.slug}}"
  polished_content_markdown:   "{{E33_link_inject.output.linked_content_markdown}}"
  polished_meta_description:   "{{E34_seo_fields.output.polished_meta_description}}"
  focus_keyword:               "{{E34_seo_fields.output.focus_keyword}}"
  excerpt:                     "{{E34_seo_fields.output.excerpt}}"
  category:                    "{{R10_outline_generation.output.category}}"
  tags:                        "{{R10_outline_generation.output.tags}}"
script: |
  set -euo pipefail

  WORKDIR="/home/openclaw/.openclaw/workspace"
  REPORT="$WORKDIR/E37_validation_report.txt"

  TITLE_FILE="$WORKDIR/E36_title.txt"
  SLUG_FILE="$WORKDIR/E36_slug.txt"
  CONTENT_FILE="$WORKDIR/E36_content.md"
  META_FILE="$WORKDIR/E36_meta_description.txt"
  FOCUS_FILE="$WORKDIR/E36_focus_keyword.txt"
  EXCERPT_FILE="$WORKDIR/E36_excerpt.txt"
  CATEGORY_FILE="$WORKDIR/E36_category.txt"
  TAGS_FILE="$WORKDIR/E36_tags.json"

  EXPECTED_TITLE='{{inputs.polished_title}}'
  EXPECTED_SLUG='{{inputs.polished_slug}}'
  EXPECTED_META='{{inputs.polished_meta_description}}'
  EXPECTED_FOCUS='{{inputs.focus_keyword}}'
  EXPECTED_EXCERPT='{{inputs.excerpt}}'
  EXPECTED_CATEGORY='{{inputs.category}}'
  EXPECTED_TAGS_RAW='{{inputs.tags}}'

  mkdir -p "$WORKDIR"

  : > "$REPORT"
  echo "Publish validation report" >> "$REPORT"
  echo "=========================" >> "$REPORT"

  # ── 1. Artifact presence check ──────────────────────────
  for f in \
    "$TITLE_FILE" \
    "$SLUG_FILE" \
    "$CONTENT_FILE" \
    "$META_FILE" \
    "$FOCUS_FILE" \
    "$EXCERPT_FILE" \
    "$CATEGORY_FILE" \
    "$TAGS_FILE"
  do
    if [ ! -s "$f" ]; then
      echo "ERROR: missing or empty exported file: $f" >> "$REPORT"
      cat "$REPORT"
      exit 1
    fi
  done

  test -s /home/openclaw/.openclaw/workspace/W21_raw_draft.md
  test -s /home/openclaw/.openclaw/workspace/E35_audit_draft.md
  echo "raw_draft_files_present=yes" >> "$REPORT"

  # ── 2. Field match checks ────────────────────────────────
  ACTUAL_TITLE="$(cat "$TITLE_FILE" 2>/dev/null || true)"
  ACTUAL_SLUG="$(cat "$SLUG_FILE" 2>/dev/null || true)"
  ACTUAL_META="$(cat "$META_FILE" 2>/dev/null || true)"
  ACTUAL_FOCUS="$(cat "$FOCUS_FILE" 2>/dev/null || true)"
  ACTUAL_EXCERPT="$(cat "$EXCERPT_FILE" 2>/dev/null || true)"
  ACTUAL_CATEGORY="$(cat "$CATEGORY_FILE" 2>/dev/null || true)"

  if [ "$ACTUAL_TITLE" != "$EXPECTED_TITLE" ]; then
    echo "ERROR: exported title does not match E34 polished_title" >> "$REPORT"
    echo "EXPECTED_TITLE=$EXPECTED_TITLE" >> "$REPORT"
    echo "ACTUAL_TITLE=$ACTUAL_TITLE" >> "$REPORT"
    cat "$REPORT"; exit 1
  fi

  if [ "$ACTUAL_SLUG" != "$EXPECTED_SLUG" ]; then
    echo "ERROR: exported slug does not match E34 slug" >> "$REPORT"
    echo "EXPECTED_SLUG=$EXPECTED_SLUG" >> "$REPORT"
    echo "ACTUAL_SLUG=$ACTUAL_SLUG" >> "$REPORT"
    cat "$REPORT"; exit 1
  fi

  if [ "$ACTUAL_META" != "$EXPECTED_META" ]; then
    echo "ERROR: exported meta description does not match E34 polished_meta_description" >> "$REPORT"
    echo "EXPECTED_META=$EXPECTED_META" >> "$REPORT"
    echo "ACTUAL_META=$ACTUAL_META" >> "$REPORT"
    cat "$REPORT"; exit 1
  fi

  if [ "$ACTUAL_FOCUS" != "$EXPECTED_FOCUS" ]; then
    echo "ERROR: exported focus keyword does not match E34 focus_keyword" >> "$REPORT"
    echo "EXPECTED_FOCUS=$EXPECTED_FOCUS" >> "$REPORT"
    echo "ACTUAL_FOCUS=$ACTUAL_FOCUS" >> "$REPORT"
    cat "$REPORT"; exit 1
  fi

  if [ "$ACTUAL_EXCERPT" != "$EXPECTED_EXCERPT" ]; then
    echo "ERROR: exported excerpt does not match E34 excerpt" >> "$REPORT"
    echo "EXPECTED_EXCERPT=$EXPECTED_EXCERPT" >> "$REPORT"
    echo "ACTUAL_EXCERPT=$ACTUAL_EXCERPT" >> "$REPORT"
    cat "$REPORT"; exit 1
  fi

  if [ "$ACTUAL_CATEGORY" != "$EXPECTED_CATEGORY" ]; then
    echo "ERROR: exported category does not match outline category" >> "$REPORT"
    echo "EXPECTED_CATEGORY=$EXPECTED_CATEGORY" >> "$REPORT"
    echo "ACTUAL_CATEGORY=$ACTUAL_CATEGORY" >> "$REPORT"
    cat "$REPORT"; exit 1
  fi

  # ── 3. Content hash check ────────────────────────────────
  EXPECTED_CONTENT_FILE="$WORKDIR/E37_expected_content.md"
  EXPECTED_TAGS_FILE="$WORKDIR/E37_expected_tags.json"

  cat > "$EXPECTED_CONTENT_FILE" <<'EOF'
{{inputs.polished_content_markdown}}
EOF
  TITLE='{{inputs.polished_title}}' \
  TARGET_FILE="$EXPECTED_CONTENT_FILE" \
  python3 - <<'PY'
import os, pathlib
p = pathlib.Path(os.environ['TARGET_FILE'])
title = os.environ['TITLE']
lines = p.read_text(encoding='utf-8').splitlines(keepends=True)
if lines and lines[0].startswith('# '):
    lines[0] = f'# {title}\n'
p.write_text(''.join(lines), encoding='utf-8')
PY

  cat > "$EXPECTED_TAGS_FILE" <<'EOF'
{{inputs.tags}}
EOF

  EXPECTED_HASH="$(sha256sum "$EXPECTED_CONTENT_FILE" | awk '{print $1}')"
  ACTUAL_HASH="$(sha256sum "$CONTENT_FILE" | awk '{print $1}')"

  if [ "$EXPECTED_HASH" != "$ACTUAL_HASH" ]; then
    echo "ERROR: exported content does not match expected (E33 body + H1 sync applied)" >> "$REPORT"
    echo "EXPECTED_HASH=$EXPECTED_HASH" >> "$REPORT"
    echo "ACTUAL_HASH=$ACTUAL_HASH" >> "$REPORT"
    cat "$REPORT"; exit 1
  fi

  echo "OK: exported files exactly match expected upstream outputs" >> "$REPORT"

  # ── 4. Content quality checks (python) ──────────────────
  python3 - <<'PY'
import json, os, re, sys

workdir      = "/home/openclaw/.openclaw/workspace"
report_path  = os.path.join(workdir, "E37_validation_report.txt")
content_file = os.path.join(workdir, "E36_content.md")
title_file   = os.path.join(workdir, "E36_title.txt")
slug_file    = os.path.join(workdir, "E36_slug.txt")
meta_file    = os.path.join(workdir, "E36_meta_description.txt")
excerpt_file = os.path.join(workdir, "E36_excerpt.txt")
tags_file    = os.path.join(workdir, "E36_tags.json")

warnings = []
errors   = []

with open(content_file,  "r", encoding="utf-8") as f: content  = f.read()
with open(title_file,    "r", encoding="utf-8") as f: title    = f.read().strip()
with open(slug_file,     "r", encoding="utf-8") as f: slug     = f.read().strip()
with open(meta_file,     "r", encoding="utf-8") as f: meta     = f.read().strip()
with open(excerpt_file,  "r", encoding="utf-8") as f: excerpt  = f.read().strip()
with open(tags_file,     "r", encoding="utf-8") as f: tags_raw = f.read().strip()

content_len = len(content)
word_count  = len(re.findall(r"\b\w+\b", content))
h1_matches  = re.findall(r"^# (.+)$",   content, flags=re.MULTILINE)
h2_matches  = re.findall(r"^## (.+)$",  content, flags=re.MULTILINE)
h3_matches  = re.findall(r"^### (.+)$", content, flags=re.MULTILINE)
h2_count    = len(h2_matches)
h3_count    = len(h3_matches)

# ── NEW: stub pattern detection ──────────────────────────
STUB_PATTERNS = [
    "(Full content for publish)",
    "(Polished/humanized draft",
    "(Full draft saved here",
    "TBD",
    "placeholder",
]
for pat in STUB_PATTERNS:
    if pat.lower() in content.lower():
        errors.append(
            f"Stub pattern detected in content: '{pat}'. "
            "File contains placeholder text, not real article body."
        )

# ── NEW: pipe table check ────────────────────────────────
pipe_table_lines = [l for l in content.splitlines() if l.strip().startswith("|")]
if len(pipe_table_lines) < 3:
    errors.append(
        f"Comparison table missing or not in pipe format. "
        f"Found only {len(pipe_table_lines)} pipe-table line(s) (need ≥ 3 for a valid table). "
        "A bullet list is not a valid comparison table."
    )

# ── NEW: H3 presence check ───────────────────────────────
if h3_count == 0:
    errors.append(
        "No H3 sections found. Product review section must use one H3 per product "
        "under a single H2, not separate H2s per product."
    )

# ── word count: raised floor 1500 → 1800 ─────────────────
if word_count < 1800:
    errors.append(f"Word count too low: {word_count} (minimum 1800 for publish)")
elif word_count < 2000:
    warnings.append(f"Word count below preferred minimum: {word_count} (target ≥ 2000)")

# ── content length ────────────────────────────────────────
if content_len < 7000:
    errors.append(f"Content too short by characters: {content_len} (< 7000)")
elif content_len < 9000:
    warnings.append(f"Content shorter than preferred: {content_len} chars (< 9000)")

# ── H1 check ──────────────────────────────────────────────
if len(h1_matches) != 1:
    errors.append(f"Expected exactly 1 H1, found {len(h1_matches)}")
else:
    h1_text = h1_matches[0].strip()
    if h1_text != title:
        errors.append("H1 does not exactly match E36_title.txt")

# ── title length ──────────────────────────────────────────
title_len = len(title)
if title_len > 60:
    errors.append(f"Title too long: {title_len} chars (> 60). Pipeline must not publish.")
elif title_len > 54:
    warnings.append(f"Title longer than preferred: {title_len} chars (> 54)")
elif title_len > 50:
    warnings.append(f"Title slightly long: {title_len} chars (> 50)")

# ── H2 count ──────────────────────────────────────────────
if h2_count > 8:
    warnings.append(f"H2 count high: {h2_count} (spec target: 6–8)")
if h2_count < 4:
    warnings.append(f"H2 count low for a comparison article: {h2_count}")

# ── meta description length ───────────────────────────────
meta_len = len(meta)
if meta_len > 160:
    warnings.append(f"Meta description long: {meta_len} chars (> 160)")
if meta_len < 135:
    warnings.append(f"Meta description below preferred minimum: {meta_len} chars (target 135–150)")
if meta_len < 110:
    errors.append(f"Meta description too short: {meta_len} chars (< 110)")
if meta.startswith("#") or meta.startswith("TL;DR") or meta.startswith("-"):
    errors.append("Meta description has invalid leading format (# / TL;DR / list marker)")

# ── TL;DR check ───────────────────────────────────────────
tldr_count = len(re.findall(r"(?im)^##?\s*TL;DR\s*$|(?im)^TL;DR\s*$", content))
if tldr_count > 1:
    errors.append(f"Multiple TL;DR sections detected: {tldr_count}")
elif tldr_count == 0:
    warnings.append("No TL;DR section detected")

# ── heading level jump ────────────────────────────────────
headings  = re.findall(r"^(#{1,6})\s+(.+)$", content, flags=re.MULTILINE)
prev_level = 0
for marks, heading_text in headings:
    level = len(marks)
    if prev_level and level > prev_level + 1:
        errors.append(
            f"Heading level jump: H{prev_level} -> H{level} ({heading_text.strip()})"
        )
    prev_level = level

# ── slug length ───────────────────────────────────────────
if len(slug) > 75:
    warnings.append(f"Slug long: {len(slug)} chars")

# ── excerpt length ────────────────────────────────────────
if len(excerpt) > 220:
    warnings.append(f"Excerpt long: {len(excerpt)} chars")

# ── duplicate headings ────────────────────────────────────
headings_text = re.findall(r"^(##+ .+)$", content, flags=re.MULTILINE)
seen_h = set()
dup_h  = []
for h in headings_text:
    key = h.strip().lower()
    if key in seen_h:
        dup_h.append(h.strip())
    seen_h.add(key)
if dup_h:
    warnings.append("Duplicate headings: " + " | ".join(dup_h[:5]))

# ── repeated paragraphs ───────────────────────────────────
paras = [p.strip() for p in re.split(r"\n\s*\n", content) if p.strip()]
norm_long = []
for p in paras:
    if p.startswith("#") or p.startswith("- ") or p.startswith("|"):
        continue
    n = re.sub(r"\s+", " ", p).strip().lower()
    if len(n) >= 180:
        norm_long.append(n)
para_seen = set()
rep_p = []
for p in norm_long:
    if p in para_seen:
        rep_p.append(p[:140])
    para_seen.add(p)
if rep_p:
    warnings.append("Repeated long paragraphs detected")

# ── FAQ dupe check ────────────────────────────────────────
lines = content.splitlines()
faq_q = []
for line in lines:
    s = line.strip()
    if s.startswith("#"): continue
    if s.endswith("?") and len(s) >= 12:
        faq_q.append(s.lower())
faq_seen = set()
faq_dup  = []
for q in faq_q:
    if q in faq_seen: faq_dup.append(q)
    faq_seen.add(q)
if faq_dup:
    warnings.append("Duplicate FAQ questions detected")

# ── title over-repetition ─────────────────────────────────
title_occ = content.lower().count(title.lower()) if title else 0
if title_occ > 2:
    warnings.append(f"Title string appears {title_occ} times in content")

# ── tags check ────────────────────────────────────────────
try:
    tags = json.loads(tags_raw)
    if not isinstance(tags, list):
        errors.append("E36_tags.json is not a JSON array")
    elif len(tags) == 0:
        warnings.append("E36_tags.json is empty")
except Exception as e:
    errors.append(f"E36_tags.json invalid JSON: {e}")

# ── write report ─────────────────────────────────────────
with open(report_path, "a", encoding="utf-8") as f:
    f.write("publish_validation=completed\n")
    f.write(f"expected_title={title}\n")
    f.write(f"title_length={title_len}\n")
    f.write(f"content_length={content_len}\n")
    f.write(f"word_count={word_count}\n")
    f.write(f"h1_count={len(h1_matches)}\n")
    f.write(f"h2_count={h2_count}\n")
    f.write(f"h3_count={h3_count}\n")
    f.write(f"pipe_table_lines={len(pipe_table_lines)}\n")
    f.write(f"tldr_count={tldr_count}\n")
    f.write(f"meta_length={meta_len}\n")
    if warnings:
        f.write("Warnings:\n")
        for w in warnings:
            f.write(f"- {w}\n")
    if errors:
        f.write("Errors:\n")
        for e in errors:
            f.write(f"- {e}\n")

if errors:
    with open(report_path) as f: print(f.read())
    sys.exit(1)

with open(report_path) as f: print(f.read())
PY
next: X40_load_publish_artifacts
```


```yaml
## [X40] Load Publish Artifacts
id: X40_load_publish_artifacts
type: shell
description: Load exported publish artifact files so WordPress publish uses file-backed source-of-truth values
output_schema:
  published_title: string
  published_slug: string
  published_content_markdown: string
  published_meta_description: string
  published_focus_keyword: string
  published_excerpt: string
  published_category: string
  published_tags_json: string
script: |
  set -euo pipefail

  WORKDIR="/home/openclaw/.openclaw/workspace"

  test -s "$WORKDIR/E36_title.txt"
  test -s "$WORKDIR/E36_slug.txt"
  test -s "$WORKDIR/E36_content.md"
  test -s "$WORKDIR/E36_meta_description.txt"
  test -s "$WORKDIR/E36_focus_keyword.txt"
  test -s "$WORKDIR/E36_excerpt.txt"
  test -s "$WORKDIR/E36_category.txt"
  test -s "$WORKDIR/E36_tags.json"

  python3 - <<'PY'
  import json
  from pathlib import Path

  workdir = Path("/home/openclaw/.openclaw/workspace")

  print(json.dumps({
      "published_title": (workdir / "E36_title.txt").read_text(encoding="utf-8").strip(),
      "published_slug": (workdir / "E36_slug.txt").read_text(encoding="utf-8").strip(),
      "published_content_markdown": (workdir / "E36_content.md").read_text(encoding="utf-8"),
      "published_meta_description": (workdir / "E36_meta_description.txt").read_text(encoding="utf-8").strip(),
      "published_focus_keyword": (workdir / "E36_focus_keyword.txt").read_text(encoding="utf-8").strip(),
      "published_excerpt": (workdir / "E36_excerpt.txt").read_text(encoding="utf-8").strip(),
      "published_category": (workdir / "E36_category.txt").read_text(encoding="utf-8").strip(),
      "published_tags_json": (workdir / "E36_tags.json").read_text(encoding="utf-8").strip()
  }, ensure_ascii=False))
  PY
next: X41_wordpress_publish
```

```yaml
## [X41] WordPress Publish
id: X41_wordpress_publish
type: http
description: Create WordPress draft via OpenClaw custom endpoint with SEO metadata
input:
  wordpress_base_url: "{{site.wordpress_base_url}}"
  wordpress_username: "{{site.wordpress_username}}"
  wordpress_application_password: "{{site.wordpress_application_password}}"
  openclaw_shared_secret: "{{site.openclaw_shared_secret}}"

  title: "{{X40_load_publish_artifacts.output.published_title}}"
  slug: "{{X40_load_publish_artifacts.output.published_slug}}"
  content_markdown: "{{X40_load_publish_artifacts.output.published_content_markdown}}"
  meta_description: "{{X40_load_publish_artifacts.output.published_meta_description}}"
  focus_keyword: "{{X40_load_publish_artifacts.output.published_focus_keyword}}"
  excerpt: "{{X40_load_publish_artifacts.output.published_excerpt}}"
  category: "{{X40_load_publish_artifacts.output.published_category}}"
  tags_json: "{{X40_load_publish_artifacts.output.published_tags_json}}"
  status: "draft"

  chosen_content_type: "{{P02_keyword_selection.output.chosen_content_type}}"
  secondary_keywords: "{{P02_keyword_selection.output.secondary_keywords}}"
  related_products: "{{P02_keyword_selection.output.likely_affiliate_products}}"

request:
  method: POST
  url: "{{inputs.wordpress_base_url}}/wp-json/openclaw/v2/post"
  headers:
    Content-Type: "application/json"
    x-openclaw-secret: "{{inputs.openclaw_shared_secret}}"
  body:
    action: "create_post"
    title: "{{inputs.title}}"
    content_markdown: "{{inputs.content_markdown}}"
    excerpt: "{{inputs.excerpt}}"
    slug: "{{inputs.slug}}"
    status: "{{inputs.status}}"
    meta_description: "{{inputs.meta_description}}"
    primary_keyword: "{{inputs.focus_keyword}}"
    chosen_content_type: "{{inputs.chosen_content_type}}"
    secondary_keywords: "{{inputs.secondary_keywords}}"
    related_products: "{{inputs.related_products}}"
    category: "{{inputs.category}}"
    tags: "{{inputs.tags_json}}"

success_condition: "response.status >= 200 && response.status < 300 && response.body.success == true"

output:
  request_sent: true
  http_status: "{{response.status}}"
  success: "{{response.body.success}}"
  error_code: ""
  error_message: "{{response.body.message}}"
  post_id: "{{response.body.post_id}}"
  wordpress_post_id: "{{response.body.wordpress_post_id}}"
  post_url: "{{response.body.url}}"
  edit_url: "{{response.body.edit_url}}"

retry: 3
timeout: 30
next: X42_verify_wordpress_post
```

```yaml
## [X42] Verify WordPress Post
id: X42_verify_wordpress_post
type: http
description: >-
  Verify created post exists in WordPress REST API.
  Hard fail: HTTP error, post missing, title mismatch, wrong status, slug mismatch.
  Soft check (recorded in output only): excerpt and Rank Math meta fields.
  Rationale: excerpt/meta can differ due to WP normalization or Rank Math save timing;
  treating them as hard fail causes false pipeline errors on already-published posts.
input:
  wordpress_base_url: "{{site.wordpress_base_url}}"
  wordpress_username: "{{site.wordpress_username}}"
  wordpress_application_password: "{{site.wordpress_application_password}}"
  wordpress_post_id: "{{X41_wordpress_publish.output.wordpress_post_id}}"
  expected_title: "{{E34_seo_fields.output.polished_title}}"
  expected_status: "draft"
  expected_slug: "{{E34_seo_fields.output.slug}}"
  expected_excerpt: "{{E34_seo_fields.output.excerpt}}"
  expected_meta_description: "{{E34_seo_fields.output.polished_meta_description}}"
  expected_focus_keyword: "{{E34_seo_fields.output.focus_keyword}}"
request:
  method: GET
  url: "{{inputs.wordpress_base_url}}/wp-json/wp/v2/posts/{{inputs.wordpress_post_id}}?context=edit"
  auth:
    type: basic
    username: "{{inputs.wordpress_username}}"
    password: "{{inputs.wordpress_application_password}}"
  headers:
    Content-Type: "application/json"
success_condition: "response.status >= 200 && response.status < 300 && response.body.id != null && response.body.title.raw.replace(/&amp;/g, '&').replace(/&lt;/g, '<').replace(/&gt;/g, '>').replace(/&quot;/g, '"').replace(/&#039;/g, "'") == inputs.expected_title && response.body.status == inputs.expected_status && (response.body.slug == inputs.expected_slug || (response.body.slug.startsWith(inputs.expected_slug + '-') && response.body.slug.substring((inputs.expected_slug + '-').length).match(/^[0-9]+$/) != null))"
# soft checks: excerpt and rank_math are recorded in output fields but do not block the pipeline
output:
  verify_http_status: "{{response.status}}"
  verify_success: "{{response.status >= 200 && response.status < 300}}"
  verified_exists: "{{response.body.id != null}}"
  verified_post_id: "{{response.body.id}}"
  verified_post_url: "{{response.body.link}}"
  verified_title: "{{response.body.title.raw}}"
  verified_status: "{{response.body.status}}"
  verified_slug: "{{response.body.slug}}"
  verified_excerpt: "{{response.body.excerpt.raw}}"
  excerpt_saved: "{{response.body.excerpt.raw != ''}}"
  verified_rank_math_description: "{{response.body.meta.rank_math_description}}"
  verified_rank_math_focus_keyword: "{{response.body.meta.rank_math_focus_keyword}}"
  meta_present: "{{response.body.meta != null}}"
  meta_description_saved: "{{response.body.meta.rank_math_description != ''}}"
  meta_focus_keyword_saved: "{{response.body.meta.rank_math_focus_keyword != ''}}"
  verified_title_match: "{{response.body.title.raw == inputs.expected_title}}"
  verified_title_normalized_match: "{{response.body.title.raw.replace(/&amp;/g, '&').replace(/&lt;/g, '<').replace(/&gt;/g, '>').replace(/&quot;/g, '"').replace(/&#039;/g, "'") == inputs.expected_title}}"
  verified_status_match: "{{response.body.status == inputs.expected_status}}"
  verified_slug_match: "{{response.body.slug == inputs.expected_slug}}"
  verified_slug_accepted_match: "{{response.body.slug == inputs.expected_slug || (response.body.slug.startsWith(inputs.expected_slug + '-') && response.body.slug.substring((inputs.expected_slug + '-').length).match(/^[0-9]+$/) != null)}}"
  verified_excerpt_match: "{{response.body.excerpt.raw == inputs.expected_excerpt}}"
  meta_match_success: "{{response.body.meta.rank_math_description == inputs.expected_meta_description && response.body.meta.rank_math_focus_keyword == inputs.expected_focus_keyword}}"
  # strict_verify_success = hard checks only (mirrors success_condition)
  # for soft check audit use: verified_excerpt_match, meta_match_success
  strict_verify_success: "{{response.status >= 200 && response.status < 300 && response.body.id != null && response.body.title.raw.replace(/&amp;/g, '&').replace(/&lt;/g, '<').replace(/&gt;/g, '>').replace(/&quot;/g, '"').replace(/&#039;/g, "'") == inputs.expected_title && response.body.status == inputs.expected_status && (response.body.slug == inputs.expected_slug || (response.body.slug.startsWith(inputs.expected_slug + '-') && response.body.slug.substring((inputs.expected_slug + '-').length).match(/^[0-9]+$/) != null))}}"
next: Z50_memory_save
```

```yaml
## [Z50] Memory Save
id: Z50_memory_save
type: memory
description: Save publishing result, SEO fields, taxonomy, and strict verification summary for future runs
input:
  chosen_primary_keyword: "{{P02_keyword_selection.output.primary_keyword}}"
  chosen_content_type: "{{P02_keyword_selection.output.chosen_content_type}}"
  category: "{{R10_outline_generation.output.category}}"
  tags: "{{R10_outline_generation.output.tags}}"
  polished_title: "{{E34_seo_fields.output.polished_title}}"
  final_slug: "{{E34_seo_fields.output.slug}}"
  final_focus_keyword: "{{E34_seo_fields.output.focus_keyword}}"
  final_meta_description: "{{E34_seo_fields.output.polished_meta_description}}"
  final_excerpt: "{{E34_seo_fields.output.excerpt}}"
  wordpress_post_id: "{{X41_wordpress_publish.output.wordpress_post_id}}"
  post_url: "{{X41_wordpress_publish.output.post_url}}"
  edit_url: "{{X41_wordpress_publish.output.edit_url}}"
  verify_success: "{{X42_verify_wordpress_post.output.verify_success}}"
  strict_verify_success: "{{X42_verify_wordpress_post.output.strict_verify_success}}"
  verified_exists: "{{X42_verify_wordpress_post.output.verified_exists}}"
  verified_title: "{{X42_verify_wordpress_post.output.verified_title}}"
  verified_status: "{{X42_verify_wordpress_post.output.verified_status}}"
  verified_slug: "{{X42_verify_wordpress_post.output.verified_slug}}"
  verified_excerpt: "{{X42_verify_wordpress_post.output.verified_excerpt}}"
  verified_title_match: "{{X42_verify_wordpress_post.output.verified_title_match}}"
  verified_title_normalized_match: "{{X42_verify_wordpress_post.output.verified_title_normalized_match}}"
  verified_status_match: "{{X42_verify_wordpress_post.output.verified_status_match}}"
  verified_slug_match: "{{X42_verify_wordpress_post.output.verified_slug_match}}"
  verified_slug_accepted_match: "{{X42_verify_wordpress_post.output.verified_slug_accepted_match}}"
  verified_excerpt_match: "{{X42_verify_wordpress_post.output.verified_excerpt_match}}"
  excerpt_saved: "{{X42_verify_wordpress_post.output.excerpt_saved}}"
  meta_present: "{{X42_verify_wordpress_post.output.meta_present}}"
  meta_description_saved: "{{X42_verify_wordpress_post.output.meta_description_saved}}"
  meta_focus_keyword_saved: "{{X42_verify_wordpress_post.output.meta_focus_keyword_saved}}"
  meta_match_success: "{{X42_verify_wordpress_post.output.meta_match_success}}"
  verified_rank_math_description: "{{X42_verify_wordpress_post.output.verified_rank_math_description}}"
  verified_rank_math_focus_keyword: "{{X42_verify_wordpress_post.output.verified_rank_math_focus_keyword}}"
payload:
  keyword: "{{inputs.chosen_primary_keyword}}"
  content_type: "{{inputs.chosen_content_type}}"
  category: "{{inputs.category}}"
  tags: "{{inputs.tags}}"
  title: "{{inputs.polished_title}}"
  slug: "{{inputs.final_slug}}"
  focus_keyword: "{{inputs.final_focus_keyword}}"
  meta_description: "{{inputs.final_meta_description}}"
  excerpt: "{{inputs.final_excerpt}}"
  wordpress_post_id: "{{inputs.wordpress_post_id}}"
  post_url: "{{inputs.post_url}}"
  edit_url: "{{inputs.edit_url}}"
  verify_success: "{{inputs.verify_success}}"
  strict_verify_success: "{{inputs.strict_verify_success}}"
  verified_exists: "{{inputs.verified_exists}}"
  verified_title: "{{inputs.verified_title}}"
  verified_status: "{{inputs.verified_status}}"
  verified_slug: "{{inputs.verified_slug}}"
  verified_excerpt: "{{inputs.verified_excerpt}}"
  verified_title_match: "{{inputs.verified_title_match}}"
  verified_title_normalized_match: "{{inputs.verified_title_normalized_match}}"
  verified_status_match: "{{inputs.verified_status_match}}"
  verified_slug_match: "{{inputs.verified_slug_match}}"
  verified_slug_accepted_match: "{{inputs.verified_slug_accepted_match}}"
  verified_excerpt_match: "{{inputs.verified_excerpt_match}}"
  excerpt_saved: "{{inputs.excerpt_saved}}"
  meta_present: "{{inputs.meta_present}}"
  meta_description_saved: "{{inputs.meta_description_saved}}"
  meta_focus_keyword_saved: "{{inputs.meta_focus_keyword_saved}}"
  meta_match_success: "{{inputs.meta_match_success}}"
  verified_rank_math_description: "{{inputs.verified_rank_math_description}}"
  verified_rank_math_focus_keyword: "{{inputs.verified_rank_math_focus_keyword}}"
next: Z51_telegram_report
```

```yaml
## [Z51] Telegram Report
id: Z51_telegram_report
type: telegram
description: Send final publishing report with strict verification summary
input:
  telegram_bot_token: "{{telegram.bot_token}}"
  telegram_chat_id: "{{telegram.chat_id}}"
  polished_title: "{{E34_seo_fields.output.polished_title}}"
  chosen_primary_keyword: "{{P02_keyword_selection.output.primary_keyword}}"
  chosen_content_type: "{{P02_keyword_selection.output.chosen_content_type}}"
  category: "{{R10_outline_generation.output.category}}"
  tags: "{{R10_outline_generation.output.tags}}"
  final_focus_keyword: "{{E34_seo_fields.output.focus_keyword}}"
  final_meta_description: "{{E34_seo_fields.output.polished_meta_description}}"
  final_excerpt: "{{E34_seo_fields.output.excerpt}}"
  wordpress_post_id: "{{X41_wordpress_publish.output.wordpress_post_id}}"
  post_url: "{{X41_wordpress_publish.output.post_url}}"
  edit_url: "{{X41_wordpress_publish.output.edit_url}}"
  review_summary:     "{{E31_editorial_review.output.review_summary}}"
  humanization_notes: "{{E32_humanize.output.humanization_notes}}"
  wp_verify_success: "{{X42_verify_wordpress_post.output.verify_success}}"
  wp_strict_verify_success: "{{X42_verify_wordpress_post.output.strict_verify_success}}"
  wp_verified_exists: "{{X42_verify_wordpress_post.output.verified_exists}}"
  wp_verified_title: "{{X42_verify_wordpress_post.output.verified_title}}"
  wp_title_match: "{{X42_verify_wordpress_post.output.verified_title_normalized_match}}"
  wp_status: "{{X42_verify_wordpress_post.output.verified_status}}"
  wp_status_match: "{{X42_verify_wordpress_post.output.verified_status_match}}"
  wp_slug: "{{X42_verify_wordpress_post.output.verified_slug}}"
  wp_slug_match: "{{X42_verify_wordpress_post.output.verified_slug_accepted_match}}"
  wp_excerpt_saved: "{{X42_verify_wordpress_post.output.excerpt_saved}}"
  wp_verified_excerpt: "{{X42_verify_wordpress_post.output.verified_excerpt}}"
  wp_excerpt_match: "{{X42_verify_wordpress_post.output.verified_excerpt_match}}"
  wp_meta_present: "{{X42_verify_wordpress_post.output.meta_present}}"
  wp_meta_description_saved: "{{X42_verify_wordpress_post.output.meta_description_saved}}"
  wp_meta_focus_keyword_saved: "{{X42_verify_wordpress_post.output.meta_focus_keyword_saved}}"
  wp_meta_match_success: "{{X42_verify_wordpress_post.output.meta_match_success}}"
  wp_verified_rank_math_description: "{{X42_verify_wordpress_post.output.verified_rank_math_description}}"
  wp_verified_rank_math_focus_keyword: "{{X42_verify_wordpress_post.output.verified_rank_math_focus_keyword}}"
message: |
  ✅ Publish completed

  Title: {{inputs.polished_title}}
  Primary keyword: {{inputs.chosen_primary_keyword}}
  Content type: {{inputs.chosen_content_type}}
  Category: {{inputs.category}}
  Tags: {{inputs.tags}}

  Focus keyword: {{inputs.final_focus_keyword}}
  Final meta description:
  {{inputs.final_meta_description}}

  Final excerpt:
  {{inputs.final_excerpt}}

  WordPress post ID: {{inputs.wordpress_post_id}}
  Post URL: {{inputs.post_url}}
  Edit URL: {{inputs.edit_url}}

  Editorial review: {{inputs.review_summary}}
  Humanization: {{inputs.humanization_notes}}

  Verify success: {{inputs.wp_verify_success}}
  Strict verify success: {{inputs.wp_strict_verify_success}}
  Exists: {{inputs.wp_verified_exists}}

  Verified title: {{inputs.wp_verified_title}}
  Title normalized match: {{inputs.wp_title_match}}
  Status: {{inputs.wp_status}}
  Status match: {{inputs.wp_status_match}}
  Slug: {{inputs.wp_slug}}
  Slug accepted match: {{inputs.wp_slug_match}}

  Excerpt saved: {{inputs.wp_excerpt_saved}}
  Verified excerpt:
  {{inputs.wp_verified_excerpt}}
  Excerpt match: {{inputs.wp_excerpt_match}}

  Meta present: {{inputs.wp_meta_present}}
  Meta description saved: {{inputs.wp_meta_description_saved}}
  Focus keyword saved: {{inputs.wp_meta_focus_keyword_saved}}
  Meta exact match: {{inputs.wp_meta_match_success}}

  Verified Rank Math description:
  {{inputs.wp_verified_rank_math_description}}

  Verified Rank Math focus keyword:
  {{inputs.wp_verified_rank_math_focus_keyword}}
```

```yaml
## [Z51_fail] Telegram Fail Report
id: Z51_telegram_fail_report
type: telegram
description: >
  E31 editorial review rejected the draft. Send a failure report via Telegram and halt the pipeline.
  Operator must decide whether to re-run from W20, R10, or P00.
input:
  telegram_bot_token:     "{{telegram.bot_token}}"
  telegram_chat_id:       "{{telegram.chat_id}}"
  chosen_primary_keyword: "{{P02_keyword_selection.output.primary_keyword}}"
  chosen_content_type:    "{{P02_keyword_selection.output.chosen_content_type}}"
  best_title:             "{{R10_outline_generation.output.best_title}}"
  verdict:                "{{E31_editorial_review.output.verdict}}"
  actual_word_count:      "{{E31_editorial_review.output.actual_word_count}}"
  reject_reasons:         "{{E31_editorial_review.output.reject_reasons}}"
  review_summary:         "{{E31_editorial_review.output.review_summary}}"
  missing_elements:       "{{E31_editorial_review.output.missing_structural_elements}}"
  scores:                 "{{E31_editorial_review.output.scores}}"
  total_score:            "{{E31_editorial_review.output.total_score}}"
message: |
  ❌ Draft rejected — pipeline halted

  Keyword: {{inputs.chosen_primary_keyword}}
  Content type: {{inputs.chosen_content_type}}
  Title: {{inputs.best_title}}

  Word count: {{inputs.actual_word_count}} (target: ≥ 2000)
  Total score: {{inputs.total_score}} / 30

  Scores: {{inputs.scores}}

  Reject reasons:
  {{inputs.reject_reasons}}

  Missing elements:
  {{inputs.missing_elements}}

  Summary: {{inputs.review_summary}}

  ── Next steps ───────────────────────
  ① Re-run from W20 — fix draft writing prompt or inputs
  ② Re-run from R10 — restructure outline if H2/H3 issues persist
  ③ Re-run from P00 — pick a different keyword if topic is too narrow
next: null
```
