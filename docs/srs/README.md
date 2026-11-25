# Spécifications des exigences logicielles (SRS)

> Version 1.1 — 2026-11-25. Dérivées de `SPEC.md` v1.3. Suite donnée à la revue :
> [`REVUE-2026-10-07.md`](REVUE-2026-10-07.md).

`SPEC.md` spécifie le **système** ClemCAN. Les SRS de ce répertoire en déclinent les exigences
par **composant logiciel** : chacune dit ce que son composant doit faire pour que les exigences
`EX-*` qui lui sont allouées soient satisfaites, à travers quelles interfaces, et comment le
vérifier à son niveau.

| Document | Composant | Langage | Exigences |
|---|---|---|---|
| [`SRS-NOY.md`](SRS-NOY.md) | Noyau : démon `clemcand`, bibliothèque `libclemcan` | C11 | 64 |
| [`SRS-ANA.md`](SRS-ANA.md) | Couche d'analyse et ligne de commande `clemcan` | Python | 95 |
| [`SRS-IHM.md`](SRS-IHM.md) | Interface graphique | Python, Qt | 36 |
| [`SRS-PNT.md`](SRS-PNT.md) | Pont Tier 2 | Python, python-can | 7 (+ 2 obsolètes) |
| [`SRS-OUT.md`](SRS-OUT.md) | Outillage de vérification et livraison | — | 6 (+ 1 obsolète) |

## Règles

1. **`SPEC.md` prévaut.** Une SRS n'assouplit jamais une exigence `EX-*`. En cas d'écart, la SRS
   est fausse et se corrige ; si c'est `SPEC.md` qui doit changer, on l'amende par une fiche de
   décision.
2. **Toute exigence SRS a une source** : une ou plusieurs exigences `EX-*`, une décision `D-*`,
   une section de `SPEC.md`, ou la mention **dérivée**. Une exigence dérivée n'existe pas dans
   `SPEC.md` mais découle de l'allocation entre composants ; sa justification est écrite à côté.
   Les exigences dérivées sont les premières à relire.
3. **Identifiants** : `SRS-<COMPOSANT>-<DOMAINE>-<nn>` (ou `SRS-<COMPOSANT>-<nn>` pour les petits
   documents). Jamais réutilisés ni renumérotés. Une exigence devenue sans objet est déclarée
   **obsolète** : son identifiant reste réservé, son texte est remplacé par la version et la
   cause ; elle n'a ni palier ni vérification et sort de la matrice.
4. **Palier** entre crochets (A, B, C) : celui auquel le composant doit fournir la capacité,
   c'est-à-dire celui de la source qui la fait naître. Il peut donc différer du palier d'une
   source transverse (`EX-ARC-06`, `EX-CAP-05`) ou d'une source à plusieurs paliers (`EX-SEC-02`) :
   les types d'événements du journal sont dus en A, l'injection d'événements par les clients ne
   sert qu'à partir de B, l'export d'image du graphe arrive en B même si `EX-ARC-06` est en A.
   Une source sans palier propre (section ou glossaire de `SPEC.md`, décision, justification
   dérivée) prend le palier de la capacité qu'elle sert : `SRS-ANA-TRC-07` et `SRS-IHM-ARC-05`
   sont dus en A, `SRS-NOY-TRG-03` en B. Une exigence qui naît à deux paliers porte les deux
   marques dans son texte ([A] … [B] …).
5. **Vérification** : même classes que `SPEC.md` (`D-23`) — auto par défaut, ⚙ matériel, IHM. « Celle
   d'`EX-…` » signifie que le critère système s'applique tel quel au composant.
6. Les tests nomment l'identifiant SRS **et** l'identifiant `EX-*` qu'ils couvrent ; la matrice de
   traçabilité (`SRS-OUT-04`) se génère depuis ces noms.

## Interfaces entre composants

```
 SRS-IHM ──I-API──▶ SRS-ANA ──I-PROT / I-LIB──▶ SRS-NOY ──I-BACK──▶ bus Tier 1
                                                   ▲
                                 SRS-PNT ──I-INJ───┘──I-PYCAN──▶ bus Tier 2
```

Aucune flèche en sens inverse : le noyau ne connaît aucun client, l'analyse ne connaît pas
l'interface. Le démon lance le pont (`D-36`) sans en dépendre : I-INJ est fournie par le noyau.
L'IHM ajoute l'export d'image au registre de l'analyse par un point d'extension
(`SRS-ANA-ARC-06`), sans que l'analyse l'importe.

## Points ouverts remontés vers `SPEC.md`

L'allocation a fait apparaître des points que `SPEC.md` ne tranchait pas. Ceux qui demandaient
un amendement ont été tranchés en version 1.3 de `SPEC.md` : `NOY-5` par `D-37`, `PNT-2` par
`D-30`, `PNT-1` par `D-36` ; `NOY-7` est clos sans amendement, le mot « événements »
d'`EX-ARC-04` couvrant le flux sortant et l'injection. `ANA-5` est repris par la décision ouverte
`O-08`. Aucun point ouvert ne demande plus d'amendement de `SPEC.md`.

## Matrice d'allocation `EX-*` → SRS

Générée par script depuis les champs *Source* des SRS, exigences obsolètes exclues ; paliers lus
dans `SPEC.md`. Toute exigence de `SPEC.md` est allouée.

| Exigence | Palier | Composants | Exigences SRS |
|---|---|---|---|
| `EX-ARC-01` | A | NOY, IHM, OUT | `SRS-IHM-ARC-04`, `SRS-NOY-ARC-01`, `SRS-NOY-INJ-02`, `SRS-OUT-01` |
| `EX-ARC-02` | A | NOY, ANA | `SRS-ANA-ARC-04`, `SRS-NOY-ARC-02` |
| `EX-ARC-03` | A | NOY | `SRS-NOY-ARC-03` |
| `EX-ARC-04` | A | NOY, ANA | `SRS-ANA-ARC-01`, `SRS-NOY-PRO-01` |
| `EX-ARC-05` | A | NOY | `SRS-NOY-SEC-01` |
| `EX-ARC-06` | A | ANA, IHM | `SRS-ANA-ARC-02`, `SRS-ANA-ARC-03`, `SRS-ANA-ARC-06`, `SRS-ANA-AUT-01`, `SRS-IHM-ARC-01`, `SRS-IHM-GRA-07` |
| `EX-BUS-01` | A | NOY, OUT | `SRS-NOY-ARC-05`, `SRS-OUT-01` |
| `EX-BUS-02` | A | NOY | `SRS-NOY-BUS-02` |
| `EX-BUS-03` | A | NOY, PNT | `SRS-NOY-BUS-03`, `SRS-NOY-BUS-08`, `SRS-PNT-02` |
| `EX-BUS-04` | A | NOY, IHM, PNT | `SRS-IHM-BUS-01`, `SRS-NOY-BUS-04`, `SRS-PNT-01` |
| `EX-BUS-05` | A | NOY, IHM | `SRS-IHM-BUS-02`, `SRS-NOY-BUS-05`, `SRS-NOY-BUS-08` |
| `EX-BUS-06` | A | NOY | `SRS-NOY-BUS-06`, `SRS-NOY-BUS-08` |
| `EX-BUS-07` | C | NOY | `SRS-NOY-BUS-07`, `SRS-NOY-BUS-08` |
| `EX-CAP-01` | A | NOY, PNT | `SRS-NOY-CAP-01`, `SRS-PNT-02` |
| `EX-CAP-02` | A | NOY, PNT | `SRS-NOY-CAP-02`, `SRS-PNT-02` |
| `EX-CAP-03` | A | NOY, OUT | `SRS-NOY-CAP-02`, `SRS-NOY-CAP-03`, `SRS-OUT-03` |
| `EX-CAP-04` | A | NOY, IHM, PNT, OUT | `SRS-IHM-BUS-02`, `SRS-NOY-BUS-08`, `SRS-NOY-CAP-04`, `SRS-NOY-INJ-02`, `SRS-OUT-01`, `SRS-PNT-03`, `SRS-PNT-07` |
| `EX-CAP-05` | A | NOY, IHM | `SRS-IHM-ARC-03`, `SRS-NOY-ABO-01`, `SRS-NOY-ABO-02`, `SRS-NOY-ABO-03`, `SRS-NOY-TRG-03` |
| `EX-JNL-01` | A | NOY | `SRS-NOY-JNL-01` |
| `EX-JNL-02` | A | NOY | `SRS-NOY-JNL-02` |
| `EX-JNL-03` | A | NOY | `SRS-NOY-BUS-02`, `SRS-NOY-JNL-03` |
| `EX-JNL-04` | B | NOY | `SRS-NOY-JNL-05` |
| `EX-JNL-05` | A | NOY, OUT | `SRS-NOY-JNL-06`, `SRS-OUT-03` |
| `EX-FMT-01` | A | ANA | `SRS-ANA-TRC-02` |
| `EX-FMT-02` | A | NOY, ANA | `SRS-ANA-TRC-01`, `SRS-NOY-JNL-07` |
| `EX-FMT-03` | A | ANA | `SRS-ANA-TRC-04` |
| `EX-FMT-04` | A | ANA | `SRS-ANA-TRC-04` |
| `EX-FMT-05` | A | ANA | `SRS-ANA-TRC-06` |
| `EX-FMT-06` | A | NOY, ANA | `SRS-ANA-TRC-05`, `SRS-NOY-JNL-03` |
| `EX-DB-01` | A | ANA | `SRS-ANA-DB-01` |
| `EX-DB-02` | A | ANA | `SRS-ANA-DB-02` |
| `EX-DB-03` | B | ANA | `SRS-ANA-DB-04` |
| `EX-DB-04` | B | ANA, IHM | `SRS-ANA-DB-05`, `SRS-IHM-DB-02` |
| `EX-DB-05` | B | ANA, IHM | `SRS-ANA-DB-06`, `SRS-IHM-DB-01` |
| `EX-DB-06` | B | ANA, IHM | `SRS-ANA-DB-07`, `SRS-IHM-DB-02` |
| `EX-DB-07` | B | ANA, IHM | `SRS-ANA-DB-08`, `SRS-IHM-DB-02` |
| `EX-DB-08` | B | ANA, IHM | `SRS-ANA-DB-09`, `SRS-IHM-DB-02` |
| `EX-DB-09` | A | ANA | `SRS-ANA-DB-03` |
| `EX-DB-10` | B | ANA | `SRS-ANA-DB-10` |
| `EX-DB-11` | B | ANA, IHM | `SRS-ANA-DB-11`, `SRS-IHM-DB-02` |
| `EX-DEC-01` | A | ANA | `SRS-ANA-DEC-01` |
| `EX-DEC-02` | A | ANA | `SRS-ANA-DEC-02` |
| `EX-DEC-03` | A | ANA | `SRS-ANA-DEC-03` |
| `EX-DEC-04` | A | ANA | `SRS-ANA-DEC-04` |
| `EX-DEC-05` | A | ANA | `SRS-ANA-DEC-05` |
| `EX-DEC-06` | A | ANA | `SRS-ANA-DEC-06` |
| `EX-DEC-07` | B | ANA | `SRS-ANA-DEC-10` |
| `EX-DEC-08` | B | ANA | `SRS-ANA-DEC-11` |
| `EX-DEC-09` | B | ANA | `SRS-ANA-DEC-12` |
| `EX-DEC-10` | B | ANA | `SRS-ANA-DEC-13` |
| `EX-DEC-11` | A | ANA, OUT | `SRS-ANA-DEC-08`, `SRS-OUT-03` |
| `EX-DEC-12` | A | ANA | `SRS-ANA-DEC-09` |
| `EX-CHK-01` | B | ANA | `SRS-ANA-CHK-01` |
| `EX-CHK-02` | B | ANA | `SRS-ANA-CHK-02` |
| `EX-CHK-03` | B | NOY, ANA | `SRS-ANA-CHK-03`, `SRS-ANA-CHK-05`, `SRS-NOY-JNL-12` |
| `EX-CHK-04` | B | ANA | `SRS-ANA-CHK-04` |
| `EX-CHK-05` | C | ANA | `SRS-ANA-CHK-06` |
| `EX-VUE-01` | A | ANA, IHM | `SRS-ANA-VUE-01`, `SRS-IHM-VUE-01` |
| `EX-VUE-02` | A | NOY, ANA, IHM | `SRS-ANA-VUE-02`, `SRS-IHM-VUE-02`, `SRS-NOY-STA-01` |
| `EX-VUE-03` | A | ANA, IHM | `SRS-ANA-VUE-03`, `SRS-IHM-VUE-03` |
| `EX-VUE-04` | A | NOY, ANA, IHM | `SRS-ANA-VUE-04`, `SRS-IHM-VUE-04`, `SRS-NOY-STA-01`, `SRS-NOY-STA-02`, `SRS-NOY-STA-03` |
| `EX-VUE-05` | B | ANA, IHM | `SRS-ANA-VUE-06`, `SRS-IHM-VUE-05` |
| `EX-VUE-06` | A | NOY, ANA, IHM, OUT | `SRS-ANA-VUE-05`, `SRS-IHM-ARC-02`, `SRS-IHM-GRA-05`, `SRS-NOY-ABO-04`, `SRS-OUT-03` |
| `EX-VUE-07` | A | IHM | `SRS-IHM-VUE-06` |
| `EX-GRA-01` | A | ANA, IHM | `SRS-ANA-GRA-01`, `SRS-IHM-GRA-01` |
| `EX-GRA-02` | A | ANA | `SRS-ANA-DEC-07`, `SRS-ANA-GRA-06` |
| `EX-GRA-03` | A | ANA, IHM | `SRS-ANA-GRA-06`, `SRS-IHM-GRA-02` |
| `EX-GRA-04` | A | ANA, IHM | `SRS-ANA-GRA-04`, `SRS-IHM-GRA-03` |
| `EX-GRA-05` | A | ANA, IHM | `SRS-ANA-GRA-05`, `SRS-IHM-GRA-04` |
| `EX-GRA-06` | A | ANA | `SRS-ANA-GRA-03` |
| `EX-GRA-07` | A | ANA, IHM | `SRS-ANA-GRA-02`, `SRS-IHM-GRA-01` |
| `EX-GRA-08` | B | ANA, IHM | `SRS-ANA-GRA-07`, `SRS-IHM-GRA-06` |
| `EX-GRA-09` | B | ANA, IHM | `SRS-ANA-ARC-06`, `SRS-ANA-GRA-08`, `SRS-IHM-GRA-07` |
| `EX-EXP-01` | B | NOY, ANA | `SRS-ANA-EXP-01`, `SRS-ANA-EXP-05`, `SRS-ANA-EXP-06`, `SRS-NOY-ABO-05` |
| `EX-EXP-02` | B | ANA | `SRS-ANA-EXP-02` |
| `EX-EXP-03` | B | ANA | `SRS-ANA-ARC-05`, `SRS-ANA-EXP-01`, `SRS-ANA-EXP-03`, `SRS-ANA-EXP-06` |
| `EX-EXP-04` | B | ANA, IHM | `SRS-ANA-EXP-04`, `SRS-IHM-EXP-01` |
| `EX-EXP-05` | B | NOY, ANA, IHM | `SRS-ANA-EXP-05`, `SRS-IHM-EXP-02`, `SRS-NOY-ABO-05` |
| `EX-EXP-06` | B | NOY, ANA, IHM | `SRS-ANA-EXP-08`, `SRS-IHM-EXP-03`, `SRS-NOY-JNL-12` |
| `EX-TRG-01` | B | NOY, ANA | `SRS-ANA-TRG-02`, `SRS-NOY-TRG-01` |
| `EX-TRG-02` | B | ANA | `SRS-ANA-TRG-01` |
| `EX-TRG-03` | B | NOY, ANA | `SRS-ANA-TRG-02`, `SRS-NOY-TRG-02` |
| `EX-TRG-04` | B | NOY, ANA | `SRS-ANA-TRG-01`, `SRS-NOY-JNL-12`, `SRS-NOY-TRG-02`, `SRS-NOY-TRG-03` |
| `EX-ANA-01` | B | ANA, IHM | `SRS-ANA-AN-01`, `SRS-ANA-TRC-07`, `SRS-IHM-ANA-01` |
| `EX-ANA-02` | B | NOY, ANA, IHM | `SRS-ANA-AN-02`, `SRS-IHM-ANA-01`, `SRS-NOY-JNL-07` |
| `EX-ANA-03` | B | ANA, IHM | `SRS-ANA-AN-03`, `SRS-IHM-ANA-01` |
| `EX-ANA-04` | B | ANA, IHM | `SRS-ANA-AN-04`, `SRS-IHM-ANA-01` |
| `EX-REJ-01` | C | ANA | `SRS-ANA-REJ-01` |
| `EX-REJ-02` | C | ANA, IHM | `SRS-ANA-REJ-02`, `SRS-IHM-REJ-01` |
| `EX-REJ-03` | C | NOY, ANA | `SRS-ANA-REJ-03`, `SRS-NOY-STA-03` |
| `EX-REJ-04` | C | NOY, ANA, OUT | `SRS-ANA-REJ-04`, `SRS-NOY-TX-03`, `SRS-OUT-03` |
| `EX-REJ-05` | C | NOY, ANA, IHM | `SRS-ANA-REJ-05`, `SRS-IHM-REJ-01`, `SRS-NOY-TX-10` |
| `EX-REJ-06` | C | NOY, ANA | `SRS-ANA-REJ-06`, `SRS-NOY-JNL-03`, `SRS-NOY-TX-10` |
| `EX-REJ-07` | C | ANA | `SRS-ANA-CHK-06` |
| `EX-REJ-08` | C | ANA | `SRS-ANA-REJ-07` |
| `EX-SAF-01` | A | NOY, IHM, PNT | `SRS-IHM-BUS-03`, `SRS-NOY-INJ-01`, `SRS-NOY-JNL-03`, `SRS-NOY-TX-01`, `SRS-PNT-04` |
| `EX-SAF-02` | C | NOY, ANA, IHM | `SRS-ANA-REJ-10`, `SRS-IHM-SAF-03`, `SRS-NOY-TX-02`, `SRS-NOY-TX-04`, `SRS-NOY-TX-07` |
| `EX-SAF-03` | C | NOY, ANA, IHM | `SRS-ANA-REJ-12`, `SRS-IHM-SAF-03`, `SRS-NOY-TX-05` |
| `EX-SAF-04` | C | NOY, ANA, IHM | `SRS-ANA-REJ-09`, `SRS-IHM-SAF-03`, `SRS-NOY-STA-04`, `SRS-NOY-TX-06` |
| `EX-SAF-05` | C | NOY, ANA, IHM | `SRS-ANA-REJ-11`, `SRS-IHM-SAF-01`, `SRS-IHM-SAF-02`, `SRS-NOY-TX-07`, `SRS-NOY-TX-08` |
| `EX-SAF-06` | C | NOY, ANA, IHM | `SRS-ANA-REJ-08`, `SRS-IHM-REJ-01`, `SRS-NOY-TX-11` |
| `EX-SAF-07` | C | NOY | `SRS-NOY-TX-09` |
| `EX-AUT-01` | A, étendue à chaque palier | ANA | `SRS-ANA-AUT-01` |
| `EX-AUT-02` | C | ANA | `SRS-ANA-AUT-02` |
| `EX-AUT-03` | C | ANA | `SRS-ANA-AUT-03` |
| `EX-AUT-04` | A, étendue à chaque palier | ANA, IHM, OUT | `SRS-ANA-ARC-03`, `SRS-ANA-AUT-04`, `SRS-ANA-REJ-09`, `SRS-IHM-ARC-05`, `SRS-OUT-05` |
| `EX-SES-01` | A, étendue à chaque palier | ANA | `SRS-ANA-SES-01` |
| `EX-SES-02` | A | ANA | `SRS-ANA-SES-02` |
| `EX-SES-03` | A | ANA, IHM | `SRS-ANA-SES-03`, `SRS-IHM-SES-01` |
| `EX-SES-04` | A | ANA, IHM | `SRS-ANA-SES-04`, `SRS-IHM-BUS-03`, `SRS-IHM-SES-01` |
| `EX-I18N-01` | A | ANA | `SRS-ANA-I18N-01` |
| `EX-I18N-02` | A | IHM | `SRS-IHM-I18N-01`, `SRS-IHM-I18N-02` |
| `EX-I18N-03` | A | IHM | `SRS-IHM-I18N-02` |
| `EX-I18N-04` | A | ANA, IHM | `SRS-ANA-I18N-01`, `SRS-IHM-I18N-01` |
| `EX-I18N-05` | A | ANA, IHM | `SRS-ANA-I18N-02`, `SRS-ANA-TRC-06`, `SRS-IHM-I18N-03` |
| `EX-PRF-01` | A | NOY, OUT | `SRS-NOY-PRF-01`, `SRS-OUT-01`, `SRS-OUT-03` |
| `EX-PRF-02` | A | NOY, OUT | `SRS-NOY-PRF-02`, `SRS-OUT-01`, `SRS-OUT-03` |
| `EX-PRF-03` | A | NOY, IHM | `SRS-IHM-BUS-02`, `SRS-NOY-CAP-04` |
| `EX-PRF-04` | C | NOY, ANA, OUT | `SRS-ANA-REJ-04`, `SRS-NOY-TX-03`, `SRS-OUT-03` |
| `EX-PRF-05` | A | OUT | `SRS-OUT-02`, `SRS-OUT-03` |
| `EX-PRF-06` | B | ANA, OUT | `SRS-ANA-PRF-01`, `SRS-OUT-03` |
| `EX-ROB-01` | A | NOY | `SRS-NOY-JNL-01` |
| `EX-ROB-02` | B | NOY | `SRS-NOY-ROB-01` |
| `EX-ROB-03` | A | NOY | `SRS-NOY-ARC-01` |
| `EX-ROB-04` | A | NOY | `SRS-NOY-JNL-10` |
| `EX-ROB-05` | B | NOY | `SRS-NOY-JNL-11` |
| `EX-POR-01` | A | NOY, PNT, OUT | `SRS-NOY-POR-01`, `SRS-OUT-05`, `SRS-PNT-08` |
| `EX-POR-02` | A | OUT | `SRS-OUT-06` |
| `EX-POR-03` | A | NOY, ANA, OUT | `SRS-ANA-ARC-03`, `SRS-NOY-INJ-01`, `SRS-NOY-POR-01`, `SRS-OUT-06` |
| `EX-POR-04` | A | NOY, OUT | `SRS-NOY-SEC-04`, `SRS-OUT-05` |
| `EX-SEC-01` | A | NOY, ANA, IHM, PNT | `SRS-ANA-SEC-01`, `SRS-IHM-SEC-01`, `SRS-NOY-SEC-01`, `SRS-PNT-09` |
| `EX-SEC-02` | B | NOY, ANA | `SRS-ANA-ARC-04`, `SRS-NOY-SEC-02`, `SRS-NOY-TX-12` |
| `EX-SEC-03` | A | NOY, ANA | `SRS-ANA-SEC-01`, `SRS-NOY-SEC-04` |
| `EX-SEC-04` | A | NOY | `SRS-NOY-SEC-03` |
