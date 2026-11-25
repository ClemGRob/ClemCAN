# SRS-OUT — Outillage de vérification et livraison

> Spécification des exigences logicielles des outils qui ne sont pas livrés à l'utilisateur
> final comme fonction, mais dont dépend la vérification : banc (`bench/`), injecteur, matrice de
> traçabilité, intégration continue, empaquetage.
> Version 1.1 — 2026-11-25. Dérivée de `SPEC.md` v1.3, qui prévaut en cas d'écart.
> Conventions : `docs/srs/README.md`.

---

## 1. Objet et périmètre

Ces outils produisent les preuves : sans eux, les exigences de performance, de traçabilité et de
portabilité ne sont pas vérifiables. Ils publient des chiffres, pas des avis.

## 2. Exigences

- **SRS-OUT-01** [A] L'**injecteur** DOIT produire sur bus virtuel des charges paramétrées : nombre
  de canaux (au moins 1, 2, 4, 8 et 32), type et débits de chaque canal, identifiants (64 par
  canal, ordre fixe publié), DLC, données pseudo-aléatoires à graine publiée, et débit par canal
  réglable de zéro jusqu'au-delà du débit saturé calculé par la fonction de durée de trame de
  `libclemcan`. Le profil nominal de §6.1 de `SPEC.md` et le demi-profil ARM64 sont des
  préréglages. Chaque trame de DLC 4 ou plus porte dans ses 4 premiers octets de données un numéro
  de séquence par canal, sur 32 bits, petit-boutien, qui permet de compter exactement les trames
  manquantes.
  *Source* : `EX-PRF-01`, `EX-PRF-02`, `EX-BUS-01`, `EX-CAP-04`, `EX-ARC-01`, §6.1 ; tolérance
  de 1 % dérivée — au-delà, la charge produite ne serait plus le profil publié.
  *Vérification* : débit mesuré par les horodatages de réception, comparé au débit calculé à 1 %
  près, la fonction de durée étant vérifiée à part sur vecteurs indépendants
  (`SRS-NOY-STA-03`) ; reproductibilité à graine égale ; séquences relues sans trou sur une
  capture sans perte.
- **SRS-OUT-02** [A] Le **banc** DOIT s'exécuter chaque nuit sur les machines de référence x86-64 et
  ARM64 (machines dédiées), publier chaque métrique avec plateforme, modèle de machine, version et
  date, et signaler une régression quand la médiane des 5 dernières exécutions nocturnes se dégrade
  de plus de 10 % par rapport à la référence publiée.
  *Source* : `EX-PRF-05`. *Vérification* : celle d'`EX-PRF-05`.
- **SRS-OUT-03** [A, étendue à chaque palier] Le banc DOIT mesurer, et confronter chacune au seuil
  de son exigence : enregistrement au profil nominal 60 min (`EX-PRF-01`), demi-profil et
  recherche de débit maximal ARM64 (`EX-PRF-02`), décodage paresseux (`EX-DEC-11`), navigation
  dans 200 millions de trames (`EX-JNL-05`), latence d'affichage (`EX-VUE-06`), écart entre les
  horloges de deux canaux (`EX-CAP-03`, ⚙) et, à leur palier, évaluation de référence
  (`EX-PRF-06`) et fidélité du rejeu (`EX-REJ-04`). Il DOIT inclure le générateur publié de la
  trace de 200 millions de trames.
  *Source* : `EX-PRF-01`, `EX-PRF-02`, `EX-PRF-04`, `EX-PRF-05`, `EX-PRF-06`, `EX-DEC-11`,
  `EX-JNL-05`, `EX-VUE-06`, `EX-CAP-03`, `EX-REJ-04`. *Vérification* : chaque métrique présente
  dans la publication nocturne avec son seuil et son verdict ; mesure hors seuil injectée →
  verdict d'échec.
- **SRS-OUT-04** [A] La **matrice de traçabilité** exigence → test(s) → classe → jalon → dernier
  résultat DOIT être générée automatiquement depuis les noms des tests, pour les identifiants
  `EX-*` et SRS, et publiée. La construction échoue si une exigence d'un palier livré, ou d'un
  jalon déclaré atteint, n'a aucun test auto.
  *Source* : §12, `D-23` ; identifiants SRS dérivés — règle 6 du README. *Vérification* :
  exigence d'un palier livré privée de test → échec de construction ; de même pour un jalon
  déclaré atteint ; matrice publiée à chaque construction.
- **SRS-OUT-05** [A] L'**intégration continue** DOIT construire et exécuter la suite auto sur Linux
  x86-64, Linux ARM64 et Windows x86-64, sans matériel, sans serveur graphique, l'entrée standard
  fermée, dans un environnement où l'accès réseau est coupé (espace de noms réseau isolé ou
  équivalent).
  *Source* : `EX-POR-01`, `EX-POR-04`, `EX-AUT-04`. *Vérification* : configuration d'intégration
  continue ; une exécution verte sur les trois plateformes ; un test sentinelle constate l'échec
  de toute connexion sortante.
- **SRS-OUT-06** [A] L'**empaquetage** DOIT produire par plateforme un exécutable autonome complet
  (ligne de commande, démon, interface, pont Tier 2 et python-can) ne nécessitant aucun
  environnement d'exécution installé, un livrable du démon seul sans composant graphique ni pont,
  et une installation par gestionnaire de paquets pour le développement.
  *Source* : `EX-POR-02`, `EX-POR-03`, `D-36`, `O-04`. *Vérification* : celles d'`EX-POR-02` et
  d'`EX-POR-03`.
- **SRS-OUT-07** *Obsolète en version 1.1* : critère de décision d'`O-04`, pas une exigence ;
  repris au point ouvert OUT-1. Identifiant réservé.

## 3. Points ouverts

| # | Point | Renvoi |
|---|---|---|
| OUT-1 | Mécanisme d'empaquetage ; critères de choix d'`O-04` (démarrage à froid sous 3 s, livrable complet sous 300 Mo) ; mesure du livrable Windows sur une machine Windows, la machine de référence étant sous Linux | `O-04` |
| OUT-2 | Hébergement des machines de référence et du banc nocturne (machines dédiées auto-hébergées) | à organiser avant la fin du palier A |
