# SRS-PNT — Pont Tier 2 (python-can)

> Spécification des exigences logicielles du composant **pont**.
> Version 1.0 — 2026-10-07. Dérivée de `SPEC.md` v1.2, qui prévaut en cas d'écart.
> Conventions : `docs/srs/README.md`.

---

## 1. Objet et périmètre

Le pont est un processus Python optionnel qui ouvre un backend python-can (Vector, Kvaser,
ixxat, slcan, gs_usb, socketcand, pilote SocketCAN non listé en Tier 1…) et le raccorde au noyau
par l'interface d'injection (`D-17`). Le noyau reste seul maître du journal, de la liste blanche
et de l'émission : le pont n'est qu'un adaptateur de transport.

**Il ne fait pas** : aucun enregistrement, aucun décodage, aucune émission de sa propre
initiative, aucun accès aux clients.

## 2. Références

`SPEC.md` v1.2 (§5.1, `D-15`, `D-17`) ; `SRS-NOY.md` (I-INJ) ; documentation python-can.

## 3. Interfaces

| Id | Interface | Sens | Description |
|---|---|---|---|
| **I-INJ** | Injection | requise | Fournie par le noyau : remontée des trames et événements, descente des trames à émettre. |
| **I-PYCAN** | python-can | requise | Backend matériel ouvert par le pont. |

---

## 4. Exigences

- **SRS-PNT-01** [A] Le pont DOIT ouvrir un canal sur tout backend python-can et le présenter au
  noyau comme un canal **Tier 2**, avec son couple (interface python-can, matériel quand il est
  connu) et la liste des garanties non assurées (débit, horodatage, détection de perte).
  *Source* : §5.1, `EX-BUS-04`, `D-17`. *Vérification* : celle d'`EX-BUS-04` sur pont simulé
  (backend `virtual` de python-can).
- **SRS-PNT-02** [A] Chaque trame reçue DOIT être injectée dans le noyau avec tous les attributs
  fournis par le backend (identifiant, étendu, DLC, données, FD, BRS, ESI, requête distante,
  rapport d'erreur) et son horodatage, accompagné de sa source déclarée (matériel si le backend le
  fournit, sinon espace utilisateur).
  *Source* : `EX-BUS-03`, `EX-CAP-01`. *Vérification* : vecteurs injectés par le backend simulé,
  relus à l'identique ; source déclarée conforme.
- **SRS-PNT-03** [A] Toute perte ou erreur signalée par le backend DOIT être remontée au noyau comme
  événement ; à défaut d'une telle information, le canal est déclaré « détection de perte non
  assurée ».
  *Source* : `EX-CAP-04`, §5.1. *Vérification* : perte signalée par le backend simulé ;
  événement présent ; métadonnée de garantie conforme.
- **SRS-PNT-04** [A] Le pont DOIT ouvrir le backend en mode écoute seule quand le backend le
  permet ; sinon, il n'ouvre le canal qu'après la confirmation explicite transmise par le noyau, et
  le fait est inscrit.
  *Source* : `EX-SAF-01`. *Vérification* : backend sans mode écoute seule ; ouverture refusée sans
  confirmation.
- **SRS-PNT-05** [C] Le pont NE DOIT émettre que les trames reçues de l'ordonnanceur du noyau par
  I-INJ, et DOIT lui retourner pour chacune l'instant réel (écho quand le backend le fournit, sinon
  instant de soumission) ou l'échec.
  *Source* : `D-17`, `EX-SAF-03`, `EX-REJ-05`. *Vérification* : celle d'`EX-SAF-03` via le pont ;
  échos et échecs restitués.
- **SRS-PNT-06** [C] Sur arrêt d'urgence, le pont DOIT cesser toute émission et vider les files
  d'émission du backend quand celui-ci le permet ; le délai effectif est mesuré et déclaré comme
  garantie Tier 2, sans seuil contractuel.
  *Source* : `EX-SAF-05`, §5.1. *Vérification* : délai mesuré sur backend simulé et publié.
- **SRS-PNT-07** [A] La fin ou le plantage du pont DOIT être détecté par le noyau et inscrit comme
  événement de fin de canal ; les autres canaux et l'enregistrement n'en sont pas affectés ; un
  canal armé est désarmé.
  *Source* : dérivée — `EX-CAP-04` (aucune perte silencieuse), `EX-ARC-01`, `EX-SAF-02`.
  *Vérification* : `kill -9` du pont pendant un enregistrement multicanal ; événement présent,
  autres canaux complets.
- **SRS-PNT-08** [A] Le pont DOIT fonctionner sur les trois plateformes de `EX-POR-01`, dans la
  mesure où le backend python-can visé y existe.
  *Source* : `EX-POR-01`. *Vérification* : backend simulé sur les trois plateformes en
  intégration continue.

---

## 5. Points ouverts

| # | Point | Renvoi |
|---|---|---|
| PNT-1 | Qui lance le pont : le démon à l'ouverture d'un canal Tier 2 (recommandé, ouverture uniforme par le protocole), ou l'utilisateur à part | fiche à écrire avant le jalon Tier 2 |
| PNT-2 | Le délai de 50 ms d'`EX-SAF-05` ne peut pas être garanti en Tier 2 ; `SPEC.md` ne le dit pas explicitement | amendement de `SPEC.md` |
