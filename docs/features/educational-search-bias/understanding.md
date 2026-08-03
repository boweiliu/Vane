# Understanding: Why We're Building Search History Recording & Educational Content Biasing

## What Problem Does This Solve?

Every AI search engine can answer a question. Very few of them *remember what you're trying to learn*.

When someone uses Vane, they are often in one of two modes:
1. **Transactional**: "What's the weather?" "Convert 100 EUR to USD." Quick facts, quick exits.
2. **Exploratory**: "How do neural networks actually learn?" "Explain the history of the Habsburg monarchy." These are journeys, not transactions.

For exploratory searches, the quality gap between "a decent answer" and "a genuinely educational answer" is enormous — and it's not just about the LLM's ability to write well. It's about the *raw material* the AI has to work with. Garbage sources in, garbage synthesis out. Even the best model can't cite papers it wasn't given.

And here's the deeper issue: **web search is not optimized for learning.** Search engines rank by recency, engagement, and SEO manipulation. They do not systematically elevate the article that teaches you the *structure* of a topic, the historical context, the dissenting views, or the underlying mechanism. They surface what is *popular*, not what is *pedagogically helpful*.

This feature asks: *What if Vane could recognize when you're in learning mode and quietly bias the entire pipeline — queries, sources, synthesis, tone — toward high-quality informational content?*

---

## Why Now?

### Vane's architecture is ready
Vane already has the right infrastructure to make this work:
- A **classifier step** that interprets intent before acting.
- A **multi-mode researcher** (speed / balanced / quality) that can search deeper when asked.
- **Academic search** and **citation support**, which means the plumbing for surfacing authoritative sources already exists.
- **Local SQLite storage** for chats and messages, so adding a lightweight profile/history layer requires no new infrastructure.
- **A settings/preferences system** where users can toggle and inspect behavior.

We're not trying to bolt personalization onto a brittle product. We're adding a small, well-contained loop to a system that already orchestrates search, research, and answer generation.

### The competition doesn't do this — yet
Perplexica, Perplexity, and other AI search engines treat every query as independent. They answer, cite, and forget. There's no memory of whether you are a deep researcher or a casual fact-checker. By adding a lightweight intent-detection and preference-memory system, Vane could be the first in its class to feel *adaptive* rather than *stateless*.

### It reinforces Vane's identity
Vane is promoted as privacy-focused, self-hosted, and capable of quality mode with academic sources. A bias toward educational content is a natural extension of that identity. It says: *this tool is built for people who actually want to understand things*, not just people who want to feel informed.

---

## What This Is Not

This is **not** an attempt to build a full-blown recommendation engine or a user-tracking surveillance layer. Key boundaries:

- **No external data sharing.** Everything stays in the user's local SQLite database or their own deployed instance.
- **No opaque behavioral profiling.** We explicitly detect *educational intent per query* and aggregate that into a lightweight session score. We are not trying to infer political leanings, purchase intent, demographics, or anything else.
- **No mandatory bias.** Users can turn this off. They can see when it's active. The default behavior remains neutral.
- **No gating of content.** We do not *remove* sources or censor non-educational results. We *reorder* and *re-query* to surface better material, while keeping the full web accessible.

---

## How This Helps Users

### For the curious learner
You come to Vane asking "how does CRISPR work." The system detects educational intent, automatically searches for `.edu` and review-article sources, and the writer is instructed to define terms, trace history, and explain mechanisms. You get a structured mini-essay with citations, not a bullet-point recap of recent news headlines.

### For the returning user
You used Vane yesterday to research the history of the Ottoman Empire and today you want to know about the Safavid Empire. The system recognizes your recent pattern of historical deep-dives and subtly biases today's search toward authoritative history sources — without you toggling "academic mode." It respects your context.

### For the researcher
You are using Vane in quality mode to write a paper. The system consistently prefers peer-reviewed or institutional sources over blog posts. Over time, because your session history is overwhelmingly academic, the classifier learns to default toward scholarly framing, saving you from manually refining queries.

### For the casual user who occasionally goes deep
Most of your searches are casual, but every few weeks you go on a learning binge. The system adapts dynamically: no bias during your weather checks, but it kicks in when you ask "explain Bayesian statistics." This is exactly the behavior you want.

---

## How This Helps Vane

### Differentiation in a crowded space
AI search is a commodity at the surface level. Everyone can stream an answer with citations. The moat is in the *depth* and *trustworthiness* of what gets surfaced. By making Vane "smarter about what you want," we create a reason to choose it over alternatives.

### Better utilization of existing features
Vane already has academic search, quality mode, and citation rendering. Those features are under-utilized if users have to manually discover them. This feature surfaces them automatically when relevant.

### Foundation for future personalization
This is a thin slice of a much larger possibility space. Once you are recording and acting on *one* dimension of user preference (educational intent), it's much easier to add others: domain preferences ("always include Stack Overflow for coding"), language preferences, recency-vs-depth tradeoffs, etc.

### Community alignment
Vane's community skews technical, self-hosting, privacy-aware. These are exactly the users who appreciate depth, transparency, and control — the values this feature is built around.

---

## The Philosophy Behind It

Modern information overload isn't caused by too many true facts. It's caused by too many *low-signal* facts.

When you search for "climate change," you don't just want to know that it exists. You want to understand the mechanisms, the uncertainties, the evidence, the dissent, the history of the science. But search engines serve you headlines, hot takes, and the most recent controversy because those drive clicks.

An AI search engine has a chance to be different. It has the *synthesis* power of a large language model and the *retrieval* power of the web. What's missing is the *judgment* to know what the user is actually looking for — and the *memory* to get better at making that judgment over time.

This feature is that judgment and memory. It's a step from "answer engine" to "learning companion."

---

## Success Criteria

We should be able to tell this is working if:
1. **Users with high educational intent see a higher proportion of `.edu`, `.gov`, academic, and long-form sources** compared to the current baseline.
2. **Subjective evaluation of responses improves** — e.g., in a blind test, users rate Vane's answers as "more thorough" or "better explained" for educational queries.
3. **Session depth increases slightly** — users who start with educational queries have longer, more multi-turn conversations because the answers actually satisfy follow-up curiosity.
4. **No degradation on transactional queries** — weather, stock, and calculation searches are unaffected in speed and accuracy.
5. **Opt-out doesn't hurt experience** — users who disable the feature should not notice a negative difference from the current behavior.

---

*This document was written to align the team on why this feature matters before implementation begins. It should be revisited after the first iteration to see if our assumptions held up.*
