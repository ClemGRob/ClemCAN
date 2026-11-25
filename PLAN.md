# ClemCAN — Plan de développement

> Jalons vérifiables. Accompagne `SPEC.md` (exigences), `docs/srs/` (exigences par composant) et
> `CLAUDE.md` (contrat de travail).
> Version : 1.1 — 2026-11-25, alignée sur `SPEC.md` v1.3. Les efforts n'ont pas été réestimés ;
> le jalon B13, ajouté en 1.1, reste à estimer.

---

## 0. Comment lire ce plan

### 0.1 Unité de mesure

Les efforts sont exprimés en **jours-homme (j)**, pas en dates calendaires, parce que la cadence
de travail n'est pas fixée. Conversions :

| Cadence | Jours utiles par mois | Durée du v1 complet (≈ 312 j, hors B13) |
|---|---|---|
| Temps plein | ≈ 21 | ≈ 15 mois |
| Soirées et week-ends (≈ 15 h/semaine) | ≈ 8 | ≈ 39 mois |
| Un jour par semaine | ≈ 4 | ≈ 78 mois (irréaliste : réduire le périmètre) |

### 0.2 Effet de l'assistance par IA

Le projet est mené en solo avec assistance d'agents d'IA. Hypothèse retenue, **à réviser après le
palier A avec les chiffres réels** : facteur d'accélération de 1,3 à 2 sur les jalons riches en
code bien spécifié et testable (lecteurs de formats, décodage, éditeur, expressions), et facteur
proche de 1 sur les jalons dont le goulot est le banc d'essai, le matériel ou la mesure de
performance. Les estimations ci-dessous sont **hors assistance** : elles ne seront pas révisées à
la baisse par anticipation, seulement constatées.

Les jalons marqués **⚙ matériel** exigent ta présence physique avec le PCAN-USB : aucune IA ne
peut les valider seule.

### 0.3 Condition de validation d'un jalon

Un jalon n'est validé que si **les quatre** conditions suivantes sont réunies :

1. Le critère de vérification du jalon est exécuté et **réussi**, par une commande reproductible.
2. Chaque exigence `EX-*` listée, et chaque exigence SRS qui en découle, porte au moins un test
   nommé avec son identifiant, de la classe déclarée (auto, ⚙, IHM : `D-23`).
3. L'intégration continue est verte sur Linux x86-64, Linux ARM64 et Windows x86-64.
4. Les décisions ouvertes assignées à ce jalon sont tranchées par une fiche écrite.

Aucune dérogation silencieuse : un jalon partiellement atteint est **non atteint**, et le reste
est déplacé explicitement dans le jalon suivant.

Les exigences « étendues à chaque palier » (`EX-AUT-01`, `EX-AUT-04`, `EX-SES-01`) s'appliquent à
tout jalon qui livre une commande ou un objet de session : la commande existe en ligne de
commande, sans invite ni serveur graphique, et l'objet est enregistré dans la session.

### 0.4 Les trois paliers

| Palier | Nom | Ce que tu peux faire à l'issue | Effort |
|---|---|---|---|
| **A** | **Observer** | Brancher, voir, enregistrer, décoder, tracer, exporter. Outil déjà utile au quotidien. | ≈ 97 j |
| **B** | **Comprendre** | Éditer la base, contrôler la cohérence, déclencher, réassembler J1939 et ISO-TP, comparer, rapporter, piloter un démon distant. | ≈ 140 j + B13 |
| **C** | **Agir** | Rejouer un jeu de test sur un système réel, sous garde-fous, et automatiser des essais. | ≈ 75 j |

---

## Palier A — Observer (≈ 97 j)

### A0 — Socle du dépôt — 4 j

**Objectif** : que la première ligne de code métier arrive dans un cadre déjà contraint.

**Contenu** : arborescence (`core/` en C, `clemcan/` en Python, `tests/`, `bench/`, `docs/adr/`),
chaînes de construction des deux langages, intégration continue sur les trois plateformes,
formatage et analyse statique imposés, squelette de ligne de commande, script de mise en place des
bus virtuels de test, injecteur de charge paramétré (`SRS-OUT-01`), matrice de traçabilité
(`EX-*` et SRS) vide mais générée.

**Vérification**
```
clemcan --version                  # affiche version, plateforme, backends détectés
make ci                           # construction + tests + analyse statique, 3 plateformes vertes
scripts/vcan-up.sh 8              # 8 bus virtuels prêts (profil nominal)
bench/inject --profile nominal --duration 10   # injecteur fonctionnel, séquences numérotées
```

**Exigences** : préparatoires. **Dépend de** : rien.

---

### A1 — Noyau de capture et journal natif — 16 j

**Objectif** : le cœur du logiciel, et le seul composant soumis au débit ligne.

**Contenu** : noyau C, réception SocketCAN, horodatage noyau ou matériel, tampon circulaire
préalloué sans allocation dans la boucle, écriture du journal en ajout seul, index reconstructible,
comptage des pertes (débordement noyau et débordement interne), métadonnées de trace,
surveillance de l'espace disque.

**Vérification**
```
# 1. Enveloppe nominale, sans perte
bench/soak --profile nominal --duration 3600
  → trames enregistrées == trames injectées ; pertes déclarées == 0 ;
    mémoire stable au sens d'EX-PRF-01

# 2. Survie à la coupure brutale, 100 fois
tests/robustness/kill-loop.sh 100
  → journal toujours lisible ; perte bornée à la fenêtre d'écriture annoncée

# 3. Reconstruction d'index
rm trace.tcx && clemcan index rebuild trace.tcj && clemcan verify trace.tcj
  → contenu relu identique bit à bit

# 4. Honnêteté sous surcharge
bench/soak --profile nominal --rate-factor 3 --kernel-buffer tiny
  → pertes déclarées == injectées − enregistrées (à zéro trame près)
```

**Exigences** : `EX-ARC-03`, `EX-CAP-01` à `EX-CAP-05`, `EX-JNL-01` à `EX-JNL-03`, `EX-PRF-01`,
`EX-PRF-03`, `EX-ROB-01`, `EX-ROB-04`, `EX-BUS-01`, `EX-BUS-03`.
**Tranche** : O-02 (disposition du journal). **Dépend de** : A0.

---

### A2 — Démon, protocole et liaison Python — 9 j

**Objectif** : rendre le noyau indépendant de tout client, condition de l'autonomie embarquée.

**Contenu** : `clemcand`, protocole de contrôle versionné, abonnements d'affichage (délestables)
et d'évaluation (bornés, jamais délestés) (`D-16`, `D-24`, `D-39`), anneau en mémoire partagée,
liaison Python, droits d'accès locaux, aucun port en écoute par défaut, mode bibliothèque liée.

**Vérification**
```
# 1. L'interface est jetable
clemcand --record trace.tcj & ; python -m tests.client_crash_loop 10min
  → trace continue, sans trou, pertes == 0

# 2. Pas de socket ouvert par défaut, en TCP comme en UDP (D-35)
clemcand & ; ss -lntup | grep clemcand   → aucune ligne

# 3. Contrat de protocole
pytest tests/protocol/   → sérialisation, versions, rejet d'une version incompatible

# 4. Accès local
connexion depuis un compte hors du groupe clemcan   → refusée
```

**Exigences** : `EX-ARC-01`, `EX-ARC-02`, `EX-ARC-04`, `EX-ARC-05`, `EX-ARC-06`, `EX-ROB-03`,
`EX-SEC-01`, `EX-SEC-04`. **Tranche** : O-01. **Dépend de** : A1.

---

### A3 — Backend PCAN et portage Windows — 8 j ⚙ matériel

**Objectif** : que le matériel réellement disponible soit un citoyen de première classe sur les
deux systèmes.

**Contenu** : matériel PEAK par SocketCAN `peak_usb` sous Linux et par PCAN-Basic sous Windows
(`D-15`), bus virtuel ClemCAN sous Windows, horodatage matériel, CAN FD, rapports d'erreur,
déclaration du niveau Tier 1 ; écoute seule par défaut, confirmation pour un matériel sans
*listen-only*, mode acquittement explicite ; pont Tier 2 lancé et surveillé par le démon, en
réception seule (`D-30`, `D-36`) ; niveau inscrit dans les métadonnées de trace.

**Vérification**
```
# 1. Deux canaux PEAK sur un même bus, horodatages cohérents  (PCAN-USB requis)
tests/hardware/timestamp-compare.py --a socketcan:can0 --b socketcan:can1
  → écart dans l'incertitude déclarée ; source déclarée = « matériel »

# 2. Écoute seule  (PCAN-USB requis)
tests/hardware/listen-only.py   → aucun acquittement reçu par un second nœud

# 3. Windows
make ci-windows   → capture, enregistrement, relecture identiques à Linux

# 4. Tier 2 dégradé mais fonctionnel
pytest tests/backends/tier2/   → ouverture par le protocole, capture, niveau annoncé « Tier 2 »,
                                 kill -9 du pont → événement de fin de canal, autres canaux intacts
```

**Exigences** : `EX-BUS-02` à `EX-BUS-05`, `EX-CAP-01`, `EX-SAF-01`, `EX-POR-01`.
**Dépend de** : A1, A2.

---

### A4 — Formats d'échange et matrice de conformité — 12 j

**Objectif** : pouvoir ouvrir n'importe quelle trace reçue, et rendre ce qu'on produit.

**Contenu** : modèle de trame intermédiaire unique ; lecture BLF, ASC, candump, TRC (1.0 à 2.1),
MF4 ; écriture journal natif, ASC, candump, CSV de signaux décodés ; matrice de conformité générée
depuis les tests ; corpus de fichiers de référence versionné.

**Vérification**
```
pytest tests/formats/roundtrip/
  → tout aller-retour perd exactement ce que la matrice annonce, ni plus ni moins

clemcan formats matrix --check
  → échec de construction si la matrice publiée diverge du comportement réel

clemcan convert big.blf out.tcj && clemcan verify out.tcj
  → conversion vérifiée ; sur la trace de 200 M de trames du banc, navigation < 1 s par saut
    au 95ᵉ centile, mémoire ≤ 500 Mo (EX-JNL-05)
```

**Exigences** : `EX-FMT-01` à `EX-FMT-06`, `EX-JNL-05`. **Dépend de** : A1.

---

### A5 — Bases de données et décodage — 13 j

**Objectif** : donner du sens aux trames, sans jamais réécrire cantools.

**Contenu** : lecture DBC complète via cantools ; import SYM, KCD, ARXML en lecture seule avec
rapport de conversion ; modèle interne ; décodage Motorola et Intel, signés, flottants ;
multiplexage simple et étendu ; CAN FD ; J1939 niveau trame ; trois formats de restitution (brut,
physique, libellé) ; décodage paresseux ; plusieurs bases par canal avec priorité déterministe.

**Vérification**
```
pytest tests/decode/vectors/
  → chaque vecteur (base + trame + valeur attendue) passe, y compris :
    multiplexeur imbriqué, valeur de multiplexeur non définie (→ « non applicable », pas 0),
    signal signé à cheval sur 3 octets en Motorola, flottant double, DLC 9 à 15 en classique
    et en FD

pytest tests/decode/oracle/   → concordance avec cantools sur 10⁵ trames aléatoires par base
bench/decode --signals 1,10,100   → coût processeur proportionnel au nombre de signaux affichés
```

**Exigences** : `EX-DB-01`, `EX-DB-02`, `EX-DB-09`, `EX-DEC-01` à `EX-DEC-06`, `EX-DEC-11`,
`EX-DEC-12`. **Dépend de** : A4.

---

### A6 — Vues trace, groupée et valeurs — 11 j

**Objectif** : les trois vues devant lesquelles on passe l'essentiel du temps.

**Contenu** : interface Qt ; trace déroulante à défilement continu ; vue groupée par identifiant
avec période mesurée, gigue, compteur et mise en évidence des octets qui changent ; fenêtre de
valeurs avec format au choix, âge de la donnée et bornes ; filtres d'affichage simples ; fenêtres
ancrables ; délestage du flux d'affichage signalé (`D-39`).

**Vérification**
```
tests/ui/latency.py --profile nominal
  → remise au modèle < 100 ms au 99ᵉ centile ; boucle d'événements jamais bloquée > 200 ms
  → consommateurs ralentis : pertes d'enregistrement == 0, délestage compté (EX-CAP-05)

pytest tests/views/   → périodes, gigues et compteurs corrects sur trace de référence
```

**Exigences** : `EX-VUE-01` à `EX-VUE-03`, `EX-VUE-06`, `EX-VUE-07`, `EX-DEC-03`, `EX-CAP-05`.
**Dépend de** : A5.

---

### A7 — Graphe — 9 j

**Objectif** : tracer sans mentir.

**Contenu** : escalier par défaut, ligne et points par signal, points d'échantillonnage en
surimpression, axes multiples, curseurs et écarts Δt/Δy, synchronisation avec la trace,
décimation préservant les extrema (couche 2, `D-39`), interruption de tracé pour les multiplexés
non applicables.

**Vérification**
```
tests/graph/spike.py
  → impulsion isolée de 1 ms dans 1 h de trace : visible au zoom le plus large (EX-GRA-06)

tests/graph/zoom-latency.py --duration 1h
  → boucle d'événements jamais bloquée > 200 ms pendant le zoom (critère d'O-03)
tests/graph/mux-gap.py   → signal multiplexé non applicable : trou, pas de valeur maintenue
```

**Exigences** : `EX-GRA-01` à `EX-GRA-07`. **Tranche** : O-03. **Dépend de** : A6.

---

### A8 — Statistiques et santé du bus — 5 j

**Contenu** : charge du bus, trames par seconde, histogramme des temps de cycle, compteurs de
rapports d'erreur, bus-off, identifiants apparus ou disparus, trames perdues. Durée de trame
exacte en classique, FD ISO et FD non ISO (`D-37`).

**Vérification**
```
tests/stats/frame-duration.py   → écart nul en bits sur les vecteurs de référence
                                   (classique, FD ISO, FD non ISO)
tests/stats/busload.py --rate-known   (⚙ PCAN)
  → charge calculée à moins de 2 % de la valeur théorique (bourrage de bits inclus)
tests/stats/errors.py   → rapports d'erreur et bus-off injectés : détectés, datés, tracés
```

**Exigences** : `EX-VUE-04`, `EX-BUS-06` (détection ; la politique de reprise vient en C1).
**Dépend de** : A6.

---

### A9 — Session, langues, livraison — 10 j

**Objectif** : que le palier A soit un logiciel qu'on installe et qu'on retrouve, pas une maquette.

**Contenu** : format de session lisible et diffable ; ouverture tolérante aux fichiers manquants ;
profils ; mécanisme de traduction avec catalogue ouvert et repli sur l'anglais ; formats
localisés ; toutes les commandes du palier A en ligne de commande, sans invite (`D-34`) ;
empaquetage en exécutable autonome pour les trois plateformes, pont Tier 2 et python-can inclus,
et livrable du démon seul (`D-36`) ; banc nocturne sur les machines de référence x86-64 et ARM64,
chiffres publiés ; documentation utilisateur du palier A ; matrice de traçabilité publiée.

**Vérification**
```
clemcan session save s.clemcan && clemcan session load s.clemcan
  → restitution intégrale ; diff Git lisible après modification d'un seul signal
clemcan profile open banc   → profil en mode acquittement sans option : ouvert en écoute seule
tests/ci/headless.sh   → suite fonctionnelle, entrée standard fermée, sans serveur graphique
tests/i18n/completeness.py   → aucune chaîne concaténée ; repli anglais fonctionnel
dist/clemcan-linux-x86_64 --version   → démarre sans environnement Python installé ;
                                        canal Tier 2 ouvert sur le backend virtuel de python-can
dist/clemcan-linux-arm64 --version ; dist/clemcan-windows.exe --version
dist/clemcand-linux-x86_64   → démon seul ; ouverture d'un canal Tier 2 refusée avec message
bench/   → demi-profil ARM64 60 min sans perte non déclarée ; publication nocturne présente
clemcan trace matrix --requirements   → toute exigence du palier A porte un test
```

**Exigences** : `EX-SES-01` à `EX-SES-04`, `EX-I18N-01` à `EX-I18N-05`, `EX-POR-01` à `EX-POR-04`,
`EX-AUT-01`, `EX-AUT-04`, `EX-PRF-02`, `EX-PRF-05`, `EX-SEC-03`, §12. **Tranche** : O-04.
**Dépend de** : A7, A8.

> **Palier A livré.** Tu peux brancher un bus, l'enregistrer sans rien perdre, l'ouvrir dans
> n'importe quel format reçu, décoder, tracer, mesurer la santé du bus et retrouver ta session.

---

## Palier B — Comprendre (≈ 140 j + B13)

### B1 — Écrivain DBC au diff minimal — 16 j

**Objectif** : l'exigence la plus technique du projet, et celle qui rend la base utilisable en équipe.

**Contenu** : conservation du texte source, modification chirurgicale des seules lignes concernées,
préservation de l'ordre, de l'indentation, des commentaires et des sections non modélisées.
Constituer le **corpus de DBC réels avant d'écrire le code**.

**Vérification**
```
pytest tests/dbc/roundtrip-identity/
  → sur tout le corpus : lecture puis écriture sans édition ⇒ fichier identique OCTET POUR OCTET
pytest tests/dbc/minimal-diff/
  → modifier un facteur ⇒ diff limité aux lignes de la définition concernée
pytest tests/dbc/exotic-sections/
  → SIG_GROUP_, BA_DEF_REL_, EV_, commentaires multilignes : intacts après édition d'autre chose
```

**Exigences** : `EX-DB-03`. **Dépend de** : A5. **Risque** : R3.

---

### B2 — Validation bloquante — 8 j

**Contenu** : toutes les règles de `EX-DB-04`, avec message désignant l'objet exact et refus
d'enregistrement.

**Vérification**
```
pytest tests/dbc/lint/   → une base fautive par règle, chacune détectée, aucun faux positif
                           sur le corpus de bases réputées saines
```

**Exigences** : `EX-DB-04`. **Dépend de** : B1.

---

### B3 — Éditeur : matrice de bits, annuler/refaire, lots — 18 j

**Contenu** : matrice de bits interactive (placement à la souris, Motorola et Intel, recouvrements
signalés en direct) ; pile d'annulation illimitée ; opérations de masse ; recherche.

**Vérification**
```
pytest tests/editor/bitmatrix/
  → placement en Motorola et en Intel : bit de départ calculé conformément aux vecteurs
  → tout recouvrement est signalé avant validation
pytest tests/editor/undo/
  → 1 000 opérations aléatoires puis 1 000 annulations ⇒ base identique à l'originale, au bit
pytest tests/editor/batch/   → renommage par motif, facteur appliqué, déplacement entre trames
```

**Exigences** : `EX-DB-05` à `EX-DB-07`, `EX-DB-11`. **Dépend de** : B2.

---

### B4 — Comparaison et fusion de bases — 10 j

**Vérification**
```
pytest tests/dbc/diff/
  → écarts détectés : trame ajoutée, signal déplacé, facteur modifié, table de valeurs changée,
    signal redimensionné ; aucun écart signalé entre un fichier et lui-même
pytest tests/dbc/merge/
  → application sélective ; le résultat respecte EX-DB-03 (diff minimal) et EX-DB-04 (lint)
```

**Exigences** : `EX-DB-08`. **Dépend de** : B3.

---

### B5 — Génération de code C — 6 j

**Vérification**
```
pytest tests/codegen/differential/
  → pour chaque base de test, ≥ 10⁵ vecteurs aléatoires : encodage puis décodage par le C généré
    == décodage interne, au bit près, y compris signés, Motorola, flottants et multiplexés
tests/codegen/compile-matrix.sh   → le code généré compile sans avertissement (gcc, clang, -Wall -Wextra)
```

**Exigences** : `EX-DB-10`, UC-4. **Dépend de** : B2.

---

### B6 — Compteurs, contrôles et cohérence — 12 j

**Contenu** : fichier de règles `.tcr` ; profils AUTOSAR E2E, CRC8 SAE-J1850, CRC15/16/32
paramétrables ; détection assistée validée par l'utilisateur ; violations de compteur ;
contrôles de cohérence (bornes, temps de cycle, longueur, valeur hors table, identifiant inconnu,
émetteur inattendu). Violations inscrites dans la trace en direct ; sur une trace déjà
enregistrée, résultats d'analyse jamais écrits dans la trace (`D-32`).

**Vérification**
```
pytest tests/checks/crc-vectors/     → chaque algorithme conforme à ses vecteurs normalisés
pytest tests/checks/counter/         → figé, sauté, dupliqué, retour à zéro légitime
tests/checks/autodetect.py           → sur trace de référence : compteur et contrôle proposés
                                       correctement, règle écrite dans .tcr, résultat reproductible
```

**Exigences** : `EX-CHK-01` à `EX-CHK-04`. **Tranche** : O-08. **Dépend de** : A5.

---

### B7 — Langage d'expression unifié — 12 j

**Contenu** : une grammaire, un évaluateur, cinq usages (filtres d'affichage, filtres de capture
traduits pour le noyau, déclencheurs, alarmes, et plus tard assertions) ; logique à trois valeurs
(`D-25`) ; opérateurs logiques et de comparaison ; fronts ; durées ; absence de trame ; drapeaux
de contrôle ; alarmes (levée, retombée, sévérité) ; application matérielle des filtres quand
possible, avec indication visible.

**Vérification**
```
pytest tests/expr/grammar/     → corpus d'expressions valides et invalides ; position d'erreur exacte
pytest tests/expr/semantics/   → absent(0x120, 500ms), fronts, durées, crc_invalid, counter_skip
tests/expr/hardware-filter.py  → filtre matériel appliqué ⇒ signalé comme tel, car il modifie
                                 ce qui est enregistré
```

**Exigences** : `EX-EXP-01` à `EX-EXP-06`. **Dépend de** : B6.

---

### B8 — Enregistrement déclenché — 8 j

**Contenu** : tampon pré-déclenchement, fenêtres déclenchées, découpage automatique, politique
de saturation par rotation, évaluation de référence en couche 2.

**Vérification**
```
tests/trigger/window.py --violation-at 42s --pre 10s --post 5s
  → le fichier produit contient exactement [32 s ; 47 s], aux tolérances documentées
  → l'événement de déclenchement est daté et porte l'expression qui l'a produit
tests/trigger/multi.py   → déclenchements successifs, un fichier par déclenchement, temps continu
tests/robustness/rotation.py   → seuls les fichiers de l'enregistrement en cours sont effacés
bench/eval --profile nominal --rules bench/rules-ref   → 60 min, aucune évaluation abandonnée
```

**Exigences** : `EX-TRG-01` à `EX-TRG-04`, `EX-JNL-04`, `EX-ROB-05`, `EX-PRF-06`, UC-2.
**Dépend de** : B7.

---

### B9 — Transport et applicatif J1939 — 14 j

**Contenu** : réassemblage BAM et RTS/CTS jusqu'à 1785 octets, temporisations normalisées, sessions
concurrentes, abandons ; cas dégradés traités comme le nœud récepteur (`D-38`) ;
`transport_incomplete` résultat d'analyse, jamais écrit dans la trace (`D-40`) ; DM1 et DM2 (SPN,
FMI, occurrences, lampes) ; réclamation d'adresse et NAME ; requête de PGN.

**Vérification**
```
pytest tests/j1939/transport/
  → nominal ; trame manquante ; trame dupliquée (ignorée) ; trame retransmise sur CTS
    (acceptée) ; dépassement de chaque temporisation à son point d'attente ; nouvelle annonce
    pendant une session ; deux sessions simultanées sur des couples d'adresses différents ;
    abandon explicite ; message tronqué en fin de trace (pas de blocage, pas de fuite mémoire)
pytest tests/j1939/dm1/   → DTC décodés conformément aux vecteurs, lampes incluses
```

**Exigences** : `EX-DEC-07`, `EX-DEC-08`. **Dépend de** : A5. **Risque** : R4.

---

### B10 — ISO-TP et UDS en lecture — 12 j

**Contenu** : ISO-TP sur trames de 8 octets, cas dégradés traités comme le récepteur (`D-38`),
`transport_incomplete` (`D-40`) ; UDS en lecture.

**Vérification**
```
pytest tests/isotp/
  → trame unique, première trame + consécutives, contrôle de flux, bloc et temps de séparation,
    adressage normal / étendu / mixte, 11 et 29 bits, temporisations dépassées, renumérotation ;
    numéro de séquence répété ou erroné, première trame pendant une réception, trame
    consécutive isolée, débordement
pytest tests/uds/read/   → services, sous-fonctions, codes de refus ; 0x22 et 0x19 décodés
```

**Exigences** : `EX-DEC-09`, `EX-DEC-10`. **Dépend de** : A5. **Risque** : R4.

---

### B11 — Annotations, extraction, comparaison, rapport — 14 j

**Contenu** : repères, extraction d'intervalle, comparaison de traces, rapport HTML ; libellés de
table de valeurs en graduation du graphe ; export du graphe en image (PNG, SVG) et de ses séries
en CSV, l'export d'image étant fourni à la ligne de commande par l'interface
(`SRS-ANA-ARC-06`).

**Vérification**
```
tests/analysis/extract.py
  → sous-trace de 3 s exportée avec sa base et ses règles, rouverte sur machine vierge :
    décodage identique à l'original, au bit
tests/analysis/compare.py   → identifiants exclusifs, temps de cycle modifiés, divergences de
                              trajectoire au-delà du seuil, violations apparues ou disparues
clemcan report --from t1.tcj --range 10s:13s --out r.html
  → rapport reproductible depuis la ligne de commande, portant versions et empreintes des bases
clemcan graph export --png g.png --size 1920x1080   → image valide aux dimensions demandées
```

**Exigences** : `EX-ANA-01` à `EX-ANA-04`, `EX-GRA-08`, `EX-GRA-09`, UC-3. **Dépend de** : B7, A7.

---

### B12 — Analyse sans base de données — 10 j

**Vérification**
```
tests/reverse/toggle.py   → bits basculants et octets changeants correctement classés sur trace
                            de référence dont la vérité terrain est connue
tests/reverse/candidate.py → sélection de bits ⇒ signal candidat injecté dans la base en édition,
                             conforme au lint (EX-DB-04), tracé immédiatement
```

**Exigences** : `EX-VUE-05`. **Dépend de** : B3, A7.

---

### B13 — Accès distant et démarrage autonome — à estimer

**Objectif** : piloter depuis un PC un démon installé sur une cible, et laisser la cible
enregistrer seule après un redémarrage.

**Contenu** : accès distant au démon en TLS 1.3 avec authentification mutuelle par certificats ;
démarrage du démon avec le système et reprise d'un enregistrement dans un nouveau fichier.

**Vérification**
```
pytest tests/remote/   → connexion sans certificat, avec certificat invalide, en TLS < 1.3 :
                         refusées ; connexion valide acceptée
tests/robustness/reboot.sh   → redémarrage brutal d'une machine virtuelle : enregistrement repris
                               dans un nouveau fichier ; fin abrupte du précédent signalée
```

**Exigences** : `EX-SEC-02` (partie B), `EX-ROB-02`. **Tranche** : choix de la bibliothèque TLS
(fiche, `SRS-NOY` point NOY-4). **Dépend de** : A2.

> **Palier B livré.** Tu édites tes bases sans détruire les diffs, tu contrôles compteurs et CRC,
> tu déclenches sur événement, tu lis le J1939 multi-trames et les dialogues de diagnostic, tu
> compares deux campagnes et tu sors un rapport. Toujours aucune écriture sur le bus.

---

## Palier C — Agir (≈ 75 j)

### C1 — Ordonnanceur d'émission et garde-fous — 12 j ⚙ matériel

**Objectif** : ouvrir l'écriture sur le bus, et la rendre difficile à déclencher par accident.

**Contenu** : ordonnanceur TX dans le noyau C derrière une interface `TxSource` (le moteur réactif
du v2 en sera un second) ; armement explicite nommant le canal, lié à la connexion cliente, non
mémorisé, avec battement (homme mort, `D-33`) ; armement refusé sur un canal Tier 2 (`D-30`),
aux clients distants par défaut et aux comptes hors du groupe d'armement ; liste blanche
d'identifiants vide par défaut ; avertissement de collision d'identifiant ; indicateur
d'émission ; arrêt d'urgence, y compris en ligne de commande ; journalisation non désactivable ;
politique de reprise après bus-off. L'écoute seule est en place depuis A3.

**Vérification**
```
pytest tests/safety/
  → émission sans armement : refusée et tracée
  → identifiant hors liste blanche : refusé, message explicite ; sans liste : rien n'est émis
  → armement non conservé au redémarrage ; armement d'un canal Tier 2 refusé
  → identifiant déjà émis par un autre nœud : averti avant émission
  → kill -9 du client armé : dernière trame < 50 ms après ; client figé : < 1 s + 50 ms
tests/safety/estop.py   → arrêt d'urgence : plus aucune trame émise sous 50 ms (mesuré, bus virtuel)
```

**Exigences** : `EX-SAF-02` à `EX-SAF-05`, `EX-SAF-07`, `EX-BUS-07`, `EX-SEC-02` (partie C),
`EX-SEC-04` (partie C). **Tranche** : O-07. **Dépend de** : A2.

---

### C2 — Document « jeu de test » et son éditeur — 12 j

**Contenu** : format `.tct` lisible et diffable ; entrées en octets bruts ou en valeurs de signaux ;
temporisations absolues, relatives ou périodiques ; création par sélection d'un intervalle de trace ;
édition tabulaire (ajout, suppression, réordonnancement, duplication, saisie par libellé).

**Vérification**
```
pytest tests/testset/
  → trace → jeu de test → trace : équivalence sémantique vérifiée
  → modification d'une valeur de signal : diff Git limité à la ligne concernée
  → jeu de test invalide (signal inconnu, valeur hors bornes) : refusé avec message précis
```

**Exigences** : `EX-REJ-01`, `EX-REJ-02`. **Tranche** : O-05. **Dépend de** : B3.

---

### C3 — Rejeu, fidélité et trace corrélée — 14 j ⚙ matériel

**Contenu** : exécution d'un jeu de test ; ordonnancement sur horloge absolue ; facteur d'échelle ;
boucle ; remappage de canal ; sélection des trames par expression ; enregistrement corrélé du trafic
émis et reçu avec marquage du sens ; rapport de fidélité ; galop d'essai obligatoire sur bus virtuel,
jugé conjointement par le noyau et la couche 2 (`D-31`).

**Vérification**
```
tests/replay/jitter.py --rate 5000 --duration 3600
  → gigue au 99ᵉ centile < 1 ms ; dérive cumulée < 1 ms sur 1 h
tests/replay/correlated.py
  → la trace de sortie contient l'émis (instants réels) et le reçu, sens marqué, base de temps unique
tests/replay/dry-run.py   → premier rejeu réel refusé si le galop d'essai virtuel n'a pas été passé ;
                            galop déclaré réussi alors qu'une émission a été refusée : non inscrit
```

**Exigences** : `EX-REJ-03` à `EX-REJ-06`, `EX-REJ-08`, `EX-SAF-06`, `EX-PRF-04`.
**Dépend de** : C1, C2.

---

### C4 — Recalcul des compteurs et contrôles à l'émission — 5 j

**Objectif** : sans ce jalon, éditer un jeu de test ne sert à rien sur un système protégé.

**Vérification**
```
tests/replay/recompute.py
  → jeu de test dérivé d'une trace protégée, un signal modifié, rejoué :
    100 % des trames émises passent la validation de compteur et de contrôle
  → un signal non modifié conserve exactement ses octets d'origine
```

**Exigences** : `EX-CHK-05`, `EX-REJ-07`. **Dépend de** : B6, C3.

---

### C5 — Automatisation et assertions — 10 j

**Contenu** : couverture complète en ligne de commande des commandes du palier C ; assertions
dans le langage `EX-EXP` avec contraintes de temps ; rapport lisible par machine et par humain ;
codes retour. L'exécution sans serveur graphique ni invite est exigée depuis A9 (`D-34`) ; ce
jalon la vérifie sur les commandes du palier C.

**Vérification**
```
clemcan run scenario.tct --assert assertions.tce --report r.json --report-html r.html ; echo $?
  → 0 si toutes les assertions passent, non nul sinon ; rapport conforme au schéma publié
tests/ci/headless.sh   → suite fonctionnelle complète sans serveur graphique
tests/ci/assert-timing.py → « ce signal atteint telle valeur en moins de 200 ms après tel événement »
```

**Exigences** : `EX-AUT-01` à `EX-AUT-04`, UC-1. **Dépend de** : C3, B7.

---

### C6 — Enveloppe de performance validée et publiée — 12 j ⚙ matériel

**Objectif** : transformer l'exigence centrale en chiffre publié, sur chaque plateforme.

**Contenu** : optimisations du noyau si nécessaire (lecture par lots, horodatage noyau, écritures
groupées, priorité temps réel, épinglage de cœur, tampons noyau dimensionnés) ; banc nocturne ;
publication des chiffres par plateforme, version et date ; chiffre ARM réellement atteint constaté
et documenté, y compris la limite matérielle rencontrée.

**Vérification**
```
bench/envelope --platform all --publish
  → x86-64 : profil nominal (4 canaux classiques et 4 canaux FD 500k/2M simultanés, saturés)
             1 h, 0 perte ; mémoire stable au sens d'EX-PRF-01
  → ARM64  : débit maximal tenu, publié, pertes comptées ; cause de la limite identifiée
             (contrôleur SPI, latence USB ou processeur)
  → médiane des 5 dernières exécutions dégradée de plus de 10 % par rapport à la référence
    publiée ⇒ régression signalée
```

**Exigences** : `EX-PRF-01`, `EX-PRF-02`, `EX-PRF-03`, `EX-PRF-05`. **Dépend de** : C3. **Risque** : R1.

---

### C7 — Durcissement et livraison 1.0 — 10 j

**Contenu** : campagne longue durée (72 h) en conditions véhicule simulées ; démarrage automatique
du démon ; documentation utilisateur et de référence complètes ; matrice de traçabilité intégrale ;
vérification du nom **faite le 2026-09-19** (INPI : 0 marque, 0 brevet, 0 dessin ou modèle ;
PyPI, npm, crates.io et GitHub libres) ; livrables des trois plateformes.

**Vérification**
```
tests/endurance/72h.sh   → aucune fuite mémoire, aucune perte non déclarée, journal intègre
                           après 30 coupures d'alimentation simulées
clemcan trace matrix --requirements --fail-on-missing   → aucune exigence v1 sans test
docs/   → guide utilisateur, référence de la ligne de commande, matrice de conformité des formats,
          format du journal documenté, protocole du démon documenté
```

**Exigences** : `EX-ROB-01` à `EX-ROB-04`, §12, R9. **Dépend de** : tout.

> **v1.0 livrée.**

---

## Après le v1

Dans cet ordre, chacun étant un jalon de plein droit avec ses critères :

| # | Sujet | Effort estimé | Pourquoi dans cet ordre |
|---|---|---|---|
| P1 | **Écriture BLF** | 6 j | Le moins coûteux et le plus utile en échange : permet de rendre une trace à qui travaille sous CANoe |
| P2 | **Émission réactive (v2)** : « si je reçois ceci, j'émets cela », simulation de nœud | 30 j | Second `TxSource` derrière l'ordonnanceur du jalon C1 : l'architecture l'attend déjà |
| P3 | **Émission UDS** et diagnostic actif | 20 j | Dépend de P2 et de B10 |
| P4 | Écriture TRC, puis MF4 | 4 j puis 15 j | Demande réelle à constater d'abord |
| P5 | Interface web de supervision | 20 j | Décision ouverte O-06, à instruire après usage embarqué réel |
| P6 | Panneaux graphiques interactifs | 25 j | Confort, sans nouvelle capacité d'analyse |
| P7 | Émission sur un canal Tier 2, par le pont (`D-30`) | à estimer | Le besoin d'émission v1 est couvert par le matériel PEAK ; une fiche devra dire comment le délai d'arrêt d'urgence d'`EX-SAF-05` y est tenu |

---

## Récapitulatif

| Palier | Jalons | Effort | Matériel requis |
|---|---|---|---|
| A — Observer | A0 … A9 | 97 j | A3 |
| B — Comprendre | B1 … B13 | 140 j + B13 | aucun |
| C — Agir | C1 … C7 | 75 j | C1, C3, C6 |
| **v1 complet** | **30 jalons** | **312 j + B13** | |

**Chemin critique** : A0 → A1 → A2 → A4 → A5 → B1 → B3 → C2 → C3 → C6 → C7.
Les jalons B5, B6, B9, B10 et B12 sont parallélisables : ils ne dépendent que de A5, ce qui les
rend idéaux à confier en parallèle (y compris à un agent d'IA), car chacun est spécifié par ses
vecteurs de test avant d'être écrit.

**Trois jalons méritent une vigilance particulière** : B1 (identité octet pour octet, la contrainte
la plus dure du projet), B9/B10 (machines à états dont les cas dégradés sont la vraie difficulté),
C6 (le chiffre qui valide ou invalide l'exigence centrale).
