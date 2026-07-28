# Rondo — Plan d'implémentation : socle logiciel

> **Pour les agents :** SOUS-COMPÉTENCE REQUISE — utiliser `superpowers:subagent-driven-development`
> (recommandé) ou `superpowers:executing-plans` pour dérouler ce plan tâche par tâche.
> Les étapes utilisent la syntaxe case à cocher (`- [ ]`).

**Objectif :** construire le socle sur lequel reposent toutes les phases suivantes — bus de
canaux, interfaces matérielles abstraites, implémentations de simulation, rejeu de scénarios
et harnais de test — le tout compilable et testable **sur PC, sans aucun matériel**.

**Architecture :** quatre couches dont les deux supérieures ne référencent jamais d'API
matérielle. Ce plan livre le bus de canaux (la pièce maîtresse), les interfaces de la couche
matérielle, leurs implémentations de simulation, et une tranche verticale complète —
un fournisseur de bout en bout — qui prouve que les interfaces tiennent ensemble.

**Pile technique :** C++17, CMake ≥ 3.20, Catch2 v3, GitHub Actions.

## Contraintes globales

- **Aucune dépendance à ESP-IDF, à LVGL ou à un quelconque matériel dans ce plan.** Tout se
  compile et s'exécute sur une machine de développement ordinaire. L'implémentation ESP32
  de la couche matérielle relève du plan `rondo-acquisition`, l'interface graphique du plan
  `rondo-ui`.
- **C++17**, pas plus récent : ESP-IDF doit pouvoir compiler `core/` tel quel.
- **Aucune allocation dynamique dans `core/`.** Le bus de canaux utilise un tableau de
  taille fixe. `sim/` et les tests s'autorisent la bibliothèque standard sans restriction —
  ils ne tournent jamais sur la cible.
- Compilation **sans le moindre avertissement** : `-Wall -Wextra -Wpedantic -Werror`.
- Les couches `core/` et `ui/` **ne référencent jamais une broche matérielle**. Toute
  affectation de broche vient du profil moto, jamais du code.
- Les identifiants de code, les noms de fichiers et les messages de commit sont en
  **anglais** ; les commentaires, la documentation et les noms de tests sont en **français**.
  Le projet est francophone mais son code doit rester contribuable à l'international.
- Le dépôt ne contient **aucune image** pour l'instant (décision projet).

## Structure des fichiers

| Fichier | Responsabilité |
|---|---|
| `CMakeLists.txt` | Projet racine, options de compilation, activation de CTest |
| `.github/workflows/ci.yml` | Compilation et exécution des tests à chaque poussée |
| `core/CMakeLists.txt` | Bibliothèque `rondo_core` |
| `core/version.h` | Version du socle |
| `core/clock.h` | `IClock` — abstraction du temps, indispensable à des tests déterministes |
| `core/channel_id.h` / `.cpp` | Énumération des canaux et leurs métadonnées |
| `core/channel_bus.h` / `.cpp` | Le bus de canaux : publication, lecture, validité |
| `core/provider.h` | `IProvider` — contrat commun à tous les fournisseurs |
| `core/providers/digital_input_provider.h` / `.cpp` | Fournisseur de référence, tranche verticale |
| `hal/i_digital_in.h` | Entrée tout ou rien |
| `hal/i_analog_in.h` | Entrée analogique brute |
| `hal/i_pulse_counter.h` | Compteur d'impulsions (capteur de vitesse) |
| `hal/i_persistent_store.h` | Stockage persistant (FRAM) |
| `sim/CMakeLists.txt` | Bibliothèque `rondo_sim` et exécutable `rondo_sim_run` |
| `sim/sim_clock.h` | Horloge pilotée à la main |
| `sim/sim_digital_in.h`, `sim_analog_in.h`, `sim_pulse_counter.h/.cpp`, `sim_store.h/.cpp` | Implémentations de simulation |
| `sim/scenario.h` / `.cpp` | Analyse des fichiers de scénario CSV |
| `sim/scenario_player.h` / `.cpp` | Rejeu d'un scénario sur les entrées simulées |
| `sim/main.cpp` | Exécutable de simulation sans interface graphique |
| `sim/scenarios/*.csv` | Scénarios de référence |
| `tests/CMakeLists.txt` | Cible `rondo_tests` |
| `tests/test_*.cpp` | Un fichier de test par unité |

**Convention d'inclusion :** toujours depuis la racine du dépôt, par exemple
`#include "core/channel_bus.h"`. Jamais de chemin relatif avec `../`.

---

## Tâche 1 : Squelette de compilation, tests et intégration continue

Rien ne peut être développé en TDD tant que lancer un test n'est pas trivial. Cette tâche
n'existe que pour rendre `ctest` opérationnel.

**Fichiers :**
- Créer : `CMakeLists.txt`, `core/CMakeLists.txt`, `core/version.h`,
  `tests/CMakeLists.txt`, `tests/test_version.cpp`, `.github/workflows/ci.yml`, `.gitignore`

**Produit :** la bibliothèque `rondo_core`, la cible de test `rondo_tests`, et la commande
`ctest --test-dir build` utilisée par toutes les tâches suivantes.

- [ ] **Étape 1 : écrire le test qui échoue**

`tests/test_version.cpp` :

```cpp
#include <catch2/catch_test_macros.hpp>

#include <string>

#include "core/version.h"

TEST_CASE("la version du socle est exposee") {
    REQUIRE(std::string(rondo::kVersion) == "0.1.0");
}
```

- [ ] **Étape 2 : écrire les fichiers de compilation**

`CMakeLists.txt` à la racine :

```cmake
cmake_minimum_required(VERSION 3.20)
project(rondo LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

if(MSVC)
  add_compile_options(/W4 /WX)
else()
  add_compile_options(-Wall -Wextra -Wpedantic -Werror)
endif()

add_subdirectory(core)

include(CTest)
if(BUILD_TESTING)
  add_subdirectory(tests)
endif()
```

`core/CMakeLists.txt` :

```cmake
add_library(rondo_core INTERFACE)
target_include_directories(rondo_core INTERFACE ${PROJECT_SOURCE_DIR})
```

`rondo_core` est une bibliothèque d'interface tant qu'elle n'a pas de fichier `.cpp` ;
elle deviendra une vraie bibliothèque en tâche 3.

`tests/CMakeLists.txt` :

```cmake
include(FetchContent)
FetchContent_Declare(
  Catch2
  GIT_REPOSITORY https://github.com/catchorg/Catch2.git
  GIT_TAG        v3.5.2
)
FetchContent_MakeAvailable(Catch2)

add_executable(rondo_tests
  test_version.cpp
)
target_link_libraries(rondo_tests PRIVATE rondo_core Catch2::Catch2WithMain)

list(APPEND CMAKE_MODULE_PATH ${catch2_SOURCE_DIR}/extras)
include(Catch)
catch_discover_tests(rondo_tests)
```

`.gitignore` — ajouter à la suite des lignes existantes :

```
build/
```

- [ ] **Étape 3 : lancer le test et vérifier qu'il échoue**

```bash
cmake -B build -DCMAKE_BUILD_TYPE=Debug && cmake --build build -j
```

Attendu : **échec de compilation**, `core/version.h: No such file or directory`.

- [ ] **Étape 4 : écrire l'implémentation minimale**

`core/version.h` :

```cpp
#pragma once

namespace rondo {

// Version du socle logiciel, independante de la version du materiel.
inline constexpr const char* kVersion = "0.1.0";

}  // namespace rondo
```

- [ ] **Étape 5 : lancer les tests et vérifier qu'ils passent**

```bash
cmake --build build -j && ctest --test-dir build --output-on-failure
```

Attendu : `100% tests passed, 0 tests failed out of 1`.

- [ ] **Étape 6 : ajouter l'intégration continue**

`.github/workflows/ci.yml` :

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure
        run: cmake -B build -DCMAKE_BUILD_TYPE=Debug

      - name: Build
        run: cmake --build build -j

      - name: Test
        run: ctest --test-dir build --output-on-failure
```

- [ ] **Étape 7 : commit**

```bash
git add CMakeLists.txt core tests .github .gitignore
git commit -m "build: cmake skeleton, Catch2 test target and CI"
```

---

## Tâche 2 : Abstraction du temps

Tester la péremption d'un canal avec l'horloge réelle imposerait des attentes dans les
tests. Le temps est donc une dépendance injectée, comme n'importe quelle autre.

**Fichiers :**
- Créer : `core/clock.h`, `sim/sim_clock.h`, `sim/CMakeLists.txt`,
  `tests/test_sim_clock.cpp`
- Modifier : `CMakeLists.txt`, `tests/CMakeLists.txt`

**Produit :**
- `rondo::IClock` — méthode virtuelle pure `uint32_t nowMs() const`
- `rondo::SimClock` — implémente `IClock`, plus `void advanceMs(uint32_t)` et
  `void setMs(uint32_t)`

- [ ] **Étape 1 : écrire le test qui échoue**

`tests/test_sim_clock.cpp` :

```cpp
#include <catch2/catch_test_macros.hpp>

#include "sim/sim_clock.h"

using namespace rondo;

TEST_CASE("l'horloge simulee demarre a zero") {
    SimClock clock;
    REQUIRE(clock.nowMs() == 0u);
}

TEST_CASE("avancer l'horloge cumule les increments") {
    SimClock clock;
    clock.advanceMs(150);
    clock.advanceMs(50);
    REQUIRE(clock.nowMs() == 200u);
}

TEST_CASE("positionner l'horloge remplace la valeur courante") {
    SimClock clock;
    clock.advanceMs(150);
    clock.setMs(42);
    REQUIRE(clock.nowMs() == 42u);
}
```

- [ ] **Étape 2 : lancer le test et vérifier qu'il échoue**

```bash
cmake --build build -j
```

Attendu : **échec**, `sim/sim_clock.h: No such file or directory`.

- [ ] **Étape 3 : écrire l'implémentation minimale**

`core/clock.h` :

```cpp
#pragma once

#include <cstdint>

namespace rondo {

// Source de temps monotone en millisecondes.
// Injectee partout plutot qu'appelee directement : c'est ce qui rend la
// peremption des canaux testable sans attente reelle.
class IClock {
 public:
    virtual ~IClock() = default;
    virtual uint32_t nowMs() const = 0;
};

}  // namespace rondo
```

`sim/sim_clock.h` :

```cpp
#pragma once

#include <cstdint>

#include "core/clock.h"

namespace rondo {

// Horloge pilotee a la main par les tests et le rejeu de scenarios.
class SimClock final : public IClock {
 public:
    uint32_t nowMs() const override { return now_ms_; }

    void advanceMs(uint32_t delta_ms) { now_ms_ += delta_ms; }
    void setMs(uint32_t t_ms) { now_ms_ = t_ms; }

 private:
    uint32_t now_ms_ = 0;
};

}  // namespace rondo
```

`sim/CMakeLists.txt` :

```cmake
add_library(rondo_sim INTERFACE)
target_include_directories(rondo_sim INTERFACE ${PROJECT_SOURCE_DIR})
target_link_libraries(rondo_sim INTERFACE rondo_core)
```

Dans `CMakeLists.txt` racine, ajouter après `add_subdirectory(core)` :

```cmake
add_subdirectory(sim)
```

Dans `tests/CMakeLists.txt`, ajouter `test_sim_clock.cpp` à la liste des sources de
`rondo_tests`, et `rondo_sim` à `target_link_libraries`.

- [ ] **Étape 4 : lancer les tests et vérifier qu'ils passent**

```bash
cmake -B build -DCMAKE_BUILD_TYPE=Debug && cmake --build build -j && ctest --test-dir build --output-on-failure
```

Attendu : `100% tests passed, 0 tests failed out of 4`.

- [ ] **Étape 5 : commit**

```bash
git add core/clock.h sim CMakeLists.txt tests
git commit -m "core: inject an abstract clock and add its simulated implementation"
```

---

## Tâche 3 : Identifiants de canaux et métadonnées

**Fichiers :**
- Créer : `core/channel_id.h`, `core/channel_id.cpp`, `tests/test_channel_id.cpp`
- Modifier : `core/CMakeLists.txt`, `tests/CMakeLists.txt`

**Produit :**
- `enum class ChannelId : uint8_t` avec le sentinelle `Count` en dernier
- `const char* toString(ChannelId)` — nom court, utilisé dans les journaux et les scénarios
- `const char* unitOf(ChannelId)` — unité affichable, chaîne vide pour les booléens

- [ ] **Étape 1 : écrire le test qui échoue**

`tests/test_channel_id.cpp` :

```cpp
#include <catch2/catch_test_macros.hpp>

#include <cstddef>
#include <set>
#include <string>

#include "core/channel_id.h"

using namespace rondo;

TEST_CASE("chaque canal a un nom et une unite") {
    for (size_t i = 0; i < static_cast<size_t>(ChannelId::Count); ++i) {
        const ChannelId id = static_cast<ChannelId>(i);
        REQUIRE(std::string(toString(id)).length() > 0);
        REQUIRE(unitOf(id) != nullptr);
    }
}

TEST_CASE("les noms de canaux sont tous distincts") {
    std::set<std::string> names;
    for (size_t i = 0; i < static_cast<size_t>(ChannelId::Count); ++i) {
        names.insert(toString(static_cast<ChannelId>(i)));
    }
    REQUIRE(names.size() == static_cast<size_t>(ChannelId::Count));
}

TEST_CASE("les canaux booleens n'ont pas d'unite") {
    REQUIRE(std::string(unitOf(ChannelId::Neutral)).empty());
    REQUIRE(std::string(unitOf(ChannelId::TurnLeft)).empty());
}

TEST_CASE("les canaux physiques portent leur unite") {
    REQUIRE(std::string(unitOf(ChannelId::Speed)) == "km/h");
    REQUIRE(std::string(unitOf(ChannelId::BatteryVolts)) == "V");
}
```

- [ ] **Étape 2 : lancer le test et vérifier qu'il échoue**

```bash
cmake --build build -j
```

Attendu : **échec**, `core/channel_id.h: No such file or directory`.

- [ ] **Étape 3 : écrire l'implémentation minimale**

`core/channel_id.h` :

```cpp
#pragma once

#include <cstdint>

namespace rondo {

// Liste des grandeurs que le tableau de bord sait transporter.
// Ajouter un canal ici n'oblige aucun theme ni aucun fournisseur a changer :
// un canal que personne n'alimente reste simplement Absent.
enum class ChannelId : uint8_t {
    Speed,         // km/h
    Odometer,      // km
    Trip,          // km
    Rpm,           // tr/min
    EngineTemp,    // degres Celsius
    FuelLevel,     // pourcentage
    BatteryVolts,  // volts
    Neutral,       // booleen
    TurnLeft,      // booleen
    TurnRight,     // booleen
    HighBeam,      // booleen
    Count          // sentinelle, doit rester en dernier
};

// Nom court et stable. Sert de cle dans les scenarios et les journaux :
// ne jamais le modifier sans migrer les fichiers de scenario.
const char* toString(ChannelId id);

// Unite affichable. Chaine vide pour les canaux booleens.
const char* unitOf(ChannelId id);

}  // namespace rondo
```

`core/channel_id.cpp` :

```cpp
#include "core/channel_id.h"

namespace rondo {

const char* toString(ChannelId id) {
    switch (id) {
        case ChannelId::Speed:        return "speed";
        case ChannelId::Odometer:     return "odometer";
        case ChannelId::Trip:         return "trip";
        case ChannelId::Rpm:          return "rpm";
        case ChannelId::EngineTemp:   return "engine_temp";
        case ChannelId::FuelLevel:    return "fuel_level";
        case ChannelId::BatteryVolts: return "battery_volts";
        case ChannelId::Neutral:      return "neutral";
        case ChannelId::TurnLeft:     return "turn_left";
        case ChannelId::TurnRight:    return "turn_right";
        case ChannelId::HighBeam:     return "high_beam";
        case ChannelId::Count:        return "invalid";
    }
    return "invalid";
}

const char* unitOf(ChannelId id) {
    switch (id) {
        case ChannelId::Speed:        return "km/h";
        case ChannelId::Odometer:     return "km";
        case ChannelId::Trip:         return "km";
        case ChannelId::Rpm:          return "tr/min";
        case ChannelId::EngineTemp:   return "°C";
        case ChannelId::FuelLevel:    return "%";
        case ChannelId::BatteryVolts: return "V";
        case ChannelId::Neutral:      return "";
        case ChannelId::TurnLeft:     return "";
        case ChannelId::TurnRight:    return "";
        case ChannelId::HighBeam:     return "";
        case ChannelId::Count:        return "";
    }
    return "";
}

}  // namespace rondo
```

Les `switch` sans `default` sont volontaires : avec `-Wall`, ajouter un canal à
l'énumération sans le traiter ici déclenche un avertissement, donc une erreur de
compilation. C'est le filet de sécurité qui garantit que le tableau reste complet.

Dans `core/CMakeLists.txt`, remplacer la bibliothèque d'interface par une vraie
bibliothèque :

```cmake
add_library(rondo_core
  channel_id.cpp
)
target_include_directories(rondo_core PUBLIC ${PROJECT_SOURCE_DIR})
```

Ajouter `test_channel_id.cpp` aux sources de `rondo_tests`.

- [ ] **Étape 4 : lancer les tests et vérifier qu'ils passent**

```bash
cmake -B build -DCMAKE_BUILD_TYPE=Debug && cmake --build build -j && ctest --test-dir build --output-on-failure
```

Attendu : `100% tests passed, 0 tests failed out of 8`.

- [ ] **Étape 5 : commit**

```bash
git add core tests
git commit -m "core: add channel identifiers and their metadata"
```

---

## Tâche 4 : Le bus de canaux

C'est la pièce maîtresse du projet. L'état `Absent` est le mécanisme qui permet à un même
thème de fonctionner sur des motos différentes sans être réécrit.

**Fichiers :**
- Créer : `core/channel_bus.h`, `core/channel_bus.cpp`, `tests/test_channel_bus.cpp`
- Modifier : `core/CMakeLists.txt`, `tests/CMakeLists.txt`

**Consomme :** `IClock` (tâche 2), `ChannelId` (tâche 3).
**Produit :**
- `enum class Validity : uint8_t { Absent, Stale, Fresh }`
- `struct Reading { float value; Validity validity; bool isUsable() const; }`
- `class ChannelBus` — `explicit ChannelBus(const IClock&)`,
  `void declare(ChannelId, uint32_t ttl_ms)`, `void publish(ChannelId, float)`,
  `Reading read(ChannelId) const`

Règles de validité, à respecter exactement :

| Situation | Validité | Valeur lue |
|---|---|---|
| Canal jamais déclaré | `Absent` | `0.0f` |
| Déclaré, jamais publié | `Stale` | `0.0f` |
| Publié, âge ≤ durée de vie | `Fresh` | dernière publiée |
| Publié, âge > durée de vie | `Stale` | dernière publiée |

- [ ] **Étape 1 : écrire les tests qui échouent**

`tests/test_channel_bus.cpp` :

```cpp
#include <catch2/catch_test_macros.hpp>

#include "core/channel_bus.h"
#include "sim/sim_clock.h"

using namespace rondo;

TEST_CASE("un canal jamais declare est absent") {
    SimClock clock;
    ChannelBus bus(clock);
    REQUIRE(bus.read(ChannelId::FuelLevel).validity == Validity::Absent);
}

TEST_CASE("un canal declare mais jamais publie est perime") {
    SimClock clock;
    ChannelBus bus(clock);
    bus.declare(ChannelId::Speed, 500);
    REQUIRE(bus.read(ChannelId::Speed).validity == Validity::Stale);
}

TEST_CASE("un canal publie est frais et porte sa valeur") {
    SimClock clock;
    ChannelBus bus(clock);
    bus.declare(ChannelId::Speed, 500);
    bus.publish(ChannelId::Speed, 68.0f);

    const Reading r = bus.read(ChannelId::Speed);
    REQUIRE(r.validity == Validity::Fresh);
    REQUIRE(r.value == 68.0f);
    REQUIRE(r.isUsable());
}

TEST_CASE("un canal devient perime passe sa duree de vie mais garde sa valeur") {
    SimClock clock;
    ChannelBus bus(clock);
    bus.declare(ChannelId::Speed, 500);
    bus.publish(ChannelId::Speed, 68.0f);
    clock.advanceMs(501);

    const Reading r = bus.read(ChannelId::Speed);
    REQUIRE(r.validity == Validity::Stale);
    REQUIRE(r.value == 68.0f);
    REQUIRE_FALSE(r.isUsable());
}

TEST_CASE("un canal reste frais jusqu'a sa duree de vie incluse") {
    SimClock clock;
    ChannelBus bus(clock);
    bus.declare(ChannelId::Speed, 500);
    bus.publish(ChannelId::Speed, 68.0f);
    clock.advanceMs(500);

    REQUIRE(bus.read(ChannelId::Speed).validity == Validity::Fresh);
}

TEST_CASE("publier sur un canal non declare est sans effet") {
    SimClock clock;
    ChannelBus bus(clock);
    bus.publish(ChannelId::FuelLevel, 42.0f);

    const Reading r = bus.read(ChannelId::FuelLevel);
    REQUIRE(r.validity == Validity::Absent);
    REQUIRE(r.value == 0.0f);
}

TEST_CASE("republier rafraichit un canal perime") {
    SimClock clock;
    ChannelBus bus(clock);
    bus.declare(ChannelId::Speed, 500);
    bus.publish(ChannelId::Speed, 68.0f);
    clock.advanceMs(501);
    REQUIRE(bus.read(ChannelId::Speed).validity == Validity::Stale);

    bus.publish(ChannelId::Speed, 70.0f);
    const Reading r = bus.read(ChannelId::Speed);
    REQUIRE(r.validity == Validity::Fresh);
    REQUIRE(r.value == 70.0f);
}

TEST_CASE("les canaux sont independants les uns des autres") {
    SimClock clock;
    ChannelBus bus(clock);
    bus.declare(ChannelId::Speed, 500);
    bus.declare(ChannelId::Rpm, 200);
    bus.publish(ChannelId::Speed, 68.0f);
    bus.publish(ChannelId::Rpm, 4200.0f);
    clock.advanceMs(300);

    REQUIRE(bus.read(ChannelId::Speed).validity == Validity::Fresh);
    REQUIRE(bus.read(ChannelId::Rpm).validity == Validity::Stale);
    REQUIRE(bus.read(ChannelId::FuelLevel).validity == Validity::Absent);
}

TEST_CASE("la peremption reste correcte au rebouclage de l'horloge") {
    SimClock clock;
    ChannelBus bus(clock);
    clock.setMs(0xFFFFFF00u);
    bus.declare(ChannelId::Speed, 500);
    bus.publish(ChannelId::Speed, 68.0f);

    clock.setMs(0x0000004Cu);  // 76 apres rebouclage, soit 332 ms ecoulees
    REQUIRE(bus.read(ChannelId::Speed).validity == Validity::Fresh);
}
```

- [ ] **Étape 2 : lancer les tests et vérifier qu'ils échouent**

```bash
cmake --build build -j
```

Attendu : **échec**, `core/channel_bus.h: No such file or directory`.

- [ ] **Étape 3 : écrire l'implémentation minimale**

`core/channel_bus.h` :

```cpp
#pragma once

#include <array>
#include <cstddef>
#include <cstdint>

#include "core/channel_id.h"
#include "core/clock.h"

namespace rondo {

// Absent : la moto n'a pas ce capteur. Le widget qui en depend se masque.
// Stale  : le capteur existe mais la valeur n'est plus a jour. Le widget
//          s'affiche, mais l'interface DOIT signaler que la valeur est vieille.
// Fresh  : valeur exploitable.
enum class Validity : uint8_t { Absent, Stale, Fresh };

struct Reading {
    float value = 0.0f;
    Validity validity = Validity::Absent;

    bool isUsable() const { return validity == Validity::Fresh; }
};

// Point de rendez-vous unique entre les fournisseurs et l'interface.
// Les fournisseurs publient, l'interface lit, et les deux s'ignorent.
// Taille fixe, aucune allocation : utilisable tel quel sur la cible.
class ChannelBus {
 public:
    explicit ChannelBus(const IClock& clock);

    // Declare qu'un canal existe sur cette moto, avec sa duree de fraicheur.
    // Un canal non declare reste Absent quoi qu'il arrive.
    void declare(ChannelId id, uint32_t ttl_ms);

    // Publie une valeur. Sans effet si le canal n'a pas ete declare :
    // un fournisseur mal configure ne peut pas faire apparaitre un canal
    // que le profil moto n'a pas prevu.
    void publish(ChannelId id, float value);

    // Ne echoue jamais. Un canal inconnu se lit Absent.
    Reading read(ChannelId id) const;

 private:
    struct Entry {
        float value = 0.0f;
        uint32_t updated_ms = 0;
        uint32_t ttl_ms = 0;
        bool declared = false;
        bool ever_published = false;
    };

    const IClock& clock_;
    std::array<Entry, static_cast<size_t>(ChannelId::Count)> entries_{};
};

}  // namespace rondo
```

`core/channel_bus.cpp` :

```cpp
#include "core/channel_bus.h"

namespace rondo {

ChannelBus::ChannelBus(const IClock& clock) : clock_(clock) {}

void ChannelBus::declare(ChannelId id, uint32_t ttl_ms) {
    Entry& e = entries_[static_cast<size_t>(id)];
    e.declared = true;
    e.ttl_ms = ttl_ms;
}

void ChannelBus::publish(ChannelId id, float value) {
    Entry& e = entries_[static_cast<size_t>(id)];
    if (!e.declared) {
        return;
    }
    e.value = value;
    e.updated_ms = clock_.nowMs();
    e.ever_published = true;
}

Reading ChannelBus::read(ChannelId id) const {
    const Entry& e = entries_[static_cast<size_t>(id)];
    if (!e.declared) {
        return Reading{};
    }
    if (!e.ever_published) {
        return Reading{0.0f, Validity::Stale};
    }
    // Soustraction non signee : le rebouclage de l'horloge a 2^32 ms
    // (environ 49 jours) donne malgre tout l'age correct.
    const uint32_t age_ms = clock_.nowMs() - e.updated_ms;
    return Reading{e.value, age_ms <= e.ttl_ms ? Validity::Fresh : Validity::Stale};
}

}  // namespace rondo
```

Ajouter `channel_bus.cpp` aux sources de `rondo_core` et `test_channel_bus.cpp` à celles
de `rondo_tests`.

- [ ] **Étape 4 : lancer les tests et vérifier qu'ils passent**

```bash
cmake -B build -DCMAKE_BUILD_TYPE=Debug && cmake --build build -j && ctest --test-dir build --output-on-failure
```

Attendu : `100% tests passed, 0 tests failed out of 17`.

- [ ] **Étape 5 : commit**

```bash
git add core tests
git commit -m "core: add the channel bus with absent, stale and fresh states"
```

---

## Tâche 5 : Interfaces matérielles et implémentations de simulation

**Fichiers :**
- Créer : `hal/i_digital_in.h`, `hal/i_analog_in.h`, `hal/i_pulse_counter.h`,
  `hal/i_persistent_store.h`, `sim/sim_digital_in.h`, `sim/sim_analog_in.h`,
  `sim/sim_pulse_counter.h`, `sim/sim_pulse_counter.cpp`, `sim/sim_store.h`,
  `sim/sim_store.cpp`, `tests/test_sim_hal.cpp`
- Modifier : `sim/CMakeLists.txt`, `tests/CMakeLists.txt`

**Produit :**
- `IDigitalIn` — `bool read() const`
- `IAnalogIn` — `uint16_t readRaw() const`
- `IPulseCounter` — `uint32_t count() const`, `uint32_t lastPeriodUs() const`
- `IPersistentStore` — `bool read(uint16_t, uint8_t*, size_t) const`,
  `bool write(uint16_t, const uint8_t*, size_t)`
- `SimDigitalIn` — plus `void set(bool)`
- `SimAnalogIn` — plus `void set(uint16_t)`
- `SimPulseCounter` — plus `void advance(uint32_t dt_ms, float hz)`
- `SimStore` — `explicit SimStore(size_t)`, plus `uint32_t writeCount() const`

- [ ] **Étape 1 : écrire les tests qui échouent**

`tests/test_sim_hal.cpp` :

```cpp
#include <catch2/catch_test_macros.hpp>

#include <cstdint>

#include "sim/sim_analog_in.h"
#include "sim/sim_digital_in.h"
#include "sim/sim_pulse_counter.h"
#include "sim/sim_store.h"

using namespace rondo;

TEST_CASE("l'entree numerique simulee restitue le niveau impose") {
    SimDigitalIn pin;
    REQUIRE_FALSE(pin.read());
    pin.set(true);
    REQUIRE(pin.read());
}

TEST_CASE("l'entree analogique simulee restitue la valeur brute imposee") {
    SimAnalogIn input;
    REQUIRE(input.readRaw() == 0u);
    input.set(2048);
    REQUIRE(input.readRaw() == 2048u);
}

TEST_CASE("le compteur d'impulsions simule compte selon la frequence et la duree") {
    SimPulseCounter counter;
    counter.advance(1000, 10.0f);
    REQUIRE(counter.count() == 10u);
    REQUIRE(counter.lastPeriodUs() == 100000u);
}

TEST_CASE("le compteur d'impulsions cumule les fractions entre deux appels") {
    SimPulseCounter counter;
    counter.advance(500, 1.0f);
    REQUIRE(counter.count() == 0u);
    counter.advance(500, 1.0f);
    REQUIRE(counter.count() == 1u);
}

TEST_CASE("une frequence nulle ne produit aucune impulsion et annule la periode") {
    SimPulseCounter counter;
    counter.advance(1000, 10.0f);
    counter.advance(1000, 0.0f);
    REQUIRE(counter.count() == 10u);
    REQUIRE(counter.lastPeriodUs() == 0u);
}

TEST_CASE("le stockage simule relit ce qui a ete ecrit") {
    SimStore store(64);
    const uint8_t written[4] = {0xDE, 0xAD, 0xBE, 0xEF};
    REQUIRE(store.write(8, written, 4));

    uint8_t readback[4] = {};
    REQUIRE(store.read(8, readback, 4));
    REQUIRE(readback[0] == 0xDE);
    REQUIRE(readback[3] == 0xEF);
}

TEST_CASE("le stockage simule refuse un acces hors limites") {
    SimStore store(64);
    const uint8_t written[4] = {1, 2, 3, 4};
    REQUIRE_FALSE(store.write(62, written, 4));

    uint8_t readback[4] = {};
    REQUIRE_FALSE(store.read(62, readback, 4));
}

TEST_CASE("le stockage simule compte les ecritures pour surveiller l'usure") {
    SimStore store(64);
    const uint8_t written[2] = {1, 2};
    store.write(0, written, 2);
    store.write(0, written, 2);
    REQUIRE(store.writeCount() == 2u);
}
```

- [ ] **Étape 2 : lancer les tests et vérifier qu'ils échouent**

```bash
cmake --build build -j
```

Attendu : **échec**, `sim/sim_digital_in.h: No such file or directory`.

- [ ] **Étape 3 : écrire les interfaces matérielles**

`hal/i_digital_in.h` :

```cpp
#pragma once

namespace rondo {

// Entree tout ou rien, telle qu'elle est electriquement.
// L'inversion logique des entrees actives-basses (opto-coupleurs, contacteur
// de point mort) est le role du fournisseur, pas celui de cette interface.
class IDigitalIn {
 public:
    virtual ~IDigitalIn() = default;
    virtual bool read() const = 0;  // true = niveau electrique haut
};

}  // namespace rondo
```

`hal/i_analog_in.h` :

```cpp
#pragma once

#include <cstdint>

namespace rondo {

// Entree analogique brute, sans mise a l'echelle.
// La conversion en grandeur physique depend du pont diviseur et de la courbe
// du capteur : elle appartient au profil moto, jamais a cette interface.
class IAnalogIn {
 public:
    virtual ~IAnalogIn() = default;
    virtual uint16_t readRaw() const = 0;  // 0 a 4095 sur un ADC 12 bits
};

}  // namespace rondo
```

`hal/i_pulse_counter.h` :

```cpp
#pragma once

#include <cstdint>

namespace rondo {

// Compteur materiel d'impulsions, alimente par le capteur a effet Hall.
// Le comptage est cumulatif pour que la distance parcourue ne puisse pas
// etre perdue entre deux scrutations.
class IPulseCounter {
 public:
    virtual ~IPulseCounter() = default;

    // Nombre total d'impulsions depuis le demarrage. Monotone, reboucle a
    // 2^32 : toujours comparer par difference, jamais par valeur absolue.
    virtual uint32_t count() const = 0;

    // Duree entre les deux dernieres impulsions, 0 si moins de deux recues.
    virtual uint32_t lastPeriodUs() const = 0;
};

}  // namespace rondo
```

`hal/i_persistent_store.h` :

```cpp
#pragma once

#include <cstddef>
#include <cstdint>

namespace rondo {

// Memoire persistante adressable, realisee par une FRAM sur la cible.
// L'odometre y est ecrit en continu tous les 10 metres : l'implementation
// doit supporter des ecritures tres frequentes sans usure.
class IPersistentStore {
 public:
    virtual ~IPersistentStore() = default;

    // Renvoient false si l'acces sort des limites. Aucune exception.
    virtual bool read(uint16_t addr, uint8_t* dst, size_t len) const = 0;
    virtual bool write(uint16_t addr, const uint8_t* src, size_t len) = 0;
};

}  // namespace rondo
```

- [ ] **Étape 4 : écrire les implémentations de simulation**

`sim/sim_digital_in.h` :

```cpp
#pragma once

#include "hal/i_digital_in.h"

namespace rondo {

class SimDigitalIn final : public IDigitalIn {
 public:
    bool read() const override { return level_; }
    void set(bool level) { level_ = level; }

 private:
    bool level_ = false;
};

}  // namespace rondo
```

`sim/sim_analog_in.h` :

```cpp
#pragma once

#include <cstdint>

#include "hal/i_analog_in.h"

namespace rondo {

class SimAnalogIn final : public IAnalogIn {
 public:
    uint16_t readRaw() const override { return raw_; }
    void set(uint16_t raw) { raw_ = raw; }

 private:
    uint16_t raw_ = 0;
};

}  // namespace rondo
```

`sim/sim_pulse_counter.h` :

```cpp
#pragma once

#include <cstdint>

#include "hal/i_pulse_counter.h"

namespace rondo {

class SimPulseCounter final : public IPulseCounter {
 public:
    uint32_t count() const override { return count_; }
    uint32_t lastPeriodUs() const override { return period_us_; }

    // Fait comme si des impulsions arrivaient a `hz` pendant `dt_ms`.
    // Les fractions d'impulsion sont conservees d'un appel a l'autre : une
    // suite d'appels courts compte exactement comme un appel long.
    void advance(uint32_t dt_ms, float hz);

 private:
    uint32_t count_ = 0;
    uint32_t period_us_ = 0;
    float accumulator_ = 0.0f;
};

}  // namespace rondo
```

`sim/sim_pulse_counter.cpp` :

```cpp
#include "sim/sim_pulse_counter.h"

namespace rondo {

void SimPulseCounter::advance(uint32_t dt_ms, float hz) {
    if (hz <= 0.0f) {
        period_us_ = 0;
        return;
    }
    accumulator_ += hz * static_cast<float>(dt_ms) / 1000.0f;
    const uint32_t whole = static_cast<uint32_t>(accumulator_);
    count_ += whole;
    accumulator_ -= static_cast<float>(whole);
    period_us_ = static_cast<uint32_t>(1000000.0f / hz);
}

}  // namespace rondo
```

`sim/sim_store.h` :

```cpp
#pragma once

#include <cstddef>
#include <cstdint>
#include <vector>

#include "hal/i_persistent_store.h"

namespace rondo {

// Memoire persistante en RAM. Le contenu survit a tout dans le simulateur :
// c'est voulu, une FRAM ne perd rien a la coupure.
class SimStore final : public IPersistentStore {
 public:
    explicit SimStore(size_t size);

    bool read(uint16_t addr, uint8_t* dst, size_t len) const override;
    bool write(uint16_t addr, const uint8_t* src, size_t len) override;

    // Nombre d'ecritures reussies, pour verifier en test que la cadence
    // d'ecriture de l'odometre reste dans les limites annoncees.
    uint32_t writeCount() const { return writes_; }

 private:
    std::vector<uint8_t> memory_;
    uint32_t writes_ = 0;
};

}  // namespace rondo
```

`sim/sim_store.cpp` :

```cpp
#include "sim/sim_store.h"

#include <algorithm>

namespace rondo {

SimStore::SimStore(size_t size) : memory_(size, 0) {}

bool SimStore::read(uint16_t addr, uint8_t* dst, size_t len) const {
    if (dst == nullptr || static_cast<size_t>(addr) + len > memory_.size()) {
        return false;
    }
    std::copy(memory_.begin() + addr, memory_.begin() + addr + len, dst);
    return true;
}

bool SimStore::write(uint16_t addr, const uint8_t* src, size_t len) {
    if (src == nullptr || static_cast<size_t>(addr) + len > memory_.size()) {
        return false;
    }
    std::copy(src, src + len, memory_.begin() + addr);
    ++writes_;
    return true;
}

}  // namespace rondo
```

`sim/CMakeLists.txt` devient :

```cmake
add_library(rondo_sim
  sim_pulse_counter.cpp
  sim_store.cpp
)
target_include_directories(rondo_sim PUBLIC ${PROJECT_SOURCE_DIR})
target_link_libraries(rondo_sim PUBLIC rondo_core)
```

Ajouter `test_sim_hal.cpp` aux sources de `rondo_tests`.

- [ ] **Étape 5 : lancer les tests et vérifier qu'ils passent**

```bash
cmake -B build -DCMAKE_BUILD_TYPE=Debug && cmake --build build -j && ctest --test-dir build --output-on-failure
```

Attendu : `100% tests passed, 0 tests failed out of 25`.

- [ ] **Étape 6 : commit**

```bash
git add hal sim tests
git commit -m "hal: add hardware interfaces and their simulated implementations"
```

---

## Tâche 6 : Contrat de fournisseur et tranche verticale

Cette tâche relie pour la première fois la couche matérielle au bus de canaux. Elle existe
pour prouver que les interfaces des tâches 4 et 5 tiennent réellement ensemble — une
interface sans consommateur est une interface fausse.

**Fichiers :**
- Créer : `core/provider.h`, `core/providers/digital_input_provider.h`,
  `core/providers/digital_input_provider.cpp`, `tests/test_digital_input_provider.cpp`
- Modifier : `core/CMakeLists.txt`, `tests/CMakeLists.txt`

**Consomme :** `ChannelBus` (tâche 4), `IDigitalIn` (tâche 5).
**Produit :**
- `IProvider` — `void declareChannels(ChannelBus&)`, `void poll(ChannelBus&)`
- `struct DigitalInputConfig { ChannelId channel; bool active_low; uint32_t ttl_ms; }`
- `class DigitalInputProvider` — `DigitalInputProvider(const IDigitalIn&, DigitalInputConfig)`

- [ ] **Étape 1 : écrire les tests qui échouent**

`tests/test_digital_input_provider.cpp` :

```cpp
#include <catch2/catch_test_macros.hpp>

#include "core/channel_bus.h"
#include "core/providers/digital_input_provider.h"
#include "sim/sim_clock.h"
#include "sim/sim_digital_in.h"

using namespace rondo;

TEST_CASE("le fournisseur declare son canal, qui cesse d'etre absent") {
    SimClock clock;
    ChannelBus bus(clock);
    SimDigitalIn pin;
    DigitalInputProvider provider(pin, {ChannelId::Neutral, true, 1000});

    REQUIRE(bus.read(ChannelId::Neutral).validity == Validity::Absent);
    provider.declareChannels(bus);
    REQUIRE(bus.read(ChannelId::Neutral).validity == Validity::Stale);
}

TEST_CASE("une entree active-basse est inversee") {
    SimClock clock;
    ChannelBus bus(clock);
    SimDigitalIn pin;
    DigitalInputProvider provider(pin, {ChannelId::Neutral, true, 1000});
    provider.declareChannels(bus);

    pin.set(false);  // contacteur ferme, point mort engage
    provider.poll(bus);
    REQUIRE(bus.read(ChannelId::Neutral).value == 1.0f);

    pin.set(true);  // contacteur ouvert, une vitesse est engagee
    provider.poll(bus);
    REQUIRE(bus.read(ChannelId::Neutral).value == 0.0f);
}

TEST_CASE("une entree active-haute est transmise telle quelle") {
    SimClock clock;
    ChannelBus bus(clock);
    SimDigitalIn pin;
    DigitalInputProvider provider(pin, {ChannelId::HighBeam, false, 1000});
    provider.declareChannels(bus);

    pin.set(true);
    provider.poll(bus);
    REQUIRE(bus.read(ChannelId::HighBeam).value == 1.0f);
}

TEST_CASE("le canal du fournisseur devient perime s'il cesse de scruter") {
    SimClock clock;
    ChannelBus bus(clock);
    SimDigitalIn pin;
    DigitalInputProvider provider(pin, {ChannelId::Neutral, true, 1000});
    provider.declareChannels(bus);
    provider.poll(bus);
    REQUIRE(bus.read(ChannelId::Neutral).validity == Validity::Fresh);

    clock.advanceMs(1001);
    REQUIRE(bus.read(ChannelId::Neutral).validity == Validity::Stale);
}
```

- [ ] **Étape 2 : lancer les tests et vérifier qu'ils échouent**

```bash
cmake --build build -j
```

Attendu : **échec**, `core/providers/digital_input_provider.h: No such file or directory`.

- [ ] **Étape 3 : écrire l'implémentation minimale**

`core/provider.h` :

```cpp
#pragma once

namespace rondo {

class ChannelBus;

// Contrat commun a toute source de donnees.
// Un fournisseur ne connait que son materiel et le bus : il ignore
// totalement qui consomme ses valeurs.
class IProvider {
 public:
    virtual ~IProvider() = default;

    // Appelee une fois au demarrage, apres le chargement du profil moto.
    virtual void declareChannels(ChannelBus& bus) = 0;

    // Appelee par la tache d'acquisition. Doit rester courte et ne jamais
    // bloquer : elle s'execute a plus haute priorite que l'interface.
    virtual void poll(ChannelBus& bus) = 0;
};

}  // namespace rondo
```

`core/providers/digital_input_provider.h` :

```cpp
#pragma once

#include <cstdint>

#include "core/channel_id.h"
#include "core/provider.h"
#include "hal/i_digital_in.h"

namespace rondo {

class ChannelBus;

struct DigitalInputConfig {
    ChannelId channel;
    // true pour les entrees isolees par opto-coupleur et pour le contacteur
    // de point mort : signal actif = niveau electrique BAS.
    bool active_low;
    uint32_t ttl_ms;
};

// Publie une entree tout ou rien sur un canal booleen (0.0 ou 1.0).
// Sert les clignotants, le plein phare, le point mort et les boutons.
class DigitalInputProvider final : public IProvider {
 public:
    DigitalInputProvider(const IDigitalIn& pin, DigitalInputConfig config);

    void declareChannels(ChannelBus& bus) override;
    void poll(ChannelBus& bus) override;

 private:
    const IDigitalIn& pin_;
    DigitalInputConfig config_;
};

}  // namespace rondo
```

`core/providers/digital_input_provider.cpp` :

```cpp
#include "core/providers/digital_input_provider.h"

#include "core/channel_bus.h"

namespace rondo {

DigitalInputProvider::DigitalInputProvider(const IDigitalIn& pin, DigitalInputConfig config)
    : pin_(pin), config_(config) {}

void DigitalInputProvider::declareChannels(ChannelBus& bus) {
    bus.declare(config_.channel, config_.ttl_ms);
}

void DigitalInputProvider::poll(ChannelBus& bus) {
    const bool level = pin_.read();
    const bool active = config_.active_low ? !level : level;
    bus.publish(config_.channel, active ? 1.0f : 0.0f);
}

}  // namespace rondo
```

Ajouter `providers/digital_input_provider.cpp` aux sources de `rondo_core` et
`test_digital_input_provider.cpp` à celles de `rondo_tests`.

- [ ] **Étape 4 : lancer les tests et vérifier qu'ils passent**

```bash
cmake -B build -DCMAKE_BUILD_TYPE=Debug && cmake --build build -j && ctest --test-dir build --output-on-failure
```

Attendu : `100% tests passed, 0 tests failed out of 29`.

- [ ] **Étape 5 : commit**

```bash
git add core tests
git commit -m "core: add the provider contract and a digital input provider"
```

---

## Tâche 7 : Scénarios et rejeu

Le harnais qui permettra de rejouer une situation de conduite — montée en vitesse, glitch
de capteur, perte de signal — de façon reproductible, sans moto et sans attente.

**Fichiers :**
- Créer : `sim/scenario.h`, `sim/scenario.cpp`, `sim/scenario_player.h`,
  `sim/scenario_player.cpp`, `sim/scenarios/neutral-toggle.csv`,
  `tests/test_scenario.cpp`, `tests/test_scenario_player.cpp`
- Modifier : `sim/CMakeLists.txt`, `tests/CMakeLists.txt`

**Consomme :** `SimClock` (tâche 2).
**Produit :**
- `struct ScenarioEvent { uint32_t t_ms; std::string signal; float value; }`
- `class Scenario` — `static Scenario parse(std::istream&)`,
  `const std::vector<ScenarioEvent>& events() const`
- `class ScenarioPlayer` — `ScenarioPlayer(Scenario, SimClock&)`,
  `void bind(const std::string&, std::function<void(float)>)`,
  `void advanceTo(uint32_t t_ms)`, `bool finished() const`

**Format de fichier :** CSV à en-tête obligatoire `t_ms,signal,value`. Les lignes vides et
celles commençant par `#` sont ignorées. Les horodatages doivent être **croissants ou
égaux**. Le nom de signal est libre : c'est la clé passée à `bind()`.

- [ ] **Étape 1 : écrire les tests d'analyse qui échouent**

`tests/test_scenario.cpp` :

```cpp
#include <catch2/catch_test_macros.hpp>

#include <sstream>
#include <string>

#include "sim/scenario.h"

using namespace rondo;

TEST_CASE("un scenario bien forme est analyse") {
    std::istringstream in(
        "t_ms,signal,value\n"
        "0,neutral,1\n"
        "1500,neutral,0\n"
        "1500,turn_left,1\n");

    const Scenario s = Scenario::parse(in);
    REQUIRE(s.events().size() == 3u);
    REQUIRE(s.events()[0].t_ms == 0u);
    REQUIRE(s.events()[0].signal == "neutral");
    REQUIRE(s.events()[0].value == 1.0f);
    REQUIRE(s.events()[2].signal == "turn_left");
}

TEST_CASE("les commentaires et les lignes vides sont ignores") {
    std::istringstream in(
        "t_ms,signal,value\n"
        "# demarrage moteur\n"
        "\n"
        "0,neutral,1\n");

    REQUIRE(Scenario::parse(in).events().size() == 1u);
}

TEST_CASE("un en-tete absent est refuse") {
    std::istringstream in("0,neutral,1\n");
    REQUIRE_THROWS_AS(Scenario::parse(in), std::runtime_error);
}

TEST_CASE("des horodatages decroissants sont refuses") {
    std::istringstream in(
        "t_ms,signal,value\n"
        "1500,neutral,1\n"
        "500,neutral,0\n");

    REQUIRE_THROWS_AS(Scenario::parse(in), std::runtime_error);
}

TEST_CASE("une ligne malformee est refusee en citant son numero") {
    std::istringstream in(
        "t_ms,signal,value\n"
        "0,neutral,1\n"
        "oops\n");

    try {
        Scenario::parse(in);
        FAIL("l'analyse aurait du echouer");
    } catch (const std::runtime_error& e) {
        REQUIRE(std::string(e.what()).find("3") != std::string::npos);
    }
}
```

- [ ] **Étape 2 : lancer les tests et vérifier qu'ils échouent**

```bash
cmake --build build -j
```

Attendu : **échec**, `sim/scenario.h: No such file or directory`.

- [ ] **Étape 3 : écrire l'analyseur**

`sim/scenario.h` :

```cpp
#pragma once

#include <cstdint>
#include <istream>
#include <string>
#include <vector>

namespace rondo {

struct ScenarioEvent {
    uint32_t t_ms = 0;
    std::string signal;
    float value = 0.0f;
};

// Suite d'evenements horodates rejouee sur les entrees simulees.
// Format CSV : en-tete "t_ms,signal,value", horodatages non decroissants,
// lignes vides et lignes commencant par '#' ignorees.
class Scenario {
 public:
    // Leve std::runtime_error sur un fichier malforme, en citant la ligne.
    static Scenario parse(std::istream& in);

    const std::vector<ScenarioEvent>& events() const { return events_; }

 private:
    std::vector<ScenarioEvent> events_;
};

}  // namespace rondo
```

`sim/scenario.cpp` :

```cpp
#include "sim/scenario.h"

#include <sstream>
#include <stdexcept>

namespace rondo {
namespace {

std::string trim(const std::string& s) {
    const size_t first = s.find_first_not_of(" \t\r\n");
    if (first == std::string::npos) {
        return "";
    }
    const size_t last = s.find_last_not_of(" \t\r\n");
    return s.substr(first, last - first + 1);
}

[[noreturn]] void fail(size_t line_number, const std::string& reason) {
    throw std::runtime_error("scenario ligne " + std::to_string(line_number) + " : " + reason);
}

}  // namespace

Scenario Scenario::parse(std::istream& in) {
    Scenario scenario;
    std::string raw;
    size_t line_number = 0;
    bool header_seen = false;
    uint32_t previous_t_ms = 0;

    while (std::getline(in, raw)) {
        ++line_number;
        const std::string line = trim(raw);
        if (line.empty() || line[0] == '#') {
            continue;
        }

        if (!header_seen) {
            if (line != "t_ms,signal,value") {
                fail(line_number, "en-tete attendu \"t_ms,signal,value\"");
            }
            header_seen = true;
            continue;
        }

        std::istringstream fields(line);
        std::string t_field;
        std::string signal;
        std::string value_field;
        if (!std::getline(fields, t_field, ',') || !std::getline(fields, signal, ',') ||
            !std::getline(fields, value_field, ',')) {
            fail(line_number, "trois champs attendus");
        }

        ScenarioEvent event;
        try {
            event.t_ms = static_cast<uint32_t>(std::stoul(trim(t_field)));
            event.value = std::stof(trim(value_field));
        } catch (const std::exception&) {
            fail(line_number, "horodatage ou valeur illisible");
        }
        event.signal = trim(signal);
        if (event.signal.empty()) {
            fail(line_number, "nom de signal vide");
        }
        if (!scenario.events_.empty() && event.t_ms < previous_t_ms) {
            fail(line_number, "horodatage en recul");
        }

        previous_t_ms = event.t_ms;
        scenario.events_.push_back(event);
    }

    if (!header_seen) {
        fail(line_number + 1, "en-tete attendu \"t_ms,signal,value\"");
    }
    return scenario;
}

}  // namespace rondo
```

Ajouter `scenario.cpp` aux sources de `rondo_sim` et `test_scenario.cpp` à celles de
`rondo_tests`.

- [ ] **Étape 4 : lancer les tests et vérifier qu'ils passent**

```bash
cmake -B build -DCMAKE_BUILD_TYPE=Debug && cmake --build build -j && ctest --test-dir build --output-on-failure
```

Attendu : `100% tests passed, 0 tests failed out of 34`.

- [ ] **Étape 5 : écrire les tests de rejeu qui échouent**

`tests/test_scenario_player.cpp` :

```cpp
#include <catch2/catch_test_macros.hpp>

#include <sstream>
#include <stdexcept>

#include "sim/scenario.h"
#include "sim/scenario_player.h"
#include "sim/sim_clock.h"
#include "sim/sim_digital_in.h"

using namespace rondo;

namespace {

Scenario makeScenario() {
    std::istringstream in(
        "t_ms,signal,value\n"
        "0,neutral,1\n"
        "1500,neutral,0\n");
    return Scenario::parse(in);
}

}  // namespace

TEST_CASE("le rejeu applique les evenements dus et pas les suivants") {
    SimClock clock;
    SimDigitalIn pin;
    ScenarioPlayer player(makeScenario(), clock);
    player.bind("neutral", [&pin](float v) { pin.set(v > 0.5f); });

    player.advanceTo(0);
    REQUIRE(pin.read());

    player.advanceTo(1000);
    REQUIRE(pin.read());

    player.advanceTo(1500);
    REQUIRE_FALSE(pin.read());
}

TEST_CASE("le rejeu positionne l'horloge sur le temps demande") {
    SimClock clock;
    SimDigitalIn pin;
    ScenarioPlayer player(makeScenario(), clock);
    player.bind("neutral", [&pin](float v) { pin.set(v > 0.5f); });

    player.advanceTo(1200);
    REQUIRE(clock.nowMs() == 1200u);
}

TEST_CASE("le rejeu signale un signal non lie") {
    SimClock clock;
    ScenarioPlayer player(makeScenario(), clock);
    REQUIRE_THROWS_AS(player.advanceTo(0), std::runtime_error);
}

TEST_CASE("le rejeu se declare termine une fois le dernier evenement applique") {
    SimClock clock;
    SimDigitalIn pin;
    ScenarioPlayer player(makeScenario(), clock);
    player.bind("neutral", [&pin](float v) { pin.set(v > 0.5f); });

    REQUIRE_FALSE(player.finished());
    player.advanceTo(2000);
    REQUIRE(player.finished());
}
```

- [ ] **Étape 6 : lancer les tests et vérifier qu'ils échouent**

```bash
cmake --build build -j
```

Attendu : **échec**, `sim/scenario_player.h: No such file or directory`.

- [ ] **Étape 7 : écrire le lecteur de scénario**

`sim/scenario_player.h` :

```cpp
#pragma once

#include <cstddef>
#include <cstdint>
#include <functional>
#include <map>
#include <string>

#include "sim/scenario.h"
#include "sim/sim_clock.h"

namespace rondo {

// Rejoue un scenario en pilotant les entrees simulees et l'horloge.
class ScenarioPlayer {
 public:
    ScenarioPlayer(Scenario scenario, SimClock& clock);

    // Associe un nom de signal au reglage d'une entree simulee.
    void bind(const std::string& signal, std::function<void(float)> setter);

    // Avance le temps simule jusqu'a t_ms en appliquant tout evenement du.
    // L'horloge est positionnee sur l'instant de chaque evenement avant de
    // l'appliquer, pour que les valeurs publiees portent le bon horodatage.
    // Leve std::runtime_error si un evenement designe un signal non lie.
    void advanceTo(uint32_t t_ms);

    bool finished() const { return next_ >= scenario_.events().size(); }

 private:
    Scenario scenario_;
    SimClock& clock_;
    std::map<std::string, std::function<void(float)>> bindings_;
    size_t next_ = 0;
};

}  // namespace rondo
```

`sim/scenario_player.cpp` :

```cpp
#include "sim/scenario_player.h"

#include <stdexcept>
#include <utility>

namespace rondo {

ScenarioPlayer::ScenarioPlayer(Scenario scenario, SimClock& clock)
    : scenario_(std::move(scenario)), clock_(clock) {}

void ScenarioPlayer::bind(const std::string& signal, std::function<void(float)> setter) {
    bindings_[signal] = std::move(setter);
}

void ScenarioPlayer::advanceTo(uint32_t t_ms) {
    while (next_ < scenario_.events().size() && scenario_.events()[next_].t_ms <= t_ms) {
        const ScenarioEvent& event = scenario_.events()[next_];
        clock_.setMs(event.t_ms);

        const auto it = bindings_.find(event.signal);
        if (it == bindings_.end()) {
            throw std::runtime_error("signal non lie dans le scenario : " + event.signal);
        }
        it->second(event.value);
        ++next_;
    }
    clock_.setMs(t_ms);
}

}  // namespace rondo
```

`sim/scenarios/neutral-toggle.csv` :

```csv
t_ms,signal,value
# Point mort engage au demarrage, premiere puis seconde engagees
0,neutral,1
3000,neutral,0
8000,neutral,1
```

Ajouter `scenario_player.cpp` aux sources de `rondo_sim` et `test_scenario_player.cpp` à
celles de `rondo_tests`.

- [ ] **Étape 8 : lancer les tests et vérifier qu'ils passent**

```bash
cmake -B build -DCMAKE_BUILD_TYPE=Debug && cmake --build build -j && ctest --test-dir build --output-on-failure
```

Attendu : `100% tests passed, 0 tests failed out of 38`.

- [ ] **Étape 9 : commit**

```bash
git add sim tests
git commit -m "sim: add scenario parsing and replay onto simulated inputs"
```

---

## Tâche 8 : Exécutable de simulation

Assemble tout le socle en un programme qui tourne réellement. C'est le point d'entrée sur
lequel `rondo-ui` viendra brancher l'affichage LVGL.

**Fichiers :**
- Créer : `sim/main.cpp`, `tests/test_end_to_end.cpp`, `docs/simulator.md`
- Modifier : `sim/CMakeLists.txt`, `tests/CMakeLists.txt`

**Consomme :** tout ce qui précède.
**Produit :** l'exécutable `rondo_sim_run`.

**Interface en ligne de commande :**

```
rondo_sim_run <scenario.csv> [--step-ms N] [--until-ms N]
    --step-ms   pas de simulation, defaut 100
    --until-ms  instant final, defaut : dernier evenement du scenario + 1000
```

**Sortie sur la sortie standard**, en CSV : `t_ms,channel,value,validity`, une ligne par
canal non absent à chaque pas.

- [ ] **Étape 1 : écrire le test de bout en bout**

Créer le fichier ci-dessous **et l'ajouter immédiatement aux sources de `rondo_tests`**
dans `tests/CMakeLists.txt` — sans quoi l'étape 2 ne compilerait pas le test et passerait
pour une fausse raison.

`tests/test_end_to_end.cpp` :

```cpp
#include <catch2/catch_test_macros.hpp>

#include <sstream>

#include "core/channel_bus.h"
#include "core/providers/digital_input_provider.h"
#include "sim/scenario.h"
#include "sim/scenario_player.h"
#include "sim/sim_clock.h"
#include "sim/sim_digital_in.h"

using namespace rondo;

TEST_CASE("un scenario pilote le bus de canaux de bout en bout") {
    std::istringstream in(
        "t_ms,signal,value\n"
        "0,neutral_pin,0\n"
        "3000,neutral_pin,1\n");

    SimClock clock;
    ChannelBus bus(clock);
    SimDigitalIn pin;
    DigitalInputProvider provider(pin, {ChannelId::Neutral, true, 500});
    provider.declareChannels(bus);

    ScenarioPlayer player(Scenario::parse(in), clock);
    player.bind("neutral_pin", [&pin](float v) { pin.set(v > 0.5f); });

    // Contacteur ferme : point mort engage.
    player.advanceTo(0);
    provider.poll(bus);
    REQUIRE(bus.read(ChannelId::Neutral).value == 1.0f);
    REQUIRE(bus.read(ChannelId::Neutral).validity == Validity::Fresh);

    // Contacteur ouvert : une vitesse est engagee.
    player.advanceTo(3000);
    provider.poll(bus);
    REQUIRE(bus.read(ChannelId::Neutral).value == 0.0f);

    // Sans scrutation, le canal se perime tout en gardant sa valeur.
    player.advanceTo(3600);
    const Reading r = bus.read(ChannelId::Neutral);
    REQUIRE(r.validity == Validity::Stale);
    REQUIRE(r.value == 0.0f);

    // Un canal qu'aucun fournisseur n'alimente reste absent : c'est ce qui
    // permet a un theme de se masquer tout seul sur une moto sans jauge.
    REQUIRE(bus.read(ChannelId::FuelLevel).validity == Validity::Absent);
}
```

- [ ] **Étape 2 : lancer le test et vérifier qu'il passe**

```bash
cmake --build build -j && ctest --test-dir build --output-on-failure
```

Ce test doit passer **immédiatement** : il n'introduit aucun code nouveau, il vérifie que
les briques des tâches 4 à 7 s'assemblent. S'il échoue, c'est qu'une interface antérieure
est fausse — corriger la tâche fautive avant de continuer, pas ce test.

Attendu : `100% tests passed, 0 tests failed out of 39`.

- [ ] **Étape 3 : écrire l'exécutable**

`sim/main.cpp` :

```cpp
#include <cstddef>
#include <cstdint>
#include <cstdlib>
#include <cstring>
#include <fstream>
#include <iostream>
#include <string>
#include <utility>

#include "core/channel_bus.h"
#include "core/providers/digital_input_provider.h"
#include "sim/scenario.h"
#include "sim/scenario_player.h"
#include "sim/sim_clock.h"
#include "sim/sim_digital_in.h"

using namespace rondo;

namespace {

const char* validityName(Validity v) {
    switch (v) {
        case Validity::Absent: return "absent";
        case Validity::Stale:  return "stale";
        case Validity::Fresh:  return "fresh";
    }
    return "absent";
}

void dump(const ChannelBus& bus, uint32_t t_ms) {
    for (size_t i = 0; i < static_cast<size_t>(ChannelId::Count); ++i) {
        const ChannelId id = static_cast<ChannelId>(i);
        const Reading r = bus.read(id);
        if (r.validity == Validity::Absent) {
            continue;
        }
        std::cout << t_ms << ',' << toString(id) << ',' << r.value << ','
                  << validityName(r.validity) << '\n';
    }
}

}  // namespace

int main(int argc, char** argv) {
    if (argc < 2) {
        std::cerr << "usage: rondo_sim_run <scenario.csv> [--step-ms N] [--until-ms N]\n";
        return 2;
    }

    uint32_t step_ms = 100;
    uint32_t until_ms = 0;
    bool until_given = false;
    for (int i = 2; i + 1 < argc; i += 2) {
        if (std::strcmp(argv[i], "--step-ms") == 0) {
            step_ms = static_cast<uint32_t>(std::strtoul(argv[i + 1], nullptr, 10));
        } else if (std::strcmp(argv[i], "--until-ms") == 0) {
            until_ms = static_cast<uint32_t>(std::strtoul(argv[i + 1], nullptr, 10));
            until_given = true;
        }
    }
    if (step_ms == 0) {
        std::cerr << "--step-ms doit etre superieur a zero\n";
        return 2;
    }

    std::ifstream file(argv[1]);
    if (!file) {
        std::cerr << "scenario illisible : " << argv[1] << '\n';
        return 1;
    }

    try {
        Scenario scenario = Scenario::parse(file);
        if (!until_given) {
            until_ms = scenario.events().empty() ? 1000 : scenario.events().back().t_ms + 1000;
        }

        SimClock clock;
        ChannelBus bus(clock);
        SimDigitalIn neutral_pin;
        DigitalInputProvider neutral(neutral_pin, {ChannelId::Neutral, true, 500});
        neutral.declareChannels(bus);

        ScenarioPlayer player(std::move(scenario), clock);
        player.bind("neutral_pin", [&neutral_pin](float v) { neutral_pin.set(v > 0.5f); });

        std::cout << "t_ms,channel,value,validity\n";
        for (uint32_t t = 0; t <= until_ms; t += step_ms) {
            player.advanceTo(t);
            neutral.poll(bus);
            dump(bus, t);
        }
    } catch (const std::exception& e) {
        std::cerr << "erreur : " << e.what() << '\n';
        return 1;
    }
    return 0;
}
```

Dans `sim/CMakeLists.txt`, ajouter après la bibliothèque :

```cmake
add_executable(rondo_sim_run main.cpp)
target_link_libraries(rondo_sim_run PRIVATE rondo_sim)
```

- [ ] **Étape 4 : vérifier l'exécutable à la main**

```bash
cmake -B build -DCMAKE_BUILD_TYPE=Debug && cmake --build build -j
./build/sim/rondo_sim_run sim/scenarios/neutral-toggle.csv --step-ms 1000
```

Attendu : un en-tête `t_ms,channel,value,validity` puis une ligne `neutral` par pas, la
valeur passant de `1` à `0` à `t=3000` et revenant à `1` à `t=8000`, la validité restant
`fresh` puisque le fournisseur est scruté à chaque pas.

Vérifier aussi les deux cas d'erreur :

```bash
./build/sim/rondo_sim_run does-not-exist.csv ; echo "code retour: $?"
```

Attendu : `scenario illisible` et un code retour de `1`.

```bash
./build/sim/rondo_sim_run ; echo "code retour: $?"
```

Attendu : la ligne d'usage et un code retour de `2`.

- [ ] **Étape 5 : écrire `docs/simulator.md`**

Doit contenir : à quoi sert le simulateur, comment compiler, comment lancer un scénario,
le format des fichiers de scénario avec un exemple commenté, le format de sortie, et la
liste des signaux liés par `main.cpp`. Préciser que l'affichage graphique n'est pas dans ce
plan mais dans `rondo-ui`, et que les scénarios écrits ici resteront valables.

- [ ] **Étape 6 : lancer la totalité des tests**

```bash
ctest --test-dir build --output-on-failure
```

Attendu : `100% tests passed, 0 tests failed out of 39`.

- [ ] **Étape 7 : commit**

```bash
git add sim tests docs/simulator.md
git commit -m "sim: add the headless simulation runner and its documentation"
```

---

## Suites

À la fin de ce plan, le socle est complet et vérifié par 39 tests s'exécutant en moins
d'une seconde, sans matériel.

| Plan | Contenu | Dépend de |
|---|---|---|
| `rondo-acquisition` | Implémentation ESP32 de la couche matérielle, fournisseurs Hall / analogique / thermistance, service d'odométrie et FRAM | ce plan + `rondo-hardware` |
| `rondo-ui` | LVGL, simulateur graphique SDL, bibliothèque de widgets, 3 archétypes, 6 thèmes | ce plan |
| `rondo-config` | Portail WiFi, profil moto JSON, import et export | `rondo-acquisition` |
| `rondo-nav` | GNSS, chargement et suivi de trace GPX | `rondo-config` |

`rondo-ui` ne dépend que de ce plan : il peut démarrer immédiatement après, en parallèle de
`rondo-acquisition`, puisque les thèmes ne lisent que le bus de canaux.
