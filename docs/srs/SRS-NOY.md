# SRS-NOY — Noyau de capture (`clemcand` / `libclemcan`)

> Spécification des exigences logicielles du composant **noyau**.
> Version 1.1 — 2026-11-25. Dérivée de `SPEC.md` v1.3, qui prévaut en cas d'écart.
> Conventions : `docs/srs/README.md`.

---

## 1. Objet et périmètre

Le noyau est le seul composant soumis au débit ligne et la seule autorité sur le bus. Il reçoit,
horodate, compte les pertes, enregistre, calcule les statistiques de transport, sert les flux aux
clients et ordonnance toute émission. Il est livré sous deux formes : le démon `clemcand` et la
bibliothèque `libclemcan` (mode lié, lecture et écriture du journal natif). Le démon lance et
surveille le pont Tier 2 (`D-36`).

**Il ne fait pas** (`EX-ARC-03`) : décodage de signal, lecture de base de données, évaluation
d'expression, réassemblage de transport (J1939, ISO-TP), calcul de compteur ou de contrôle,
interprétation de protocole applicatif. Il transporte des paramètres de canal (bases associées,
protocole) **sans les interpréter**, pour les restituer et les inscrire dans les métadonnées.

## 2. Références

`SPEC.md` v1.3 ; ISO 11898-1:2015 ; spécification CAN FD 1.0 de Bosch (2012) ; documentation
SocketCAN du noyau Linux ; API PCAN-Basic (PEAK-System) ; RFC 8446 (TLS 1.3).

## 3. Interfaces

| Id | Interface | Sens | Description |
|---|---|---|---|
| **I-PROT** | Protocole client | fournie | Contrôle des canaux, abonnements, statistiques, événements, enregistrement, émission, armement (`EX-ARC-04`). Local : socket de contrôle et anneau en mémoire partagée ; distant : TCP sous TLS 1.3 mutuel. Encodage : `O-01`. |
| **I-INJ** | Injection | fournie | Point d'entrée du pont Tier 2, lancé et surveillé par le démon : configuration du canal et confirmation d'ouverture vers le pont, trames et événements reçus vers le noyau (`D-17`, `D-36`). Aucune émission en v1 (`D-30`). |
| **I-LIB** | API `libclemcan` | fournie | Capture en mode lié, lecture et écriture du journal natif, calcul de durée de trame. ABI C stable et versionnée. |
| **I-BACK** | Backends | requise | SocketCAN (Linux, dont `peak_usb` et `vcan`), PCAN-Basic (Windows), bus virtuel ClemCAN (Windows). |
| **I-FS** | Système de fichiers | requise | Écriture en ajout, synchronisation sur le support, mesure de l'espace libre. |

Services d'I-PROT, à couvrir intégralement par le protocole :

| Service | Contenu |
|---|---|
| P1 Version et capacités | Version du protocole, backends détectés, couples (pilote, matériel), Tier |
| P2 Canaux | Lister, ouvrir, configurer, fermer ; état, mode de contrôleur, paramètres (`SRS-NOY-BUS-02`) |
| P3 Abonnements | Création et suppression d'abonnements d'affichage et d'évaluation filtrés (`SRS-NOY-ABO-*`) |
| P4 Statistiques | Par identifiant et par canal (`SRS-NOY-STA-*`) |
| P5 Événements | Flux sortant des événements (`SRS-NOY-JNL-04`) ; injection d'événements par un client (`SRS-NOY-JNL-12`) |
| P6 Enregistrement | Démarrer, arrêter, découper, filtres de capture, fenêtres déclenchées |
| P7 Émission | Armement, battement, liste blanche, soumission horodatée, écho, arrêt d'urgence, registre de galop |

---

## 4. Exigences

### 4.1 Processus et architecture

- **SRS-NOY-ARC-01** [A] Le démon `clemcand` DOIT s'exécuter indépendamment de tout client : la
  fermeture, le plantage ou la mise à jour d'un client n'interrompt ni ne perturbe un
  enregistrement en cours.
  *Source* : `EX-ARC-01`, `EX-ROB-03`, `D-02`. *Vérification* : celle d'`EX-ARC-01`.
- **SRS-NOY-ARC-02** [A] `libclemcan` DOIT offrir, sans démon, l'ouverture de canaux et
  l'enregistrement d'une trace, avec un contenu identique à celui du démon pour la même injection.
  *Source* : `EX-ARC-02`. *Vérification* : celle d'`EX-ARC-02`.
- **SRS-NOY-ARC-03** [A] Le code de `core/` NE DOIT dépendre que d'une liste blanche publiée :
  bibliothèque C standard, API du système, API des backends, bibliothèque TLS. Aucun décodage,
  aucune base, aucune expression, aucun protocole applicatif.
  *Source* : `EX-ARC-03`. *Vérification* : celle d'`EX-ARC-03`.
- **SRS-NOY-ARC-04** [A] Le noyau DOIT être le seul composant à ouvrir un canal Tier 1. Les
  canaux Tier 2 sont ouverts par le pont, lancé par le démon, qui ne communique qu'à travers I-INJ.
  *Source* : `D-15`, `D-17`, `D-36`. *Vérification* : analyse automatique des appels d'ouverture
  de socket CAN et de PCAN-Basic hors de `core/`, le pont, `bench/` et `tests/` exclus.
- **SRS-NOY-ARC-05** [A] Le nombre de canaux simultanés NE DOIT être limité par aucune constante
  ni structure de taille fixe ; les ressources d'un canal sont allouées à son ouverture ; à
  ressources épuisées, l'ouverture est refusée avec un message explicite.
  *Source* : `EX-BUS-01`. *Vérification* : celle d'`EX-BUS-01`.

### 4.2 Canaux et backends

- **SRS-NOY-BUS-01** [A] Le noyau DOIT gérer les backends Tier 1 : SocketCAN sous Linux (`peak_usb`
  avec PCAN-USB et PCAN-USB FD, `vcan`), PCAN-Basic sous Windows (mêmes matériels), et le bus
  virtuel ClemCAN sous Windows. Le Tier est attaché au couple (pilote, matériel) détecté à
  l'ouverture ; un pilote SocketCAN non listé relève du pont (Tier 2).
  *Source* : §5.1, `D-15`. *Vérification* : ouverture de chaque backend disponible ; couple et
  Tier exposés ; ⚙ PCAN-USB et PCAN-USB FD sous Linux et Windows.
- **SRS-NOY-BUS-02** [A] Chaque canal DOIT porter : débit nominal ; débit de phase de données et
  points d'échantillonnage (FD) ; mode (classique, FD ISO, FD non ISO) ; mode de contrôleur (écoute
  seule, acquittement, émission) ; protocole (aucun ou J1939) ; références et empreintes des bases
  et des règles associées, fournies par le client et transportées sans interprétation. Chaque
  paramètre est exposé par P2 et inscrit dans les métadonnées.
  *Source* : `EX-BUS-02`, `EX-JNL-03`. *Vérification* : celle d'`EX-BUS-02`.
- **SRS-NOY-BUS-03** [A] Le noyau DOIT recevoir et restituer à l'identique : trames 11 et 29 bits,
  requêtes distantes, trames FD (BRS, ESI, DLC 9 à 15), trames classiques de DLC 9 à 15 (DLC brut
  conservé), rapports d'erreur décodés selon le backend (classe, état du contrôleur, TEC et REC
  quand disponibles).
  *Source* : `EX-BUS-03`. *Vérification* : celle d'`EX-BUS-03`.
- **SRS-NOY-BUS-04** [A] Le Tier et le couple (pilote, matériel) de chaque canal DOIVENT être
  exposés par P1 et P2 et inscrits dans les métadonnées de la trace.
  *Source* : `EX-BUS-04`. *Vérification* : celle d'`EX-BUS-04`, hors IHM.
- **SRS-NOY-BUS-05** [A] Le noyau DOIT produire un événement d'avertissement, signalé par P5, en
  cas de configuration probablement incohérente : aucune trame valide dans les 2 s suivant
  l'ouverture, plus de 10 rapports d'erreur par seconde, ou passage en *error-passive*. Seuils
  configurables par canal.
  *Source* : `EX-BUS-05`. *Vérification* : celle d'`EX-BUS-05`.
- **SRS-NOY-BUS-06** [A] Les changements d'état du contrôleur (*error-active*, *error-warning*,
  *error-passive*, bus-off) DOIVENT être détectés, horodatés, inscrits comme événements et
  signalés par P5.
  *Source* : `EX-BUS-06`. *Vérification* : celle d'`EX-BUS-06`.
- **SRS-NOY-BUS-07** [C] Une politique de reprise après bus-off (aucune, manuelle, automatique
  après délai) DOIT être configurable par canal ; défaut fixé par `O-07`. Chaque reprise est un
  événement.
  *Source* : `EX-BUS-07`. *Vérification* : celle d'`EX-BUS-07`.
- **SRS-NOY-BUS-08** [A] Le noyau DOIT offrir un **backend de simulation**, réservé aux tests,
  capable de produire sur commande des trames numérotées, des rapports d'erreur, des changements
  d'état (bus-off compris) et des débordements, sans débit binaire. Il est désigné comme tel dans
  les métadonnées, avec le niveau « simulation » à la place du Tier, et n'est jamais Tier 1.
  *Source* : dérivée — les vérifications auto d'`EX-BUS-03`, `EX-BUS-05`, `EX-BUS-06`,
  `EX-BUS-07` et `EX-CAP-04` exigent des états que `vcan` ne sait pas produire.
  *Vérification* : chaque type de stimulus produit l'événement attendu.

### 4.3 Capture, horodatage et pertes

- **SRS-NOY-CAP-01** [A] L'horodatage DOIT provenir de la source la plus précise disponible
  (matériel, puis noyau du système, puis espace utilisateur) ; la source effective est inscrite
  par canal dans les métadonnées. Pour un canal Tier 2, la source est celle que déclare le pont,
  qui livre des horodatages déjà convertis dans la base monotone de l'hôte (`SRS-PNT-02`).
  *Source* : `EX-CAP-01`. *Vérification* : celle d'`EX-CAP-01`.
- **SRS-NOY-CAP-02** [A] Tout horodatage DOIT être exprimé dans la base monotone de l'hôte, commune
  à tous les canaux ; la correspondance avec l'UTC est relevée au démarrage de l'enregistrement et
  inscrite dans les métadonnées. Aucune durée n'est calculée entre deux horodatages de sources
  différentes sans conversion préalable dans la base commune.
  *Source* : `EX-CAP-02`, `EX-CAP-03`. *Vérification* : celle d'`EX-CAP-02`.
- **SRS-NOY-CAP-03** [A] Quand un canal fournit un horodatage matériel, le noyau DOIT estimer en
  continu le décalage et la dérive de son horloge par rapport à l'horloge monotone de l'hôte, et
  inscrire l'incertitude résultante dans les métadonnées.
  *Source* : `EX-CAP-03`. *Vérification* : celle d'`EX-CAP-03`.
- **SRS-NOY-CAP-04** [A] Toute perte DOIT produire un événement horodaté à sa détection, être
  signalée par P5 et comptée : exactement pour le tampon de socket (`SO_RXQ_OVFL`), l'anneau
  interne et l'écriture disque ; comme « perte de nombre inconnu », avec son intervalle, pour un
  débordement de FIFO matérielle signalé par simple drapeau.
  *Source* : `EX-CAP-04`, `EX-PRF-03`. *Vérification* : celle d'`EX-CAP-04`.

### 4.4 Journal natif

- **SRS-NOY-JNL-01** [A] L'enregistrement DOIT s'écrire en **ajout seul**, sans compression en
  cours d'écriture, synchronisé sur le support au moins une fois par fenêtre d'écriture (défaut
  100 ms, de 10 ms à 1 s). À la relecture, une fin abrupte est détectée et signalée par un
  événement portant la borne de perte.
  *Source* : `EX-JNL-01`, `EX-ROB-01`, `D-04`. *Vérification* : celle d'`EX-JNL-01`.
- **SRS-NOY-JNL-02** [A] L'index `.tcx` DOIT être reconstructible intégralement depuis le journal
  seul ; sa perte ou sa corruption n'est jamais une perte de données.
  *Source* : `EX-JNL-02`. *Vérification* : celle d'`EX-JNL-02`.
- **SRS-NOY-JNL-03** [A] Le journal DOIT porter les métadonnées : version de format, fenêtre
  d'écriture, version de ClemCAN, correspondance UTC ; par canal : paramètres de
  `SRS-NOY-BUS-02`, backend, couple, Tier, source d'horodatage et son incertitude, filtres de
  capture et leur nature, mode de contrôleur, garanties non assurées d'un canal Tier 2 ; origine
  pour une trace importée (format, fichier, empreinte, nombre de trames déplacées) ; par trame, le
  sens. [C] Source de l'instant des trames émises.
  *Source* : `EX-JNL-03`, `EX-FMT-06`, `EX-SAF-01`, `EX-REJ-06`, `D-17`. *Vérification* : celle
  d'`EX-JNL-03`.
- **SRS-NOY-JNL-04** [A] Le journal DOIT porter des **événements** typés, aux champs documentés,
  écrits par le noyau : au moins perte, perte de nombre inconnu, rapport d'erreur, changement
  d'état, avertissement de configuration, reconfiguration (avec durée d'interruption), découpage,
  disque saturé, évaluation abandonnée (`SRS-NOY-ABO-02`), fin de canal (`SRS-NOY-INJ-02`). La
  fin abrupte n'est pas écrite : elle est synthétisée à la relecture (`SRS-NOY-JNL-01`), avec des
  champs documentés de même manière.
  *Source* : §2 « Événement ». *Vérification* : chaque type écrit relu par `clemcan trace info` ;
  fin abrupte synthétisée à la relecture d'un journal tronqué.
- **SRS-NOY-JNL-05** [B] Le découpage automatique DOIT être configurable par taille, durée et
  nombre de trames, le premier critère atteint déclenchant le découpage, avec base de temps
  continue entre fichiers.
  *Source* : `EX-JNL-04`. *Vérification* : celle d'`EX-JNL-04`.
- **SRS-NOY-JNL-06** [A] `libclemcan` DOIT lire un ensemble de fichiers consécutifs comme une
  seule trace et y naviguer (saut à un instant, à la *n*-ième occurrence d'un identifiant) par son
  index, en moins d'une seconde par saut au 95ᵉ centile sur 200 millions de trames, avec une
  mémoire résidente ≤ 500 Mo indépendante de la taille de la trace.
  *Source* : `EX-JNL-05`. *Vérification* : celle d'`EX-JNL-05`.
- **SRS-NOY-JNL-07** [A] `libclemcan` DOIT être la **seule** implémentation de l'écriture et de la
  lecture du journal natif ; les imports, extractions et conversions de la couche 2 l'utilisent.
  *Source* : dérivée — `EX-FMT-02`, `EX-ANA-02` ; une seconde implémentation du format créerait
  deux comportements à maintenir conformes. *Vérification* : analyse automatique : ni la
  signature ni la disposition du format `.tcj` n'apparaissent hors de `core/` (`tests/` exclus),
  en lecture comme en écriture.
- **SRS-NOY-JNL-08** [A] Le format du journal et de l'index DOIT être documenté et versionné ; le
  lecteur lit toutes les versions publiées.
  *Source* : §5.4, `O-02`. *Vérification* : corpus d'un fichier par version publiée, relu.
- **SRS-NOY-JNL-09** [A] Le noyau NE DOIT jamais rouvrir en écriture un journal clos.
  *Source* : §2 « Trace ». *Vérification* : tentative d'ajout à un journal clos refusée ;
  empreinte inchangée.
- **SRS-NOY-JNL-10** [A] L'espace disque DOIT être surveillé : avertissement à un seuil
  configurable ; sous la réserve (défaut 256 Mo), arrêt propre avec événement de fin écrit dans la
  réserve, trames non écrites comptées et signalées par P5 et dans le journal du système.
  *Source* : `EX-ROB-04`. *Vérification* : celle d'`EX-ROB-04`.
- **SRS-NOY-JNL-11** [B] La politique de saturation **rotation** DOIT être configurable : effacement
  des plus anciens fichiers de l'enregistrement en cours uniquement, chaque effacement étant un
  événement.
  *Source* : `EX-ROB-05`. *Vérification* : celle d'`EX-ROB-05`.
- **SRS-NOY-JNL-12** [B] Un client autorisé DOIT pouvoir **injecter** par P5, dans
  l'enregistrement en cours, un événement typé produit par la couche 2 (déclenchement, violation,
  levée ou retombée d'alarme), horodaté dans la base de la trace. Le noyau l'inscrit sans
  l'interpréter. Sans enregistrement en cours, l'injection est refusée avec un message explicite.
  *Source* : `EX-TRG-04`, `EX-EXP-06`, `EX-CHK-03`, `D-32` ; dérivée — ces exigences inscrivent
  dans la trace des événements que seule la couche 2 sait calculer, alors que seul le noyau écrit
  le journal. *Vérification* : événement injecté pendant un enregistrement, relu à son
  horodatage ; injection sans enregistrement refusée.

### 4.5 Statistiques de transport et charge

- **SRS-NOY-STA-01** [A] Le noyau DOIT tenir, par canal et par identifiant : nombre d'occurrences ;
  période moyenne, minimale et maximale ; gigue de période (écart-type sur fenêtre glissante,
  défaut 10 s) ; date de dernière réception ; masque des octets changés depuis la réception
  précédente ; apparition et disparition (absence depuis plus de 3 × la période mesurée et au
  moins 1 s) ; histogramme des temps de cycle (intervalles entre occurrences successives, classes
  publiées).
  *Source* : `EX-VUE-02`, `EX-VUE-04`, `D-16`. *Vérification* : celle d'`EX-VUE-02` ;
  histogramme d'un identifiant de période connue conforme à l'attendu.
- **SRS-NOY-STA-02** [A] Le noyau DOIT tenir, par canal : trames par seconde, charge sur fenêtre
  glissante de 1 s, compteurs de rapports d'erreur, de changements d'état et de bus-off, pertes.
  Les rapports d'erreur ne comptent ni comme trames ni dans la charge. Sans débit connu, la charge
  n'est pas calculée.
  *Source* : `EX-VUE-04`. *Vérification* : celle d'`EX-VUE-04` ; rapports d'erreur injectés sans
  effet sur les trames par seconde ni sur la charge.
- **SRS-NOY-STA-03** [A] La durée d'une trame DOIT être calculée exactement depuis son contenu
  selon ISO 11898-1 : bits de trame, CRC calculé, bourrage dynamique, bits de bourrage fixes et
  compteur de bourrage en FD ISO, 3 bits d'intertrame ; phases nominale et de données à leurs
  débits, commutation aux points d'échantillonnage de BRS et du délimiteur de CRC. En FD non ISO,
  même calcul selon la spécification CAN FD 1.0 de Bosch, sans compteur de bourrage. Cette
  implémentation est **unique** et exposée par I-LIB pour le calcul hors ligne.
  *Source* : `EX-VUE-04`, `D-37` ; unicité dérivée — l'injecteur (§6.1 de `SPEC.md`), le contrôle
  de charge du rejeu (`EX-REJ-03`) et la charge d'une trace importée doivent donner le même
  résultat.
  *Vérification* : celle d'`EX-VUE-04`.
- **SRS-NOY-STA-04** [C] Le noyau DOIT tenir, par canal, l'ensemble des identifiants reçus depuis
  l'ouverture sans avoir été émis par ClemCAN.
  *Source* : `EX-SAF-04`. *Vérification* : identifiant injecté par un nœud simulé présent dans
  l'ensemble ; identifiant émis par ClemCAN absent.

### 4.6 Abonnements et filtres

- **SRS-NOY-ABO-01** [A] Un **abonnement d'affichage** DOIT filtrer par canal, identifiant et
  masque. Quand son consommateur ne suit pas, le noyau le **déleste** (§2 de `SPEC.md`) : les
  trames abandonnées sont comptées, le compteur de délestage est exposé et signalé à l'abonné,
  l'enregistrement n'est pas affecté. Le noyau ne décime pas.
  *Source* : `EX-CAP-05`, `D-16`, `D-39`. *Vérification* : celle d'`EX-CAP-05`.
- **SRS-NOY-ABO-02** [A] Un **abonnement d'évaluation** DOIT filtrer par canal, identifiant et
  masque, n'être jamais délesté, et porter une file bornée dont la taille en trames est fixée à
  sa création (défaut 65 536). File pleine : abandon des trames suivantes de cet abonnement
  jusqu'à ce que la file soit vidée de moitié ; l'abandon est un événement portant l'intervalle
  et le nombre de trames, signalé à l'abonné et inscrit dans l'enregistrement en cours.
  *Source* : `D-24`, `EX-CAP-05`. *Vérification* : abonné ralenti ; abandon déclaré égal au
  nombre de trames non remises ; intervalle exact ; enregistrement complet.
- **SRS-NOY-ABO-03** [A] Aucun abonné NE DOIT exercer de contre-pression : un abonné lent n'affecte
  ni l'enregistrement, ni les autres abonnés.
  *Source* : `EX-CAP-05`, `D-14`, `D-24`. *Vérification* : celle d'`EX-CAP-05`, avec un second
  abonné rapide dont aucune trame n'est perdue.
- **SRS-NOY-ABO-04** [A] Au profil nominal, une trame DOIT être remise à un abonnement
  d'affichage local en moins de 50 ms au 99ᵉ centile après son horodatage de réception.
  *Source* : dérivée — part du noyau dans le budget de 100 ms d'`EX-VUE-06`.
  *Vérification* : instrumentation au profil nominal sur bus virtuel pendant 10 min.
- **SRS-NOY-ABO-05** [B] Les **filtres de capture** (listes canal, identifiant, masque, étendu,
  FD) DOIVENT être appliqués au niveau matériel ou système quand le backend le permet, sinon par
  le noyau ; la nature effective est exposée et inscrite dans les métadonnées.
  *Source* : `EX-EXP-05`, `EX-EXP-01`. *Vérification* : celle d'`EX-EXP-05`.

### 4.7 Enregistrement déclenché

- **SRS-NOY-TRG-01** [B] Le noyau DOIT tenir, par enregistrement déclenché, un tampon circulaire
  pré-déclenchement préalloué, limité en durée et/ou en taille (la limite la plus restrictive
  s'applique). Une taille maximale est toujours fixée à la création : si seule la durée est
  donnée, la taille en est déduite au débit saturé du canal.
  *Source* : `EX-TRG-01` ; plafond dérivé — le tampon est préalloué (contraintes de conception de
  cette SRS : aucune allocation dans la boucle de réception). *Vérification* : celle d'`EX-TRG-01` ; tampon créé avec une durée seule,
  taille fixée exposée.
- **SRS-NOY-TRG-02** [B] Sur commande d'un client portant un instant de déclenchement (base de
  temps de la trace), le noyau DOIT écrire un fichier contenant les trames de
  [*t* − pré ; *t* + post] ; un déclenchement reçu pendant la fenêtre post-déclenchement la
  prolonge dans le même fichier ; une fenêtre dépassant la durée maximale (défaut 10 min) se
  poursuit dans un nouveau fichier, base de temps continue, la coupure étant un événement des deux
  fichiers.
  *Source* : `EX-TRG-03`, `EX-TRG-04`. *Vérification* : celles d'`EX-TRG-03` et d'`EX-TRG-04`.
- **SRS-NOY-TRG-03** [B] Si une commande arrive alors que *t* − pré a déjà quitté le tampon, la
  fenêtre DOIT être écrite depuis la plus ancienne trame disponible, et le retard inscrit comme
  événement compté.
  *Source* : dérivée — §5.11, `EX-CAP-05` : la commande vient de la couche 2 et peut arriver en
  retard ; la vérification d'`EX-TRG-04` admet ce cas (*pré* effectivement retenu).
  *Vérification* : commande retardée artificiellement au-delà de la durée pré ; fenêtre tronquée
  et événement présent.

### 4.8 Émission et garde-fous

- **SRS-NOY-TX-01** [A] Tout canal DOIT s'ouvrir en mode *listen-only*. Si le matériel ne l'offre
  pas, l'ouverture exige une confirmation explicite transmise par le client, relayée au pont pour
  un canal Tier 2 (`SRS-NOY-INJ-01`), et le fait est signalé et inscrit. Le mode acquittement est une option explicite par canal, inscrite dans les
  métadonnées, distincte de l'armement.
  *Source* : `EX-SAF-01`, `D-20`. *Vérification* : celle d'`EX-SAF-01`.
- **SRS-NOY-TX-02** [C] Toute émission, quel que soit le `TxSource`, DOIT passer par
  l'**ordonnanceur unique** du noyau, qui applique armement, liste blanche, journalisation et arrêt
  d'urgence. Une soumission sur un canal non armé est refusée avec un message explicite et un
  événement.
  *Source* : §4, `D-17`, `EX-SAF-02`. *Vérification* : celles d'`EX-SAF-03` et d'`EX-SAF-07` ;
  émission sans armement refusée et tracée.
- **SRS-NOY-TX-03** [C] L'ordonnanceur DOIT accepter des trames soumises **à l'avance** avec un
  instant d'émission absolu dans l'horloge monotone de l'hôte, et les émettre à cet instant, sans
  accumulation de délais. Sur un canal Tier 1 sous Linux, l'erreur d'émission respecte
  `EX-REJ-04`.
  *Source* : `EX-REJ-04`, `EX-PRF-04` ; soumission anticipée dérivée — la couche 2, en Python, ne peut pas
  tenir 1 ms au 99ᵉ centile en émettant à l'instant. *Vérification* : celle d'`EX-REJ-04`.
- **SRS-NOY-TX-04** [C] L'armement DOIT nommer le canal, être lié à la connexion cliente qui le
  demande, n'être jamais mémorisé ni restauré. Le démon constate la perte de cette connexion à sa
  fermeture (fermeture ou plantage du client) ou, à défaut, au terme de 1 s sans battement ; il
  désarme alors le canal et arrête toute émission dans le délai de `SRS-NOY-TX-07`, compté depuis
  ce constat. Le passage en mode émission est une reconfiguration inscrite avec sa durée
  d'interruption de capture. L'armement d'un canal Tier 2 est refusé avec un message explicite.
  *Source* : `EX-SAF-02`, `D-20`, `D-30`, `D-33`. *Vérification* : celle d'`EX-SAF-02`.
- **SRS-NOY-TX-05** [C] La liste blanche d'identifiants émissibles DOIT être tenue par canal, **vide
  par défaut**, donnée à l'armement ou dans la configuration du démon ; hors liste, l'émission est
  refusée avec un message explicite et un événement ; une entrée générique explicite autorise tout
  identifiant et est signalée à l'armement.
  *Source* : `EX-SAF-03`, `D-28`. *Vérification* : celle d'`EX-SAF-03`.
- **SRS-NOY-TX-06** [C] La soumission d'un identifiant présent dans l'ensemble de
  `SRS-NOY-STA-04` DOIT être refusée, sauf drapeau d'acceptation explicite fourni par le client
  pour cet identifiant.
  *Source* : `EX-SAF-04`. *Vérification* : refus sans drapeau, émission avec drapeau.
- **SRS-NOY-TX-07** [C] L'**arrêt d'urgence**, accessible par P7 à tout client autorisé (armé ou
  non), DOIT couper toute émission en moins de 50 ms entre la réception de la commande et la
  dernière trame émise, files d'émission du système et du matériel vidées, puis désarmer tous les
  canaux.
  *Source* : `EX-SAF-05`, `D-20` ; désarmement dérivé — après un arrêt d'urgence, reprendre
  l'émission doit exiger un nouvel armement délibéré (`EX-SAF-02`). *Vérification* : celle
  d'`EX-SAF-05` ; tous les canaux désarmés après l'arrêt.
- **SRS-NOY-TX-08** [C] L'état armé de chaque canal DOIT être exposé par P2 et notifié par P5 à
  chaque changement.
  *Source* : `EX-SAF-05`. *Vérification* : armement puis désarmement ; notifications reçues par
  un second client.
- **SRS-NOY-TX-09** [C] Toute trame émise DOIT être journalisée, sans possibilité de désactivation ;
  si la journalisation est impossible (disque plein, journal fermé), l'émission est refusée.
  *Source* : `EX-SAF-07`. *Vérification* : celle d'`EX-SAF-07`.
- **SRS-NOY-TX-10** [C] Pour chaque trame émise, le noyau DOIT restituer au client l'instant réel
  (écho horodaté quand le backend le fournit, sinon instant de soumission au backend), sa source,
  et tout échec (refus du backend ; absence d'écho dans un délai configurable, pour un backend qui
  fournit l'écho).
  *Source* : `EX-REJ-05`, `EX-REJ-06`. *Vérification* : celle d'`EX-REJ-05`.
- **SRS-NOY-TX-11** [C] Le démon DOIT tenir un **registre persistant de galops d'essai**. Pour un
  galop, il ouvre lui-même un bus virtuel avec les paramètres des canaux cibles, sur lequel le
  client conduit le galop comme un rejeu. Il constate lui-même les émissions refusées ou échouées,
  la conformité à la liste blanche, l'égalité entre trames émises et nombre annoncé par le client,
  et la charge. Il n'inscrit l'empreinte fournie par le client que si ces constats et la
  déclaration de conformité aux règles de contrôle faite par le client sont tous favorables. Il
  refuse toute émission réelle d'un rejeu dont l'empreinte n'y figure pas.
  *Source* : `EX-SAF-06`, `D-26`, `D-31`. *Vérification* : celle d'`EX-SAF-06`.
- **SRS-NOY-TX-12** [C] L'armement DOIT être refusé aux clients distants, sauf autorisation
  explicite dans la configuration du démon.
  *Source* : `EX-SEC-02`. *Vérification* : armement distant refusé par défaut, accepté après
  configuration.

### 4.9 Protocole et sécurité

- **SRS-NOY-PRO-01** [A] I-PROT DOIT être documenté et versionné, couvrir les services P1 à P7, et
  permettre à un client écrit dans un autre langage de s'y brancher sans modifier le noyau ; un
  client annonçant une version majeure incompatible est refusé avec un message explicite.
  *Source* : `EX-ARC-04`. *Vérification* : celle d'`EX-ARC-04`.
- **SRS-NOY-SEC-01** [A] Par défaut, le démon N'OUVRE AUCUN port réseau en écoute, en TCP comme
  en UDP ; seule l'option d'accès distant (`SRS-NOY-SEC-02`) en ouvre un.
  *Source* : `EX-ARC-05`, `EX-SEC-01`, `D-35`. *Vérification* : celle d'`EX-ARC-05` ; celle
  d'`EX-SEC-01` pour le démon.
- **SRS-NOY-SEC-02** [B] L'accès distant, activé explicitement, DOIT exiger TLS 1.3 avec
  authentification mutuelle par certificats ; toute autre connexion est refusée.
  *Source* : `EX-SEC-02`. *Vérification* : celle d'`EX-SEC-02`.
- **SRS-NOY-SEC-03** [A] Le socket de contrôle et la mémoire partagée DOIVENT n'être accessibles
  qu'aux comptes autorisés (Linux : propriétaire et groupe `clemcan`, droits `0660` ; Windows :
  liste de contrôle d'accès configurée). [C] L'armement exige en outre le groupe `clemcan-tx`
  (Linux) ou un compte désigné (Windows), à défaut le compte du démon.
  *Source* : `EX-SEC-04`, `D-28`. *Vérification* : celle d'`EX-SEC-04`.
- **SRS-NOY-SEC-04** [A] Le noyau NE DOIT établir aucune connexion sortante ni émettre aucune
  télémétrie.
  *Source* : `EX-SEC-03`, `EX-POR-04`. *Vérification* : celle d'`EX-SEC-03`.

### 4.10 Robustesse, performance, portabilité

- **SRS-NOY-ROB-01** [B] Le démon DOIT pouvoir démarrer avec le système et reprendre un
  enregistrement selon sa configuration, sans intervention, dans un nouveau fichier.
  *Source* : `EX-ROB-02`. *Vérification* : celle d'`EX-ROB-02`.
- **SRS-NOY-PRF-01** [A] Sur la machine de référence x86-64, le profil de charge nominal (§6.1)
  DOIT être enregistré 60 min sans perte, avec une mémoire résidente stable au sens
  d'`EX-PRF-01`.
  *Source* : `EX-PRF-01`. *Vérification* : celle d'`EX-PRF-01`.
- **SRS-NOY-PRF-02** [A] Sur la machine de référence ARM64, le demi-profil DOIT être enregistré
  60 min sans perte non déclarée ; le débit maximal sans perte est mesuré et publié.
  *Source* : `EX-PRF-02`. *Vérification* : celle d'`EX-PRF-02`.
- **SRS-NOY-POR-01** [A] Le noyau DOIT se construire et passer sa suite auto sur Linux x86-64,
  Linux ARM64 et Windows x86-64, et le démon DOIT être livrable seul, sans composant graphique ni
  environnement Python. Ce livrable n'inclut pas le pont (`SRS-NOY-INJ-01`).
  *Source* : `EX-POR-01`, `EX-POR-03`, `D-36`. *Vérification* : celles d'`EX-POR-01` et
  d'`EX-POR-03`.

### 4.11 Pont Tier 2

- **SRS-NOY-INJ-01** [A] À l'ouverture d'un canal Tier 2 par P2, le démon DOIT lancer le pont,
  lui transmettre par I-INJ la configuration du canal et, le cas échéant, la confirmation
  d'ouverture sans écoute seule (`SRS-NOY-TX-01`), puis présenter le canal comme tout autre
  canal. Si le pont est absent du livrable, l'ouverture est refusée avec un message explicite.
  *Source* : `D-36`, `EX-POR-03`, `EX-SAF-01`. *Vérification* : ouverture par P2 d'un canal sur le
  backend `virtual` de python-can ; pont lancé, canal exposé avec son Tier ; celle d'`EX-POR-03`.
- **SRS-NOY-INJ-02** [A] Le démon DOIT détecter sans délai la fin ou le plantage du pont, et la
  perte de la liaison I-INJ, et les inscrire comme événement de fin de canal, signalé par P5. Les
  autres canaux et l'enregistrement n'en sont pas affectés.
  *Source* : `EX-CAP-04`, `EX-ARC-01`, `D-36`. *Vérification* : `kill -9` du pont pendant un
  enregistrement multicanal ; événement de fin de canal présent ; autres canaux complets
  (séquences numérotées).

---

## 5. Contraintes de conception

- C11, `-Wall -Wextra -Werror`, aucune fuite ni comportement indéfini sous sanitizers (adresse,
  comportement indéfini, fils d'exécution), tout retour d'erreur vérifié.
- Boucle de réception : lecture par lots, aucune allocation, aucun verrou, aucun appel bloquant
  autre que l'attente de réception ; tampons dimensionnés à l'ouverture du canal.
- ABI de `libclemcan` stable et versionnée ; seul le module `clemcan/core_api` la connaît.
- Aucune nouvelle dépendance sans fiche de décision (bibliothèque TLS comprise).

## 6. Points ouverts

| # | Point | Renvoi |
|---|---|---|
| NOY-1 | Encodage d'I-PROT, contre-pression locale, encapsulation distante | `O-01` |
| NOY-2 | Disposition binaire du journal et de l'index | `O-02` |
| NOY-3 | Politique de reprise après bus-off par défaut | `O-07` |
| NOY-4 | Choix de la bibliothèque TLS (licence, ARM, Windows) | fiche à écrire avant le palier B |
| NOY-5 | Formule de durée de trame pour le CAN FD **non ISO** (pas de compteur de bourrage, CRC différent) : `EX-VUE-04` ne la précise pas | clos par `D-37` |
| NOY-6 | Le registre de galop fait confiance à l'empreinte fournie par le client : il protège contre l'erreur, pas contre la malveillance d'un compte autorisé | à documenter |
| NOY-7 | L'injection d'événements par les clients (`SRS-NOY-JNL-12`) n'est pas citée dans la liste des services d'`EX-ARC-04` | clos sans amendement : « événements » couvre le flux sortant et l'injection |
