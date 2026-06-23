# 📚 Dossier d'apprentissage — TechWatch IA

Ce dossier explique, **un sujet à la fois**, chaque principe et chaque brique technique nécessaires pour construire **TechWatch IA** (plateforme de veille techno assistée par IA : détecter les nouvelles versions d'une techno — ex. *React 18 → 19* — et produire une synthèse de migration **fiable, sourcée et vérifiable**).

Chaque fiche suit le même plan : **En une phrase** · **Pourquoi pour TechWatch IA** · **Le principe** · **Exemples concrets** · **Subtilités & pièges** · **Checklist** · **10 questions / réponses** · **Sources** (annotées, vérifiées).

> **▶ [Réviser en ligne (quiz & fiches interactives)](index.html)** — 140 QCM, 160 fiches de révision, liens vers les sources. *(Ouvrir `docs/learn/index.html` dans un navigateur.)*
>
> Vue d'ensemble visuelle du projet : [`../visuel.excalidraw`](../visuel.excalidraw) · Dossier d'architecture complet (vision cible) : [`../architecture-plateforme-veille-ia.md`](../architecture-plateforme-veille-ia.md)

---

## 🧠 Principes d'ingénierie IA

| # | Fiche | En bref |
|---|-------|---------|
| 01 | [Workflow déterministe vs agents autonomes](01-workflow-deterministe-vs-agents.md) | Pourquoi orchestrer des étapes fixes plutôt que des agents autonomes (coût, latence, non-déterminisme). |
| 02 | [Long-context, RAG ou fine-tuning](02-long-context-rag-fine-tuning.md) | L'arbre de décision ; pourquoi le long-context suffit au MVP (zéro vector DB). |
| 03 | [Prompt engineering & sorties structurées](03-prompt-engineering-sorties-structurees.md) | n-shot, chain-of-thought, JSON schema, tool use, prefill. |
| 04 | [Grounding, citations & anti-hallucination](04-grounding-anti-hallucination.md) | Citations vérifiables, « no evidence, no answer », groundedness. |
| 05 | [Eval-driven development & golden sets](05-eval-driven-development.md) | Golden sets, assertions déterministes, « evals are the new unit tests ». |
| 06 | [LLM-as-a-judge : usages et pièges](06-llm-as-a-judge.md) | Quand l'utiliser, biais (position, verbosité), pairwise vs Likert. |
| 07 | [Guardrails (garde-fous en couches)](07-guardrails.md) | Couches pas-cher→cher, fail-closed, « schéma ≠ vérité ». |
| 08 | [Human-in-the-loop](08-human-in-the-loop.md) | L'IA suggère, l'humain valide ; revue par échantillonnage ; data flywheel. |
| 09 | [Sécurité LLM : OWASP LLM Top 10](09-securite-llm-owasp.md) | Prompt injection indirecte, données ≠ instructions, sorties contraintes. |
| 10 | [FinOps IA : coûts & caching](10-finops-ia-couts.md) | Prompt caching, model routing, coût par synthèse. |
| 11 | [Observabilité LLM (OpenTelemetry)](11-observabilite-llm.md) | Traces LLM (tokens/coût/latence), conventions GenAI, SLO qualité/coût. |

## 🛠️ Pratiques techniques (réalisation)

| # | Fiche | En bref |
|---|-------|---------|
| 12 | [Détection de version](12-detection-de-version.md) | Polling GitHub Releases, SemVer, ETag/If-None-Match, webhooks. |
| 13 | [Collecte API-first & extraction](13-collecte-api-first-extraction.md) | APIs officielles > scraping ; HTML→Markdown ; « garbage in, garbage out ». |
| 14 | [Sécurité du crawler : SSRF & allowlist](14-ssrf-allowlist-crawler.md) | Allowlist de domaines, blocage des IP internes / métadonnées cloud. |
| 15 | [Monolithe modulaire & DDD](15-monolithe-modulaire-ddd.md) | Frontières logiques vs microservices ; « MonolithFirst » ; quand extraire. |
| 16 | [Serverless event-driven & scale-to-zero](16-serverless-event-driven.md) | Pourquoi le serverless pour une charge batch/bursty tolérante à la latence. |
| 17 | [Notifications : fan-out & idempotence](17-notifications-fan-out-idempotence.md) | 1→N, at-least-once, clés d'idempotence, transactional outbox. |
| 18 | [Choix & gestion des modèles (Claude)](18-choix-gestion-modeles.md) | Haiku/Sonnet/Opus, choisir le plus petit qui suffit, pinning, routing. |
| 19 | [Provenance & versionnement du contenu](19-provenance-versionnement-contenu.md) | Contenu brut + provenance (URL/date/hash), audit trail, retraitement. |
| 20 | [Adéquation échelle/complexité (YAGNI)](20-right-sizing-yagni.md) | L'over-engineering comme risque dominant ; « innovation tokens ». |

---

## Méthode & fiabilité des sources

- **Sources priorisées** : documentation officielle (Anthropic, OpenAI, GitHub, OWASP, NIST, OpenTelemetry, AWS/Google Well-Architected) et praticiens reconnus (Martin Fowler, *applied-llms.org*, Chip Huyen, Eugene Yan, Hamel Husain, Simon Willison, Lilian Weng, Dan McKinley). Fermes de contenu SEO exclues.
- Chaque fiche a été **rédigée puis relue de façon adversariale** (vérification des sources par requête HTTP, contrôle des 10 Q/R et de l'exactitude technique).
- Les fiches sont en **français** ; certaines sources de référence sont en anglais (l'essentiel de la littérature LLM l'est).
