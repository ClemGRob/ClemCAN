# SRS-ANA — Couche d'analyse (`clemcan/`, hors `ui/`)

> Spécification des exigences logicielles du composant **analyse**.
> Version 1.1 — 2026-11-25. Dérivée de `SPEC.md` v1.3, qui prévaut en cas d'écart.
> Conventions : `docs/srs/README.md`.

---

## 1. Objet et périmètre

La couche d'analyse porte toute la logique métier : formats de trace, bases de données,
décodage, transport J1939 et ISO-TP, UDS, contrôles, langage d'expression, déclencheurs, alarmes,
analyse, rejeu, assertions, sessions, et la ligne de commande `clemcan`. Elle est cliente du noyau
et s'exécute sans interface graphique, sur un PC ou sur une cible embarquée.

Modules : `core_api` (seul à connaître l'ABI et le protocole), `trace`, `db`, `decode`, `checks`,
`expr`, `replay`, `cli`, ainsi que les modèles de vues, la session et les rapports.

**Elle ne fait pas** : capture, horodatage, écriture directe du journal, émission directe sur un
bus (tout passe par le noyau) ; rendu graphique (composant IHM).

## 2. Références

`SPEC.md` v1.3 ; `SRS-NOY.md` ; cantools ; SAE J1939-21, -73, -81 ; ISO 15765-2:2016 ;
ISO 14229-1 ; spécification AUTOSAR du protocole E2E.

## 3. Interfaces

| Id | Interface | Sens | Description |
|---|---|---|---|
| **I-PROT** | Protocole client | requise | Fourni par le noyau (`SRS-NOY`), local ou distant (TLS). |
| **I-LIB** | `libclemcan` | requise | Mode lié, lecture et écriture `.tcj`, durée de trame. |
| **I-API** | API Python | fournie | Modèles (vues, graphe, éditeur), registre de commandes, consommés par l'IHM. |
| **I-CLI** | Ligne de commande `clemcan` | fournie | Toute fonction (§2 de `SPEC.md`), codes retour, rapports. |
| **I-FIC** | Fichiers | fournie | `.clemcan`, `.tcr`, `.tct`, `.tce`, profils, rapports, exports, code C généré. |

---

## 4. Exigences

### 4.1 Architecture

- **SRS-ANA-ARC-01** [A] Seul le module `core_api` DOIT connaître l'ABI de `libclemcan` et
  l'encodage d'I-PROT ; les autres modules passent par son API Python.
  *Source* : §4 de `SPEC.md`, `EX-ARC-04`. *Vérification* : analyse automatique : aucun import de
  `ctypes` ni de `cffi` hors de `core_api` ; les constantes et dispositions d'I-PROT n'apparaissent
  pas hors de `core_api`.
- **SRS-ANA-ARC-02** [A] Toute fonction DOIT être une commande du **registre de commandes** :
  nommée, paramétrée, documentée, exécutable par la ligne de commande et par l'IHM avec le même
  code. Aucune donnée, au sens de « Fonction » (§2 de `SPEC.md`), n'est produite ni modifiée hors
  d'une commande ; les interactions de visualisation en sont exclues.
  *Source* : `EX-ARC-06`, `D-27`. *Vérification* : celle d'`EX-ARC-06`.
- **SRS-ANA-ARC-03** [A] La couche d'analyse NE DOIT importer aucun module graphique ; elle
  s'installe et s'exécute sans PySide6.
  *Source* : `EX-ARC-06`, `EX-AUT-04`, `EX-POR-03`. *Vérification* : suite fonctionnelle
  exécutée dans un environnement sans PySide6.
- **SRS-ANA-ARC-04** [A] La couche d'analyse DOIT pouvoir travailler avec un démon local ou avec
  `libclemcan` en mode lié. En mode lié, seules les fonctions d'I-LIB sont disponibles (capture,
  journal, durée de trame) ; une commande qui en exige d'autres le signale. [B] Elle DOIT aussi
  travailler avec un démon distant (`EX-SEC-02`).
  *Source* : `EX-ARC-02`, `D-15`, `EX-SEC-02`. *Vérification* : [A] une même commande
  d'enregistrement exécutée avec un démon local et en mode lié ; traces de même contenu. [B] De
  même avec un démon distant.
- **SRS-ANA-ARC-05** [B] Les machines à états de transport (J1939, ISO-TP) et d'évaluation
  temporelle DOIVENT être des unités isolées, testables sans flux ni noyau, alimentées par des
  vecteurs.
  *Source* : `R4` ; extension à l'évaluation temporelle dérivée — `held` et `absent`
  (`EX-EXP-03`) ont des cas limites aussi fins que les temporisations de transport.
  *Vérification* : les suites de vecteurs s'exécutent sans démon.
- **SRS-ANA-ARC-06** [B] Le registre de commandes DOIT offrir un point d'extension par lequel
  l'IHM, quand elle est installée, fournit la commande d'export d'image du graphe. Sans IHM, cette
  commande est absente et la ligne de commande le signale.
  *Source* : `EX-GRA-09`, `EX-ARC-06`, `D-27` ; dérivée — l'export d'image exige le moteur de
  rendu de l'IHM, que la couche d'analyse n'importe pas (`SRS-ANA-ARC-03`). *Vérification* :
  sans PySide6, commande absente avec message explicite ; avec l'IHM, commande listée et exécutée
  par la ligne de commande.

### 4.2 Traces et formats

- **SRS-ANA-TRC-01** [A] Un **modèle de trame intermédiaire unique** DOIT porter : identifiant,
  étendu, longueur, DLC brut, données, FD, BRS, ESI, requête distante, rapport d'erreur et ses
  détails, canal, sens, horodatage, commentaire, ainsi que les événements.
  *Source* : `EX-FMT-02`. *Vérification* : celle d'`EX-FMT-02`.
- **SRS-ANA-TRC-02** [A] Les lecteurs DOIVENT couvrir : journal natif (par `libclemcan`), Vector
  ASC, candump, Vector BLF, PEAK TRC 1.0 à 2.1, ASAM MDF 4.0 à 4.2, avec les variantes du tableau
  §5.4 de `SPEC.md`. Toute variante non listée (dont TRC 3.x) est refusée avec un message
  explicite ; les objets BLF non CAN sont ignorés, comptés et signalés.
  *Source* : `EX-FMT-01`. *Vérification* : celle d'`EX-FMT-01`.
- **SRS-ANA-TRC-03** [A] Les écrivains DOIVENT couvrir : journal natif (par `libclemcan`), ASC,
  candump, CSV de signaux décodés. BLF, TRC et MF4 ne sont pas écrits.
  *Source* : §5.4, `D-05`. *Vérification* : tests d'aller-retour de `SRS-ANA-TRC-04`.
- **SRS-ANA-TRC-04** [A] La **matrice de conformité** (format × sens × attribut : conservé,
  approximé selon règle documentée, perdu) DOIT être générée depuis les tests d'aller-retour sur
  `corpus/` ; toute perte non déclarée ou tout écart avec la matrice publiée casse la
  construction.
  *Source* : `EX-FMT-03`, `EX-FMT-04`. *Vérification* : celle d'`EX-FMT-03`.
- **SRS-ANA-TRC-05** [A] Une trace importée DOIT être convertie en journal natif portant son
  origine (format, fichier, empreinte), triée de façon stable par horodatage, avec le nombre de
  trames déplacées dans les métadonnées.
  *Source* : `EX-FMT-06`. *Vérification* : celle d'`EX-FMT-06`.
- **SRS-ANA-TRC-06** [A] L'export CSV de signaux DOIT offrir trois modes explicites (instants
  d'arrivée, pas fixe avec maintien, pas fixe interpolé — jamais par défaut), produire une cellule
  vide pour NA, et utiliser un couple séparateur décimal / séparateur de colonne explicite (point
  décimal et virgule de colonne, ou virgule décimale et point-virgule de colonne ; le couple
  virgule-virgule est refusé). Pour les modes à pas fixe, le pas est un paramètre explicite.
  *Source* : `EX-FMT-05`, `EX-I18N-05`, `D-21`. *Vérification* : celle d'`EX-FMT-05`.
- **SRS-ANA-TRC-07** [A] Aucune commande NE DOIT modifier un fichier de trace : extraction,
  conversion, annotation ou tri produisent un nouveau fichier ou vivent dans la session.
  *Source* : §2 « Trace », `EX-ANA-01`. *Vérification* : empreinte de la trace source inchangée
  après chaque commande de la suite fonctionnelle.

### 4.3 Bases de données

- **SRS-ANA-DB-01** [A] La lecture DBC DOIT passer par cantools et couvrir `BO_`, `SG_`, `CM_`,
  `BA_DEF_`, `BA_`, `VAL_`, `VAL_TABLE_`, `SIG_VALTYPE_`, `SG_MUL_VAL_`, `BO_TX_BU_`,
  `SIG_GROUP_`, `EV_`. Les sections refusées par cantools (`SGTYPE_`, `CAT_DEF_`, `FILTER`,
  `BA_DEF_SGTYPE_`…) sont écartées avant lecture par un filtre isolé, conservées pour l'écriture,
  et signalées.
  *Source* : `EX-DB-01`, `D-19`. *Vérification* : celle d'`EX-DB-01`.
- **SRS-ANA-DB-02** [A] SYM, KCD et ARXML DOIVENT être importés en lecture seule, avec un rapport
  des constructions non transposables.
  *Source* : `EX-DB-02`. *Vérification* : celle d'`EX-DB-02`.
- **SRS-ANA-DB-03** [A] Plusieurs bases DOIVENT pouvoir être associées à un canal, la première de
  la liste l'emportant ; tout identifiant défini dans plusieurs bases est signalé avec les bases
  concernées.
  *Source* : `EX-DB-09`. *Vérification* : celle d'`EX-DB-09`.
- **SRS-ANA-DB-04** [B] L'écriture DBC DOIT se faire au **diff minimal** par un analyseur concret
  propre au projet, conservant chaque octet et la position de chaque définition ; seules les
  lignes définies par `EX-DB-03` pour chaque type d'édition sont réécrites.
  *Source* : `EX-DB-03`, `D-06`, `D-19`. *Vérification* : celle d'`EX-DB-03`.
- **SRS-ANA-DB-05** [B] La validation DOIT appliquer les règles d'`EX-DB-04`, **bloquantes** pour
  un défaut introduit ou touché par l'édition, **avertissements** pour un défaut préexistant ; le
  DLC est vérifié contre le mode du canal à l'association, en avertissement, jamais dans
  l'éditeur. Chaque message désigne l'objet exact.
  *Source* : `EX-DB-04`, `D-22`. *Vérification* : celle d'`EX-DB-04`.
- **SRS-ANA-DB-06** [B] Un **modèle de matrice de bits** DOIT offrir placement et redimensionnement
  d'un signal, conversion du bit de départ entre conventions Intel et Motorola, et détection des
  recouvrements à chaque modification.
  *Source* : `EX-DB-05`. *Vérification* : celle d'`EX-DB-05`, hors IHM.
- **SRS-ANA-DB-07** [B] Annuler et refaire DOIVENT être illimités depuis l'ouverture de la base,
  opérations de masse comprises.
  *Source* : `EX-DB-06`. *Vérification* : celle d'`EX-DB-06`.
- **SRS-ANA-DB-08** [B] Les opérations par lot DOIVENT couvrir : renommage par motif, facteur ou
  unité appliqué à une sélection, déplacement de signaux entre trames, duplication d'une trame
  vers un autre émetteur.
  *Source* : `EX-DB-07`. *Vérification* : celle d'`EX-DB-07`.
- **SRS-ANA-DB-09** [B] La comparaison et la fusion sémantiques DOIVENT détecter trames et signaux
  ajoutés, supprimés, déplacés, redimensionnés, et les écarts de facteur, offset, unité et table de
  valeurs, et permettre l'application sélective avec prévisualisation.
  *Source* : `EX-DB-08`. *Vérification* : celle d'`EX-DB-08`.
- **SRS-ANA-DB-10** [B] Le générateur DOIT produire le code C de sérialisation et de
  désérialisation d'une base, conforme au décodage interne selon `EX-DB-10`.
  *Source* : `EX-DB-10`, UC-4. *Vérification* : celle d'`EX-DB-10`.
- **SRS-ANA-DB-11** [B] La recherche DOIT porter sur le nom, l'identifiant, le nœud, l'unité et le
  commentaire.
  *Source* : `EX-DB-11`. *Vérification* : celle d'`EX-DB-11`.

### 4.4 Décodage

- **SRS-ANA-DEC-01** [A] L'extraction de bits par signal DOIT être propre au projet, Intel (bit de
  départ = poids faible) et Motorola (bit de départ = poids fort), mélangeables dans une trame ;
  cantools sert d'oracle de test.
  *Source* : `EX-DEC-01`, `D-18` ; volume de 10⁵ trames dérivé — de l'ordre de quelques secondes
  d'exécution en intégration continue. *Vérification* : celle d'`EX-DEC-01`, plus concordance
  avec cantools sur 10⁵ trames aléatoires par base hors multiplexeur non défini.
- **SRS-ANA-DEC-02** [A] Le décodage DOIT gérer signaux signés (extension de signe sur la largeur
  du signal), non signés, flottants simple et double précision, facteur et offset s'appliquant
  aussi aux flottants.
  *Source* : `EX-DEC-02`. *Vérification* : celle d'`EX-DEC-02`.
- **SRS-ANA-DEC-03** [A] Chaque signal DOIT être restituable en valeur brute (hexadécimale et
  décimale), valeur physique avec unité, et libellé de table `VAL_`, les trois simultanément, la
  représentation étant sélectionnable par signal.
  *Source* : `EX-DEC-03`. *Vérification* : celle d'`EX-DEC-03`.
- **SRS-ANA-DEC-04** [A] Le multiplexage simple et étendu (`SG_MUL_VAL_`, plages, imbrication)
  DOIT être géré. Un signal multiplexé est NA si son multiplexeur, ou un multiplexeur ancêtre, ne
  sélectionne pas sa définition ; un multiplexeur lui-même multiplexé et non sélectionné est NA ;
  les autres signaux de la trame restent décodés.
  *Source* : `EX-DEC-04`. *Vérification* : celle d'`EX-DEC-04`.
- **SRS-ANA-DEC-05** [A] Les trames FD jusqu'à 64 octets et les trames classiques de DLC 9 à 15
  (8 octets) DOIVENT être décodées en distinguant DLC brut et longueur réelle.
  *Source* : `EX-DEC-05`. *Vérification* : celle d'`EX-DEC-05`.
- **SRS-ANA-DEC-06** [A] Le décodage J1939 au niveau trame DOIT suivre `EX-DEC-06` : 29 bits
  seulement, PDU1/PDU2, EDP = 1 exclu, application si et seulement si le canal est déclaré J1939
  ou si une base associée porte `ProtocolType=J1939`, rapprochement par identifiant exact puis par
  PGN, un rapprochement par PGN ambigu étant signalé.
  *Source* : `EX-DEC-06`. *Vérification* : celle d'`EX-DEC-06`.
- **SRS-ANA-DEC-07** [A] La **valeur à l'instant t** (escalier, NA avant la première réception et
  hors sélection du multiplexeur) DOIT avoir une implémentation unique, utilisée par le graphe en
  escalier, l'export avec maintien, la comparaison, les contrôles, les déclencheurs, les alarmes et
  les assertions.
  *Source* : `EX-GRA-02`, `D-21`. *Vérification* : celle d'`EX-GRA-02`.
- **SRS-ANA-DEC-08** [A] Le décodage DOIT être **paresseux** : seuls sont décodés les signaux requis
  par un affichage, un export ou une expression active ; aucun décodage n'a lieu à la réception.
  *Source* : `EX-DEC-11`. *Vérification* : celle d'`EX-DEC-11`.
- **SRS-ANA-DEC-09** [A] Un identifiant absent des bases DOIT rester visible et exploitable en brut
  dans toutes les vues et tous les exports bruts.
  *Source* : `EX-DEC-12`. *Vérification* : celle d'`EX-DEC-12`.
- **SRS-ANA-DEC-10** [B] Le réassemblage J1939-21 (BAM, RTS/CTS) DOIT suivre `EX-DEC-07` :
  1 785 octets au plus, sessions concurrentes indexées par couple d'adresses, temporisations
  normalisées configurables appliquées au point d'attente du nœud qui attend, cas dégradés traités
  comme le nœud récepteur, et pour toute session incomplète un résultat d'analyse
  `transport_incomplete` (horodaté, compté, portant son motif, jamais écrit dans la trace) et la
  libération de sa mémoire.
  *Source* : `EX-DEC-07`, `D-38`, `D-40`. *Vérification* : celle d'`EX-DEC-07`.
- **SRS-ANA-DEC-11** [B] Le décodage applicatif J1939 DOIT couvrir DM1, DM2, réclamation d'adresse
  et NAME, requête de PGN.
  *Source* : `EX-DEC-08`. *Vérification* : celle d'`EX-DEC-08`.
- **SRS-ANA-DEC-12** [B] Le décodage ISO-TP sur trames de 8 octets DOIT suivre `EX-DEC-09` (types
  de trame, adressages, STmin, temporisations, appariement par paire d'identifiants configurée),
  y compris ses cas dégradés (numéro de séquence erroné ou répété, trame unique ou première trame
  pendant une réception, trame consécutive isolée, débordement) et le résultat
  `transport_incomplete` de `SRS-ANA-DEC-10`.
  *Source* : `EX-DEC-09`, `D-38`, `D-40`. *Vérification* : celle d'`EX-DEC-09`.
- **SRS-ANA-DEC-13** [B] Le décodage UDS en lecture DOIT suivre `EX-DEC-10` ; aucune requête UDS
  n'est émise.
  *Source* : `EX-DEC-10`. *Vérification* : celle d'`EX-DEC-10`.

### 4.5 Contrôles de cohérence

- **SRS-ANA-CHK-01** [B] Le fichier `.tcr` (YAML) DOIT décrire, par trame, compteur (modulo, pas,
  tolérance), contrôle, algorithme, étendue couverte et traitement des octets exclus, avec les
  profils E2E et le catalogue CRC d'`EX-CHK-01`.
  *Source* : `EX-CHK-01`, `D-08`. *Vérification* : celle d'`EX-CHK-01`.
- **SRS-ANA-CHK-02** [B] La détection assistée DOIT suivre `EX-CHK-02` et n'écrire une règle dans
  le `.tcr` qu'après validation par l'utilisateur.
  *Source* : `EX-CHK-02`. *Vérification* : celle d'`EX-CHK-02`.
- **SRS-ANA-CHK-03** [B] Les violations de compteur (sauté, dupliqué, figé) et de contrôle DOIVENT
  être détectées selon `EX-CHK-03` ; un saut couvrant une perte déclarée ou une évaluation
  abandonnée est « non évaluable ».
  *Source* : `EX-CHK-03`, `D-24`. *Vérification* : celle d'`EX-CHK-03`.
- **SRS-ANA-CHK-04** [B] Les contrôles de cohérence d'`EX-CHK-04` (bornes, temps de cycle, longueur,
  valeur hors table, identifiant inconnu, émetteur inattendu J1939) DOIVENT être disponibles.
  *Source* : `EX-CHK-04`. *Vérification* : celle d'`EX-CHK-04`.
- **SRS-ANA-CHK-05** [B] Pendant un enregistrement, chaque violation DOIT être injectée comme
  événement dans le journal (`SRS-NOY-JNL-12`). Sur une trace déjà enregistrée, les violations
  sont des résultats d'analyse : visibles dans la chronologie, exploitables comme condition,
  exportables, jamais écrits dans la trace ; leur support suit `O-08`.
  *Source* : `EX-CHK-03`, `D-32`. *Vérification* : celle d'`EX-CHK-03`.
- **SRS-ANA-CHK-06** [C] Au rejeu, les compteurs DOIVENT être régénérés en suite continue et les
  contrôles recalculés selon `EX-CHK-05`. Un jeu contenant une trame couverte par une règle non
  recalculable est refusé avec un message explicite, sauf émission explicitement demandée en
  octets bruts inchangés.
  *Source* : `EX-CHK-05`, `EX-REJ-07`, `D-29`. *Vérification* : celle d'`EX-CHK-05`.

### 4.6 Langage d'expression, alarmes, déclencheurs

- **SRS-ANA-EXP-01** [B] Un **unique** analyseur DOIT servir aux filtres d'affichage, filtres de
  capture, déclencheurs, alarmes et assertions, et un unique évaluateur à tous ces usages sauf les
  filtres de capture, qui sont traduits (`SRS-ANA-EXP-05`), jamais évalués par la couche
  d'analyse ; la grammaire est publiée en EBNF, générée depuis l'analyseur ou testée contre lui.
  *Source* : `EX-EXP-01`, `EX-EXP-03`, `D-11`. *Vérification* : celle d'`EX-EXP-01`.
- **SRS-ANA-EXP-02** [B] Le langage DOIT désigner canal, identifiant, signal par nom (qualifié en
  cas d'ambiguïté, sinon erreur), valeur physique, brute ou libellé, longueur, sens, drapeaux de
  trame, résultats de contrôle.
  *Source* : `EX-EXP-02`. *Vérification* : celle d'`EX-EXP-02`.
- **SRS-ANA-EXP-03** [B] L'évaluation DOIT suivre la sémantique d'`EX-EXP-03` : à chaque trame
  pertinente et à chaque échéance de minuterie (résolution 1 ms), sur la base de temps de la
  trace ; logique
  à trois valeurs de Kleene ; condition satisfaite seulement si vraie ; `rise`, `fall`, `held`,
  `absent`, `isna`.
  *Source* : `EX-EXP-03`, `D-25`. *Vérification* : celle d'`EX-EXP-03`.
- **SRS-ANA-EXP-04** [B] Une expression invalide DOIT être refusée à la saisie avec sa position et
  sa cause.
  *Source* : `EX-EXP-04`. *Vérification* : celle d'`EX-EXP-04`.
- **SRS-ANA-EXP-05** [B] Une expression de filtre de capture DOIT être traduite en listes canal,
  identifiant, masque, étendu, FD, transmises au noyau ; hors de ce sous-ensemble, elle est
  refusée.
  *Source* : `EX-EXP-01`, `EX-EXP-05`. *Vérification* : celle d'`EX-EXP-01` pour le refus ;
  traduction comparée à l'attendu sur un corpus.
- **SRS-ANA-EXP-06** [B] Une même expression DOIT donner les mêmes résultats évaluée en direct sur
  un abonnement d'évaluation et hors ligne sur la trace enregistrée, hors intervalles abandonnés.
  *Source* : `EX-EXP-01`, `EX-EXP-03`, `D-21`, `D-24`. *Vérification* : scénario rejoué sur bus
  virtuel avec enregistrement ; résultats directs et hors ligne identiques.
- **SRS-ANA-EXP-07** [B] Tout résultat (violation, déclenchement, alarme, verdict) dépendant d'un
  intervalle d'évaluation abandonnée DOIT être marqué « non évaluable ». Un résultat en dépend
  quand son évaluation a besoin d'une trame de cet intervalle : trame évaluée elle-même, valeur à
  l'instant t d'un signal reçu dans l'intervalle, ou fenêtre de `held` ou `absent` qui le recouvre.
  *Source* : `D-24`. *Vérification* : abandon provoqué ; résultats dépendants marqués.
- **SRS-ANA-EXP-08** [B] Les **alarmes** DOIVENT suivre `EX-EXP-06` : nom, expression, sévérité ;
  levée au passage à vrai, retombée à la fin de vrai ; événements injectés dans le journal en
  direct (`SRS-NOY-JNL-12`) ; sur une trace déjà enregistrée, résultats d'analyse jamais écrits
  dans la trace, dont le support suit `O-08` ; exposés et listés par la ligne de commande ;
  enregistrées dans la session ; aucune action sur le bus.
  *Source* : `EX-EXP-06`, `D-32`. *Vérification* : celle d'`EX-EXP-06`, hors IHM.
- **SRS-ANA-TRG-01** [B] Le module de déclenchement DOIT évaluer la condition, calculer l'instant
  de déclenchement (horodatage de la trame qui rend la condition vraie, ou échéance de minuterie),
  commander la fenêtre au noyau avec cet instant et les durées pré et post, et injecter
  l'événement de déclenchement portant l'expression (`SRS-NOY-JNL-12`).
  *Source* : `EX-TRG-02`, `EX-TRG-04`, §5.11. *Vérification* : celles d'`EX-TRG-02` et
  d'`EX-TRG-04`.
- **SRS-ANA-TRG-02** [B] La configuration d'un enregistrement déclenché DOIT couvrir durée ou
  taille pré-déclenchement, durée post-déclenchement, durée maximale de fenêtre et déclenchements
  multiples.
  *Source* : `EX-TRG-01`, `EX-TRG-03`. *Vérification* : celle d'`EX-TRG-03`.

### 4.7 Modèles des vues et du graphe

- **SRS-ANA-VUE-01** [A] Le modèle de **trace déroulante** DOIT fournir le flux chronologique brut,
  des colonnes configurables, la pause sans effet sur l'enregistrement, le dépliage des signaux
  d'une trame.
  *Source* : `EX-VUE-01`. *Vérification* : celle d'`EX-VUE-01`, hors IHM.
- **SRS-ANA-VUE-02** [A] Le modèle de **vue groupée** DOIT restituer les statistiques par
  identifiant calculées par le noyau.
  *Source* : `EX-VUE-02`. *Vérification* : celle d'`EX-VUE-02`, hors IHM.
- **SRS-ANA-VUE-03** [A] Le modèle de **fenêtre de valeurs** DOIT fournir, par signal choisi, la
  valeur courante au format choisi, l'âge de la dernière mise à jour, le franchissement de bornes et
  le regroupement.
  *Source* : `EX-VUE-03`. *Vérification* : celle d'`EX-VUE-03`, hors IHM.
- **SRS-ANA-VUE-04** [A] Le modèle de **santé du bus** DOIT restituer les grandeurs d'`EX-VUE-04` ;
  pour une trace importée dont le débit est connu, la charge est calculée par la fonction unique
  de `libclemcan` ; sans débit, elle n'est pas fournie ; la sous-estimation possible en présence
  d'erreurs est signalée.
  *Source* : `EX-VUE-04`. *Vérification* : charge d'une trace importée égale à celle calculée par
  le noyau en direct sur la même séquence.
- **SRS-ANA-VUE-05** [A] Au profil nominal, une trame DOIT être remise au modèle d'affichage en
  moins de 100 ms au 99ᵉ centile après son horodatage de réception.
  *Source* : `EX-VUE-06`. *Vérification* : celle d'`EX-VUE-06`, sonde au niveau du modèle.
- **SRS-ANA-VUE-06** [B] Le modèle d'**analyse sans base** DOIT fournir bits basculants et octets
  changeants par identifiant, compteurs et contrôles probables (règles d'`EX-CHK-02`), corrélation
  entre un intervalle et les changements, et la création d'un signal candidat depuis une sélection
  de bits dans la base en édition.
  *Source* : `EX-VUE-05`. *Vérification* : celle d'`EX-VUE-05`.
- **SRS-ANA-GRA-01** [A] Le modèle de graphe DOIT produire, par signal, la série du mode choisi
  (escalier par défaut, ligne interpolée, points nus) et les instants d'échantillonnage réels.
  *Source* : `EX-GRA-01`, `D-07`. *Vérification* : celle d'`EX-GRA-01`.
- **SRS-ANA-GRA-02** [A] Toute série DOIT présenter une rupture sur chaque intervalle NA.
  *Source* : `EX-GRA-07`. *Vérification* : celle d'`EX-GRA-07`.
- **SRS-ANA-GRA-03** [A] La décimation DOIT préserver les extrema à toute échelle. En direct, un
  délestage du flux d'affichage par le noyau est signalé sur la série ; la série relue depuis la
  trace contient les trames délestées.
  *Source* : `EX-GRA-06`, `D-39`. *Vérification* : celle d'`EX-GRA-06`.
- **SRS-ANA-GRA-04** [A] Le modèle DOIT calculer, pour au moins deux curseurs, les valeurs à leur
  position et les écarts Δt et Δy.
  *Source* : `EX-GRA-04`. *Vérification* : celle d'`EX-GRA-04`.
- **SRS-ANA-GRA-05** [A] Le modèle de sélection DOIT associer un instant à la trame correspondante
  (en escalier, la trame précédente) et une trame à son instant.
  *Source* : `EX-GRA-05`. *Vérification* : celle d'`EX-GRA-05`, hors IHM.
- **SRS-ANA-GRA-06** [A] L'état du graphe (axes, échelles, empilement, couleurs, modes) DOIT être
  sérialisable dans la session.
  *Source* : `EX-GRA-02`, `EX-GRA-03`. *Vérification* : celle d'`EX-GRA-03`.
- **SRS-ANA-GRA-07** [B] Pour un signal à table `VAL_`, le modèle DOIT fournir les libellés comme
  graduations.
  *Source* : `EX-GRA-08`. *Vérification* : celle d'`EX-GRA-08`.
- **SRS-ANA-GRA-08** [B] Les séries affichées DOIVENT être exportables en CSV selon
  `SRS-ANA-TRC-06`.
  *Source* : `EX-GRA-09`. *Vérification* : CSV comparé à l'attendu.

### 4.8 Repères, extraction, comparaison, rapport

- **SRS-ANA-AN-01** [B] Les repères nommés et commentés DOIVENT vivre dans la session, jamais dans
  la trace.
  *Source* : `EX-ANA-01`. *Vérification* : celle d'`EX-ANA-01`.
- **SRS-ANA-AN-02** [B] L'extraction d'un intervalle DOIT produire un nouveau journal, avec en
  option les bases, les règles et une session jointe portant les repères de l'intervalle.
  *Source* : `EX-ANA-02`. *Vérification* : celle d'`EX-ANA-02`.
- **SRS-ANA-AN-03** [B] La comparaison de deux traces DOIT suivre `EX-ANA-03`.
  *Source* : `EX-ANA-03`. *Vérification* : celle d'`EX-ANA-03`.
- **SRS-ANA-AN-04** [B] Le rapport HTML DOIT suivre `EX-ANA-04`, reproductible depuis la ligne de
  commande au champ de date près.
  *Source* : `EX-ANA-04`. *Vérification* : celle d'`EX-ANA-04`.

### 4.9 Jeu de test et rejeu

- **SRS-ANA-REJ-01** [C] Le **jeu de test** `.tct` DOIT suivre `EX-REJ-01` (entrées en octets ou en
  valeurs de signaux ; temporisation par instant relatif au début du rejeu, par délai depuis
  l'entrée précédente, ou par période avec nombre de répétitions ; règle de délai après une entrée
  périodique) ; format fixé par `O-05`.
  *Source* : `EX-REJ-01`. *Vérification* : celle d'`EX-REJ-01`.
- **SRS-ANA-REJ-02** [C] Un jeu DOIT pouvoir être créé depuis un intervalle de trace ; son modèle
  d'édition couvre ajout, suppression, réordonnancement, duplication, modification par libellé.
  *Source* : `EX-REJ-02`. *Vérification* : celle d'`EX-REJ-02`.
- **SRS-ANA-REJ-03** [C] Le rejeu DOIT offrir sélection des trames, facteur d'échelle (0,1 à 10),
  boucle et remappage de canal, et refuser avant émission un jeu dont la charge exigée dépasse
  100 % sur une fenêtre de 100 ms, sauf option explicite.
  *Source* : `EX-REJ-03`. *Vérification* : celle d'`EX-REJ-03`.
- **SRS-ANA-REJ-04** [C] Le moteur de rejeu DOIT être un `TxSource` : il calcule les instants
  d'émission absolus dans l'horloge monotone de l'hôte, régénère compteurs et contrôles, et soumet
  les trames à l'ordonnanceur du noyau avec l'avance fixée par le point ouvert ANA-3 ; il n'émet
  jamais autrement.
  *Source* : `EX-REJ-04`, `EX-PRF-04`, §4 de `SPEC.md`, `SRS-NOY-TX-03`. *Vérification* : celle
  d'`EX-REJ-04`.
- **SRS-ANA-REJ-05** [C] Chaque rejeu DOIT produire le **rapport de fidélité** d'`EX-REJ-05` depuis
  les échos et échecs restitués par le noyau.
  *Source* : `EX-REJ-05`. *Vérification* : celle d'`EX-REJ-05`.
- **SRS-ANA-REJ-06** [C] Chaque rejeu DOIT commander au noyau l'enregistrement d'une trace corrélée
  (émis et reçu, sens marqué, base de temps unique).
  *Source* : `EX-REJ-06`. *Vérification* : celle d'`EX-REJ-06`.
- **SRS-ANA-REJ-07** [C] Un rejeu sur bus virtuel DOIT émettre la même séquence (canal,
  identifiant, données, ordre) que sur bus réel.
  *Source* : `EX-REJ-08`. *Vérification* : celle d'`EX-REJ-08`.
- **SRS-ANA-REJ-08** [C] Avant tout rejeu réel, la couche d'analyse DOIT calculer l'empreinte
  SHA-256 du jeu, de ses bases, de ses règles et de la liste blanche. Si le démon cible ne l'a pas
  enregistrée, elle DOIT conduire le galop d'essai comme un rejeu sur le bus virtuel ouvert par ce
  démon, constater la conformité des trames émises à leurs règles de contrôle, la déclarer au
  démon, et rapporter le résultat ; les assertions y sont évaluées à titre d'information. Si le
  démon n'inscrit pas la réussite, la commande échoue avec un code retour non nul
  (`EX-AUT-03`), sans rejeu réel.
  *Source* : `EX-SAF-06`, `D-26`, `D-31`. *Vérification* : celle d'`EX-SAF-06` ; code retour non
  nul sur galop non inscrit.
- **SRS-ANA-REJ-09** [C] Quand le noyau signale un risque de collision (`SRS-NOY-TX-06`), la
  couche d'analyse DOIT le présenter et ne transmettre le drapeau d'acceptation qu'avec l'accord
  explicite de l'utilisateur : dans l'IHM par une confirmation, en ligne de commande par une
  option, sans laquelle l'émission est refusée.
  *Source* : `EX-SAF-04`, `EX-AUT-04`, `D-34`. *Vérification* : celle d'`EX-SAF-04`.
- **SRS-ANA-REJ-10** [C] L'intervalle entre deux battements successifs d'un client armé NE DOIT
  PAS dépasser 250 ms.
  *Source* : dérivée — `EX-SAF-02` désarme après 1 s sans battement ; quatre battements par
  seconde évitent un désarmement intempestif sur une latence passagère. *Vérification* : client
  armé pendant 10 min, tous les cœurs chargés à 100 % par des processus concurrents de même
  priorité ; intervalle maximal relevé ≤ 250 ms ; aucun désarmement.
- **SRS-ANA-REJ-11** [C] La ligne de commande DOIT offrir l'**arrêt d'urgence** en une commande
  unique, sans armement préalable, transmise au démon par P7. La commande rend la main après
  confirmation par le démon, avec un code retour nul.
  *Source* : `EX-SAF-05`, `D-20`. *Vérification* : commande lancée par un second client non armé
  pendant un rejeu sur bus virtuel ; délai d'`EX-SAF-05` respecté ; code retour nul.
- **SRS-ANA-REJ-12** [C] La liste blanche proposée par un jeu de test NE DOIT être transmise au
  démon que si l'utilisateur la reprend explicitement à l'armement (option de la commande ou
  action dans l'IHM) ; elle ne s'applique jamais d'elle-même.
  *Source* : `EX-SAF-03`. *Vérification* : jeu portant une liste, armement sans reprise →
  émission refusée (liste vide) ; armement avec reprise explicite → émission acceptée.

### 4.10 Automatisation et ligne de commande

- **SRS-ANA-AUT-01** [A, étendue à chaque palier] La ligne de commande DOIT exposer chaque commande
  du registre livrée au palier courant.
  *Source* : `EX-AUT-01`, `EX-ARC-06`. *Vérification* : celle d'`EX-AUT-01`.
- **SRS-ANA-AUT-02** [C] Les assertions `.tce` DOIVENT suivre `EX-AUT-02` (contraintes de temps,
  évaluation par occurrence, verdicts `pass`, `fail`, `not_evaluated`).
  *Source* : `EX-AUT-02`, `D-25`. *Vérification* : celle d'`EX-AUT-02`.
- **SRS-ANA-AUT-03** [C] Les codes retour et rapports DOIVENT suivre `EX-AUT-03` (0, 1, 2, 3 ;
  JSON à schéma versionné publié ; HTML ou texte).
  *Source* : `EX-AUT-03`. *Vérification* : celle d'`EX-AUT-03`.
- **SRS-ANA-AUT-04** [A, étendue à chaque palier] Aucune commande NE DOIT attendre d'entrée ni
  exiger de serveur graphique : toute confirmation (canal sans écoute seule, mode acquittement,
  collision, entrée générique de liste blanche) s'exprime par une option ; sans elle, la commande
  se comporte comme si la confirmation avait été refusée, et le signale.
  *Source* : `EX-AUT-04`, `D-34`. *Vérification* : celle d'`EX-AUT-04`.

### 4.11 Session, profils, langues

- **SRS-ANA-SES-01** [A, étendue à chaque palier] La session DOIT sérialiser et restituer les objets
  du palier courant listés par `EX-SES-01`.
  *Source* : `EX-SES-01`. *Vérification* : celle d'`EX-SES-01`.
- **SRS-ANA-SES-02** [A] Le format de session DOIT être un texte lisible et diffable : ordre des
  clés stable, une valeur par ligne, chemins relatifs à la session quand c'est possible.
  *Source* : `EX-SES-02`. *Vérification* : celle d'`EX-SES-02`.
- **SRS-ANA-SES-03** [A] Une référence manquante DOIT être signalée par son chemin, sans perdre le
  reste, et conservée à l'enregistrement suivant.
  *Source* : `EX-SES-03`. *Vérification* : celle d'`EX-SES-03`.
- **SRS-ANA-SES-04** [A] Les profils DOIVENT suivre `EX-SES-04` : enregistrés, listés, ouverts par
  leur nom en créant une nouvelle session ; jamais d'armement ; mode acquittement soumis à
  confirmation.
  *Source* : `EX-SES-04`. *Vérification* : celle d'`EX-SES-04`.
- **SRS-ANA-I18N-01** [A] Toute chaîne destinée à un lecteur humain (messages de la ligne de
  commande, rapports HTML ou texte) DOIT avoir l'anglais pour langue source et passer par le
  catalogue de traduction partagé avec l'IHM, en chaînes complètes paramétrées, jamais par
  concaténation. Les sorties machine (rapport JSON, CSV, codes retour) ne sont pas traduites.
  *Source* : `EX-I18N-01`, `EX-I18N-04` ; extension à la ligne de commande dérivée — les messages
  des commandes sont partagés avec l'IHM par le registre (`D-27`), un seul catalogue évite deux
  mécanismes. *Vérification* : celle d'`EX-I18N-04` étendue à `clemcan/` ; sorties machine
  identiques sous deux locales.
- **SRS-ANA-I18N-02** [A] Nombres, dates et durées DOIVENT être formatés selon la locale dans
  toute sortie destinée à un lecteur humain ; les exports suivent leur format explicite
  (`SRS-ANA-TRC-06`).
  *Source* : `EX-I18N-05`. *Vérification* : celle d'`EX-I18N-05`.

### 4.12 Performance

- **SRS-ANA-PRF-01** [B] L'ensemble de règles de référence d'`EX-PRF-06` DOIT être évalué au profil
  nominal sur la machine de référence x86-64 pendant 60 min sans évaluation abandonnée ; le débit
  maximal sur ARM64 est mesuré et publié.
  *Source* : `EX-PRF-06`. *Vérification* : celle d'`EX-PRF-06`.

### 4.13 Sécurité

- **SRS-ANA-SEC-01** [A] Aucun processus de la couche d'analyse, ligne de commande comprise,
  N'OUVRE de port réseau en écoute, en TCP comme en UDP, ni n'établit de connexion sortante autre
  que vers un démon distant configuré ; aucune télémétrie.
  *Source* : `EX-SEC-01`, `EX-SEC-03`, `D-35`. *Vérification* : celle d'`EX-SEC-01` pour
  l'enregistrement en ligne de commande ; celle d'`EX-SEC-03`.

---

## 5. Contraintes de conception

- Python 3.11+, annotations de type partout, vérification statique, formatage et analyse
  automatiques en intégration continue.
- cantools n'est pas réécrit ; tout contournement est isolé, commenté et référencé à un ticket
  amont (filtre de sections, `D-19`). Exceptions décidées : extraction de bits (`D-18`) et
  écriture DBC (`D-06`, `D-19`).
- Les vecteurs de test des fonctions protocolaires (décodage, multiplexage, J1939, ISO-TP, UDS,
  CRC, E2E) sont écrits avant le code et couvrent les cas dégradés.
- Aucune nouvelle dépendance sans fiche de décision.

## 6. Points ouverts

| # | Point | Renvoi |
|---|---|---|
| ANA-1 | Format du jeu de test (`.tct`) | `O-05` |
| ANA-2 | Format texte de la session (`.clemcan`) et des profils : YAML, TOML ou autre — aucune décision ne le fixe | fiche à écrire avant le jalon de session |
| ANA-3 | Horizon d'anticipation de la soumission au noyau pendant le rejeu (compromis entre réactivité de l'arrêt et tolérance aux à-coups de Python) | fiche à écrire avant le palier C |
| ANA-4 | Bibliothèques de lecture BLF, TRC et MF4 (python-can, asammdf), version de cantools à figer | fiches de dépendance |
| ANA-5 | Support des résultats d'analyse hors ligne (`SRS-ANA-CHK-05`, `SRS-ANA-EXP-08`) : dans la session ou dans un fichier annexe | `O-08` |
