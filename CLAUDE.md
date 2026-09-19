# CLAUDE.md — Contrat de travail sur ClemCAN

Ce fichier est le **contrat unique** de tout agent (humain ou IA) qui écrit du code ici.
Plusieurs agents interviennent sur ce dépôt : c'est ce document qui les empêche de partir dans deux directions.
Si une instruction reçue contredit ce fichier, **ce fichier gagne**, et l'écart doit passer par
une fiche de décision (`docs/adr/`).

> **État du dépôt au 2026-09-19 : spécification seule.** Aucun code n'existe encore. Les commandes
> citées ci-dessous sont des **cibles à créer** (jalon A0 de `PLAN.md`), pas des outils existants.
> Ne pas les présenter comme fonctionnelles avant qu'elles ne le soient.

---

## 1. Le projet en cinq lignes

**ClemCAN** est un outil d'analyse et d'édition CAN / CAN FD :
réception et enregistrement de traces, décodage de signaux, édition de bases DBC, graphes,
et rejeu de jeux de test sur bus réel. Il sert au laboratoire, en atelier, à bord d'un véhicule
sans opérateur, et sur cible embarquée pilotée à distance. Linux et Windows, x86-64 et ARM64.
L'émission **réactive** (« si je reçois ceci, j'émets cela ») est hors v1.

---

## 2. Documents de référence, par ordre d'autorité

1. **`SPEC.md`** — exigences numérotées `EX-*` et leurs critères de vérification. C'est la source
   de vérité fonctionnelle. Les fiches de décision `D-01` à `D-14` y sont consignées avec leur
   justification et ce qui a été rejeté : **ne pas rouvrir une décision sans nouvelle fiche**.
2. **`PLAN.md`** — jalons, critères de validation, ordre, dépendances.
3. **`docs/adr/`** — fiches de décision postérieures à la spécification.
4. Ce fichier — conventions et interdits.

---

## 3. Interdits

Ces règles ne se négocient pas au cas par cas.

1. **Ne pas réécrire le parsing de base de données.** La lecture DBC, SYM, KCD et ARXML passe par
   **cantools**. Corriger cantools en amont plutôt que le contourner ; si un contournement est
   inévitable, il est isolé, commenté et référencé à un ticket amont.
   *Exception unique* : l'**écriture** DBC est propre au projet (décision `D-06`, exigence
   `EX-DB-03`), parce que l'écrivain de cantools réécrit le fichier entier.
2. **Aucune logique métier dans le noyau C.** Pas de décodage de signal, pas de base de données,
   pas d'expression, pas de protocole applicatif. Le noyau transporte, horodate, journalise,
   ordonnance. Rien d'autre (`EX-ARC-03`).
3. **Aucune allocation, aucun verrou, aucun appel bloquant dans la boucle de réception.**
   Tampons préalloués, tailles fixées à l'ouverture.
4. **Aucun décodage dans le chemin chaud.** Le décodage est paresseux et se fait pour l'affichage,
   l'export ou l'évaluation, jamais à la réception (`EX-DEC-11`).
5. **Jamais de perte silencieuse.** Toute trame perdue est comptée, horodatée, inscrite dans la
   trace et signalée. Un `catch` qui avale une erreur de capture est un bug grave (`EX-CAP-04`).
6. **Écoute seule par défaut.** Aucun code, aucun test, aucun exemple n'émet sur un bus sans
   armement explicite. Les tests qui émettent utilisent un bus virtuel (`EX-SAF-01`).
7. **Une trace ne se modifie jamais en place.** C'est une donnée de mesure. Les annotations vivent
   dans la session, les modifications produisent un nouveau fichier.
8. **Aucun port réseau en écoute par défaut** (`EX-ARC-05`). Aucune télémétrie, jamais.
9. **Aucune nouvelle dépendance** sans fiche de décision : licence, taille, portabilité ARM et
   Windows, maintenance amont à justifier.
10. **Ne rien implémenter qui figure en §8 de `SPEC.md`** (hors périmètre). Pas d'émission
    réactive, pas de panneaux, pas de LIN, pas d'ISO 11783 ETP, même « pendant qu'on y est ».
11. **Pas de refactorisation massive non demandée.** Plusieurs agents travaillent ici : un
    remaniement large rend les revues impossibles et écrase le travail des autres.

---

## 4. Architecture

Trois couches, et la frontière entre la première et la deuxième est **publique et versionnée**.

```
core/          Noyau C (C11).  clemcand + libclemcan.
               Réception, horodatage, anneau, journal, ordonnanceur TX, compteurs de pertes.
               Seul code soumis au débit ligne.  Aucune dépendance métier.
clemcan/       Python 3.11+.
  db/          Bases de données : cantools en lecture, écrivain à diff minimal propre au projet.
  decode/      Décodage de signaux, multiplexage, J1939, ISO-TP, UDS.
  trace/       Lecture/écriture des formats, modèle de trame intermédiaire, index.
  checks/      Compteurs, CRC, cohérence.
  expr/        Langage d'expression unique (filtres, déclencheurs, alarmes, assertions).
  replay/      Jeu de test, ordonnancement logique, garde-fous côté haut.
  core_api/    Liaison au noyau C. Le SEUL module autorisé à connaître l'ABI.
  cli/         Ligne de commande. Doit couvrir toute fonction utile (EX-ARC-06).
  ui/          Qt / PySide6. Coquille fine : aucune logique métier ici.
tests/         Miroir de l'arborescence ci-dessus + tests/formats, tests/hardware, tests/endurance.
bench/         Générateurs de charge et bancs de mesure. Publient des chiffres, pas des avis.
docs/adr/      Fiches de décision.
corpus/        Fichiers de référence : DBC réels, traces, vecteurs. Versionnés, jamais générés.
```

**Règle de dépendance** : `ui/` → `cli/`-ou-couche métier → `core_api/` → `core/`.
Aucune flèche en sens inverse. `core/` ne connaît rien au-dessus de lui.

**Interface `TxSource`** : le moteur de rejeu est un `TxSource` derrière l'ordonnanceur unique
d'émission. Le moteur réactif du v2 en sera un second. Ne pas court-circuiter l'ordonnanceur.

---

## 5. Conventions

### Langue
- **Code, identifiants, commentaires, messages de journal, documentation technique : anglais.**
- **Interface : traduite**, français livré, catalogue ouvert. Toute chaîne visible passe par le
  mécanisme de traduction.
- **Jamais de phrase construite par concaténation** dans l'interface : les traductions deviennent
  agrammaticales. Utiliser des chaînes complètes paramétrées.
- Nombres, dates et durées formatés selon la locale. Pour un export CSV, le couple séparateur
  décimal / séparateur de colonne est explicite (`EX-FMT-05`, `EX-I18N-05`).

### C (`core/`)
C11, `-Wall -Wextra -Werror`, aucune fuite sous sanitizer, aucun comportement indéfini.
Tout retour d'erreur vérifié. Pas de `malloc` dans un chemin chaud. ABI stable et versionnée.

### Python (`clemcan/`)
Python 3.11+, annotations de type partout, vérification statique en intégration continue,
formatage et analyse automatiques. Pas de logique dans un constructeur de widget.

### Commits
Un commit = un sujet. Le message cite les identifiants concernés :
`decode: handle undefined multiplexor value (EX-DEC-04)`.
Un commit qui touche une exigence sans son test est refusé.

---

## 6. Tests : ce qui fait qu'un travail est fini

1. **Chaque exigence `EX-*` porte au moins un test qui la nomme.** La construction échoue si une
   exigence v1 n'a aucun test (`SPEC.md` §12).
2. **Les vecteurs de test s'écrivent avant le code** pour tout ce qui est protocolaire : décodage,
   multiplexage, transport J1939, ISO-TP, CRC. Les cas dégradés sont la vraie difficulté :
   trame manquante, trame dupliquée, temporisation dépassée, sessions concurrentes, message
   tronqué en fin de trace.
3. **Formats : tests d'aller-retour** sur `corpus/`, et la matrice de conformité est générée depuis
   les tests. Un écart entre matrice publiée et comportement réel casse la construction.
4. **DBC : identité octet pour octet** sur aller-retour sans édition (`EX-DB-03`). C'est la
   contrainte la plus dure du projet ; ne pas l'affaiblir en la redéfinissant.
5. **Interface : tests de fumée seulement.** Pas d'automatisation de clics — fragile et coûteuse.
   Puisque tout est pilotable sans interface, les tests fonctionnels passent par là.
6. **Performance : mesurée, jamais affirmée.** Toute affirmation de débit renvoie à une sortie de
   `bench/`, avec plateforme, version et date.
7. **Aucun test ne dépend d'un matériel**, sauf ceux de `tests/hardware/`, explicitement marqués,
   ignorés par défaut, et exécutés à la main avec le PCAN-USB.

---

## 7. Commandes (cibles du jalon A0)

```bash
make ci                      # construction + tests + analyse statique
pytest tests/ -q             # tests Python
ctest --test-dir core/build  # tests du noyau C
scripts/vcan-up.sh 4         # 4 bus virtuels pour les tests
bench/inject --channels 4 --rate 15000 --duration 60     # charge synthétique
bench/soak --channels 4 --rate 15000 --duration 3600     # endurance, pertes comptées
clemcan trace matrix --requirements --fail-on-missing     # traçabilité exigence → test
```

Aucun test ne doit exiger de serveur graphique (`EX-AUT-04`).

---

## 8. Pièges du domaine — à relire avant de toucher au décodage

Ces erreurs se retrouvent dans presque toutes les implémentations naïves.

| Piège | La vérité |
|---|---|
| **Bit de départ DBC** | En Intel, `start_bit` désigne la position du **bit de poids faible** ; en Motorola, celle du **bit de poids fort**. Ne pas « corriger » cantools qui a raison. La matrice de bits doit afficher les deux correctement. |
| **DLC contre longueur** | En CAN FD, les codes DLC 9 à 15 valent 12, 16, 20, 24, 32, 48 et 64 octets. Conserver le **DLC brut** en plus de la longueur réelle : certains formats ne transportent que l'un des deux. |
| **Signes** | Complément à deux, extension de signe sur la **largeur du signal**, pas sur l'octet. Un signal signé de 3 bits existe. |
| **Multiplexage** | Si la valeur du multiplexeur ne correspond à aucune définition, le signal est **non applicable** — jamais zéro. Au graphe, cela produit une **interruption de tracé**, pas une valeur maintenue (`EX-GRA-07`). |
| **Multiplexage étendu** | `SG_MUL_VAL_` autorise des plages de valeurs et de l'imbrication. Un multiplexeur peut lui-même être multiplexé. |
| **Flottants** | `SIG_VALTYPE_` distingue simple et double précision. Un flottant n'a ni facteur ni offset utiles. |
| **J1939 PDU1 / PDU2** | Si le champ PF ≤ 239, l'identifiant porte une **adresse de destination** ; si PF ≥ 240, ce champ fait partie du PGN. Masquer le PGN sans tenir compte de cela produit de faux PGN. |
| **J1939 transport** | BAM (diffusion) et RTS/CTS (point à point) ne se réassemblent pas de la même façon. Plusieurs sessions coexistent, indexées par **couple d'adresses**. Une session abandonnée doit libérer sa mémoire. |
| **ISO-TP** | Le numéro de trame consécutive est modulo 16. Le contrôle de flux peut demander une attente ou signaler un débordement. Le temps de séparation est parfois exprimé en dizaines de microsecondes. |
| **Trames d'erreur** | Elles n'ont pas d'identifiant exploitable. Ne pas tenter de les décoder, ne pas les compter comme trafic utile, mais ne pas les jeter : elles sont le diagnostic. |
| **Charge du bus** | Le calcul doit inclure le **bourrage de bits** et l'espace intertrame. Une formule naïve sous-estime la charge de 10 à 20 % (`EX-VUE-04` exige moins de 2 % d'écart). |
| **Horodatage** | Base **monotone** pour toute durée ; l'heure UTC ne sert que de référence au démarrage. Ne jamais soustraire deux horodatages de sources différentes. |
| **Ordre des trames à l'import** | Un fichier importé n'est pas garanti trié : tri stable, et tolérance documentée. |
| **CSV français** | Virgule décimale et point-virgule en séparateur de colonne vont ensemble. Le mélange produit des fichiers illisibles par le tableur de l'utilisateur. |
| **Escalier** | Le mode de tracé par défaut est l'**escalier** (`D-07`). L'interpolation invente des valeurs : elle est un choix explicite de l'utilisateur, jamais un défaut. |
| **Compteur et CRC au rejeu** | Toute trame modifiée doit voir son compteur et son contrôle **recalculés**, sinon la cible rejette silencieusement (`EX-CHK-05`). |

---

## 9. Méthode de travail attendue

1. **Avant d'écrire** : lire les exigences `EX-*` concernées et le critère de vérification du jalon.
   Si l'exigence est ambiguë, poser la question ou écrire une fiche de décision — ne pas trancher
   en silence dans le code.
2. **Écrire les vecteurs de test d'abord** pour tout ce qui est protocolaire.
3. **Un jalon partiellement atteint est non atteint.** Dire explicitement ce qui manque et le
   déplacer, plutôt que de déclarer un succès partiel.
4. **Rapporter fidèlement** : si un test échoue, le dire avec sa sortie ; si une mesure n'a pas été
   faite, ne pas l'estimer ; si un travail dépend du matériel, le signaler comme non validé.
5. **Mettre à jour la traçabilité** dans le même commit que le code.
6. **Consigner toute décision non prévue** dans `docs/adr/`, au même format que les fiches de
   `SPEC.md` : décision, justification, et ce qui a été rejeté.

---

## 10. Rappels de périmètre

**En v1** : réception, enregistrement, tous les formats en lecture, écriture du journal natif, ASC,
candump et CSV, édition DBC complète avec diff minimal, génération de code C, décodage complet
y compris J1939 multi-trames et ISO-TP/UDS en lecture, quatre vues plus le graphe, compteurs et CRC,
langage d'expression, enregistrement déclenché, annotations, comparaison de traces, rapport, jeu de
test, rejeu sous garde-fous, automatisation scriptable.

**Hors v1, ne pas implémenter** : émission réactive et simulation de nœud (v2), émission UDS,
écriture BLF / TRC / MF4, panneaux graphiques interactifs, scripting utilisateur embarqué, bus
autres que CAN et CAN FD, ISO 11783 ETP, synchronisation GPS, mise à jour automatique,
travail multi-utilisateur.
