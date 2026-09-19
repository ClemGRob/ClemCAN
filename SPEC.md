# ClemCAN — Spécification

> **ClemCAN** — analyseur et éditeur CAN / CAN FD.
> Version du document : 1.0 — 2026-09-19. Licence du projet : MIT.

---

## 1. Objet

ClemCAN est un outil d'observation, d'analyse, d'édition et de rejeu de bus CAN et CAN FD,
utilisable au laboratoire, en atelier, à bord d'un véhicule sans opérateur, et depuis une
cible embarquée pilotée à distance.

Il couvre quatre métiers :

1. **Observer** un bus en temps réel et enregistrer une trace fidèle.
2. **Comprendre** ce trafic : décodage, contrôles de cohérence, statistiques, rétro-ingénierie.
3. **Éditer** la base de données de signaux (DBC) qui donne son sens au trafic.
4. **Agir** : rejouer un jeu de test sur un système réel et en mesurer la réaction.

Ce document décrit **ce que** le logiciel doit faire et **comment le vérifier**.
Il ne décrit pas l'implémentation, sauf lorsqu'une contrainte d'architecture est une exigence
(section 4) ou qu'une décision a été arrêtée (section 10).

### 1.1 Conventions du document

- Chaque exigence porte un identifiant stable `EX-<DOMAINE>-<nn>`. Un identifiant n'est jamais
  réutilisé ni renuméroté : il est amendé ou marqué obsolète.
- Chaque exigence porte un **critère de vérification** exécutable. Une exigence sans critère
  vérifiable est un souhait, pas une exigence.
- `DOIT` = obligatoire en v1. `DEVRAIT` = souhaitable, non bloquant. `PEUT` = optionnel.
- Le palier de livraison (A, B ou C, cf. PLAN.md) est indiqué entre crochets.

---

## 2. Glossaire

| Terme | Définition retenue dans ce document |
|---|---|
| **Trame** | Une unité de transport sur le bus : identifiant, longueur, données, drapeaux, horodatage, canal, sens. |
| **Signal** | Champ de bits à l'intérieur d'une trame, avec endianness, signe, facteur, offset, unité, table de valeurs. |
| **Base de données** | Fichier décrivant trames et signaux : DBC (éditable), SYM, KCD, ARXML (import seul). |
| **Trace** | Journal horodaté de trames observées ou émises. Donnée de mesure : jamais modifiée en place. |
| **Jeu de test** | Document décrivant des trames à émettre, avec temporisation. Donnée fabriquée, versionnable, éditable. |
| **Session** | Espace de travail : disposition, bases chargées, signaux tracés, filtres, règles, jeux ouverts. |
| **Journal natif** | Format d'enregistrement propre à ClemCAN, en ajout seul, conçu pour survivre à une coupure. |
| **Enveloppe nominale** | Charge pour laquelle un fonctionnement sans perte est contractuellement garanti et mesuré. |
| **Décimation** | Réduction du nombre de points transmis à l'affichage, préservant les extrema. |
| **Armement** | Action délibérée et explicite autorisant l'écriture sur un bus, pour la session en cours. |

Extensions de fichiers : journal natif `.tcj` (+ index `.tcx`), session `.clemcan`,
jeu de test `.tct`, règles de contrôle `.tcr` (YAML).
Binaires : démon `clemcand`, ligne de commande `clemcan`, noyau `libclemcan`.

---

## 3. Acteurs et contextes d'emploi

Les quatre contextes suivants sont **tous** dans le périmètre, et chacun impose ses exigences.

| Contexte | Ce qu'il impose |
|---|---|
| **Banc / laboratoire, PC connecté** | Latence d'affichage faible, mise en œuvre rapide, sessions réutilisables. |
| **Véhicule / machine en marche, sans opérateur** | Enregistrement autonome, survie aux coupures d'alimentation brutales, rotation de fichiers, démarrage automatique, aucune interaction requise. |
| **Atelier / diagnostic terrain** | Profils prêts à l'emploi, garde-fous forts avant toute émission, lisibilité immédiate. |
| **Rétro-ingénierie de bus inconnu** | Analyse sans base de données, construction incrémentale du DBC depuis l'observation. |

Cas d'emploi structurants retenus, en plus des besoins initiaux :

- **UC-1 Test automatisé** — le logiciel est piloté par un script ou une chaîne d'intégration
  continue : charger une base, rejouer un scénario, évaluer des assertions, rendre un code
  retour et un rapport. Conséquence : toute fonction utile existe hors interface graphique.
- **UC-2 Enregistrement déclenché** — campagne longue dont on ne garde que les fenêtres
  autour d'événements.
- **UC-3 Analyse partagée** — annoter, extraire un intervalle, le transmettre avec sa base.
- **UC-4 Source de vérité du firmware** — la base éditée produit le code C de sérialisation.

Déploiement : **x86-64 et ARM64**, **Linux et Windows**.

---

## 4. Architecture imposée

L'architecture suivante est une exigence, parce qu'elle seule satisfait simultanément
l'enveloppe de performance (§7), l'autonomie embarquée et le pilotage à distance.

```
┌─────────────────────────────────────────────────────────────┐
│  Couche 3 — Interface (Qt / PySide6)                        │
│  N'affiche que du décimé. Cliente, jetable, remplaçable.    │
└────────────────────────┬────────────────────────────────────┘
                         │  protocole documenté (local ou TCP)
┌────────────────────────┴────────────────────────────────────┐
│  Couche 2 — Analyse (Python)                                │
│  DBC (cantools), décodage, J1939/ISO-TP, statistiques,      │
│  contrôles, expressions, rapports, orchestration du rejeu.  │
│  Ne voit jamais le débit ligne.                             │
└────────────────────────┬────────────────────────────────────┘
                         │  ABI C stable + anneau en mémoire partagée
┌────────────────────────┴────────────────────────────────────┐
│  Couche 1 — Noyau C (clemcand / libclemcan)                 │
│  Propriétaire exclusif du bus : réception, horodatage,      │
│  tampon circulaire, écriture du journal, ordonnanceur TX,   │
│  comptage des pertes. Seul code soumis au débit ligne.      │
└─────────────────────────────────────────────────────────────┘
```

- **EX-ARC-01** [A] Le noyau DOIT être un démon autonome : l'enregistrement se poursuit sans
  interruption si l'interface est fermée, mise à jour ou plantée.
  *Vérification* : `kill -9` du client pendant un enregistrement d'au moins 10 min à charge
  nominale ; la trace résultante est continue, sans trou, et le compteur de pertes reste à zéro.
- **EX-ARC-02** [A] Le noyau DOIT être utilisable en bibliothèque liée, sans démon, pour les
  usages en ligne de commande.
- **EX-ARC-03** [A] Le noyau NE DOIT contenir aucune logique métier : ni décodage de signal,
  ni base de données, ni expression, ni interprétation de protocole applicatif.
  *Vérification* : revue ; absence de toute dépendance du noyau à une base de données.
- **EX-ARC-04** [A] Le protocole entre noyau et clients DOIT être documenté et versionné, de
  sorte qu'une interface écrite dans un autre langage puisse s'y brancher sans modifier le noyau.
- **EX-ARC-05** [A] Par défaut, le démon N'OUVRE AUCUN port réseau en écoute. L'accès distant
  est une option explicite, et il exige alors authentification et chiffrement.
  *Vérification* : `ss -lntp` après démarrage par défaut ne montre aucun socket du démon.
- **EX-ARC-06** [A] Toute fonction accessible depuis l'interface graphique DOIT être accessible
  sans elle (ligne de commande ou protocole). L'interface est une coquille fine.
  *Vérification* : la suite de tests fonctionnels s'exécute sans serveur graphique.

---

## 5. Exigences fonctionnelles

### 5.1 Interfaces matérielles — `EX-BUS`

Deux niveaux de garantie sont définis, et affichés à l'utilisateur.

| Niveau | Contenu | Garanties |
|---|---|---|
| **Tier 1** | SocketCAN (Linux), PCAN-Basic (Windows et Linux), bus virtuel | Débit nominal, horodatage matériel, trames d'erreur, détection de perte, CAN FD |
| **Tier 2** | Tout backend python-can (Vector, Kvaser, ixxat, slcan, gs_usb, socketcand…) | Fonctionnel, sans garantie de débit ni de précision d'horodatage |

- **EX-BUS-01** [A] ClemCAN DOIT gérer un nombre non borné de canaux simultanés. Aucune limite
  ne figure dans le code ni dans les structures de données.
  *Vérification* : test paramétré à 1, 2, 4 et 8 canaux virtuels.
- **EX-BUS-02** [A] Chaque canal DOIT porter son propre débit, son mode (classique ou FD), ses
  bases de données associées et ses règles de contrôle.
- **EX-BUS-03** [A] ClemCAN DOIT lire les trames d'erreur, les trames de requête distante, les
  trames étendues (29 bits) et les trames FD (BRS, ESI, DLC 9 à 15).
- **EX-BUS-04** [A] Le niveau (Tier 1 / Tier 2) du backend utilisé DOIT être affiché dans
  l'interface et inscrit dans les métadonnées de chaque trace.
- **EX-BUS-05** [A] ClemCAN DOIT signaler, avant enregistrement, une incohérence probable de
  débit (aucune trame reçue, ou taux d'erreur anormal après ouverture du bus).
- **EX-BUS-06** [C] Le passage en bus-off DOIT être détecté, horodaté, inscrit dans la trace et
  signalé ; une politique de reprise (aucune, manuelle, automatique) DOIT être configurable.

### 5.2 Capture et horodatage — `EX-CAP`

- **EX-CAP-01** [A] L'horodatage DOIT provenir du matériel ou du noyau du système quand il est
  disponible, jamais de l'espace utilisateur si une source plus précise existe. La source
  effective DOIT être inscrite dans les métadonnées de la trace.
  *Vérification* : trace enregistrée avec un PCAN réel ; la source déclarée est « matériel ».
- **EX-CAP-02** [A] Chaque trace DOIT porter une base de temps monotone (pour les durées) **et**
  une correspondance avec l'heure absolue UTC au démarrage (pour la corrélation externe).
- **EX-CAP-03** [A] Tous les canaux d'une même session DOIVENT partager une base de temps
  commune, permettant de comparer deux bus à la microseconde près quand le matériel le permet.
- **EX-CAP-04** [A] **Aucune perte silencieuse.** Toute trame perdue (débordement de tampon
  noyau, débordement de l'anneau interne, écriture disque en retard) DOIT être comptée,
  horodatée, inscrite dans la trace sous forme d'événement, et signalée dans l'interface.
  *Vérification* : injection délibérée à débit supérieur à l'enveloppe, avec tampon réduit ;
  le nombre de trames perdues déclaré correspond au nombre injecté moins le nombre enregistré.
- **EX-CAP-05** [A] En surcharge, **l'enregistrement est prioritaire** : l'affichage, le décodage
  et le graphe décimatent ou gèlent ; la trace reste complète.
  *Vérification* : à 150 % de l'enveloppe nominale, la trace ne perd aucune trame tandis que le
  flux d'affichage est mesurablement décimé.

### 5.3 Journal natif et robustesse — `EX-JNL`

- **EX-JNL-01** [A] L'enregistrement DOIT s'écrire dans un journal en **ajout seul**, sans bloc
  compressé en cours d'écriture, de sorte qu'une coupure d'alimentation ne perde que les
  dernières trames non encore écrites.
  *Vérification* : `kill -9` puis coupure simulée pendant écriture, 100 fois ; le journal
  résultant est toujours lisible, et la perte n'excède jamais la fenêtre d'écriture annoncée.
- **EX-JNL-02** [A] L'index de navigation DOIT être reconstructible intégralement depuis le
  journal seul. Sa perte n'est jamais une perte de données.
  *Vérification* : suppression de l'index, reconstruction, comparaison bit à bit du contenu relu.
- **EX-JNL-03** [A] Le journal DOIT porter ses métadonnées : version de format, canaux, débits,
  backend et son niveau, source d'horodatage, version de ClemCAN, empreinte des bases de données
  et des règles de contrôle utilisées, sens (reçu / émis) de chaque trame.
- **EX-JNL-04** [B] Le découpage automatique en fichiers DOIT être configurable (par taille, par
  durée, par nombre de trames), avec continuité garantie de la base de temps entre fichiers.
- **EX-JNL-05** [A] ClemCAN DOIT ouvrir une trace de 50 millions de trames et y naviguer
  (saut à un instant, à un identifiant, à une occurrence) en moins d'une seconde par saut, sans
  charger l'intégralité du fichier en mémoire.
  *Vérification* : trace de référence générée, mesures publiées par le banc de performance.

### 5.4 Formats d'échange — `EX-FMT`

| Format | Lecture v1 | Écriture v1 |
|---|---|---|
| Journal natif `.tcj` | oui | oui |
| Vector ASC | oui | oui |
| candump `.log` (SocketCAN) | oui | oui |
| CSV de signaux décodés | — | oui |
| Vector BLF | oui | **non** (premier jalon post-v1) |
| PEAK TRC (versions 1.0 à 2.1) | oui | non |
| ASAM MDF v4 (MF4) | oui | non |

- **EX-FMT-01** [A] Tous les formats de la colonne « lecture » DOIVENT être lus, y compris leurs
  variantes de version, et convertis vers le modèle de trame interne unique.
- **EX-FMT-02** [A] Un **modèle de trame intermédiaire unique** DOIT porter tous les attributs
  transportables par au moins un format : identifiant, étendu ou non, longueur, DLC brut,
  données, FD, BRS, ESI, trame d'erreur, requête distante, canal, sens, horodatage, marqueur
  utilisateur, commentaire.
- **EX-FMT-03** [A] Une **matrice de conformité** DOIT être publiée avec le logiciel, indiquant
  pour chaque format et chaque attribut : conservé, approximé, ou perdu.
  *Vérification* : la matrice est générée automatiquement depuis les tests d'aller-retour ; un
  écart entre la matrice et le comportement réel casse la construction.
- **EX-FMT-04** [A] Les conversions DOIVENT être testées par aller-retour sur des fichiers de
  référence versionnés, et toute perte DOIT être soit documentée dans la matrice, soit un échec.
- **EX-FMT-05** [A] L'export CSV de signaux décodés DOIT proposer explicitement le mode
  d'échantillonnage (aux instants d'arrivée, ou rééchantillonné à pas fixe avec maintien) et le
  format de nombre localisé (point ou virgule décimale, séparateur de colonne cohérent).
- **EX-FMT-06** [A] Une trace importée DOIT rester distinguable d'une trace enregistrée par
  ClemCAN : l'origine figure dans les métadonnées.

### 5.5 Bases de données — `EX-DB`

Le parsing s'appuie sur **cantools**, qui NE DOIT PAS être réécrit.

- **EX-DB-01** [A] DBC : lecture complète, y compris `BO_`, `SG_`, `CM_`, `BA_DEF_`, `BA_`,
  `VAL_`, `VAL_TABLE_`, `SIG_VALTYPE_`, `SG_MUL_VAL_`, `BO_TX_BU_`, `SIG_GROUP_`, `EV_`.
- **EX-DB-02** [A] SYM (PEAK), KCD et ARXML (AUTOSAR) : **import en lecture seule**, converti
  vers le modèle interne, accompagné d'un rapport explicite des constructions non transposables.
- **EX-DB-03** [B] **Écriture DBC au diff minimal.** Le texte source est conservé ; seules les
  lignes concernées par une modification sont réécrites. Tout octet non touché est préservé :
  ordre, indentation, commentaires, sections inconnues.
  *Vérification* : sur un corpus de DBC réels et synthétiques, un aller-retour sans édition rend
  un fichier **identique octet pour octet** ; une édition d'un facteur produit un diff Git
  n'excédant pas les lignes de la définition concernée.
- **EX-DB-04** [B] **Validation bloquante** avant enregistrement, couvrant au minimum :
  chevauchement de signaux, signal dépassant la longueur de trame, identifiants dupliqués,
  multiplexage incohérent ou orphelin, `min`/`max` incompatibles avec facteur, offset et largeur
  de bits, noms invalides ou dupliqués, table de valeurs hors plage, incohérences J1939
  (PGN/adresse), DLC incompatible avec le mode du canal.
  *Vérification* : un jeu de bases volontairement fautives, une par règle, chacune détectée avec
  un message désignant l'objet exact.
- **EX-DB-05** [B] **Matrice de bits interactive** : édition visuelle de l'implantation des
  signaux en grille de 8 bits par octet, placement et redimensionnement à la souris, gestion de
  l'ordre Motorola et Intel, recouvrements signalés en temps réel.
- **EX-DB-06** [B] **Annuler / refaire** sur toute la session d'édition, sans limite de
  profondeur, y compris pour les opérations de masse.
- **EX-DB-07** [B] **Édition par lot** : renommage par motif, application d'un facteur ou d'une
  unité à une sélection, déplacement de signaux entre trames, duplication d'une trame vers un
  autre émetteur.
- **EX-DB-08** [B] **Comparaison et fusion sémantiques** de deux bases : trames et signaux
  ajoutés, supprimés, déplacés, redimensionnés ; facteurs, offsets, unités et tables de valeurs
  modifiés ; application sélective des écarts, avec prévisualisation.
- **EX-DB-09** [A] Plusieurs bases DOIVENT pouvoir être associées à un même canal, avec un ordre
  de priorité déterministe et un signalement explicite des identifiants définis deux fois.
- **EX-DB-10** [B] **Génération de code C** de sérialisation et désérialisation depuis la base.
  *Vérification* : pour un échantillon aléatoire de trames et de valeurs, l'encodage puis le
  décodage par le code C généré DOIT rendre exactement le même résultat que le décodage interne
  (comparaison différentielle automatisée, au moins 10⁵ vecteurs par base de test).
- **EX-DB-11** [B] Recherche de signaux et de trames par nom, identifiant, nœud, unité, commentaire.

### 5.6 Décodage — `EX-DEC`

- **EX-DEC-01** [A] Ordre d'octets Motorola (gros-boutien) **et** Intel (petit-boutien), dans une
  même trame au besoin.
- **EX-DEC-02** [A] Signaux signés (complément à deux), non signés, flottants simple et double
  précision (`SIG_VALTYPE_`).
- **EX-DEC-03** [A] Valeur brute (hexadécimale et décimale), valeur physique (facteur, offset,
  unité), et libellé de table de valeurs (`VAL_`) — les trois disponibles au choix, simultanément
  affichables, et sélectionnables par signal.
- **EX-DEC-04** [A] **Multiplexage simple** (multiplexeur et signaux multiplexés) **et
  multiplexage étendu** (`SG_MUL_VAL_`, plages de valeurs, imbrication).
  *Vérification* : vecteurs de test dédiés, y compris un cas de multiplexeur imbriqué et un cas
  de valeur de multiplexeur non définie (le signal est alors déclaré non applicable, non « zéro »).
- **EX-DEC-05** [A] CAN FD jusqu'à 64 octets, avec distinction entre DLC brut et longueur réelle.
- **EX-DEC-06** [A] J1939 au niveau trame : décomposition de l'identifiant en priorité, PGN,
  adresse source, adresse de destination (PDU1 / PDU2).
- **EX-DEC-07** [B] **Réassemblage du transport J1939** : diffusion BAM et mode connexion
  RTS/CTS (`TP.CM` / `TP.DT`), jusqu'à 1785 octets, avec gestion des temporisations normalisées,
  des sessions concurrentes par couple d'adresses, et des abandons.
  *Vérification* : vecteurs couvrant le cas nominal, la trame manquante, la trame dupliquée, le
  dépassement de temporisation, deux sessions simultanées, un abandon explicite.
- **EX-DEC-08** [B] **J1939 applicatif** : DM1 et DM2 (SPN, FMI, compteur d'occurrences, état des
  lampes), réclamation d'adresse et champ NAME, requête de PGN.
- **EX-DEC-09** [B] **ISO-TP (ISO 15765-2)** : trame unique, première trame, trames consécutives,
  contrôle de flux ; adressage normal, étendu et mixte ; identifiants 11 et 29 bits ; gestion des
  temporisations, du bloc et du temps de séparation.
- **EX-DEC-10** [B] **UDS en lecture** : identification des services, des sous-fonctions et des
  codes de refus ; décodage au minimum de `0x22` (lecture par identifiant) et `0x19` (défauts).
  L'**émission** de requêtes UDS est hors périmètre v1 (cf. §8).
- **EX-DEC-11** [A] Le décodage DOIT être **paresseux** : seuls les signaux effectivement
  affichés, exportés ou évalués sont décodés.
  *Vérification* : à charge nominale, le temps processeur consommé par le décodage est
  proportionnel au nombre de signaux affichés, mesuré par le banc.
- **EX-DEC-12** [A] Un identifiant absent des bases DOIT rester visible et exploitable en brut,
  jamais masqué.

### 5.7 Contrôles de cohérence, compteurs et CRC — `EX-CHK`

Le DBC ne sait pas décrire un compteur ni un CRC : ces règles vivent dans un fichier annexe.

- **EX-CHK-01** [B] Un fichier de règles `.tcr` lisible et versionnable DOIT décrire, par trame :
  le signal compteur (modulo, pas, tolérance), le signal de contrôle, l'algorithme (profils
  AUTOSAR E2E, CRC8 SAE-J1850, CRC15/16/32 avec polynôme, valeur initiale, ou-exclusif final,
  réflexions), l'étendue d'octets couverte, et le traitement des octets exclus.
- **EX-CHK-02** [B] ClemCAN DOIT proposer une **détection assistée** des compteurs et des contrôles
  par convention de nommage et par vérification statistique sur une trace, l'utilisateur validant
  la proposition. La règle retenue est écrite dans le fichier `.tcr` : le résultat est reproductible.
- **EX-CHK-03** [B] Les violations détectées DOIVENT être : compteur figé, compteur sauté,
  compteur dupliqué, contrôle invalide. Chacune est horodatée, comptée, visible dans la trace et
  dans la chronologie, et exploitable comme condition (cf. `EX-EXP`).
- **EX-CHK-04** [B] Contrôles de cohérence supplémentaires : franchissement des bornes physiques,
  temps de cycle hors tolérance (trop lent, trop rapide, gigue), longueur de trame inattendue,
  valeur absente de la table `VAL_`, identifiant inconnu des bases, émetteur inattendu.
- **EX-CHK-05** [C] **Au rejeu, compteurs et contrôles DOIVENT être recalculés** pour toute trame
  modifiée ou régénérée, sinon le système cible rejette les trames sans diagnostic.
  *Vérification* : un jeu de test dérivé d'une trace protégée, modifié sur un signal, rejoué sur
  bus virtuel ; toutes les trames émises passent la validation de contrôle.

### 5.8 Vues — `EX-VUE`

- **EX-VUE-01** [A] **Trace déroulante** : flux chronologique brut, défilement continu,
  colonnes configurables, mise en pause sans perte d'enregistrement, dépliage des signaux d'une
  trame sélectionnée.
- **EX-VUE-02** [A] **Vue groupée par identifiant** : une ligne par identifiant, mise à jour sur
  place, avec période mesurée (moyenne, minimum, maximum, gigue), compteur d'occurrences, date de
  dernière réception, et mise en évidence des octets qui viennent de changer.
- **EX-VUE-03** [A] **Fenêtre de valeurs** : tableau des signaux choisis, valeur courante au
  format choisi (brut, physique, libellé), âge de la dernière mise à jour, indication de
  franchissement de bornes, regroupement libre.
- **EX-VUE-04** [A] **Statistiques et santé du bus** : charge en pourcentage, trames par seconde,
  histogramme des temps de cycle par identifiant, compteurs de trames d'erreur, passages en
  bus-off, identifiants apparus ou disparus, trames perdues.
  *Vérification* : sur une injection de débit connu, la charge calculée s'écarte de moins de 2 %
  de la valeur théorique (longueur de trame, bourrage de bits inclus).
- **EX-VUE-05** [B] **Analyse sans base de données** : bits qui basculent par identifiant, octets
  qui changent, détection de compteurs probables et d'octets de contrôle probables, corrélation
  entre un intervalle de temps et les changements observés, et **création d'un signal candidat
  directement depuis une sélection de bits**, injecté dans la base en cours d'édition.
- **EX-VUE-06** [A] Toute vue DOIT rester réactive à charge nominale : première trame affichée en
  moins de 100 ms après réception, et aucune opération d'interface ne bloque plus de 200 ms.
- **EX-VUE-07** [A] Les vues DOIVENT être disposables librement (fenêtres ancrables, plusieurs
  écrans) et leur disposition DOIT être enregistrée dans la session.

### 5.9 Graphe — `EX-GRA`

- **EX-GRA-01** [A] Modes de tracé par signal : **escalier** (maintien de la dernière valeur,
  **par défaut**), ligne interpolée, points nus. Les instants d'échantillonnage réels DOIVENT
  pouvoir être affichés en surimpression pour ne jamais confondre mesure et maintien.
  *Justification* : un signal CAN n'existe qu'aux instants d'arrivée de sa trame ; interpoler
  invente des valeurs, ce qui est faux pour un compteur et absurde pour une énumération.
- **EX-GRA-02** [A] Le choix de mode par signal DOIT être mémorisé dans la session, et DOIT être
  la même sémantique que celle utilisée pour l'export, la comparaison et les assertions.
- **EX-GRA-03** [A] Plusieurs axes verticaux, échelles automatiques ou fixées, courbes empilées
  ou superposées, couleurs mémorisées par signal.
- **EX-GRA-04** [A] Curseurs de mesure (au moins deux) avec affichage des valeurs à la position
  et des écarts Δt et Δy.
- **EX-GRA-05** [A] Synchronisation bidirectionnelle entre graphe, trace et repères : cliquer un
  point sélectionne la trame correspondante, et inversement.
- **EX-GRA-06** [A] La décimation d'affichage DOIT préserver les extrema : une impulsion d'une
  seule trame reste visible à toute échelle de zoom.
  *Vérification* : trace contenant une impulsion isolée de durée 1 ms dans une heure de données ;
  l'impulsion est visible au zoom le plus large.
- **EX-GRA-07** [A] Un signal multiplexé non applicable à un instant donné DOIT présenter une
  interruption de tracé, non une valeur maintenue.
- **EX-GRA-08** [B] Signaux énumérés tracés avec leurs libellés en ordonnée.
- **EX-GRA-09** [B] Export du graphe en image, et export des séries affichées en CSV.

### 5.10 Langage d'expression unifié — `EX-EXP`

- **EX-EXP-01** [B] Un **unique langage d'expression** DOIT servir aux filtres d'affichage, aux
  filtres de capture, aux conditions de déclenchement, aux alarmes et aux assertions de test.
- **EX-EXP-02** [B] Il DOIT permettre de désigner : un canal, un identifiant, un signal par nom
  (qualifié par canal et par base en cas d'ambiguïté), une valeur physique, brute ou de libellé,
  la longueur, le sens, les drapeaux de trame, et les résultats de contrôle (`crc_invalid`,
  `counter_skip`…).
- **EX-EXP-03** [B] Il DOIT gérer les opérateurs de comparaison et logiques, les changements
  d'état (front montant d'une condition), et le **temps** : durée pendant laquelle une condition
  est vraie, et **absence** de trame pendant un délai.
  *Exemple* : `can0.EngineSpeed > 3000 && Gear == "R"` ; `absent(0x120, 500ms)` ; `crc_invalid`.
- **EX-EXP-04** [B] Une expression invalide DOIT être refusée à la saisie avec la position et la
  cause de l'erreur, jamais silencieusement ignorée.
- **EX-EXP-05** [B] Quand le backend le permet, les filtres de capture DOIVENT être appliqués au
  niveau matériel ou noyau ; le fait qu'un filtre soit matériel ou logiciel DOIT être visible,
  car un filtre matériel modifie ce qui est enregistré.

### 5.11 Enregistrement déclenché — `EX-TRG`

- **EX-TRG-01** [B] Tampon circulaire **pré-déclenchement** de durée ou de taille configurable :
  au déclenchement, les trames antérieures sont conservées.
- **EX-TRG-02** [B] Conditions de déclenchement exprimées dans le langage `EX-EXP`, y compris sur
  trame d'erreur, contrôle invalide, absence de trame et franchissement de borne.
- **EX-TRG-03** [B] Durée post-déclenchement configurable, déclenchements multiples, et
  découpage automatique d'un fichier par déclenchement.
- **EX-TRG-04** [B] Le déclenchement DOIT être inscrit dans la trace comme un événement daté
  portant l'expression qui l'a produit.
  *Vérification* : scénario injecté avec une violation à un instant connu ; le fichier produit
  contient exactement la fenêtre demandée autour de cet instant, aux tolérances près.

### 5.12 Annotations, extraction, comparaison, rapport — `EX-ANA`

- **EX-ANA-01** [B] Poser des repères nommés et commentés sur la base de temps, les parcourir, et
  les conserver dans la session (jamais dans la trace d'origine, qui reste inaltérée).
- **EX-ANA-02** [B] Sélectionner un intervalle et l'exporter seul, avec option d'inclure les
  bases de données et les règles employées, pour transmission à un tiers.
  *Vérification* : la sous-trace exportée, réouverte sur une machine vierge, décode exactement
  comme l'originale.
- **EX-ANA-03** [B] **Comparaison de deux traces**, alignées sur la base de temps ou sur un
  événement de référence : identifiants présents dans l'une seulement, temps de cycle modifiés,
  trajectoires de signaux divergeant au-delà d'un seuil, violations apparues ou disparues.
- **EX-ANA-04** [B] **Rapport exportable** (HTML, et PDF si disponible) : intervalle, graphes,
  repères, statistiques, violations, résultats d'assertions, versions et empreintes des bases
  employées. Le rapport DOIT être reproductible depuis la ligne de commande.

### 5.13 Jeu de test et rejeu — `EX-REJ`

- **EX-REJ-01** [C] Le **jeu de test** est un document propre, lisible, versionnable, contenant
  une liste ordonnée d'entrées : canal, trame exprimée **soit en octets bruts soit en valeurs de
  signaux**, et temporisation (instant absolu, délai depuis la précédente, ou période avec
  nombre de répétitions).
- **EX-REJ-02** [C] Un jeu de test DOIT pouvoir être créé par sélection d'un intervalle de trace,
  puis édité dans un tableau (ajout, suppression, réordonnancement, duplication, modification de
  valeurs de signaux par libellé).
- **EX-REJ-03** [C] Le rejeu DOIT permettre de choisir les trames à émettre (par identifiant, par
  expression, par sélection), un facteur d'échelle de temps, une répétition en boucle, et un
  remappage de canal.
- **EX-REJ-04** [C] **Fidélité temporelle** : gigue au 99ᵉ centile inférieure à 1 ms jusqu'à
  5 000 trames par seconde, et dérive cumulée inférieure à 1 ms sur une heure (ordonnancement sur
  horloge absolue, jamais par accumulation de délais). Les mécanismes de cadencement offerts par
  la plateforme DOIVENT être exploités quand ils existent.
- **EX-REJ-05** [C] Chaque rejeu DOIT produire un **rapport de fidélité** : médiane, 99ᵉ centile
  et pire cas de l'écart entre instant demandé et instant réel, nombre d'émissions échouées.
  *Un rejeu dont la fidélité n'est pas mesurée n'est pas un test reproductible.*
- **EX-REJ-06** [C] Le rejeu DOIT enregistrer une **nouvelle trace corrélée** contenant le trafic
  réellement émis (avec ses instants réels) **et** le trafic reçu pendant le rejeu, chaque trame
  marquée par son sens, dans une base de temps unique.
- **EX-REJ-07** [C] Les compteurs et contrôles DOIVENT être recalculés conformément à `EX-CHK-05`.
- **EX-REJ-08** [C] Un rejeu DOIT être exécutable sur bus virtuel, sans matériel, avec le même
  résultat fonctionnel.

### 5.14 Garde-fous d'émission — `EX-SAF`

- **EX-SAF-01** [C] ClemCAN DOIT démarrer **en écoute seule**, sur tous les canaux, sans exception.
- **EX-SAF-02** [C] Émettre exige un **armement délibéré**, nommant explicitement l'interface
  cible, valable pour la session en cours seulement, et jamais mémorisé d'une session à l'autre.
- **EX-SAF-03** [C] Une **liste blanche d'identifiants émissibles** DOIT pouvoir être définie ;
  hors liste, l'émission est refusée avec un message explicite.
- **EX-SAF-04** [C] ClemCAN DOIT avertir si un identifiant que l'on s'apprête à émettre est déjà
  émis par un autre nœud observé sur le bus (risque de collision d'arbitrage).
- **EX-SAF-05** [C] Un **indicateur d'émission très visible** DOIT être présent dès qu'un canal
  est armé, et un **arrêt d'urgence** DOIT couper toute émission immédiatement, accessible en une
  action unique, y compris au clavier.
- **EX-SAF-06** [C] Un **galop d'essai sur bus virtuel** DOIT être exécuté et réussi avant le
  premier rejeu réel d'un jeu de test donné.
- **EX-SAF-07** [C] Toute trame émise DOIT être journalisée, sans exception et sans possibilité de
  désactiver cette journalisation.
  *Vérification* : une tentative d'émission sans armement échoue et est tracée ; l'arrêt d'urgence
  interrompt l'émission en moins de 50 ms, mesuré sur bus virtuel.

### 5.15 Automatisation — `EX-AUT`

- **EX-AUT-01** [C] Une ligne de commande DOIT couvrir : chargement de bases, ouverture de bus,
  enregistrement, conversion de formats, décodage vers CSV, rejeu d'un jeu de test, évaluation
  d'assertions, génération de rapport, génération de code C.
- **EX-AUT-02** [C] Les **assertions** DOIVENT s'exprimer dans le langage `EX-EXP`, avec des
  contraintes de temps (« ce signal atteint telle valeur en moins de 200 ms après tel événement »).
- **EX-AUT-03** [C] Le code retour DOIT être exploitable en intégration continue : zéro si toutes
  les assertions passent, non nul sinon, avec un rapport lisible par une machine (JSON) **et** par
  un humain (HTML ou texte).
- **EX-AUT-04** [C] Aucune sortie interactive ne DOIT être requise : aucune invite, aucun serveur
  graphique, comportement identique sous intégration continue et en local.

### 5.16 Session et internationalisation — `EX-SES`, `EX-I18N`

- **EX-SES-01** [A] Une session DOIT restituer : disposition des fenêtres, bases chargées par
  canal, signaux tracés avec échelles, couleurs et modes, filtres actifs, règles de contrôle,
  jeux de test ouverts, repères.
- **EX-SES-02** [A] Le format de session DOIT être lisible par un humain et diffable.
- **EX-SES-03** [B] Une session DOIT pouvoir être ouverte en l'absence d'un fichier référencé,
  en signalant précisément ce qui manque, sans perdre le reste.
- **EX-I18N-01** [A] La langue source est l'**anglais** (code, journaux, documentation technique).
- **EX-I18N-02** [A] L'interface DOIT être traduite en français dès le v1, la langue étant
  sélectionnable dans l'interface et mémorisée.
- **EX-I18N-03** [A] Le catalogue de langues DOIT être **ouvert** : ajouter une langue consiste à
  déposer un fichier de traduction, sans recompiler.
- **EX-I18N-04** [A] Repli sur l'anglais pour toute chaîne non traduite ; **aucune phrase
  construite par concaténation** dans le code (les traductions resteraient agrammaticales).
- **EX-I18N-05** [A] Nombres, dates et durées DOIVENT être formatés selon la locale, et les
  exports CSV DOIVENT rendre explicite le couple séparateur décimal / séparateur de colonne.

---

## 6. Exigences non fonctionnelles de performance

### 6.1 Enveloppe nominale — `EX-PRF`

L'architecture est **non bornée en canaux** ; l'enveloppe ci-dessous est ce qui est
**contractuellement garanti et mesuré**.

- **EX-PRF-01** [A] **x86-64** : 4 canaux CAN classiques à 1 Mbit/s saturés (≈ 60 000 trames/s
  agrégées) **et** 4 canaux CAN FD (500 kbit/s d'arbitrage, 2 Mbit/s de données, ≈ 100 000
  trames/s agrégées), pendant 60 minutes, **sans aucune perte d'enregistrement**, empreinte
  mémoire stable (variation inférieure à 5 % après les 5 premières minutes).
- **EX-PRF-02** [A] **ARM64** : tenir le débit que le matériel présente effectivement, et compter
  toute perte. *Note factuelle* : sur cartes de type Raspberry Pi, le facteur limitant est
  généralement le contrôleur (SPI MCP2515/MCP2518FD, ou latence USB), non le processeur. L'objectif
  indicatif est de 2 canaux à 1 Mbit/s saturés ; le chiffre réellement atteint est publié par le
  banc, par plateforme.
- **EX-PRF-03** [A] Au-delà de l'enveloppe, la dégradation DOIT être **observable** : compteurs de
  pertes, avertissement dans l'interface, événement dans la trace. Jamais de silence.
- **EX-PRF-04** [C] Rejeu : conforme à `EX-REJ-04`.
- **EX-PRF-05** [A] Les chiffres de performance DOIVENT être produits par un banc automatisé,
  exécuté au moins chaque nuit, et publiés avec la plateforme, la version et la date.

### 6.2 Robustesse

- **EX-ROB-01** [A] Une coupure d'alimentation ne DOIT jamais rendre une trace illisible
  (cf. `EX-JNL-01`).
- **EX-ROB-02** [B] Le démon DOIT pouvoir démarrer automatiquement au démarrage du système et
  reprendre un enregistrement selon une configuration, sans intervention.
- **EX-ROB-03** [A] Un plantage de l'interface ne DOIT jamais interrompre un enregistrement
  (cf. `EX-ARC-01`).
- **EX-ROB-04** [A] L'espace disque DOIT être surveillé : avertissement à un seuil configurable,
  et politique explicite à la saturation (arrêt propre, ou rotation avec effacement du plus ancien).

### 6.3 Portabilité et livraison

- **EX-POR-01** [A] Linux x86-64, Linux ARM64, Windows x86-64.
- **EX-POR-02** [A] Livraison sous forme d'**exécutable autonome** ne nécessitant l'installation
  d'aucun environnement d'exécution chez l'utilisateur (un exécutable Windows, une image portable
  Linux), plus une installation par gestionnaire de paquets pour les usages de développement.
- **EX-POR-03** [A] Le démon DOIT être livrable et utilisable seul, sans composant graphique.
- **EX-POR-04** [A] Aucune dépendance à un service en ligne. Fonctionnement intégralement hors
  réseau.

### 6.4 Sécurité

- **EX-SEC-01** [A] Aucun port en écoute par défaut (`EX-ARC-05`).
- **EX-SEC-02** [B] L'accès distant, s'il est activé, DOIT être authentifié et chiffré, et
  l'armement d'émission DOIT être refusé par défaut aux clients distants.
- **EX-SEC-03** [A] Aucune télémétrie, aucune remontée d'information.

---

## 7. Conformité protocolaire couverte

| Domaine | Couvert en v1 | Palier |
|---|---|---|
| CAN 2.0A / 2.0B (11 et 29 bits) | oui | A |
| CAN FD (BRS, ESI, DLC 9–15, 64 octets) | oui | A |
| Ordre Motorola et Intel, signaux signés, flottants | oui | A |
| Multiplexage simple et étendu (`SG_MUL_VAL_`) | oui | A |
| J1939 niveau trame (PGN, SA, DA, priorité) | oui | A |
| J1939 transport BAM et CM/DT | oui | B |
| J1939 applicatif DM1/DM2, address claim, NAME, requête PGN | oui | B |
| ISO 15765-2 (ISO-TP), adressages normal/étendu/mixte | oui | B |
| UDS en lecture (services, sous-fonctions, codes de refus) | oui | B |
| ISO 11783 transport étendu (ETP) | **non** | — |
| LIN, FlexRay, Ethernet automobile | **non** | — |

---

## 8. Hors périmètre

**Hors v1, prévu v2 :**
- **Émission réactive** : « si je reçois ceci, j'émets cela » — simulation de nœud, séquenceur,
  panneau d'envoi manuel, générateurs périodiques interactifs, réponses automatiques.
  *L'architecture le prévoit* : le moteur de rejeu est un `TxSource` derrière un ordonnanceur
  d'émission unique ; le moteur réactif sera un second `TxSource`.
- Émission de requêtes UDS et fonctions de diagnostic actives (lecture de DTC à la demande,
  scan d'ECU, écriture de paramètres).
- Écriture BLF (**premier jalon post-v1**, le moins coûteux et le plus utile en échange),
  puis écriture TRC, puis écriture MF4.

**Hors v1, non planifié :**
- Panneaux graphiques interactifs (jauges, boutons, voyants) à la manière de CANoe.
- Scripting utilisateur embarqué (au-delà du pilotage par ligne de commande).
- Bus autres que CAN et CAN FD.
- Synchronisation GPS ou PPS, corrélation vidéo.
- Travail multi-utilisateur simultané sur une même session.
- Mise à jour automatique du logiciel.
- ISO 11783 ETP et variantes de transport sur FD.

---

## 9. Risques

| # | Risque | Impact | Traitement retenu |
|---|---|---|---|
| R1 | L'enveloppe 4 canaux saturés n'est pas tenue | Exigence centrale non satisfaite | Chemin chaud isolé en C dès le premier jalon ; banc de charge automatisé permanent ; optimisations identifiées d'avance (lecture par lots, horodatage noyau, écritures groupées, priorité temps réel, épinglage de cœur) |
| R2 | Frontière noyau C ↔ Python coûteuse ou fragile | Complexité, régressions | Interface étroite et versionnée, anneau en mémoire partagée, tests de contrat des deux côtés, aucune logique métier dans le noyau |
| R3 | L'écriture DBC au diff minimal est plus difficile que prévu | Jalon B glissé | Corpus de DBC réels constitué **avant** l'implémentation ; critère « aller-retour identique au bit » testé en continu ; repli documenté : refus d'enregistrer plutôt que perte silencieuse |
| R4 | Réassemblage J1939/ISO-TP incorrect sur cas dégradés | Fausses analyses | Vecteurs de test des cas dégradés écrits avant le code ; machines à états explicites et testées isolément |
| R5 | Périmètre v1 trop large pour un développeur seul | Abandon | Trois paliers livrables et utilisables (PLAN.md), chacun ayant sa valeur d'usage propre |
| R6 | Plusieurs agents d'IA produisent du code divergent | Dette, incohérences | CLAUDE.md comme contrat unique ; exigences numérotées ; toute décision hors spécification fait l'objet d'une fiche de décision |
| R7 | Matériel ARM limitant (SPI/USB) mal anticipé | Attentes déçues | `EX-PRF-02` formulé en « tenir ce que le matériel présente et compter les pertes » ; mesures publiées par plateforme |
| R8 | Émission accidentelle sur un système réel | Matériel, sécurité des personnes | `EX-SAF-01` à `EX-SAF-07` : écoute seule par défaut, armement explicite, liste blanche, arrêt d'urgence, galop d'essai obligatoire |
| R9 | Nom du projet | Confusion de marque | **Clos le 2026-09-19.** Décision D-09 : « ClemCAN ». Libre sur PyPI, npm et crates.io, aucune mention sur GitHub. Recherche `data.inpi.fr` — base couvrant les marques françaises, de l'Union européenne et internationales en vigueur en France — : **0 marque, 0 brevet, 0 dessin ou modèle**, et 1 seule entreprise homonyme (SCI de location immobilière, SIREN 844 974 618, sans recouvrement d'activité donc sans risque de confusion). Seul reliquat, sans objet aujourd'hui : aucun registre hors Europe consulté |

---

## 10. Fiches de décision

Décisions arrêtées au terme de l'entretien de conception du 2026-09-19. Toute remise en cause
exige une nouvelle fiche, pas une modification silencieuse.

| # | Décision | Justification | Rejeté |
|---|---|---|---|
| **D-01** | **Noyau C** (démon `clemcand` + `libclemcan`) propriétaire du bus ; **analyse et interface en Python**, interface **Qt (PySide6)** | Le seul code soumis au débit ligne est isolé et optimisable ; le reste du périmètre (l'essentiel du travail) bénéficie de cantools, asammdf, numpy, python-can. Une fois le chemin chaud en C, le langage de l'interface devient un choix d'ergonomie, non de performance | Tout Python (appel système par trame trop coûteux sur ARM, `recvmmsg` inaccessible) ; tout C++ (couche driver, BLF et MF4 à écrire, 20–32 mois) ; interface web (port en écoute sur un outil qui écrit sur un bus, deux langages, graphe temps réel plus difficile) |
| **D-02** | Le noyau est un **démon**, pas seulement une bibliothèque | Résout d'une seule pièce l'enregistrement autonome en véhicule et le pilotage d'une cible embarquée depuis un PC ; l'interface devient jetable | Bibliothèque en processus unique (un plantage d'interface perdrait l'enregistrement) |
| **D-03** | **Tier 1** : SocketCAN + PCAN-Basic + bus virtuel, natifs dans le noyau C. **Tier 2** : tout backend python-can, sans garantie de débit ni d'horodatage | Couvre le matériel réellement disponible (PEAK PCAN-USB) sur les deux systèmes, sans écrire cinq pilotes, tout en restant ouvert | Écrire tous les pilotes ; ou tout déléguer à python-can |
| **D-04** | **Journal natif en ajout seul** comme format d'enregistrement ; conversion vers les formats d'échange après coup | Un format à blocs compressés perd la fin du fichier sur coupure ; l'ajout seul borne la perte à la fenêtre d'écriture | Écrire directement en BLF ou MF4 |
| **D-05** | **Écriture v1 limitée** au journal natif, ASC, candump et CSV ; BLF, TRC et MF4 en lecture seule | Choix explicite de l'utilisateur, périmètre v1 resserré | Parité lecture/écriture complète (MF4 coûteux et difficile à valider) |
| **D-06** | **Écriture DBC au diff minimal**, écrivain propre au projet, lecture par cantools | La base éditée part en génération de code C et vit dans Git : un fichier entièrement reformaté rend la revue impossible et fait disparaître les sections non modélisées | Écriture par cantools (réécriture complète, pertes silencieuses) |
| **D-07** | **Escalier par défaut**, choix de mode par signal, sémantique partagée avec export et assertions | Un signal CAN n'existe qu'aux instants d'arrivée ; interpoler invente des valeurs | Ligne interpolée par défaut ; escalier imposé partout |
| **D-08** | **Règles de compteur et de contrôle dans un fichier annexe** `.tcr`, avec détection assistée validée par l'utilisateur | Le DBC ne sait pas décrire un CRC ; le fichier annexe reste portable, versionnable, partageable, et ne modifie pas une base fournie par un tiers | Attributs `BA_` dans le DBC (risque de suppression par les outils tiers) ; détection automatique seule (non reproductible, et inutilisable pour recalculer à l'émission) |
| **D-09** | Nom **ClemCAN** (`clemcand`, `clemcan`, `libclemcan`) | Vérifié libre le 2026-09-19 : disponible sur PyPI, npm et crates.io, **aucune** mention sur GitHub, et recherche INPI (marques françaises, de l'Union européenne et internationales en vigueur en France) **sans antériorité : 0 marque, 0 brevet, 0 dessin ou modèle**. Aucune ambiguïté possible dans le domaine CAN | `CanAlyzer` (quasi-homonyme de CANalyzer, marque Vector, même catégorie de produit) ; `Toucan` (Rusoku Technologies commercialise une gamme d'adaptateurs USB-CAN **TouCAN** livrée avec sa propre interface Qt d'analyse ; TouCAN est en outre le nom du module contrôleur CAN 2.0B de Motorola/Freescale ; et `toucan` est déjà pris sur PyPI, npm et crates.io) ; `Boucan`, `Busard`, `ClémenCAN` (libres, écartés par préférence) |
| **D-10** | **Anglais** langue source, français livré, **catalogue de langues ouvert** | Le domaine est anglophone ; l'interface doit être utilisable par un technicien francophone ; prévu tôt, le coût est nul | Français seul (ferme la contribution) ; anglais seul (interface moins immédiate en atelier) |
| **D-11** | **Un seul langage d'expression** pour filtres, déclencheurs, alarmes et assertions | Ces quatre besoins posent la même question ; trois moteurs séparés seraient incompatibles entre eux | Une syntaxe par usage |
| **D-12** | **Trois paliers livrables** (Observer, Comprendre, Agir) à périmètre inchangé | Un v1 de 12 à 16 mois sans usage intermédiaire se trompe longtemps sans le savoir | Un seul v1 complet ; réduction du périmètre |
| **D-13** | **Chaque exigence porte un test automatisé** ; l'interface reste une coquille fine avec tests de fumée seulement | L'automatisation de clics est fragile et coûteuse ; puisque tout est pilotable sans interface, les tests passent par là | Validation manuelle guidée ; tests limités au noyau |
| **D-14** | **L'enregistrement est prioritaire** en surcharge ; toute perte est comptée et déclarée | La trace est souvent le seul livrable d'un essai ; un outil de mesure ne mentira jamais par omission | Priorité à l'affichage ; arrêt en erreur |

---

## 11. Décisions ouvertes

À trancher au jalon indiqué, par une fiche de décision.

| # | Question ouverte | Tranchée au | Critère |
|---|---|---|---|
| **O-01** | Mécanique exacte du transport noyau ↔ clients (anneau en mémoire partagée seul, ou socket de contrôle + anneau) | Jalon A2 | Débit soutenu, latence, simplicité de la version distante |
| **O-02** | Disposition binaire exacte du journal natif (`.tcj`) et de son index | Jalon A1 | Débit d'écriture, coût de reconstruction d'index, compacité |
| **O-03** | Bibliothèque de tracé (pyqtgraph ou équivalent) | Jalon A7 | `EX-GRA-06` (préservation des extrema) et latence de zoom sous 200 ms sur une heure de trace |
| **O-04** | Mécanisme d'empaquetage par plateforme | Jalon A9 | Un seul fichier livrable par plateforme, démarrage sous 3 s, taille raisonnable |
| **O-05** | Format du jeu de test (`.tct`) : YAML, TOML ou CSV structuré | Jalon C2 | Lisibilité d'un diff Git, expressivité (valeurs de signaux, périodes), facilité de génération par script |
| **O-06** | Opportunité d'une interface web **de supervision** (état, démarrage/arrêt, récupération de trace) | Après le palier C | Besoin réel constaté ; ne remet pas en cause `EX-ARC-05` (activation explicite) |
| **O-07** | Stratégie de reprise après bus-off par défaut | Jalon C1 | Comportement attendu en véhicule autonome |

---

## 12. Traçabilité

- Chaque exigence `EX-*` DOIT être référencée par au moins un test automatisé, dont le nom
  contient l'identifiant.
- Une matrice exigence → test(s) → jalon DOIT être générée automatiquement et publiée.
- La construction DOIT échouer si une exigence marquée v1 n'a aucun test associé.
- Les exigences de performance renvoient aux chiffres publiés par le banc, avec plateforme,
  version et date.
