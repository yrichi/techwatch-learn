# Provenance & versionnement du contenu

> **En une phrase.** Capturer le contenu brut tel quel et l'immuabiliser par un hash (SHA-256), versionner les synthèses Markdown dans Git pour un audit trail gratuit, et attacher à chaque synthèse une attestation de provenance (sources hashées + modèle + prompt) — parce que la *confiance* de TechWatch IA ne tient que si chaque affirmation reste vérifiable même quand la source disparaît.

## Pourquoi c'est nécessaire pour TechWatch IA

TechWatch IA vend de la **confiance** : chaque affirmation d'une synthèse de migration doit citer une source vérifiable. Or les pages officielles changent, déplacent ou suppriment leur contenu : une citation qui ne pointe que vers une URL devient invérifiable dès que `react.dev` réédite sa page. Conserver le brut + son hash, tracer l'origine, et versionner les synthèses dans Git transforme « je l'ai lu quelque part » en « voici exactement ce que disait la source ce jour-là, prouvable par hash ». C'est aussi une assurance conformité : l'EU AI Act impose de documenter origine, transformations et qualité des données pour les systèmes à risque ([Atlan](https://atlan.com/know/training-data-lineage-for-llms/)).

## Le principe

**Capturer le brut, ne jamais le muter.** À l'ingestion, on stocke le contenu *tel quel* avec ses métadonnées : URL source, date/heure de capture (UTC + timezone), en-têtes HTTP utiles (`ETag`, `Last-Modified`) et le tag de version de la techno (ex. `v19.0.0`). On **hashe** ce brut en SHA-256 et on l'enregistre dans un stockage *content-addressable* : un même contenu donne toujours le même hash, et la moindre modification change le hash. C'est la base de la déduplication, de l'immuabilité et de la détection de tampering ([C2PA](https://c2paviewer.com/articles/what-is-c2pa)).

**Séparer le brut de Git.** Le gros contenu brut vit hors-Git (object store type S3, DVC, Git-LFS) ; dans Git on ne commite que des **pointeurs/manifestes** légers `{url, hash, date, taille}`. Git reste léger et redevient la *source de vérité du lineage* ([DVC](https://doc.dvc.org/start/data-pipelines/data-pipelines)).

**Versionner les synthèses dans Git.** Chaque synthèse est un fichier Markdown commité (ex. `migrations/react-18-to-19/v1.md`). L'historique Git — auteur, date, diff, commit hash — fournit un audit trail *gratuit* et un versionnement sémantique des migrations.

**Matérialiser le lineage explicitement.** On modélise la chaîne « source brute → retraitement → synthèse » avec le vocabulaire W3C PROV : **Entity** (brut, synthèse), **Activity** (fetch, nettoyage, génération LLM), **Agent** (worker, modèle, humain), reliés par `wasDerivedFrom`, `wasGeneratedBy`, `wasAttributedTo`, sérialisés en JSON-LD à côté de l'artefact ([W3C PROV](https://www.w3.org/TR/prov-overview/)).

**Rendre le pipeline reproductible et incrémental.** On déclare les étapes (sources → nettoyage → chunking → synthèse) avec leurs inputs/outputs et on **fige les hash dans un lockfile** (modèle `dvc.lock`) : seules les étapes dont les inputs ont changé sont rejouées, et chaque étape est rejouable à l'identique ([DVC](https://doc.dvc.org/start/data-pipelines/data-pipelines)). Enfin, on **pinne le modèle et le prompt** (model id exact + version du prompt-template) car les fournisseurs changent leurs modèles silencieusement ([applied-llms.org](https://applied-llms.org/)).

## Exemples concrets

**1. Capture brute datée et hashée (React 18→19).** À la détection de React 19, on capture le `CHANGELOG.md`, le guide « React 19 Upgrade Guide » et le blog `react.dev`, chacun stocké avec sa preuve d'origine.

```python
import hashlib, datetime

def capture(url: str, raw: bytes, headers: dict, tech_tag: str) -> dict:
    # Normaliser AVANT de hasher : UTF-8, fins de ligne LF, pas de BOM
    canon = raw.decode("utf-8-sig").replace("\r\n", "\n").encode("utf-8")
    digest = hashlib.sha256(canon).hexdigest()
    blob_store.put(digest, canon)          # content-addressable, hors Git
    return {                                # ce dict est commité dans Git
        "url": url,
        "sha256": digest,
        "captured_at": datetime.datetime.now(datetime.UTC).isoformat(),
        "etag": headers.get("ETag"),
        "last_modified": headers.get("Last-Modified"),
        "tech_tag": tech_tag,              # ex. "v19.0.0"
        "bytes": len(canon),
    }
```

**2. Sidecar de provenance (esprit SLSA/in-toto).** Pour chaque synthèse Git, on commite `v1.md` **et** `v1.prov.json`. L'attestation rend la synthèse rejouable et vérifiable.

```json
{
  "_type": "https://in-toto.io/Statement/v1",
  "subject": [{ "name": "react-18-to-19/v1.md",
                "digest": { "sha256": "ab12…" } }],
  "predicate": {
    "buildDefinition": {
      "externalParameters": {
        "prompt_template": { "id": "migration-synth", "version": "3",
                             "sha256": "9f01…" },
        "temperature": 0
      },
      "resolvedDependencies": [
        { "uri": "https://github.com/facebook/react/blob/main/CHANGELOG.md",
          "digest": { "sha256": "c3d4…" } },
        { "uri": "https://react.dev/blog/2024/12/05/react-19",
          "digest": { "sha256": "e5f6…" } }
      ]
    },
    "runDetails": {
      "builder": { "id": "techwatch-worker", "version": "1.4.2" },
      "metadata": { "invocationId": "run-2026-06-23-001",
                    "startedOn": "2026-06-23T10:00:00Z" }
    }
  }
}
```

Les noms de champs `subject`/`digest`, `buildDefinition.externalParameters`/`resolvedDependencies` et `runDetails.builder.id`/`metadata.invocationId` suivent la spec SLSA v1.0 ([SLSA](https://slsa.dev/spec/v1.0/provenance)).

**3. Citation au hash, pas seulement à l'URL.** Une affirmation de la synthèse (« React 19 retire `propTypes` ») pointe vers `{url, sha256, section}` du brut capturé : on peut prouver que la source disait bien cela ce jour-là, même si la page change ensuite. Quand React publie 19.1, on rejoue le pipeline ; grâce aux hash, seules les sources réellement modifiées déclenchent un retraitement, et le diff Git entre `v1.md` et `v2.md` montre quelles affirmations ont changé et pourquoi ([AWS](https://aws.amazon.com/blogs/machine-learning/end-to-end-lineage-with-dvc-and-amazon-sagemaker-ai-mlflow-apps/)).

## Subtilités & pièges à éviter

- **Normaliser avant de hasher.** Encodage, BOM, CRLF vs LF, ordre des clés JSON, espaces : du bruit crée de faux « changements » de source, déclenche des retraitements inutiles et casse la déduplication. Décider d'un *canon* et le documenter.
- **Les dates déclarées par la source mentent.** `Last-Modified`/`ETag` sont souvent faux ou absents ; la date de capture côté système (quand *vous* avez récupéré) est plus fiable que toute date annoncée par la page.
- **Provenance-preserving > provenance-inferring.** Citer exactement le passage récupéré (preserving — ce que vise le projet) est plus fort que déduire a posteriori d'où vient une affirmation (inferring), qui est faillible.
- **Logger les I/O LLM est nécessaire mais insuffisant.** Un log dit *ce qui s'est passé et quand*, pas *qu'un passage supportait réellement une affirmation*. Le lineage doit relier `affirmation → evidence`, pas seulement horodater.
- **Granularité de la citation.** Une provenance au niveau document est trop grossière : il faut descendre au niveau *claim/passage* pour que « chaque affirmation cite sa source » soit réellement vérifiable.
- **Deux niveaux de reproductibilité.** Reproduire le *pipeline* (mêmes étapes, mêmes inputs hashés) est déterministe ; reproduire la *sortie* d'un LLM ne l'est pas toujours — d'où l'intérêt de figer aussi la sortie générée comme artefact versionné.
- **PIÈGE — ne stocker que l'URL.** La page change ou disparaît (lien mort, paywall, édition silencieuse) et la synthèse devient invérifiable : l'exact contraire du différenciateur « confiance ».
- **PIÈGE — mettre le brut dans Git.** Dépôts qui explosent, clones lents. Pointeurs + object store, jamais des blobs.
- **PIÈGE — réécrire l'historique Git** (`force-push`, rebase, squash agressif) sur les synthèses : détruit l'audit trail « gratuit ». L'immuabilité de l'historique est une exigence — protéger la branche et/ou signer les commits.
- **PIÈGE — oublier model id / version du prompt.** Le fournisseur met à jour le modèle, les sorties dérivent, on ne peut plus reproduire ([applied-llms.org](https://applied-llms.org/)).
- **PIÈGE — auditer en scannant les artefacts.** Ça ne passe pas à l'échelle (anti-pattern AWS) ; au-delà de quelques runs, il faut un index/manifeste interrogeable.
- **PIÈGE — retraitement boîte noire non versionné.** Si le code de nettoyage/chunking n'est pas versionné et lié à la sortie, deux exécutions divergent sans trace.

## Checklist d'application

- [ ] À l'ingestion, je capture le brut **tel quel** + `{url, captured_at (UTC), ETag, Last-Modified, tech_tag}`.
- [ ] Je **normalise** (UTF-8, LF, sans BOM) selon un canon documenté **avant** de hasher en SHA-256.
- [ ] Le brut est stocké hors-Git (S3/DVC/LFS) ; seuls les **pointeurs/manifestes** sont commités.
- [ ] Chaque synthèse est un fichier Markdown versionné dans Git (`migrations/<from>-to-<to>/vN.md`).
- [ ] Chaque synthèse a son **sidecar** `vN.prov.json` (sources hashées + model id + hash du prompt + paramètres + run id).
- [ ] Le **modèle** et le **prompt-template** sont pinnés (id + version).
- [ ] Chaque affirmation cite `{url + hash + section/offset}`, pas seulement l'URL.
- [ ] Le pipeline est déclaré avec un **lockfile** (hash des inputs) pour rejouer/diffuser de façon incrémentale.
- [ ] L'historique est protégé : branche protégée, **commits signés** (GPG/sigstore), pas de force-push.
- [ ] Un **manifeste interrogeable** (CSV/JSON versionné) relie chaque brut aux synthèses qui l'ont consommé.
- [ ] Pour l'audit critique : **S3 Object Lock** (mode compliance) sur le brut + journal append-only (CloudTrail-like).
- [ ] Une **politique de rétention** ne casse jamais les hash référencés par d'anciennes synthèses encore vivantes.

## 10 questions / réponses

**Q1 — Pourquoi conserver le contenu brut au lieu de ne garder que l'URL ?**
Parce que les pages officielles changent, déplacent ou suppriment leur contenu. Sans le brut capturé et son hash, une citation devient invérifiable, ce qui ruine le différenciateur « confiance » du projet. Stocker `{url, hash, date}` permet de prouver ce que disait la source ce jour-là.

**Q2 — À quoi sert le hash SHA-256 exactement ?**
Un même contenu produit toujours le même hash, et la moindre modification le change. Cela donne trois propriétés : déduplication (stockage content-addressable), immuabilité, et détection de tampering. Le hash devient l'identifiant stable cité par la synthèse ([C2PA](https://c2paviewer.com/articles/what-is-c2pa)).

**Q3 — Pourquoi ne pas mettre le gros contenu brut directement dans Git ?**
Git n'est pas conçu pour les gros blobs : le dépôt explose, les clones deviennent lents et l'historique impossible à manipuler. On stocke le brut hors-Git (S3, DVC, Git-LFS) et on ne commite que des pointeurs `{url, hash, date, taille}` ([DVC](https://doc.dvc.org/start/data-pipelines/data-pipelines)).

**Q4 — En quoi versionner les synthèses dans Git donne un « audit trail gratuit » ?**
Chaque synthèse est un fichier commité ; Git enregistre auteur, date, diff et commit hash sans infra supplémentaire. On obtient un versionnement sémantique (`v1.md`, `v2.md`) et la traçabilité de qui a validé quoi. Attention : c'est « gratuit » seulement si l'historique reste immuable (pas de force-push/rebase).

**Q5 — Qu'apporte le modèle W3C PROV par rapport à un format ad hoc ?**
Un vocabulaire interopérable et sérialisable (JSON-LD) : Entity / Activity / Agent et des relations standard (`wasDerivedFrom`, `wasGeneratedBy`, `wasAttributedTo`). On exprime le lineage « brut → retraitement → synthèse » de manière auditable plutôt qu'inventée maison ([W3C PROV](https://www.w3.org/TR/prov-overview/)).

**Q6 — Pourquoi pinner le model id et la version du prompt dans la provenance ?**
Parce que les fournisseurs changent leurs modèles silencieusement : sans pin, les sorties dérivent et la reproductibilité est illusoire. Enregistrer model id exact + version du prompt-template versionné permet d'expliquer et rejouer une ancienne synthèse ([applied-llms.org](https://applied-llms.org/)).

**Q7 — Que doit contenir l'attestation attachée à chaque synthèse ?**
Dans l'esprit SLSA/in-toto : le `subject` (artefact + digest), les `resolvedDependencies` (hash des sources brutes), les `externalParameters` (hash du prompt, température) et `runDetails` (builder.id, invocationId, timestamps). C'est ce qui rend la synthèse vérifiable et rejouable ([SLSA](https://slsa.dev/spec/v1.0/provenance)).

**Q8 — Pourquoi citer un hash et une section, pas juste une URL ?**
Une URL seule ne prouve rien si la page change. Pointer vers `{url + hash + offset/section}` du brut capturé permet de retrouver le passage exact tel qu'il était à la capture, même si la page est éditée ou supprimée ensuite. C'est de la provenance *preserving*, plus forte que l'*inferring*.

**Q9 — Comment rendre le retraitement reproductible et incrémental ?**
On déclare les étapes (sources → nettoyage → chunking → synthèse) avec leurs inputs/outputs et on fige les hash dans un lockfile (modèle `dvc.lock`). Seules les étapes dont les inputs ont changé sont rejouées, et le code de retraitement doit lui-même être versionné et lié à la sortie ([DVC](https://doc.dvc.org/start/data-pipelines/data-pipelines)).

**Q10 — Comment durcir l'immuabilité quand l'audit est critique (conformité) ?**
Combiner plusieurs couches : S3 Object Lock en mode compliance pour le brut, journal append-only type CloudTrail, et commits Git signés (GPG/sigstore) pour empêcher la réécriture d'historique. Pour passer à l'échelle, maintenir un manifeste interrogeable plutôt que scanner les artefacts. C'est aussi un atout conformité EU AI Act ([Atlan](https://atlan.com/know/training-data-lineage-for-llms/), [AWS](https://aws.amazon.com/blogs/machine-learning/end-to-end-lineage-with-dvc-and-amazon-sagemaker-ai-mlflow-apps/)).

## Sources

1. [DVC — Get Started: Data Pipelines (dvc.yaml / dvc.lock)](https://doc.dvc.org/start/data-pipelines/data-pipelines) — *Documentation officielle* — versionner le brut hors-Git via content-addressable storage + pointeurs `.dvc`, lockfile figeant les hash, et `dvc repro` qui ne rejoue que les étapes dont les inputs ont changé.
2. [End-to-end lineage with DVC and Amazon SageMaker AI MLflow apps](https://aws.amazon.com/blogs/machine-learning/end-to-end-lineage-with-dvc-and-amazon-sagemaker-ai-mlflow-apps/) — *Documentation cloud / Well-Architected* — patron de lineage de bout en bout : tag Git = version de dataset, commit loggé comme paramètre de run, S3 Object Lock + CloudTrail pour l'immuabilité, et anti-patterns (oublier le commit hash, scanner les artefacts).
3. [W3C PROV — Overview / Data Model (PROV-DM)](https://www.w3.org/TR/prov-overview/) — *Standard W3C* — modèle de référence Entity / Activity / Agent et relations (`wasGeneratedBy`, `used`, `wasDerivedFrom`, `wasAttributedTo`) pour modéliser la provenance de façon interopérable et sérialisable (JSON-LD).
4. [What is C2PA? Content Provenance Explained](https://c2paviewer.com/articles/what-is-c2pa) — *Standard industriel (vulgarisation)* — provenance tamper-evident : manifeste signé, hard binding par hash cryptographique liant le manifeste au contenu, et historique des éditions (qui, quand, quels outils, IA ou non).
5. [SLSA v1.0 — Provenance specification](https://slsa.dev/spec/v1.0/provenance) — *Standard supply-chain (in-toto / DSSE)* — attestation vérifiable : `subject` (+ digest), `buildDefinition` (`externalParameters`, `resolvedDependencies`) et `runDetails` (`builder.id`, `metadata.invocationId`, timestamps).
6. [What We Learned from a Year of Building with LLMs](https://applied-llms.org/) — *Praticiens reconnus* — logger inputs ET outputs, examiner des échantillons, et « version and pin your models » car les fournisseurs changent les modèles silencieusement.
7. [LLM Training Data Lineage: Provenance, Tracking & Compliance](https://atlan.com/know/training-data-lineage-for-llms/) — *Pratique outillée + cadrage réglementaire* — lineage = provenance + transformations + ownership, snapshots immuables à chaque étape, attribution (qui a approuvé), et contexte EU AI Act rendant la provenance non optionnelle.
