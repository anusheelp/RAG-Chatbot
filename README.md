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
