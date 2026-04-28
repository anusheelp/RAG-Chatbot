# Event RAG Chatbot

A retrieval-augmented generation (RAG) chatbot built for wedding guests — answering logistics questions without requiring host intervention.

---

## The Problem

A destination wedding across multiple days and locations generates a flood of repetitive guest questions: What time does the ceremony start? Where is the venue? What's the dress code for the mehendi? Is there transport from the hotel?

The default solution is a wedding coordinator fielding WhatsApp messages. That doesn't scale, breaks down at peak moments (exactly when you need it most), and pulls the host away from the event.

I wanted guests to get accurate, instant answers — and I wanted to stop being the answer.

---

## What I Built

A RAG-based chatbot trained on wedding-specific documents — itineraries, venue details, FAQs, transport schedules — that guests could query in natural language and get accurate, grounded responses.

The key design constraint: **the system had to be right, not just helpful**. Hallucinated answers about wrong venue times or transport details would've caused real problems with real guests.

---

## Key Decisions

**RAG over fine-tuning**
Fine-tuning an LLM on wedding-specific content would've taken significantly more time and produced a less maintainable system. RAG let me update the knowledge base (e.g., last-minute schedule changes) without touching the model. For a time-sensitive, frequently-changing information set, retrieval wins over baking in knowledge at training time.

**Grounding over fluency**
I constrained the model to answer only from retrieved context rather than generating freely. A chatbot that says "I don't have that information" is better than one that confidently invents a time or location. Accuracy was the non-negotiable.

**Scope discipline**
I explicitly scoped out anything the chatbot couldn't do reliably: RSVPs, dietary requests, anything requiring a human decision. Better to do fewer things well than to create false confidence in a system that might fail on edge cases.

---

## Architecture

```
Wedding Documents (itineraries, FAQs, venue info, transport)
  → Document chunking + embedding
  → Vector store (retrieval index)
  → Query → Retrieval → Augmented prompt → LLM response
  → Guest-facing chat interface
```

---

## What I'd Do Differently

- **Evaluation before launch** — I did informal testing, but a small structured eval set (known questions with known right answers) would've given me more confidence before guests started using it.
- **Fallback routing** — when the chatbot hits its confidence floor, it should route to a human rather than just saying it doesn't know. I'd build that escalation path from the start next time.
- **Usage analytics** — post-event, I had no clean data on which questions were most common, where it succeeded, or where it failed. Lightweight logging from day one would've been valuable.

---

## Status

Complete. Deployed and used live at Em & Anu 2026 wedding event.

**Potential extensions**: Event chatbot as a service for wedding planners · Conference/corporate event Q&A assistant · Multilingual support layered on top

---

## Stack

- Python
- RAG architecture (retrieval-augmented generation)
- Vector store / embedding pipeline
- LLM (response generation with retrieval grounding)

## Text Chunking strategies considered

1. . 🔤** Character Text Splitting**
What it is: Split text every N characters, with optional overlap.

Pros:
Extremely simple to implement
No dependencies, fast

Cons:
Completely ignores meaning, sentences, or paragraphs
Frequently cuts mid-word or mid-sentence
Overlap helps slightly but doesn't fix the fundamental problem

Verdict: Baseline only. It produces uniform chunk sizes, simplifying indexing, but context loss is significant — critical information may span multiple chunks, reducing retrieval effectiveness.

2. 🔢 **Token Text Splitting**
What it is: Same concept as character splitting, but split on tokens (the actual units LLMs operate on) rather than raw characters.

Pros:
More aligned with how embedding models and LLMs actually process text
Predictable token counts prevent embedding model overflow

Cons:
Still ignores semantic boundaries — cuts happen based on count, not meaning
OpenAI's default settings (800 tokens with 400 overlap) resulted in below-average recall and the lowest scores across other metrics Medium

Verdict: Slightly better than character splitting in practice, but still a blunt instrument.

3. 🔁 **Recursive Character / Token Splitting**
What it is: Tries a hierarchy of separators (paragraphs → newlines → sentences → characters) and only falls back to a cruder split if the text still exceeds the target size.
Pros:

Respects natural text structure much better than flat splitting
Simple to implement via LangChain's RecursiveCharacterTextSplitter
Surprisingly competitive with more sophisticated methods

Cons:

Recursive splitters apply a hierarchy of separators to enforce length constraints, but they can group unrelated topics together, reducing cohesion arxiv
Still rule-based — no understanding of meaning

Verdict: The best default for most use cases. Recursive character splitting at 400–512 tokens with 10–20% overlap is the recommended starting point for most RAG systems.

4. 📐 **Kamradt & Modified Semantic Chunking**
What it is: Greg Kamradt's method embeds every sentence, then finds "breakpoints" where cosine similarity between adjacent sentences drops sharply — indicating a topic shift — and splits there.
Pros:

Actually content-aware; splits at genuine topic boundaries
More semantically coherent chunks than fixed-size methods

Cons:

Requires embedding every single sentence (expensive at scale)
Default settings underperform: the KamradtSemanticChunker with default settings scores slightly below average across all metrics, with a recall of 0.836 Trychroma
The modified version (KamradtModified) is better: recall rises to 0.871 with a similar boost in remaining metrics when the modifications of the KamradtModifiedChunker are applied, despite mean chunk length dropping Trychroma

Verdict: Good concept, but fragile to hyperparameter defaults. The modified version is meaningfully better, but still not the top performer.

5. 🔵 **Cluster Semantic Chunking**
What it is: Embed sentences, then use clustering algorithms (e.g., k-means) to group semantically similar sentences together into chunks — rather than relying on adjacent similarity like Kamradt.
Pros:

Can group topically related content even if it's not adjacent in the text
With the right max chunk size, achieves excellent recall and precision
ClusterSemanticChunker with a max chunk size of 400 tokens achieves the second-highest recall of 0.913; dropping to 200 tokens results in average recall but the highest precision and IoU across all methods Trychroma

Cons:

Computationally heavier than recursive methods
Requires tuning the max chunk size parameter carefully

Verdict: The best balance of recall and precision among all methods. The 200-token variant is the precision champion; the 400-token variant maximizes recall.

6. 🤖 **LLM Semantic Chunking**
What it is: Feed the document to an LLM and ask it to identify natural semantic boundaries directly — using its language understanding rather than embedding similarity.
Pros:

LLMSemanticChunker achieves the highest recall of 0.919, suggesting LLMs are relatively capable at this task Trychroma
Best at preserving complex, multi-hop reasoning and interweaved topics

Cons:

Relies on the performance of the underlying LLM, is potentially expensive in terms of compute and API calls, and is harder to standardize or replicate consistently Databricks
High latency; not practical for large document corpora on a tight budget

Verdict: Highest raw recall, but the cost/complexity is hard to justify unless your documents are complex and your budget allows it.

**Decision:** I just went with token chunking for simplicity. 
