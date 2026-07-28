# Rondo — Plan d'implémentation : profil moto et portail de configuration

> **Pour les agents :** SOUS-COMPÉTENCE REQUISE — utiliser `superpowers:subagent-driven-development`
> (recommandé) ou `superpowers:executing-plans` pour dérouler ce plan tâche par tâche.
> Les étapes utilisent la syntaxe case à cocher (`- [ ]`).

**Objectif :** faire passer Rondo de « le compteur d'une KDX » à « un compteur que
n'importe qui peut installer sur sa moto » — sans recompiler, sans savoir programmer.

**Architecture :** un profil moto décrit tout ce qui dépend de la machine : mappage des
broches, circonférence de roue, courbes de capteurs, seuils, thème. Il est décrit en JSON,
validé, stocké en NVS, et modifiable depuis un téléphone via un point d'accès WiFi porté
par l'ESP32. Le démarrage construit les fournisseurs **à partir du profil**, plus à partir
du code.

**Pile technique :** C++17, cJSON, ESP-IDF 5.2+ (`esp_http_server`, `esp_wifi`, NVS),
HTML et JavaScript sans dépendance, CMake, Catch2 v3.

## Contraintes globales

- **Aucune recompilation pour changer de moto.** Si une modification du profil impose de
  reflasher, c'est un défaut de conception, pas une limite acceptable.
- **Le profil est validé avant d'être appliqué.** Un profil invalide n'est jamais chargé :
  le système garde le précédent et signale l'erreur. Un compteur qui ne démarre plus après
  une faute de frappe serait pire que pas de portail du tout.
- **Un champ JSON inconnu est ignoré, pas rejeté.** Les profils partagés par la communauté
  survivront à des versions différentes du firmware.
- **Une entrée non câblée (`gpio: -1`) laisse son canal `Absent`**, ce qui masque
  automatiquement le widget correspondant. C'est le mécanisme central du multi-moto, déjà
  posé dans `rondo-core`.
- **Pas de WiFi en roulant.** Le portail ne s'ouvre qu'à l'arrêt, et se ferme tout seul.
- **Aucune allocation dynamique dans `core/`.** Le profil utilise des tableaux de taille
  fixe. La composition (`app/`) s'autorise une allocation unique au démarrage.
- **C++17**, `-Wall -Wextra -Wpedantic -Werror` hors code tiers.
- Identifiants et messages de commit en **anglais**, commentaires et tests en **français**.
- **Prérequis :** `rondo-core`, `rondo-ui` et `rondo-acquisition` terminés (113 tests).

## Structure des fichiers

| Fichier | Responsabilité |
|---|---|
| `core/profile.h` / `.cpp` | Structure du profil moto et valeurs par défaut |
| `core/profile_validation.h` / `.cpp` | Contrôles de cohérence |
| `core/profile_json.h` / `.cpp` | Lecture et écriture JSON |
| `core/profile_store.h` | `IProfileStore` |
| `sim/file_profile_store.h` / `.cpp` | Stockage fichier, pour l'hôte et les tests |
| `app/hal_factory.h` | `IHalFactory` — création des entrées décrites par le profil |
| `app/system_builder.h` / `.cpp` | Construit les fournisseurs à partir du profil |
| `sim/sim_hal_factory.h` / `.cpp` | Fabrique de simulation |
| `firmware/main/nvs_profile_store.*` | Stockage NVS |
| `firmware/main/esp32_hal_factory.*` | Fabrique ESP32 |
| `firmware/main/config_portal.*` | Point d'accès WiFi et serveur HTTP |
| `firmware/data/index.html` | Page de configuration, embarquée en LittleFS |
| `profiles/kawasaki-kdx125.json` | Profil de référence |
| `docs/profiles.md` | Format du profil et guide de contribution |
| `tests/test_*.cpp` | Un fichier par unité |

---

## Tâche 1 : Structure du profil

**Fichiers :**
- Créer : `core/profile.h`, `core/profile.cpp`, `tests/test_profile.cpp`
- Modifier : `core/channel_id.h` / `.cpp`, `core/CMakeLists.txt`, `tests/CMakeLists.txt`

**Produit :**
- `bool channelFromString(const char* name, ChannelId& out)` — réciproque de `toString`
- `struct PinConfig`, `DigitalInputProfile`, `AnalogInputProfile`, `SpeedProfile`,
  `BikeProfile`
- `BikeProfile defaultProfile()`

- [ ] **Étape 1 : écrire les tests qui échouent**

`tests/test_profile.cpp` :

```cpp
#include <catch2/catch_test_macros.hpp>

#include <cstddef>
#include <string>

#include "core/profile.h"

using namespace rondo;

TEST_CASE("le nom d'un canal se retrouve depuis sa chaine") {
    ChannelId id = ChannelId::Count;
    REQUIRE(channelFromString("speed", id));
    REQUIRE(id == ChannelId::Speed);

    REQUIRE(channelFromString("battery_volts", id));
    REQUIRE(id == ChannelId::BatteryVolts);
}

TEST_CASE("un nom de canal inconnu est refuse") {
    ChannelId id = ChannelId::Speed;
    REQUIRE_FALSE(channelFromString("inexistant", id));
    REQUIRE_FALSE(channelFromString(nullptr, id));
    REQUIRE(id == ChannelId::Speed);  // inchange
}

TEST_CASE("la conversion aller-retour est stable pour tous les canaux") {
    for (size_t i = 0; i < static_cast<size_t>(ChannelId::Count); ++i) {
        const ChannelId original = static_cast<ChannelId>(i);
        ChannelId roundtrip = ChannelId::Count;
        INFO("canal : " << toString(original));
        REQUIRE(channelFromString(toString(original), roundtrip));
        REQUIRE(roundtrip == original);
    }
}

TEST_CASE("le profil par defaut est vide mais coherent") {
    const BikeProfile p = defaultProfile();
    REQUIRE(p.digital_count == 0u);
    REQUIRE(p.analog_count == 0u);
    REQUIRE(p.speed.pin.gpio == kUnwiredPin);
    REQUIRE(std::string(p.theme) == "moderne");
}

TEST_CASE("une broche non cablee porte la valeur convenue") {
    const BikeProfile p = defaultProfile();
    REQUIRE(kUnwiredPin == -1);
    REQUIRE(p.speed.pin.gpio == kUnwiredPin);
}

TEST_CASE("les capacites du profil couvrent les besoins connus") {
    // 4 temoins + 2 boutons = 6 entrees tout ou rien.
    REQUIRE(kMaxDigitalInputs >= 6u);
    // tension, temperature, essence = 3, plus une de reserve.
    REQUIRE(kMaxAnalogInputs >= 4u);
    REQUIRE(kMaxCurvePoints >= 8u);
}
```

- [ ] **Étape 2 : lancer les tests et vérifier qu'ils échouent**

```bash
cmake --build build -j
```

Attendu : **échec**, `core/profile.h: No such file or directory`.

- [ ] **Étape 3 : ajouter la conversion inverse des canaux**

Dans `core/channel_id.h`, déclarer :

```cpp
// Reciproque de toString. Laisse `out` inchange et renvoie false si le nom
// est inconnu : un profil communautaire peut citer un canal qu'une version
// plus ancienne du firmware ne connait pas.
bool channelFromString(const char* name, ChannelId& out);
```

Dans `core/channel_id.cpp` :

```cpp
bool channelFromString(const char* name, ChannelId& out) {
    if (name == nullptr) {
        return false;
    }
    for (size_t i = 0; i < static_cast<size_t>(ChannelId::Count); ++i) {
        const ChannelId id = static_cast<ChannelId>(i);
        if (std::strcmp(name, toString(id)) == 0) {
            out = id;
            return true;
        }
    }
    return false;
}
```

Ajouter `#include <cstring>` et `#include <cstddef>` en tête du `.cpp`.

- [ ] **Étape 4 : écrire la structure du profil**

`core/profile.h` :

```cpp
#pragma once

#include <cstddef>
#include <cstdint>

#include "core/channel_id.h"
#include "core/curve.h"
#include "core/scale.h"
#include "core/speed_calculator.h"

namespace rondo {

inline constexpr int8_t kUnwiredPin = -1;
inline constexpr size_t kMaxDigitalInputs = 8;
inline constexpr size_t kMaxAnalogInputs = 4;
inline constexpr size_t kMaxCurvePoints = 12;
inline constexpr size_t kMaxNameLength = 32;
inline constexpr size_t kMaxThemeLength = 16;

struct PinConfig {
    // kUnwiredPin signifie que l'entree n'existe pas sur cette moto.
    // Le canal correspondant restera Absent, et son widget masque.
    int8_t gpio = kUnwiredPin;
    bool active_low = true;
};

struct DigitalInputProfile {
    ChannelId channel = ChannelId::Neutral;
    PinConfig pin;
};

struct AnalogInputProfile {
    ChannelId channel = ChannelId::BatteryVolts;
    int8_t adc_channel = kUnwiredPin;
    LinearScale scale;
    CurvePoint curve[kMaxCurvePoints] = {};
    uint8_t curve_count = 0;  // 0 = conversion affine par `scale`
};

struct SpeedProfile {
    PinConfig pin;
    uint32_t wheel_circumference_mm = 2170;
    uint8_t magnets = 2;
    float max_plausible_speed_kmh = 180.0f;
    uint32_t zero_speed_timeout_ms = 1500;

    SpeedConfig toSpeedConfig() const;
};

// Tout ce qui depend de la moto. Aucun pointeur, aucune allocation :
// le profil se copie, se compare et se stocke tel quel.
struct BikeProfile {
    uint16_t schema = 1;
    char name[kMaxNameLength] = {};
    char theme[kMaxThemeLength] = {};

    SpeedProfile speed;

    DigitalInputProfile digitals[kMaxDigitalInputs] = {};
    uint8_t digital_count = 0;

    AnalogInputProfile analogs[kMaxAnalogInputs] = {};
    uint8_t analog_count = 0;

    uint32_t odometer_offset_km = 0;
    float voltage_warn_low = 11.8f;
    float voltage_warn_high = 15.2f;
};

// Profil vierge : rien de cable, donc tous les canaux absents et un ecran
// vide plutot que des valeurs fausses.
BikeProfile defaultProfile();

}  // namespace rondo
```

`core/profile.cpp` :

```cpp
#include "core/profile.h"

#include <cstring>

namespace rondo {

SpeedConfig SpeedProfile::toSpeedConfig() const {
    SpeedConfig c;
    c.wheel_circumference_mm = wheel_circumference_mm;
    c.magnets_per_revolution = magnets;
    c.zero_speed_timeout_ms = zero_speed_timeout_ms;
    c.max_plausible_speed_kmh = max_plausible_speed_kmh;
    return c;
}

BikeProfile defaultProfile() {
    BikeProfile p;
    std::strncpy(p.theme, "moderne", kMaxThemeLength - 1);
    return p;
}

}  // namespace rondo
```

Ajouter `profile.cpp` aux sources de `rondo_core` et `test_profile.cpp` à `rondo_tests`.

- [ ] **Étape 5 : lancer les tests et vérifier qu'ils passent**

```bash
cmake -B build -DCMAKE_BUILD_TYPE=Debug && cmake --build build -j && ctest --test-dir build --output-on-failure
```

Attendu : `100% tests passed, 0 tests failed out of 119`.

- [ ] **Étape 6 : commit**

```bash
git add core tests
git commit -m "core: add the bike profile structure and channel name lookup"
```

---

## Tâche 2 : Validation du profil

C'est la tâche qui décide si le portail est utilisable ou dangereux. Un profil accepté à
tort produit un compteur qui ment ; un profil rejeté à tort produit un compteur qui ne
démarre pas.

**Fichiers :**
- Créer : `core/profile_validation.h`, `core/profile_validation.cpp`,
  `tests/test_profile_validation.cpp`
- Modifier : `core/CMakeLists.txt`, `tests/CMakeLists.txt`

**Produit :**
- `enum class ProfileError : uint8_t`
- `struct ValidationResult { ProfileError error; int index; bool ok() const; }`
- `ValidationResult validate(const BikeProfile&)`
- `const char* toString(ProfileError)`

**Règles**, toutes testées :

| Contrôle | Motif |
|---|---|
| Nom non vide | Un profil anonyme est ininstallable par un tiers |
| Circonférence entre 500 et 5000 mm | Hors de cette plage, c'est une faute de frappe |
| Aimants entre 1 et 12 | Idem |
| Vitesse maximale entre 10 et 400 km/h | Idem |
| GPIO entre −1 et 48 | Plage de l'ESP32-S3 |
| **Aucun GPIO en double** | Deux entrées sur la même broche : la plus vicieuse des erreurs |
| Aucun canal en double | Deux fournisseurs se marcheraient dessus |
| Courbe : 0 point, ou ≥ 2 points d'abscisses croissantes | Contrat de `Curve` |
| Thème non vide | Sinon aucun affichage |
| Seuil bas < seuil haut | Alertes incohérentes |

- [ ] **Étape 1 : écrire les tests qui échouent**

`tests/test_profile_validation.cpp` :

```cpp
#include <catch2/catch_test_macros.hpp>

#include <cstring>
#include <string>

#include "core/profile_validation.h"

using namespace rondo;

namespace {

// Profil minimal valide, point de depart de chaque test.
BikeProfile good() {
    BikeProfile p = defaultProfile();
    std::strncpy(p.name, "Kawasaki KDX 125", kMaxNameLength - 1);
    p.speed.pin.gpio = 4;
    p.digital_count = 1;
    p.digitals[0] = {ChannelId::Neutral, {5, true}};
    p.analog_count = 1;
    p.analogs[0].channel = ChannelId::BatteryVolts;
    p.analogs[0].adc_channel = 3;
    p.analogs[0].scale = {0.005804f, 0.0f};
    return p;
}

}  // namespace

TEST_CASE("un profil correct est accepte") {
    REQUIRE(validate(good()).ok());
}

TEST_CASE("un profil sans nom est refuse") {
    BikeProfile p = good();
    p.name[0] = '\0';
    REQUIRE(validate(p).error == ProfileError::EmptyName);
}

TEST_CASE("une circonference invraisemblable est refusee") {
    BikeProfile p = good();
    p.speed.wheel_circumference_mm = 217;  // oubli d'un zero
    REQUIRE(validate(p).error == ProfileError::WheelOutOfRange);

    p.speed.wheel_circumference_mm = 21700;
    REQUIRE(validate(p).error == ProfileError::WheelOutOfRange);
}

TEST_CASE("un nombre d'aimants invraisemblable est refuse") {
    BikeProfile p = good();
    p.speed.magnets = 0;
    REQUIRE(validate(p).error == ProfileError::MagnetsOutOfRange);

    p.speed.magnets = 50;
    REQUIRE(validate(p).error == ProfileError::MagnetsOutOfRange);
}

TEST_CASE("une vitesse maximale invraisemblable est refusee") {
    BikeProfile p = good();
    p.speed.max_plausible_speed_kmh = 5.0f;
    REQUIRE(validate(p).error == ProfileError::SpeedLimitOutOfRange);
}

TEST_CASE("un numero de broche hors plage est refuse") {
    BikeProfile p = good();
    p.digitals[0].pin.gpio = 99;
    const ValidationResult r = validate(p);
    REQUIRE(r.error == ProfileError::InvalidGpio);
    REQUIRE(r.index == 0);
}

TEST_CASE("deux entrees sur la meme broche sont refusees") {
    BikeProfile p = good();
    p.digital_count = 2;
    p.digitals[1] = {ChannelId::TurnLeft, {5, true}};  // deja pris par neutral
    const ValidationResult r = validate(p);
    REQUIRE(r.error == ProfileError::DuplicateGpio);
    REQUIRE(r.index == 1);
}

TEST_CASE("une entree partageant la broche du capteur de vitesse est refusee") {
    BikeProfile p = good();
    p.digitals[0].pin.gpio = 4;  // broche du capteur Hall
    REQUIRE(validate(p).error == ProfileError::DuplicateGpio);
}

TEST_CASE("plusieurs broches non cablees ne comptent pas comme des doublons") {
    BikeProfile p = good();
    p.digital_count = 3;
    p.digitals[1] = {ChannelId::TurnLeft, {kUnwiredPin, true}};
    p.digitals[2] = {ChannelId::TurnRight, {kUnwiredPin, true}};
    REQUIRE(validate(p).ok());
}

TEST_CASE("deux entrees sur le meme canal sont refusees") {
    BikeProfile p = good();
    p.digital_count = 2;
    p.digitals[1] = {ChannelId::Neutral, {6, true}};
    REQUIRE(validate(p).error == ProfileError::DuplicateChannel);
}

TEST_CASE("une courbe d'un seul point est refusee") {
    BikeProfile p = good();
    p.analogs[0].curve_count = 1;
    p.analogs[0].curve[0] = {100.0f, 20.0f};
    REQUIRE(validate(p).error == ProfileError::InvalidCurve);
}

TEST_CASE("une courbe aux abscisses decroissantes est refusee") {
    BikeProfile p = good();
    p.analogs[0].curve_count = 2;
    p.analogs[0].curve[0] = {200.0f, 20.0f};
    p.analogs[0].curve[1] = {100.0f, 60.0f};
    REQUIRE(validate(p).error == ProfileError::InvalidCurve);
}

TEST_CASE("des seuils de tension incoherents sont refuses") {
    BikeProfile p = good();
    p.voltage_warn_low = 16.0f;
    p.voltage_warn_high = 12.0f;
    REQUIRE(validate(p).error == ProfileError::InvalidVoltageThresholds);
}

TEST_CASE("un profil sans theme est refuse") {
    BikeProfile p = good();
    p.theme[0] = '\0';
    REQUIRE(validate(p).error == ProfileError::EmptyTheme);
}

TEST_CASE("trop d'entrees declarees est refuse") {
    BikeProfile p = good();
    p.digital_count = static_cast<uint8_t>(kMaxDigitalInputs + 1);
    REQUIRE(validate(p).error == ProfileError::TooManyInputs);
}

TEST_CASE("chaque code d'erreur porte un libelle") {
    REQUIRE(std::string(toString(ProfileError::DuplicateGpio)).length() > 0);
    REQUIRE(std::string(toString(ProfileError::None)).length() > 0);
}
```

- [ ] **Étape 2 : lancer les tests et vérifier qu'ils échouent**

```bash
cmake --build build -j
```

Attendu : **échec**, `core/profile_validation.h: No such file or directory`.

- [ ] **Étape 3 : écrire la validation**

`core/profile_validation.h` :

```cpp
#pragma once

#include <cstdint>

#include "core/profile.h"

namespace rondo {

enum class ProfileError : uint8_t {
    None,
    EmptyName,
    EmptyTheme,
    WheelOutOfRange,
    MagnetsOutOfRange,
    SpeedLimitOutOfRange,
    InvalidGpio,
    DuplicateGpio,
    DuplicateChannel,
    InvalidCurve,
    InvalidVoltageThresholds,
    TooManyInputs,
};

struct ValidationResult {
    ProfileError error = ProfileError::None;
    int index = -1;  // entree fautive, -1 si sans objet

    bool ok() const { return error == ProfileError::None; }
};

// Un profil invalide n'est JAMAIS applique : l'appelant garde le
// precedent et affiche l'erreur. Mieux vaut un compteur qui refuse une
// modification qu'un compteur qui affiche des valeurs fausses.
ValidationResult validate(const BikeProfile& profile);

const char* toString(ProfileError error);

}  // namespace rondo
```

`core/profile_validation.cpp` :

```cpp
#include "core/profile_validation.h"

#include <cstddef>

namespace rondo {
namespace {

constexpr int8_t kMaxGpio = 48;  // ESP32-S3

constexpr uint32_t kMinWheelMm = 500;
constexpr uint32_t kMaxWheelMm = 5000;
constexpr uint8_t kMinMagnets = 1;
constexpr uint8_t kMaxMagnets = 12;
constexpr float kMinSpeedLimit = 10.0f;
constexpr float kMaxSpeedLimit = 400.0f;

bool gpioValid(int8_t gpio) { return gpio == kUnwiredPin || (gpio >= 0 && gpio <= kMaxGpio); }

bool curveValid(const AnalogInputProfile& a) {
    if (a.curve_count == 0) {
        return true;  // conversion affine, pas de courbe
    }
    if (a.curve_count == 1 || a.curve_count > kMaxCurvePoints) {
        return false;
    }
    for (uint8_t i = 1; i < a.curve_count; ++i) {
        if (a.curve[i].x <= a.curve[i - 1].x) {
            return false;
        }
    }
    return true;
}

}  // namespace

ValidationResult validate(const BikeProfile& p) {
    if (p.name[0] == '\0') {
        return {ProfileError::EmptyName, -1};
    }
    if (p.theme[0] == '\0') {
        return {ProfileError::EmptyTheme, -1};
    }
    if (p.digital_count > kMaxDigitalInputs || p.analog_count > kMaxAnalogInputs) {
        return {ProfileError::TooManyInputs, -1};
    }
    if (p.speed.wheel_circumference_mm < kMinWheelMm ||
        p.speed.wheel_circumference_mm > kMaxWheelMm) {
        return {ProfileError::WheelOutOfRange, -1};
    }
    if (p.speed.magnets < kMinMagnets || p.speed.magnets > kMaxMagnets) {
        return {ProfileError::MagnetsOutOfRange, -1};
    }
    if (p.speed.max_plausible_speed_kmh < kMinSpeedLimit ||
        p.speed.max_plausible_speed_kmh > kMaxSpeedLimit) {
        return {ProfileError::SpeedLimitOutOfRange, -1};
    }
    if (!gpioValid(p.speed.pin.gpio)) {
        return {ProfileError::InvalidGpio, -1};
    }
    if (p.voltage_warn_low >= p.voltage_warn_high) {
        return {ProfileError::InvalidVoltageThresholds, -1};
    }

    // Deux entrees sur la meme broche donneraient des lectures aberrantes
    // sans aucun message d'erreur : c'est le controle qui compte le plus.
    // Le capteur de vitesse participe au controle.
    int8_t used_gpio[kMaxDigitalInputs + 1] = {};
    size_t used_count = 0;
    used_gpio[used_count++] = p.speed.pin.gpio;

    for (uint8_t i = 0; i < p.digital_count; ++i) {
        const int8_t gpio = p.digitals[i].pin.gpio;
        if (!gpioValid(gpio)) {
            return {ProfileError::InvalidGpio, static_cast<int>(i)};
        }
        if (gpio != kUnwiredPin) {
            for (size_t j = 0; j < used_count; ++j) {
                if (used_gpio[j] == gpio) {
                    return {ProfileError::DuplicateGpio, static_cast<int>(i)};
                }
            }
        }
        used_gpio[used_count++] = gpio;

        for (uint8_t j = 0; j < i; ++j) {
            if (p.digitals[j].channel == p.digitals[i].channel) {
                return {ProfileError::DuplicateChannel, static_cast<int>(i)};
            }
        }
    }

    for (uint8_t i = 0; i < p.analog_count; ++i) {
        if (!curveValid(p.analogs[i])) {
            return {ProfileError::InvalidCurve, static_cast<int>(i)};
        }
        for (uint8_t j = 0; j < i; ++j) {
            if (p.analogs[j].channel == p.analogs[i].channel) {
                return {ProfileError::DuplicateChannel, static_cast<int>(i)};
            }
        }
    }

    return {};
}

const char* toString(ProfileError error) {
    switch (error) {
        case ProfileError::None:            return "profil valide";
        case ProfileError::EmptyName:       return "nom de moto vide";
        case ProfileError::EmptyTheme:      return "aucun theme choisi";
        case ProfileError::WheelOutOfRange: return "circonference de roue invraisemblable";
        case ProfileError::MagnetsOutOfRange: return "nombre d'aimants invraisemblable";
        case ProfileError::SpeedLimitOutOfRange: return "vitesse maximale invraisemblable";
        case ProfileError::InvalidGpio:     return "numero de broche hors plage";
        case ProfileError::DuplicateGpio:   return "deux entrees sur la meme broche";
        case ProfileError::DuplicateChannel: return "deux entrees sur le meme canal";
        case ProfileError::InvalidCurve:    return "courbe de capteur invalide";
        case ProfileError::InvalidVoltageThresholds: return "seuils de tension incoherents";
        case ProfileError::TooManyInputs:   return "trop d'entrees declarees";
    }
    return "erreur inconnue";
}

}  // namespace rondo
```

Ajouter `profile_validation.cpp` aux sources de `rondo_core` et le fichier de test à
`rondo_tests`.

- [ ] **Étape 4 : lancer les tests et vérifier qu'ils passent**

```bash
cmake -B build -DCMAKE_BUILD_TYPE=Debug && cmake --build build -j && ctest --test-dir build --output-on-failure
```

Attendu : `100% tests passed, 0 tests failed out of 135`.

- [ ] **Étape 5 : commit**

```bash
git add core tests
git commit -m "core: validate bike profiles, rejecting duplicate pins and bad curves"
```

---

## Tâche 3 : Lecture et écriture JSON

**Fichiers :**
- Créer : `core/profile_json.h`, `core/profile_json.cpp`, `tests/test_profile_json.cpp`
- Modifier : `core/CMakeLists.txt`, `tests/CMakeLists.txt`

**Produit :**
- `bool parseProfile(const char* json, BikeProfile& out, ProfileError& error)`
- `size_t serializeProfile(const BikeProfile& p, char* out, size_t size)`

**Format**, qui est le contrat public du projet :

```json
{
  "schema": 1,
  "name": "Kawasaki KDX 125 SR",
  "theme": "moderne",
  "odometer_offset_km": 2566,
  "voltage_warn_low": 11.8,
  "voltage_warn_high": 15.2,
  "speed": {
    "gpio": 4,
    "active_low": true,
    "wheel_circumference_mm": 2170,
    "magnets": 2,
    "max_plausible_speed_kmh": 180,
    "zero_speed_timeout_ms": 1500
  },
  "digital_inputs": [
    {"channel": "neutral",    "gpio": 5,  "active_low": true},
    {"channel": "turn_left",  "gpio": 6,  "active_low": true},
    {"channel": "turn_right", "gpio": 7,  "active_low": true},
    {"channel": "high_beam",  "gpio": 15, "active_low": true}
  ],
  "analog_inputs": [
    {"channel": "battery_volts", "adc": 3, "scale": 0.005804, "offset": 0.0},
    {"channel": "engine_temp",   "adc": 4,
     "curve": [[500, 20], [1200, 60], [2000, 90], [3000, 120]]}
  ]
}
```

Les noms de canaux sont exactement ceux de `toString(ChannelId)`.

- [ ] **Étape 1 : compléter le jeu de codes d'erreur**

Deux erreurs ne peuvent naître qu'à la lecture JSON. Les ajouter à `ProfileError` dans
`core/profile_validation.h` :

```cpp
    UnknownChannel,
    MalformedJson,
```

et leurs libellés dans `toString` :

```cpp
        case ProfileError::UnknownChannel: return "nom de canal inconnu";
        case ProfileError::MalformedJson:  return "JSON illisible";
```

- [ ] **Étape 2 : ajouter cJSON au build**

Dans `core/CMakeLists.txt` :

```cmake
include(FetchContent)
FetchContent_Declare(
  cjson
  GIT_REPOSITORY https://github.com/DaveGamble/cJSON.git
  GIT_TAG        v1.7.18
)
set(ENABLE_CJSON_TEST OFF CACHE BOOL "" FORCE)
set(BUILD_SHARED_AND_STATIC_LIBS OFF CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(cjson)

target_link_libraries(rondo_core PUBLIC cjson)
```

cJSON est déjà fourni par ESP-IDF : sur la cible, remplacer ce bloc par `REQUIRES json`
dans `firmware/main/CMakeLists.txt`. Le code source, lui, est identique des deux côtés.

- [ ] **Étape 3 : écrire les tests qui échouent**

`tests/test_profile_json.cpp` :

```cpp
#include <catch2/catch_test_macros.hpp>
#include <catch2/matchers/catch_matchers_floating_point.hpp>

#include <cstring>
#include <string>

#include "core/profile_json.h"
#include "core/profile_validation.h"

using namespace rondo;
using Catch::Matchers::WithinAbs;

namespace {

const char* kKdxJson = R"({
  "schema": 1,
  "name": "Kawasaki KDX 125 SR",
  "theme": "moderne",
  "odometer_offset_km": 2566,
  "voltage_warn_low": 11.8,
  "voltage_warn_high": 15.2,
  "speed": {
    "gpio": 4, "active_low": true,
    "wheel_circumference_mm": 2170, "magnets": 2,
    "max_plausible_speed_kmh": 180, "zero_speed_timeout_ms": 1500
  },
  "digital_inputs": [
    {"channel": "neutral", "gpio": 5, "active_low": true},
    {"channel": "turn_left", "gpio": 6, "active_low": true}
  ],
  "analog_inputs": [
    {"channel": "battery_volts", "adc": 3, "scale": 0.005804, "offset": 0.0},
    {"channel": "engine_temp", "adc": 4, "curve": [[500, 20], [2000, 90]]}
  ]
})";

}  // namespace

TEST_CASE("un profil complet est lu correctement") {
    BikeProfile p;
    ProfileError err = ProfileError::None;
    REQUIRE(parseProfile(kKdxJson, p, err));

    REQUIRE(std::string(p.name) == "Kawasaki KDX 125 SR");
    REQUIRE(std::string(p.theme) == "moderne");
    REQUIRE(p.odometer_offset_km == 2566u);
    REQUIRE(p.speed.pin.gpio == 4);
    REQUIRE(p.speed.wheel_circumference_mm == 2170u);
    REQUIRE(p.speed.magnets == 2u);
    REQUIRE(p.digital_count == 2u);
    REQUIRE(p.digitals[1].channel == ChannelId::TurnLeft);
    REQUIRE(p.analog_count == 2u);
    REQUIRE(p.analogs[1].curve_count == 2u);
    REQUIRE_THAT(p.analogs[1].curve[1].y, WithinAbs(90.0f, 0.01f));
}

TEST_CASE("le profil lu est valide") {
    BikeProfile p;
    ProfileError err = ProfileError::None;
    REQUIRE(parseProfile(kKdxJson, p, err));
    REQUIRE(validate(p).ok());
}

TEST_CASE("l'aller-retour conserve le profil") {
    BikeProfile original;
    ProfileError err = ProfileError::None;
    REQUIRE(parseProfile(kKdxJson, original, err));

    char buffer[2048] = {};
    REQUIRE(serializeProfile(original, buffer, sizeof(buffer)) > 0);

    BikeProfile roundtrip;
    REQUIRE(parseProfile(buffer, roundtrip, err));
    REQUIRE(std::memcmp(&original, &roundtrip, sizeof(BikeProfile)) == 0);
}

TEST_CASE("un JSON malforme est refuse") {
    BikeProfile p;
    ProfileError err = ProfileError::None;
    REQUIRE_FALSE(parseProfile("{ ceci n'est pas du json", p, err));
    REQUIRE_FALSE(parseProfile(nullptr, p, err));
}

TEST_CASE("un champ inconnu est ignore, pas rejete") {
    // Un profil ecrit pour une version plus recente du firmware doit
    // rester chargeable : c'est ce qui permet le partage communautaire.
    const char* json = R"({
      "schema": 1, "name": "Test", "theme": "moderne",
      "future_feature": {"nested": [1, 2, 3]},
      "speed": {"gpio": 4, "wheel_circumference_mm": 2170, "magnets": 2}
    })";
    BikeProfile p;
    ProfileError err = ProfileError::None;
    REQUIRE(parseProfile(json, p, err));
    REQUIRE(std::string(p.name) == "Test");
}

TEST_CASE("un canal inconnu fait echouer la lecture avec un code clair") {
    const char* json = R"({
      "schema": 1, "name": "Test", "theme": "moderne",
      "speed": {"gpio": 4, "wheel_circumference_mm": 2170, "magnets": 2},
      "digital_inputs": [{"channel": "teleportation", "gpio": 5}]
    })";
    BikeProfile p;
    ProfileError err = ProfileError::None;
    REQUIRE_FALSE(parseProfile(json, p, err));
    REQUIRE(err == ProfileError::UnknownChannel);
}

TEST_CASE("un champ absent prend sa valeur par defaut") {
    const char* json = R"({
      "schema": 1, "name": "Minimal", "theme": "moderne",
      "speed": {"gpio": 4, "wheel_circumference_mm": 2170, "magnets": 2}
    })";
    BikeProfile p;
    ProfileError err = ProfileError::None;
    REQUIRE(parseProfile(json, p, err));
    REQUIRE(p.digital_count == 0u);
    REQUIRE(p.speed.zero_speed_timeout_ms == 1500u);
    REQUIRE_THAT(p.voltage_warn_low, WithinAbs(11.8f, 0.01f));
}

TEST_CASE("trop d'entrees dans le JSON est refuse plutot que tronque") {
    std::string json = R"({"schema":1,"name":"T","theme":"moderne",)"
                       R"("speed":{"gpio":4,"wheel_circumference_mm":2170,"magnets":2},)"
                       R"("digital_inputs":[)";
    for (size_t i = 0; i < kMaxDigitalInputs + 2; ++i) {
        json += R"({"channel":"neutral","gpio":5},)";
    }
    json.pop_back();
    json += "]}";

    BikeProfile p;
    ProfileError err = ProfileError::None;
    REQUIRE_FALSE(parseProfile(json.c_str(), p, err));
    REQUIRE(err == ProfileError::TooManyInputs);
}

TEST_CASE("un tampon de sortie trop petit ne deborde pas") {
    BikeProfile p = defaultProfile();
    std::strncpy(p.name, "Test", kMaxNameLength - 1);
    char tiny[8] = {};
    REQUIRE(serializeProfile(p, tiny, sizeof(tiny)) == 0u);
}
```

- [ ] **Étape 4 : lancer les tests et vérifier qu'ils échouent**

```bash
cmake --build build -j
```

Attendu : **échec**, `core/profile_json.h: No such file or directory`.

- [ ] **Étape 5 : écrire la lecture et l'écriture**

`core/profile_json.h` :

```cpp
#pragma once

#include <cstddef>

#include "core/profile.h"
#include "core/profile_validation.h"

namespace rondo {

// Lit un profil JSON. Les champs absents prennent leur valeur par defaut,
// les champs INCONNUS sont ignores : un profil ecrit pour une version plus
// recente du firmware doit rester chargeable.
//
// Ne valide PAS le profil obtenu : appeler validate() ensuite.
bool parseProfile(const char* json, BikeProfile& out, ProfileError& error);

// Ecrit le profil en JSON. Renvoie le nombre d'octets ecrits, ou 0 si le
// tampon est trop petit.
size_t serializeProfile(const BikeProfile& profile, char* out, size_t size);

}  // namespace rondo
```

`core/profile_json.cpp` — implémentation avec cJSON. Points à respecter :

- `cJSON_Parse` puis `cJSON_Delete` systématique, y compris sur les chemins d'erreur.
- Chaque lecture passe par un accesseur qui laisse la valeur par défaut si le champ est
  absent ou du mauvais type — jamais de `cJSON_GetNumberValue` sans vérifier `cJSON_IsNumber`.
- `channelFromString` échoue → `ProfileError::UnknownChannel`.
- Nombre d'entrées supérieur à `kMaxDigitalInputs` ou `kMaxAnalogInputs` →
  `ProfileError::TooManyInputs`, **jamais de troncature silencieuse**.
- La sérialisation utilise `cJSON_PrintPreallocated` pour ne pas allouer sur le tas de la
  cible ; si elle échoue, renvoyer 0 sans rien écrire.

- [ ] **Étape 6 : lancer les tests et vérifier qu'ils passent**

```bash
cmake -B build -DCMAKE_BUILD_TYPE=Debug && cmake --build build -j && ctest --test-dir build --output-on-failure
```

Attendu : `100% tests passed, 0 tests failed out of 144`.

L'aller-retour compare les profils avec `memcmp` sur toute la structure : si ce test échoue
alors que les champs visibles semblent corrects, c'est qu'un octet de bourrage n'est pas
initialisé. Corriger l'initialisation dans `defaultProfile()`, ne pas affaiblir le test.

- [ ] **Étape 7 : commit**

```bash
git add core tests
git commit -m "core: read and write bike profiles as JSON"
```

---

## Tâche 4 : Stockage du profil

**Fichiers :**
- Créer : `core/profile_store.h`, `sim/file_profile_store.h` / `.cpp`,
  `tests/test_file_profile_store.cpp`
- Modifier : `sim/CMakeLists.txt`, `tests/CMakeLists.txt`

**Produit :**
- `class IProfileStore` — `bool load(BikeProfile&)`, `bool save(const BikeProfile&)`,
  `bool clear()`
- `class FileProfileStore` — `explicit FileProfileStore(std::string path)`

- [ ] **Étape 1 : écrire les tests qui échouent**

`tests/test_file_profile_store.cpp` :

```cpp
#include <catch2/catch_test_macros.hpp>

#include <cstdio>
#include <cstring>
#include <string>

#include "core/profile_validation.h"
#include "sim/file_profile_store.h"

using namespace rondo;

namespace {

std::string tempPath() { return "/tmp/rondo-profile-test.json"; }

BikeProfile sample() {
    BikeProfile p = defaultProfile();
    std::strncpy(p.name, "Kawasaki KDX 125", kMaxNameLength - 1);
    p.speed.pin.gpio = 4;
    return p;
}

}  // namespace

TEST_CASE("un fichier absent ne fournit aucun profil") {
    std::remove(tempPath().c_str());
    FileProfileStore store(tempPath());
    BikeProfile p;
    REQUIRE_FALSE(store.load(p));
}

TEST_CASE("un profil enregistre se relit a l'identique") {
    std::remove(tempPath().c_str());
    FileProfileStore store(tempPath());
    REQUIRE(store.save(sample()));

    BikeProfile reloaded;
    REQUIRE(store.load(reloaded));
    REQUIRE(std::string(reloaded.name) == "Kawasaki KDX 125");
    REQUIRE(reloaded.speed.pin.gpio == 4);
}

TEST_CASE("effacer supprime le profil enregistre") {
    FileProfileStore store(tempPath());
    store.save(sample());
    REQUIRE(store.clear());

    BikeProfile p;
    REQUIRE_FALSE(store.load(p));
}

TEST_CASE("un fichier corrompu ne fournit aucun profil et ne plante pas") {
    std::remove(tempPath().c_str());
    FILE* f = std::fopen(tempPath().c_str(), "w");
    REQUIRE(f != nullptr);
    std::fputs("{ pas du json", f);
    std::fclose(f);

    FileProfileStore store(tempPath());
    BikeProfile p;
    REQUIRE_FALSE(store.load(p));
}

TEST_CASE("un profil invalide est refuse a l'enregistrement") {
    // Le stockage est la derniere barriere : rien d'invalide ne doit
    // pouvoir arriver sur la moto, quelle que soit la voie d'entree.
    FileProfileStore store(tempPath());
    BikeProfile bad = sample();
    bad.name[0] = '\0';
    REQUIRE_FALSE(store.save(bad));
}
```

- [ ] **Étape 2 : lancer les tests et vérifier qu'ils échouent**

```bash
cmake --build build -j
```

Attendu : **échec**, `sim/file_profile_store.h: No such file or directory`.

- [ ] **Étape 3 : écrire l'interface et l'implémentation fichier**

`core/profile_store.h` :

```cpp
#pragma once

#include "core/profile.h"

namespace rondo {

// Stockage persistant du profil : NVS sur la cible, fichier sur l'hote.
class IProfileStore {
 public:
    virtual ~IProfileStore() = default;

    // Renvoie false si aucun profil n'est enregistre ou s'il est illisible.
    virtual bool load(BikeProfile& out) = 0;

    // Renvoie false si le profil est invalide ou si l'ecriture echoue.
    // Un profil invalide n'est jamais enregistre, quelle que soit la voie
    // par laquelle il arrive.
    virtual bool save(const BikeProfile& profile) = 0;

    virtual bool clear() = 0;
};

}  // namespace rondo
```

`sim/file_profile_store.h` / `.cpp` — lit et écrit le JSON produit par `serializeProfile`
dans un fichier. `save()` appelle `validate()` et refuse si le profil est invalide.
`load()` appelle `parseProfile` puis `validate()` et refuse un profil invalide même s'il a
été écrit à la main dans le fichier.

Tampon de travail : 4 Kio sur la pile, suffisant pour un profil complet et sans allocation.

Ajouter les sources et le test aux cibles correspondantes.

- [ ] **Étape 4 : lancer les tests et vérifier qu'ils passent**

```bash
cmake -B build -DCMAKE_BUILD_TYPE=Debug && cmake --build build -j && ctest --test-dir build --output-on-failure
```

Attendu : `100% tests passed, 0 tests failed out of 149`.

- [ ] **Étape 5 : commit**

```bash
git add core sim tests
git commit -m "core: add the profile store interface and its file implementation"
```

---

## Tâche 5 : Construction du système depuis le profil

C'est ici que le projet cesse d'être spécifique à une moto.

**Fichiers :**
- Créer : `app/hal_factory.h`, `app/system_builder.h` / `.cpp`, `app/CMakeLists.txt`,
  `sim/sim_hal_factory.h` / `.cpp`, `tests/test_system_builder.cpp`
- Modifier : `CMakeLists.txt`, `sim/CMakeLists.txt`, `tests/CMakeLists.txt`

**Produit :**
- `class IHalFactory` — `const IDigitalIn* makeDigitalIn(int8_t gpio, bool pull_up)`,
  `const IAnalogIn* makeAnalogIn(int8_t adc_channel)`,
  `const IPulseCounter* makePulseCounter(int8_t gpio, uint32_t min_interval_us)`
- `struct BuiltSystem { std::vector<std::unique_ptr<IProvider>> providers; }`
- `BuiltSystem buildFromProfile(const BikeProfile&, IHalFactory&, OdometerService&, const IClock&)`

`app/` est la **racine de composition** : c'est le seul endroit du projet où une allocation
dynamique est permise, et elle n'a lieu qu'une fois, au démarrage. `core/` reste sans
allocation.

- [ ] **Étape 1 : écrire les tests qui échouent**

`tests/test_system_builder.cpp` :

```cpp
#include <catch2/catch_test_macros.hpp>

#include <cstring>

#include "app/system_builder.h"
#include "core/channel_bus.h"
#include "sim/sim_clock.h"
#include "sim/sim_hal_factory.h"
#include "sim/sim_store.h"

using namespace rondo;

namespace {

BikeProfile kdx() {
    BikeProfile p = defaultProfile();
    std::strncpy(p.name, "KDX", kMaxNameLength - 1);
    p.speed.pin.gpio = 4;
    p.digital_count = 2;
    p.digitals[0] = {ChannelId::Neutral, {5, true}};
    p.digitals[1] = {ChannelId::TurnLeft, {6, true}};
    p.analog_count = 1;
    p.analogs[0].channel = ChannelId::BatteryVolts;
    p.analogs[0].adc_channel = 3;
    p.analogs[0].scale = {0.005804f, 0.0f};
    return p;
}

}  // namespace

TEST_CASE("le constructeur produit un fournisseur par entree declaree") {
    SimClock clock;
    SimStore store(128);
    OdometerService odo(store, 10000);
    SimHalFactory hal;

    const BuiltSystem system = buildFromProfile(kdx(), hal, odo, clock);
    // 2 entrees tout ou rien + 1 analogique + 1 vitesse
    REQUIRE(system.providers.size() == 4u);
}

TEST_CASE("les canaux declares cessent d'etre absents") {
    SimClock clock;
    ChannelBus bus(clock);
    SimStore store(128);
    OdometerService odo(store, 10000);
    SimHalFactory hal;

    BuiltSystem system = buildFromProfile(kdx(), hal, odo, clock);
    for (auto& provider : system.providers) {
        provider->declareChannels(bus);
    }

    REQUIRE(bus.read(ChannelId::Neutral).validity != Validity::Absent);
    REQUIRE(bus.read(ChannelId::TurnLeft).validity != Validity::Absent);
    REQUIRE(bus.read(ChannelId::BatteryVolts).validity != Validity::Absent);
    REQUIRE(bus.read(ChannelId::Speed).validity != Validity::Absent);
}

TEST_CASE("une entree non cablee laisse son canal absent") {
    // Le coeur du multi-moto : une moto sans retour de clignotant droit
    // masque simplement le temoin, sans configuration supplementaire.
    SimClock clock;
    ChannelBus bus(clock);
    SimStore store(128);
    OdometerService odo(store, 10000);
    SimHalFactory hal;

    BikeProfile p = kdx();
    p.digital_count = 3;
    p.digitals[2] = {ChannelId::TurnRight, {kUnwiredPin, true}};

    BuiltSystem system = buildFromProfile(p, hal, odo, clock);
    for (auto& provider : system.providers) {
        provider->declareChannels(bus);
    }

    REQUIRE(bus.read(ChannelId::TurnRight).validity == Validity::Absent);
    REQUIRE(system.providers.size() == 4u);  // le 3e n'a pas ete construit
}

TEST_CASE("un capteur de vitesse non cable ne produit pas de fournisseur") {
    SimClock clock;
    ChannelBus bus(clock);
    SimStore store(128);
    OdometerService odo(store, 10000);
    SimHalFactory hal;

    BikeProfile p = kdx();
    p.speed.pin.gpio = kUnwiredPin;

    BuiltSystem system = buildFromProfile(p, hal, odo, clock);
    for (auto& provider : system.providers) {
        provider->declareChannels(bus);
    }

    REQUIRE(bus.read(ChannelId::Speed).validity == Validity::Absent);
    REQUIRE(bus.read(ChannelId::Odometer).validity == Validity::Absent);
    REQUIRE(system.providers.size() == 3u);
}

TEST_CASE("le fournisseur de vitesse recoit la configuration du profil") {
    SimClock clock;
    ChannelBus bus(clock);
    SimStore store(128);
    OdometerService odo(store, 10000);
    SimHalFactory hal;

    BikeProfile p = kdx();
    p.speed.wheel_circumference_mm = 1900;
    p.speed.magnets = 4;

    BuiltSystem system = buildFromProfile(p, hal, odo, clock);
    for (auto& provider : system.providers) {
        provider->declareChannels(bus);
        provider->poll(bus);
    }

    // 4 impulsions = 1 tour = 1900 mm
    hal.pulseCounter(4)->advance(1000, 4.0f);
    clock.advanceMs(1000);
    for (auto& provider : system.providers) {
        provider->poll(bus);
    }
    REQUIRE(odo.odometerMm() == 1900u);
}

TEST_CASE("la fabrique materielle reutilise une broche deja creee") {
    SimHalFactory hal;
    const IDigitalIn* first = hal.makeDigitalIn(5, true);
    const IDigitalIn* second = hal.makeDigitalIn(5, true);
    REQUIRE(first == second);
}
```

- [ ] **Étape 2 : lancer les tests et vérifier qu'ils échouent**

```bash
cmake --build build -j
```

Attendu : **échec**, `app/system_builder.h: No such file or directory`.

- [ ] **Étape 3 : écrire la fabrique et le constructeur**

`app/hal_factory.h` :

```cpp
#pragma once

#include <cstdint>

#include "hal/i_analog_in.h"
#include "hal/i_digital_in.h"
#include "hal/i_pulse_counter.h"

namespace rondo {

// Cree les entrees materielles decrites par le profil.
// La fabrique POSSEDE les objets crees : ils vivent aussi longtemps
// qu'elle, c'est-a-dire toute la duree de fonctionnement.
// Renvoie nullptr si la broche vaut kUnwiredPin.
class IHalFactory {
 public:
    virtual ~IHalFactory() = default;

    virtual const IDigitalIn* makeDigitalIn(int8_t gpio, bool pull_up) = 0;
    virtual const IAnalogIn* makeAnalogIn(int8_t adc_channel) = 0;
    virtual const IPulseCounter* makePulseCounter(int8_t gpio, uint32_t min_interval_us) = 0;
};

}  // namespace rondo
```

`app/system_builder.h` :

```cpp
#pragma once

#include <memory>
#include <vector>

#include "app/hal_factory.h"
#include "core/clock.h"
#include "core/odometer_service.h"
#include "core/profile.h"
#include "core/provider.h"

namespace rondo {

struct BuiltSystem {
    std::vector<std::unique_ptr<IProvider>> providers;
};

// Construit les fournisseurs decrits par le profil.
//
// Une entree dont la broche vaut kUnwiredPin ne produit AUCUN fournisseur :
// son canal reste donc Absent et le widget correspondant se masque tout
// seul. C'est ce qui permet au meme firmware et aux memes themes de servir
// des motos differemment equipees.
//
// Racine de composition : seule allocation dynamique du projet, au
// demarrage uniquement.
BuiltSystem buildFromProfile(const BikeProfile& profile, IHalFactory& hal,
                             OdometerService& odometer, const IClock& clock);

}  // namespace rondo
```

`app/system_builder.cpp` — pour chaque entrée tout ou rien câblée, un
`DigitalInputProvider` ; pour chaque entrée analogique câblée, un `AnalogSensorProvider`
avec sa courbe si `curve_count > 0` ; si le capteur de vitesse est câblé, un
`HallSpeedProvider` construit avec `profile.speed.toSpeedConfig()` et un compteur
d'impulsions dont l'intervalle minimal est calculé par :

```
min_interval_us = 3600 x circonference_mm / (aimants x vitesse_max_kmh)
```

`sim/sim_hal_factory.h` / `.cpp` — conserve les `SimDigitalIn`, `SimAnalogIn` et
`SimPulseCounter` créés dans des `std::map` indexées par broche, renvoie le même objet si la
broche est redemandée, et expose des accesseurs `SimDigitalIn* digitalIn(int8_t gpio)` et
`SimPulseCounter* pulseCounter(int8_t gpio)` pour que les tests puissent les piloter.

- [ ] **Étape 4 : lancer les tests et vérifier qu'ils passent**

```bash
cmake -B build -DCMAKE_BUILD_TYPE=Debug && cmake --build build -j && ctest --test-dir build --output-on-failure
```

Attendu : `100% tests passed, 0 tests failed out of 155`.

- [ ] **Étape 5 : commit**

```bash
git add app sim tests CMakeLists.txt
git commit -m "app: build providers from the bike profile instead of hard-coded wiring"
```

---

## Tâche 6 : Point d'accès WiFi et interface REST

À partir d'ici, le matériel est nécessaire.

**Fichiers :**
- Créer : `firmware/main/nvs_profile_store.h` / `.cpp`,
  `firmware/main/esp32_hal_factory.h` / `.cpp`, `firmware/main/config_portal.h` / `.cpp`
- Modifier : `firmware/main/CMakeLists.txt`, `firmware/main/app_main.cpp`

**Produit :** un point d'accès WiFi et quatre routes HTTP.

| Route | Effet |
|---|---|
| `GET /` | Page de configuration |
| `GET /api/profile` | Profil courant en JSON |
| `POST /api/profile` | Valide puis enregistre ; renvoie 400 et le libellé d'erreur si invalide |
| `GET /api/channels` | Valeurs et validité de tous les canaux, en direct |

`GET /api/channels` n'est pas un extra : c'est ce qui rend le portail utilisable. On
configure une broche, on actionne le clignotant, et on **voit** le canal réagir dans le
navigateur. Sans ça, calibrer un mappage relève de la devinette.

- [ ] **Étape 1 : écrire le stockage NVS**

`NvsProfileStore` — enregistre le JSON produit par `serializeProfile` sous forme de blob
NVS, espace de noms `rondo`, clé `profile`. Même contrat que `FileProfileStore` : `save()`
valide avant d'écrire, `load()` valide après avoir lu.

- [ ] **Étape 2 : écrire la fabrique matérielle ESP32**

`Esp32HalFactory` — implémente `IHalFactory` avec `Esp32DigitalIn`, `Esp32AnalogIn` et
`Esp32PulseCounter` du plan `rondo-acquisition`. Conserve les objets créés dans des
tableaux de taille fixe indexés par broche, et renvoie le même objet si la broche est
redemandée — exactement comme `SimHalFactory`.

- [ ] **Étape 3 : ouvrir le point d'accès**

`esp_wifi` en mode `WIFI_MODE_AP` : SSID `Rondo-XXXX` (les quatre derniers chiffres de
l'adresse MAC), WPA2, mot de passe par défaut `rondo1234` **modifiable dans le profil**,
canal 1, quatre connexions au maximum.

Serveur `esp_http_server` sur le port 80, taille de requête limitée à 8 Kio.

- [ ] **Étape 4 : implémenter les quatre routes**

`POST /api/profile` doit suivre exactement cet enchaînement, sans raccourci :

```
1. lire le corps de la requete (8 Kio maximum)
2. parseProfile()      -> si echec : 400 + toString(erreur)
3. validate()          -> si echec : 400 + toString(erreur) + index fautif
4. store.save()        -> si echec : 500
5. 200 + { "restart_required": true }
```

Le profil n'est **jamais appliqué à chaud** : il prend effet au redémarrage suivant.
Reconstruire les fournisseurs pendant que la tâche d'acquisition tourne exposerait à des
pointeurs pendants dans une tâche temps réel, pour un gain nul — on ne reconfigure pas sa
moto en roulant.

- [ ] **Étape 5 : vérifier au banc**

```bash
cd firmware && idf.py build flash monitor
```

Depuis un téléphone ou un ordinateur, se connecter au réseau `Rondo-XXXX`, puis :

```bash
curl http://192.168.4.1/api/profile
curl http://192.168.4.1/api/channels
curl -X POST http://192.168.4.1/api/profile -H 'Content-Type: application/json' \
     --data @profiles/kawasaki-kdx125.json
```

Attendu : le premier appel renvoie le profil courant, le deuxième les canaux avec leur
validité, le troisième `{"restart_required":true}`.

Vérifier aussi le chemin d'échec, qui compte autant :

```bash
curl -X POST http://192.168.4.1/api/profile -H 'Content-Type: application/json' \
     -d '{"schema":1,"name":"","theme":"moderne"}'
```

Attendu : **400** et le libellé `nom de moto vide`. Après redémarrage, le compteur doit
fonctionner **avec l'ancien profil** — le profil refusé ne doit avoir laissé aucune trace.

- [ ] **Étape 6 : commit**

```bash
git add firmware
git commit -m "firmware: add the WiFi access point, NVS store and REST API"
```

---

## Tâche 7 : Page de configuration

**Fichiers :**
- Créer : `firmware/data/index.html`
- Modifier : `firmware/CMakeLists.txt`, `firmware/main/config_portal.cpp`

**Produit :** une page unique, embarquée en LittleFS, sans dépendance externe.

- [ ] **Étape 1 : écrire la page**

Un seul fichier, HTML, CSS et JavaScript compris. **Aucun CDN, aucune bibliothèque** : le
téléphone est connecté à l'ESP32, il n'a pas d'accès à Internet — une page qui charge une
feuille de style distante s'afficherait nue.

Sections :

1. **Identité** — nom de la moto, thème (liste déroulante alimentée par le registre).
2. **Capteur de vitesse** — broche, circonférence, aimants, vitesse maximale. Un bouton
   « calibrer » qui affiche le nombre d'impulsions en direct pendant qu'on fait tourner la
   roue à la main.
3. **Entrées tout ou rien** — un tableau, une ligne par canal, avec broche, actif-bas, et
   **la valeur en direct** lue sur `/api/channels`.
4. **Entrées analogiques** — broche ADC, coefficients ou table de courbe, valeur brute et
   valeur convertie en direct.
5. **Seuils et odomètre** — seuils de tension, offset de mise en service.
6. **Import et export** — un bouton qui télécharge le profil en `.json`, un champ qui en
   charge un.

Le rafraîchissement des valeurs en direct interroge `/api/channels` toutes les 500 ms, et
**s'arrête quand l'onglet passe en arrière-plan** (`document.hidden`) : inutile de solliciter
l'ESP32 en permanence.

Un canal `Absent` s'affiche en grisé avec la mention « non câblé », un canal `Stale` en
orange. Les mêmes trois états que sur le compteur, pour que l'utilisateur retrouve le même
vocabulaire des deux côtés.

- [ ] **Étape 2 : embarquer la page**

Partition LittleFS déclarée dans `partitions.csv`, image construite et flashée par
`idf.py littlefs-flash` (ou `esptool` selon le composant retenu). La route `GET /` sert le
fichier avec `Content-Type: text/html` et `Cache-Control: no-store`.

- [ ] **Étape 3 : vérifier sur un téléphone**

Se connecter au réseau `Rondo-XXXX` et ouvrir `http://192.168.4.1`.

| Vérification | Attendu |
|---|---|
| Affichage sur écran de téléphone | Tout est lisible et utilisable sans zoom |
| Actionner le clignotant gauche | La ligne `turn_left` réagit en moins d'une seconde |
| Faire tourner la roue à la main | Le compteur d'impulsions progresse |
| Régler `gpio` à −1 sur une entrée | La ligne passe en grisé « non câblé » |
| Enregistrer un profil valide | Message de succès et invitation à redémarrer |
| Enregistrer un profil invalide | Message d'erreur explicite, **rien n'est enregistré** |
| Exporter puis réimporter | Le profil revient à l'identique |

- [ ] **Étape 4 : commit**

```bash
git add firmware
git commit -m "firmware: add the single-page configuration portal"
```

---

## Tâche 8 : Cycle de vie, profil de référence et documentation

**Fichiers :**
- Créer : `profiles/kawasaki-kdx125.json`, `docs/profiles.md`
- Modifier : `firmware/main/app_main.cpp`, `README.md`

- [ ] **Étape 1 : câbler l'ouverture et la fermeture du portail**

Le portail s'ouvre sur **appui long du bouton B** — l'action réservée en `rondo-ui` tâche 7.

Conditions et garde-fous :

```
Ouverture   : appui long sur B ET vitesse nulle depuis plus de 3 secondes
Fermeture   : appui long sur B a nouveau
              OU vitesse non nulle detectee
              OU 10 minutes sans requete HTTP
```

La fermeture sur détection de mouvement n'est pas une precaution de principe : le WiFi
consomme, chauffe dans un boîtier clos de 80 × 50 mm, et l'écran doit rester entièrement
disponible pour la conduite.

Pendant que le portail est ouvert, afficher un écran dédié : le SSID, le mot de passe et
l'adresse `192.168.4.1`. Personne ne devrait avoir à lire la documentation pour s'y
connecter.

- [ ] **Étape 2 : écrire le profil de référence**

`profiles/kawasaki-kdx125.json` — le profil complet de la moto de référence, reprenant
exactement le brochage figé dans `hardware/harness.md` et les valeurs mesurées lors de la
validation de `rondo-acquisition` (circonférence de roue réelle, courbe de la thermistance,
coefficient du diviseur de tension).

Vérifier qu'il est valide avant de le commiter :

```bash
curl -X POST http://192.168.4.1/api/profile -H 'Content-Type: application/json' \
     --data @profiles/kawasaki-kdx125.json
```

- [ ] **Étape 3 : écrire `docs/profiles.md`**

Doit contenir : le format JSON complet champ par champ avec les plages acceptées, la liste
des noms de canaux, la marche à suivre pour créer un profil pour une nouvelle moto, la
procédure de calibration de la circonférence de roue, la façon de relever une courbe de
thermistance, et les instructions pour contribuer un profil au dépôt.

- [ ] **Étape 4 : mettre le README à jour**

Le dépôt doit maintenant se présenter comme un projet utilisable par des tiers : ce que
c'est, à quoi ça ressemble, ce qu'il faut acheter (renvoi vers `hardware/bom.csv`), comment
flasher, comment configurer sa moto, et comment contribuer un profil.

- [ ] **Étape 5 : vérifier le parcours complet d'un nouvel utilisateur**

Le test qui compte vraiment. Effacer la NVS et repartir de zéro :

```bash
cd firmware && idf.py erase-flash flash monitor
```

Attendu, sans jamais recompiler :

1. Le compteur démarre avec le profil par défaut : aucun canal câblé, **écran vide plutôt
   que valeurs fausses**, aucun plantage.
2. Un appui long sur B ouvre le portail, l'écran affiche SSID et adresse.
3. Depuis un téléphone, importer `profiles/kawasaki-kdx125.json`.
4. Redémarrer : le compteur fonctionne entièrement.

Si l'une de ces quatre étapes impose de toucher au code, l'objectif du plan n'est pas
atteint.

- [ ] **Étape 6 : lancer la totalité des tests hôte**

```bash
ctest --test-dir build --output-on-failure
```

Attendu : `100% tests passed, 0 tests failed out of 155`.

- [ ] **Étape 7 : commit**

```bash
git add firmware profiles docs README.md
git commit -m "firmware: gate the portal on standstill and ship the reference profile"
```

---

## Suites

À la fin de ce plan, Rondo est installable sur une autre moto sans compilateur. Il reste un
plan.

| Plan | Contenu | Dépend de |
|---|---|---|
| `rondo-nav` | GNSS, chargement et suivi de trace GPX, écran de navigation | ce plan |

Le profil accueillera alors trois champs supplémentaires — broches de la liaison série
GNSS et débit — qui suivront exactement le même chemin : structure, validation, JSON,
portail. Si l'ajout de ces champs demande de toucher à autre chose que ces quatre endroits,
c'est que le profil a mal vieilli.
