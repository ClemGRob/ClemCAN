# ClemCAN — Spécification

> **ClemCAN** — analyseur et éditeur CAN / CAN FD.
> Version du document : 1.3 — 2026-11-25. Licence du projet : MIT.
>
> **1.3** : arbitrage des points remontés par la revue des SRS du 2026-10-07. Pas d'émission en
> Tier 2 en v1 ; galop d'essai jugé conjointement par le noyau et la couche 2 ; violations et
> alarmes inscrites dans la trace en direct seulement ; délai de désarmement compté depuis le
> constat de la perte de connexion ; aucune invite en ligne de commande dès le palier A ; aucun
> port en écoute pour tout processus ; pont inclus dans l'exécutable autonome et lancé par le
> démon ; durée exacte des trames FD non ISO ; cas dégradés J1939 et ISO-TP ; délestage du flux
> d'affichage distinct de la décimation ; `transport_incomplete` résultat d'analyse ; fiches
> `D-30` à `D-40` ; décision ouverte `O-08`. Aucun identifiant renuméroté, aucune exigence ajoutée.
>
> **1.2** : seconde relecture de cohérence. Écoute seule exigée dès le palier A ; surcharge
> d'`EX-CAP-05` redéfinie côté consommateurs ; logique trivalente pour NA ; abonnements
> d'évaluation bornés ; galop d'essai recentré sur l'émission ; compteurs régénérés en continu au
> rejeu ; liste blanche vide par défaut ; droits d'accès locaux ; périmètre de « fonction »
> (`EX-ARC-06`) ; résolution J1939 par PGN ; machine de référence ARM64 ; fiches `D-24` à `D-29`.
> Exigences ajoutées : `EX-EXP-06`, `EX-SES-04`, `EX-SEC-04`, `EX-PRF-06`.
>
> **1.1** : relecture de cohérence. Enveloppe de performance recalculée sur la physique du bus,
> critère de vérification ajouté à chaque exigence, sémantiques d'évaluation fixées, paliers
> réalignés, fiches `D-15` à `D-23` ajoutées. Aucun identifiant existant renuméroté ;
> exigences ajoutées : `EX-BUS-07`, `EX-ROB-05`.

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
- Chaque exigence porte un **critère de vérification** (*Vérification*). Une exigence sans critère
  vérifiable est un souhait, pas une exigence.
- Chaque critère appartient à une **classe de vérification** (décision `D-23`) :
  - **auto** (par défaut, non marqué) : test automatisé exécuté en intégration continue, sans
    matériel ni serveur graphique ;
  - **⚙** : test automatisé exigeant le banc matériel de référence (§6.1), exécuté à la main ;
  - **IHM** : test de fumée de l'interface, en rendu hors écran, sans simulation de clics, qui
    vérifie la présence et le câblage d'un élément dont la logique est testée en **auto**.
- Le verbe normatif porte l'obligation : `DOIT` = obligatoire en v1 au palier indiqué.
  `DEVRAIT` = souhaitable, non bloquant. `PEUT` = optionnel. Une exigence dont l'énoncé n'a pas de
  verbe (forme nominale) se lit `DOIT`.
- Le palier de livraison (A, B ou C, cf. PLAN.md) est indiqué entre crochets. Une exigence dont
  une partie relève d'un palier ultérieur l'indique dans son texte.
- Sauf mention contraire, une mesure de performance est faite sur la **machine de référence**
  avec le **profil de charge nominal** (§6.1).

---

## 2. Glossaire

| Terme | Définition retenue dans ce document |
|---|---|
| **Canal** | Une interface CAN ouverte par ClemCAN, reliée à un bus physique ou virtuel. Un canal a un nom (`can0`), un backend, un débit, un mode. « Interface » et « bus » désignent dans ce document le matériel et le support physique ; l'unité logique est toujours le canal. |
| **Trame** | Une unité de transport sur le bus : identifiant, longueur, DLC brut, données, drapeaux, horodatage, canal, sens. |
| **Rapport d'erreur** | Information d'erreur remontée par le contrôleur ou le pilote (trame d'erreur SocketCAN, statut PCAN) : classe d'erreur, état du contrôleur, compteurs TEC/REC. Ce n'est pas du trafic utile. |
| **Événement** | Enregistrement horodaté non-trame dans une trace : perte, rapport d'erreur, changement d'état du bus, reconfiguration, déclenchement, fin abrupte, etc. Chaque événement a un type et des champs documentés. |
| **Signal** | Champ de bits à l'intérieur d'une trame, avec endianness, signe, facteur, offset, unité, table de valeurs. |
| **Non applicable (NA)** | État d'un signal qui n'a pas de valeur à un instant : avant sa première réception, ou quand la valeur du multiplexeur ne le sélectionne pas. NA n'est jamais zéro. |
| **Valeur à l'instant t** | Dernière valeur reçue du signal à un instant inférieur ou égal à *t*, ou NA (maintien en escalier). Fonction unique, partagée par toutes les fonctions de ClemCAN (décision `D-21`). |
| **Fonction** | Opération qui produit ou modifie une donnée ou un état : fichier, base, configuration ou état d'un canal, enregistrement, émission, export, rapport, résultat d'analyse. S'oppose à l'**interaction de visualisation** (zoom, défilement, curseurs, disposition, couleurs), qui ne produit rien (décision `D-27`). |
| **Alarme** | Expression nommée, dotée d'une sévérité, dont la levée (passage à vrai) et la retombée (fin de vrai) sont signalées et inscrites comme événements (`EX-EXP-06`). |
| **Profil** | Session modèle, rangée dans le répertoire des profils, ouverte par son nom pour créer une nouvelle session sans modifier le modèle (`EX-SES-04`). |
| **Abonnement d'évaluation** | Flux de trames du noyau vers la couche 2 destiné aux contrôles, déclencheurs, alarmes et assertions : jamais délesté, porté par une file bornée (décisions `D-24`, `D-39`). |
| **Base de données** | Fichier décrivant trames et signaux : DBC (éditable), SYM, KCD, ARXML (import seul). |
| **Trace** | Journal horodaté de trames et d'événements, observés ou émis, éventuellement découpé en plusieurs fichiers consécutifs. Donnée de mesure : jamais modifiée en place. |
| **Repère** | Marque nommée et commentée posée par l'utilisateur sur la base de temps. Synonymes non employés : marqueur, annotation. Un repère vit dans la session, jamais dans la trace. |
| **Jeu de test** | Document décrivant des trames à émettre, avec temporisation. Donnée fabriquée, versionnable, éditable. |
| **Session** | Espace de travail enregistré dans un fichier `.clemcan` : disposition, bases chargées, signaux tracés, filtres, règles, jeux ouverts, repères. Ce terme ne désigne rien d'autre ; on parle de « session de transport » pour J1939/ISO-TP, et de « connexion cliente » pour le lien avec le démon. |
| **Connexion cliente** | Lien entre un client (interface, ligne de commande, script) et le démon, au sens du protocole `EX-ARC-04`. |
| **Journal natif** | Format d'enregistrement propre à ClemCAN, en ajout seul, conçu pour survivre à une coupure. |
| **Fenêtre d'écriture** | Durée maximale entre la réception d'une trame et sa présence durable sur le support (synchronisation effectuée). Configurable, inscrite dans les métadonnées. |
| **Profil de charge nominal** | Trafic de référence défini en §6.1 : canaux, débits, longueur d'identifiant, DLC, contenu des données. |
| **Enveloppe nominale** | Profil de charge nominal tenu sans perte sur la machine de référence. Contractuellement garanti et mesuré. |
| **Saturé** | Canal portant des trames dos à dos, au débit maximal théorique du profil considéré (formule de `EX-VUE-04`). |
| **Décimation** | Réduction du nombre de points d'une série décodée transmis à l'affichage, préservant les extrema. Faite par la couche 2 (`EX-GRA-06`). |
| **Délestage** | Abandon, par le noyau, de trames d'un abonnement d'affichage dont le consommateur ne suit pas. Compté et signalé, sans garantie d'extrema, sans effet sur l'enregistrement (`EX-CAP-05`, `D-39`). |
| **Gigue de période** | Écart-type des intervalles entre réceptions successives d'un même identifiant, sur une fenêtre glissante (défaut 10 s). |
| **Erreur d'émission** | Écart *e = t_réel − t_demandé* entre l'instant où une trame est effectivement émise et celui demandé par le jeu de test. |
| **Écoute seule** | Mode d'un canal dans lequel ClemCAN n'écrit aucun bit sur le bus : contrôleur en mode *listen-only* (surveillance de bus, ISO 11898-1), donc sans acquittement ni drapeau d'erreur (décision `D-20`). |
| **Armement** | Action délibérée et explicite autorisant l'émission sur un canal, liée à la connexion cliente qui l'a demandée (`EX-SAF-02`). |
| **Galop d'essai** | Rejeu complet d'un jeu de test sur bus virtuel, préalable obligatoire au rejeu réel (`EX-SAF-06`). |
| **Tier** | Niveau de garantie d'un backend matériel (§5.1). |

Extensions de fichiers : journal natif `.tcj` (+ index `.tcx`), session `.clemcan`,
jeu de test `.tct`, règles de contrôle `.tcr` (YAML), assertions `.tce`.
Binaires : démon `clemcand`, ligne de commande `clemcan`, noyau `libclemcan`.

---

## 3. Acteurs et contextes d'emploi

Les quatre contextes suivants sont **tous** dans le périmètre, et chacun impose ses exigences.

| Contexte | Ce qu'il impose |
|---|---|
| **Banc / laboratoire, PC connecté** | Latence d'affichage faible, mise en œuvre rapide, sessions réutilisables. |
| **Véhicule / machine en marche, sans opérateur** | Enregistrement autonome, survie aux coupures d'alimentation brutales, rotation de fichiers, démarrage automatique, aucune interaction requise. |
| **Atelier / diagnostic terrain** | Profils prêts à l'emploi (`EX-SES-04`), garde-fous forts avant toute émission, lisibilité immédiate. |
| **Rétro-ingénierie de bus inconnu** | Analyse sans base de données, construction incrémentale du DBC depuis l'observation. |

Cas d'emploi structurants retenus, en plus des besoins initiaux :

- **UC-1 Test automatisé** — le logiciel est piloté par un script ou une chaîne d'intégration
  continue : charger une base, rejouer un scénario, évaluer des assertions, rendre un code
  retour et un rapport. Conséquence : toute fonction utile existe hors interface graphique.
- **UC-2 Enregistrement déclenché** — campagne longue dont on ne garde que les fenêtres
  autour d'événements.
- **UC-3 Analyse partagée** — poser des repères, extraire un intervalle, le transmettre avec sa
  base et ses repères.
- **UC-4 Source de vérité du firmware** — la base éditée produit le code C de sérialisation.

Déploiement : **Linux x86-64, Linux ARM64, Windows x86-64** (`EX-POR-01`). Windows ARM64 n'est
pas une cible v1.

---

## 4. Architecture imposée

L'architecture suivante est une exigence, parce qu'elle seule satisfait simultanément
l'enveloppe de performance (§6.1), l'autonomie embarquée et le pilotage à distance
(décisions `D-02`, `D-15`, `D-16`).

```
┌─────────────────────────────────────────────────────────────┐
│  Couche 3 — Interface (Qt / PySide6)                        │
│  N'affiche que du décimé. Jetable, remplaçable.             │
│  Aucune logique métier.                                     │
└────────────────────────┬────────────────────────────────────┘
                         │  appels en processus (même programme Python)
┌────────────────────────┴────────────────────────────────────┐
│  Couche 2 — Analyse (Python)                                │
│  DBC (cantools), décodage, J1939/ISO-TP, contrôles,         │
│  expressions, déclencheurs, rapports, orchestration du      │
│  rejeu. Cliente du noyau. Ne reçoit que les flux auxquels   │
│  elle s'abonne (D-16). Exécutable sans interface (CLI).     │
└────────────────────────┬────────────────────────────────────┘
                         │  protocole documenté et versionné (EX-ARC-04)
                         │  local : socket de contrôle + anneau en mémoire partagée
                         │  distant (option) : TCP, TLS 1.3 mutuel (EX-SEC-02)
┌────────────────────────┴────────────────────────────────────┐
│  Couche 1 — Noyau C (clemcand / libclemcan)                 │
│  Seule autorité sur le bus : réception, horodatage,         │
│  tampon circulaire et pré-déclenchement, écriture du        │
│  journal, filtres identifiant/masque, statistiques de       │
│  transport, ordonnanceur TX unique, comptage des pertes.    │
│  Seul code soumis au débit ligne.                           │
└──────────────┬──────────────────────────────────────────────┘
               │  interface d'injection documentée (D-17)
┌──────────────┴──────────────────────────────────────────────┐
│  Pont Tier 2 (Python, python-can) — processus optionnel     │
└─────────────────────────────────────────────────────────────┘
```

Le point d'accès réseau est **le démon** : une interface et une couche 2 exécutées sur un PC
pilotent un démon exécuté sur une cible embarquée (`D-15`). Pour un usage autonome sur la cible
(enregistrement déclenché, contrôles), la couche 2 y tourne sans interface.

- **EX-ARC-01** [A] Le noyau DOIT être un démon autonome : l'enregistrement se poursuit sans
  interruption si un client est fermé, mis à jour ou planté.
  *Vérification* : l'injecteur numérote ses trames ; `kill -9` du client pendant un
  enregistrement d'au moins 10 min au profil nominal sur bus virtuel ; la trace contient la
  séquence 1..N complète, dans l'ordre par canal, et le compteur de pertes reste à zéro.
- **EX-ARC-02** [A] Le noyau DOIT être utilisable en bibliothèque liée, sans démon, pour les
  usages en ligne de commande.
  *Vérification* : un programme de test lié à `libclemcan`, sans démon lancé, enregistre une
  trace sur bus virtuel ; elle est identique en contenu à celle du démon pour la même injection.
- **EX-ARC-03** [A] Le noyau NE DOIT contenir aucune logique métier : ni décodage de signal,
  ni base de données, ni expression, ni interprétation de protocole applicatif. Les filtres
  identifiant/masque et les statistiques de transport (`D-16`) ne sont pas de la logique métier.
  *Vérification* : analyse automatique en intégration continue des inclusions et des
  dépendances de liaison de `core/` contre une liste blanche ; tout écart casse la construction.
- **EX-ARC-04** [A] Le protocole entre noyau et clients DOIT être documenté et versionné, de
  sorte qu'un client écrit dans un autre langage puisse s'y brancher sans modifier le noyau. Il
  couvre : contrôle des canaux, abonnement aux flux, statistiques, événements, enregistrement,
  émission et armement.
  *Vérification* : tests de contrat exécutés contre le démon par un client minimal indépendant
  de la couche 2 ; un client annonçant une version majeure incompatible est refusé avec un
  message explicite.
- **EX-ARC-05** [A] Par défaut, le démon N'OUVRE AUCUN port réseau en écoute. L'accès distant
  est une option explicite, livrée au palier B selon `EX-SEC-02`.
  *Vérification* : `ss -lntp` (Linux) et `netstat -ano` (Windows) après démarrage par défaut ne
  montrent aucun socket réseau du démon.
- **EX-ARC-06** [A] Toute **fonction** (§2) accessible depuis l'interface graphique DOIT être
  accessible sans elle, par la ligne de commande (`EX-AUT-01`). Les interactions de
  visualisation en sont exclues. L'interface est une coquille fine : elle déclenche les fonctions
  par un registre de commandes unique, partagé avec la ligne de commande (`D-27`).
  *Vérification* : la suite de tests fonctionnels s'exécute sans serveur graphique ; la liste des
  commandes du registre est générée, et un test vérifie que chacune a son équivalent en ligne de
  commande ; une analyse statique vérifie que l'interface ne modifie aucune donnée hors du
  registre.

---

## 5. Exigences fonctionnelles

### 5.1 Interfaces matérielles — `EX-BUS`

Deux niveaux de garantie sont définis, et affichés à l'utilisateur (décision `D-15`).
Les garanties Tier 1 sont attachées à un **couple (pilote, matériel)** nommé, détecté à
l'exécution : SocketCAN est une API, ses capacités dépendent du pilote.

| Niveau | Couples (pilote, matériel) | Garanties |
|---|---|---|
| **Tier 1** | SocketCAN `peak_usb` + PEAK PCAN-USB / PCAN-USB FD (Linux) ; PCAN-Basic + mêmes matériels (Windows) ; bus virtuel ClemCAN (Linux et Windows, `vcan` sous Linux) | Débit du profil nominal, rapports d'erreur, détection de perte, horodatage matériel (matériels PEAK), CAN FD (PCAN-USB FD), écho d'émission horodaté |
| **Tier 2** | Tout backend python-can (Vector, Kvaser, ixxat, slcan, gs_usb, socketcand…), et tout pilote SocketCAN non listé en Tier 1, via le pont (`D-17`) | Fonctionnel en réception ; garanties de débit, d'horodatage et de détection de perte non assurées, déclarées comme telles. Aucune émission en v1 : l'armement est refusé (`D-30`) |

Le bus virtuel ne porte ni débit binaire, ni rapports d'erreur, ni horodatage matériel : il
sert à valider le débit **logiciel** et la logique, jamais la physique du bus.

Un canal Tier 2 s'ouvre par le même protocole qu'un canal Tier 1 : le démon lance alors le pont,
le surveille, et inscrit sa fin ou son plantage comme événement de la trace (`D-36`).

- **EX-BUS-01** [A] ClemCAN DOIT gérer un nombre non borné de canaux simultanés : aucune
  constante de compilation ni structure de taille fixe ne limite ce nombre. À ressources
  épuisées, l'ouverture d'un canal est refusée avec un message explicite.
  *Vérification* : test paramétré à 1, 2, 4, 8 et 32 canaux virtuels ; analyse statique
  vérifiant l'absence de constante de nombre de canaux dans `core/`.
- **EX-BUS-02** [A] Chaque canal DOIT porter son propre débit nominal, son débit de phase de
  données et ses points d'échantillonnage (FD), son mode (classique, FD ISO, FD non ISO), son mode
  de contrôleur (écoute seule, acquittement, émission, `EX-SAF-01`), son protocole (aucun ou
  J1939, `EX-DEC-06`), ses bases de données associées et ses règles de contrôle.
  *Vérification* : deux canaux virtuels configurés différemment ; les métadonnées de la trace et
  l'état exposé par le protocole restituent chaque paramètre par canal.
- **EX-BUS-03** [A] ClemCAN DOIT lire les rapports d'erreur (décodés selon le backend : classe,
  état du contrôleur, compteurs TEC/REC quand disponibles), les trames de requête distante, les
  trames étendues (29 bits), les trames FD (BRS, ESI, DLC 9 à 15) et les trames classiques de
  DLC 9 à 15 (huit octets de données, DLC brut conservé).
  *Vérification* : vecteurs injectés sur bus virtuel pour chaque type, relus à l'identique ;
  ⚙ rapports d'erreur provoqués sur PCAN (bus non terminé) et relus.
- **EX-BUS-04** [A] Le niveau (Tier 1 / Tier 2) et le couple (pilote, matériel) de chaque canal
  DOIVENT être exposés par le protocole, inscrits dans les métadonnées de chaque trace, et
  affichés dans l'interface dès qu'elle existe (palier A, jalon d'interface).
  *Vérification* : métadonnées d'une trace sur bus virtuel et sur pont Tier 2 simulé ; IHM.
- **EX-BUS-05** [A] ClemCAN DOIT avertir, avant et pendant l'enregistrement, d'une incohérence
  probable de configuration : aucune trame valide dans les 2 s suivant l'ouverture, plus de
  10 rapports d'erreur par seconde, ou passage du contrôleur en état *error-passive*. Seuils
  configurables. L'avertissement est un événement de la trace.
  *Vérification* : bus virtuel silencieux → avertissement en 2 s ± 0,2 s ; rapports d'erreur
  injectés au-delà du seuil → avertissement ; ⚙ débit volontairement erroné sur PCAN.
- **EX-BUS-06** [A] Le passage en bus-off du contrôleur local et les changements d'état
  (*error-active*, *error-warning*, *error-passive*, bus-off) DOIVENT être détectés, horodatés,
  inscrits dans la trace et signalés. En écoute seule, le contrôleur local ne peut pas passer en
  bus-off : seuls les rapports d'erreur sont enregistrés, et cette limite est documentée. Sans
  émission, un contrôleur ne peut atteindre que l'état *error-passive* (compteur REC) : la
  vérification matérielle du bus-off relève donc du palier C.
  *Vérification* : événements d'état, bus-off compris, simulés par le bus virtuel ; ⚙ [A] passage
  en *error-passive* provoqué en mode acquittement sur PCAN (court-circuit CAN_H/CAN_L sur banc) ;
  ⚙ [C] bus-off provoqué en mode émission sur PCAN.
- **EX-BUS-07** [C] Une politique de reprise après bus-off (aucune, manuelle, automatique après
  délai) DOIT être configurable par canal ; la valeur par défaut est tranchée par `O-07`.
  *Vérification* : chaque politique exercée sur bus virtuel simulant un bus-off ; ⚙ sur PCAN.

### 5.2 Capture et horodatage — `EX-CAP`

- **EX-CAP-01** [A] L'horodatage DOIT provenir de la source la plus précise disponible, dans
  l'ordre : matériel, puis noyau du système, puis espace utilisateur. La source effective DOIT
  être inscrite **par canal** dans les métadonnées de la trace.
  *Vérification* : bus virtuel → source « noyau » ; ⚙ PCAN sous SocketCAN et PCAN-Basic →
  source « matériel » ; pont Tier 2 → source déclarée par le pont.
- **EX-CAP-02** [A] Chaque trace DOIT porter une base de temps monotone (pour les durées) **et**
  une correspondance avec l'heure absolue UTC au démarrage (pour la corrélation externe).
  *Vérification* : changement de l'heure système pendant un enregistrement ; les durées
  relues restent monotones et continues, la correspondance UTC est celle du démarrage.
- **EX-CAP-03** [A] Tous les canaux d'une même trace DOIVENT être exprimés dans une base de temps
  commune. Quand les horloges matérielles de deux canaux diffèrent, ClemCAN DOIT estimer leur
  décalage et leur dérive par rapport à l'horloge monotone de l'hôte, et inscrire l'incertitude
  résultante dans les métadonnées.
  *Vérification* : ⚙ une même trame observée par deux canaux sur un même bus ; l'écart médian et
  le 99ᵉ centile sont mesurés par le banc et publiés par couple de backends ; l'écart observé
  reste dans l'incertitude déclarée.
- **EX-CAP-04** [A] **Aucune perte silencieuse.** Toute perte (débordement de tampon de socket,
  de l'anneau interne, écriture disque en retard, débordement de FIFO du contrôleur) DOIT
  produire un événement horodaté dans la trace, être signalée par le protocole, et comptée :
  - **exactement** quand la source fournit le nombre (tampon de socket via `SO_RXQ_OVFL`, anneau
    interne, disque) ;
  - comme **« perte de nombre inconnu »**, avec l'intervalle horodaté où elle s'est produite,
    quand la source ne fournit qu'un drapeau (débordement de FIFO matérielle).
  L'événement porte l'horodatage de sa détection ; une trame perdue n'a pas d'horodatage propre.
  *Vérification* : injection sur bus virtuel au-delà de l'enveloppe, avec tampon réduit ; le
  nombre déclaré égale le nombre injecté moins le nombre enregistré ; ⚙ débordement provoqué sur
  PCAN, événement « nombre inconnu » présent.
- **EX-CAP-05** [A] Quand un consommateur (affichage, décodage pour affichage, graphe, évaluation)
  ne suit pas, **l'enregistrement est prioritaire** : le noyau déleste le flux d'affichage (§2,
  `D-39`), et l'affichage, le décodage pour affichage et le graphe gèlent ou s'appauvrissent ; la
  trace reste complète. Aucun consommateur n'exerce de
  contre-pression sur le noyau. Toute évaluation abandonnée (contrôles, déclencheurs, alarmes,
  assertions) est comptée et signalée selon `D-24`. La surcharge de capture elle-même (au-delà de
  l'enveloppe nominale) relève d'`EX-CAP-04` et d'`EX-PRF-03`.
  *Vérification* : sur bus virtuel, au profil nominal, consommateurs d'affichage et d'évaluation
  ralentis artificiellement (débit traité limité à 10 % du flux) pendant 10 min ; la trace ne perd
  aucune trame, le compteur de délestage du flux d'affichage est non nul, et les évaluations
  abandonnées sont comptées avec leurs intervalles.

### 5.3 Journal natif et robustesse — `EX-JNL`

- **EX-JNL-01** [A] L'enregistrement DOIT s'écrire dans un journal en **ajout seul**, sans bloc
  compressé en cours d'écriture, synchronisé sur le support au moins une fois par fenêtre
  d'écriture (défaut 100 ms, configurable de 10 ms à 1 s). Une coupure d'alimentation ne peut
  perdre que les trames de la dernière fenêtre. À la réouverture, le lecteur détecte la fin
  abrupte, la signale par un événement, et indique la borne de perte (la fenêtre). Avec la perte
  « de nombre inconnu » d'`EX-CAP-04`, c'est l'un des deux seuls cas de perte non comptée
  exactement ; tous deux sont bornés dans le temps et signalés. Sur un support dont le
  cache d'écriture ignore la synchronisation, la borne n'est pas garantie et l'avertissement
  figure dans la documentation.
  *Vérification* : 100 coupures simulées par réinitialisation brutale d'une machine virtuelle
  (ou cible de bloc à défaillance injectée) pendant écriture ; à chaque fois, le journal s'ouvre
  sans erreur, contient toutes les trames antérieures au dernier point de synchronisation,
  signale la fin abrupte, et la perte n'excède pas la fenêtre.
- **EX-JNL-02** [A] L'index de navigation DOIT être reconstructible intégralement depuis le
  journal seul. Sa perte n'est jamais une perte de données.
  *Vérification* : suppression de l'index, reconstruction, comparaison bit à bit du contenu relu.
- **EX-JNL-03** [A] Le journal DOIT porter ses métadonnées : version de format, fenêtre
  d'écriture ; **par canal** : débits, mode, backend, couple (pilote, matériel), niveau, source
  d'horodatage, filtres de capture actifs et leur nature (matériel ou logiciel) ; version de
  ClemCAN, empreinte des bases de données et des règles de contrôle utilisées ; et, par trame,
  le sens (reçu / émis).
  *Vérification* : une trace enregistrée avec deux canaux configurés différemment ; chaque champ
  est relu par la ligne de commande (`clemcan trace info`) et comparé à la configuration.
- **EX-JNL-04** [B] Le découpage automatique en fichiers DOIT être configurable (par taille, par
  durée, par nombre de trames), avec continuité garantie de la base de temps entre fichiers.
  Si plusieurs critères sont fixés, le premier atteint déclenche le découpage.
  *Vérification* : découpage à chaque critère ; l'ensemble de fichiers relu comme une seule trace
  est identique à l'enregistrement non découpé.
- **EX-JNL-05** [A] ClemCAN DOIT ouvrir une trace de 200 millions de trames (de l'ordre d'une
  heure au profil nominal), en un ou plusieurs fichiers, et y naviguer (saut à un instant, à la
  *n*-ième occurrence d'un identifiant) en moins d'une seconde par saut au 95ᵉ centile, avec une
  mémoire résidente bornée (≤ 500 Mo) indépendante de la taille de la trace.
  *Vérification* : trace de référence générée ; 100 sauts aléatoires, cache système vidé avant
  ouverture, index présent ; mesures publiées par le banc.

### 5.4 Formats d'échange — `EX-FMT`

| Format | Lecture v1 | Écriture v1 | Variantes couvertes en lecture |
|---|---|---|---|
| Journal natif `.tcj` | oui | oui | toutes versions de format publiées |
| Vector ASC | oui | oui | horodatage absolu et relatif, base hexadécimale et décimale, lignes CAN FD |
| candump `.log` (SocketCAN) | oui | oui | trames classiques, FD (`##`), DLC classique 9–15 (`_X`), trames d'erreur |
| CSV de signaux décodés | — | oui | — |
| Vector BLF | oui | **non** (premier jalon post-v1) | objets CAN, CAN FD et erreur ; autres objets ignorés, comptés et signalés |
| PEAK TRC | oui (1.0 à 2.1) | non | 1.0, 1.1, 1.2, 1.3, 2.0, 2.1 ; TRC 3.x refusé avec message explicite (CAN XL hors périmètre) |
| ASAM MDF v4 (MF4) | oui | non | 4.0 à 4.2, journalisation de bus ASAM (trames de données, distantes, d'erreur) |

- **EX-FMT-01** [A] Tous les formats et toutes les variantes de la colonne « lecture » DOIVENT
  être lus et convertis vers le modèle de trame interne unique. Une variante non listée est
  refusée avec un message explicite, jamais lue approximativement en silence.
  *Vérification* : chaque variante listée est représentée par au moins un fichier de `corpus/`,
  lu sans erreur et comparé à son contenu attendu.
- **EX-FMT-02** [A] Un **modèle de trame intermédiaire unique** DOIT porter tous les attributs
  transportables par au moins un format : identifiant, étendu ou non, longueur, DLC brut,
  données, FD, BRS, ESI, requête distante, rapport d'erreur et ses détails, canal, sens,
  horodatage, commentaire ; ainsi que les événements (§2).
  *Vérification* : test d'aller-retour modèle → journal natif → modèle sur un vecteur exerçant
  chaque attribut.
- **EX-FMT-03** [A] Une **matrice de conformité** DOIT être publiée avec le logiciel, indiquant
  pour chaque format, en lecture et en écriture, et pour chaque attribut : conservé, approximé
  (valeur dégradée selon une règle documentée, par exemple horodatage arrondi à la résolution du
  format), ou perdu.
  *Vérification* : la matrice est générée automatiquement depuis les tests d'aller-retour ; un
  écart entre la matrice et le comportement réel casse la construction.
- **EX-FMT-04** [A] Les conversions DOIVENT être testées par aller-retour sur des fichiers de
  référence versionnés, et toute perte DOIT être soit documentée dans la matrice, soit un échec.
  *Vérification* : celle de `EX-FMT-03`.
- **EX-FMT-05** [A] L'export CSV de signaux décodés DOIT proposer explicitement le mode
  d'échantillonnage : aux instants d'arrivée (une ligne par trame, colonnes des autres signaux
  vides), rééchantillonné à pas fixe avec maintien (valeur à l'instant t), ou rééchantillonné à
  pas fixe interpolé (choix explicite, jamais par défaut). Un signal NA produit une cellule vide.
  Le format de nombre est explicite et cohérent (`EX-I18N-05`). Cet export est livré avec le
  décodage (palier A, jalon de décodage).
  *Vérification* : vecteur contenant un signal multiplexé ; chaque mode comparé à un fichier
  attendu, dans les deux conventions de nombre.
- **EX-FMT-06** [A] Une trace importée DOIT rester distinguable d'une trace enregistrée par
  ClemCAN : l'origine (format, fichier, empreinte) figure dans les métadonnées. Une trace
  importée non triée est triée de façon stable par horodatage, et le nombre de trames déplacées
  est inscrit dans les métadonnées.
  *Vérification* : import d'un fichier ASC volontairement désordonné ; ordre, stabilité et
  métadonnées vérifiés.

### 5.5 Bases de données — `EX-DB`

La lecture sémantique s'appuie sur **cantools**, qui NE DOIT PAS être réécrit. L'écriture DBC
(`D-06`, `D-19`) et l'extraction de bits par signal (`D-18`) sont propres au projet.

- **EX-DB-01** [A] DBC : lecture complète, y compris `BO_`, `SG_`, `CM_`, `BA_DEF_`, `BA_`,
  `VAL_`, `VAL_TABLE_`, `SIG_VALTYPE_`, `SG_MUL_VAL_`, `BO_TX_BU_`, `SIG_GROUP_`, `EV_`. Les
  sections que cantools refuse (`SGTYPE_`, `CAT_DEF_`, `FILTER`, `BA_DEF_SGTYPE_`…) sont
  écartées avant lecture par un filtre isolé (`D-19`), conservées pour l'écriture, et signalées.
  *Vérification* : corpus `corpus/dbc/` exerçant chaque mot-clé, y compris les sections
  refusées par cantools ; chaque élément est relu et comparé à l'attendu.
- **EX-DB-02** [A] SYM (PEAK), KCD et ARXML (AUTOSAR) : **import en lecture seule**, converti
  vers le modèle interne, accompagné d'un rapport explicite des constructions non transposables.
  *Vérification* : un fichier par format dans `corpus/`, dont un contenant des constructions non
  transposables ; décodage comparé à l'attendu, rapport comparé à la liste attendue.
- **EX-DB-03** [B] **Écriture DBC au diff minimal.** Le fichier source est conservé **en octets** ;
  seules les lignes concernées par une modification sont réécrites. Tout octet non touché est
  préservé : ordre, indentation, commentaires, sections inconnues ou refusées par cantools,
  encodage et fins de ligne. Les lignes concernées sont définies par édition :
  - modification d'un attribut de signal : la ligne `SG_` du signal, et la ligne `BA_`, `CM_` ou
    `VAL_` qui porte l'attribut modifié ;
  - renommage : toutes les lignes qui citent l'objet renommé ;
  - ajout : les nouvelles lignes, insérées après le dernier élément du même mot-clé du même
    parent (ou en fin de section) ;
  - suppression : les lignes de l'objet et les lignes qui le citent.
  *Vérification* : sur le corpus de DBC réels et synthétiques, (1) aller-retour sans édition
  identique octet pour octet ; (2) pour chaque type d'édition, le diff est limité aux lignes
  définies ci-dessus ; (3) édition puis annulation rend le fichier identique octet pour octet ;
  (4) le fichier écrit, relu par cantools, est sémantiquement égal au modèle édité.
- **EX-DB-04** [B] **Validation avant enregistrement.** Un défaut **introduit ou touché** par
  l'édition est **bloquant** ; un défaut préexistant non touché est un **avertissement**, pour
  qu'un DBC réel imparfait reste éditable (`D-22`). Règles :
  chevauchement de signaux ; signal dépassant la longueur de trame ; identifiants dupliqués ;
  multiplexage incohérent ou orphelin ; `min`/`max` incompatibles avec facteur, offset et largeur
  de bits ; signal flottant de longueur autre que 32 ou 64 bits ; noms ne respectant pas
  `[A-Za-z_][A-Za-z0-9_]*` ou dupliqués dans leur portée ; table de valeurs hors plage ; trame
  FD déclarée en requête distante ; incohérence J1939 (identifiant 11 bits dans une trame
  marquée J1939, PGN incompatible avec le champ PF, PDU1 déclarée avec adresse de destination
  fixée dans le PGN). Le DLC est vérifié contre le mode du canal au moment où la base est
  associée à un canal (avertissement), pas dans l'éditeur.
  *Vérification* : un jeu de bases volontairement fautives, une par règle, chacune détectée avec
  un message désignant l'objet exact ; un DBC à défaut préexistant, édité ailleurs, s'enregistre
  avec avertissement.
- **EX-DB-05** [B] **Matrice de bits interactive** : édition visuelle de l'implantation des
  signaux en grille de 8 bits par octet, placement et redimensionnement à la souris, gestion de
  l'ordre Motorola et Intel (bit de départ affiché selon la convention de chacun), recouvrements
  signalés en temps réel.
  *Vérification* : tests du modèle de la matrice (placement, redimensionnement, conversion
  Motorola/Intel, recouvrement) sur vecteurs de référence ; IHM pour la présence de la vue.
- **EX-DB-06** [B] **Annuler / refaire** depuis l'ouverture de la base dans l'éditeur, sans limite
  de profondeur, y compris pour les opérations de masse.
  *Vérification* : séquence aléatoire de 1 000 opérations puis 1 000 annulations ; le modèle
  et le fichier écrit sont identiques à l'état initial.
- **EX-DB-07** [B] **Édition par lot** : renommage par motif, application d'un facteur ou d'une
  unité à une sélection, déplacement de signaux entre trames, duplication d'une trame vers un
  autre émetteur.
  *Vérification* : chaque opération sur une base de test, résultat comparé à l'attendu, puis
  validée par `EX-DB-04`.
- **EX-DB-08** [B] **Comparaison et fusion sémantiques** de deux bases : trames et signaux
  ajoutés, supprimés, déplacés, redimensionnés ; facteurs, offsets, unités et tables de valeurs
  modifiés ; application sélective des écarts, avec prévisualisation.
  *Vérification* : paires de bases de référence avec liste d'écarts attendue ; application d'un
  sous-ensemble, résultat comparé à l'attendu.
- **EX-DB-09** [A] Plusieurs bases DOIVENT pouvoir être associées à un même canal. L'ordre de
  priorité est celui de la liste d'association, la première base l'emportant ; tout identifiant
  défini dans plusieurs bases est signalé explicitement.
  *Vérification* : deux bases définissant un même identifiant différemment ; le décodage suit la
  première, l'avertissement nomme l'identifiant et les deux bases ; inverser l'ordre inverse le
  décodage.
- **EX-DB-10** [B] **Génération de code C** de sérialisation et désérialisation depuis la base.
  *Vérification* : pour un échantillon de trames et de valeurs tiré avec une graine fixe et
  publiée, l'encodage puis le décodage par le code C généré rendent exactement les mêmes octets
  et les mêmes valeurs brutes que le décodage interne, et des valeurs physiques égales à 10⁻⁹ près
  en relatif (comparaison différentielle automatisée, au moins 10⁵ vecteurs par base de test).
- **EX-DB-11** [B] Recherche de signaux et de trames par nom, identifiant, nœud, unité, commentaire.
  *Vérification* : requêtes sur une base de test, résultats comparés à l'attendu.

### 5.6 Décodage — `EX-DEC`

- **EX-DEC-01** [A] Ordre d'octets Motorola (gros-boutien) **et** Intel (petit-boutien), dans une
  même trame au besoin. En Intel, le bit de départ désigne le bit de poids faible ; en Motorola,
  le bit de poids fort.
  *Vérification* : vecteurs de test dédiés, comparés à cantools.
- **EX-DEC-02** [A] Signaux signés (complément à deux, extension de signe sur la largeur du
  signal), non signés, flottants simple et double précision (`SIG_VALTYPE_`). Le facteur et
  l'offset déclarés s'appliquent aussi aux flottants.
  *Vérification* : vecteurs couvrant un signé de 3 bits, un flottant avec facteur et offset non
  triviaux ; comparaison à cantools.
- **EX-DEC-03** [A] Valeur brute (hexadécimale et décimale), valeur physique (facteur, offset,
  unité), et libellé de table de valeurs (`VAL_`) — les trois disponibles au choix, simultanément
  affichables, et sélectionnables par signal.
  *Vérification* : les trois représentations d'un même signal obtenues par la ligne de commande
  et comparées à l'attendu.
- **EX-DEC-04** [A] **Multiplexage simple** (multiplexeur et signaux multiplexés) **et
  multiplexage étendu** (`SG_MUL_VAL_`, plages de valeurs, imbrication). Si la valeur du
  multiplexeur ne sélectionne aucune définition, les signaux multiplexés concernés sont NA et
  **les autres signaux de la trame restent décodés** (`D-18`).
  *Vérification* : vecteurs de test dédiés, y compris un multiplexeur imbriqué et une valeur de
  multiplexeur non définie : signaux multiplexés NA, signaux non multiplexés décodés.
- **EX-DEC-05** [A] CAN FD jusqu'à 64 octets, avec distinction entre DLC brut et longueur réelle.
  *Vérification* : vecteurs pour chaque DLC 0 à 15, classique et FD.
- **EX-DEC-06** [A] J1939 au niveau trame, sur identifiants 29 bits seulement : décomposition en
  priorité, PGN sur 18 bits (EDP, DP, PF, PS), adresse source, et adresse de destination si
  PF ≤ 239 (PDU1). Un identifiant avec EDP = 1 n'est pas décodé comme J1939 (réservé ou
  ISO 15765-3).
  Une trame est interprétée en J1939 si et seulement si son canal est déclaré J1939 (paramètre de
  canal, `EX-BUS-02`), ou si une base associée au canal porte l'attribut `ProtocolType` égal à
  `J1939` ; sinon un identifiant 29 bits est traité comme un identifiant étendu ordinaire. Sur
  un canal J1939, la trame est rapprochée de la base d'abord par identifiant exact, puis par PGN
  (priorité, adresse source et, en PDU1, adresse de destination ignorées) ; un rapprochement par
  PGN ambigu entre plusieurs définitions est signalé.
  *Vérification* : vecteurs PDU1, PDU2, EDP = 1, et identifiant 11 bits ; même identifiant
  29 bits sur canal J1939 et non J1939 ; trame d'une adresse source absente de la base décodée par
  son PGN.
- **EX-DEC-07** [B] **Réassemblage du transport J1939** (SAE J1939-21) : diffusion BAM et mode
  connexion RTS/CTS (`TP.CM` / `TP.DT`), jusqu'à 1 785 octets, sessions de transport concurrentes
  indexées par couple d'adresses. Temporisations par défaut de la norme, configurables :
  T1 = 750 ms, T2 = T3 = 1 250 ms, T4 = 1 050 ms, Tr = 200 ms, Th = 500 ms. Toute session
  incomplète (trame manquante, temporisation dépassée, abandon, annonce de plus de 1 785 octets)
  produit un résultat d'analyse `transport_incomplete`, horodaté, compté, portant son motif,
  visible dans la chronologie et jamais écrit dans la trace (`D-40`) ; la charge
  partielle n'est jamais présentée comme un message complet ; la mémoire de la session est
  libérée. Dans les cas dégradés, le réassemblage se comporte comme le nœud récepteur (`D-38`) :
  - une `TP.DT` dont le numéro a déjà été reçu est ignorée et comptée, sauf retransmission
    demandée par une CTS, qui remplace la donnée ;
  - une nouvelle annonce (BAM ou RTS) pour un couple d'adresses en session clôt la session
    ouverte comme incomplète (motif « remplacée ») et ouvre la nouvelle ;
  - chaque attente applique la temporisation du nœud qui attend : en BAM, T1 entre deux trames ;
    en RTS/CTS, T3 pour une CTS ou l'accusé de fin, T2 pour la première `TP.DT` après une CTS,
    T1 entre deux `TP.DT`, T4 après une CTS de maintien. Tr et Th ne closent aucune session.
  *Vérification* : vecteurs couvrant le cas nominal, la trame manquante, la trame dupliquée
  (ignorée) et la trame retransmise sur CTS (acceptée), le dépassement de chaque temporisation à
  son point d'attente, une nouvelle annonce pendant une session, deux sessions simultanées, un
  abandon explicite, une annonce hors limite ; mémoire revenue à l'état initial après
  10⁴ sessions abandonnées.
- **EX-DEC-08** [B] **J1939 applicatif** : DM1 et DM2 (SPN, FMI, compteur d'occurrences, état des
  lampes, bit de méthode de conversion SPN), réclamation d'adresse et champ NAME (J1939-81),
  requête de PGN.
  *Vérification* : vecteurs de référence pour chaque message, dont un DM1 multi-trames.
- **EX-DEC-09** [B] **ISO-TP (ISO 15765-2) sur trames de 8 octets** : trame unique, première
  trame, trames consécutives (numéro de séquence débutant à 1, modulo 16), contrôle de flux
  (continuer, attendre, débordement) ; adressage normal, normal fixe (29 bits), étendu et mixte ;
  identifiants 11 et 29 bits ; taille de bloc et temps de séparation (STmin 0x00–0x7F = 0 à
  127 ms, 0xF1–0xF9 = 100 à 900 µs, autres valeurs traitées comme 127 ms). Temporisations
  N_Bs et N_Cr par défaut 1 000 ms, configurables. L'appariement requête/réponse est configuré
  par paire d'identifiants. Une transmission incomplète suit la règle de `EX-DEC-07`
  (résultat `transport_incomplete`). ISO-TP sur trames FD de plus de 8 octets est hors v1 (§8).
  Dans les cas dégradés, le réassemblage se comporte comme le récepteur de la norme (`D-38`) :
  - un numéro de séquence différent de l'attendu (erroné ou répété) clôt la transmission comme
    incomplète (motif « numéro de séquence erroné ») ;
  - une trame unique ou une première trame reçue pendant une réception en cours clôt celle-ci
    comme incomplète (motif « interrompue ») et ouvre la nouvelle ; une trame consécutive reçue
    hors de toute réception est ignorée et comptée ;
  - un contrôle de flux « débordement » clôt la transmission comme incomplète (motif
    « débordement »).
  *Vérification* : vecteurs couvrant chaque type de trame, chaque adressage, le repli du numéro
  de séquence, l'attente, le débordement, la trame consécutive manquante et la temporisation, un
  numéro de séquence répété, un numéro erroné, une première trame pendant une réception, une
  trame consécutive isolée.
- **EX-DEC-10** [B] **UDS en lecture** (ISO 14229-1) : identification de tous les services et
  codes de refus de la norme, y compris l'appariement d'une réponse finale après un ou plusieurs
  refus 0x78 (réponse en attente) ; décodage de `0x22` (numéro d'identifiant et données brutes,
  sans description d'identifiant) et de `0x19` sous-fonctions 0x01, 0x02 et 0x0A (codes défaut et
  octet d'état). L'**émission** de requêtes UDS est hors périmètre v1 (cf. §8).
  *Vérification* : vecteurs de référence pour chaque service décodé, et une séquence 0x78.
- **EX-DEC-11** [A] Le décodage DOIT être **paresseux** et hors de la boucle de réception du
  noyau : seuls sont décodés les signaux requis par un affichage, un export ou une expression
  active (filtre, déclencheur, alarme, contrôle, assertion).
  *Vérification* : au profil nominal, sans aucun signal requis, le temps processeur de décodage
  est inférieur à 1 % d'un cœur ; avec 1, 10 et 100 signaux requis, il croît linéairement
  (coefficient de détermination R² ≥ 0,95), mesuré par le banc.
- **EX-DEC-12** [A] Un identifiant absent des bases DOIT rester visible et exploitable en brut,
  jamais masqué.
  *Vérification* : trace mêlant identifiants connus et inconnus ; tous figurent dans les vues et
  les exports bruts.

### 5.7 Contrôles de cohérence, compteurs et CRC — `EX-CHK`

Le DBC ne sait pas décrire un compteur ni un CRC : ces règles vivent dans un fichier annexe.

- **EX-CHK-01** [B] Un fichier de règles `.tcr` lisible et versionnable DOIT décrire, par trame :
  le signal compteur (modulo, pas, tolérance), le signal de contrôle, l'algorithme, l'étendue
  d'octets couverte, et le traitement des octets exclus. Algorithmes :
  - profils AUTOSAR E2E P01 (modes d'identifiant de données *both*, *alt*, *low*, *nibble*), P02
    (liste de 16 identifiants de données), P04, P05 et P11, selon la spécification AUTOSAR du
    protocole E2E, avec leur **identifiant de données** (non transmis, inclus dans le calcul) ;
  - CRC paramétrique de 8, 16, 32 ou 64 bits (polynôme, valeur initiale, ou-exclusif final,
    réflexions), avec catalogue nommé : CRC-8/SAE-J1850, CRC-8/AUTOSAR, CRC-16/CCITT-FALSE,
    CRC-32, CRC-32/AUTOSAR, CRC-64/ECMA-182.
  Les trames protégées par authentification (AUTOSAR SecOC) sont hors périmètre : elles ne
  peuvent pas être recalculées.
  *Vérification* : chaque algorithme du catalogue rend sa valeur de contrôle de référence
  (« check » sur `123456789`) ; chaque profil E2E valide ses vecteurs de la spécification.
- **EX-CHK-02** [B] ClemCAN DOIT proposer une **détection assistée** des compteurs et des contrôles
  par convention de nommage et par vérification statistique sur une trace d'au moins 200 trames
  de l'identifiant ; un compteur est proposé si au moins 99 % des transitions respectent un pas
  constant modulo *M*, *M* étant soit une puissance de deux de 2 à 2¹⁶, soit la valeur maximale
  observée plus un (compteurs E2E P01 et P11, de 0 à 14) ; un contrôle est proposé si un algorithme du catalogue
  (identifiant de données inclus, recherché exhaustivement pour P01 et P02) valide au moins 99 %
  des trames. L'utilisateur valide la proposition ; la règle retenue est écrite dans le fichier
  `.tcr` : le résultat est reproductible.
  *Vérification* : traces synthétiques à compteur et contrôle connus, avec et sans bruit ; la
  règle proposée est la règle d'origine, y compris pour un compteur E2E P01 de 0 à 14 ; aucune
  proposition sur des octets aléatoires.
- **EX-CHK-03** [B] Les violations détectées DOIVENT être : compteur sauté, compteur dupliqué,
  compteur figé, contrôle invalide. Les deux dernières catégories de compteur sont exclusives :
  une valeur répétée sur moins de *N* trames consécutives (*N* configurable, au moins 3, défaut 3)
  est un **dupliqué** (une violation par répétition) ; à partir de *N* trames, la série entière
  est **une seule** violation **figé**, horodatée au début, close à la reprise, portant le nombre
  de trames, et ses répétitions ne sont pas comptées comme dupliquées. Chaque violation est
  horodatée, comptée, visible dans la chronologie et exploitable comme condition (cf. `EX-EXP`) ;
  pendant un enregistrement, elle est aussi inscrite dans la trace comme événement. Sur une trace
  déjà enregistrée, elle reste un résultat d'analyse, jamais écrit dans la trace (`D-32`). Un saut couvrant une perte déclarée (`EX-CAP-04`) ou une évaluation abandonnée
  (`D-24`) est classé « non évaluable », pas violation.
  *Vérification* : vecteurs contenant chaque violation à des instants connus, une répétition de
  longueur *N* − 1 puis *N*, et un saut coïncidant avec une perte déclarée ; en direct, violations
  présentes dans la trace ; sur une trace déjà enregistrée, violations présentes dans les
  résultats et empreinte de la trace inchangée.
- **EX-CHK-04** [B] Contrôles de cohérence supplémentaires : franchissement des bornes physiques ;
  temps de cycle hors tolérance (trop lent, trop rapide, gigue de période), la référence venant du
  `.tcr`, à défaut de l'attribut DBC `GenMsgCycleTime` avec une tolérance par défaut de ± 20 %,
  à défaut le contrôle est inactif ; longueur de trame inattendue ; valeur absente de la table
  `VAL_` ; identifiant inconnu des bases ; émetteur inattendu, pour J1939 seulement (adresse
  source).
  *Vérification* : un vecteur par contrôle, avec la violation à un instant connu.
- **EX-CHK-05** [C] **Au rejeu, compteurs et contrôles DOIVENT être régénérés**, sinon le système
  cible rejette les trames sans diagnostic (`D-29`). Pour chaque identifiant couvert par une
  règle, le compteur émis forme une suite continue selon la règle (pas, modulo), partant de la
  valeur de la première trame émise ; il continue d'une boucle à l'autre, et ignore les trames
  écartées par la sélection (`EX-REJ-03`). Le contrôle est recalculé sur chaque trame émise. Une
  trame dont la séquence et le contenu coïncident avec l'original est donc émise avec ses octets
  d'origine. Une règle est **non recalculable** si l'un de ses paramètres est inconnu (identifiant
  de données E2E non résolu, algorithme non identifié) : rejouer un jeu contenant une trame
  couverte par une telle règle est refusé avec un message explicite, sauf émission explicitement
  demandée en octets bruts inchangés.
  *Vérification* : un jeu de test dérivé d'une trace protégée (CRC et E2E), modifié sur un signal,
  filtré sur une partie de ses trames, et rejoué deux fois en boucle sur bus virtuel ; toutes les
  trames émises passent la validation de compteur et de contrôle ; un jeu non modifié et non
  filtré, issu d'une trace sans violation, est émis octet pour octet ; une règle non recalculable provoque le refus.

### 5.8 Vues — `EX-VUE`

Les vues sont des coquilles (couche 3) sur des modèles de la couche 2 ; la logique se vérifie sur
les modèles, la vue par un test de fumée.

- **EX-VUE-01** [A] **Trace déroulante** : flux chronologique brut, défilement continu,
  colonnes configurables, mise en pause sans perte d'enregistrement, dépliage des signaux d'une
  trame sélectionnée.
  *Vérification* : le modèle de la vue, alimenté au profil nominal, reste cohérent avec la trace ;
  pause et reprise sans perte d'enregistrement ; IHM.
- **EX-VUE-02** [A] **Vue groupée par identifiant** : une ligne par identifiant, mise à jour sur
  place, avec période mesurée (moyenne, minimum, maximum, gigue de période), compteur
  d'occurrences, date de dernière réception, et mise en évidence des octets qui ont changé depuis
  la réception précédente. Les statistiques par identifiant sont calculées par le noyau (`D-16`).
  *Vérification* : injection de périodes connues sur bus virtuel ; valeurs exposées par le
  protocole comparées aux valeurs théoriques à 1 % près ; IHM.
- **EX-VUE-03** [A] **Fenêtre de valeurs** : tableau des signaux choisis, valeur courante au
  format choisi (brut, physique, libellé), âge de la dernière mise à jour, indication de
  franchissement de bornes, regroupement libre.
  *Vérification* : tests du modèle sur vecteurs ; IHM.
- **EX-VUE-04** [A] **Statistiques et santé du bus** : charge en pourcentage, trames par seconde,
  histogramme des temps de cycle par identifiant, compteurs de rapports d'erreur, changements
  d'état et passages en bus-off, identifiants apparus ou disparus (absents depuis plus de
  3 × leur période mesurée, et au moins 1 s), trames perdues.
  La **charge** est la fraction du temps de bus occupée par les trames observées, sur une fenêtre
  glissante de 1 s. La durée d'une trame est calculée exactement depuis son contenu, selon
  ISO 11898-1 : bits de la trame, bourrage dynamique, bits de bourrage fixes et compteur de
  bourrage en FD ISO, 3 bits d'intertrame ; en FD, phase nominale et phase de données à leurs
  débits respectifs, la commutation ayant lieu aux points d'échantillonnage de BRS et du
  délimiteur de CRC. En FD non ISO, la durée se calcule de même selon la spécification CAN FD 1.0
  de Bosch (2012), qui ne porte pas de compteur de bourrage (`D-37`). Les trames d'erreur et de surcharge, non observables comme trames, ne sont
  pas comptées : la charge peut être sous-estimée en présence d'erreurs, et l'interface l'indique.
  Sur trace importée sans débit, la charge n'est pas affichée.
  *Vérification* : la durée calculée pour un corpus de trames classiques, FD ISO et FD non ISO est
  comparée à des vecteurs de référence établis indépendamment (trames encodées bit à bit) ; écart
  nul en bits. ⚙ sur PCAN, à
  débit injecté connu, la charge calculée s'écarte de moins de 2 % de la valeur théorique.
- **EX-VUE-05** [B] **Analyse sans base de données** : bits qui basculent par identifiant, octets
  qui changent, détection de compteurs probables et d'octets de contrôle probables (règles de
  `EX-CHK-02`), corrélation entre un intervalle de temps et les changements observés, et
  **création d'un signal candidat directement depuis une sélection de bits**, injecté dans la
  base en cours d'édition.
  *Vérification* : trace synthétique à structure connue ; compteurs et contrôles proposés ; un
  signal créé depuis une sélection de bits décode comme le signal d'origine.
- **EX-VUE-06** [A] Toute vue DOIT rester réactive au profil nominal : au 99ᵉ centile, moins de
  100 ms entre l'horodatage de réception d'une trame et sa remise au modèle d'affichage ; la
  boucle d'événements de l'interface n'est jamais bloquée plus de 200 ms. Une opération longue
  (ouverture de trace, chargement de base) s'exécute hors du fil principal avec indication de
  progression.
  *Vérification* : instrumentation de la latence et sonde de la boucle d'événements, en rendu hors
  écran sur la machine de référence, au profil nominal sur bus virtuel, pendant 10 min.
- **EX-VUE-07** [A] Les vues DOIVENT être disposables librement (fenêtres ancrables, plusieurs
  écrans) et leur disposition DOIT être enregistrée dans la session.
  *Vérification* : la disposition enregistrée puis restituée est identique (comparaison de
  l'état sérialisé) ; IHM.

### 5.9 Graphe — `EX-GRA`

- **EX-GRA-01** [A] Modes de tracé par signal : **escalier** (valeur à l'instant t, **par
  défaut**), ligne interpolée, points nus. Les instants d'échantillonnage réels DOIVENT pouvoir
  être affichés en surimpression pour ne jamais confondre mesure et maintien.
  *Justification* : un signal CAN n'existe qu'aux instants d'arrivée de sa trame ; interpoler
  invente des valeurs, ce qui est faux pour un compteur et absurde pour une énumération.
  *Vérification* : la série produite pour chaque mode, sur un vecteur, est comparée à l'attendu.
- **EX-GRA-02** [A] Le choix de mode par signal DOIT être mémorisé dans la session. Ce choix est
  un choix d'**affichage** : les assertions, la comparaison de traces, les contrôles et les
  déclencheurs évaluent toujours la valeur à l'instant t (escalier, NA compris), quel que soit le
  mode affiché ; l'export applique le mode choisi dans sa propre commande (`EX-FMT-05`). Toutes
  ces fonctions utilisent la même implémentation de la valeur à l'instant t (`D-21`).
  *Vérification* : un même signal affiché en mode interpolé ; une assertion et une comparaison
  donnent le même résultat qu'en mode escalier ; test de l'unicité de l'implémentation.
- **EX-GRA-03** [A] Plusieurs axes verticaux, échelles automatiques ou fixées, courbes empilées
  ou superposées, couleurs mémorisées par signal.
  *Vérification* : état du modèle de graphe enregistré et restitué ; IHM.
- **EX-GRA-04** [A] Curseurs de mesure (au moins deux) avec affichage des valeurs à la position
  et des écarts Δt et Δy.
  *Vérification* : valeurs et écarts calculés par le modèle pour des positions données,
  comparés à l'attendu.
- **EX-GRA-05** [A] Synchronisation bidirectionnelle entre graphe, trace et repères : cliquer un
  point sélectionne la trame correspondante (en escalier, entre deux points, la trame
  précédente), et inversement.
  *Vérification* : le modèle de sélection, sollicité par un instant, désigne la trame attendue,
  et inversement ; IHM.
- **EX-GRA-06** [A] La décimation d'affichage DOIT préserver les extrema : une impulsion d'une
  seule trame reste visible à toute échelle de zoom. En direct, une trame délestée par le noyau
  (`EX-CAP-05`) peut manquer à l'affichage ; le délestage est signalé, et la série relue depuis
  la trace la contient (`D-39`).
  *Vérification* : trace contenant une impulsion isolée de 1 ms dans une heure de données ; la
  série décimée pour une largeur de 1 920 pixels couvrant l'heure entière contient la valeur
  extrême de l'impulsion (test sur la couche de décimation, sans interface).
- **EX-GRA-07** [A] Un signal NA à un instant donné DOIT présenter une interruption de tracé, non
  une valeur maintenue.
  *Vérification* : la série produite pour un signal multiplexé contient une rupture sur chaque
  intervalle NA.
- **EX-GRA-08** [B] Signaux énumérés tracés avec leurs libellés en ordonnée.
  *Vérification* : les graduations produites par le modèle pour un signal à table `VAL_` sont
  ses libellés ; IHM.
- **EX-GRA-09** [B] Export du graphe en image (PNG et SVG), et export des séries affichées en CSV
  selon `EX-FMT-05`.
  *Vérification* : export hors écran d'un graphe de référence ; fichier image valide aux
  dimensions demandées ; CSV comparé à l'attendu.

### 5.10 Langage d'expression unifié — `EX-EXP`

- **EX-EXP-01** [B] Un **unique langage d'expression** DOIT servir aux filtres d'affichage, aux
  filtres de capture, aux conditions de déclenchement, aux alarmes (`EX-EXP-06`) et aux
  assertions de test.
  Un **filtre de capture** n'accepte que le sous-ensemble traduisible en listes d'identifiants et
  de masques par canal (canal, identifiant, masque, étendu, FD), appliqué par le noyau (`D-16`) ;
  une expression de capture hors de ce sous-ensemble est refusée à la saisie (`EX-EXP-04`).
  *Vérification* : un même texte d'expression accepté par les cinq usages donne le même
  résultat ; une expression de capture sur un signal est refusée avec un message explicite.
- **EX-EXP-02** [B] Il DOIT permettre de désigner : un canal, un identifiant, un signal par nom
  (qualifié par canal et par base en cas d'ambiguïté ; un nom ambigu non qualifié est une erreur
  de saisie), une valeur physique, brute ou de libellé, la longueur, le sens, les drapeaux de
  trame, et les résultats de contrôle (`crc_invalid`, `counter_skip`…).
  *Vérification* : un vecteur par désignation ; un nom ambigu est refusé.
- **EX-EXP-03** [B] Il DOIT gérer les opérateurs de comparaison et logiques, les changements
  d'état et le **temps**, avec la sémantique suivante :
  - une expression est évaluée à chaque trame pertinente reçue et à chaque échéance de
    minuterie (résolution 1 ms), sur la base de temps de la trace, jamais sur l'horloge murale de
    l'évaluation ; un signal y vaut sa valeur à l'instant t ;
  - **logique à trois valeurs** (`D-25`) : une expression vaut vrai, faux ou NA ; toute
    comparaison ou opération arithmétique impliquant NA vaut NA ; `!NA` vaut NA ; `&&` et `||`
    suivent la logique de Kleene (`faux && NA` = faux, `vrai || NA` = vrai, sinon NA) ;
    `isna(e)` vaut vrai ou faux, jamais NA ;
  - une condition (filtre, déclencheur, alarme) n'est **satisfaite que si elle vaut vrai** ;
    NA ne la satisfait pas ;
  - `rise(c)` : passage de `c` de faux à vrai ; `fall(c)` : passage de vrai à faux ; un passage
    depuis ou vers NA n'est pas un front ;
  - `held(c, d)` : `c` vaut vrai sans interruption depuis au moins la durée `d` ;
  - `absent(id, d)` : aucune trame `id` depuis au moins la durée `d`, comptée depuis la dernière
    trame `id` ou, à défaut, depuis l'ouverture du canal ou le début de la trace.
  La grammaire est publiée en EBNF.
  *Exemple* : `can0.EngineSpeed > 3000 && Gear == "R"` ; `absent(0x120, 500ms)` ;
  `held(crc_invalid, 50ms)`.
  *Vérification* : vecteurs couvrant chaque opérateur, la table de vérité complète des trois
  valeurs (dont `!(s > 3)` avec `s` NA, qui ne satisfait pas la condition), deux signaux de
  trames différentes, et les fonctions temporelles à des instants connus à 1 ms près ; la grammaire publiée est celle
  de l'analyseur (générée ou testée contre lui).
- **EX-EXP-04** [B] Une expression invalide DOIT être refusée à la saisie avec la position et la
  cause de l'erreur, jamais silencieusement ignorée.
  *Vérification* : corpus d'expressions invalides, chacune refusée avec position et cause
  attendues.
- **EX-EXP-05** [B] Quand le backend le permet, les filtres de capture DOIVENT être appliqués au
  niveau matériel ou noyau du système, sinon par le noyau ClemCAN. La nature effective (matériel,
  système, ClemCAN) DOIT être visible et inscrite dans les métadonnées de la trace
  (`EX-JNL-03`), car un filtre modifie ce qui est enregistré.
  *Vérification* : un filtre sur bus virtuel ; trames hors filtre absentes de la trace,
  métadonnées conformes ; ⚙ filtre matériel sur PCAN.
- **EX-EXP-06** [B] Une **alarme** (§2) DOIT être définie par un nom, une expression `EX-EXP` et
  une sévérité (information, avertissement, erreur). Elle est **levée** quand son expression
  devient vraie et **retombe** quand elle cesse de l'être (vers faux ou NA) ; un maintien minimal
  s'exprime par `held`. Levée et retombée sont des événements horodatés, exposés par le protocole,
  affichés dans l'interface (état courant et historique) et listés par la ligne de commande ;
  pendant un enregistrement, ils sont inscrits dans la trace ; sur une trace déjà enregistrée, ce
  sont des résultats d'analyse, jamais écrits dans la trace (`D-32`). Les alarmes sont enregistrées dans la session. Aucune alarme n'agit sur le
  bus.
  *Vérification* : scénario injecté sur bus virtuel ; levées et retombées aux instants attendus à
  1 ms près, dont une retombée par passage à NA ; événements présents dans la trace et dans la
  sortie de la ligne de commande ; mêmes levées et retombées obtenues sur la trace une fois
  enregistrée, empreinte de la trace inchangée ; IHM pour l'affichage.

### 5.11 Enregistrement déclenché — `EX-TRG`

Le noyau conserve le tampon pré-déclenchement ; la couche 2 évalue la condition et commande la
fenêtre. L'**instant de déclenchement** est l'instant, sur la base de temps de la trace, où la
condition devient vraie : horodatage de la trame qui la rend vraie, ou échéance de minuterie
calculée (`absent`, `held`) ; jamais l'instant de la commande. La latence d'évaluation est donc
sans effet tant qu'elle reste inférieure à la durée pré-déclenchement, sinon le retard est un
événement compté (`EX-CAP-05`).

- **EX-TRG-01** [B] Tampon circulaire **pré-déclenchement** de durée ou de taille configurable
  (si les deux sont fixés, la plus restrictive s'applique) : au déclenchement, les trames
  antérieures sont conservées.
  *Vérification* : celle de `EX-TRG-04`, avec chaque type de limite.
- **EX-TRG-02** [B] Conditions de déclenchement exprimées dans le langage `EX-EXP`, y compris sur
  rapport d'erreur, contrôle invalide, absence de trame et franchissement de borne.
  *Vérification* : un scénario par type de condition, déclenchement à l'instant attendu.
- **EX-TRG-03** [B] Durée post-déclenchement configurable, déclenchements multiples, et un fichier
  par fenêtre. Un déclenchement survenant pendant la fenêtre post-déclenchement d'un précédent
  prolonge la fenêtre courante, dans le même fichier, et y est inscrit comme second événement.
  Une fenêtre a une durée maximale configurable (défaut 10 min) : atteinte, le fichier est clos
  et la fenêtre se poursuit dans un nouveau fichier, base de temps continue, la coupure étant un
  événement des deux fichiers.
  *Vérification* : deux déclenchements disjoints → deux fichiers ; deux déclenchements
  rapprochés → un fichier prolongé portant deux événements ; déclenchements répétés au-delà de la
  durée maximale → fichiers consécutifs, sans trame perdue ni dupliquée à la coupure.
- **EX-TRG-04** [B] Le déclenchement DOIT être inscrit dans la trace comme un événement daté
  portant l'expression qui l'a produit.
  *Vérification* : scénario injecté avec une violation à l'instant *t* ; le fichier produit
  contient exactement les trames de [*t* − pré ; *t* + post], bornes à une trame près, *pré*
  étant la durée effectivement retenue par le tampon (`EX-TRG-01`), et l'événement de
  déclenchement ; même scénario avec un déclenchement par `absent`, *t* étant l'échéance.

### 5.12 Repères, extraction, comparaison, rapport — `EX-ANA`

- **EX-ANA-01** [B] Poser des repères nommés et commentés sur la base de temps, les parcourir, et
  les conserver dans la session (jamais dans la trace d'origine, qui reste inaltérée).
  *Vérification* : repères posés, session enregistrée et rouverte ; repères restitués ; empreinte
  de la trace inchangée.
- **EX-ANA-02** [B] Sélectionner un intervalle et l'exporter seul, en un nouveau fichier, avec
  option d'inclure les bases de données, les règles employées et les repères de l'intervalle
  (dans une session jointe), pour transmission à un tiers.
  *Vérification* : la sous-trace exportée, réouverte dans un environnement neuf (conteneur sans
  configuration ni cache), décode exactement comme l'originale sur l'intervalle et restitue ses
  repères.
- **EX-ANA-03** [B] **Comparaison de deux traces**, alignées sur la base de temps ou sur un
  événement de référence : identifiants présents dans l'une seulement ; temps de cycle modifiés
  (écart de période moyenne supérieur à un seuil, défaut 10 %) ; trajectoires de signaux
  divergeant au-delà d'un seuil absolu pendant au moins une durée minimale (configurables) ;
  violations apparues ou disparues.
  *Vérification* : paire de traces synthétiques à écarts connus ; liste d'écarts comparée à
  l'attendu.
- **EX-ANA-04** [B] **Rapport exportable** en HTML (PDF : `DEVRAIT`) : intervalle, graphes,
  repères, statistiques, violations, versions et empreintes des bases employées, et résultats
  d'assertions quand elles existent (palier C). Le rapport DOIT être reproductible depuis la
  ligne de commande : deux générations identiques ne diffèrent que par la date de génération.
  *Vérification* : génération en ligne de commande, deux fois ; comparaison après masquage de la
  date.

### 5.13 Jeu de test et rejeu — `EX-REJ`

- **EX-REJ-01** [C] Le **jeu de test** est un document propre, lisible, versionnable, contenant
  une liste ordonnée d'entrées : canal, trame exprimée **soit en octets bruts soit en valeurs de
  signaux**, et temporisation (instant relatif au début du rejeu, délai depuis l'entrée
  précédente, ou période avec nombre de répétitions). Une entrée périodique s'exécute en parallèle
  des entrées suivantes : son échéancier est indépendant. Le délai d'une entrée qui suit une
  entrée périodique court depuis la **première** émission de celle-ci. Le rejeu se termine quand
  toutes les entrées, répétitions périodiques comprises, ont été émises.
  *Vérification* : un jeu exerçant chaque type de temporisation, dont une entrée relative placée
  après une entrée périodique, rejoué sur bus virtuel ; instants d'émission et instant de fin
  comparés à l'attendu.
- **EX-REJ-02** [C] Un jeu de test DOIT pouvoir être créé par sélection d'un intervalle de trace,
  puis édité dans un tableau (ajout, suppression, réordonnancement, duplication, modification de
  valeurs de signaux par libellé).
  *Vérification* : trace → jeu → rejeu sur bus virtuel → trace ; même séquence (canal,
  identifiant, données) et instants relatifs à 1 ms près ; opérations d'édition testées sur le
  modèle.
- **EX-REJ-03** [C] Le rejeu DOIT permettre de choisir les trames à émettre (par identifiant, par
  expression, par sélection), un facteur d'échelle de temps (0,1 à 10), une répétition en boucle
  (les compteurs continuent leur progression d'une boucle à l'autre, `EX-CHK-05`), et un
  remappage de canal. Avant émission, la charge exigée par le jeu (formule d'`EX-VUE-04`, au
  débit configuré du canal cible) est calculée sur des fenêtres glissantes de 100 ms ; si elle
  dépasse 100 % sur l'une d'elles, le rejeu est refusé avec l'instant fautif, sauf option
  explicite : les trames sont alors émises dans l'ordre, au plus tôt, et le rapport de fidélité
  (`EX-REJ-05`) en porte l'erreur.
  *Vérification* : chaque option exercée sur bus virtuel, séquence émise comparée à l'attendu ;
  un jeu accéléré au-delà de la capacité du bus refusé, puis accepté avec l'option, ordre
  préservé.
- **EX-REJ-04** [C] **Fidélité temporelle.** L'ordonnancement se fait sur l'horloge monotone de
  l'hôte, en instants absolus, jamais par accumulation de délais. Sur un canal Tier 1 sous Linux,
  à 1 Mbit/s sans autre trafic, jusqu'à 5 000 trames par seconde, l'erreur d'émission mesurée par
  l'horodatage noyau de l'écho d'émission respecte : |*e*| < 1 ms au 99ᵉ centile, et
  |*e*(fin) − *e*(début)| < 1 ms sur une heure. Sous Windows, les mêmes grandeurs sont mesurées et
  publiées, sans seuil contractuel (cadencement USB).
  *Vérification* : ⚙ rejeu d'une heure sur PCAN-USB sous SocketCAN ; erreurs calculées par le
  rapport de fidélité ; bus virtuel avec les mêmes seuils sur l'écho, sur la machine de référence
  (banc nocturne, `EX-PRF-05`). En intégration continue, sur machines partagées, le rejeu sur bus
  virtuel vérifie la séquence et l'absence d'accumulation de délais (dérive), sans seuil de
  gigue.
- **EX-REJ-05** [C] Chaque rejeu DOIT produire un **rapport de fidélité** : médiane, 99ᵉ centile
  et pire cas de l'erreur d'émission, nombre d'émissions échouées (refusées par le backend, ou
  sans écho dans le délai configuré quand le backend fournit l'écho), et source de l'instant réel.
  *Un rejeu dont la fidélité n'est pas mesurée n'est pas un test reproductible.*
  *Vérification* : rejeu sur bus virtuel avec émissions refusées injectées ; rapport comparé à
  l'attendu.
- **EX-REJ-06** [C] Le rejeu DOIT enregistrer une **nouvelle trace corrélée** contenant le trafic
  réellement émis **et** le trafic reçu pendant le rejeu, chaque trame marquée par son sens, dans
  une base de temps unique. L'instant d'une trame émise est celui de son écho horodaté quand le
  backend le fournit, sinon l'instant de soumission ; la source est déclarée dans les métadonnées.
  *Vérification* : rejeu sur bus virtuel avec un nœud répondeur simulé ; trace corrélée
  contenant les deux sens dans l'ordre attendu.
- **EX-REJ-07** [C] Les compteurs et contrôles DOIVENT être recalculés conformément à `EX-CHK-05`.
  *Vérification* : celle de `EX-CHK-05`.
- **EX-REJ-08** [C] Un rejeu DOIT être exécutable sur bus virtuel, sans matériel, avec le même
  résultat fonctionnel : même séquence (canal, identifiant, données) dans le même ordre.
  *Vérification* : un même jeu rejoué sur bus virtuel et ⚙ sur PCAN ; séquences identiques.

### 5.14 Garde-fous d'émission — `EX-SAF`

- **EX-SAF-01** [A] ClemCAN DOIT ouvrir tout canal **en écoute seule** (§2), sans exception, dès
  le premier palier : c'est la seule protection du bus tant que l'émission n'existe pas, un
  contrôleur en mode normal écrivant déjà des acquittements et des drapeaux d'erreur. Si le
  matériel n'offre pas le mode *listen-only*, le canal ne s'ouvre qu'après confirmation explicite,
  et le fait est signalé et inscrit dans les métadonnées. Un mode **acquittement** (contrôleur en
  mode normal, aucune émission applicative, utile au banc avec un seul calculateur) est une option
  explicite par canal, signalée, inscrite dans les métadonnées, distincte de l'armement et jamais
  restaurée depuis une session ou un profil sans confirmation.
  *Vérification* : au démarrage, l'état de chaque canal exposé par le protocole est « écoute
  seule » ; aucun test n'émet sans armement ; ⚙ aucun bit émis sur le bus (contrôle par un second
  nœud qui ne reçoit aucun acquittement).
- **EX-SAF-02** [C] Émettre exige un **armement délibéré**, nommant explicitement le canal cible,
  lié à la connexion cliente qui l'a demandé, jamais mémorisé ni restauré. La perte de cette
  connexion (fermeture, plantage, absence de battement pendant 1 s) **désarme** le canal et arrête
  toute émission dans le délai de `EX-SAF-05`, compté depuis le constat de la perte par le démon :
  à la fermeture de la connexion (fermeture ou plantage du client), ou à défaut au terme de 1 s
  sans battement (`D-33`). Le passage en mode émission est une
  reconfiguration du canal : l'éventuelle interruption de capture est un événement horodaté de la
  trace, avec sa durée. L'armement d'un canal Tier 2 est refusé en v1, avec un message explicite
  (`D-30`).
  *Vérification* : tentative d'émission sans armement refusée et tracée ; `kill -9` du client
  armé pendant un rejeu sur bus virtuel → dernière trame émise moins de 50 ms après le `kill` ;
  client armé suspendu sans fermeture de sa connexion → dernière trame émise moins de 1 s + 50 ms
  après son dernier battement ; événement de reconfiguration présent ; armement d'un canal de pont Tier 2 simulé refusé.
- **EX-SAF-03** [C] Une **liste blanche d'identifiants émissibles** DOIT être définie par canal ;
  hors liste, l'émission est refusée avec un message explicite. Elle est **vide par défaut** :
  sans liste, rien n'est émissible (`D-28`). Autoriser tout identifiant exige une entrée
  générique explicite, signalée à l'armement. La liste est donnée à l'armement ou dans la
  configuration du démon ; un jeu de test peut en proposer une, jamais l'imposer. L'ordonnanceur
  du noyau applique la liste, quel que soit le `TxSource` ou le backend.
  *Vérification* : armement sans liste puis émission → refus ; émission d'un identifiant hors
  liste refusée et tracée ; entrée générique acceptée avec avertissement.
- **EX-SAF-04** [C] ClemCAN DOIT avertir si un identifiant que l'on s'apprête à émettre a été reçu
  sur ce canal depuis son ouverture sans avoir été émis par ClemCAN (risque de collision
  d'arbitrage). En mode non interactif, l'émission est alors refusée, sauf option explicite.
  *Vérification* : identifiant injecté par un nœud simulé puis demandé à l'émission ;
  avertissement en interactif, refus en ligne de commande sans l'option.
- **EX-SAF-05** [C] Dès qu'un canal est armé, l'état armé DOIT être exposé par le protocole et
  affiché en permanence dans l'interface (indicateur sur chaque fenêtre). Un **arrêt d'urgence**
  DOIT couper toute émission, accessible en une action unique, y compris au clavier, en ligne de
  commande et par le protocole. Délai : moins de 50 ms entre la réception de la commande par le
  démon et la dernière trame émise, files d'émission du système et du matériel vidées.
  *Vérification* : délai mesuré sur bus virtuel ; ⚙ mesuré sur PCAN par un second nœud ; IHM pour
  l'indicateur et le raccourci.
- **EX-SAF-06** [C] Un **galop d'essai sur bus virtuel** DOIT être exécuté et réussi avant le
  premier rejeu réel d'un jeu de test donné. Le galop valide **l'émission**, pas la réaction de
  la cible, absente du bus virtuel (`D-26`). Réussi : rejeu complet, sans émission refusée ni
  échouée, toutes trames conformes à la liste blanche et à leurs règles de contrôle, charge
  admissible (`EX-REJ-03`). Les assertions associées sont évaluées et rapportées à titre
  d'information, sans conditionner la réussite. Le galop est exécuté **par le démon qui fera le
  rejeu réel**, sur un bus virtuel qu'il ouvre lui-même avec les paramètres des canaux cibles ;
  la réussite est liée à l'empreinte (SHA-256) du jeu, de ses bases, de ses règles et de la liste
  blanche, et consignée dans un registre persistant de ce démon ; toute modification de l'un
  d'eux l'invalide. La réussite est jugée conjointement (`D-31`) : le noyau constate lui-même les
  émissions refusées ou échouées, la conformité à la liste blanche, le nombre de trames émises
  égal au nombre annoncé et la charge ; la couche 2, qui pilote le galop, constate la conformité
  des trames émises à leurs règles de contrôle et la déclare au noyau. Le noyau n'inscrit la
  réussite que si ses constats et cette déclaration sont tous favorables.
  *Vérification* : rejeu réel refusé sans galop ; accepté après galop, même si une assertion
  portant sur une réponse de la cible y est `not_evaluated` ; refusé à nouveau après modification
  d'une base ; galop exécuté sur un démon, rejeu demandé sur un autre → refusé ; galop déclaré
  réussi par la couche 2 alors qu'une émission a été refusée, ou qu'il manque une trame → non
  inscrit ; contrôle invalide déclaré par la couche 2 → non inscrit.
- **EX-SAF-07** [C] Toute trame émise DOIT être journalisée, sans exception et sans possibilité de
  désactiver cette journalisation. Si la journalisation est impossible (disque plein, journal
  fermé), l'émission est refusée.
  *Vérification* : chaque trame émise lors d'un rejeu sur bus virtuel est présente dans le journal ;
  émission refusée sur journal saturé.

### 5.15 Automatisation — `EX-AUT`

- **EX-AUT-01** [A, étendue à chaque palier] Une ligne de commande DOIT couvrir chaque fonction
  (§2) livrée, dès le palier où elle est livrée (`EX-ARC-06`), notamment : chargement de bases,
  ouverture et configuration de canal, enregistrement, navigation et extraction dans une trace,
  conversion de formats, décodage vers CSV, statistiques, sessions et profils (A) ; édition de
  base (modification, ajout, renommage, suppression, opérations par lot, comparaison et fusion),
  validation, génération de code C, détection assistée et règles de contrôle, alarmes,
  enregistrement déclenché, repères, comparaison de traces, génération de rapport (B) ; édition et
  rejeu d'un jeu de test, armement et arrêt d'urgence, évaluation d'assertions (C). Cette liste
  est indicative : la liste normative est le registre de commandes (`EX-ARC-06`).
  *Vérification* : pour chaque commande du registre livrée, un test l'exerce par la ligne de
  commande.
- **EX-AUT-02** [C] Les **assertions** DOIVENT s'exprimer dans le langage `EX-EXP`, dans un
  fichier `.tce`, avec des contraintes de temps (« ce signal atteint telle valeur en moins de
  200 ms après tel événement »). Une assertion relative à un événement est évaluée à chaque
  occurrence de l'événement ; son verdict est `pass`, `fail` ou `not_evaluated`. Le verdict est
  `not_evaluated` si l'événement n'est jamais survenu, si l'expression a valu NA sur toute sa
  fenêtre d'évaluation (`D-25`), ou si cette fenêtre recouvre une évaluation abandonnée (`D-24`) ;
  sinon `pass` ou `fail` se décide sur les seules valeurs vrai et faux.
  *Vérification* : vecteurs pour chaque verdict, dont un événement à plusieurs occurrences, un
  signal NA sur toute la fenêtre, et une fenêtre recouvrant une évaluation abandonnée.
- **EX-AUT-03** [C] Le code retour DOIT être exploitable en intégration continue : 0 si toutes les
  assertions passent ; 1 si une assertion échoue ou n'est pas évaluée (sauf option rendant
  `not_evaluated` non bloquant) ; 2 si la configuration ou les entrées sont invalides ; 3 en cas
  d'erreur d'exécution ou de bus. Le rapport est lisible par une machine (JSON, schéma versionné
  publié) **et** par un humain (HTML ou texte).
  *Vérification* : un scénario par code retour ; le JSON produit est validé contre le schéma.
- **EX-AUT-04** [A, étendue à chaque palier] Aucune sortie interactive ne DOIT être requise :
  aucune invite, aucun serveur graphique, pour chaque commande dès le palier où elle est livrée.
  Toute confirmation (canal sans *listen-only*, mode acquittement, collision…) se donne par une
  option explicite de la commande ; sans elle, la commande se comporte comme si la confirmation
  avait été refusée, et le signale (`D-34`).
  *Vérification* : la suite fonctionnelle s'exécute avec l'entrée standard fermée et sans
  variable d'affichage ; aucune commande n'attend d'entrée ; un profil en mode acquittement
  ouvert sans l'option s'ouvre en écoute seule, avec l'option en mode acquittement.

### 5.16 Session et internationalisation — `EX-SES`, `EX-I18N`

- **EX-SES-01** [A, étendue à chaque palier] Une session DOIT restituer les objets disponibles au
  palier : disposition des fenêtres, bases chargées par canal, signaux tracés avec échelles,
  couleurs et modes (A) ; filtres actifs, règles de contrôle, alarmes, repères (B) ; jeux de test ouverts
  (C).
  *Vérification* : pour chaque type d'objet livré, enregistrement puis réouverture ; état
  restitué identique (comparaison de l'état sérialisé).
- **EX-SES-02** [A] Le format de session DOIT être un texte lisible par un humain et diffable :
  ordre des clés stable, une valeur par ligne, chemins relatifs à la session quand c'est possible.
  *Vérification* : la modification d'un seul réglage produit un diff limité aux lignes de ce
  réglage ; deux enregistrements successifs sans modification sont identiques.
- **EX-SES-03** [A] Une session DOIT pouvoir être ouverte en l'absence d'un fichier référencé, en
  signalant précisément ce qui manque, sans perdre le reste. La référence manquante est conservée
  à l'enregistrement suivant.
  *Vérification* : session référençant une base absente ; ouverture avec message nommant le
  fichier ; réenregistrement ; la référence est toujours présente.
- **EX-SES-04** [A] Des **profils** (§2) DOIVENT pouvoir être enregistrés, listés et ouverts par
  leur nom, depuis l'interface et la ligne de commande. Ouvrir un profil crée une nouvelle session
  sans modifier le modèle. Un profil peut porter la configuration des canaux (débits, modes,
  protocole), les bases, les vues et, aux paliers suivants, les règles, les alarmes et les jeux de
  test ; il ne porte jamais d'armement, et un mode acquittement qu'il contient est soumis à
  confirmation (`EX-SAF-01`). Le répertoire des profils est configurable ; un profil est un
  fichier de session ordinaire (`EX-SES-02`), partageable par copie.
  *Vérification* : profil enregistré, ouvert par son nom en ligne de commande, session modifiée et
  enregistrée ; le fichier du profil est inchangé ; un profil en mode acquittement demande
  confirmation et s'ouvre en écoute seule sans elle.
- **EX-I18N-01** [A] La langue source est l'**anglais** (code, journaux, documentation technique).
  *Vérification* : contrôle automatique que le catalogue source est en anglais et complet.
- **EX-I18N-02** [A] L'interface DOIT être traduite en français dès le v1, la langue étant
  sélectionnable dans l'interface et mémorisée dans les préférences de l'utilisateur (pas dans
  la session).
  *Vérification* : catalogue français complet (aucune chaîne source sans traduction) ; IHM en
  français.
- **EX-I18N-03** [A] Le catalogue de langues DOIT être **ouvert** : ajouter une langue consiste à
  déposer un fichier de traduction, sans recompiler.
  *Vérification* : un catalogue de test déposé dans le répertoire prévu est proposé et appliqué.
- **EX-I18N-04** [A] Repli sur l'anglais pour toute chaîne non traduite ; **aucune phrase
  construite par concaténation** dans le code (les traductions resteraient agrammaticales).
  *Vérification* : catalogue partiel → chaînes manquantes en anglais ; règle d'analyse statique
  refusant toute concaténation avec une chaîne traduisible.
- **EX-I18N-05** [A] Nombres, dates et durées DOIVENT être formatés selon la locale, et les
  exports CSV DOIVENT rendre explicite le couple séparateur décimal / séparateur de colonne
  (point et virgule, ou virgule et point-virgule ; jamais virgule et virgule).
  *Vérification* : formatage sous `fr_FR` et `en_US` ; export CSV dans les deux couples ; le
  couple interdit est refusé.

---

## 6. Exigences non fonctionnelles de performance

### 6.1 Enveloppe nominale — `EX-PRF`

L'architecture est **non bornée en canaux** ; l'enveloppe ci-dessous est ce qui est
**contractuellement garanti et mesuré** (décision `D-14`, révisée par `D-15`).

**Machine de référence x86-64** : 4 cœurs physiques à 2,5 GHz ou plus, 16 Go de mémoire, SSD
NVMe, Linux. Le modèle exact est publié avec chaque mesure.

**Machine de référence ARM64** : carte à 4 cœurs Cortex-A72 ou plus récents, à 1,5 GHz ou plus,
4 Go de mémoire, stockage SSD sur USB 3 (une carte SD n'est pas un support de référence pour
la fenêtre d'écriture), Linux. Le modèle exact est publié avec chaque mesure.

**Profil de charge nominal**, huit canaux **simultanés**, identifiants 11 bits, données
pseudo-aléatoires à graine fixe publiée ; chaque canal émet à tour de rôle 64 identifiants
distincts, dans un ordre fixe publié :

| Canaux | Type | Débits | DLC | Débit par canal (saturé) |
|---|---|---|---|---|
| 4 | CAN classique | 1 Mbit/s | 8 | maximum théorique du profil, de l'ordre de 7 400 à 9 000 trames/s selon le bourrage |
| 4 | CAN FD ISO | 500 kbit/s nominal, 2 Mbit/s données | 15 (64 octets) | maximum théorique du profil, de l'ordre de 2 500 à 3 000 trames/s selon le bourrage |

Le débit exact de chaque canal est calculé par la formule de `EX-VUE-04` pour la séquence de
trames effectivement générée ; c'est ce débit que l'injecteur produit sur bus virtuel. À titre
d'ordre de grandeur, le profil représente environ 45 000 trames/s agrégées et 160 millions de
trames par heure.

**Banc matériel de référence (⚙)** : au minimum un bus classique et un bus FD, chacun équipé d'un
canal générateur et d'un canal enregistreur Tier 1 ; matériel FD de référence : PEAK PCAN-USB FD
(`D-15`).

- **EX-PRF-01** [A] Sur la machine de référence, le profil de charge nominal sur bus virtuel
  DOIT être enregistré pendant 60 minutes **sans aucune perte**, avec une mémoire stable : la
  différence entre maximum et minimum de la mémoire résidente du démon, entre la 5ᵉ et la
  60ᵉ minute, reste inférieure à 5 % de sa valeur à la 5ᵉ minute.
  *Vérification* : `bench/soak` au profil nominal ; compteurs de pertes à zéro, séquences
  numérotées complètes, mémoire relevée chaque seconde. ⚙ Le banc matériel enregistre ses deux
  bus saturés pendant 60 minutes sans perte.
- **EX-PRF-02** [A] **ARM64** : ClemCAN DOIT tenir le débit que le matériel présente effectivement,
  et compter toute perte. *Note factuelle* : sur cartes de type Raspberry Pi, le facteur limitant
  est généralement le contrôleur (SPI MCP2515/MCP2518FD, ou latence USB), non le processeur.
  *Vérification* : sur la machine de référence ARM64 et bus virtuel, zéro perte non déclarée au
  **demi-profil** (2 canaux classiques et 2 canaux FD du profil nominal, saturés) pendant
  60 min ; le débit maximal sans perte est recherché par paliers de 10 min (dichotomie sur le
  débit injecté) et publié par plateforme. Objectif indicatif publié sur matériel réel :
  2 canaux classiques saturés.
- **EX-PRF-03** [A] Au-delà de l'enveloppe, la dégradation DOIT être **observable** : compteurs de
  pertes et événements dans la trace (`EX-CAP-04`), signalement par le protocole, et
  avertissement dans l'interface dès qu'elle existe.
  *Vérification* : celle de `EX-CAP-04` ; IHM pour l'avertissement.
- **EX-PRF-04** [C] Rejeu : conforme à `EX-REJ-04`.
  *Vérification* : celle de `EX-REJ-04`.
- **EX-PRF-05** [A] Les chiffres de performance DOIVENT être produits par un banc automatisé,
  exécuté au moins chaque nuit, et publiés avec la plateforme, la version et la date. Une
  régression est signalée quand la médiane de 5 exécutions d'une métrique se dégrade de plus de
  10 % par rapport à la référence publiée.
  Le banc s'exécute sur les machines de référence (machines dédiées, pas sur les machines
  partagées de l'intégration continue).
  *Vérification* : présence de l'exécution nocturne et de sa publication ; une régression
  artificielle injectée est signalée.
- **EX-PRF-06** [B] **Évaluation en couche 2.** Sur la machine de référence x86-64, au profil
  nominal, un ensemble de règles de référence publié dans `bench/` (8 règles compteur et contrôle,
  4 déclencheurs et 4 alarmes, portant sur 16 identifiants) DOIT être évalué pendant 60 min sans
  aucune évaluation abandonnée (`D-24`). Sur ARM64, le débit d'évaluation maximal sans abandon est
  mesuré et publié, sans seuil contractuel.
  *Vérification* : `bench/` au profil nominal avec l'ensemble de référence ; compteur
  d'évaluations abandonnées à zéro ; mesure ARM64 publiée.

### 6.2 Robustesse

- **EX-ROB-01** [A] Une coupure d'alimentation ne DOIT jamais rendre une trace illisible
  (cf. `EX-JNL-01`).
  *Vérification* : celle de `EX-JNL-01`.
- **EX-ROB-02** [B] Le démon DOIT pouvoir démarrer automatiquement au démarrage du système et
  reprendre un enregistrement selon une configuration, sans intervention. La reprise ouvre un
  nouveau fichier ; le journal précédent, s'il est incomplet, est signalé par son événement de fin
  abrupte (`EX-JNL-01`).
  *Vérification* : redémarrage brutal d'une machine virtuelle configurée ; enregistrement repris
  dans un nouveau fichier sans intervention ; ancien journal lisible.
- **EX-ROB-03** [A] Un plantage de l'interface ne DOIT jamais interrompre un enregistrement
  (cf. `EX-ARC-01`).
  *Vérification* : celle de `EX-ARC-01`.
- **EX-ROB-04** [A] L'espace disque DOIT être surveillé : avertissement à un seuil configurable.
  La saturation est atteinte quand l'espace libre descend sous une réserve (défaut 256 Mo) ; la
  politique par défaut est l'**arrêt propre** : l'événement de fin est écrit dans la réserve, les
  trames non écrites sont comptées et signalées par le protocole et dans le journal du système.
  *Vérification* : système de fichiers de petite taille rempli pendant un enregistrement ;
  avertissement au seuil, arrêt propre, journal lisible, pertes comptées.
- **EX-ROB-05** [B] Une politique de saturation alternative, **rotation** avec effacement des
  fichiers les plus anciens de l'enregistrement en cours, DOIT être configurable (requiert
  `EX-JNL-04`). Elle n'efface jamais un fichier d'un autre enregistrement.
  *Vérification* : rotation sur système de fichiers de petite taille ; seuls les plus anciens
  fichiers de l'enregistrement en cours sont effacés, et chaque effacement est un événement.

### 6.3 Portabilité et livraison

- **EX-POR-01** [A] Linux x86-64, Linux ARM64, Windows x86-64.
  *Vérification* : construction et suite auto vertes sur les trois plateformes en intégration
  continue.
- **EX-POR-02** [A] Livraison sous forme d'**exécutable autonome** ne nécessitant l'installation
  d'aucun environnement d'exécution chez l'utilisateur (un exécutable Windows, une image portable
  Linux), plus une installation par gestionnaire de paquets pour les usages de développement.
  L'exécutable autonome complet inclut le pont Tier 2 et python-can ; les bibliothèques propres
  aux constructeurs (Vector, Kvaser…) restent à installer par l'utilisateur (`D-36`).
  *Vérification* : l'exécutable lance la ligne de commande et le démon dans un environnement neuf
  (conteneur ou machine virtuelle sans Python ni bibliothèques de développement) ; dans le même
  environnement, un canal Tier 2 s'ouvre sur le backend virtuel de python-can.
- **EX-POR-03** [A] Le démon DOIT être livrable et utilisable seul, sans composant graphique.
  Ce livrable n'inclut pas le pont : l'ouverture d'un canal Tier 2 y est refusée avec un message
  explicite (`D-36`).
  *Vérification* : livrable du démon seul installé dans un environnement neuf sans bibliothèque
  graphique ; enregistrement sur bus virtuel ; ouverture d'un canal Tier 2 refusée avec message.
- **EX-POR-04** [A] Aucune dépendance à un service en ligne. Fonctionnement intégralement hors
  réseau.
  *Vérification* : suite fonctionnelle exécutée sans accès réseau.

### 6.4 Sécurité

- **EX-SEC-01** [A] Aucun processus de ClemCAN (démon, ligne de commande, interface, pont)
  N'OUVRE de port réseau en écoute par défaut, en TCP comme en UDP ; seule une option explicite
  dont c'est l'objet en ouvre un (accès distant, `EX-SEC-02`). Pour le démon, cf. `EX-ARC-05`
  (`D-35`).
  *Vérification* : `ss -lntup` (Linux) et `netstat -ano` (Windows) pendant l'exécution de chaque
  processus dans sa configuration par défaut (démon, enregistrement en ligne de commande,
  interface en rendu hors écran, pont sur backend simulé) ne montrent aucun socket en écoute de
  ces processus.
- **EX-SEC-02** [B] L'accès distant au démon, s'il est activé, DOIT être chiffré et authentifié
  mutuellement : TLS 1.3, certificats serveur et client. Sans certificat client valide, la
  connexion est refusée. [C] L'armement d'émission DOIT être refusé par défaut aux clients
  distants, sauf autorisation explicite dans la configuration du démon.
  *Vérification* : connexion sans certificat, avec certificat invalide, et en TLS < 1.3 :
  refusées ; connexion valide acceptée ; armement distant refusé par défaut.
- **EX-SEC-03** [A] Aucune télémétrie, aucune remontée d'information.
  *Vérification* : suite fonctionnelle exécutée sous capture réseau ; aucune connexion sortante.
- **EX-SEC-04** [A] **Accès local.** Le socket de contrôle et la mémoire partagée du démon DOIVENT
  n'être accessibles qu'aux comptes autorisés (`D-28`) : sous Linux, propriétaire du démon et
  groupe `clemcan` (droits `0660`, groupe configurable) ; sous Windows, liste de contrôle d'accès
  limitée au compte du démon et aux comptes désignés dans sa configuration. [C] L'armement
  d'émission exige en outre une autorisation distincte : sous Linux, appartenance au groupe
  `clemcan-tx` (configurable) ; sous Windows, compte désigné à cet effet ; à défaut, seul le
  compte du démon peut armer.
  *Vérification* : connexion depuis un compte hors groupe refusée ; lecture de la mémoire
  partagée refusée ; armement par un compte du groupe `clemcan` hors `clemcan-tx` refusé.

---

## 7. Conformité protocolaire couverte

Références : ISO 11898-1:2015 (CAN classique et CAN FD ISO ; CAN FD non ISO lu, signalé) ;
spécification CAN FD 1.0 de Bosch, 2012 (durée des trames FD non ISO, `D-37`) ;
SAE J1939-21 (transport), J1939-73 (diagnostic), J1939-81 (gestion du réseau) ;
ISO 15765-2:2016 (ISO-TP) ; ISO 14229-1 (UDS) ; spécification AUTOSAR du protocole E2E.

| Domaine | Couvert en v1 | Palier |
|---|---|---|
| CAN 2.0A / 2.0B (11 et 29 bits), DLC classique 9–15 | oui | A |
| CAN FD ISO et non ISO (BRS, ESI, DLC 9–15, 64 octets) | oui | A |
| Ordre Motorola et Intel, signaux signés, flottants | oui | A |
| Multiplexage simple et étendu (`SG_MUL_VAL_`) | oui | A |
| J1939 niveau trame (PGN, SA, DA, priorité) | oui | A |
| J1939 transport BAM et CM/DT | oui | B |
| J1939 applicatif DM1/DM2, address claim, NAME, requête PGN | oui | B |
| ISO 15765-2 (ISO-TP) sur trames de 8 octets, adressages normal/normal fixe/étendu/mixte | oui | B |
| UDS en lecture (services, sous-fonctions, codes de refus) | oui | B |
| ISO-TP sur trames FD de plus de 8 octets, J1939-22 | **non** | — |
| ISO 11783 transport étendu (ETP) | **non** | — |
| CAN XL, LIN, FlexRay, Ethernet automobile | **non** | — |

---

## 8. Hors périmètre

**Hors v1, prévu v2 :**
- **Émission réactive** : « si je reçois ceci, j'émets cela » — simulation de nœud, séquenceur,
  panneau d'envoi manuel, générateurs périodiques interactifs, réponses automatiques.
  *L'architecture le prévoit* : le moteur de rejeu est un `TxSource` derrière un ordonnanceur
  d'émission unique, dans le noyau ; le moteur réactif sera un second `TxSource`.
- Émission de requêtes UDS et fonctions de diagnostic actives (lecture de DTC à la demande,
  scan d'ECU, écriture de paramètres).
- Écriture BLF (**premier jalon post-v1**, le moins coûteux et le plus utile en échange),
  puis écriture TRC, puis écriture MF4.
- Émission sur un canal Tier 2, par le pont (`D-30`) ; une fiche devra dire comment le délai
  d'arrêt d'urgence d'`EX-SAF-05` y est tenu ou amendé.

**Hors v1, non planifié :**
- Panneaux graphiques interactifs (jauges, boutons, voyants) à la manière de CANoe.
- Scripting utilisateur embarqué (au-delà du pilotage par ligne de commande).
- Bus autres que CAN et CAN FD, dont CAN XL et la lecture des fichiers TRC 3.x.
- Synchronisation GPS ou PPS, corrélation vidéo.
- Travail multi-utilisateur simultané sur une même session.
- Mise à jour automatique du logiciel.
- ISO 11783 ETP ; ISO-TP sur trames FD de plus de 8 octets ; J1939-22.
- Trames authentifiées (AUTOSAR SecOC) : ni vérification, ni recalcul.
- Description des identifiants de données UDS (ODX, CDD).

---

## 9. Risques

| # | Risque | Impact | Traitement retenu |
|---|---|---|---|
| R1 | Le profil de charge nominal (huit canaux simultanés) n'est pas tenu | Exigence centrale non satisfaite | Chemin chaud isolé en C dès le premier jalon ; banc de charge automatisé permanent ; optimisations identifiées d'avance (lecture par lots, horodatage noyau, écritures groupées, priorité temps réel, épinglage de cœur) |
| R2 | Frontière noyau C ↔ Python coûteuse ou fragile | Complexité, régressions | Protocole étroit et versionné, anneau en mémoire partagée en local, abonnements filtrés (`D-16`), tests de contrat des deux côtés, aucune logique métier dans le noyau |
| R3 | L'écriture DBC au diff minimal est plus difficile que prévu | Jalon B glissé | Corpus de DBC réels constitué **avant** l'implémentation ; analyseur concret propre à l'écriture (`D-19`) ; critères de `EX-DB-03` testés en continu ; repli documenté : refus d'enregistrer plutôt que perte silencieuse |
| R4 | Réassemblage J1939/ISO-TP incorrect sur cas dégradés | Fausses analyses | Vecteurs de test des cas dégradés écrits avant le code ; machines à états explicites et testées isolément |
| R5 | Périmètre v1 trop large pour un développeur seul | Abandon | Trois paliers livrables et utilisables (PLAN.md), chacun ayant sa valeur d'usage propre |
| R6 | Plusieurs agents d'IA produisent du code divergent | Dette, incohérences | CLAUDE.md comme contrat unique ; exigences numérotées ; toute décision hors spécification fait l'objet d'une fiche de décision |
| R7 | Matériel ARM limitant (SPI/USB) mal anticipé | Attentes déçues | `EX-PRF-02` formulé en « tenir ce que le matériel présente et compter les pertes » ; mesures publiées par plateforme |
| R8 | Émission accidentelle sur un système réel | Matériel, sécurité des personnes | `EX-SAF-01` à `EX-SAF-07` : écoute seule par défaut, armement lié à la connexion, liste blanche, arrêt d'urgence, galop d'essai obligatoire |
| R9 | Nom du projet | Confusion de marque | **Clos le 2026-09-19.** Décision D-09 : « ClemCAN ». Libre sur PyPI, npm et crates.io, aucune mention sur GitHub. Recherche `data.inpi.fr` — base couvrant les marques françaises, de l'Union européenne et internationales en vigueur en France — : **0 marque, 0 brevet, 0 dessin ou modèle**, et 1 seule entreprise homonyme (SCI de location immobilière, SIREN 844 974 618, sans recouvrement d'activité donc sans risque de confusion). Seul reliquat, sans objet aujourd'hui : aucun registre hors Europe consulté |
| R10 | cantools ne couvre pas certaines constructions DBC réelles ni le décodage par signal | Bases réelles illisibles, décodage faux sur multiplexeur inconnu | Filtre isolé des sections refusées et correctif proposé en amont (`D-19`) ; extraction de bits propre au projet, cantools en oracle (`D-18`) |
| R11 | Banc matériel insuffisant pour valider le FD et l'enveloppe | Exigences ⚙ non validées | Matériel de référence nommé (`D-15`) ; validation du débit logiciel sur bus virtuel, de la physique sur banc ⚙ |

---

## 10. Fiches de décision

Décisions `D-01` à `D-14` arrêtées au terme de l'entretien de conception du 2026-09-19 ;
`D-15` à `D-23` arrêtées lors de la relecture de cohérence du 2026-10-07 ; `D-24` à `D-29`
arrêtées lors de la seconde relecture (version 1.2) ; `D-30` et suivantes arrêtées lors de
l'arbitrage de la revue des SRS (version 1.3). Toute remise en cause
exige une nouvelle fiche, pas une modification silencieuse. Une fiche postérieure prévaut sur le
point des fiches antérieures qu'elle amende, et le dit.

| # | Décision | Justification | Rejeté |
|---|---|---|---|
| **D-01** | **Noyau C** (démon `clemcand` + `libclemcan`) propriétaire du bus ; **analyse et interface en Python**, interface **Qt (PySide6)** | Le seul code soumis au débit ligne est isolé et optimisable ; le reste du périmètre (l'essentiel du travail) bénéficie de cantools, asammdf, numpy, python-can. Une fois le chemin chaud en C, le langage de l'interface devient un choix d'ergonomie, non de performance | Tout Python (appel système par trame trop coûteux sur ARM, `recvmmsg` inaccessible) ; tout C++ (couche driver, BLF et MF4 à écrire, 20–32 mois) ; interface web (port en écoute sur un outil qui écrit sur un bus, deux langages, graphe temps réel plus difficile) |
| **D-02** | Le noyau est un **démon**, pas seulement une bibliothèque | Résout d'une seule pièce l'enregistrement autonome en véhicule et le pilotage d'une cible embarquée depuis un PC ; l'interface devient jetable | Bibliothèque en processus unique (un plantage d'interface perdrait l'enregistrement) |
| **D-03** | **Tier 1** : SocketCAN + PCAN-Basic + bus virtuel, natifs dans le noyau C. **Tier 2** : tout backend python-can, sans garantie de débit ni d'horodatage. *Amendée par `D-15` et `D-17`.* | Couvre le matériel réellement disponible (PEAK PCAN-USB) sur les deux systèmes, sans écrire cinq pilotes, tout en restant ouvert | Écrire tous les pilotes ; ou tout déléguer à python-can |
| **D-04** | **Journal natif en ajout seul** comme format d'enregistrement ; conversion vers les formats d'échange après coup | Un format à blocs compressés perd la fin du fichier sur coupure ; l'ajout seul borne la perte à la fenêtre d'écriture | Écrire directement en BLF ou MF4 |
| **D-05** | **Écriture v1 limitée** au journal natif, ASC, candump et CSV ; BLF, TRC et MF4 en lecture seule | Choix explicite de l'utilisateur, périmètre v1 resserré | Parité lecture/écriture complète (MF4 coûteux et difficile à valider) |
| **D-06** | **Écriture DBC au diff minimal**, écrivain propre au projet, lecture par cantools. *Précisée par `D-19`.* | La base éditée part en génération de code C et vit dans Git : un fichier entièrement reformaté rend la revue impossible et fait disparaître les sections non modélisées | Écriture par cantools (réécriture complète, pertes silencieuses) |
| **D-07** | **Escalier par défaut**, choix de mode par signal, sémantique partagée avec export et assertions. *Précisée par `D-21`.* | Un signal CAN n'existe qu'aux instants d'arrivée ; interpoler invente des valeurs | Ligne interpolée par défaut ; escalier imposé partout |
| **D-08** | **Règles de compteur et de contrôle dans un fichier annexe** `.tcr`, avec détection assistée validée par l'utilisateur | Le DBC ne sait pas décrire un CRC ; le fichier annexe reste portable, versionnable, partageable, et ne modifie pas une base fournie par un tiers | Attributs `BA_` dans le DBC (risque de suppression par les outils tiers) ; détection automatique seule (non reproductible, et inutilisable pour recalculer à l'émission) |
| **D-09** | Nom **ClemCAN** (`clemcand`, `clemcan`, `libclemcan`) | Vérifié libre le 2026-09-19 : disponible sur PyPI, npm et crates.io, **aucune** mention sur GitHub, et recherche INPI (marques françaises, de l'Union européenne et internationales en vigueur en France) **sans antériorité : 0 marque, 0 brevet, 0 dessin ou modèle**. Aucune ambiguïté possible dans le domaine CAN | `CanAlyzer` (quasi-homonyme de CANalyzer, marque Vector, même catégorie de produit) ; `Toucan` (Rusoku Technologies commercialise une gamme d'adaptateurs USB-CAN **TouCAN** livrée avec sa propre interface Qt d'analyse ; TouCAN est en outre le nom du module contrôleur CAN 2.0B de Motorola/Freescale ; et `toucan` est déjà pris sur PyPI, npm et crates.io) ; `Boucan`, `Busard`, `ClémenCAN` (libres, écartés par préférence) |
| **D-10** | **Anglais** langue source, français livré, **catalogue de langues ouvert** | Le domaine est anglophone ; l'interface doit être utilisable par un technicien francophone ; prévu tôt, le coût est nul | Français seul (ferme la contribution) ; anglais seul (interface moins immédiate en atelier) |
| **D-11** | **Un seul langage d'expression** pour filtres, déclencheurs, alarmes et assertions | Ces quatre besoins posent la même question ; trois moteurs séparés seraient incompatibles entre eux | Une syntaxe par usage |
| **D-12** | **Trois paliers livrables** (Observer, Comprendre, Agir) à périmètre inchangé | Un v1 de 12 à 16 mois sans usage intermédiaire se trompe longtemps sans le savoir | Un seul v1 complet ; réduction du périmètre |
| **D-13** | **Chaque exigence porte un test automatisé** ; l'interface reste une coquille fine avec tests de fumée seulement. *Précisée par `D-23`.* | L'automatisation de clics est fragile et coûteuse ; puisque tout est pilotable sans interface, les tests passent par là | Validation manuelle guidée ; tests limités au noyau |
| **D-14** | **L'enregistrement est prioritaire** en surcharge ; toute perte est comptée et déclarée | La trace est souvent le seul livrable d'un essai ; un outil de mesure ne mentira jamais par omission | Priorité à l'affichage ; arrêt en erreur |
| **D-15** | **Enveloppe fondée sur la physique du bus et matériel de référence nommé.** Profil nominal défini par trames (§6.1), débit dérivé de la formule de charge ; débit logiciel validé sur bus virtuel, physique validée sur banc ⚙. Garanties Tier 1 attachées à des couples (pilote, matériel). Sous Linux, le matériel PEAK passe par SocketCAN (`peak_usb`) ; PCAN-Basic est réservé à Windows. Matériel FD de référence : PCAN-USB FD. Point d'accès réseau : le démon. *Amende `D-03`.* | Les chiffres de la v1.0 (≈ 15 000 trames/s par canal classique saturé, ≈ 25 000 par canal FD à 500k/2M) dépassaient le maximum physique ; le PCAN-USB n'a pas de CAN FD ; PCAN-Basic sous Linux exige un pilote hors arbre incompatible avec `peak_usb` sur le même boîtier ; le démon est le seul composant présent sur une cible embarquée | Garder des chiffres agrégés sans profil de trames ; exiger PCAN-Basic sous Linux ; exposer le réseau depuis la couche Python |
| **D-16** | **Flux vers la couche 2 par abonnement.** La couche 2 ne reçoit que les trames auxquelles elle s'abonne (par canal, identifiant, masque), filtrées par le noyau ; un abonnement d'évaluation est sans perte, un abonnement d'affichage est décimable. Le noyau calcule les statistiques de transport par identifiant (compteurs, périodes, gigue, octets changés, charge) et applique les filtres de capture identifiant/masque ; il conserve le tampon pré-déclenchement. Toute évaluation retardée ou abandonnée est comptée. *Amendée par `D-24`, précisée par `D-39`.* | Les contrôles, déclencheurs et statistiques doivent voir chaque trame concernée, alors que la couche 2 ne peut pas suivre le débit ligne et que le noyau ne peut pas porter de logique métier ; les statistiques de transport et le filtrage par identifiant n'en sont pas | Couche 2 recevant tout le débit ; expressions évaluées dans le noyau ; statistiques dans la couche 2 |
| **D-17** | **Tier 2 par pont.** Un processus Python optionnel ouvre le backend python-can et injecte ses trames dans le noyau par une interface d'injection documentée ; l'émission passe par l'ordonnanceur du noyau, qui transmet au pont. Le noyau reste la seule autorité sur le journal, la liste blanche et l'émission. Les garanties non assurées sont déclarées dans les métadonnées. *Amende `D-03`. Amendée par `D-30`, précisée par `D-36`.* | Concilie l'ouverture Tier 2 avec l'ordonnanceur unique, la journalisation obligatoire et la survie de l'enregistrement | Backends Python propriétaires du bus hors du noyau ; abandon du Tier 2 |
| **D-18** | **Extraction de bits par signal propre au projet**, cantools restant la source du modèle de base de données et l'oracle de test du décodage. | cantools décode des messages entiers et rejette tout le message quand la valeur du multiplexeur n'est pas définie (vérifié sur cantools 44.2), ce qui contredit `EX-DEC-04` et empêche le décodage paresseux par signal (`EX-DEC-11`) ; l'extraction de bits n'est pas du parsing de base de données | Contournement par interception d'exception ; décodage de messages entiers ; correctif amont seul (délai non maîtrisé) |
| **D-19** | **Écriture DBC par analyseur concret propre au projet**, limité à l'écriture : il conserve chaque octet et la position de chaque définition ; la lecture sémantique reste cantools, oracle de la relecture. Les sections refusées par cantools sont écartées par un filtre isolé avant lecture et conservées pour l'écriture ; un correctif est proposé en amont. *Précise `D-06`.* | cantools ne conserve aucune position source et son écrivain régénère tout (ordre fixe, fins de ligne imposées, tri des `VAL_`) ; il refuse `SGTYPE_`, `CAT_DEF_`, `FILTER`, présents dans des DBC réels (vérifié sur cantools 44.2) | Écrire par cantools ; réécrire la lecture sémantique ; refuser les DBC contenant ces sections |
| **D-20** | **Écoute seule = mode *listen-only* du contrôleur** par défaut ; mode **acquittement** optionnel et explicite ; **armement lié à la connexion cliente**, révoqué à sa perte (homme mort) ; arrêt d'urgence aussi par la ligne de commande et le protocole. | Seul le *listen-only* garantit qu'aucun bit n'est écrit sur un bus véhicule ; au banc avec un seul calculateur, l'absence d'acquittement fausse le trafic, d'où l'option ; le démon survivant au client, un armement non lié à la connexion laisserait une émission sans témoin ni arrêt | Mode normal par défaut ; armement « pour la session » ; arrêt d'urgence dans l'interface seule |
| **D-21** | **Une seule fonction « valeur à l'instant t »** (escalier, NA), utilisée par l'affichage en escalier, l'export avec maintien, la comparaison, les contrôles, les déclencheurs et les assertions. Le mode de tracé est un choix d'affichage ; l'export choisit son mode explicitement. *Précise `D-07`.* | La v1.0 imposait la même sémantique au graphe et à l'export alors que l'export n'offrait pas l'interpolation ; une assertion ne doit pas changer de verdict selon l'affichage | Propager le mode d'affichage aux assertions et à l'export |
| **D-22** | **Validation DBC bloquante sur ce qui est introduit ou touché**, avertissement sur l'existant. | Les DBC réels contiennent souvent des défauts ; une validation bloquante globale rendrait impossible l'édition et le test d'aller-retour du corpus réel | Validation bloquante globale ; validation consultative seule |
| **D-23** | **Classes de vérification** auto, ⚙ et IHM (§1.1). La construction échoue si une exigence d'un palier livré n'a aucun test auto ; les tests ⚙ et IHM figurent dans la matrice, avec leur dernier résultat. *Précise `D-13`.* | Certaines exigences sont matérielles ou propres à l'interface ; sans classe déclarée, elles seraient couvertes par des tests vides, ou la construction échouerait dès le premier jalon | Tout automatiser, y compris le matériel ; validation manuelle non tracée |
| **D-24** | **Abonnements d'évaluation bornés, jamais de contre-pression.** Chaque abonnement d'évaluation a une file bornée (taille en trames fixée à l'abonnement, configurable, défaut 65 536). File pleine : les trames suivantes de cet abonnement sont abandonnées jusqu'à ce qu'elle se vide de moitié ; l'abandon produit un événement horodaté portant l'intervalle et le nombre de trames, et tout résultat dépendant de cet intervalle (violation, déclenchement, alarme, verdict) est « non évaluable ». L'enregistrement n'est jamais affecté. *Amende `D-16`, dont l'abonnement d'évaluation « sans perte » devient « non décimé, à file bornée ».* | `D-16` déclarait l'évaluation « sans perte » tout en comptant les abandons : une file non bornée épuise la mémoire d'une cible embarquée, une contre-pression mettrait l'enregistrement en danger, ce que `D-14` interdit | File non bornée ; contre-pression sur le noyau ; abandon silencieux |
| **D-25** | **Logique à trois valeurs pour NA** (Kleene) ; une condition n'est satisfaite que si elle vaut vrai ; un passage depuis ou vers NA n'est pas un front ; une assertion entièrement NA est `not_evaluated`. *Précise `EX-EXP-03`.* | Avec « toute comparaison impliquant NA est fausse », `!(s > 3)` devenait vraie sur un signal absent : une assertion ou un filtre pouvait réussir sur une donnée inexistante. La première réception d'un signal aurait aussi produit un faux front | Comparaisons fausses sur NA ; NA traité comme zéro ; erreur d'évaluation sur NA |
| **D-26** | **Le galop d'essai valide l'émission, pas la réaction**, et il est exécuté et consigné par le démon qui fera le rejeu réel. *Précise `EX-SAF-06`. Précisée par `D-31`.* | La cible est absente du bus virtuel : exiger les assertions rendait le galop impossible. Quand un PC pilote une cible distante, un registre côté client pouvait valider un galop fait sur une autre machine que celle qui émet | Assertions exigées ; registre tenu par le client ; galop facultatif |
| **D-27** | **Fonction contre interaction de visualisation, registre de commandes unique.** Seules les fonctions (§2) doivent exister en ligne de commande ; l'interface les déclenche par le même registre que la ligne de commande. *Précise `EX-ARC-06`.* | Exiger un équivalent pour chaque action d'interface (zoom, curseurs) était irréalisable et sans valeur ; un registre partagé rend la couverture vérifiable mécaniquement et garantit que l'interface reste une coquille | Équivalent pour toute action ; liste maintenue à la main |
| **D-28** | **Moindre privilège par défaut** : liste blanche d'émission vide par défaut ; accès local au démon limité à un groupe ; armement réservé à un second groupe. *Complète `D-20`.* | Une liste blanche absente qui autorise tout annulerait le garde-fou au moment où il compte (atelier, véhicule) ; sans droits sur le socket local, tout processus de la machine pourrait armer l'émission | Liste absente = tout autorisé ; socket local ouvert à tous ; un seul niveau d'accès |
| **D-29** | **Compteurs régénérés en continu au rejeu**, pour tout identifiant couvert par une règle, et non seulement pour les trames modifiées. *Précise `EX-CHK-05`.* | La sélection d'une partie des trames d'un identifiant et la boucle créaient des sauts de compteur que la cible rejette ; régénérer en continu couvre ces cas et rend les octets d'origine quand rien n'a changé | Recalcul des seules trames modifiées ; interdiction de filtrer un identifiant protégé |
| **D-30** | **Pas d'émission en Tier 2 en v1.** L'armement d'un canal Tier 2 est refusé ; le pont est en réception seule. Le délai d'arrêt d'urgence d'`EX-SAF-05` vaut donc sans exception pour tout canal armable. L'émission par le pont est reportée après la v1 (§8). *Amende `D-17`.* | Le délai d'`EX-SAF-05` court jusqu'à la dernière trame émise, files du système et du matériel vidées. Par le pont, une trame remise à python-can est dans la file du pilote : son vidage est une méthode optionnelle que tous les backends n'implémentent pas, socketcand ajoute un trajet réseau, et l'ordonnancement d'un processus Python n'est pas borné. Une file bornée côté pont limite le nombre de trames émises après l'arrêt, pas le délai. Le besoin d'émission v1 est couvert par le Tier 1 (matériel PEAK). Le report n'empêche rien : l'émission par l'ordonnanceur unique reste l'architecture cible du pont | Exception de Tier 2 à `EX-SAF-05` (délai garanti jusqu'au pont seulement, armement Tier 2 soumis à confirmation) ; délai Tier 2 mesuré sans seuil, sans amender `EX-SAF-05` (assouplissement silencieux) |
| **D-31** | **Galop d'essai jugé conjointement.** Le « démon » d'`EX-SAF-06` est le noyau : il ouvre le bus virtuel, tient le registre et constate lui-même ce qui relève du transport (émissions refusées ou échouées, liste blanche, nombre de trames émises égal au nombre annoncé, charge). La couche 2 pilote le galop comme un rejeu, depuis n'importe quel poste, constate la conformité des trames émises à leurs règles de contrôle et la déclare au noyau. Le noyau n'inscrit la réussite que si tous ces constats sont favorables. *Précise `EX-SAF-06` et `D-26`.* | Le noyau ne peut ni lire un jeu de test ni recalculer un compteur ou un contrôle (`EX-ARC-03`) : il ne peut pas juger seul. Exiger la couche 2 sur la machine du démon rendrait Python obligatoire sur une cible qui émet et empêcherait le pilotage à distance (`D-15`). Le registre fait déjà confiance à l'empreinte fournie par le client ; la déclaration de conformité est de même nature : elle protège contre l'erreur, pas contre un compte autorisé malveillant | Jugement par le noyau seul (logique métier dans le noyau) ; couche 2 obligatoire sur la machine du démon ; réussite déclarée par le client seul (le noyau ne constaterait rien) |
| **D-32** | **Événements d'analyse inscrits dans la trace en direct seulement.** Pendant un enregistrement, les violations (`EX-CHK-03`) et les levées et retombées d'alarme (`EX-EXP-06`) sont inscrites comme événements dans la trace en cours. Sur une trace déjà enregistrée, ce sont des résultats d'analyse : visibles dans la chronologie, exploitables comme condition, exportables, jamais écrits dans la trace. Leur support est tranché par `O-08`. L'événement de déclenchement (`EX-TRG-04`) n'est pas concerné : l'enregistrement déclenché n'existe qu'en direct. *Précise `EX-CHK-03` et `EX-EXP-06`.* | Une trace est une donnée de mesure, jamais modifiée en place (§2). Hors ligne, ces résultats se recalculent depuis la trace, les bases et les règles. Une copie annotée dupliquerait la trace à chaque analyse, soit plusieurs gigaoctets par heure au profil nominal | Copie annotée de la trace à chaque analyse ; écriture dans la trace d'origine |
| **D-33** | **Délai de désarmement compté depuis le constat de la perte.** Le démon constate la perte de la connexion armée à sa fermeture (fermeture ou plantage du client) ou, à défaut, au terme de 1 s sans battement ; le délai d'`EX-SAF-05` court depuis ce constat. Un plantage arrête donc l'émission en moins de 50 ms ; un client figé, en moins de 1 s + 50 ms après son dernier battement. *Précise `EX-SAF-02`.* | Le délai d'`EX-SAF-05` part de la réception d'une commande, absente en cas de perte de connexion. Le plantage d'un client ferme sa connexion, ce que le démon voit sans attendre : tolérer 1 s de plus laisserait émettre sans témoin. Le battement ne sert qu'au client figé ou à la liaison coupée sans fermeture | Borne unique de 1 s + 50 ms après le dernier battement, quelle que soit la cause |
| **D-34** | **Aucune question en ligne de commande dès le palier A.** `EX-AUT-04` passe au palier A, étendue à chaque palier comme `EX-AUT-01`. Toute confirmation se donne par une option explicite ; sans elle, la commande se comporte comme si la confirmation avait été refusée, et le signale. *Amende `EX-AUT-04`.* | Des confirmations existent dès le palier A (canal sans *listen-only*, `EX-SAF-01` ; profil en mode acquittement, `EX-SES-04`). Une invite bloquerait un script ou un enregistreur sans opérateur. L'absence de serveur graphique était déjà exigée au palier A par la vérification d'`EX-ARC-06` | Garder le palier C (commandes des paliers A et B pouvant bloquer, reprise au palier C) |
| **D-35** | **Aucun port en écoute, pour tout processus de ClemCAN.** `EX-SEC-01` couvre le démon, la ligne de commande, l'interface et le pont, en TCP comme en UDP ; seule une option explicite dont c'est l'objet ouvre un port. `EX-ARC-05` reste la règle propre au démon. *Précise `EX-SEC-01`.* | `EX-SEC-01` ne nommait pas son sujet, et sa vérification ne contrôlait que le démon. La couche 2 et l'interface dépendent de nombreuses bibliothèques, dont l'une peut ouvrir un port sans que le code du projet le demande : seule une vérification sur chaque processus le détecte | Règle limitée au démon |
| **D-36** | **Pont inclus dans l'exécutable autonome et lancé par le démon.** L'exécutable autonome complet inclut le pont et python-can ; les bibliothèques des constructeurs restent à la charge de l'utilisateur. Un canal Tier 2 s'ouvre par le même protocole qu'un canal Tier 1 : le démon lance alors le pont, le surveille, et inscrit sa fin ou son plantage comme événement. Le livrable du démon seul, sans Python, n'inclut pas le pont et refuse l'ouverture d'un canal Tier 2. *Précise `D-17`, `EX-POR-02` et `EX-POR-03`.* | Lire un adaptateur Tier 2 ne doit pas exiger d'installer un environnement d'exécution (`EX-POR-02`). Une ouverture uniforme garde une seule commande pour tous les canaux ; le démon, parent du pont, détecte sa fin sans délai ni protocole supplémentaire (`EX-CAP-04`). Les bibliothèques des constructeurs sont propriétaires et livrées par chacun d'eux | Pont installé à part par le gestionnaire de paquets et lancé par l'utilisateur (Python requis pour lire un adaptateur Tier 2, deux procédures d'ouverture) |
| **D-37** | **Durée de trame exacte aussi en CAN FD non ISO.** Sur un canal en FD non ISO, la durée d'une trame se calcule par la même fonction qu'en FD ISO, selon la spécification CAN FD 1.0 de Bosch (2012), sans compteur de bourrage ; l'exigence d'écart nul en bits sur vecteurs de référence s'applique. *Précise `EX-VUE-04`.* | `EX-VUE-04` ne définissait la durée qu'en ISO, alors qu'un canal peut être configuré en FD non ISO (`EX-BUS-02`) et que §7 en prévoit la lecture. La différence est fixe et documentée ; appliquer la formule ISO allongerait chaque trame de quelques bits, ce qui peut dépasser l'écart de 2 % sur les trames courtes | Formule ISO appliquée au non ISO, charge signalée comme approchée |
| **D-38** | **Cas dégradés du transport : le réassemblage se comporte comme le nœud récepteur.** J1939 : `TP.DT` déjà reçue ignorée et comptée, sauf retransmission demandée par une CTS ; nouvelle annonce pour un couple d'adresses en session → session ouverte close « remplacée » ; chaque attente suit la temporisation du nœud qui attend (BAM : T1 ; RTS/CTS : T3, T2, T1, T4), Tr et Th ne closant aucune session. ISO-TP : numéro de séquence erroné ou répété → transmission incomplète ; trame unique ou première trame pendant une réception → réception close « interrompue », la nouvelle est traitée ; trame consécutive hors réception ignorée ; débordement → transmission incomplète. Le résultat `transport_incomplete` porte son motif (`D-40`). *Précise `EX-DEC-07` et `EX-DEC-09`.* | Ces cas figuraient dans les vérifications sans issue définie, alors que les vecteurs s'écrivent avant le code (R4). Se comporter comme le récepteur montre ce que le calculateur a réellement accepté ; un observateur plus tolérant présenterait comme complets des messages rejetés par leur destinataire | Réassemblage tolérant, récupérant le plus de données possible ; abandon de session à toute anomalie, doublons compris |
| **D-39** | **Délestage distinct de la décimation.** La décimation (§2) réduit les points d'une série décodée en préservant les extrema ; elle est faite par la couche 2. Le délestage (§2) est l'abandon, par le noyau, de trames d'un abonnement d'affichage dont le consommateur ne suit pas : compté, signalé, sans garantie d'extrema, sans effet sur l'enregistrement. *Précise `D-16`, `EX-CAP-05` et `EX-GRA-06`.* | Le noyau ne décode aucun signal (`EX-ARC-03`) : il ne peut pas préserver les extrema d'un signal. Un même mot pour deux opérations aux garanties opposées rendait `EX-GRA-06` intenable en surcharge | Préservation des extrema par le noyau (décodage dans le noyau) ; un seul terme pour les deux opérations |
| **D-40** | **`transport_incomplete` est un résultat d'analyse, jamais un événement de la trace.** Il est horodaté, compté, porte son motif et figure dans la chronologie. *Précise `EX-DEC-07` et `EX-DEC-09`.* | Le réassemblage ne s'exécute que si un affichage, un export ou une expression le demande (`EX-DEC-11`) : l'inscrire dans la trace rendrait son contenu dépendant des vues ouvertes pendant l'enregistrement. Le résultat se recalcule à l'identique depuis la trace | Inscription dans la trace quand le réassemblage est actif en direct ; réassemblage systématique à la réception (contraire à `EX-DEC-11`) |

---

## 11. Décisions ouvertes

À trancher au jalon indiqué, par une fiche de décision.

| # | Question ouverte | Tranchée au | Critère |
|---|---|---|---|
| **O-01** | Mécanique exacte du protocole noyau ↔ clients en local (socket de contrôle + anneau en mémoire partagée : disposition, abonnements, contre-pression) et en distant (encapsulation sur TLS) ; le point d'accès est fixé par `D-15` | Jalon A2 | Débit soutenu, latence, simplicité de la version distante |
| **O-02** | Disposition binaire exacte du journal natif (`.tcj`) et de son index | Jalon A1 | Débit d'écriture, coût de reconstruction d'index, compacité |
| **O-03** | Bibliothèque de tracé (pyqtgraph ou équivalent) | Jalon A7 | `EX-GRA-06` (préservation des extrema) et latence de zoom sous 200 ms sur une heure de trace au profil nominal |
| **O-04** | Mécanisme d'empaquetage par plateforme | Jalon A9 | Un livrable autonome complet et un livrable du démon seul par plateforme (`EX-POR-02`, `EX-POR-03`) ; démarrage à froid sous 3 s sur la machine de référence ; taille du livrable complet sous 300 Mo |
| **O-05** | Format du jeu de test (`.tct`) : YAML, TOML ou CSV structuré | Jalon C2 | Lisibilité d'un diff Git, expressivité (valeurs de signaux, périodes), facilité de génération par script |
| **O-06** | Opportunité d'une interface web **de supervision** (état, démarrage/arrêt, récupération de trace) | Après le palier C | Besoin réel constaté ; ne remet pas en cause `EX-ARC-05` (activation explicite) |
| **O-07** | Stratégie de reprise après bus-off par défaut (`EX-BUS-07`) | Jalon C1 | Comportement attendu en véhicule autonome |
| **O-08** | Support des résultats d'analyse d'une trace déjà enregistrée (violations, alarmes, `D-32`) : dans la session ou dans un fichier annexe | Palier B, avant les contrôles hors ligne | Partage avec la trace (`EX-ANA-02`), lisibilité d'un diff de session, coût de recalcul |

---

## 12. Traçabilité

- Chaque exigence `EX-*` DOIT être référencée par au moins un test, dont le nom contient
  l'identifiant, et dont la classe (auto, ⚙, IHM) est déclarée (`D-23`).
- Une matrice exigence → test(s) → classe → jalon → dernier résultat DOIT être générée
  automatiquement et publiée.
- La construction DOIT échouer si une exigence d'un palier **déjà livré**, ou d'un jalon déclaré
  atteint, n'a aucun test auto associé. Les exigences des paliers à venir figurent dans la
  matrice sans faire échouer la construction.
- Les exigences de performance renvoient aux chiffres publiés par le banc, avec plateforme,
  version et date.
