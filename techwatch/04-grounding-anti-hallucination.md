# Grounding, citations & anti-hallucination

> **En une phrase.** Pour qu'une veille techno soit digne de confiance, chaque affirmation de migration doit pointer vers un passage réel d'une source officielle — sinon elle est traitée comme « non vérifiable » et renvoyée à `null`, jamais inventée.

## Pourquoi c'est nécessaire pour TechWatch IA

Le différenciateur de TechWatch IA, c'est la **confiance** : un seul faux breaking change (ex. annoncer à tort que `ReactDOM.render` est supprimé dans une version où il ne l'est pas) détruit la crédibilité du produit. C'est exactement ce que codifient les règles 1 (citations obligatoires) et 2 (« null si absent ») du projet. Le grounding et la citation vérifiable sont les mécanismes techniques qui transforment ces deux règles en garanties opérationnelles : aucune affirmation sans source pointable, aucune extrapolation à partir des connaissances internes du modèle. Dans un workflow déterministe à 9 étapes, ces garanties deviennent des étapes de validation testables, pas des vœux pieux.

## Le principe

Le **grounding** (ou *faithfulness*) mesure le degré auquel une réponse est effectivement supportée par les documents fournis — c'est l'opposé de l'hallucination (deepset). Attention : c'est une notion distincte de la *correctness*. Une affirmation peut être **vraie** (le modèle le « sait » par son pré-entraînement) tout en étant **non groundée** dans les sources fournies. Pour une veille de confiance, seule la faithfulness compte : une vérité non sourcée doit être traitée comme non vérifiable.

Trois leviers se combinent.

**1. La citation native plutôt que la citation par prompt.** L'API Citations d'Anthropic découpe les documents en phrases côté serveur et garantit que chaque `cited_text` pointe vers un passage réellement présent dans le document — le pointeur est *valide par construction*, le modèle ne peut pas fabriquer une référence (Anthropic, Citations docs). Le repérage dépend du type de document : index de caractères (0-indexés, fin exclusive) pour le texte brut, numéros de page (1-indexés) pour le PDF, index de bloc pour le *custom content*. Mesurée, cette approche native augmente la *recall accuracy* jusqu'à +15 % par rapport aux implémentations par prompt ; le client Endex rapporte une chute des hallucinations de source de 10 % à 0 % avec +20 % de références par réponse (Anthropic, blog Citations).

**2. La clause « no evidence, no answer ».** Instruire explicitement le modèle de renvoyer `null` / « non documenté » quand aucune source ne supporte une affirmation, et de signaler quand le contexte est insuffisant. C'est la traduction directe de la règle 2 et la défense n°1 contre un faux breaking change : sans instruction d'abstention testée, les LLM ont tendance à combler les trous avec leurs connaissances internes (potentiellement périmées sur une version récente comme React 19) plutôt qu'à s'abstenir (O'Reilly, *A Year of Building with LLMs*).

**3. La vérification en deux temps.** En amont, la *prévention* (chain-of-thought, entrées/sorties structurées, contexte dense et pertinent) ; en aval, la *détection* via un « factual inconsistency guardrail » qui filtre ou régénère les sorties non supportées (O'Reilly). Concrètement : décomposer chaque réponse en *claims* atomiques (style RAGAS), vérifier chaque claim contre le contexte, puis re-valider déterministiquement que le `cited_text` apparaît bien aux index annoncés — et rejeter sinon.

## Exemples concrets

**Exemple 1 — Pipeline React 18→19 sourcé (custom content).** On ingère le *React 19 Upgrade Guide* via l'API Citations en utilisant des `custom content documents` : un bloc = une section de breaking change, sans re-chunking automatique. Chaque entrée de la synthèse porte alors une citation pointant vers un bloc précis.

```python
# Étape « grounding » du workflow déterministe : un bloc citable par breaking change
resp = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=4096,
    messages=[{
        "role": "user",
        "content": [
            {
                "type": "document",
                "source": {
                    "type": "content",
                    "content": [
                        {"type": "text", "text": "ReactDOM.render n'est plus supporté. Utilisez createRoot."},
                        {"type": "text", "text": "Les string refs sont supprimées au profit des callback refs."},
                    ],
                },
                "title": "React 19 Upgrade Guide",
                "context": "Source officielle react.dev",
                "citations": {"enabled": True},
                "cache_control": {"type": "ephemeral"},  # changelog long -> prompt caching
            },
            {"type": "text", "text": "Liste les breaking changes, un par affirmation, sourcés."},
        ],
    }],
)
```

**Exemple 2 — Vérification déterministe du span (étape aval).** Après génération, on re-valide que chaque `cited_text` est bien un sous-texte du bloc annoncé. C'est une étape parfaite pour un workflow déterministe : pas de LLM, juste une comparaison.

```python
def verifier_citation(bloc_source: str, cite: dict) -> bool:
    # Le pointeur natif est VALIDE (existe), mais on revérifie le span et le SUPPORT en aval
    return cite["cited_text"].strip() in bloc_source

# Si un claim n'a aucune citation valide -> on le marque "non vérifiable", on ne publie pas.
```

**Exemple 3 — Test d'abstention « no evidence, no answer ».** On injecte le changelog React 19, puis on demande une affirmation absente (ex. « React 19 supprime-t-il les class components ? »). Le système **doit** répondre `null` / « non documenté dans les sources » au lieu d'inventer. C'est un cas d'éval binaire : `correct = (réponse == abstention)`.

## Subtilités & pièges à éviter

- **Correctness ≠ Faithfulness.** Une affirmation vraie mais non sourcée reste non vérifiable pour TechWatch IA. Ne jamais publier ce que le modèle « sait » sans citation.
- **Le pointeur est valide, pas forcément pertinent.** Anthropic garantit que `cited_text` *existe* dans le document, **pas** qu'il *supporte sémantiquement* le claim. La vérification du support reste nécessaire (LLM-judge + revue).
- **Le « partial support » est l'angle mort.** Les métriques automatiques distinguent mal une source qui supporte *partiellement* d'une qui supporte *totalement* ou *pas du tout* — exactement le cas d'un breaking change nuancé (arXiv 2408.12398). Flaguer « à vérifier » plutôt qu'affirmer.
- **Le LLM-as-judge sature vers 1.0** et tend à être trop indulgent ; RAGAS et DeepEval donnent des taux d'hallucination très différents sur les mêmes données. Ne pas prendre un score de juge unique pour argent comptant — croiser avec la vérification programmatique du span.
- **Le grounding ne sauve pas une mauvaise source.** Citer parfaitement une RC présentée comme finale, ou un blog tiers erroné, produit une affirmation fausse avec une mécanique de citation impeccable. La **fiabilité de la source prime** sur la mécanique de citation.
- **Citations natives chunke en PHRASES par défaut** : un breaking change décrit en tableau ou sur plusieurs phrases peut être mal découpé. Utiliser le `custom content` pour contrôler le grain. Les PDF scannés sans texte extractible ne sont **pas** citables.
- **Piège d'architecture : Citations est INCOMPATIBLE avec les Structured Outputs** chez Anthropic — activer `citations` + `output_config.format` ensemble renvoie une **erreur 400** (Anthropic, Citations docs). Séparer l'étape « citation/grounding » de l'étape « extraction structurée en JSON ».
- **Le taux d'hallucination résiduel tombe rarement sous ~2 %** même avec un grounding fort (O'Reilly). Concevoir le système en supposant qu'il reste des erreurs : d'où l'importance du filtre aval et de la traçabilité.
- **Ne pas injecter en vrac d'énormes documents** en pensant que « plus de contexte = moins d'hallucination ». La densité et la pertinence priment ; le bruit dilue l'attention et peut dégrader la faithfulness.
- **Anti-pattern classique :** « cite tes sources » dans le system prompt sans validation aval. Le modèle peut fabriquer des numéros de section/URL plausibles — c'est le mécanisme des fausses jurisprudences ChatGPT (OWASP LLM09).
- **Coût caché de la verifiability :** re-matching des spans, NLI, juge ajoutent latence et coût. À budgéter dans l'architecture serverless event-driven.

## Checklist d'application

- [ ] Toutes les sources ont `citations: {enabled: true}` (tout ou rien dans une requête).
- [ ] Chaque affirmation de migration porte au moins une citation valide ; sinon → `null` / « non documenté ».
- [ ] Une étape déterministe re-vérifie que chaque `cited_text` existe littéralement dans le document source aux index annoncés ; rejet sinon.
- [ ] La clause d'abstention est **explicite dans le prompt ET testée** par une éval binaire dédiée.
- [ ] Les réponses sont décomposées en claims atomiques, vérifiés claim-par-claim (pas réponse-par-réponse).
- [ ] Un score de groundedness/faithfulness (0-1) est calculé et suivi comme métrique de production de premier ordre (au même titre que coût et latence).
- [ ] Les évals sont **binaires** (« ce claim est-il supporté par cette source ? oui/non ») plutôt que des échelles de Likert.
- [ ] La vérification combine LLM-as-judge + vérification programmatique du span + revue humaine sur échantillon.
- [ ] Le pipeline **ne mélange jamais** Citations et Structured Outputs dans une même requête (sinon 400).
- [ ] Le `cache_control` est posé sur les documents source longs (changelogs, release notes) pour réduire les coûts en long-context.
- [ ] Les sources sont **officielles** (release notes, migration guides, RFC), versionnées correctement (pas de RC présentée comme finale).
- [ ] Le livrable affiche la source de chaque breaking change, marque visuellement les affirmations non sourcées, et rend chaque citation cliquable vers le passage exact.

## 10 questions / réponses

**Q1. C'est quoi le « grounding » en une phrase ?**
C'est le degré auquel une réponse est réellement supportée par les documents fournis — synonyme de *faithfulness*, opposé de l'hallucination (deepset). Pour la veille, c'est la garantie que chaque affirmation s'appuie sur une source citable et non sur les connaissances internes du modèle.

**Q2. Pourquoi préférer la citation native d'Anthropic à un « cite tes sources » dans le prompt ?**
Parce que la citation par prompt n'est pas vérifiée : le modèle peut fabriquer des numéros de section ou des URL plausibles (OWASP LLM09). L'API Citations chunke les documents côté serveur et garantit que chaque `cited_text` pointe vers un passage réel ; elle augmente la *recall accuracy* jusqu'à +15 % et fait chuter les hallucinations de source de 10 % à 0 % chez Endex (Anthropic, blog Citations).

**Q3. Comment l'API repère-t-elle un passage cité ?**
Selon le type de document : index de caractères (0-indexés, fin exclusive) pour le texte brut, numéros de page (1-indexés) pour le PDF, index de bloc pour le *custom content* (Anthropic, Citations docs). Chaque citation porte aussi un `document_index` 0-indexé.

**Q4. Concrètement, que veut dire la règle 2 « null si absent » ?**
Que le système doit renvoyer `null` ou « non documenté » dès qu'aucune source ne supporte une affirmation, et signaler un contexte insuffisant. Sans cette clause d'abstention testée, le modèle comble les trous avec ses connaissances internes — potentiellement périmées sur une version récente (O'Reilly).

**Q5. Une affirmation vraie mais non sourcée, je la garde ?**
Non. *Correctness* ≠ *faithfulness* : une vérité non groundée dans les sources fournies est traitée comme non vérifiable. Pour un produit dont le différenciateur est la confiance, seule la faithfulness compte.

**Q6. Comment mesure-t-on la faithfulness en pratique ?**
On décompose la réponse en claims atomiques (style RAGAS), on vérifie chaque claim contre le contexte, et on calcule un score de groundedness (0-1) par sortie qu'on suit comme métrique de production (deepset). Les évals doivent être **binaires** (supporté oui/non), plus reproductibles que des échelles de Likert (O'Reilly).

**Q7. Un score de LLM-judge suffit-il à prouver le grounding ?**
Non. Le juge sature vers 1.0 et est trop indulgent ; RAGAS et DeepEval divergent fortement sur les mêmes données. Et aucune métrique automatique unique n'excelle partout — la vérification entièrement automatisée des citations reste non résolue et nécessite une validation humaine (arXiv 2408.12398). Il faut croiser juge + vérification programmatique du span + revue sur échantillon.

**Q8. Quel est le cas le plus dangereux pour un breaking change ?**
Le « partial support » : les évaluateurs distinguent mal une source qui supporte *partiellement* d'une qui supporte *totalement* ou *pas du tout* (arXiv 2408.12398) — exactement le cas d'un breaking change nuancé. La bonne réponse est de flaguer « à vérifier » plutôt que d'affirmer.

**Q9. Quel piège d'architecture guette le pipeline Anthropic ?**
Activer Citations **et** Structured Outputs dans la même requête : ça renvoie une **erreur 400** (Anthropic, Citations docs). Il faut séparer l'étape « citation/grounding » de l'étape « extraction structurée en JSON », ou citer d'abord puis post-parser.

**Q10. Si la mécanique de citation est parfaite, le résultat est-il forcément fiable ?**
Non, à deux titres. D'abord, le pointeur natif est *valide* (il existe) mais pas forcément *pertinent* sémantiquement — le support reste à vérifier. Ensuite, le grounding ne sauve pas une mauvaise source : citer une RC présentée comme finale ou un blog tiers erroné produit une affirmation fausse impeccablement citée. La fiabilité de la source prime, et le risque légal d'une fausse autorité est réel (OWASP LLM09 : Air Canada, fausses jurisprudences).

## Sources

1. [Citations - Claude API Docs (Anthropic)](https://platform.claude.com/docs/en/build-with-claude/citations) — *official-docs* — mécanisme exact des Citations : chunking en phrases côté serveur, formats de repérage (char index / page / bloc), garantie que `cited_text` pointe vers un passage réel, incompatibilité avec les Structured Outputs (400), compatibilité prompt caching / batch, et contrôle du grain via *custom content*.
2. [Introducing Citations on the Anthropic API](https://claude.com/blog/introducing-citations-api) — *official-blog* — métriques chiffrées : +15 % de recall accuracy vs prompt-based, et chez Endex hallucinations de source 10 % → 0 % avec +20 % de références par réponse.
3. [What We Learned from a Year of Building with LLMs (Part I)](https://www.oreilly.com/radar/what-we-learned-from-a-year-of-building-with-llms-part-i/) — *practitioner* — taux d'hallucination résiduel (~2-10 %), approche en deux temps (prévention amont + guardrail de détection aval), évals binaires plutôt que Likert, et instruction d'abstention quand aucune source ne suffit.
4. [LLM09:2025 Misinformation - OWASP Top 10 for LLM Applications](https://genai.owasp.org/llmrisk/llm092025-misinformation/) — *standard* — cadre sécurité : hallucination / fausse autorité comme vulnérabilité avec conséquences légales (Air Canada, fausses jurisprudences), mitigations (RAG sur sources vérifiées, cross-verification, supervision humaine, design d'UI contre l'overreliance).
5. [Measuring LLM Groundedness in RAG Systems with Evaluation Metrics (deepset)](https://www.deepset.ai/blog/rag-llm-evaluation-groundedness) — *practitioner* — définition opérationnelle de la groundedness (= faithfulness, opposé de l'hallucination), Groundedness Score 0-1 par réponse, et son usage pour optimiser le pipeline.
6. [A Comparative Analysis of Faithfulness Metrics and Humans in Citation Evaluation](https://arxiv.org/abs/2408.12398) — *academic* — aucune métrique automatique unique n'excelle partout, le « partial support » est l'angle mort, schéma à trois niveaux (full / partial / no support), et la vérification entièrement automatisée reste non résolue (validation humaine nécessaire).
7. [Awesome Hallucination Detection (EdinburghNLP)](https://github.com/EdinburghNLP/awesome-hallucination-detection) — *reference-list* — liste curatée des techniques de détection d'hallucination (NLI post-hoc, self-consistency, abstention…), point d'entrée pour concevoir la couche de vérification automatique du workflow.
