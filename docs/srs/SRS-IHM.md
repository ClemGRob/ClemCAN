# SRS-IHM — Interface graphique (`clemcan/ui/`)

> Spécification des exigences logicielles du composant **interface**.
> Version 1.1 — 2026-11-25. Dérivée de `SPEC.md` v1.3, qui prévaut en cas d'écart.
> Conventions : `docs/srs/README.md`.

---

## 1. Objet et périmètre

L'interface Qt (PySide6) est une **coquille fine** : elle présente les modèles de la couche
d'analyse et déclenche ses commandes. Elle est jetable et remplaçable : son plantage n'affecte ni
l'enregistrement ni les données.

**Elle ne fait pas** : aucune logique métier, aucun décodage, aucun calcul de statistique, aucun
accès au protocole du noyau, aucune écriture de fichier hors d'une commande du registre.

## 2. Références

`SPEC.md` v1.3 ; `SRS-ANA.md` (I-API) ; `SRS-NOY.md`.

## 3. Interfaces

| Id | Interface | Sens | Description |
|---|---|---|---|
| **I-API** | API Python de la couche d'analyse | requise | Modèles, registre de commandes, notifications. Seule voie d'accès aux données et au noyau. |
| **I-PREF** | Préférences utilisateur | fournie | Langue, raccourcis : propres à l'utilisateur, hors session. |

---

## 4. Exigences

### 4.1 Architecture

- **SRS-IHM-ARC-01** [A] L'interface NE DOIT contenir aucune logique métier ; elle lit les modèles
  et déclenche les commandes du registre (`SRS-ANA-ARC-02`), et n'accède jamais au noyau ni à
  `core_api` directement.
  *Source* : `EX-ARC-06`, `D-27`, §4 de `SPEC.md`. *Vérification* : celle d'`EX-ARC-06` ; analyse
  automatique des imports de `ui/`.
- **SRS-IHM-ARC-02** [A] La boucle d'événements NE DOIT jamais être bloquée plus de 200 ms ; toute
  opération longue, dont l'ouverture de trace et le chargement de base, s'exécute hors du fil
  principal avec indication de progression.
  *Source* : `EX-VUE-06`. *Vérification* : celle d'`EX-VUE-06`.
- **SRS-IHM-ARC-03** [A] Quand le noyau déleste un flux d'affichage, l'interface DOIT le signaler
  sur les vues concernées, qui gèlent ou s'appauvrissent sans bloquer l'interface. Elle n'ouvre
  elle-même aucun abonnement : elle passe par les modèles de la couche d'analyse.
  *Source* : §4 de `SPEC.md`, `EX-CAP-05`, `D-39`. *Vérification* : celle d'`EX-CAP-05`, avec
  l'interface en rendu hors écran ; délestage signalé (fumée) ; boucle d'événements jamais bloquée
  plus de 200 ms.
- **SRS-IHM-ARC-04** [A] L'interface DOIT pouvoir se reconnecter à un démon en cours
  d'enregistrement et restituer son état (canaux, enregistrement, pertes ; [B] alarmes) sans
  l'interrompre.
  *Source* : dérivée — `EX-ARC-01` rend l'interface jetable : la relancer doit retrouver le
  travail en cours. *Vérification* : interface tuée et relancée pendant un enregistrement ; état
  restitué ; trace continue.
- **SRS-IHM-ARC-05** [A] L'interface DOIT pouvoir s'exécuter en rendu hors écran pour les tests de
  fumée et les exports d'image, sans serveur graphique.
  *Source* : `D-13`, `D-23`, `EX-AUT-04`. *Vérification* : suite de fumée en rendu hors écran.

### 4.2 Canaux et santé

- **SRS-IHM-BUS-01** [A] Le Tier et le couple (pilote, matériel) de chaque canal DOIVENT être
  affichés.
  *Source* : `EX-BUS-04`. *Vérification* : IHM.
- **SRS-IHM-BUS-02** [A] Les avertissements de configuration, les pertes et toute dégradation
  au-delà de l'enveloppe DOIVENT être signalés de façon visible et persistante jusqu'à acquittement
  par l'utilisateur.
  *Source* : `EX-BUS-05`, `EX-CAP-04`, `EX-PRF-03` ; persistance dérivée — un signalement fugace
  passe inaperçu pendant un enregistrement sans opérateur devant l'écran. *Vérification* : IHM,
  perte provoquée sur bus virtuel.
- **SRS-IHM-BUS-03** [A] Le mode de contrôleur de chaque canal (écoute seule, acquittement,
  émission) DOIT être affiché. L'interface DOIT demander une confirmation explicite à l'ouverture
  d'un canal dont le matériel n'offre pas l'écoute seule, à l'activation du mode acquittement, et à
  la restauration d'un mode acquittement depuis une session ou un profil ; sans confirmation, le
  canal ne s'ouvre pas, ou s'ouvre en écoute seule dans le cas d'un profil.
  *Source* : `EX-SAF-01`, `EX-SES-04`. *Vérification* : IHM ; logique vérifiée par celles
  d'`EX-SAF-01` et d'`EX-SES-04` en ligne de commande.

### 4.3 Vues

- **SRS-IHM-VUE-01** [A] La **trace déroulante** DOIT offrir un défilement continu, des colonnes
  configurables, la pause et le dépliage des signaux d'une trame.
  *Source* : `EX-VUE-01`. *Vérification* : IHM.
- **SRS-IHM-VUE-02** [A] La **vue groupée** DOIT afficher une ligne par identifiant mise à jour sur
  place, avec période mesurée (moyenne, minimum, maximum, gigue), compteur d'occurrences, date de
  dernière réception et mise en évidence des octets changés.
  *Source* : `EX-VUE-02`. *Vérification* : IHM.
- **SRS-IHM-VUE-03** [A] La **fenêtre de valeurs** DOIT afficher la valeur courante au format
  choisi (brut, physique, libellé), l'âge de la dernière mise à jour, le franchissement de bornes et
  le regroupement libre.
  *Source* : `EX-VUE-03`. *Vérification* : IHM.
- **SRS-IHM-VUE-04** [A] La vue **statistiques et santé** DOIT afficher les grandeurs
  d'`EX-VUE-04`, indiquer la sous-estimation possible de la charge en présence d'erreurs, et ne
  pas afficher de charge sans débit connu.
  *Source* : `EX-VUE-04`. *Vérification* : IHM.
- **SRS-IHM-VUE-05** [B] La vue d'**analyse sans base** DOIT présenter bits basculants, octets
  changeants, propositions de compteur et de contrôle, corrélation entre un intervalle et les
  changements observés, et permettre de créer un signal candidat depuis une sélection de bits,
  injecté dans la base en cours d'édition.
  *Source* : `EX-VUE-05`. *Vérification* : IHM.
- **SRS-IHM-VUE-06** [A] Les vues DOIVENT être des fenêtres ancrables, disposables sur plusieurs
  écrans ; leur disposition est enregistrée dans la session et restituée.
  *Source* : `EX-VUE-07`. *Vérification* : celle d'`EX-VUE-07`.

### 4.4 Graphe

- **SRS-IHM-GRA-01** [A] Le graphe DOIT tracer chaque signal dans son mode (escalier par défaut,
  ligne interpolée, points), avec surimpression optionnelle des instants d'échantillonnage réels et
  interruption de tracé sur NA.
  *Source* : `EX-GRA-01`, `EX-GRA-07`, `D-07`. *Vérification* : IHM ; logique vérifiée par
  `SRS-ANA-GRA-01` et `SRS-ANA-GRA-02`.
- **SRS-IHM-GRA-02** [A] Le graphe DOIT offrir plusieurs axes verticaux, échelles automatiques ou
  fixées, courbes empilées ou superposées, couleurs par signal.
  *Source* : `EX-GRA-03`. *Vérification* : IHM.
- **SRS-IHM-GRA-03** [A] Au moins deux curseurs DOIVENT afficher valeurs et écarts Δt, Δy.
  *Source* : `EX-GRA-04`. *Vérification* : IHM.
- **SRS-IHM-GRA-04** [A] La sélection DOIT être synchronisée dans les deux sens entre graphe, trace
  et repères.
  *Source* : `EX-GRA-05`. *Vérification* : IHM.
- **SRS-IHM-GRA-05** [A] Pendant un zoom sur une heure de trace au profil nominal, la boucle
  d'événements NE DOIT jamais être bloquée plus de 200 ms.
  *Source* : `EX-VUE-06` ; scénario de zoom dérivé — `O-03` retient la latence de zoom comme
  critère de choix de la bibliothèque de tracé. *Vérification* : sonde de la boucle d'événements
  en rendu hors écran sur la machine de référence x86-64, scénario publié dans `bench/`.
- **SRS-IHM-GRA-06** [B] Les signaux énumérés DOIVENT être tracés avec leurs libellés en ordonnée.
  *Source* : `EX-GRA-08`. *Vérification* : IHM.
- **SRS-IHM-GRA-07** [B] Le graphe DOIT être exportable en PNG et SVG aux dimensions demandées, et
  ses séries affichées en CSV (`SRS-ANA-GRA-08`), depuis l'interface. L'IHM fournit la commande
  d'export d'image au registre par le point d'extension de `SRS-ANA-ARC-06`, ce qui la rend
  exécutable en ligne de commande par rendu hors écran.
  *Source* : `EX-GRA-09`, `EX-ARC-06`. *Vérification* : celle d'`EX-GRA-09`.

### 4.5 Éditeur de base

- **SRS-IHM-DB-01** [B] L'éditeur DOIT présenter la matrice de bits (grille de 8 bits par octet),
  avec placement et redimensionnement à la souris, bit de départ affiché selon la convention Intel
  ou Motorola, recouvrements signalés en temps réel.
  *Source* : `EX-DB-05`. *Vérification* : IHM ; logique vérifiée par `SRS-ANA-DB-06`.
- **SRS-IHM-DB-02** [B] L'éditeur DOIT offrir annuler et refaire, opérations par lot, recherche,
  comparaison et fusion avec prévisualisation, et afficher les résultats de validation (bloquants
  et avertissements) en désignant l'objet.
  *Source* : `EX-DB-04`, `EX-DB-06`, `EX-DB-07`, `EX-DB-08`, `EX-DB-11`. *Vérification* : IHM.

### 4.6 Expressions, alarmes, analyse

- **SRS-IHM-EXP-01** [B] Toute saisie d'expression DOIT signaler une erreur à sa position, avec sa
  cause, avant validation.
  *Source* : `EX-EXP-04`. *Vérification* : IHM.
- **SRS-IHM-EXP-02** [B] La nature effective des filtres de capture (matériel, système, ClemCAN)
  DOIT être affichée.
  *Source* : `EX-EXP-05`. *Vérification* : IHM.
- **SRS-IHM-EXP-03** [B] Les alarmes DOIVENT être affichées : état courant, sévérité, historique des
  levées et retombées.
  *Source* : `EX-EXP-06`. *Vérification* : celle d'`EX-EXP-06`, partie IHM.
- **SRS-IHM-ANA-01** [B] L'interface DOIT permettre de poser, parcourir et commenter des repères,
  d'extraire un intervalle avec ses options (bases, règles, repères dans une session jointe), de
  comparer deux traces et de générer un rapport.
  *Source* : `EX-ANA-01`, `EX-ANA-02`, `EX-ANA-03`, `EX-ANA-04`. *Vérification* : IHM.

### 4.7 Émission

- **SRS-IHM-SAF-01** [C] Dès qu'un canal est armé, un **indicateur d'émission** DOIT être affiché en
  permanence sur chaque fenêtre.
  *Source* : `EX-SAF-05`. *Vérification* : IHM.
- **SRS-IHM-SAF-02** [C] L'**arrêt d'urgence** DOIT être accessible en une action unique : par un
  bouton visible sur chaque fenêtre dès qu'un canal est armé, par quelque client que ce soit, et
  par un raccourci clavier actif dans chaque fenêtre de l'application, y compris quand un dialogue
  modal est ouvert. La touche par défaut et la portée hors de l'application relèvent du point
  ouvert IHM-3.
  *Source* : `EX-SAF-05` ; dialogue modal dérivé — un arrêt d'urgence masqué par un dialogue ne
  remplit pas son office. *Vérification* : IHM, canal armé par un second client, dialogue ouvert.
- **SRS-IHM-SAF-03** [C] L'armement DOIT passer par un dialogue nommant le canal, affichant la liste
  blanche (et signalant une entrée générique) et nommant les identifiants en risque de collision.
  *Source* : `EX-SAF-02`, `EX-SAF-03`, `EX-SAF-04`. *Vérification* : IHM.
- **SRS-IHM-REJ-01** [C] L'interface DOIT offrir l'éditeur tabulaire du jeu de test, la création
  depuis un intervalle de trace, l'état du galop d'essai et le rapport de fidélité.
  *Source* : `EX-REJ-02`, `EX-REJ-05`, `EX-SAF-06`. *Vérification* : IHM.

### 4.8 Session, profils, langues

- **SRS-IHM-SES-01** [A] L'interface DOIT ouvrir et enregistrer sessions et profils, lister les
  profils, et signaler précisément les fichiers manquants à l'ouverture.
  *Source* : `EX-SES-03`, `EX-SES-04`. *Vérification* : IHM.
- **SRS-IHM-I18N-01** [A] Toute chaîne visible DOIT passer par le mécanisme de traduction, en
  chaînes complètes paramétrées ; le français est livré complet ; repli sur l'anglais pour toute
  chaîne non traduite.
  *Source* : `EX-I18N-02`, `EX-I18N-04`. *Vérification* : celles d'`EX-I18N-02` et
  d'`EX-I18N-04`.
- **SRS-IHM-I18N-02** [A] La langue DOIT être sélectionnable dans l'interface et mémorisée dans les
  préférences de l'utilisateur, pas dans la session ; un fichier de traduction déposé dans le
  répertoire prévu est proposé et appliqué sans recompilation.
  *Source* : `EX-I18N-02`, `EX-I18N-03`. *Vérification* : celle d'`EX-I18N-03`.
- **SRS-IHM-I18N-03** [A] Nombres, dates et durées affichés DOIVENT suivre la locale.
  *Source* : `EX-I18N-05`. *Vérification* : IHM sous `fr_FR` et `en_US`.

### 4.9 Sécurité

- **SRS-IHM-SEC-01** [A] L'interface N'OUVRE AUCUN port réseau en écoute, en TCP comme en UDP.
  *Source* : `EX-SEC-01`, `D-35`. *Vérification* : celle d'`EX-SEC-01` pour l'interface en rendu
  hors écran.

---

## 5. Contraintes de conception

- Qt via PySide6 ; aucun constructeur de widget ne contient de logique.
- Tests : fumée seulement, en rendu hors écran, sans simulation de clics ; ils vérifient la
  présence et le câblage d'éléments dont la logique est testée dans `SRS-ANA` (`D-23`).

## 6. Points ouverts

| # | Point | Renvoi |
|---|---|---|
| IHM-1 | Bibliothèque de tracé | `O-03` |
| IHM-2 | Empaquetage par plateforme | `O-04` |
| IHM-3 | Raccourci clavier par défaut de l'arrêt d'urgence, et sa portée hors de la fenêtre active (raccourci global au système ou non) | à trancher au palier C |
