# SRS-OUT — Outillage de vérification et livraison

> Spécification des exigences logicielles des outils qui ne sont pas livrés à l'utilisateur
> final comme fonction, mais dont dépend la vérification : banc (`bench/`), injecteur, matrice de
> traçabilité, intégration continue, empaquetage.
> Version 1.0 — 2026-10-07. Dérivée de `SPEC.md` v1.2, qui prévaut en cas d'écart.
> Conventions : `docs/srs/README.md`.

---

## 1. Objet et périmètre

Ces outils produisent les preuves : sans eux, les exigences de performance, de traçabilité et de
portabilité ne sont pas vérifiables. Ils publient des chiffres, pas des avis.

## 2. Exigences

- **SRS-OUT-01** [A] L'**injecteur** DOIT produire sur bus virtuel le profil de charge nominal de
  §6.1 de `SPEC.md` (et le demi-profil ARM64) : canaux, débits, identifiants (64 par canal, ordre
  fixe publié), DLC, données pseudo-aléatoires à graine publiée, au débit saturé calculé par la
  fonction de durée de trame de `libclemcan`. Chaque trame porte un numéro de séquence par canal
  permettant de compter exactement les trames manquantes.
  *Source* : `EX-PRF-01`, `EX-PRF-02`, §6.1. *Vérification* : débit produit comparé au débit
  théorique de la séquence générée à 1 % près ; reproductibilité à graine égale.
- **SRS-OUT-02** [A] Le **banc** DOIT s'exécuter chaque nuit sur les machines de référence x86-64 et
  ARM64 (machines dédiées), publier chaque métrique avec plateforme, modèle de machine, version et
  date, et signaler une régression quand la médiane de 5 exécutions se dégrade de plus de 10 % par
  rapport à la référence publiée.
  *Source* : `EX-PRF-05`. *Vérification* : celle d'`EX-PRF-05`.
- **SRS-OUT-03** [A] Le banc DOIT couvrir au moins : enregistrement au profil nominal 60 min
  (`EX-PRF-01`), demi-profil et recherche de débit maximal ARM64 (`EX-PRF-02`), décodage paresseux
  (`EX-DEC-11`), navigation dans 200 millions de trames (`EX-JNL-05`), latence d'affichage
  (`EX-VUE-06`) et, à leur palier, évaluation de référence (`EX-PRF-06`) et fidélité du rejeu
  (`EX-REJ-04`).
  *Source* : `EX-PRF-01`, `EX-PRF-02`, `EX-PRF-04`, `EX-PRF-05`, `EX-PRF-06`, `EX-DEC-11`,
  `EX-JNL-05`, `EX-VUE-06`. *Vérification* : chaque métrique présente dans la publication nocturne.
- **SRS-OUT-04** [A] La **matrice de traçabilité** exigence → test(s) → classe → jalon → dernier
  résultat DOIT être générée automatiquement depuis les noms des tests ; la construction échoue si
  une exigence d'un palier livré n'a aucun test auto.
  *Source* : §12, `D-23`. *Vérification* : exigence d'un palier livré privée de test → échec de
  construction.
- **SRS-OUT-05** [A] L'**intégration continue** DOIT construire et exécuter la suite auto sur Linux
  x86-64, Linux ARM64 et Windows x86-64, sans matériel, sans serveur graphique et sans accès
  réseau pendant les tests.
  *Source* : `EX-POR-01`, `EX-POR-04`, `EX-AUT-04`. *Vérification* : configuration d'intégration
  continue ; une exécution verte sur les trois plateformes.
- **SRS-OUT-06** [A] L'**empaquetage** DOIT produire par plateforme un exécutable autonome complet
  (ligne de commande, démon, interface) ne nécessitant aucun environnement d'exécution installé,
  un livrable du démon seul sans composant graphique, et une installation par gestionnaire de
  paquets pour le développement.
  *Source* : `EX-POR-02`, `EX-POR-03`, `O-04`. *Vérification* : celles d'`EX-POR-02` et
  d'`EX-POR-03`.
- **SRS-OUT-07** [A] Le livrable autonome complet DOIT démarrer à froid en moins de 3 s sur la
  machine de référence et peser moins de 300 Mo.
  *Source* : `O-04` (critère de décision). *Vérification* : mesure sur chaque livrable publié.

## 3. Points ouverts

| # | Point | Renvoi |
|---|---|---|
| OUT-1 | Mécanisme d'empaquetage | `O-04` |
| OUT-2 | Hébergement des machines de référence et du banc nocturne (machines dédiées auto-hébergées) | à organiser avant la fin du palier A |
