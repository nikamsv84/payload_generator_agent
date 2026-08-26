# payload_generator_agent

An AI agent that generates attack payloads for controlled penetration testing against LLM API endpoints, grounded in the OWASP Top 10 for LLM Applications and PortSwigger's documented techniques. Built to integrate with [LLM Inspector](#) as an on-demand red-teaming tool — not an automated attacker.

## Why this exists

[LLM Inspector](#) and its [detection models](#) answer *"is this request an attack?"*. This project answers a different question: *"what attack should we try next, given everything we've already seen?"* It's the offensive counterpart to the defensive pipeline — used to actively probe an endpoint's defenses, refine the detector's training data with real findings, and validate that the proxy's own protections actually hold up under a directed attack chain.

## Design principles (non-negotiable)

1. **No autonomous delivery.** The agent never sends a payload to a real target on its own. It proposes a payload; a human reviews it, may edit it, applies it to the proxy's Packet Inspector, and only then chooses to forward it. This is the same `wait_for_user_action` checkpoint the proxy already uses for intercepted requests — the agent's output flows into that existing gate, it doesn't bypass it.
2. **Grounded, not improvised.** Payloads are generated from retrieved OWASP/PortSwigger reference material (via RAG), not purely from the model's own training memory — this is meant to reduce hallucinated or outdated attack techniques.
3. **Self-checked before it reaches a human.** Before a payload is shown to a person, the agent critiques its own output and sandbox-tests it against a local model — never the real target — so obviously-broken proposals don't waste review time.

## Agentic design patterns used

| Pattern | Used? | Where |
|---|---|---|
| **Tool Use** | Yes | Core of the architecture — history retrieval, RAG lookup, payload generation, response evaluation |
| **Planning** | Yes | Strategy selection based on attack history; explicit replanning when a technique fails |
| **Reflection** | Yes | Self-critique + local sandbox test before a payload is surfaced to a human |
| **Multi-agent collaboration** | No (for now) | Unnecessary complexity at this scale; revisit only if a single agent's responsibilities grow too large to reason about |

## Architecture
<img width="500" alt="payload_generator_agent_architecture_en" src="https://github.com/user-attachments/assets/78e8b8cf-1938-43fb-8b29-920d630146b0" />


### Why `check_for_leak_or_anomaly` combines regex and LLM judgment

Relying on either alone has a known failure mode: pure regex misses semantic leaks (e.g. a model explaining internal system behavior without ever printing something that looks like a key), while pure LLM judgment can miss or hallucinate on well-known secret formats it should catch deterministically. Known key/token formats are matched with regex (high-confidence, deterministic); everything else is judged by the agent's own LLM as a lower-confidence signal a human still reviews.

## Project status

Early architecture/design phase. No agent code has been written yet.

### Planned phases

- **Phase 1** — Minimal agent loop: a single `generate_payload` tool, no database, no RAG, no reflection. Goal: get the tool-calling mechanism working end to end with a local model (Ollama).
- **Phase 2** — RAG pipeline: collect OWASP LLM Top 10 + PortSwigger reference material, chunk it, embed it, stand up a vector store, wire up `retrieve_attack_technique`.
- **Phase 3** — Reflection: `self_critique` and `sandbox_test` added before any payload reaches a human.
- **Phase 4** — Full integration: real attack history from the proxy's database, human-in-the-loop UI, Packet Inspector hookup.

## Relationship to other repos

- [`llm-attack-detector-training`](#) — trains the ML classifiers this agent's payloads are, indirectly, meant to help improve over time (successful attacks are candidates for future training data).
- [`LLM-Inspector`](#) — the MITM proxy this agent integrates with; owns the actual request database and the human approval flow this agent's output feeds into.

## Responsible use

This tool is built for authorized security testing only — testing endpoints
you own or have explicit permission to test. Do not use it against systems
without authorization. The authors are not responsible for misuse.
