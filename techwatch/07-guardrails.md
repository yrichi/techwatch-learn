# Guardrails (garde-fous en couches)

> **En une phrase.** Un garde-fou unique ne suffit jamais : on empile des couches ordonnées du moins cher au plus cher (regex → classifieur → LLM-juge → humain), en séparant les contrôles d'entrée et de sortie, en chaînant validation de schéma et vérification factuelle, et en échouant *fermé* (fail-closed) dès qu'un doute subsiste.

## Pourquoi c'est nécessaire pour TechWatch IA

Le différenciateur de TechWatch IA est la **confiance** : chaque affirmation d'une synthèse de migration (« React 19 supprime `X` ») doit citer une source officielle vérifiable. Or les LLM hallucinent à un taux de base de 5–10 %, qu'il est difficile de descendre sous ~2 % même sur des tâches simples comme le résumé ([applied-llms.org](https://applied-llms.org/)). Le prompt seul ne ferme pas ce risque : il faut des contrôles programmatiques en aval. Les garde-fous en couches — **schéma JSON validé → check de cohérence factuelle → revue humaine** — sont l'infrastructure qui transforme une sortie « plausible » en sortie « publiable », et qui fait passer en échec sûr (revue humaine) ce qui ne peut être grounded sur la source citée.

## Le principe

Un garde-fou est un contrôle déterministe et programmatique placé **autour** du LLM, distinct du modèle lui-même ([applied-llms.org](https://applied-llms.org/)). On distingue deux familles, car elles traitent des modes d'échec différents :

- **Input guardrails** (avant le LLM) : sanitization, détection d'injection de prompt, limite de longueur, scrubbing PII, vérification que la source est dans l'allowlist. Ils protègent contre OWASP LLM01 (prompt injection) et LLM02 (fuite d'info) ([OWASP/Improving](https://www.improving.com/thoughts/owasp-top-10-llm-security-guide/)).
- **Output guardrails** (après le LLM) : validation de schéma, cohérence factuelle, toxicité, fuite PII, et surtout *Improper Output Handling* (LLM05) — ne jamais passer la sortie directement à un shell, du SQL, un `eval` ou du HTML.

La clé est la **cascade ordonnée du moins cher au plus cher** : on n'escalade vers une couche coûteuse que ce qui n'a pas été tranché en amont ([Datadog](https://www.datadoghq.com/blog/llm-guardrails-best-practices/), [Particula](https://particula.tech/blog/ai-guardrails-compared-nemo-guardrails-ai-llama-guard)) :

1. **Filtres statiques regex/heuristiques** (~ms) : motifs d'injection connus, secrets, taille.
2. **Classifieurs ML légers** (dizaines de ms) : Llama Prompt Guard / Llama Guard, détection PII et contenu nuisible. À privilégier sur le LLM-juge : meilleure précision, coût et latence inférieurs, et moins jailbreakables ([applied-llms.org](https://applied-llms.org/)).
3. **LLM-juge** (secondes) : réservé aux cas ambigus restants.
4. **Revue humaine** : la couche la plus chère, déclenchée *sur seuil* (chemin à fort enjeu, faible confiance), jamais systématiquement.

Deux principes transversaux : (a) **schéma ≠ vérité** — un JSON parfaitement valide peut contenir une affirmation hallucinée ; validation de forme et vérification factuelle sont deux checks orthogonaux à chaîner ([Lil'Log](https://lilianweng.github.io/posts/2024-07-07-hallucination/)). (b) **fail-closed** — si un garde-fou tombe en panne, time-out, ou si la confiance est sous le seuil, on bloque ou on retombe sur une réponse bornée (« non confirmé par la source »), jamais on ne laisse passer par défaut. Aucun outil unique ne couvre tous les modes d'échec, d'où la défense en profondeur ([arXiv 2402.01822](https://arxiv.org/pdf/2402.01822)).

## Exemples concrets

**1. Pipeline TechWatch sur React 18→19 (couches enchaînées)**

```text
INPUT GUARDRAIL
  - regex anti-injection ("ignore previous instructions"…)
  - allowlist domaines: l'URL source ∈ {react.dev, github.com/facebook/react}
        │
        ▼
GÉNÉRATION de la synthèse (LLM)
        │
        ▼
OUTPUT — couche 1 : SCHÉMA JSON strict
  chaque "claim" DOIT avoir source_url non vide  →  sinon re-prompt / reject
        │
        ▼
OUTPUT — couche 2 : COHÉRENCE FACTUELLE
  entailment NLI de chaque affirmation vs snippet de doc officielle cité
        │
        ▼
OUTPUT — couche 3 (escalade conditionnelle) : LLM-JUGE
  uniquement sur les claims marqués "faible support"
        │
        ▼
ÉCHEC SÛR : si un claim n'est pas grounded → l'item passe en REVUE HUMAINE
            (jamais publié automatiquement)
```

**2. Garde-fou anti-hallucination de citation (FActScore-like).** Pour « React 19 supprime `X` », on décompose en faits atomiques et on vérifie la présence de chacun dans la source citée. Si le fait n'est pas grounded, on retombe sur « non confirmé par la source » plutôt que de l'affirmer — c'est la calibration « je ne sais pas » et le cœur du différenciateur *confiance* ([Lil'Log](https://lilianweng.github.io/posts/2024-07-07-hallucination/)).

**3. Budget latence (justifie l'ordre des couches).** Coût cumulé d'une synthèse, en empilant du moins cher au plus cher :

| Couche | Outil typique | Latence | Trafic concerné |
|---|---|---|---|
| 1. Filtre statique | LLM Guard | quelques ms (single-digit→tens) | 100 % |
| 2. Rails entrée/dialog | NeMo Guardrails | < 50 ms (GPU) | 100 % |
| 3. Classifieur contenu | Llama Guard 3 | dizaines–centaines ms | survivants |
| 4. Validation sortie | Guardrails AI | 50–200 ms / check | sorties valides |
| 5. LLM-juge | (API) | ~secondes | ~10 % des claims |

(Chiffres : [Particula](https://particula.tech/blog/ai-guardrails-compared-nemo-guardrails-ai-llama-guard).) On ne paie les ~2 s du juge que sur la fraction non tranchée par les couches rapides.

## Subtilités & pièges à éviter

- **Le triangle vitesse / sûreté / précision n'est pas maximisable simultanément.** Chaque couche ajoute de la latence ET du taux de faux positifs. Le budget de latence se dépense couche par couche.
- **Schéma valide ≠ affirmation vraie.** Une synthèse de migration peut être parfaitement bien formée et factuellement fausse. Ne jamais confondre les deux checks.
- **Le LLM-juge est lui-même jailbreakable et non déterministe.** Le contrôler (CoT, comparaisons par paires plutôt qu'échelle Likert, contrôle du biais de position, double passage) et ne jamais le mettre en première ligne sur 100 % du trafic.
- **Les regex d'injection sont fragiles** face aux variantes/obfuscation : elles réduisent la surface, ne la ferment pas — d'où la nécessité des couches supérieures.
- **Fail-open par défaut = piège classique.** Time-out, exception ou doute d'un garde-fou doit basculer en blocage ou réponse bornée, jamais laisser passer.
- **Fail-closed trop agressif dégrade l'UX.** Un blocage sur faux positifs pousse les utilisateurs à contourner le système. Calibrer les seuils par criticité et offrir un chemin de réessai/escalade plutôt qu'un refus opaque.
- **Ne pas passer la sortie LLM directement à un shell/SQL/eval/HTML** (OWASP LLM05) : la traiter comme entrée non fiable (requêtes paramétrées, encodage de sortie).
- **Ne mesurer que le taux de blocage des attaques et ignorer les faux positifs** sur-estime la robustesse et bloque du trafic légitime.
- **Tester seulement en mono-tour est insuffisant** : les jailbreaks réels sont multi-tours et persistants.
- **Un seul outil ne couvre pas les trois fonctions** (flow control via NeMo/Colang, validation de structure via Guardrails AI, classification de contenu via Llama Guard) : c'est l'erreur de déploiement la plus fréquente ([Particula](https://particula.tech/blog/ai-guardrails-compared-nemo-guardrails-ai-llama-guard)).
- **Sur-investir dans du tooling custom de structured output** : les fournisseurs (function calling, JSON mode) convergent vers des sorties valides ; investir plutôt dans la cohérence factuelle et l'observabilité ([applied-llms.org](https://applied-llms.org/)).
- **SelfCheckGPT (auto-consistance) coûte N générations** : pertinent sans base externe, mais quand on possède déjà la source officielle (cas d'une veille sourcée), un check par entailment contre cette source unique est moins cher.

## Checklist d'application

- [ ] Séparer explicitement les **input guardrails** des **output guardrails** dans le code.
- [ ] Ordonner les couches **du moins cher au plus cher** ; n'escalader que les cas ambigus.
- [ ] **Valider la sortie contre un schéma JSON strict** avant toute consommation downstream (chaque `claim` a un `source_url` non vide).
- [ ] Utiliser une lib de **structured output** (Instructor côté API, Outlines en self-hosted) plutôt qu'un parsing maison.
- [ ] Ajouter un **check de cohérence factuelle** (entailment NLI / décomposition en faits atomiques) distinct de la validation de schéma.
- [ ] Combiner **amont** (CoT, grounding RAG) et **aval** (filtrer/régénérer les hallucinations).
- [ ] Implémenter le **fail-closed** : panne / time-out / faible confiance → blocage ou réponse bornée (« non confirmé par la source »).
- [ ] Déclencher la **revue humaine sur seuil**, pas systématiquement.
- [ ] **Ne jamais** passer la sortie LLM à un shell / SQL / eval / HTML sans encodage (OWASP LLM05).
- [ ] **Scrubber PII/secrets** en sortie (regex + classifieurs) ; marquer les données protégées par délimiteurs.
- [ ] Durcir le prompt système, **isoler les rôles**, gater les appels d'outils (moindre privilège, OWASP LLM01).
- [ ] **Mettre en cache** les réponses déjà validées.
- [ ] **Logger** chaque entrée/sortie et chaque déclenchement de garde-fou (bypass, faux positifs).
- [ ] **Tester de façon adversariale** et multi-tours ; mesurer aussi les **faux positifs**.
- [ ] Préférer **classifieurs/reward models** au LLM-juge quand c'est possible.
- [ ] Quand le retrieval ne renvoie rien de pertinent : retourner **« je ne sais pas »** déterministe.

## 10 questions / réponses

**Q1. C'est quoi un « guardrail » concrètement ?**
C'est un contrôle programmatique et déterministe placé autour du LLM (avant ou après l'appel) pour bloquer, filtrer ou régénérer une entrée/sortie indésirable. Il est distinct du modèle et constitue une infrastructure non négociable en production ([applied-llms.org](https://applied-llms.org/)).

**Q2. Pourquoi le prompt système ne suffit-il pas ?**
Parce que le modèle générera quand même des sorties indésirables : le prompt réduit la probabilité mais ne ferme pas le risque. Il faut le compléter par des garde-fous programmatiques en aval qui détectent et filtrent/régénèrent ([applied-llms.org](https://applied-llms.org/)).

**Q3. Quelle différence entre input et output guardrails ?**
Les input guardrails (avant le LLM) gèrent l'injection, la longueur, la PII, l'allowlist de sources. Les output guardrails (après) gèrent la validation de schéma, la cohérence factuelle, la toxicité et l'*Improper Output Handling*. Ils traitent des modes d'échec différents et ne sont pas interchangeables ([Datadog](https://www.datadoghq.com/blog/llm-guardrails-best-practices/)).

**Q4. Pourquoi ordonner les couches du moins cher au plus cher ?**
Pour ne dépenser le budget de latence et de coût que sur le trafic qui le mérite : les couches rapides (regex ~ms, classifieurs dizaines de ms) tranchent la majorité, et seul l'ambigu remonte au LLM-juge (~secondes) ([Particula](https://particula.tech/blog/ai-guardrails-compared-nemo-guardrails-ai-llama-guard)).

**Q5. Un JSON valide garantit-il une affirmation vraie ?**
Non. La validation de schéma contrôle la *forme* (champs, types), pas la *vérité*. Un JSON parfaitement valide peut contenir une hallucination. Schéma et cohérence factuelle sont deux checks orthogonaux à chaîner ([Lil'Log](https://lilianweng.github.io/posts/2024-07-07-hallucination/)).

**Q6. Comment vérifier la factualité en pratique ?**
On combine l'amont (CoT, grounding RAG) et l'aval (un garde-fou de cohérence factuelle). Techniques concrètes : décomposition en faits atomiques vérifiés contre la source (FActScore), entailment NLI, auto-consistance (SelfCheckGPT), Chain-of-Verification ([Lil'Log](https://lilianweng.github.io/posts/2024-07-07-hallucination/), [applied-llms.org](https://applied-llms.org/)).

**Q7. Faut-il toujours mettre un humain dans la boucle ?**
Non — c'est la couche la plus chère. On la réserve aux chemins à fort enjeu (sortie publiée, action sensible, faible confiance) et on la déclenche sur seuil, pas systématiquement ([OWASP/Improving](https://www.improving.com/thoughts/owasp-top-10-llm-security-guide/)).

**Q8. Que signifie « fail-closed » et quand le nuancer ?**
Fail-closed = en cas de panne, time-out ou confiance insuffisante, on bloque ou on retombe sur une réponse bornée, jamais on ne laisse passer. À nuancer pour l'UX : un blocage trop agressif (faux positifs) dégrade le produit ; calibrer les seuils par criticité et offrir un chemin de réessai ([OWASP/Improving](https://www.improving.com/thoughts/owasp-top-10-llm-security-guide/)).

**Q9. Pourquoi préférer un classifieur à un LLM-juge ?**
Parce que les classifieurs et reward models entraînés atteignent une meilleure précision, à coût et latence inférieurs, et sont moins jailbreakables qu'un LLM généraliste servant de garde. Le LLM-juge, lui, est non déterministe et attaquable ; on le réserve aux cas ambigus ([applied-llms.org](https://applied-llms.org/)).

**Q10. Un seul outil (NeMo, Guardrails AI, Llama Guard) suffit-il ?**
Non. Le flow control (NeMo/Colang), la validation de structure (Guardrails AI) et la classification de contenu (Llama Guard) couvrent des modes d'échec distincts. Croire qu'un seul outil couvre les trois est l'erreur de déploiement la plus fréquente ; aucune solution unique ne couvre toutes les exigences (toxicité, factualité, vie privée…), d'où la défense en profondeur ([Particula](https://particula.tech/blog/ai-guardrails-compared-nemo-guardrails-ai-llama-guard), [arXiv 2402.01822](https://arxiv.org/pdf/2402.01822)).

## Sources

1. [What We've Learned From A Year of Building with LLMs](https://applied-llms.org/) — *praticiens reconnus (Yan, Husain, Bartowski et al.)* — guardrails comme infrastructure ; classifieurs/reward models supérieurs au LLM-juge en coût/latence/précision ; taux de base d'hallucination 5–10 % et plancher ~2 % ; combo CoT amont + garde-fou de cohérence factuelle aval.
2. [LLM guardrails: Best practices for deploying LLM apps securely](https://www.datadoghq.com/blog/llm-guardrails-best-practices/) — *doc éditeur / observabilité* — architecture en couches du moins cher au plus cher (regex → classifieurs légers → LLM-in-the-loop), input vs output, validation de schéma, role isolation, scrubbing PII par regex et marquage par délimiteurs.
3. [OWASP Top 10 for LLMs: A Practitioner's Implementation Guide](https://www.improving.com/thoughts/owasp-top-10-llm-security-guide/) — *guide d'implémentation aligné OWASP* — LLM01 (injection), LLM02 (fuite), LLM05 (improper output handling) traduits en mitigations : schéma strict, jamais de sortie directe vers shell/SQL, scan PII/secrets, human-in-the-loop sur chemins sensibles.
4. [AI Guardrails Compared: NeMo vs Guardrails AI vs Llama Guard](https://particula.tech/blog/ai-guardrails-compared-nemo-guardrails-ai-llama-guard) — *comparatif outillage* — latences par couche (LLM Guard quelques ms, NeMo < 50 ms GPU, Guardrails AI 50–200 ms/check, Llama Guard 3 dizaines–centaines ms) ; pipeline en 5 étapes ordonné par coût ; fonctions distinctes (flow control, schéma, classification).
5. [Extrinsic Hallucinations in LLMs](https://lilianweng.github.io/posts/2024-07-07-hallucination/) — *praticien reconnu / synthèse technique* — méthodes de détection/mitigation utilisables comme garde-fou de sortie : FActScore, SelfCheckGPT, entailment NLI, RAG+édition (RARR/FAVA), Chain-of-Verification, calibration « je ne sais pas ».
6. [NVIDIA NeMo Guardrails (dépôt officiel)](https://github.com/NVIDIA-NeMo/Guardrails) — *doc officielle / outil open-source* — rails programmables (input, dialog, retrieval, execution, output) en DSL Colang ; découpage en étapes et usage de petits modèles/classifieurs pour le contrôle pré- et post-LLM à faible latence.
7. [Building Guardrails for Large Language Models](https://arxiv.org/pdf/2402.01822) — *publication académique (survey)* — besoin de garde-fous multi-exigences (toxicité, fairness, factualité, vie privée) et impossibilité d'une solution unique ; argumente formellement pour la défense en profondeur / approche en couches.
