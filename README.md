# Suite EPM : connecteurs Pigment ⇄ TM1 et aide à la décision quantique

Trois outils pour faire circuler et exploiter les données de planification financière entre **Pigment** et **IBM Planning Analytics (TM1)**, jusqu'au **calcul quantique**.

Ils partagent quatre exigences :

| Exigence | Ce que cela veut dire ici |
|---|---|
| **Vérifiable** | Chaque écriture est relue puis comparée cellule par cellule à la source. Les empreintes SHA‑256 et les preuves de chaque exécution sont conservées. |
| **Infaillible** | Rien n'est écrit tant que les données ne sont pas validées. L'écriture est atomique, et l'état antérieur est restauré automatiquement en cas d'écart. |
| **Sécurisé** | Aucun secret dans les fichiers. TLS est obligatoire, les secrets sont masqués dans les logs, les droits sur les fichiers sont restreints. |
| **Gouvernable** | Les flux sont déclarés et versionnés. L'écriture exige une approbation nominative, et les quatre yeux en production. Un journal d'audit chaîné et signé enregistre tout. |

---

## Sommaire

1. [Vue d'ensemble](#vue-densemble)
2. [Connecteur Pigment ⇄ TM1 — Python](#1-connecteur-pigment--tm1--python)
3. [Connecteur Pigment ⇄ TM1 — Java](#2-connecteur-pigment--tm1--java)
4. [Aide à la décision TM1 ⇄ calcul quantique — Python](#3-aide-à-la-décision-tm1--calcul-quantique--python)
5. [Socle commun : sécurité, audit, gouvernance](#socle-commun--sécurité-audit-gouvernance)
6. [Codes de sortie](#codes-de-sortie)
7. [État de validation et points à confirmer](#état-de-validation-et-points-à-confirmer)

---

## Vue d'ensemble

| Dossier | Rôle | Langage | Tests |
|---|---|---|---|
| [`pigment_tm1/`](pigment_tm1/) | Échange de données bidirectionnel Pigment ⇄ TM1 | Python 3.10+ | 26 |
| [`pigment-tm1-java/`](pigment-tm1-java/) | Le même connecteur, pour les environnements Java | Java 17, Maven | 36 |
| [`tm1_quantum/`](tm1_quantum/) | Aide à la décision : toutes les solutions, probabilités, calcul quantique, réécriture dans TM1 | Python 3.10+ | 39 |

```
suite-epm/
├── pigment_tm1/          connecteur Python (TM1py)
├── pigment-tm1-java/     connecteur Java (API REST TM1 directe)
└── tm1_quantum/          aide à la décision TM1 ⇄ quantique
```

Chaque dossier est autonome et a son propre README détaillé.

---

## 1. Connecteur Pigment ⇄ TM1 — Python

**Dossier** : [`pigment_tm1/`](pigment_tm1/)

Il échange des données dans les deux sens :
- **Pigment → TM1** : par exemple, le budget saisi dans Pigment est chargé dans un cube TM1.
- **TM1 → Pigment** : par exemple, le réel clôturé dans TM1 est importé dans Pigment.

### Comment ça marche

1. **Extraction** de la source : vue Pigment exportée en CSV, ou lecture MDX d'un cube TM1.
2. **Normalisation** via un mapping unique (colonne Pigment ↔ dimension TM1), utilisé dans les deux sens. Des tables de correspondance traduisent les membres (par exemple `Jan 27` ↔ `2027-01`).
3. **Validation bloquante**. Sont refusés : éléments inconnus ou non feuilles, doublons, valeurs non numériques, membres protégés, lignes hors périmètre, source vide.
4. **Prévisualisation** : cellules créées, modifiées, effacées, et variation du total.
5. **Gouvernance** : seuil de variation, approbation, ticket, quatre yeux en production.
6. **Écriture atomique** :
   - vers TM1 : dans une sandbox, publiée seulement si la relecture est parfaite ; à défaut, sauvegarde puis restauration automatique ;
   - vers Pigment : import, relecture d'une vue de vérification, et ré-import de l'état antérieur en cas d'écart.
7. **Réconciliation** cellule par cellule. Le flux ne se termine en succès qu'à 100 % de concordance.

### Démarrage rapide

```bash
cd pigment_tm1
pip install -r requirements.txt
cp config/flows.example.yaml config/flows.yaml        # à adapter
python -m connector -c config/flows.yaml run budget_pigment_to_tm1 --dry-run
python -m connector -c config/flows.yaml run budget_pigment_to_tm1 --approved-by j.martin --ticket CHG-1234
python -m unittest discover -s tests -t .
```

**Points notables** :
- configuration en YAML ;
- appel à TM1 via la bibliothèque TM1py ;
- l'effacement du périmètre utilise une fonction TM1py qui requiert des droits administrateur TM1.

---

## 2. Connecteur Pigment ⇄ TM1 — Java

**Dossier** : [`pigment-tm1-java/`](pigment-tm1-java/)

C'est le même connecteur, avec les mêmes garanties, pour les DSI standardisées sur Java. Les deux versions produisent des **empreintes de données identiques** : elles peuvent tourner en parallèle et être comparées.

**Différences avec la version Python** :
- **Une seule dépendance d'exécution, Jackson.** HTTP, TLS, CSV et HMAC passent par le JDK.
- **TM1 est appelé directement par son API REST** (OData v4), sans bibliothèque intermédiaire.
- **L'effacement ne demande aucun droit administrateur** : le connecteur relit les cellules puis écrit des zéros.
- **Quatre modes d'authentification TM1** : natif, CAM/LDAP, clé API (Planning Analytics SaaS), jeton.
- Possibilité de déclarer un truststore PKCS12 pour une autorité de certification interne. La vérification TLS n'est jamais désactivable.
- Configuration en JSON, avec les mêmes clés qu'en Python. Tout champ inconnu est refusé.
- Les tests vérifient aussi **les requêtes HTTP réellement émises**, contre de faux serveurs TM1 et Pigment.

### Démarrage rapide

```bash
cd pigment-tm1-java
mvn test                                   # 36 tests
mvn package                                # target/pigment-tm1-connector.jar
java -jar target/pigment-tm1-connector.jar -c config/flows.json run budget_pigment_to_tm1 --dry-run
```

---

## 3. Aide à la décision TM1 ⇄ calcul quantique — Python

**Dossier** : [`tm1_quantum/`](tm1_quantum/)

L'outil lit un problème de décision dans TM1, par exemple : *quels projets financer avec une enveloppe de Capex et d'ETP limitée ?* Il le résout ensuite de toutes les façons pertinentes, et explique le résultat à un comité de décision.

### Ce qu'il produit

- **Toutes les solutions possibles** :
  - jusqu'à 20 options, chaque combinaison est examinée (2²⁰ ≈ 1 million) ;
  - au-delà, les K meilleures solutions sont **prouvées** par un solveur exact (MILP).
- **Un classement** par valeur attendue.
- **Des probabilités pour chaque décision**, par simulation de Monte-Carlo sur les incertitudes :
  - la probabilité d'être le meilleur choix, avec son intervalle de confiance à 95 % ;
  - la fourchette de valeur entre les 5 % de scénarios les plus défavorables et les 5 % les plus favorables (P5–P95) ;
  - la probabilité de faire mieux que la n°1.
- **Une analyse par option** : la probabilité que chaque projet fasse partie de la meilleure décision.
- **Une sensibilité** : quelle ressource limite, et ce que rapporterait une rallonge de 10 %.
- **Le calcul quantique** : le problème est traduit en QUBO puis en Hamiltonien d'Ising, et résolu par l'algorithme QAOA. Il tourne sur un simulateur local exact ou sur IBM Quantum. **Chaque mesure est vérifiée classiquement**.
- **Un rapport HTML en français**, lisible sur ordinateur et mobile, en mode clair ou sombre. Il contient :
  - une recommandation, avec son niveau de confiance ;
  - les alternatives plus sûres ou plus souvent gagnantes ;
  - les décisions quasi équivalentes ;
  - les décisions à écarter ;
  - les choix fragiles à challenger.

### Le calcul quantique, en clair

- **Formulation compacte par défaut** (pénalisation déséquilibrée) : un qubit par option, sans qubit supplémentaire. Elle est **calibrée et vérifiée** pour que son minimum soit exactement la meilleure décision.
- Si aucun réglage exact n'existe, l'outil **bascule** vers une formulation garantie (avec qubits d'écart), ou vers une formulation approchée, signalée dans le rapport.
- Le **circuit est exporté en OpenQASM 2.0**. C'est la seule donnée qui quitterait l'entreprise : des angles et des numéros de qubits, sans libellé ni montant. Il est **simulé porte par porte** pour prouver qu'il réalise exactement le calcul annoncé.
- Honnêteté du résultat : aujourd'hui, **le calcul quantique ne bat pas le calcul exact**. Il est mesuré contre lui à chaque analyse (fréquence de l'optimum, gain par rapport au hasard).

### Vérifié, pas seulement vérifiable

- **10 contrôles automatiques** à chaque analyse. Un seul échec bloquant interdit toute publication.
- Une **contre-vérification indépendante** dans les tests : une seconde implémentation en Python standard, sans aucun code de l'outil. Elle retrouve le même nombre de solutions, le même classement et les mêmes probabilités.

### Gouvernance

1. `analyze` lit TM1 sans rien écrire, puis produit le rapport et un résultat **scellé** (SHA‑256 inscrit au journal d'audit).
2. Une personne habilitée relit le rapport.
3. `writeback` réécrit la décision dans TM1. **Il refuse** si :
   - le journal d'audit a été altéré ;
   - le résultat a été modifié depuis l'analyse ;
   - les données TM1 ont changé depuis l'analyse ;
   - l'approbation manque (et, en production, le ticket ou les quatre yeux) ;
   - la cible est un membre protégé (par exemple `Actual`).

### Démarrage rapide (sans TM1, sur données de démonstration)

```bash
cd tm1_quantum
pip install -r requirements.txt
python -m tm1q -c config/decision.example.yaml analyze portefeuille_2027 --input examples/projets_demo.csv
python -m unittest discover -s tests -t .          # 39 tests
```

Un exemple de rapport est fourni : [`tm1_quantum/examples/rapport_demo.html`](tm1_quantum/examples/rapport_demo.html).

### Performances observées (simulateur local, 2 cœurs)

| Options | Combinaisons examinées | Solutions admissibles | Qubits | Durée |
|---|---|---|---|---|
| 10 | 1 024 | 143 | 9 | 4 s |
| 16 | 65 536 | 5 079 | 15 | 14 s |
| 20 | 1 048 576 | 61 226 | 19 | 2 min 30 |

---

## Socle commun : sécurité, audit, gouvernance

**Secrets**
- Ils sont lus dans des variables d'environnement (`VAR`) ou des fichiers montés (`VAR_FILE`) : Vault, Key Vault, secrets Kubernetes.
- Un mot de passe écrit en clair dans la configuration est **rejeté**.

**Transport**
- HTTPS obligatoire, vérification TLS non désactivable, aucune redirection suivie silencieusement.

**Journal d'audit**
- Il est append-only et **chaîné** : chaque événement contient l'empreinte du précédent.
- Il est **signé HMAC-SHA256** (obligatoire en production).
- Toute modification, suppression ou insertion est détectée par `verify-audit`.

**Preuves par exécution**
- Source brute, données normalisées, état antérieur de la cible, écarts, rapport.
- Fichiers créés en droits restreints (`rw-r-----`).
- Elles contiennent des données financières : appliquez votre politique de rétention et de chiffrement.

**Verrou**
- Deux exécutions simultanées du même flux sont impossibles.

**Mode `--dry-run` / `analyze`**
- Ils extraient, valident et prévisualisent sans rien écrire.

**Production**
- Une référence de ticket est exigée.
- L'approbateur doit être différent de l'opérateur.
- L'audit doit être signé.

---

## Codes de sortie

| Code | Signification | Pigment ⇄ TM1 | Quantique |
|---|---|---|---|
| 0 | Succès | ✔ | ✔ |
| 2 | Configuration invalide | ✔ | ✔ |
| 3 | Secret manquant | ✔ | ✔ |
| 4 | Données invalides, **rien écrit** | ✔ | ✔ |
| 5 | Règle de gouvernance, **rien écrit** | ✔ | ✔ |
| 6 | Échec de chargement, état restauré | ✔ | ✔ |
| 7 | Écart de réconciliation, état **restauré et vérifié** | ✔ | ✔ |
| 8 | Exécution déjà en cours | ✔ | ✔ |
| 9 | **CRITIQUE** : restauration impossible, intervention manuelle | ✔ | ✔ |
| 10 | Journal d'audit altéré | ✔ | ✔ |
| 11 | Contrôle de vérification en échec | | ✔ |
| 12 | Aucune solution admissible | | ✔ |
| 64 | Erreur d'usage de la ligne de commande | Java | |

Ils sont conçus pour les ordonnanceurs : Control‑M, Airflow, cron, chores TM1.

---

## État de validation et points à confirmer

**Ce qui est testé**
- 101 tests au total.
- Pigment et TM1 sont simulés en mémoire, et des serveurs HTTP locaux imitent les vraies API.
- Le tout couvre les cas nominaux et les échecs provoqués :
  - corruption silencieuse d'une écriture ;
  - import partiel ;
  - falsification de l'audit ;
  - données modifiées entre l'analyse et l'écriture ;
  - exécutions concurrentes.

**Ce qui n'a pas pu être testé ici** et doit être validé dans votre environnement avant la production :

1. **Vos instances Pigment et TM1 réelles.** Points à vérifier :
   - côté Pigment (sur `https://pigment.app/api/swagger`) : les URL, le champ identifiant d'import et les libellés de statut ;
   - côté TM1 : le comportement REST selon la version (11.x ou 12) et le mode d'authentification SaaS.
2. **Le build Maven** : le dépôt Maven Central était inaccessible pendant le développement. Le code Java a été compilé et testé avec `javac` ; lancez `mvn test` en premier.
3. **L'exécution sur IBM Quantum** : Qiskit était indisponible. Le circuit soumis est vérifié, mais l'appel à IBM Quantum reste à valider sur votre compte.
4. **La volumétrie** au-delà de quelques millions de cellules : découpez le périmètre en plusieurs flux si besoin.

**Démarche recommandée**
1. `validate-config`.
2. `--dry-run` sur un environnement de test.
3. Écriture réelle en test, puis échecs provoqués volontairement (membre inconnu, seuil, membre protégé, concurrence, audit falsifié).
4. Passage en production.

---

## Licence

À définir par le propriétaire du dépôt.
