# Human-in-the-loop

> **En une phrase.** L'IA *suggère* chaque affirmation de la synthèse de migration et l'humain *valide / édite / rejette* : ce checkpoint protège le différenciateur « confiance » de TechWatch IA, réduit la charge cognitive du relecteur, et chaque action humaine alimente gratuitement le golden set — un *data flywheel* qui fait passer la revue de 100 % (bloc 1) à ciblée (bloc 2).

## Pourquoi c'est nécessaire pour TechWatch IA

Le différenciateur de TechWatch IA, c'est la **confiance** : chaque affirmation d'une synthèse (React 18→19, etc.) doit citer une source officielle vérifiable. Or un LLM renvoie une sortie *confiante même quand il ne devrait pas*, et le **hallucination floor de 5–10 % est tenace** — aucun prompt ne le supprime ([applied-llms.org](https://applied-llms.org/)). Publier automatiquement trahirait donc la promesse du produit. Le human-in-the-loop (HITL) est la réponse : au MVP, **chaque affirmation est une suggestion à valider, pas une vérité publiée**, et la revue humaine constitue le golden set initial qui rendra possible, plus tard, une éval automatique fiable.

## Le principe

Le paradigme à retenir : **« AI in the loop; Humans at the center »** ([applied-llms.org](https://applied-llms.org/)). L'IA assiste et propose ; l'humain reste le décideur. Concrètement, on privilégie le pattern UX **« l'IA suggère en temps réel → l'humain valide / édite »** plutôt que « tout auto » (qui trahit la confiance) ou « tout manuel » (qui ne passe pas à l'échelle). Ce pattern a trois vertus : il **réduit la charge cognitive**, il garde **transparence et contrôle**, et il crée **une boucle de feedback naturelle**.

Cette boucle se nourrit de deux types de signaux. Le **feedback implicite** est gratuit : une édition du relecteur, une acceptation, un reformatage sont des signaux honnêtes et volumineux (le modèle de Copilot/Midjourney). Le **feedback explicite** se rend granulaire et peu coûteux : boutons accept/reject/correct *par affirmation*, pouce haut/bas, champ de critique libre — Microsoft **G15** recommande d'« encourager le feedback granulaire pendant l'interaction régulière » ([Microsoft Research](https://www.microsoft.com/en-us/research/blog/guidelines-for-human-ai-interaction-design/)).

Pour que la revue scale, on construit un **visualiseur de données *domain-specific***, pas un dashboard générique : tout le contexte sur un écran (affirmation + source citée + diff de version), feedback en un clic, hotkeys, filtrage par type d'erreur ([hamel.dev](https://hamel.dev/blog/posts/field-guide/)). On pratique l'**error analysis bottom-up** : regarder de vrais échantillons chaque jour, noter les échecs en texte libre, *puis* laisser émerger une taxonomie — au lieu de définir des critères abstraits a priori. Les critères d'éval sont un **document vivant** : « il est impossible de déterminer complètement les critères avant d'avoir jugé des sorties ».

L'opérationnalisation suit le cycle **look → label → evaluate → optimize** ([eugeneyan.com](https://eugeneyan.com/writing/aligneval/)) avec des labels **binaires pass/fail** (plus rapides, moins de charge cognitive) + une **critique textuelle** expliquant le « pourquoi ». Ces corrections deviennent le golden set ; viser **50–100 exemples minimum** (20 est insuffisant), équilibrés et représentatifs. Et le mantra du juge LLM aligné : **« Align AI to human. Calibrate human to AI. Repeat. »**

## Exemples concrets

**1. Bloc 1 — revue 100 % sourcée (intuition + golden set initial).** Pour la synthèse React 18→19, chaque affirmation s'affiche avec sa citation officielle et trois actions. Chaque clic écrit un cas daté et versionné dans le golden set ; l'édition libre capture le feedback implicite (diff stocké).

```
┌──────────────────────────────────────────────────────────────┐
│ « useFormState devient useActionState. »                       │
│ Source : react.dev/reference/react/useActionState   (React 19) │
│ Diff : v18 useFormState → v19 useActionState                   │
│   [ Valider ]   [ Corriger ✎ ]   [ Rejeter ✗ ]                 │
└──────────────────────────────────────────────────────────────┘
        │
        └── action → golden_set/append({
              claim, source, verdict, edited_text?,
              version_tag: "react@19", reviewed_at, reviewer })
```

**2. Bloc 2 — revue ciblée + flywheel.** Un juge LLM note chaque affirmation (sourçabilité, fidélité à la source). On combine deux strates et on n'envoie en revue humaine que la part utile :

```python
def selectionner_pour_revue(affirmations, juge, taux_aleatoire=0.10):
    echec_eval  = [a for a in affirmations if not juge.passe(a)]      # strate ciblée
    reste       = [a for a in affirmations if juge.passe(a)]
    aleatoire   = echantillon(reste, k=int(taux_aleatoire * len(reste)))  # strate aléatoire
    pieges      = injecter_cas_pieges()  # ex. 2 sources React contradictoires
    return echec_eval + aleatoire + pieges  # le reste est publié

# Job hebdomadaire : re-mesurer l'accord juge↔humain et l'exactitude
# sur le golden set versionné (recall/precision/helpfulness),
# à la manière du data flywheel d'Agent-in-the-Loop.
```

**3. Capture de feedback granulaire (Microsoft G15/G9).** L'expert surligne une phrase et choisit une étiquette d'erreur — *source manquante*, *source obsolète (React 18)*, *contresens API*, *hallucination*. Ces labels nourrissent la taxonomie d'erreurs (error analysis bottom-up) et deviennent des classifieurs/guardrails ([HAX Toolkit](https://www.microsoft.com/en-us/haxtoolkit/ai-guidelines/)).

**4. Garde-fou de confiance (éval = guardrail).** Toute affirmation dont la confiance de sourçage est sous le seuil n'est **pas publiée mais mise en quarantaine** pour revue (Microsoft **G10**, « scope when in doubt »). Le data flywheel structuré d'Agent-in-the-Loop montre l'enjeu : capturé en live, il a apporté **+11,7 % recall@75, +14,8 % précision@8, +8,4 % helpfulness** et **réduit les cycles de retraining de mois à semaines** ([arXiv 2510.06674](https://arxiv.org/abs/2510.06674)).

## Subtilités & pièges à éviter

- **Éval ↔ guardrail sont interchangeables.** Une éval de cohérence factuelle devient un guardrail quand on *filtre* les sorties mal notées au lieu de les afficher : au MVP, une affirmation non-sourçable est bloquée par la même logique qui sert à l'éval.
- **Le feedback implicite est puissant mais ambigu.** Une non-correction peut signifier « bon » *ou* « l'humain n'a pas regardé ». Distinguer une **validation active** d'une **absence d'action**, sinon le flywheel apprend du bruit.
- **Automation bias.** En réduisant le seuil de revue, l'humain qui fait confiance valide sans lire. Réinjecter périodiquement des **cas piégés** (sources contradictoires) pour mesurer la vigilance réelle.
- **Sampling biaisé.** N'échantillonner *que* les échecs gonfle la perception d'erreur et déséquilibre le golden set, rendant le juge calibré dessus inutilisable. Garder une **strate aléatoire représentative** en plus de la strate « échec d'éval ».
- **Loi de Goodhart.** Sur-optimiser une métrique unique dégrade la performance globale (metric gaming). Maintenir **plusieurs critères** et continuer à regarder les données qualitativement.
- **Le golden set dérive avec la techno.** Un cas « correct » sur React 18 devient un faux positif sur React 19 : **dater et taguer chaque cas par version**, versionner le golden set comme du code.
- **L'alignement va dans les deux sens.** Parfois l'attente humaine est irréaliste vu l'état de l'art : calibrer aussi l'humain vers l'IA, pas seulement l'IA vers l'humain.
- **L'outillage des experts métier doit avoir une friction nulle.** Les domain experts améliorent souvent mieux l'IA que les ingénieurs seuls, mais un outil de revue trop technique annule ce gain — leur donner la **version admin de l'UI réelle**, pas un playground séparé, et traduire le jargon (« RAG » → « donner le bon contexte au modèle »).
- **Piège n°1 : définir des critères d'éval élaborés *avant* de regarder les données.** Produit des métriques génériques (« utile », « grammaire ») déconnectées des vrais défauts.
- **Piège : tout automatiser sans checkpoint dès le MVP** — trahit le différenciateur confiance et empêche de constituer le golden set initial.
- **Piège : dashboard générique** au lieu d'un visualiseur custom → feedback trop coûteux, revue qui ne scale pas, flywheel qui se tarit.
- **Piège : feedback theater.** Des pouces haut/bas que personne ne route vers un cas de golden set ou une mise à jour de prompt ne servent à rien.
- **Piège : passer à l'échantillon trop tôt**, avant d'avoir une intuition solide des modes d'échec via la revue 100 % du bloc 1 → on échantillonne à l'aveugle et on rate des classes d'erreurs entières.
- **Piège : déployer un juge LLM sans vérifier son alignement** ni le re-mesurer → la dérive silencieuse fait passer la revue ciblée à côté des vraies régressions.

## Checklist d'application

- [ ] Poser le paradigme **« AI in the loop; Humans at the center »** : aucune affirmation publiée sans validation au MVP.
- [ ] Implémenter le pattern **suggère → valide/édite** avec actions par affirmation (Valider / Corriger / Rejeter).
- [ ] Capturer le **feedback implicite** (édits, diffs, accept/reject) *et* le **feedback explicite granulaire** (étiquettes d'erreur, critique libre).
- [ ] Distinguer **validation active** vs **absence d'action** pour ne pas polluer le flywheel.
- [ ] Construire un **visualiseur domain-specific** (contexte complet, un clic, hotkeys, filtre par type d'erreur) — pas un dashboard générique.
- [ ] Faire de l'**error analysis bottom-up** quotidienne ; laisser émerger la taxonomie d'erreurs ; traiter les critères comme un **document vivant**.
- [ ] Utiliser des labels **binaires pass/fail + critique textuelle**.
- [ ] Bloc 1 : **revue 100 %** pour bâtir l'intuition et le golden set initial (**50–100 cas min**, équilibrés, représentatifs).
- [ ] Bloc 2 : **revue ciblée** = strate « échec d'éval » **+** strate aléatoire (~10 %) **+** cas piégés.
- [ ] Chaque correction → **cas versionné** du golden set, **tagué par version** (react@19) et daté.
- [ ] **Mettre en quarantaine** (G10) toute affirmation à faible confiance de sourçage plutôt que la publier.
- [ ] Job périodique : re-mesurer l'**accord juge↔humain** et l'exactitude sur le golden set ; corriger la dérive.
- [ ] Donner aux experts métier la **version admin de l'UI réelle** ; traduire le jargon ; friction d'outillage nulle.

## 10 questions / réponses

**Q1. C'est quoi le human-in-the-loop, en une idée ?**
L'IA suggère, l'humain décide : « AI in the loop; Humans at the center » ([applied-llms.org](https://applied-llms.org/)). Au lieu de publier automatiquement, le système propose chaque affirmation comme un brouillon à valider, éditer ou rejeter. L'humain reste responsable de ce qui sort.

**Q2. Pourquoi ne pas tout automatiser puisque le LLM est rapide ?**
Parce que le **hallucination floor de 5–10 %** est tenace : aucun prompt ne l'élimine ([applied-llms.org](https://applied-llms.org/)). Pour une plateforme dont le différenciateur est la confiance, publier des affirmations confiantes mais fausses détruirait la valeur. Le HITL et les guardrails downstream sont donc structurels, pas optionnels.

**Q3. Quel pattern UX choisir ?**
Le pattern **« l'IA suggère en temps réel → l'humain valide/édite »**, entre le « tout auto » et le « tout manuel » ([applied-llms.org](https://applied-llms.org/)). Il réduit la charge cognitive, garde transparence et contrôle, et génère une boucle de feedback naturelle. Microsoft formalise les moments-clés : invocation efficace (G7), rejet (G8), correction (G9), dégradation gracieuse en cas de doute (G10) ([Microsoft Research](https://www.microsoft.com/en-us/research/blog/guidelines-for-human-ai-interaction-design/)).

**Q4. Feedback implicite ou explicite ?**
Les deux. L'**implicite** (édits, acceptations, reformatages) est gratuit, volumineux et honnête ; l'**explicite** (accept/reject par affirmation, pouce, critique libre) est ciblé et granulaire — Microsoft G13 « apprendre du comportement » et G15 « encourager le feedback granulaire » ([Microsoft Research](https://www.microsoft.com/en-us/research/blog/guidelines-for-human-ai-interaction-design/)). Attention : une non-correction est ambiguë, ne pas la confondre avec une validation.

**Q5. Pourquoi un visualiseur custom plutôt qu'un dashboard générique ?**
Parce que la revue ne scale que si le feedback coûte presque rien : tout le contexte (affirmation + source + diff) sur un écran, feedback en un clic, hotkeys, filtre par type d'erreur ([hamel.dev](https://hamel.dev/blog/posts/field-guide/)). Un dashboard générique force l'humain à naviguer entre systèmes ; le coût d'effort tue le flywheel.

**Q6. Pourquoi des labels binaires pass/fail plutôt qu'une note 1–5 ?**
Parce qu'ils sont **plus rapides et moins coûteux en charge cognitive**, ce qui augmente le volume et la cohérence des labels ([eugeneyan.com](https://eugeneyan.com/writing/aligneval/)). On ajoute une **critique textuelle** pour le « pourquoi » : ces critiques servent ensuite de few-shots au juge LLM et améliorent sa fidélité.

**Q7. Combien d'exemples pour que la revue alimente un bon golden set ?**
Viser **50–100 labels minimum** ; 20 est insuffisant ([eugeneyan.com](https://eugeneyan.com/writing/aligneval/)). Surtout, le set doit être **équilibré** (mix bons/mauvais) et **représentatif** : un set biaisé vers les échecs rend le juge calibré dessus inutilisable.

**Q8. Pourquoi passer de la revue 100 % (bloc 1) à la revue ciblée (bloc 2), et comment ?**
La revue 100 % au lancement bâtit l'intuition des modes d'échec et le golden set initial ; passer à l'échantillon trop tôt fait rater des classes d'erreurs entières. Ensuite, on concentre l'attention là où elle informe le plus : **échantillonnage stratégique** = cas où l'accord juge↔humain est le plus faible + cas à fort impact + une part d'aléatoire pour les angles morts ([hamel.dev](https://hamel.dev/blog/posts/field-guide/)).

**Q9. Comment garder le juge LLM aligné dans le temps ?**
Mantra : **« Align AI to human. Calibrate human to AI. Repeat. »** ([eugeneyan.com](https://eugeneyan.com/writing/aligneval/)). On mesure régulièrement l'accord juge↔humain sur le golden set versionné et on corrige la dérive ; déployer un juge sans re-mesure le fait diverger silencieusement, et la revue ciblée passe alors à côté des vraies régressions.

**Q10. Concrètement, comment boucler le data flywheel et avec quel gain attendu ?**
Humains évaluent → annotations fines → mise à jour prompt/few-shot/classifieurs → re-mesure sur le golden set, en boucle. Le feedback doit être **capturé structuré en live** (préférences, adoption + rationale, pertinence, manque) — du feedback libre non structuré ne nourrit pas le flywheel. L'étude Agent-in-the-Loop rapporte **+11,7 % recall@75, +14,8 % précision@8, +8,4 % helpfulness** et une réduction des cycles de retraining **de mois à semaines** ([arXiv 2510.06674](https://arxiv.org/abs/2510.06674)).

## Sources

1. [What We've Learned From A Year of Building with LLMs — applied-llms.org](https://applied-llms.org/) — *praticiens reconnus (Yan, Husain, Bischof, Shankar, et al.)* — paradigme « AI in the loop; Humans at the center », pattern UX « suggère → valide/édite », feedback implicite vs explicite, data flywheel, hallucination floor 5–10 %, loi de Goodhart.
2. [A Field Guide to Rapidly Improving AI Products — Hamel Husain](https://hamel.dev/blog/posts/field-guide/) — *praticien reconnu (ex-GitHub/Airbnb)* — « look at your data », error analysis bottom-up, visualiseurs custom (un clic, hotkeys, contexte complet), autonomiser les domain experts, sampling stratégique, l'infra d'éval comme flywheel.
3. [AlignEval: Building an App to Make Evals Easy, Fun, and Semi-Automated — Eugene Yan](https://eugeneyan.com/writing/aligneval/) — *praticien reconnu (Principal Applied Scientist, Amazon) + outil open-source* — cycle look→label→evaluate→optimize, 50–100 labels min (20 insuffisant), labels binaires pass/fail, « Align AI to human. Calibrate human to AI. Repeat. », données équilibrées/représentatives.
4. [Guidelines for Human-AI Interaction (18 guidelines) — Microsoft Research](https://www.microsoft.com/en-us/research/blog/guidelines-for-human-ai-interaction-design/) — *doc officielle / recherche (CHI 2019, validé empiriquement)* — G7 invocation, G8 rejet, G9 correction, G10 scope/dégradation gracieuse, G11 expliquer pourquoi, G13 feedback implicite, G15 feedback granulaire, G17 contrôles globaux.
5. [Human-AI eXperience (HAX) Toolkit — AI Guidelines & Design Patterns — Microsoft](https://www.microsoft.com/en-us/haxtoolkit/ai-guidelines/) — *doc officielle (toolkit, design patterns + pitfalls)* — bibliothèque de design patterns (Problem/Solution/When/How/Benefits/Pitfalls) pour concevoir l'UI de validation/correction et la capture de feedback granulaire.
6. [Agent-in-the-Loop: A Data Flywheel for Continuous Improvement in LLM-based Customer Support (2025) — arXiv 2510.06674](https://arxiv.org/abs/2510.06674) — *publication académique avec pilote en production* — 4 types d'annotations live, feedback structuré réduisant les cycles de retraining de mois à semaines, gains mesurés (+11,7 % recall@75, +14,8 % précision@8, +8,4 % helpfulness).
