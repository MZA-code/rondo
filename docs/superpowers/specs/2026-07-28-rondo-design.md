# Rondo — Compteur moto numérique open source

**Date :** 2026-07-28
**Statut :** conception validée, prêt pour plan d'implémentation
**Dépôt :** `github.com/MZA-code/rondo`

## 1. Objectif

Remplacer le compteur d'origine d'une Kawasaki KDX 125 par un tableau de bord numérique
sur écran rond tactile, intégré dans la cloche du support d'origine.

Le projet est publié en open source. Il doit donc être **utilisable sur d'autres modèles de
moto sans modification du code**, uniquement par configuration. La KDX 125 est la moto de
référence, pas la cible unique : rien dans le code ne doit lui être spécifique, tout ce qui
la concerne vit dans un profil sous `profiles/`.

Le nom « Rondo » renvoie à la forme ronde de l'écran et à la forme musicale bâtie sur un
thème qui revient — le compteur est construit autour de thèmes graphiques interchangeables.

## 2. Contexte matériel

La moto reçoit une fourche Kayaba de KX 125/250 de 1994, qui ne possède pas d'entraînement
mécanique de compteur dans la roue. La mesure de vitesse et le kilométrage doivent donc être
reconstruits électroniquement.

La batterie d'origine est remplacée par une batterie lithium 12 V (LiFePO4), avec le
régulateur/redresseur d'origine. L'alimentation disponible est donc du 12 V continu stable.

Support d'origine : 80 mm hors-tout, 50 mm de profondeur utile, avec un passage de câbles
existant sous le support. Le compteur central a été déposé, seule la cloche est conservée.

## 3. Décisions structurantes

| Sujet | Décision | Raison |
|---|---|---|
| Navigation | Suivi de trace GPX (flèche, écart, waypoints, breadcrumb) — **pas de fond de carte** | Tient sur un seul MCU, boot instantané, correspond à l'usage réel en trail/enduro |
| Cerveau | ESP32-S3 unique | Suffisant sans fond de carte ; démarrage immédiat, pas de risque de corruption SD |
| Vitesse | Capteur à effet Hall sur disque de frein avant | Éprouvé en tout-terrain (Trail Tech Vapor/Vector), fiable, insensible au couvert forestier |
| Commandes | Tactile capacitif + 2 boutons au guidon | Le capacitif ne répond pas à travers un gant d'enduro |
| Développement | Couche matérielle abstraite + simulateur PC | Permet de développer l'UI et de tester la logique critique sans matériel |
| Configuration | Portail WiFi + profils moto en JSON | Aucune recompilation pour un nouvel utilisateur ; profils partageables via le dépôt |
| Généricité v1 | Interfaces génériques, une seule implémentation par rôle | Extension sans refonte, sans déboguer du code non testable sur la moto de référence |

**Explicitement hors périmètre :** fond de carte raster ou vectoriel, routing, bus CAN,
calculs astronomiques (lever/coucher du soleil, phase de lune).

## 4. Matériel

### 4.1 Écran et calculateur

Module tout-en-un ESP32-S3 à écran rond 480×480 avec tactile capacitif (type Waveshare
ESP32-S3-Touch-LCD-2.1). Le module intègre le contrôleur d'affichage RGB, ce qui évite de
câbler un bus parallèle.

**Règle de décision sur le diamètre :** mesurer le diamètre intérieur utile de la cloche.
Si ≥ 70 mm, un module 2.8" est possible ; sinon retenir le 2.1" (PCB ≈ 65 mm).
La profondeur de 50 mm est suffisante dans les deux cas.

**Luminosité :** vérifier la valeur annoncée avant achat. Viser ≥ 800 cd/m². Prévoir un film
antireflet et une casquette pare-soleil imprimée si nécessaire.

**Thermique :** platine arrière en aluminium avec pad thermique en contact avec le support
d'origine, qui sert de dissipateur. Volume clos de 80 × 50 mm, ESP32-S3 et rétroéclairage
en plein soleil : à traiter à la conception, très coûteux à rattraper après.

### 4.2 Capteur de vitesse

- Capteur à effet Hall **unipolaire** (US5881 ou A1104). Un latch bipolaire imposerait
  d'alterner les polarités des aimants ; l'unipolaire fonctionne avec plusieurs aimants
  identiques orientés dans le même sens.
- 2 aimants néodyme Ø6 × 3 mm à 180°, encastrés dans les ajours du disque avant ou fixés
  sur les têtes de vis du disque. Le nombre d'aimants est un paramètre du profil moto.
- Support capteur boulonné sur la patte d'étrier, câble blindé le long du fourreau.
- Circonférence de roue configurable, calibrable par mesure d'un tour de roue.

### 4.3 Entrées faisceau

| Signal | Interface |
|---|---|
| Clignotant gauche | Opto-coupleur PC817 + filtre RC |
| Clignotant droit | Opto-coupleur PC817 + filtre RC |
| Plein phare | Opto-coupleur PC817 |
| Point mort | Entrée à la masse, pull-up + RC |
| Tension batterie | Pont diviseur 10:1 + filtre → ADC |
| Température | Entrée ADC, courbe de linéarisation en table dans le profil |

L'isolation optique des entrées 12 V est obligatoire : elle protège le MCU des pics du
faisceau et rend le montage tolérant à un câblage utilisateur approximatif.

### 4.4 Alimentation

Convertisseur abaisseur 12 V → 5 V (≥ 2 A) avec fusible, protection contre l'inversion de
polarité et écrêtage des surtensions transitoires (load dump). Condensateur réservoir
1000 µF en amont, pour encaisser les creux de tension du faisceau.

Aucune détection de coupure d'alimentation n'est nécessaire : voir §5.6.

### 4.5 Persistance

FRAM I2C (MB85RC64). Endurance en écriture quasi illimitée, écriture en microsecondes,
pas de gestion d'usure à implémenter, aucune perte de donnée sur coupure brutale.

### 4.6 GPS

Module u-blox NEO-M9N en liaison série, antenne céramique déportée à l'extérieur de la
cloche (la dalle et le support masquent le ciel). Traces GPX stockées en mémoire flash
(LittleFS) ou sur carte microSD selon le module retenu, et téléversées via le portail WiFi.

### 4.7 Commandes

Deux poussoirs étanches au guidon, câblés en pull-up. Appui court et appui long donnent
quatre actions accessibles avec des gants.

### 4.8 Intégration

Cloche d'origine conservée. Carte porteuse empilée derrière le module d'écran, boîtier
imprimé 3D en fond de cloche, joint torique, connecteur étanche type Deutsch DT sur le
passage de câbles existant. Conformal coating sur les cartes, mousse anti-vibration et
colle sur les connecteurs.

Budget matériel estimé : 120 à 180 €.

## 5. Architecture logicielle

### 5.1 Stack

ESP-IDF, LVGL 9, build PlatformIO/CMake. LVGL fournit un pilote SDL2 : le code d'interface
compile à l'identique sur PC et sur la cible.

### 5.2 Couches

```
┌──────────────────────────────────────────────┐
│ UI (LVGL)      écrans, thèmes, widgets       │  PC + ESP32
├──────────────────────────────────────────────┤
│ Core           bus de canaux, odo/trip,      │  C++ pur, testable
│                navigation GPX, états         │
├──────────────────────────────────────────────┤
│ Providers      hall_speed, gpio_tor,         │  interfaces communes
│                ntc_analog, gnss_uart         │
├──────────────────────────────────────────────┤
│ HAL            GPIO, ADC, UART, I2C,         │  impl. esp32 | sim
│                timers, stockage, horloge     │
└──────────────────────────────────────────────┘
```

Les deux couches supérieures ne référencent jamais d'API matérielle. C'est ce qui rend le
simulateur possible et les tests unitaires réalistes.

### 5.3 Bus de canaux

Pièce maîtresse de l'architecture. Un **canal** est composé de :

- un identifiant typé (`SPEED`, `ODO`, `TRIP`, `RPM`, `TEMP`, `FUEL`, `VBAT`, `NEUTRAL`,
  `TURN_L`, `TURN_R`, `HIGH_BEAM`, `GNSS_FIX`, `HEADING`, `NAV_XTE`, …) ;
- une valeur et son unité ;
- un horodatage ;
- un **état de validité** : `frais`, `périmé`, `absent`.

Les providers publient sur le bus, l'UI s'y abonne. Les deux ne se connaissent pas.

L'état `absent` est le mécanisme central du multi-moto : si un profil ne déclare aucune
source pour la jauge d'essence, le canal `FUEL` est absent et **tout widget qui en dépend se
masque automatiquement**. Un thème fonctionne alors sur n'importe quelle moto sans être
réécrit. L'état `périmé` doit toujours être visible à l'écran : aucune valeur figée ne doit
être affichée comme si elle était à jour.

### 5.4 Providers

Interface unique : un provider déclare les canaux qu'il alimente, s'initialise à partir d'un
fragment du profil moto, et publie ses valeurs sur le bus.

Implémentations en v1 : `hall_speed` (vitesse et distance depuis le capteur Hall),
`gpio_tor` (entrées tout ou rien), `ntc_analog` (température), `vbat_analog`,
`gnss_uart` (position, cap, heure).

Un provider de bus CAN, un provider de vitesse sur pignon de sortie de boîte, ou d'autres
courbes de jauges pourront être ajoutés sans modification du core ni de l'UI. Ils ne sont
pas implémentés en v1.

### 5.5 Thèmes

Les six styles du mockup se ramènent à **trois archétypes de mise en page** :

1. **Aiguille analogique** — thème « Classic »
2. **Arc de vitesse + grand chiffre** — thèmes « Moderne rond », « Cyberpunk »
3. **Chiffre nu + graduations** — thèmes « Arcade », « Rallye/Dakar », « Minimaliste »

On construit une bibliothèque de widgets paramétrés (arc de vitesse, bloc numérique, rangée
de témoins, ligne d'état) et un thème devient : un archétype + une palette + une police.
Les six styles sortent pour le coût de trois, et un contributeur peut en ajouter un en
quelques lignes.

Les thèmes sont en code C++ derrière une interface commune. Ils ne lisent que le bus de
canaux, jamais un provider ni le matériel.

### 5.6 Odomètre et trip

L'intégration de la distance vit dans le core et se teste unitairement sur PC.

Persistance : valeur écrite en FRAM en **double exemplaire avec CRC**, relecture avec
sélection de la copie valide au démarrage.

**L'écriture est continue, tous les 10 mètres parcourus.** Une FRAM supporte plus de 10¹²
cycles d'écriture : à 100 km/h cela représente une écriture toutes les 0,36 s, soit environ
10 millions d'écritures sur 100 000 km — quatre ordres de grandeur sous la limite du
composant. La valeur en mémoire est donc toujours à jour à 10 mètres près.

Il n'y a par conséquent **rien à sauvegarder au moment de la coupure du contact** : pas de
détection de brownout, pas de comparateur, pas d'ISR de coupure. C'est la raison d'être du
choix de la FRAM plutôt que d'une EEPROM.

Un **offset d'odomètre configurable** permet de repartir du kilométrage réel de la moto.

### 5.7 Navigation

Chargement d'une trace GPX, simplification Douglas-Peucker au chargement pour tenir en
mémoire, projection de la position courante sur la trace, calcul du cap à suivre, de l'écart
latéral à la trace et de la distance au prochain waypoint. Breadcrumb du parcours effectué.

Aucun fond de carte n'est affiché.

### 5.8 Profil moto et portail de configuration

Le **profil moto** est un document JSON décrivant :

- le mappage des broches et la polarité de chaque entrée ;
- la source de vitesse, le nombre d'aimants, la circonférence de roue ;
- les courbes de linéarisation des capteurs (NTC de température, jauge d'essence — les
  sondes vont de 0-90 Ω en européen à 240-33 Ω en américain) ;
- les impulsions par tour pour le régime ;
- les seuils d'alerte, dépendants de la chimie de batterie (une LiFePO4 a une courbe de
  décharge quasi plate : on affiche la tension brute, jamais un pourcentage de charge
  déduit de la tension) ;
- le thème actif et les unités.

Le profil est stocké en NVS. Les profils validés par modèle de moto sont versionnés dans le
dépôt, dans `profiles/`.

**Portail de configuration :** point d'accès WiFi porté par l'ESP32, serveur HTTP embarqué,
page unique servie depuis LittleFS, API REST JSON. Import et export de profil. Le portail
n'est activable **qu'à l'arrêt ou par appui long** : pas de WiFi actif en roulant.

### 5.9 Temps réel

Sous FreeRTOS :

| Tâche | Cadence | Priorité |
|---|---|---|
| ISR capteur Hall (timer capture) | événementiel | maximale |
| Acquisition et publication des canaux | 50 Hz | haute |
| GNSS (série) | 10 Hz | moyenne |
| Interface LVGL | 30 fps | basse |
| Persistance FRAM | tous les 10 m parcourus | basse |

**Règle non négociable :** l'interface ne doit jamais pouvoir retarder l'acquisition ni
l'odomètre. Si l'affichage sature, le kilométrage reste juste.

## 6. Stratégie de test

- **Tests unitaires sur PC** pour toute la logique critique : calcul de vitesse, filtrage
  des parasites du capteur, intégration de la distance, sélection de copie FRAM,
  simplification et projection GPX. Développement en TDD sur cette couche.
- **Simulateur** avec rejeu de scénarios depuis des fichiers de données : montée en vitesse,
  coupure brutale du contact, glitch de capteur, perte de fix GPS, canal absent.
- **Validation sur banc** avant montage : entrées faisceau simulées, capteur Hall entraîné
  manuellement, vérification du kilométrage sur une distance connue.
- Chien de garde matériel et surveillance de fraîcheur des canaux en fonctionnement.

## 7. Découpage en phases

| # | Contenu | Livrable |
|---|---|---|
| 1 | HAL, simulateur, bus de canaux, harnais de test | socle testé, rien de visible |
| 2 | Providers KDX : Hall, entrées TOR, NTC, tension ; odo/trip et persistance FRAM | données réelles sur banc |
| 3 | UI : bibliothèque de widgets, 3 archétypes, 6 thèmes, navigation aux boutons | le compteur fonctionne |
| 4 | Portail WiFi, profil JSON, import/export | utilisable sur une autre moto |
| 5 | GPS, chargement et suivi de trace GPX | navigation |
| 6 | Intégration mécanique, étanchéité, thermique, essais moto | monté et roulant |

Chaque phase fait l'objet de son propre plan d'implémentation.

## 8. Structure du dépôt

```
firmware/     application ESP32 (HAL esp32, providers, main)
core/         C++ pur : bus de canaux, odo, navigation — partagé firmware/sim
ui/           LVGL : widgets, archétypes, thèmes — partagé firmware/sim
sim/          HAL de simulation, rejeu de scénarios, binaire PC
hardware/     schéma et routage KiCad, STL du boîtier et des supports
profiles/     profils moto contribués, un fichier JSON par modèle
tests/        tests unitaires du core
docs/         documentation d'installation et de contribution
```

## 9. Points ouverts

Ces points ne bloquent pas le démarrage. Chacun a une règle de décision ou une phase de
résolution associée.

| Point | Résolution |
|---|---|
| Diamètre intérieur utile de la cloche | À mesurer avant achat. ≥ 70 mm → 2.8" ; sinon 2.1" |
| Jauge d'essence | Aucune d'origine sur la KDX. Soit sonde à flotteur ajoutée, soit estimation d'autonomie au kilométrage. Le canal `FUEL` reste `absent` tant que non tranché — l'UI le gère nativement. À décider en phase 2 |
| Nature du capteur « OIL TEMP » d'origine | À identifier sur la moto (probablement une thermistance NTC). La courbe de linéarisation est de toute façon dans le profil. À décider en phase 2 |
| Régime moteur | Captation sur le fil de bougie avec limiteur et trigger de Schmitt. Facile sur un 2 temps. Optionnel, à traiter en phase 2 si le temps le permet |
| Licence | Typiquement GPLv3 ou MIT pour le firmware, CERN-OHL-S pour le matériel. À choisir avant la première publication |
