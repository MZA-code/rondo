# Rondo — Plan d'implémentation : acquisition et odométrie

> **Pour les agents :** SOUS-COMPÉTENCE REQUISE — utiliser `superpowers:subagent-driven-development`
> (recommandé) ou `superpowers:executing-plans` pour dérouler ce plan tâche par tâche.
> Les étapes utilisent la syntaxe case à cocher (`- [ ]`).

**Objectif :** transformer les signaux du faisceau en canaux exploitables — vitesse,
odomètre, trip, température, tension, entrées tout ou rien — et faire tourner l'ensemble
sur l'ESP32-S3.

**Architecture :** toute la logique de calcul (vitesse, distance, odométrie, courbes de
capteurs) est écrite en C++ pur et testée sur PC. La couche ESP32 se réduit à quatre pilotes
qui implémentent les interfaces de `hal/`. Aucun calcul ne vit dans le code spécifique à la
cible : ce qui est difficile à tester est aussi ce qu'il ne faut pas y mettre.

**Pile technique :** C++17, ESP-IDF 5.2+, FreeRTOS, CMake, Catch2 v3.

## Contraintes globales

- **Aucun calcul dans les pilotes ESP32.** Un pilote lit un registre et renvoie une valeur
  brute. Toute conversion en grandeur physique appartient à `core/`, donc au domaine testé.
- **L'odomètre est écrit en continu, tous les 10 mètres parcourus.** La FRAM supporte plus
  de 10¹² cycles ; il n'y a **ni détection de coupure d'alimentation, ni comparateur, ni
  interruption de sauvegarde d'urgence** (voir spec §4.4 et §5.6).
- **La distance est comptée en millimètres entiers**, jamais en flottant. Un accumulateur
  flottant dérive sur 100 000 km ; un `uint64_t` en millimètres couvre 1,8 × 10¹⁰ km.
- **Aucune broche n'est câblée en dur.** Tout mappage vient du profil moto. Le plan
  `rondo-hardware` fige le câblage physique, pas l'affectation logique.
- **L'interface ne doit jamais pouvoir retarder l'acquisition ni l'odomètre.** Priorités
  FreeRTOS : acquisition à 50 Hz au-dessus de LVGL à 30 fps.
- **C++17**, `-Wall -Wextra -Wpedantic -Werror` sur `core/`, désactivé sur ESP-IDF.
- Identifiants et messages de commit en **anglais**, commentaires et tests en **français**.
- **Prérequis :** `rondo-core` terminé (39 tests) et `rondo-ui` terminé (75 tests).
  `rondo-hardware` doit être terminé pour les tâches 6 à 8 uniquement.

## Structure des fichiers

| Fichier | Responsabilité |
|---|---|
| `core/curve.h` / `.cpp` | Interpolation linéaire par morceaux (courbes de capteurs) |
| `core/scale.h` | Mise à l'échelle affine d'une lecture ADC |
| `core/crc32.h` / `.cpp` | Somme de contrôle des enregistrements FRAM |
| `core/providers/analog_sensor_provider.*` | Température, tension, jauge |
| `core/speed_calculator.h` / `.cpp` | Vitesse et distance depuis le compteur d'impulsions |
| `core/odometer_service.h` / `.cpp` | Odomètre, trip, persistance FRAM |
| `core/providers/hall_speed_provider.*` | Publie vitesse, odomètre et trip sur le bus |
| `firmware/CMakeLists.txt` | Projet ESP-IDF |
| `firmware/main/esp32_clock.*` | `IClock` sur `esp_timer` |
| `firmware/main/esp32_digital_in.*` | `IDigitalIn` sur `driver/gpio.h` |
| `firmware/main/esp32_analog_in.*` | `IAnalogIn` sur `esp_adc/adc_oneshot.h` |
| `firmware/main/esp32_pulse_counter.*` | `IPulseCounter` sur interruption GPIO + `esp_timer` |
| `firmware/main/esp32_fram.*` | `IPersistentStore` sur `driver/i2c_master.h` |
| `firmware/main/app_main.cpp` | Assemblage, tâches FreeRTOS |
| `docs/bench-acquisition.md` | Procédure de validation sur banc |
| `tests/test_*.cpp` | Un fichier par unité |

---

## Tâche 1 : Courbes de capteurs

Une thermistance n'est pas linéaire, et une jauge d'essence varie d'un modèle de moto à
l'autre. Les deux se décrivent par une table de points, donnée dans le profil moto.

**Fichiers :**
- Créer : `core/curve.h`, `core/curve.cpp`, `tests/test_curve.cpp`
- Modifier : `core/CMakeLists.txt`, `tests/CMakeLists.txt`

**Produit :**
- `struct CurvePoint { float x; float y; }`
- `class Curve` — `Curve(const CurvePoint* points, size_t count)`, `float at(float x) const`,
  `bool valid() const`

- [ ] **Étape 1 : écrire les tests qui échouent**

`tests/test_curve.cpp` :

```cpp
#include <catch2/catch_test_macros.hpp>
#include <catch2/matchers/catch_matchers_floating_point.hpp>

#include "core/curve.h"

using namespace rondo;
using Catch::Matchers::WithinAbs;

namespace {

// Courbe NTC typique : resistance decroissante, donc lecture ADC
// croissante avec la temperature.
constexpr CurvePoint kNtc[] = {
    {500.0f, 20.0f}, {1200.0f, 60.0f}, {2000.0f, 90.0f}, {3000.0f, 120.0f}};

}  // namespace

TEST_CASE("une courbe rend la valeur exacte sur un point de la table") {
    const Curve c(kNtc, 4);
    REQUIRE_THAT(c.at(1200.0f), WithinAbs(60.0f, 0.001f));
}

TEST_CASE("une courbe interpole lineairement entre deux points") {
    const Curve c(kNtc, 4);
    // A mi-chemin entre 1200 et 2000, donc entre 60 et 90.
    REQUIRE_THAT(c.at(1600.0f), WithinAbs(75.0f, 0.001f));
}

TEST_CASE("une courbe est bornee en dehors de sa plage") {
    const Curve c(kNtc, 4);
    REQUIRE_THAT(c.at(0.0f), WithinAbs(20.0f, 0.001f));
    REQUIRE_THAT(c.at(9999.0f), WithinAbs(120.0f, 0.001f));
}

TEST_CASE("une courbe de moins de deux points est invalide") {
    constexpr CurvePoint one[] = {{0.0f, 0.0f}};
    REQUIRE_FALSE(Curve(one, 1).valid());
    REQUIRE_FALSE(Curve(nullptr, 0).valid());
}

TEST_CASE("une courbe dont les abscisses ne croissent pas est invalide") {
    constexpr CurvePoint unsorted[] = {{100.0f, 1.0f}, {50.0f, 2.0f}};
    REQUIRE_FALSE(Curve(unsorted, 2).valid());
}

TEST_CASE("une courbe invalide rend zero sans planter") {
    constexpr CurvePoint unsorted[] = {{100.0f, 1.0f}, {50.0f, 2.0f}};
    REQUIRE_THAT(Curve(unsorted, 2).at(75.0f), WithinAbs(0.0f, 0.001f));
}
```

- [ ] **Étape 2 : lancer les tests et vérifier qu'ils échouent**

```bash
cmake --build build -j
```

Attendu : **échec**, `core/curve.h: No such file or directory`.

- [ ] **Étape 3 : écrire l'implémentation**

`core/curve.h` :

```cpp
#pragma once

#include <cstddef>

namespace rondo {

struct CurvePoint {
    float x = 0.0f;  // lecture brute (compte ADC, ohms)
    float y = 0.0f;  // grandeur physique (degres, pourcentage)
};

// Interpolation lineaire par morceaux, bornee aux extremites.
//
// Ne possede pas ses points : la table vit dans le profil moto et doit
// survivre a la courbe. Aucune allocation, utilisable sur la cible.
class Curve {
 public:
    Curve(const CurvePoint* points, size_t count);

    // Renvoie 0 si la courbe est invalide. Borne en dehors de la plage :
    // une sonde debranchee doit donner une valeur extreme, pas une
    // extrapolation fantaisiste.
    float at(float x) const;

    // Une courbe valide a au moins deux points d'abscisses strictement
    // croissantes.
    bool valid() const { return valid_; }

 private:
    const CurvePoint* points_;
    size_t count_;
    bool valid_;
};

}  // namespace rondo
```

`core/curve.cpp` :

```cpp
#include "core/curve.h"

namespace rondo {
namespace {

bool isStrictlyIncreasing(const CurvePoint* points, size_t count) {
    for (size_t i = 1; i < count; ++i) {
        if (points[i].x <= points[i - 1].x) {
            return false;
        }
    }
    return true;
}

}  // namespace

Curve::Curve(const CurvePoint* points, size_t count)
    : points_(points),
      count_(count),
      valid_(points != nullptr && count >= 2 && isStrictlyIncreasing(points, count)) {}

float Curve::at(float x) const {
    if (!valid_) {
        return 0.0f;
    }
    if (x <= points_[0].x) {
        return points_[0].y;
    }
    if (x >= points_[count_ - 1].x) {
        return points_[count_ - 1].y;
    }
    for (size_t i = 1; i < count_; ++i) {
        if (x <= points_[i].x) {
            const CurvePoint& a = points_[i - 1];
            const CurvePoint& b = points_[i];
            const float t = (x - a.x) / (b.x - a.x);
            return a.y + t * (b.y - a.y);
        }
    }
    return points_[count_ - 1].y;
}

}  // namespace rondo
```

Ajouter `curve.cpp` aux sources de `rondo_core` et `test_curve.cpp` à `rondo_tests`.

- [ ] **Étape 4 : lancer les tests et vérifier qu'ils passent**

```bash
cmake -B build -DCMAKE_BUILD_TYPE=Debug && cmake --build build -j && ctest --test-dir build --output-on-failure
```

Attendu : `100% tests passed, 0 tests failed out of 81`.

- [ ] **Étape 5 : commit**

```bash
git add core tests
git commit -m "core: add piecewise linear sensor curves"
```

---

## Tâche 2 : Fournisseur analogique

**Fichiers :**
- Créer : `core/scale.h`, `core/providers/analog_sensor_provider.h` / `.cpp`,
  `tests/test_analog_sensor_provider.cpp`
- Modifier : `core/CMakeLists.txt`, `tests/CMakeLists.txt`

**Consomme :** `Curve` (tâche 1), `IAnalogIn`, `ChannelBus`.
**Produit :**
- `struct LinearScale { float scale; float offset; }`,
  `float applyScale(uint16_t raw, LinearScale s)`
- `struct AnalogSensorConfig { ChannelId channel; uint32_t ttl_ms; LinearScale scale; const CurvePoint* curve_points; size_t curve_count; }`
- `class AnalogSensorProvider` — `AnalogSensorProvider(const IAnalogIn&, AnalogSensorConfig)`

**Règle :** si `curve_points` est nul, la lecture passe par `applyScale` (cas de la tension
batterie, strictement linéaire). Sinon elle passe par la courbe (cas de la thermistance et
de la jauge). Les deux voies s'excluent.

- [ ] **Étape 1 : écrire les tests qui échouent**

`tests/test_analog_sensor_provider.cpp` :

```cpp
#include <catch2/catch_test_macros.hpp>
#include <catch2/matchers/catch_matchers_floating_point.hpp>

#include "core/channel_bus.h"
#include "core/providers/analog_sensor_provider.h"
#include "sim/sim_analog_in.h"
#include "sim/sim_clock.h"

using namespace rondo;
using Catch::Matchers::WithinAbs;

namespace {

constexpr CurvePoint kNtc[] = {{500.0f, 20.0f}, {2000.0f, 90.0f}};

// Diviseur 100k/15k, ADC 12 bits, pleine echelle 3,1 V :
// volts par point = (3,1 / 4095) / 0,13043 = 0,005804
constexpr LinearScale kVbatScale{0.005804f, 0.0f};

}  // namespace

TEST_CASE("l'echelle affine convertit une lecture brute") {
    REQUIRE_THAT(applyScale(2158, kVbatScale), WithinAbs(12.52f, 0.02f));
    REQUIRE_THAT(applyScale(0, kVbatScale), WithinAbs(0.0f, 0.001f));
}

TEST_CASE("le fournisseur analogique declare son canal") {
    SimClock clock;
    ChannelBus bus(clock);
    SimAnalogIn input;
    AnalogSensorProvider provider(
        input, {ChannelId::BatteryVolts, 1000, kVbatScale, nullptr, 0});

    REQUIRE(bus.read(ChannelId::BatteryVolts).validity == Validity::Absent);
    provider.declareChannels(bus);
    REQUIRE(bus.read(ChannelId::BatteryVolts).validity == Validity::Stale);
}

TEST_CASE("sans courbe, le fournisseur applique l'echelle affine") {
    SimClock clock;
    ChannelBus bus(clock);
    SimAnalogIn input;
    AnalogSensorProvider provider(
        input, {ChannelId::BatteryVolts, 1000, kVbatScale, nullptr, 0});
    provider.declareChannels(bus);

    input.set(2158);
    provider.poll(bus);
    REQUIRE_THAT(bus.read(ChannelId::BatteryVolts).value, WithinAbs(12.52f, 0.02f));
}

TEST_CASE("avec une courbe, le fournisseur interpole") {
    SimClock clock;
    ChannelBus bus(clock);
    SimAnalogIn input;
    AnalogSensorProvider provider(input, {ChannelId::EngineTemp, 1000, {1.0f, 0.0f}, kNtc, 2});
    provider.declareChannels(bus);

    input.set(1250);  // a mi-chemin de 500 et 2000
    provider.poll(bus);
    REQUIRE_THAT(bus.read(ChannelId::EngineTemp).value, WithinAbs(55.0f, 0.1f));
}

TEST_CASE("la valeur publiee est fraiche puis se perime") {
    SimClock clock;
    ChannelBus bus(clock);
    SimAnalogIn input;
    AnalogSensorProvider provider(
        input, {ChannelId::BatteryVolts, 1000, kVbatScale, nullptr, 0});
    provider.declareChannels(bus);
    provider.poll(bus);

    REQUIRE(bus.read(ChannelId::BatteryVolts).validity == Validity::Fresh);
    clock.advanceMs(1001);
    REQUIRE(bus.read(ChannelId::BatteryVolts).validity == Validity::Stale);
}

TEST_CASE("une courbe invalide ne publie rien plutot que n'importe quoi") {
    SimClock clock;
    ChannelBus bus(clock);
    SimAnalogIn input;
    constexpr CurvePoint bad[] = {{100.0f, 1.0f}, {50.0f, 2.0f}};
    AnalogSensorProvider provider(input, {ChannelId::EngineTemp, 1000, {1.0f, 0.0f}, bad, 2});
    provider.declareChannels(bus);

    input.set(75);
    provider.poll(bus);
    REQUIRE(bus.read(ChannelId::EngineTemp).validity == Validity::Stale);
}
```

- [ ] **Étape 2 : lancer les tests et vérifier qu'ils échouent**

```bash
cmake --build build -j
```

Attendu : **échec**, `core/providers/analog_sensor_provider.h: No such file or directory`.

- [ ] **Étape 3 : écrire l'implémentation**

`core/scale.h` :

```cpp
#pragma once

#include <cstdint>

namespace rondo {

// Conversion affine d'une lecture ADC brute en grandeur physique.
// Les deux coefficients viennent du profil moto : ils dependent du pont
// diviseur et de la reference de tension, donc du cablage.
struct LinearScale {
    float scale = 1.0f;
    float offset = 0.0f;
};

inline float applyScale(uint16_t raw, LinearScale s) {
    return static_cast<float>(raw) * s.scale + s.offset;
}

}  // namespace rondo
```

`core/providers/analog_sensor_provider.h` :

```cpp
#pragma once

#include <cstddef>
#include <cstdint>

#include "core/channel_id.h"
#include "core/curve.h"
#include "core/provider.h"
#include "core/scale.h"
#include "hal/i_analog_in.h"

namespace rondo {

class ChannelBus;

struct AnalogSensorConfig {
    ChannelId channel;
    uint32_t ttl_ms;
    LinearScale scale;
    // Si nul, la conversion se fait par `scale`. Sinon par la courbe.
    const CurvePoint* curve_points = nullptr;
    size_t curve_count = 0;
};

// Publie une entree analogique sur un canal, apres conversion.
// Sert la tension batterie (affine), la temperature et la jauge d'essence
// (par courbe).
class AnalogSensorProvider final : public IProvider {
 public:
    AnalogSensorProvider(const IAnalogIn& input, AnalogSensorConfig config);

    void declareChannels(ChannelBus& bus) override;
    void poll(ChannelBus& bus) override;

 private:
    const IAnalogIn& input_;
    AnalogSensorConfig config_;
    Curve curve_;
};

}  // namespace rondo
```

`core/providers/analog_sensor_provider.cpp` :

```cpp
#include "core/providers/analog_sensor_provider.h"

#include "core/channel_bus.h"

namespace rondo {

AnalogSensorProvider::AnalogSensorProvider(const IAnalogIn& input, AnalogSensorConfig config)
    : input_(input), config_(config), curve_(config.curve_points, config.curve_count) {}

void AnalogSensorProvider::declareChannels(ChannelBus& bus) {
    bus.declare(config_.channel, config_.ttl_ms);
}

void AnalogSensorProvider::poll(ChannelBus& bus) {
    const uint16_t raw = input_.readRaw();

    if (config_.curve_points == nullptr) {
        bus.publish(config_.channel, applyScale(raw, config_.scale));
        return;
    }

    // Une courbe mal saisie dans le profil ne doit pas produire une valeur
    // credible mais fausse : mieux vaut laisser le canal se perimer, ce que
    // l'interface signalera.
    if (!curve_.valid()) {
        return;
    }
    bus.publish(config_.channel, curve_.at(static_cast<float>(raw)));
}

}  // namespace rondo
```

Ajouter `providers/analog_sensor_provider.cpp` aux sources de `rondo_core` et le fichier de
test à `rondo_tests`.

- [ ] **Étape 4 : lancer les tests et vérifier qu'ils passent**

```bash
cmake -B build -DCMAKE_BUILD_TYPE=Debug && cmake --build build -j && ctest --test-dir build --output-on-failure
```

Attendu : `100% tests passed, 0 tests failed out of 87`.

- [ ] **Étape 5 : commit**

```bash
git add core tests
git commit -m "core: add the analog sensor provider with scale and curve paths"
```

---

## Tâche 3 : Calcul de vitesse et de distance

Le cœur du problème posé au départ : reconstruire vitesse et kilométrage sans entraînement
mécanique.

**Fichiers :**
- Créer : `core/speed_calculator.h`, `core/speed_calculator.cpp`,
  `tests/test_speed_calculator.cpp`
- Modifier : `core/CMakeLists.txt`, `tests/CMakeLists.txt`

**Consomme :** `IPulseCounter`.
**Produit :**
- `struct SpeedConfig { uint32_t wheel_circumference_mm; uint8_t magnets_per_revolution; uint32_t zero_speed_timeout_ms; float max_plausible_speed_kmh; }`
- `class SpeedCalculator` — `explicit SpeedCalculator(SpeedConfig)`,
  `void update(uint32_t pulse_count, uint32_t last_period_us, uint32_t now_ms)`,
  `float speedKmh() const`, `uint32_t takeDistanceMm()`

**Formules, à respecter exactement :**

```
distance par impulsion = circonference / aimants           [mm]
vitesse [km/h] = 3600 x circonference / (aimants x periode_us)

Verification : circonference 2170 mm, 2 aimants, 68 km/h
  -> 3600 x 2170 / (2 x 57441) = 68,0 km/h
```

**Comptage exact de la distance.** La division `circonference / aimants` n'est pas
toujours entière (2170 / 3 = 723,33). Accumuler des millimètres arrondis ferait dériver
l'odomètre de plusieurs kilomètres sur la vie de la moto. On accumule donc le **numérateur**
et on ne divise qu'au moment de restituer :

```
reste += delta_impulsions x circonference
distance_mm = reste / aimants
reste       = reste % aimants
```

- [ ] **Étape 1 : écrire les tests qui échouent**

`tests/test_speed_calculator.cpp` :

```cpp
#include <catch2/catch_test_macros.hpp>
#include <catch2/matchers/catch_matchers_floating_point.hpp>

#include "core/speed_calculator.h"

using namespace rondo;
using Catch::Matchers::WithinAbs;

namespace {

constexpr SpeedConfig kKdx{2170, 2, 1500, 180.0f};

}  // namespace

TEST_CASE("la vitesse est nulle au demarrage") {
    SpeedCalculator calc(kKdx);
    calc.update(0, 0, 0);
    REQUIRE_THAT(calc.speedKmh(), WithinAbs(0.0f, 0.001f));
}

TEST_CASE("une periode stable donne la vitesse attendue") {
    SpeedCalculator calc(kKdx);
    calc.update(0, 0, 0);
    calc.update(1, 57441, 100);
    REQUIRE_THAT(calc.speedKmh(), WithinAbs(68.0f, 0.1f));
}

TEST_CASE("la distance vaut exactement une circonference par tour de roue") {
    SpeedCalculator calc(kKdx);
    calc.update(0, 0, 0);
    calc.update(2, 57441, 100);  // 2 aimants = 1 tour
    REQUIRE(calc.takeDistanceMm() == 2170u);
}

TEST_CASE("le comptage reste exact avec un nombre d'aimants non diviseur") {
    // 2170 / 3 ne tombe pas juste : trois impulsions doivent malgre tout
    // donner exactement une circonference, sans perte.
    SpeedCalculator calc({2170, 3, 1500, 180.0f});
    calc.update(0, 0, 0);
    calc.update(1, 60000, 100);
    calc.update(2, 60000, 200);
    calc.update(3, 60000, 300);
    REQUIRE(calc.takeDistanceMm() == 2170u);
}

TEST_CASE("le comptage ne derive pas sur un grand nombre de tours") {
    SpeedCalculator calc({2170, 3, 1500, 180.0f});
    calc.update(0, 0, 0);
    uint64_t total = 0;
    for (uint32_t i = 1; i <= 3000; ++i) {  // 1000 tours de roue
        calc.update(i, 60000, i * 60);
        total += calc.takeDistanceMm();
    }
    REQUIRE(total == 2170u * 1000u);
}

TEST_CASE("prendre la distance remet le compteur a zero") {
    SpeedCalculator calc(kKdx);
    calc.update(0, 0, 0);
    calc.update(2, 57441, 100);
    REQUIRE(calc.takeDistanceMm() == 2170u);
    REQUIRE(calc.takeDistanceMm() == 0u);
}

TEST_CASE("une periode invraisemblable est rejetee, vitesse et distance comprises") {
    SpeedCalculator calc(kKdx);
    calc.update(0, 0, 0);
    calc.update(1, 57441, 100);
    const float before = calc.speedKmh();
    calc.takeDistanceMm();

    // 3600 x 2170 / (2 x 1000) = 3906 km/h : parasite d'allumage.
    calc.update(2, 1000, 200);
    REQUIRE_THAT(calc.speedKmh(), WithinAbs(before, 0.001f));
    REQUIRE(calc.takeDistanceMm() == 0u);
}

TEST_CASE("la vitesse retombe a zero apres le delai sans impulsion") {
    SpeedCalculator calc(kKdx);
    calc.update(0, 0, 0);
    calc.update(1, 57441, 100);
    REQUIRE(calc.speedKmh() > 0.0f);

    calc.update(1, 57441, 1600);  // meme compte, 1500 ms plus tard
    REQUIRE_THAT(calc.speedKmh(), WithinAbs(0.0f, 0.001f));
}

TEST_CASE("le rebouclage du compteur materiel ne perd pas de distance") {
    SpeedCalculator calc(kKdx);
    calc.update(0xFFFFFFFEu, 0, 0);
    calc.update(0x00000000u, 57441, 100);  // 2 impulsions a cheval sur le rebouclage
    REQUIRE(calc.takeDistanceMm() == 2170u);
}
```

- [ ] **Étape 2 : lancer les tests et vérifier qu'ils échouent**

```bash
cmake --build build -j
```

Attendu : **échec**, `core/speed_calculator.h: No such file or directory`.

- [ ] **Étape 3 : écrire l'implémentation**

`core/speed_calculator.h` :

```cpp
#pragma once

#include <cstdint>

namespace rondo {

struct SpeedConfig {
    uint32_t wheel_circumference_mm = 2170;  // roue de 21 pouces typique
    uint8_t magnets_per_revolution = 2;
    uint32_t zero_speed_timeout_ms = 1500;
    // Au-dela, l'impulsion est consideree comme un parasite d'allumage.
    float max_plausible_speed_kmh = 180.0f;
};

// Convertit les impulsions du capteur a effet Hall en vitesse et en
// distance parcourue.
//
// La vitesse vient de la PERIODE entre deux impulsions, pas du comptage :
// a 8 km/h en enduro, compter sur une fenetre donnerait un affichage
// hache. La distance vient du COMPTAGE, seul moyen de ne rien perdre.
class SpeedCalculator {
 public:
    explicit SpeedCalculator(SpeedConfig config);

    // A appeler a chaque scrutation avec l'etat du compteur materiel.
    void update(uint32_t pulse_count, uint32_t last_period_us, uint32_t now_ms);

    float speedKmh() const { return speed_kmh_; }

    // Distance accumulee depuis le dernier appel, remise a zero.
    // L'appelant DOIT la consommer, sinon elle est comptee deux fois.
    uint32_t takeDistanceMm();

 private:
    bool isPlausible(uint32_t period_us) const;

    SpeedConfig config_;
    bool primed_ = false;
    uint32_t last_count_ = 0;
    uint32_t last_pulse_ms_ = 0;
    float speed_kmh_ = 0.0f;

    // Numerateur en attente de division : garantit un comptage exact
    // meme quand la circonference n'est pas divisible par le nombre
    // d'aimants.
    uint64_t remainder_ = 0;
    uint32_t pending_mm_ = 0;
};

}  // namespace rondo
```

`core/speed_calculator.cpp` :

```cpp
#include "core/speed_calculator.h"

namespace rondo {

SpeedCalculator::SpeedCalculator(SpeedConfig config) : config_(config) {
    if (config_.magnets_per_revolution == 0) {
        config_.magnets_per_revolution = 1;
    }
}

bool SpeedCalculator::isPlausible(uint32_t period_us) const {
    if (period_us == 0) {
        return false;
    }
    const float kmh = 3600.0f * static_cast<float>(config_.wheel_circumference_mm) /
                      (static_cast<float>(config_.magnets_per_revolution) *
                       static_cast<float>(period_us));
    return kmh <= config_.max_plausible_speed_kmh;
}

void SpeedCalculator::update(uint32_t pulse_count, uint32_t last_period_us, uint32_t now_ms) {
    if (!primed_) {
        primed_ = true;
        last_count_ = pulse_count;
        last_pulse_ms_ = now_ms;
        return;
    }

    // Soustraction non signee : le rebouclage du compteur materiel a 2^32
    // donne malgre tout le bon nombre d'impulsions.
    const uint32_t delta = pulse_count - last_count_;
    last_count_ = pulse_count;

    if (delta > 0) {
        if (!isPlausible(last_period_us)) {
            // Parasite : on ne compte ni la distance ni la vitesse, et on
            // ne rearme pas le delai d'arret. Le filtre RC du schema
            // elimine le gros du bruit, ceci attrape le reste.
            return;
        }
        remainder_ += static_cast<uint64_t>(delta) * config_.wheel_circumference_mm;
        const uint64_t magnets = config_.magnets_per_revolution;
        pending_mm_ += static_cast<uint32_t>(remainder_ / magnets);
        remainder_ %= magnets;

        speed_kmh_ = 3600.0f * static_cast<float>(config_.wheel_circumference_mm) /
                     (static_cast<float>(config_.magnets_per_revolution) *
                      static_cast<float>(last_period_us));
        last_pulse_ms_ = now_ms;
        return;
    }

    if (now_ms - last_pulse_ms_ >= config_.zero_speed_timeout_ms) {
        speed_kmh_ = 0.0f;
    }
}

uint32_t SpeedCalculator::takeDistanceMm() {
    const uint32_t mm = pending_mm_;
    pending_mm_ = 0;
    return mm;
}

}  // namespace rondo
```

Ajouter `speed_calculator.cpp` aux sources de `rondo_core` et le fichier de test à
`rondo_tests`.

- [ ] **Étape 4 : lancer les tests et vérifier qu'ils passent**

```bash
cmake -B build -DCMAKE_BUILD_TYPE=Debug && cmake --build build -j && ctest --test-dir build --output-on-failure
```

Attendu : `100% tests passed, 0 tests failed out of 96`.

- [ ] **Étape 5 : commit**

```bash
git add core tests
git commit -m "core: compute speed from pulse period and exact distance from count"
```

---

## Tâche 4 : Odomètre et persistance FRAM

**Fichiers :**
- Créer : `core/crc32.h`, `core/crc32.cpp`, `core/odometer_service.h`,
  `core/odometer_service.cpp`, `tests/test_crc32.cpp`, `tests/test_odometer_service.cpp`
- Modifier : `core/CMakeLists.txt`, `tests/CMakeLists.txt`

**Consomme :** `IPersistentStore`.
**Produit :**
- `uint32_t crc32(const uint8_t* data, size_t len)`
- `class OdometerService` — `OdometerService(IPersistentStore&, uint32_t write_interval_mm)`,
  `bool load()`, `void addDistance(uint32_t mm)`, `uint64_t odometerMm() const`,
  `uint64_t tripMm() const`, `void resetTrip()`, `void setOdometerMm(uint64_t)`,
  `void flush()`

**Format d'enregistrement**, 20 octets, écrit en **deux exemplaires** aux adresses 0 et 32 :

```
octets  0-7   odo_mm   (uint64, petit-boutiste)
octets  8-15  trip_mm  (uint64, petit-boutiste)
octets 16-19  crc32    (des 16 octets precedents)
```

- [ ] **Étape 1 : écrire les tests de somme de contrôle qui échouent**

`tests/test_crc32.cpp` :

```cpp
#include <catch2/catch_test_macros.hpp>

#include <cstring>

#include "core/crc32.h"

using namespace rondo;

TEST_CASE("la somme de controle correspond au vecteur de reference") {
    const char* data = "123456789";
    REQUIRE(crc32(reinterpret_cast<const uint8_t*>(data), 9) == 0xCBF43926u);
}

TEST_CASE("un octet different change la somme de controle") {
    const uint8_t a[4] = {1, 2, 3, 4};
    const uint8_t b[4] = {1, 2, 3, 5};
    REQUIRE(crc32(a, 4) != crc32(b, 4));
}

TEST_CASE("une entree vide a une somme de controle definie") {
    REQUIRE(crc32(nullptr, 0) == 0u);
}
```

- [ ] **Étape 2 : écrire la somme de contrôle**

`core/crc32.h` :

```cpp
#pragma once

#include <cstddef>
#include <cstdint>

namespace rondo {

// CRC-32 (polynome IEEE 802.3, reflechi 0xEDB88320).
// Protege les enregistrements FRAM contre une ecriture interrompue.
uint32_t crc32(const uint8_t* data, size_t len);

}  // namespace rondo
```

`core/crc32.cpp` :

```cpp
#include "core/crc32.h"

namespace rondo {

uint32_t crc32(const uint8_t* data, size_t len) {
    if (data == nullptr || len == 0) {
        return 0u;
    }
    // Calcul bit a bit : pas de table de 1 Kio en flash, et le debit est
    // sans objet pour 16 octets toutes les quelques centaines de
    // millisecondes.
    uint32_t crc = 0xFFFFFFFFu;
    for (size_t i = 0; i < len; ++i) {
        crc ^= data[i];
        for (int bit = 0; bit < 8; ++bit) {
            crc = (crc >> 1) ^ (0xEDB88320u & (~(crc & 1u) + 1u));
        }
    }
    return ~crc;
}

}  // namespace rondo
```

- [ ] **Étape 3 : écrire les tests d'odométrie qui échouent**

`tests/test_odometer_service.cpp` :

```cpp
#include <catch2/catch_test_macros.hpp>

#include "core/odometer_service.h"
#include "sim/sim_store.h"

using namespace rondo;

namespace {

constexpr uint32_t kTenMetresMm = 10000;
constexpr size_t kStoreSize = 128;

}  // namespace

TEST_CASE("une memoire vierge ne fournit aucun enregistrement valide") {
    SimStore store(kStoreSize);
    OdometerService odo(store, kTenMetresMm);
    REQUIRE_FALSE(odo.load());
    REQUIRE(odo.odometerMm() == 0u);
    REQUIRE(odo.tripMm() == 0u);
}

TEST_CASE("une distance inferieure au seuil n'ecrit pas") {
    SimStore store(kStoreSize);
    OdometerService odo(store, kTenMetresMm);
    odo.load();
    const uint32_t before = store.writeCount();

    odo.addDistance(9999);
    REQUIRE(odo.odometerMm() == 9999u);
    REQUIRE(store.writeCount() == before);
}

TEST_CASE("franchir le seuil declenche une ecriture des deux copies") {
    SimStore store(kStoreSize);
    OdometerService odo(store, kTenMetresMm);
    odo.load();
    const uint32_t before = store.writeCount();

    odo.addDistance(kTenMetresMm);
    REQUIRE(store.writeCount() == before + 2);
}

TEST_CASE("la valeur survit a un redemarrage") {
    SimStore store(kStoreSize);
    {
        OdometerService odo(store, kTenMetresMm);
        odo.load();
        odo.addDistance(123456);
        odo.flush();
    }
    OdometerService reloaded(store, kTenMetresMm);
    REQUIRE(reloaded.load());
    REQUIRE(reloaded.odometerMm() == 123456u);
}

TEST_CASE("une premiere copie corrompue laisse la seconde prendre le relais") {
    SimStore store(kStoreSize);
    {
        OdometerService odo(store, kTenMetresMm);
        odo.load();
        odo.addDistance(123456);
        odo.flush();
    }
    const uint8_t garbage[20] = {};
    store.write(0, garbage, 20);

    OdometerService reloaded(store, kTenMetresMm);
    REQUIRE(reloaded.load());
    REQUIRE(reloaded.odometerMm() == 123456u);
}

TEST_CASE("deux copies corrompues repartent de zero sans planter") {
    SimStore store(kStoreSize);
    const uint8_t garbage[20] = {9, 9, 9, 9, 9, 9, 9, 9, 9, 9,
                                 9, 9, 9, 9, 9, 9, 9, 9, 9, 9};
    store.write(0, garbage, 20);
    store.write(32, garbage, 20);

    OdometerService odo(store, kTenMetresMm);
    REQUIRE_FALSE(odo.load());
    REQUIRE(odo.odometerMm() == 0u);
}

TEST_CASE("la remise a zero du trip laisse l'odometre intact") {
    SimStore store(kStoreSize);
    OdometerService odo(store, kTenMetresMm);
    odo.load();
    odo.addDistance(50000);
    REQUIRE(odo.tripMm() == 50000u);

    odo.resetTrip();
    REQUIRE(odo.tripMm() == 0u);
    REQUIRE(odo.odometerMm() == 50000u);
}

TEST_CASE("l'offset de mise en service est ecrit immediatement") {
    SimStore store(kStoreSize);
    OdometerService odo(store, kTenMetresMm);
    odo.load();
    const uint32_t before = store.writeCount();

    odo.setOdometerMm(2566000000ull);  // 2566 km
    REQUIRE(odo.odometerMm() == 2566000000ull);
    REQUIRE(store.writeCount() == before + 2);
}

TEST_CASE("la cadence d'ecriture reste tres loin de l'endurance de la FRAM") {
    // 100 000 km a une ecriture tous les 10 m, deux copies : 2 x 10^7.
    // Une FRAM tient plus de 10^12 cycles, soit quatre ordres de grandeur
    // de marge. C'est ce qui autorise a se passer de detection de coupure.
    SimStore store(kStoreSize);
    OdometerService odo(store, kTenMetresMm);
    odo.load();

    for (uint32_t i = 0; i < 1000; ++i) {
        odo.addDistance(kTenMetresMm);  // 10 km parcourus
    }
    REQUIRE(store.writeCount() == 2000u);
    REQUIRE(odo.odometerMm() == 10000000ull);
}
```

- [ ] **Étape 4 : lancer les tests et vérifier qu'ils échouent**

```bash
cmake --build build -j
```

Attendu : **échec**, `core/odometer_service.h: No such file or directory`.

- [ ] **Étape 5 : écrire l'odométrie**

`core/odometer_service.h` :

```cpp
#pragma once

#include <cstdint>

#include "hal/i_persistent_store.h"

namespace rondo {

// Odometre et trip, en millimetres entiers et persistes en FRAM.
//
// L'ecriture est CONTINUE, tous les `write_interval_mm`. Il n'y a
// volontairement aucune detection de coupure d'alimentation : la valeur
// en memoire est toujours a jour a l'intervalle pres, donc il n'y a rien
// a sauver quand le contact est coupe. C'est la raison d'etre du choix
// d'une FRAM plutot que d'une EEPROM.
class OdometerService {
 public:
    static constexpr uint16_t kPrimaryAddr = 0;
    static constexpr uint16_t kBackupAddr = 32;
    static constexpr size_t kRecordSize = 20;

    OdometerService(IPersistentStore& store, uint32_t write_interval_mm);

    // Relit les deux copies et retient la premiere valide.
    // Renvoie false si aucune ne l'est : on repart alors de zero.
    bool load();

    void addDistance(uint32_t mm);

    uint64_t odometerMm() const { return odo_mm_; }
    uint64_t tripMm() const { return trip_mm_; }

    void resetTrip();

    // Offset de mise en service, pour repartir du kilometrage reel.
    void setOdometerMm(uint64_t mm);

    // Ecriture immediate des deux copies.
    void flush();

 private:
    bool readRecord(uint16_t addr, uint64_t& odo, uint64_t& trip) const;
    void writeRecord(uint16_t addr) const;

    IPersistentStore& store_;
    uint32_t write_interval_mm_;
    uint64_t odo_mm_ = 0;
    uint64_t trip_mm_ = 0;
    uint64_t last_written_odo_mm_ = 0;
};

}  // namespace rondo
```

`core/odometer_service.cpp` :

```cpp
#include "core/odometer_service.h"

#include <cstring>

#include "core/crc32.h"

namespace rondo {
namespace {

void put64(uint8_t* p, uint64_t v) {
    for (int i = 0; i < 8; ++i) {
        p[i] = static_cast<uint8_t>((v >> (8 * i)) & 0xFFu);
    }
}

uint64_t get64(const uint8_t* p) {
    uint64_t v = 0;
    for (int i = 0; i < 8; ++i) {
        v |= static_cast<uint64_t>(p[i]) << (8 * i);
    }
    return v;
}

void put32(uint8_t* p, uint32_t v) {
    for (int i = 0; i < 4; ++i) {
        p[i] = static_cast<uint8_t>((v >> (8 * i)) & 0xFFu);
    }
}

uint32_t get32(const uint8_t* p) {
    uint32_t v = 0;
    for (int i = 0; i < 4; ++i) {
        v |= static_cast<uint32_t>(p[i]) << (8 * i);
    }
    return v;
}

}  // namespace

OdometerService::OdometerService(IPersistentStore& store, uint32_t write_interval_mm)
    : store_(store), write_interval_mm_(write_interval_mm > 0 ? write_interval_mm : 1) {}

bool OdometerService::readRecord(uint16_t addr, uint64_t& odo, uint64_t& trip) const {
    uint8_t buf[kRecordSize] = {};
    if (!store_.read(addr, buf, kRecordSize)) {
        return false;
    }
    if (get32(buf + 16) != crc32(buf, 16)) {
        return false;
    }
    odo = get64(buf);
    trip = get64(buf + 8);
    return true;
}

void OdometerService::writeRecord(uint16_t addr) const {
    uint8_t buf[kRecordSize] = {};
    put64(buf, odo_mm_);
    put64(buf + 8, trip_mm_);
    put32(buf + 16, crc32(buf, 16));
    store_.write(addr, buf, kRecordSize);
}

bool OdometerService::load() {
    uint64_t odo = 0;
    uint64_t trip = 0;
    if (readRecord(kPrimaryAddr, odo, trip) || readRecord(kBackupAddr, odo, trip)) {
        odo_mm_ = odo;
        trip_mm_ = trip;
        last_written_odo_mm_ = odo;
        return true;
    }
    odo_mm_ = 0;
    trip_mm_ = 0;
    last_written_odo_mm_ = 0;
    return false;
}

void OdometerService::addDistance(uint32_t mm) {
    odo_mm_ += mm;
    trip_mm_ += mm;
    if (odo_mm_ - last_written_odo_mm_ >= write_interval_mm_) {
        flush();
    }
}

void OdometerService::resetTrip() {
    trip_mm_ = 0;
    flush();
}

void OdometerService::setOdometerMm(uint64_t mm) {
    odo_mm_ = mm;
    flush();
}

void OdometerService::flush() {
    writeRecord(kPrimaryAddr);
    writeRecord(kBackupAddr);
    last_written_odo_mm_ = odo_mm_;
}

}  // namespace rondo
```

Ajouter `crc32.cpp` et `odometer_service.cpp` aux sources de `rondo_core`, et les deux
fichiers de test à `rondo_tests`.

- [ ] **Étape 6 : lancer les tests et vérifier qu'ils passent**

```bash
cmake -B build -DCMAKE_BUILD_TYPE=Debug && cmake --build build -j && ctest --test-dir build --output-on-failure
```

Attendu : `100% tests passed, 0 tests failed out of 108`.

- [ ] **Étape 7 : commit**

```bash
git add core tests
git commit -m "core: add the odometer service with dual CRC-protected FRAM records"
```

---

## Tâche 5 : Fournisseur de vitesse

Relie le calculateur et l'odomètre au bus de canaux.

**Fichiers :**
- Créer : `core/providers/hall_speed_provider.h` / `.cpp`,
  `tests/test_hall_speed_provider.cpp`
- Modifier : `core/CMakeLists.txt`, `tests/CMakeLists.txt`

**Consomme :** `SpeedCalculator` (tâche 3), `OdometerService` (tâche 4), `IPulseCounter`,
`IClock`.
**Produit :** `class HallSpeedProvider` —
`HallSpeedProvider(const IPulseCounter&, const IClock&, OdometerService&, SpeedConfig, uint32_t ttl_ms)`

Publie `Speed` en km/h, `Odometer` et `Trip` en **kilomètres** (conversion depuis les
millimètres au moment de publier, jamais avant).

- [ ] **Étape 1 : écrire les tests qui échouent**

`tests/test_hall_speed_provider.cpp` :

```cpp
#include <catch2/catch_test_macros.hpp>
#include <catch2/matchers/catch_matchers_floating_point.hpp>

#include "core/channel_bus.h"
#include "core/odometer_service.h"
#include "core/providers/hall_speed_provider.h"
#include "sim/sim_clock.h"
#include "sim/sim_pulse_counter.h"
#include "sim/sim_store.h"

using namespace rondo;
using Catch::Matchers::WithinAbs;

namespace {

constexpr SpeedConfig kKdx{2170, 2, 1500, 180.0f};

}  // namespace

TEST_CASE("le fournisseur declare vitesse, odometre et trip") {
    SimClock clock;
    ChannelBus bus(clock);
    SimPulseCounter counter;
    SimStore store(128);
    OdometerService odo(store, 10000);
    HallSpeedProvider provider(counter, clock, odo, kKdx, 1000);

    provider.declareChannels(bus);
    REQUIRE(bus.read(ChannelId::Speed).validity == Validity::Stale);
    REQUIRE(bus.read(ChannelId::Odometer).validity == Validity::Stale);
    REQUIRE(bus.read(ChannelId::Trip).validity == Validity::Stale);
}

TEST_CASE("le fournisseur publie une vitesse coherente") {
    SimClock clock;
    ChannelBus bus(clock);
    SimPulseCounter counter;
    SimStore store(128);
    OdometerService odo(store, 10000);
    HallSpeedProvider provider(counter, clock, odo, kKdx, 1000);
    provider.declareChannels(bus);
    provider.poll(bus);

    // 17,41 Hz : 2 aimants, roue de 2,17 m, soit 68 km/h.
    clock.advanceMs(1000);
    counter.advance(1000, 17.41f);
    provider.poll(bus);

    REQUIRE_THAT(bus.read(ChannelId::Speed).value, WithinAbs(68.0f, 1.0f));
}

TEST_CASE("l'odometre s'exprime en kilometres sur le bus") {
    SimClock clock;
    ChannelBus bus(clock);
    SimPulseCounter counter;
    SimStore store(128);
    OdometerService odo(store, 10000);
    odo.load();
    odo.setOdometerMm(2566000000ull);  // 2566 km
    HallSpeedProvider provider(counter, clock, odo, kKdx, 1000);
    provider.declareChannels(bus);
    provider.poll(bus);

    REQUIRE_THAT(bus.read(ChannelId::Odometer).value, WithinAbs(2566.0f, 0.01f));
}

TEST_CASE("la distance parcourue alimente l'odometre") {
    SimClock clock;
    ChannelBus bus(clock);
    SimPulseCounter counter;
    SimStore store(128);
    OdometerService odo(store, 10000);
    odo.load();
    HallSpeedProvider provider(counter, clock, odo, kKdx, 1000);
    provider.declareChannels(bus);
    provider.poll(bus);

    // 100 tours de roue = 100 x 2170 mm = 217 m
    clock.advanceMs(10000);
    counter.advance(10000, 20.0f);
    provider.poll(bus);

    REQUIRE(odo.odometerMm() == 217000u);
}

TEST_CASE("le fournisseur ne consomme la distance qu'une seule fois") {
    SimClock clock;
    ChannelBus bus(clock);
    SimPulseCounter counter;
    SimStore store(128);
    OdometerService odo(store, 10000);
    odo.load();
    HallSpeedProvider provider(counter, clock, odo, kKdx, 1000);
    provider.declareChannels(bus);
    provider.poll(bus);

    clock.advanceMs(10000);
    counter.advance(10000, 20.0f);
    provider.poll(bus);
    const uint64_t after_first = odo.odometerMm();

    clock.advanceMs(100);
    provider.poll(bus);
    REQUIRE(odo.odometerMm() == after_first);
}
```

- [ ] **Étape 2 : lancer les tests et vérifier qu'ils échouent**

```bash
cmake --build build -j
```

Attendu : **échec**, `core/providers/hall_speed_provider.h: No such file or directory`.

- [ ] **Étape 3 : écrire l'implémentation**

`core/providers/hall_speed_provider.h` :

```cpp
#pragma once

#include <cstdint>

#include "core/clock.h"
#include "core/odometer_service.h"
#include "core/provider.h"
#include "core/speed_calculator.h"
#include "hal/i_pulse_counter.h"

namespace rondo {

class ChannelBus;

// Publie vitesse, odometre et trip a partir du capteur a effet Hall.
// Seul point du systeme qui consomme la distance calculee : elle serait
// comptee deux fois si un autre appelant prenait aussi takeDistanceMm().
class HallSpeedProvider final : public IProvider {
 public:
    HallSpeedProvider(const IPulseCounter& counter, const IClock& clock,
                      OdometerService& odometer, SpeedConfig config, uint32_t ttl_ms);

    void declareChannels(ChannelBus& bus) override;
    void poll(ChannelBus& bus) override;

 private:
    const IPulseCounter& counter_;
    const IClock& clock_;
    OdometerService& odometer_;
    SpeedCalculator calculator_;
    uint32_t ttl_ms_;
};

}  // namespace rondo
```

`core/providers/hall_speed_provider.cpp` :

```cpp
#include "core/providers/hall_speed_provider.h"

#include "core/channel_bus.h"

namespace rondo {
namespace {

constexpr float kMmPerKm = 1000000.0f;

}  // namespace

HallSpeedProvider::HallSpeedProvider(const IPulseCounter& counter, const IClock& clock,
                                     OdometerService& odometer, SpeedConfig config,
                                     uint32_t ttl_ms)
    : counter_(counter), clock_(clock), odometer_(odometer), calculator_(config),
      ttl_ms_(ttl_ms) {}

void HallSpeedProvider::declareChannels(ChannelBus& bus) {
    bus.declare(ChannelId::Speed, ttl_ms_);
    bus.declare(ChannelId::Odometer, ttl_ms_);
    bus.declare(ChannelId::Trip, ttl_ms_);
}

void HallSpeedProvider::poll(ChannelBus& bus) {
    calculator_.update(counter_.count(), counter_.lastPeriodUs(), clock_.nowMs());

    const uint32_t travelled_mm = calculator_.takeDistanceMm();
    if (travelled_mm > 0) {
        odometer_.addDistance(travelled_mm);
    }

    bus.publish(ChannelId::Speed, calculator_.speedKmh());
    // Conversion en kilometres seulement au moment de publier : la valeur
    // de reference reste entiere en millimetres.
    bus.publish(ChannelId::Odometer, static_cast<float>(odometer_.odometerMm()) / kMmPerKm);
    bus.publish(ChannelId::Trip, static_cast<float>(odometer_.tripMm()) / kMmPerKm);
}

}  // namespace rondo
```

Ajouter `providers/hall_speed_provider.cpp` aux sources de `rondo_core` et le fichier de
test à `rondo_tests`.

- [ ] **Étape 4 : lancer les tests et vérifier qu'ils passent**

```bash
cmake -B build -DCMAKE_BUILD_TYPE=Debug && cmake --build build -j && ctest --test-dir build --output-on-failure
```

Attendu : `100% tests passed, 0 tests failed out of 113`.

- [ ] **Étape 5 : commit**

```bash
git add core tests
git commit -m "core: publish speed, odometer and trip from the hall sensor"
```

---

## Tâche 6 : Projet ESP-IDF et pilotes simples

À partir d'ici, le matériel est nécessaire. Le plan `rondo-hardware` doit être terminé.

**Fichiers :**
- Créer : `firmware/CMakeLists.txt`, `firmware/sdkconfig.defaults`,
  `firmware/main/CMakeLists.txt`, `firmware/main/esp32_clock.h` / `.cpp`,
  `firmware/main/esp32_digital_in.h` / `.cpp`, `firmware/main/esp32_analog_in.h` / `.cpp`,
  `firmware/main/app_main.cpp`

**Produit :** `Esp32Clock`, `Esp32DigitalIn`, `Esp32AnalogIn`, implémentant les interfaces
de `hal/`.

- [ ] **Étape 1 : créer le projet ESP-IDF**

```bash
. $IDF_PATH/export.sh
idf.py --version    # doit repondre 5.2 ou plus
```

`firmware/CMakeLists.txt` :

```cmake
cmake_minimum_required(VERSION 3.20)
include($ENV{IDF_PATH}/tools/cmake/project.cmake)
project(rondo)
```

`firmware/main/CMakeLists.txt` :

```cmake
idf_component_register(
  SRCS
    "app_main.cpp"
    "esp32_clock.cpp"
    "esp32_digital_in.cpp"
    "esp32_analog_in.cpp"
    "../../core/channel_id.cpp"
    "../../core/channel_bus.cpp"
    "../../core/curve.cpp"
    "../../core/crc32.cpp"
    "../../core/speed_calculator.cpp"
    "../../core/odometer_service.cpp"
    "../../core/input/button_decoder.cpp"
    "../../core/providers/digital_input_provider.cpp"
    "../../core/providers/analog_sensor_provider.cpp"
    "../../core/providers/hall_speed_provider.cpp"
  INCLUDE_DIRS "." "../.."
  REQUIRES driver esp_adc esp_timer
)
```

Le fait de compiler `core/` **tel quel**, sans le moindre `#ifdef`, est la vérification que
l'abstraction tient. Si un fichier de `core/` refuse de compiler ici, c'est qu'il a acquis
une dépendance à l'hôte : la corriger dans `core/`, pas ici.

`firmware/sdkconfig.defaults` :

```
CONFIG_IDF_TARGET="esp32s3"
CONFIG_FREERTOS_HZ=1000
CONFIG_ESP_MAIN_TASK_STACK_SIZE=8192
CONFIG_COMPILER_CXX_EXCEPTIONS=n
CONFIG_SPIRAM=y
```

- [ ] **Étape 2 : écrire l'horloge**

`firmware/main/esp32_clock.h` :

```cpp
#pragma once

#include "core/clock.h"
#include "esp_timer.h"

namespace rondo {

class Esp32Clock final : public IClock {
 public:
    uint32_t nowMs() const override {
        return static_cast<uint32_t>(esp_timer_get_time() / 1000);
    }
};

}  // namespace rondo
```

- [ ] **Étape 3 : écrire l'entrée numérique**

`firmware/main/esp32_digital_in.h` :

```cpp
#pragma once

#include "driver/gpio.h"
#include "hal/i_digital_in.h"

namespace rondo {

// Lit un niveau logique. L'inversion des entrees actives-basses est le
// role du fournisseur : ce pilote ne fait aucune interpretation.
class Esp32DigitalIn final : public IDigitalIn {
 public:
    // `pull_up` a activer pour le contacteur de point mort et les boutons,
    // qui mettent la ligne a la masse. Les sorties d'opto-coupleur ont
    // deja leur resistance de rappel sur la carte.
    Esp32DigitalIn(gpio_num_t pin, bool pull_up);

    bool read() const override { return gpio_get_level(pin_) != 0; }

 private:
    gpio_num_t pin_;
};

}  // namespace rondo
```

`firmware/main/esp32_digital_in.cpp` :

```cpp
#include "esp32_digital_in.h"

namespace rondo {

Esp32DigitalIn::Esp32DigitalIn(gpio_num_t pin, bool pull_up) : pin_(pin) {
    gpio_config_t cfg = {};
    cfg.pin_bit_mask = 1ULL << pin;
    cfg.mode = GPIO_MODE_INPUT;
    cfg.pull_up_en = pull_up ? GPIO_PULLUP_ENABLE : GPIO_PULLUP_DISABLE;
    cfg.pull_down_en = GPIO_PULLDOWN_DISABLE;
    cfg.intr_type = GPIO_INTR_DISABLE;
    gpio_config(&cfg);
}

}  // namespace rondo
```

- [ ] **Étape 4 : écrire l'entrée analogique**

`firmware/main/esp32_analog_in.h` / `.cpp` — enveloppe `esp_adc/adc_oneshot.h` :
`adc_oneshot_new_unit`, `adc_oneshot_config_channel` avec `ADC_ATTEN_DB_12` (plage utile
0 à ~3,1 V, celle retenue au dimensionnement du diviseur), `adc_oneshot_read`.

`readRaw()` renvoie la **moyenne de 8 lectures consécutives** : l'ADC de l'ESP32-S3 est
bruyant, et un moyennage matériel léger évite de faire trembler l'affichage de tension.
Aucune autre conversion : la mise à l'échelle appartient à `AnalogSensorProvider`.

- [ ] **Étape 5 : écrire un `app_main` de vérification**

`firmware/main/app_main.cpp` — première version : instancier `Esp32Clock`, trois
`Esp32DigitalIn` (clignotants et point mort) et un `Esp32AnalogIn` (tension batterie), puis
imprimer leur état sur la liaison série toutes les 500 ms.

- [ ] **Étape 6 : flasher et vérifier sur le banc**

```bash
cd firmware
idf.py set-target esp32s3
idf.py build flash monitor
```

Vérifier au moniteur série, en s'appuyant sur le prototype de `rondo-hardware` tâche 7 :

| Stimulus | Attendu à l'écran |
|---|---|
| Aucun | les trois entrées à `1`, tension ≈ celle de l'alimentation de laboratoire |
| +12 V sur la voie clignotant gauche | l'entrée correspondante passe à `0` |
| Voie point mort à la masse | l'entrée point mort passe à `0` |
| Entrée à 12,0 V | valeur brute ≈ 2158, soit 12,5 V une fois mise à l'échelle |

Si la valeur brute s'écarte de plus de 5 %, mesurer les résistances réelles du diviseur et
corriger le coefficient d'échelle dans le profil — **pas dans le code**.

- [ ] **Étape 7 : commit**

```bash
git add firmware
git commit -m "firmware: add the ESP-IDF project and the clock, GPIO and ADC drivers"
```

---

## Tâche 7 : Compteur d'impulsions et FRAM

**Fichiers :**
- Créer : `firmware/main/esp32_pulse_counter.h` / `.cpp`, `firmware/main/esp32_fram.h` / `.cpp`
- Modifier : `firmware/main/CMakeLists.txt`, `firmware/main/app_main.cpp`

**Produit :** `Esp32PulseCounter` et `Esp32Fram`.

- [ ] **Étape 1 : écrire le compteur d'impulsions**

Interruption sur front descendant, `esp_timer_get_time()` pour la période. À 30 Hz au
maximum en usage réel, la charge est négligeable.

`firmware/main/esp32_pulse_counter.h` :

```cpp
#pragma once

#include <cstdint>

#include "driver/gpio.h"
#include "hal/i_pulse_counter.h"

namespace rondo {

// Compte les impulsions du capteur a effet Hall et mesure leur periode.
//
// Un intervalle minimal est impose DANS l'interruption : une salve
// parasite d'allumage ne doit pas pouvoir gonfler le compteur, sinon
// l'odometre serait faux de facon invisible. Le filtre RC du schema
// (15,9 kHz) et le controle de vraisemblance du SpeedCalculator sont les
// deux autres lignes de defense.
class Esp32PulseCounter final : public IPulseCounter {
 public:
    Esp32PulseCounter(gpio_num_t pin, uint32_t min_interval_us);

    uint32_t count() const override;
    uint32_t lastPeriodUs() const override;

 private:
    static void IRAM_ATTR isr(void* arg);

    gpio_num_t pin_;
    uint32_t min_interval_us_;
    volatile uint32_t count_ = 0;
    volatile uint32_t period_us_ = 0;
    volatile int64_t last_edge_us_ = 0;
};

}  // namespace rondo
```

`min_interval_us` se déduit de la vitesse maximale plausible du profil :

```
min_interval_us = 3600 x circonference_mm / (aimants x vitesse_max_kmh)
Pour la KDX : 3600 x 2170 / (2 x 180) = 21 700 us
```

- [ ] **Étape 2 : écrire le pilote FRAM**

`firmware/main/esp32_fram.h` / `.cpp` — enveloppe `driver/i2c_master.h` :
`i2c_new_master_bus`, `i2c_master_bus_add_device` à l'adresse **0x50**, puis
`i2c_master_transmit` et `i2c_master_transmit_receive`.

Protocole du MB85RC64 : l'adresse mémoire est envoyée sur **deux octets, poids fort en
premier**, avant les données. Bus à 400 kHz.

`read()` et `write()` renvoient `false` si la transaction I²C échoue ou si l'accès dépasse
les 8 Kio du composant — jamais d'exception, comme le veut l'interface.

- [ ] **Étape 3 : vérifier la FRAM au banc**

Compléter `app_main.cpp` : au démarrage, écrire un motif connu, relire, comparer, imprimer
le résultat, puis effacer.

```
FRAM: write 20 bytes at 0x0000 ... ok
FRAM: read back ... ok, match
```

Couper puis rétablir l'alimentation sans reflasher, et vérifier que la relecture retrouve
le motif. **C'est la vérification qui valide toute la stratégie d'odométrie** : si la FRAM
ne retient pas, rien d'autre ne sert.

- [ ] **Étape 4 : vérifier le compteur d'impulsions au banc**

Approcher et éloigner l'aimant du capteur à la main, dix fois.

Attendu : `count` progresse **exactement de 10**. Un compte supérieur signale des rebonds
que le filtre ne rattrape pas — vérifier le condensateur de 10 nF avant d'augmenter
`min_interval_us`.

Puis faire tourner la roue à la main à vitesse à peu près constante et vérifier que
`lastPeriodUs` est stable à ± 10 % près.

- [ ] **Étape 5 : commit**

```bash
git add firmware
git commit -m "firmware: add the hall pulse counter and the FRAM driver"
```

---

## Tâche 8 : Application cible et validation

**Fichiers :**
- Modifier : `firmware/main/app_main.cpp`, `firmware/main/CMakeLists.txt`
- Créer : `docs/bench-acquisition.md`

**Produit :** le firmware complet, acquisition et affichage.

- [ ] **Étape 1 : assembler l'application**

Dans `app_main.cpp` : instancier les pilotes, `OdometerService`, tous les fournisseurs et le
`ChannelBus`, puis créer deux tâches FreeRTOS.

```cpp
// L'acquisition passe AVANT l'affichage. Si LVGL sature, le kilometrage
// doit rester juste : c'est la seule valeur du systeme qu'on ne peut pas
// reconstruire apres coup.
xTaskCreatePinnedToCore(acquisitionTask, "acq",  4096, &ctx, 6, nullptr, 0);
xTaskCreatePinnedToCore(uiTask,          "ui",  12288, &ctx, 3, nullptr, 1);
```

`acquisitionTask` appelle `poll()` de chaque fournisseur à **50 Hz**.
`uiTask` appelle `lv_timer_handler()` et `update()` de l'archétype courant à **30 fps**.

Le bus est écrit par une tâche et lu par l'autre : protéger les accès par un mutex
FreeRTOS, ou déclarer les entrées du bus `std::atomic`. **Choisir le mutex** — plus simple à
relire, et la contention est nulle à ces cadences.

- [ ] **Étape 2 : vérifier que rien de `core/` ni de `ui/` n'a été modifié**

```bash
git diff --name-only HEAD~6 -- core/ ui/ hal/
```

Attendu : **aucun fichier**, hormis ceux créés par les tâches 1 à 5 de ce plan. Si le
portage a imposé de modifier `ui/`, une frontière a été franchie : la corriger plutôt que
la contourner.

- [ ] **Étape 3 : valider l'odométrie sur une distance connue**

Sur la moto, roue avant levée, faire tourner la roue à la main **exactement 100 tours**.

```
Attendu : odometre progresse de 100 x 2170 mm = 217,0 m, a +/- 1 %
```

Si l'écart dépasse 1 %, mesurer la circonférence réelle en marquant le pneu et en faisant
rouler la moto sur un tour complet — **et corriger le profil, pas le code**.

- [ ] **Étape 4 : valider la persistance**

Noter l'odomètre, couper le contact franchement (pas d'arrêt propre), remettre le contact.

Attendu : l'odomètre reprend à la valeur notée, **à 10 mètres près au maximum**. C'est la
vérification de bout en bout de la décision d'écriture continue.

Répéter cinq fois, dont une coupure pendant que la roue tourne.

- [ ] **Étape 5 : valider les entrées sur la moto**

| Stimulus | Canal attendu |
|---|---|
| Point mort engagé | `neutral` à 1 |
| Clignotant gauche | `turn_left` clignote à ~1,5 Hz |
| Plein phare | `high_beam` à 1 |
| Moteur chaud | `engine_temp` cohérent avec le témoin d'origine |
| Contact mis, moteur arrêté | `battery_volts` ≈ 12,8 V (LiFePO4 au repos) |
| Moteur tournant | `battery_volts` ≈ 14,4 V (en charge) |

Vérifier aussi qu'un canal sans capteur — la jauge d'essence si elle n'est pas installée —
**reste absent et masque son widget**, sans message d'erreur.

- [ ] **Étape 6 : écrire `docs/bench-acquisition.md`**

Consigner les cinq procédures ci-dessus avec une colonne « mesuré », la date, et la
circonférence de roue retenue. Ce document est la recette de réception de l'acquisition.

- [ ] **Étape 7 : lancer la totalité des tests hôte**

```bash
ctest --test-dir build --output-on-failure
```

Attendu : `100% tests passed, 0 tests failed out of 113`.

- [ ] **Étape 8 : commit**

```bash
git add firmware docs/bench-acquisition.md
git commit -m "firmware: wire the acquisition and UI tasks and validate on the bike"
```

---

## Suites

À la fin de ce plan, le compteur fonctionne sur la moto : vitesse, kilométrage persistant,
température, tension et témoins, avec les six thèmes. 113 tests couvrent toute la logique
de calcul sur PC.

| Plan | Contenu | Dépend de |
|---|---|---|
| `rondo-config` | Portail WiFi, profil moto JSON, import et export | ce plan |
| `rondo-nav` | GNSS, chargement et suivi de trace GPX | `rondo-config` |

Jusqu'ici, la configuration est **codée en dur dans `app_main.cpp`** : circonférence de
roue, nombre d'aimants, broches, courbes. C'est acceptable pour valider le matériel sur une
seule moto, et c'est précisément ce que `rondo-config` remplace pour rendre le projet
utilisable par d'autres.
