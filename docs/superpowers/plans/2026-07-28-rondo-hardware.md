# Rondo — Plan d'implémentation : matériel

> **Pour les agents :** SOUS-COMPÉTENCE REQUISE — utiliser `superpowers:subagent-driven-development`
> (recommandé) ou `superpowers:executing-plans` pour dérouler ce plan tâche par tâche.
> Les étapes utilisent la syntaxe case à cocher (`- [ ]`).

**Objectif :** produire tout ce qu'il faut pour fabriquer et câbler la carte électronique du
compteur — nomenclature commandable, schéma électrique vérifié, prototype testé au banc,
brochage du faisceau et notice de montage.

**Architecture :** une carte porteuse passive accueille l'alimentation, la protection, les
étages d'entrée isolés et les capteurs, et vient s'empiler derrière un module ESP32-S3 à
écran rond du commerce. Aucun microcontrôleur n'est routé à la main.

**Outils :** KiCad 8 ou 9, multimètre, alimentation de laboratoire réglable 0-20 V à
limitation de courant, fer à souder.

## Contraintes globales

- Tension d'entrée nominale **12 V continu** (batterie LiFePO4 + régulateur d'origine),
  plage de fonctionnement **9 à 16 V**, transitoires écrêtés au-delà.
- Toute entrée reliée au faisceau 12 V est **isolée galvaniquement par opto-coupleur**.
  Aucune exception : cela protège le MCU et rend le montage tolérant à un câblage
  utilisateur approximatif.
- Toutes les broches du MCU sont **paramétrables par le profil moto** : le schéma fige le
  câblage physique, jamais l'affectation logique. Aucune fonction ne doit dépendre d'une
  broche particulière côté firmware.
- Volume d'intégration : **80 mm de diamètre hors-tout, 50 mm de profondeur**, dans la
  cloche du compteur d'origine.
- Le dépôt ne contient **aucune image** pour l'instant (décision projet). Les livrables
  visuels sont des exports PDF et des fichiers texte.
- Consommation cible **< 1 A sous 5 V**, rétroéclairage compris.
- Toutes les références de composants sont données à titre de **référence type** :
  tout équivalent respectant les caractéristiques indiquées convient.

## Structure des fichiers

| Fichier | Responsabilité |
|---|---|
| `hardware/README.md` | Point d'entrée matériel : quoi commander, quoi fabriquer, dans quel ordre |
| `hardware/bom.csv` | Nomenclature commandable, une ligne par référence |
| `hardware/display-selection.md` | Critères et résultat du choix du module d'écran |
| `hardware/rondo.kicad_pro` / `.kicad_sch` | Projet et schéma KiCad |
| `hardware/rondo-schematic.pdf` | Export PDF du schéma, lisible sans KiCad |
| `hardware/harness.md` | Brochage du connecteur, couleurs de fils, section, longueurs |
| `hardware/bench-test.md` | Procédure de test au banc et relevés attendus |
| `hardware/assembly.md` | Notice de montage mécanique et d'installation sur la moto |

---

## Tâche 1 : Figer le choix du module d'écran

Toute la suite en dépend : le brochage, l'encombrement et le budget découlent de ce choix.

**Fichiers :**
- Créer : `hardware/display-selection.md`

**Produit :** le diamètre retenu, la référence du module, la liste des broches libres
utilisables par la carte porteuse.

- [ ] **Étape 1 : mesurer la cloche**

Au pied à coulisse, sur le support d'origine démonté :
- diamètre intérieur utile (le trou où le module se pose)
- profondeur utile (connue : 50 mm)
- diamètre du passage de câbles sous le support

Consigner les trois valeurs dans `hardware/display-selection.md`.

- [ ] **Étape 2 : appliquer la règle de décision**

```
diamètre intérieur >= 70 mm  ->  module 2.8" rond 480x480
diamètre intérieur <  70 mm  ->  module 2.1" rond 480x480 (PCB ~65 mm)
```

- [ ] **Étape 3 : vérifier les quatre critères bloquants sur la fiche du module retenu**

À vérifier avant commande, et à consigner dans le document :

1. **Résolution 480×480** et interface **RGB parallèle** (les modules SPI sont trop lents
   pour un rafraîchissement à 30 fps sur cette taille).
2. **PSRAM présente** — obligatoire pour le double tampon d'affichage LVGL. Un ESP32-S3
   sans PSRAM ne tiendra pas la cadence.
3. **Tactile capacitif** avec contrôleur documenté (type CST816/CST820/GT911).
4. **Broches libres** : il en faut au minimum **11** — 4 entrées d'opto et de contact,
   2 entrées analogiques, 1 entrée capteur Hall, 2 boutons, 2 pour l'UART GNSS — plus un
   bus I2C accessible pour la FRAM. Un module RGB parallèle consomme beaucoup de broches :
   **c'est le critère qui élimine le plus de modules, le vérifier en premier.**

- [ ] **Étape 4 : relever la luminosité annoncée**

Cible ≥ 800 cd/m². Si le module retenu est en dessous, le noter comme risque dans le
document et prévoir au budget un film antireflet et une casquette pare-soleil imprimée.
Ce n'est pas bloquant pour la suite du plan.

- [ ] **Étape 5 : dresser le tableau des broches disponibles**

Lister dans le document, en tableau, chaque broche exposée du module avec son état
(libre / utilisée par l'écran / utilisée par le tactile). Ce tableau est l'entrée directe
de la tâche 3.

- [ ] **Étape 6 : commit**

```bash
git add hardware/display-selection.md
git commit -m "hardware: fige le choix du module d'ecran"
```

---

## Tâche 2 : Nomenclature commandable

**Fichiers :**
- Créer : `hardware/bom.csv`, `hardware/README.md`

**Consomme :** la référence du module retenue en tâche 1.
**Produit :** `hardware/bom.csv`, utilisé comme liste de commande et comme source des
références dans le schéma.

- [ ] **Étape 1 : écrire `hardware/bom.csv`**

Colonnes : `ref,designation,qte,boitier,reference_type,prix_unitaire_eur,notes`

```csv
ref,designation,qte,boitier,reference_type,prix_unitaire_eur,notes
U1,Module ESP32-S3 ecran rond 480x480 tactile,1,module,Waveshare ESP32-S3-Touch-LCD-2.1,45.00,Voir hardware/display-selection.md
U2,Module abaisseur 12V vers 5V 3A,1,module,TPS5430,4.00,Entree 36V min obligatoire - verifier avant commande
U3,FRAM I2C 64 kbit,1,SOP-8,MB85RC64TA,3.00,Adresse 0x50 - A0 A1 A2 a la masse
U4,Module GNSS multi-constellation,1,module,u-blox NEO-M9N,30.00,UART 3.3V - antenne active incluse
U5,Opto-coupleur,1,DIP-4,PC817C,0.30,Clignotant gauche
U6,Opto-coupleur,1,DIP-4,PC817C,0.30,Clignotant droit
U7,Opto-coupleur,1,DIP-4,PC817C,0.30,Plein phare
U8,Capteur effet Hall unipolaire,1,TO-92,US5881LUA,2.50,Sortie collecteur ouvert - alim 3.8 a 24V
D1,Diode Schottky 3A 40V,1,SMA,SS34,0.20,Protection inversion de polarite
D2,Diode transil unidirectionnelle 24V,1,SMB,SMBJ24A,0.40,Ecretage load dump
D3,Diode signal,3,DO-35,1N4148,0.05,Protection inverse LED opto
D4,Diode double Schottky de clamp,2,SOT-23,BAT54S,0.20,Clamp entrees point mort et boutons
F1,Porte-fusible etanche en ligne + fusible lame 2A,1,-,ATO,2.00,Sur le +12V
C1,Condensateur electrolytique 1000uF 35V faible ESR,1,radial,-,0.80,Reservoir d'entree
C2,Condensateur electrolytique 10uF 25V,2,radial,-,0.10,Decouplage
C3,Condensateur ceramique 100nF X7R 50V,12,0805 ou pas 2.54,-,0.05,Decouplage et filtrage
C4,Condensateur ceramique 10nF X7R,1,0805 ou pas 2.54,-,0.05,Filtre capteur Hall
R1,Resistance 2.2k 1/4W 1%,3,0805 ou axial,-,0.05,Serie LED opto
R2,Resistance 10k 1/4W 1%,8,0805 ou axial,-,0.05,Pull-up
R3,Resistance 1k 1/4W 1%,4,0805 ou axial,-,0.05,Serie protection entrees
R4,Resistance 100k 1/4W 1%,1,0805 ou axial,-,0.05,Diviseur VBAT branche haute
R5,Resistance 15k 1/4W 1%,1,0805 ou axial,-,0.05,Diviseur VBAT branche basse
R6,Resistance 4.7k 1/4W 1%,2,0805 ou axial,-,0.05,Pull-up I2C - a poser seulement si absents du module
J1,Connecteur etanche 12 voies + contacts,1,-,Deutsch DT06-12S / DT04-12P,12.00,Cote carte et cote faisceau
SW1-SW2,Bouton poussoir etanche guidon,2,-,-,6.00,NO - contact a la masse
M1,Aimant neodyme 6x3 mm,4,-,N42,0.50,2 utiles + 2 de rechange
W1,Cable blinde 3 conducteurs 0.25 mm2,1.5,metre,-,2.00,Liaison capteur Hall
W2,Fil souple 0.5 mm2 multibrins,10,metre,-,0.50,Faisceau - couleurs selon hardware/harness.md
P1,Plaque a trous 70x70 mm pas 2.54,1,-,-,3.00,Prototype
P2,Plaque aluminium 1.5 mm decoupee,1,-,-,3.00,Dissipateur arriere
P3,Pad thermique silicone 1 mm,1,-,-,2.00,Entre carte et plaque alu
P4,Presse-etoupe M12 etanche,1,-,IP68,2.00,Passage de cables sous le support
P5,Joint torique,1,-,-,1.00,Diametre selon mesure tache 1
```

- [ ] **Étape 2 : calculer et vérifier le total**

```bash
awk -F, 'NR>1 {gsub(/"/,"",$3); gsub(/"/,"",$6); t+=$3*$6} END {printf "Total: %.2f EUR\n", t}' hardware/bom.csv
```

Attendu : **123,00 €**, et dans tous les cas un total compris entre **110 et 140 €**. Si le
résultat sort de cette plage, c'est qu'une quantité ou un prix est erroné — relire la ligne
fautive.

Ce total n'inclut ni les frais de port, ni la fabrication d'une carte gravée si le prototype
sur plaque à trous est ensuite converti en PCB. Prévoir **150 à 180 € tout compris**.

- [ ] **Étape 3 : écrire `hardware/README.md`**

Doit contenir, dans cet ordre : à quoi sert le matériel, la nomenclature à commander en
premier (les délais les plus longs sont le module d'écran, le GNSS et le connecteur
Deutsch), l'ordre de fabrication (prototype plaque à trous, puis carte définitive), et un
renvoi vers `assembly.md` et `bench-test.md`.

Rappeler explicitement le point de vérification bloquant : **le module abaisseur doit
accepter au moins 36 V en entrée**. Les modules courants à base de MP1584 sont limités à
28 V et seraient détruits par l'écrêtage à 38,9 V de la transil D2.

- [ ] **Étape 4 : commit**

```bash
git add hardware/bom.csv hardware/README.md
git commit -m "hardware: nomenclature commandable et guide de commande"
```

---

## Tâche 3 : Projet KiCad et bloc alimentation

**Fichiers :**
- Créer : `hardware/rondo.kicad_pro`, `hardware/rondo.kicad_sch`

**Consomme :** le tableau de broches de la tâche 1, les références de `bom.csv`.
**Produit :** les nets `+12V_PROT`, `+5V`, `+3V3`, `GND`, utilisés par toutes les tâches
suivantes.

- [ ] **Étape 1 : créer le projet KiCad**

Nouveau projet `hardware/rondo`. Feuille A3, cartouche renseigné : titre « Rondo — carte
porteuse », révision A, licence.

- [ ] **Étape 2 : saisir la chaîne d'alimentation**

Ordre des composants depuis l'entrée, **cet ordre est fonctionnel et ne doit pas changer** :

```
J1.1 (+12V moto)
  -> F1  fusible lame 2 A
  -> D1  SS34 en serie, cathode vers l'aval   [protection inversion de polarite]
  -> noeud +12V_PROT
       |- D2  SMBJ24A vers GND                [ecretage transitoires]
       |- C1  1000 uF / 35 V vers GND         [reservoir]
       |- C3a 100 nF vers GND                 [decouplage HF]
       |- diviseur VBAT (tache 5)
       -> U2  module abaisseur 12V -> 5V
            -> +5V
                 |- C2a 10 uF vers GND
                 |- C3b 100 nF vers GND
                 -> U1 (module ecran, entree 5V)
                 -> U8 (capteur Hall, alimentation)
```

Le `+3V3` est **fourni par le régulateur du module U1**, il n'est pas généré sur la carte
porteuse. Il alimente U3 (FRAM), U4 (GNSS), et sert de référence à tous les pull-ups.

- [ ] **Étape 3 : justifier les valeurs en note de schéma**

Ajouter un bloc de texte sur la feuille, reprenant ces calculs :

```
D2 SMBJ24A : tension de service 24 V > 16 V max en charge, ecretage 38,9 V.
             U2 doit donc accepter >= 36 V en entree.
C1 1000 uF : reservoir contre les creux de tension du faisceau.
             AUCUN role dans la persistance de l'odometre : la FRAM est
             ecrite en continu tous les 10 m (voir spec 5.6). Pas de
             detection de coupure, pas de comparateur.
Bilan conso : ecran + retroeclairage ~500 mA, ESP32-S3 en pointe WiFi
             ~350 mA, GNSS ~35 mA -> ~900 mA sous 5 V.
             U2 dimensionne a 3 A : marge x3.
```

- [ ] **Étape 4 : lancer le contrôle des règles électriques**

Dans KiCad : *Inspect → Electrical Rules Checker*.
Attendu à ce stade : **zéro erreur**. Les avertissements « broche non connectée » sur les
blocs pas encore saisis sont normaux et seront résorbés en tâche 6.

- [ ] **Étape 5 : commit**

```bash
git add hardware/rondo.kicad_pro hardware/rondo.kicad_sch
git commit -m "hardware: projet KiCad et bloc alimentation protege"
```

---

## Tâche 4 : Entrées isolées et entrées de contact

**Fichiers :**
- Modifier : `hardware/rondo.kicad_sch`

**Consomme :** nets `+12V_PROT`, `+3V3`, `GND` de la tâche 3.
**Produit :** nets `IN_TURN_L`, `IN_TURN_R`, `IN_HIGH_BEAM` (logique **inversée**),
`IN_NEUTRAL` (logique inversée), vers les broches libres de U1.

- [ ] **Étape 1 : saisir les trois étages isolés (clignotant G, clignotant D, plein phare)**

Le motif est identique pour U5, U6 et U7 — le saisir trois fois à l'identique :

```
Entree faisceau 12 V (J1.3 / J1.4 / J1.5)
  -> R1 2,2 k -> anode LED de l'opto
  -> cathode LED -> GND faisceau (J1.2)
  D3 1N4148 en antiparallele sur la LED   [protection inverse]

Cote isole :
  collecteur -> R2 10 k -> +3V3, et -> broche MCU
  emetteur   -> GND
  C3 100 nF entre collecteur et GND       [filtrage]
```

- [ ] **Étape 2 : noter le dimensionnement et la polarité en note de schéma**

```
R1 = 2,2 k : If = (12 - 1,2) / 2200 = 4,9 mA a 12 V
                (14,4 - 1,2) / 2200 = 6,0 mA a 14,4 V (charge)
             Dissipation max = 6 mA^2 x 2200 = 79 mW -> 1/4 W suffit.
CTR du PC817C a 5 mA : 100 % mini -> Ic >= 4,9 mA.
Saturation avec pull-up 10 k sous 3,3 V : 0,33 mA requis. Marge x15.

LOGIQUE INVERSEE : 12 V present -> transistor sature -> broche MCU a l'etat BAS.
Le profil moto doit declarer ces entrees en actif-bas.
```

- [ ] **Étape 3 : saisir l'entrée point mort**

Le contacteur de point mort d'origine met la ligne **à la masse**. Aucun 12 V n'est présent,
l'isolation optique est donc inutile ici :

```
J1.6 -> R3 1 k -> noeud IN_NEUTRAL
IN_NEUTRAL -> R2 10 k -> +3V3      [pull-up]
IN_NEUTRAL -> C3 100 nF -> GND     [anti-rebond]
IN_NEUTRAL -> D4 BAT54S            [clamp vers +3V3 et GND]
```

Logique inversée également : point mort engagé → contact fermé → état BAS.

- [ ] **Étape 4 : contrôle des règles électriques**

*Inspect → Electrical Rules Checker*. Attendu : zéro erreur.

- [ ] **Étape 5 : commit**

```bash
git add hardware/rondo.kicad_sch
git commit -m "hardware: entrees isolees clignotants, phare et point mort"
```

---

## Tâche 5 : Entrées analogiques et capteur de vitesse

**Fichiers :**
- Modifier : `hardware/rondo.kicad_sch`

**Consomme :** nets `+12V_PROT`, `+5V`, `+3V3`, `GND`.
**Produit :** nets `ADC_VBAT`, `ADC_TEMP`, `IN_SPEED` vers les broches libres de U1.

- [ ] **Étape 1 : saisir le diviseur de tension batterie**

```
+12V_PROT -> R4 100 k -> noeud ADC_VBAT -> R5 15 k -> GND
ADC_VBAT  -> C3 100 nF -> GND
```

Note de schéma à ajouter :

```
Rapport = 15 / (100 + 15) = 0,1304
 9,0 V -> 1,17 V      (batterie a plat)
12,8 V -> 1,67 V      (LiFePO4 au repos)
14,4 V -> 1,88 V      (en charge)
18,0 V -> 2,35 V      (defaut regulateur - reste dans la plage ADC)
Plage ADC ESP32-S3 en attenuation 12 dB : 0 a ~3,1 V. Marge conservee.
Courant permanent : 14,4 / 115 k = 125 uA, coupe avec le contact.

Ne JAMAIS deduire un pourcentage de charge de cette tension : la courbe de
decharge d'une LiFePO4 est quasi plate entre 13,2 et 12,8 V. On affiche la
tension brute, avec des seuils d'alerte declares dans le profil moto.
```

- [ ] **Étape 2 : saisir l'entrée sonde de température**

La sonde d'origine est une thermistance dont le retour se fait par la masse moteur :

```
+3V3 -> R2 10 k -> noeud ADC_TEMP -> J1.10 (sonde) -> masse moteur
ADC_TEMP -> C3 100 nF -> GND
```

Note de schéma :

```
Pont diviseur avec la thermistance en branche basse.
La courbe de linearisation est une table dans le profil moto, PAS dans le
code : la sonde differe d'un modele de moto a l'autre.
La masse de la carte doit etre reliee a la masse moteur par un fil dedie
(J1.2), pas seulement par le chassis : sinon la mesure derive.
```

- [ ] **Étape 3 : saisir l'entrée capteur Hall**

```
U8 US5881LUA :
  broche 1 (VDD)  -> +5V, avec C3 100 nF vers GND au plus pres
  broche 2 (GND)  -> GND
  broche 3 (OUT)  -> collecteur ouvert -> R2 10 k -> +3V3
                     -> R3 1 k -> noeud IN_SPEED
IN_SPEED -> C4 10 nF -> GND
```

Le capteur est déporté au bout du câble blindé W1. **Le blindage n'est relié à la masse
que du côté carte**, jamais des deux côtés — sinon il forme une boucle de masse qui capte
l'allumage.

Note de schéma :

```
Filtre RC 1 k / 10 nF : fc = 1 / (2 pi x 1000 x 10e-9) = 15,9 kHz
Frequence utile maximale :
  120 km/h = 33,3 m/s ; circonference 2,17 m (roue 21 pouces)
  -> 15,4 tr/s x 2 aimants = 30,7 Hz
Marge x500 entre le signal utile et la coupure du filtre : le filtre elimine
les parasites d'allumage sans jamais tronquer une impulsion utile.

Sortie collecteur ouvert : le pull-up vers +3V3 est compatible avec une
alimentation du capteur en 5 V. Pas d'adaptateur de niveau necessaire.
```

- [ ] **Étape 4 : contrôle des règles électriques**

*Inspect → Electrical Rules Checker*. Attendu : zéro erreur.

- [ ] **Étape 5 : commit**

```bash
git add hardware/rondo.kicad_sch
git commit -m "hardware: entrees analogiques et capteur de vitesse a effet Hall"
```

---

## Tâche 6 : Bus numériques, boutons, connecteur et export

**Fichiers :**
- Modifier : `hardware/rondo.kicad_sch`
- Créer : `hardware/rondo-schematic.pdf`

**Consomme :** tous les nets des tâches 3 à 5.
**Produit :** le schéma complet et vérifié, exporté en PDF lisible sans KiCad.

- [ ] **Étape 1 : saisir la FRAM**

```
U3 MB85RC64 :
  VDD -> +3V3, C3 100 nF vers GND au plus pres
  VSS -> GND
  A0, A1, A2 -> GND        [adresse I2C 0x50]
  WP -> GND                [ecriture autorisee]
  SDA, SCL -> bus I2C de U1
R6 4,7 k sur SDA et SCL vers +3V3 : A NE POSER QUE si le module U1 n'en a pas
deja. Verifier a l'ohmmetre avant de souder - des pull-ups en double font
chuter les fronts et corrompent le bus.
```

- [ ] **Étape 2 : saisir le module GNSS**

```
U4 NEO-M9N :
  VCC -> +3V3, C2 10 uF + C3 100 nF vers GND
  GND -> GND
  TX  -> broche RX de U1
  RX  -> broche TX de U1
Antenne active deportee a l'exterieur de la cloche.
Liaison serie 38400 bauds, niveaux 3,3 V, pas d'adaptation necessaire.
```

- [ ] **Étape 3 : saisir les deux boutons**

Motif identique pour SW1 et SW2 :

```
J1.11 / J1.12 -> R3 1 k -> noeud IN_BTN_A / IN_BTN_B
  -> R2 10 k -> +3V3        [pull-up]
  -> C3 100 nF -> GND       [anti-rebond materiel]
  -> D4 BAT54S              [clamp]
Boutons cables entre la broche et la masse : appui -> etat BAS.
```

- [ ] **Étape 4 : saisir le connecteur J1 et figer le brochage**

```
1  +12V apres contact       rouge
2  Masse                    noir
3  Clignotant gauche        vert clair
4  Clignotant droit         vert fonce
5  Plein phare              bleu
6  Point mort               marron
7  Capteur Hall +5V         orange
8  Capteur Hall signal      blanc
9  Capteur Hall masse       noir (blindage, relie cote carte uniquement)
10 Sonde temperature        jaune
11 Bouton A                 gris
12 Bouton B                 violet
```

- [ ] **Étape 5 : contrôle des règles électriques sur le schéma complet**

*Inspect → Electrical Rules Checker*.
Attendu : **zéro erreur et zéro avertissement**. Toute broche réellement inutilisée doit
porter un symbole « non connecté » explicite, pas être laissée en l'air.

- [ ] **Étape 6 : exporter le PDF**

*File → Plot → format PDF*, sortie `hardware/rondo-schematic.pdf`.

Vérifier en ouvrant le PDF que les notes de calcul des tâches 3 à 5 sont lisibles : elles
constituent la justification du dimensionnement pour quiconque reprend le projet.

- [ ] **Étape 7 : commit**

```bash
git add hardware/rondo.kicad_sch hardware/rondo-schematic.pdf
git commit -m "hardware: bus numeriques, boutons, connecteur et export du schema"
```

---

## Tâche 7 : Prototype sur plaque à trous et validation au banc

C'est la tâche qui attrape les erreurs avant qu'elles ne coûtent une carte gravée ou un
module d'écran grillé.

**Fichiers :**
- Créer : `hardware/bench-test.md`

**Consomme :** le schéma de la tâche 6.
**Produit :** un prototype fonctionnel et un relevé de mesures signé.

- [ ] **Étape 1 : câbler l'étage d'alimentation seul**

Sur la plaque P1, monter uniquement F1, D1, D2, C1, C3a et U2. **Ne connecter ni le module
d'écran ni aucun autre composant.**

- [ ] **Étape 2 : essai en tension sans charge**

Alimentation de laboratoire, limitation de courant réglée à **100 mA**.
Monter progressivement de 0 à 14,4 V en mesurant la sortie de U2.

Attendu :
```
Entree 9,0 V  -> sortie 5,00 V +/- 0,15 V
Entree 12,0 V -> sortie 5,00 V +/- 0,15 V
Entree 14,4 V -> sortie 5,00 V +/- 0,15 V
Courant absorbe a vide < 30 mA
```

Si la limitation de courant se déclenche : **couper immédiatement**, un composant est à
l'envers. Vérifier le sens de D1 et de C1 avant toute autre chose.

- [ ] **Étape 3 : essai d'inversion de polarité**

Limitation de courant à **100 mA**. Appliquer **−12 V** à l'entrée pendant 5 secondes.

Attendu : sortie de U2 à **0 V**, courant absorbé **quasi nul**, aucun échauffement.
C'est le rôle de D1. Si du courant passe, D1 est montée à l'envers — la corriger avant
d'aller plus loin, sous peine de détruire le module d'écran à la première erreur de câblage.

- [ ] **Étape 4 : monter le reste des composants**

Ajouter les étages des tâches 4 et 5, puis seulement en dernier le module d'écran U1.

- [ ] **Étape 5 : valider chaque entrée une par une**

| Entrée | Stimulus | Attendu au multimètre sur la broche MCU |
|---|---|---|
| Clignotant G / D, plein phare | +12 V appliqué sur la voie J1 correspondante | passe de 3,3 V à < 0,4 V |
| Point mort | J1.6 relié à la masse | passe de 3,3 V à < 0,2 V |
| Boutons A / B | appui | passe de 3,3 V à < 0,2 V |
| Tension batterie | entrée à 12,0 V | 1,565 V ± 0,05 V sur `ADC_VBAT` |
| Tension batterie | entrée à 14,4 V | 1,878 V ± 0,05 V sur `ADC_VBAT` |
| Capteur Hall | aimant approché face marquée | passe de 3,3 V à < 0,4 V |

Le capteur Hall est **unipolaire** : il ne réagit qu'à une seule polarité. S'il ne réagit
pas, retourner l'aimant avant de conclure à une panne. **Repérer la face active de l'aimant
d'un point de peinture** : c'est cette face qui devra être orientée vers le capteur au
montage sur le disque.

- [ ] **Étape 6 : consigner les relevés dans `hardware/bench-test.md`**

Reprendre la procédure ci-dessus, avec une colonne « mesuré » à remplir et la date de
l'essai. Ce document sert de recette de réception à quiconque refabrique la carte.

- [ ] **Étape 7 : commit**

```bash
git add hardware/bench-test.md
git commit -m "hardware: procedure de test au banc et releves du prototype"
```

---

## Tâche 8 : Faisceau, montage mécanique et notice

**Fichiers :**
- Créer : `hardware/harness.md`, `hardware/assembly.md`

**Consomme :** le brochage figé en tâche 6, les relevés de la tâche 7.
**Produit :** de quoi installer le compteur sur la moto sans relire le schéma.

- [ ] **Étape 1 : écrire `hardware/harness.md`**

Reprendre le tableau de brochage de la tâche 6, en ajoutant pour chaque voie : section de
fil (0,5 mm² partout sauf le +12 V et la masse en 0,75 mm²), longueur, et **où se piquer
sur le faisceau d'origine**. Préciser que les points de piquage diffèrent selon le modèle
de moto et que seule la KDX 125 est documentée pour l'instant.

Ajouter un avertissement : les trois entrées isolées supportent indifféremment un signal
12 V positif ou négatif tant qu'il traverse la LED dans le bon sens, grâce à D3. En cas
de doute sur la polarité d'une sortie du faisceau, **mesurer avant de brancher**.

- [ ] **Étape 2 : écrire la section montage du capteur dans `hardware/assembly.md`**

```
2 aimants Neodyme 6x3 mm a 180 degres sur le disque de frein avant.
Points de fixation possibles, par ordre de preference :
 1. dans les ajours du disque, colles a l'epoxy structurale
 2. sur deux tetes de vis de disque opposees
Face active reperee (voir tache 7, etape 5) tournee vers le capteur.

Support capteur boulonne sur la patte d'etrier, entrefer 2 a 4 mm.
Verifier l'entrefer roue tournee a la main sur un tour complet : le disque
voile toujours un peu.
Cable blinde plaque le long du fourreau, avec du mou en boucle au niveau
de la fourche pour absorber le debattement complet. Verifier en comprimant
la fourche a fond que le cable n'est jamais tendu.

Le nombre d'aimants et la circonference de roue sont des parametres du
profil moto : changer le nombre d'aimants ne demande aucune modification
du code.
```

- [ ] **Étape 3 : écrire la section intégration dans la cloche**

```
Empilage depuis l'avant, dans 50 mm de profondeur :
  module ecran U1            ~12 mm
  entretoises                  5 mm
  carte porteuse             ~15 mm avec les composants
  plaque aluminium P2 + pad thermique P3
  passage de cables et presse-etoupe P4

La plaque aluminium P2 est en contact avec le support d'origine, qui sert
de dissipateur. NE PAS omettre : volume clos de 80 x 50 mm avec le
retroeclairage en plein soleil.

Joint torique P5 entre le module d'ecran et la cloche.
Presse-etoupe P4 dans le passage de cables existant sous le support.
Vernis de tropicalisation sur la carte porteuse APRES validation au banc.
Colle souple sur les connecteurs, mousse anti-vibration derriere la carte.

Antenne GNSS deportee a l'exterieur de la cloche, vue ciel degagee. La
dalle et le support metallique la masquent totalement si elle est a
l'interieur.
```

- [ ] **Étape 4 : relire la notice du point de vue d'un tiers**

Vérifier que quelqu'un qui n'a pas suivi le projet peut monter le compteur avec seulement
`bom.csv`, `rondo-schematic.pdf`, `harness.md` et `assembly.md`. Toute étape qui suppose
une connaissance implicite doit être explicitée.

- [ ] **Étape 5 : commit**

```bash
git add hardware/harness.md hardware/assembly.md
git commit -m "hardware: faisceau, montage mecanique et notice d'installation"
```

---

## Suites

Ce plan couvre le matériel. Les plans suivants, à rédiger séparément, reprennent les phases
de la spec :

| Plan | Contenu | Peut démarrer |
|---|---|---|
| `rondo-core` | Couche matérielle abstraite, simulateur PC, bus de canaux, harnais de test | immédiatement, en parallèle des commandes |
| `rondo-acquisition` | Providers Hall, entrées TOR, thermistance, tension ; odomètre et FRAM | après ce plan et `rondo-core` |
| `rondo-ui` | Bibliothèque de widgets, 3 archétypes, 6 thèmes, navigation aux boutons | après `rondo-core` |
| `rondo-config` | Portail WiFi, profil JSON, import/export | après `rondo-acquisition` |
| `rondo-nav` | GNSS, chargement et suivi de trace GPX | après `rondo-config` |

`rondo-core` ne dépend d'aucun matériel : c'est le plan à enchaîner pendant que les pièces
sont en transit.
