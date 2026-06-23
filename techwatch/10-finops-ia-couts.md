# FinOps IA : coûts, prompt caching, model routing

> **En une phrase.** Maîtriser le coût d'un produit LLM, c'est d'abord mesurer chaque appel, puis empiler des leviers multiplicatifs — cache de préfixe (~90 % d'économie sur le contexte réutilisé), routing vers le plus petit modèle acceptable, et Batch API (−50 %) — sans jamais sacrifier le différenciateur « confiance / sourcé » du projet.

## Pourquoi c'est nécessaire pour TechWatch IA

TechWatch IA repose sur un LLM managé en architecture serverless event-driven : chaque détection de version et chaque synthèse de migration déclenche des appels facturés au token. Comme le projet charge **toute la doc source d'une techno en long-context** (zéro vector DB au MVP), le poste de coût n°1 est le contexte réutilisé d'une question à l'autre — exactement ce que le prompt caching attaque. Suivre le coût/synthèse dès J1 et router les tâches simples (« est-ce une nouvelle version ? ») vers un modèle bon marché tout en réservant les synthèses complexes au modèle le plus capable est ce qui rend le produit économiquement viable sans dégrader la fiabilité qui fait sa valeur.

## Le principe

Trois familles de leviers, à appliquer dans cet ordre.

**1. Mesurer avant d'optimiser.** Chaque réponse de l'API expose un objet `usage` : `input_tokens` (tokens non cachés, plein tarif), `cache_creation_input_tokens` (écriture cache), `cache_read_input_tokens` (lecture cache, ~0,1× le prix input), `output_tokens`. Le total des tokens d'entrée = `input_tokens + cache_creation_input_tokens + cache_read_input_tokens`. Sans cette métrologie, toute optimisation est aveugle. On capture `usage` par appel, on l'agrège par techno surveillée et par étape du workflow, et on expose un **coût/synthèse** dans l'observabilité (OpenTelemetry).

**2. Le prompt caching — levier n°1.** Le cache est un **match de préfixe exact** : le contenu stable (system prompt figé, liste d'outils déterministe, doc source de la techno) va en tête ; le contenu volatil (question, timestamp, ID de session) va **après** le dernier point de césure (`cache_control`). Économie chez Anthropic : lecture à **0,1×** le prix input (~90 % d'économie sur la partie en cache). Le coût d'écriture est de **1,25×** (TTL 5 min) ou **2×** (TTL 1 h). Le break-even : ≥ 2 lectures pour rentabiliser le TTL 5 min, ≥ 3 pour le TTL 1 h.

**3. Le model routing/tiering.** On route chaque tâche vers le plus petit modèle qui « fait le job » : tâches simples (classification, extraction de numéro de version, détection de doublon) vers **Haiku 4.5** ($1/$5 par MTok), synthèse standard vers **Sonnet 4.6** ($3/$15), synthèses de migration complexes et raisonnement long vers **Opus 4.8** ($5/$25). Le routing préserve mieux la qualité avec du prompting (un Haiku bien promté, n-shot, peut égaler un Opus zero-shot) qu'avec du sous-dimensionnement brutal. Pour le non temps-réel (re-synthèses planifiées, évals), le **Batch API** ajoute −50 % sur tous les tokens, cumulable avec le caching dans chaque requête. Ces leviers s'empilent de façon **multiplicative** car chacun s'applique à un type de trafic différent.

## Exemples concrets

**Pipeline de synthèse React 18→19 avec cache de préfixe.** Le préfixe stable (system prompt + changelog officiel React 19 + guide de migration) est mis en cache ; seule la question variable paie le plein tarif.

```python
import anthropic

client = anthropic.Anthropic()

SYSTEM_FIGE = "Tu es un assistant de veille. Cite la source de chaque affirmation..."
DOC_SOURCE_REACT19 = open("sources/react-19-migration.md").read()  # préfixe stable

def repondre(question: str):
    resp = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=16000,
        system=[
            {"type": "text", "text": SYSTEM_FIGE},
            {
                "type": "text",
                "text": DOC_SOURCE_REACT19,
                "cache_control": {"type": "ephemeral"},  # césure à la FIN du préfixe partagé
            },
        ],
        messages=[{"role": "user", "content": question}],  # volatil, APRÈS la césure
    )
    u = resp.usage
    # Prouver l'économie : cache_read_input_tokens doit être > 0 dès le 2e appel
    print(f"read={u.cache_read_input_tokens} write={u.cache_creation_input_tokens} input={u.input_tokens}")
    return resp

repondre("Quels breaking changes sur les refs ?")       # 1er appel : écrit le cache (1,25×)
repondre("Comment migrer Suspense ?")                    # 2e appel : lit le cache (0,1×)
```

**Routing par étape du workflow.** L'étape 1 (« une nouvelle version React est-elle sortie ? », déduplication des annonces) part vers Haiku 4.5 ; la rédaction de la synthèse sourcée part vers Sonnet 4.6 ou Opus 4.8 selon un score de complexité.

```python
def router(tache: str, complexite: float) -> str:
    if tache in ("detecter_version", "dedupliquer"):
        return "claude-haiku-4-5"          # $1/$5 — classification simple
    if tache == "synthese" and complexite >= 0.7:
        return "claude-opus-4-8"           # $5/$25 — raisonnement long
    return "claude-sonnet-4-6"             # $3/$15 — synthèse standard
```

**Re-génération nocturne en Batch API (−50 %).** Re-synthétiser toutes les technos surveillées une fois par nuit, chaque requête réutilisant le cache de préfixe de sa doc source — non temps-réel, la fenêtre 24 h est acceptable. Les résultats arrivent dans un ordre quelconque : on indexe par `custom_id`, jamais par position.

## Subtilités & pièges à éviter

- **Match de préfixe EXACT.** Un seul octet modifié dans le préfixe (ordre de rendu `tools` → `system` → `messages`) invalide tout ce qui suit. Sérialiser le JSON avec `sort_keys`, figer l'ordre des outils.
- **`datetime.now()`, UUID ou ID de session dans le system prompt** = cache invalidé à chaque requête. Symptôme : `cache_read_input_tokens` reste à 0. Injecter le contexte dynamique APRÈS le dernier breakpoint.
- **Seuil minimal de mise en cache.** ~1024 tokens pour Opus 4.8 / Sonnet 4.6 (la doc officielle indique 4096 pour Haiku 4.5). En-dessous, rien n'est mis en cache **silencieusement** (`cache_creation_input_tokens = 0`) malgré le marqueur `cache_control`.
- **Vérifier les hits, ne pas supposer.** Si `cache_read_input_tokens` reste à 0 sur des préfixes identiques, un invalidateur silencieux est à l'œuvre (timestamp, JSON non trié, set d'outils variable, switch de modèle).
- **Hiérarchie d'invalidation à trois niveaux** (`tools` / `system` / `messages`). Changer `tool_choice` ou activer/désactiver le thinking n'invalide PAS le cache `tools`+`system` ; seuls un changement de définition d'outils ou de modèle force une reconstruction complète. Ne pas sur-optimiser les paramètres inoffensifs.
- **Fenêtre de lookback de 20 blocs.** Chaque breakpoint ne remonte que 20 blocs en arrière. Dans les boucles avec beaucoup de paires `tool_use`/`tool_result`, placer un breakpoint intermédiaire tous les ~15 blocs.
- **Changer de modèle invalide tout le cache** (caches scoped par modèle). Épingler l'ID exact (`claude-sonnet-4-6`) pour la reproductibilité ET la stabilité du cache.
- **Breakpoint mal placé.** Le mettre à la fin du prompt complet (incluant la question variable) au lieu de la fin du préfixe partagé : chaque requête écrit une entrée jamais relue — on paie le premium d'écriture sans aucun read.
- **Ne pas cacher du contenu lu une seule fois.** Le caching coûte 1,25× sans bénéfice si le préfixe n'est lu qu'une fois ; rentable seulement à partir de ~2 lectures (TTL 5 min).
- **`tiktoken` ment pour Claude.** C'est le tokenizer d'OpenAI ; il sous-estime de 15-20 % (davantage sur le code). Toujours utiliser `count_tokens` du fournisseur ciblé pour estimer coûts et tenir le contexte.
- **Batch API ≠ Zero Data Retention** chez Anthropic — point de conformité à vérifier si la veille traite des sources sensibles. Résultats conservés 29 jours, jusqu'à 100 000 requêtes ou 256 MB par batch.
- **Le coût caché de la qualité.** Un routing trop agressif vers Haiku peut produire des synthèses moins fiables, ruinant le différenciateur « confiance ». Le routing doit être **éval-driven**, pas seulement coût-driven : un Haiku qui hallucine une source détruit la valeur du produit.
- **Requêtes concurrentes.** N requêtes parallèles à préfixe identique paient toutes le plein tarif — une entrée de cache n'est lisible qu'une fois la première réponse commencée en streaming. Pour un fan-out : envoyer 1 requête, attendre le 1er token, puis lancer les N-1 autres.
- **Ne pas tronquer silencieusement** les entrées trop longues pour « faire rentrer » dans le contexte : préférer le chunking/résumé explicite et le signaler, sinon on produit des synthèses incomplètes et non fiables.

## Checklist d'application

- [ ] `usage` (input / output / cache_read / cache_creation) capturé et loggé à chaque appel
- [ ] Coût/synthèse calculé et exposé par techno et par version (OpenTelemetry)
- [ ] Coût tracé par étape du workflow 9 étapes
- [ ] System prompt **figé** : aucun `datetime.now()`, UUID ni ID de session avant le dernier breakpoint
- [ ] JSON sérialisé avec `sort_keys`, ordre des outils déterministe
- [ ] `cache_control` placé à la **fin du préfixe partagé**, pas après la question variable
- [ ] Préfixe ≥ seuil minimal (~1024 tokens pour Opus 4.8 / Sonnet 4.6)
- [ ] TTL choisi selon le trafic (5 min par défaut ; 1 h seulement si requêtes espacées de > 5 min)
- [ ] `cache_read_input_tokens > 0` vérifié sur des préfixes identiques (preuve d'économie)
- [ ] Routing Haiku / Sonnet / Opus défini par difficulté ET validé par évals
- [ ] ID de modèle épinglé (`claude-sonnet-4-6`, etc.) pour reproductibilité et stabilité du cache
- [ ] Batch API utilisé pour le non temps-réel (re-synthèses nocturnes, évals de régression)
- [ ] `max_tokens` borné au juste nécessaire ; tokens comptés avec `count_tokens` (jamais `tiktoken`)
- [ ] Conformité ZDR vérifiée si Batch API + sources sensibles

## 10 questions / réponses

**Q1. C'est quoi le prompt caching, en une phrase ?**
C'est la mise en cache d'un préfixe stable de prompt (instructions, doc source) de sorte que les requêtes suivantes qui partagent ce préfixe le relisent à ~0,1× du prix input au lieu du plein tarif (Anthropic — Prompt Caching).

**Q2. Pourquoi mesurer avant d'optimiser ?**
Parce que sans métrologie on optimise à l'aveugle. On instrumente le coût/synthèse dès J1 via l'objet `usage`, puis on cible les postes réels. C'est le « meter before you manage » des praticiens (applied-llms.org).

**Q3. Quel est le levier de coût n°1 pour TechWatch IA ?**
Le prompt caching du préfixe partagé (system prompt + doc source de la techno), car le projet recharge le même long-context à chaque question. La lecture cache à 0,1× donne ~90 % d'économie sur cette portion (Anthropic — Prompt Caching).

**Q4. Comment je vérifie qu'un cache hit a vraiment eu lieu ?**
On lit `usage.cache_read_input_tokens` dans la réponse : s'il reste à 0 sur des requêtes à préfixe identique, un invalidateur silencieux est en cause (timestamp, JSON non trié, set d'outils variable). Total entrée = `input + cache_creation + cache_read` (Anthropic — Prompt Caching).

**Q5. 5 minutes ou 1 heure de TTL ?**
TTL 5 min (écriture 1,25×) pour le trafic continu ; TTL 1 h (écriture 2×) seulement si les requêtes sont espacées de plus de 5 min. Le 5 min est rentable dès 2 lectures, le 1 h dès 3 lectures (Anthropic — Prompt Caching).

**Q6. Comment router entre Haiku, Sonnet et Opus ?**
Par difficulté : Haiku 4.5 ($1/$5) pour la classification/extraction/détection de doublon, Sonnet 4.6 ($3/$15) pour la synthèse standard, Opus 4.8 ($5/$25) pour les migrations complexes et le raisonnement long. « Choose the smallest model that gets the job done » (applied-llms.org).

**Q7. Sous-dimensionner partout pour économiser, bonne idée ?**
Non. Un routing trop agressif vers Haiku casse le différenciateur fiabilité/sourcé. Le routing doit être éval-driven : on valide la qualité avant de descendre en gamme. Un petit modèle bien promté (n-shot) peut égaler un gros modèle zero-shot, mais cela se prouve par l'éval (applied-llms.org ; Eugene Yan — LLM patterns).

**Q8. Quand utiliser le Batch API ?**
Pour le non temps-réel : re-synthèses planifiées, évals de régression, traitement de masse. −50 % sur tous les tokens, la plupart des batchs < 1 h (fenêtre max 24 h), jusqu'à 100k requêtes / 256 MB, cumulable avec le caching. Attention : non éligible au Zero Data Retention, résultats dans un ordre quelconque indexés par `custom_id` (Anthropic — Batch Processing).

**Q9. Peut-on charger toute la doc en contexte sans vector DB ?**
Oui, et c'est précisément ce que le caching rend viable : lire la doc source à 0,1× évite de maintenir une infra RAG/embeddings dès le MVP — cohérent avec « choose boring technology » et « the system around it is the product » (applied-llms.org ; Anthropic — Prompt Caching).

**Q10. Comment empiler tous les leviers ?**
De façon multiplicative, chacun sur un trafic différent : cache de préfixe (~90 % sur le contexte réutilisé) + routing vers le modèle le moins cher acceptable + Batch (−50 % sur le non temps-réel) + cache de réponse complet (100 % sur les répétitions exactes). On compte les tokens AVANT envoi avec `count_tokens` et jamais `tiktoken`, qui sous-estime de 15-20 % pour Claude (Anthropic — Prompt Caching ; applied-llms.org).

## Sources

1. [Anthropic — Prompt Caching (documentation officielle)](https://platform.claude.com/docs/en/build-with-claude/prompt-caching.md) — *official_docs* — référence sur le levier de coût n°1 : lecture cache 0,1×, écriture 1,25× (5 min) / 2× (1 h), seuils minimaux par modèle, invalidation par préfixe exact, vérification via `usage.cache_read_input_tokens`.
2. [OpenAI — Prompt Caching guide (documentation officielle)](https://developers.openai.com/api/docs/guides/prompt-caching) — *official_docs* — point de comparaison fournisseur : caching automatique sans changement de code (> 1024 tokens, incréments de 128), remise ~50 %, latence −80 %, pas de purge manuelle ; contraste avec le caching explicite d'Anthropic.
3. [Anthropic — Batch Processing / Message Batches API (documentation officielle)](https://platform.claude.com/docs/en/build-with-claude/batch-processing.md) — *official_docs* — remise 50 % sur tous les tokens, traitement asynchrone (< 1 h en général, fenêtre max 24 h), jusqu'à 100 000 requêtes ou 256 MB par batch, résultats conservés 29 jours, non éligible au Zero Data Retention.
4. [What We Learned from a Year of Building with LLMs (applied-llms.org)](https://applied-llms.org/) — *practitioner* — « choose the smallest model that gets the job done », exemple Haiku 10-shot battant Opus zero-shot, caching pour éliminer recomputation/latence, normalisation des entrées pour augmenter le taux de hit, « the model isn't the product, the system around it is ».
5. [Eugene Yan — Patterns for Building LLM-based Systems & Products](https://eugeneyan.com/writing/llm-patterns/) — *practitioner* — caching comme pattern de production (50 %+ d'économies), RAG, guardrails, et l'évaluation comme fondation — cohérent avec l'approche éval-driven du projet.
6. [OpenAI — Prompt Caching in the API (annonce officielle)](https://openai.com/index/api-prompt-caching/) — *official_docs* — annonce officielle (1er octobre 2024) datant le caching automatique et la remise 50 % ; utile pour dater le pinning de version et la comparaison fournisseurs.
