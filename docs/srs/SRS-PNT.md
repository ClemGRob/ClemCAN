# SRS-PNT — Pont Tier 2 (python-can)

> Spécification des exigences logicielles du composant **pont**.
> Version 1.1 — 2026-11-25. Dérivée de `SPEC.md` v1.3, qui prévaut en cas d'écart.
> Conventions : `docs/srs/README.md`.

---

## 1. Objet et périmètre

Le pont est un processus Python qui ouvre un backend python-can (Vector, Kvaser, ixxat, slcan,
gs_usb, socketcand, pilote SocketCAN non listé en Tier 1…) et le raccorde au noyau par
l'interface d'injection (`D-17`). Le démon le lance à l'ouverture d'un canal Tier 2 et le surveille
(`D-36`). En v1, le pont est en **réception seule** (`D-30`). Le noyau reste seul maître du
journal, de la liste blanche et de l'émission : le pont n'est qu'un adaptateur de transport.

**Il ne fait pas** : aucun enregistrement, aucun décodage, aucune émission, aucun accès aux
clients.

## 2. Références

`SPEC.md` v1.3 (§5.1, `D-15`, `D-17`, `D-30`, `D-35`, `D-36`) ; `SRS-NOY.md` (I-INJ) ;
documentation python-can.

## 3. Interfaces

| Id | Interface | Sens | Description |
|---|---|---|---|
| **I-INJ** | Injection | requise | Fournie par le noyau : configuration du canal et confirmation d'ouverture vers le pont, trames et événements vers le noyau. |
| **I-PYCAN** | python-can | requise | Backend matériel ouvert par le pont. |

---

## 4. Exigences

- **SRS-PNT-01** [A] Le pont DOIT ouvrir un canal sur tout backend python-can et le présenter au
  noyau comme un canal **Tier 2**, avec son couple (interface python-can, matériel quand il est
  connu) et la liste des garanties non assurées (débit, horodatage, détection de perte). Il
  n'émet aucune trame.
  *Source* : §5.1, `EX-BUS-04`, `D-17`, `D-30`. *Vérification* : celle d'`EX-BUS-04` sur pont
  simulé (backend `virtual` de python-can) ; aucune trame émise par le pont.
- **SRS-PNT-02** [A] Chaque trame reçue DOIT être injectée dans le noyau avec tous les attributs
  fournis par le backend (identifiant, étendu, DLC, données, FD, BRS, ESI, requête distante,
  rapport d'erreur) et son horodatage. Le pont convertit l'horodatage dans la base monotone de
  l'hôte et déclare sa source (matériel, noyau du système ou espace utilisateur) et la base
  d'origine fournie par le backend.
  *Source* : `EX-BUS-03`, `EX-CAP-01`, `EX-CAP-02`. *Vérification* : vecteurs injectés par le
  backend simulé, relus à l'identique ; horodatages croissants dans la base monotone ; source et
  base d'origine déclarées conformes.
- **SRS-PNT-03** [A] Toute perte ou erreur signalée par le backend DOIT être remontée au noyau comme
  événement ; à défaut d'une telle information, le canal est déclaré « détection de perte non
  assurée ».
  *Source* : `EX-CAP-04`, §5.1. *Vérification* : perte signalée par le backend simulé ;
  événement présent ; métadonnée de garantie conforme.
- **SRS-PNT-04** [A] Le pont DOIT ouvrir le backend en mode écoute seule quand le backend le
  permet ; sinon, il n'ouvre le canal qu'après la confirmation explicite transmise par le noyau, et
  le fait est signalé et inscrit. La capacité d'écoute seule de chaque backend est déclarée dans
  une table publiée ; un backend absent de la table est traité comme sans écoute seule.
  *Source* : `EX-SAF-01`. *Vérification* : backend sans mode écoute seule ; ouverture refusée sans
  confirmation ; backend absent de la table traité de même.
- **SRS-PNT-05** *Obsolète en version 1.1* : le pont n'émet pas en v1 (`D-30`). Identifiant
  réservé.
- **SRS-PNT-06** *Obsolète en version 1.1* : sans émission, pas d'arrêt d'urgence côté pont
  (`D-30`). Identifiant réservé.
- **SRS-PNT-07** [A] Le pont DOIT signaler au noyau par I-INJ sa fin normale avant de s'arrêter.
  La détection de sa fin anormale incombe au démon (`SRS-NOY-INJ-02`).
  *Source* : `EX-CAP-04`, `D-36`. *Vérification* : arrêt normal du pont ; fin signalée et
  inscrite comme événement de fin de canal.
- **SRS-PNT-08** [A] Le pont DOIT fonctionner sur les trois plateformes de `EX-POR-01`, dans la
  mesure où le backend python-can visé y existe.
  *Source* : `EX-POR-01`. *Vérification* : backend simulé sur les trois plateformes en
  intégration continue.
- **SRS-PNT-09** [A] Le pont N'OUVRE AUCUN port réseau en écoute, en TCP comme en UDP ; il ne
  communique avec le noyau que par I-INJ.
  *Source* : `EX-SEC-01`, `D-35`, `D-36`. *Vérification* : celle d'`EX-SEC-01` pour le pont sur
  backend simulé.

---

## 5. Points ouverts

| # | Point | Renvoi |
|---|---|---|
| PNT-1 | Qui lance le pont : le démon à l'ouverture d'un canal Tier 2, ou l'utilisateur à part | clos par `D-36` : le démon |
| PNT-2 | Le délai de 50 ms d'`EX-SAF-05` ne peut pas être garanti en Tier 2 ; `SPEC.md` ne le dit pas explicitement | clos par `D-30` : pas d'émission en Tier 2 en v1 |
