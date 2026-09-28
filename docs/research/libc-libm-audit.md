# Audit des symboles et de l'empreinte libc / libm (soft-float) en Flash

**Projet :** Firmware Custom UV-K1 / UV-K5 V3 (Puya PY32F071 MCU, Cortex-M0+, 118 KiB Flash / 120 832 octets max)  
**Ticket de recherche :** Issue #2 — Alignement Roadmap Wayfinder (#1)  
**Branche :** `research/libc-libm-audit`  
**Date :** 28 septembre 2026  
**Auteur :** Subagent ResearchLibcLibm  

---

## 1. Synthèse exécutive & Chiffres clés

L'audit approfondi de la chaîne de compilation (`arm-none-eabi-gcc 13.3.rel1`), des scripts de link, des fichiers map (`f4hwn.fusion.map`, `f4hwn.fieldops.map`) et de l'ensemble de l'arborescence source `App/`, `Core/` et `Drivers/` révèle une situation remarquable et deux découvertes majeures :

1. **Aucune routine soft-float ni fonction `libm` n'est actuellement liée dans les builds de production (`Fusion`, `FieldOps`) :**
   - Le code applicatif historique et actuel repose déjà presque exclusivement sur de l'arithmétique entière ou virgule fixe (unités de 10 Hz pour les fréquences, dixièmes de millivolts pour la batterie, demi-dBm pour le RSSI, virgule fixe Q17 pour le synthétiseur BK4819, racine carrée entière `iSqrt` pour le spectre).
   - `libm.a` contribue pour **exactement 0 octet** (bien que `-lm` soit déclaré dans CMake, `--gc-sections` élimine l'archive dans son intégralité).
   - Les routines soft-float (`__aeabi_fadd`, `__aeabi_dmul`, etc.) contribuent pour **0 octet**.
   - La bibliothèque de formatage `App/external/printf/printf.c` a son support flottant désactivé par configuration (`PRINTF_DISABLE_SUPPORT_FLOAT` dans `printf_config.h`), évitant à elle seule une surcharge mesurée de **+7,2 KiB** en Flash.

2. **L'empreinte actuelle des bibliothèques externes (`libgcc.a` + `libg_nano.a`) est de 1 130 octets :**
   - `libgcc.a` : **888 octets** (divisions logicielles 32 bits `_divsi3`, `_udivsi3`, helper `_clzsi2`, et tables de saut switch).
   - `libg_nano.a` (newlib-nano) : **242 octets** (10 fonctions mémoires et chaînes de caractères ultra-compactes).

3. **Découverte majeure #1 — Bug de macro vendor `POSITION_VAL` (+256 octets de Flash récupérables immédiatement) :**
   - Dans `Drivers/CMSIS/Device/PY32F071/Include/py32f0xx.h`, la macro `POSITION_VAL(VAL)` est définie sous la forme `(__CLZ(__RBIT(VAL)))`. Sur Cortex-M0+, `__RBIT` est implémenté par une boucle `for` à l'exécution.
   - Les appels inline de l'HAL/LL (`LL_ADC_SetResolution`, `LL_ADC_SetChannelSamplingTime`, etc.) dans `BOARD_ADC_Init` génèrent ainsi du code lourd et appellent la routine logicielle `_clzsi2.o` de `libgcc.a`.
   - Remplacer cette macro par `(__builtin_ctz(VAL))` permet à GCC de replier intégralement les décalages de masques à la compilation.
   - **Gain validé par compilation :** `board.c.obj` passe de 728 B à 532 B (-196 B), et `_clzsi2.o` (60 B) est éliminé du link. **Gain net en Flash : 256 octets**.

4. **Découverte majeure #2 — Élimination possible de la division signée `_divsi3.o` (+472 octets de Flash récupérables) :**
   - La division non signée `_udivsi3.o` (276 B) est indispensable. En revanche, la division signée `_divsi3.o` (468 B) et `_dvmd_tls.o` (4 B) n'est appelée que par **4 fonctions** dans tout le firmware (`ACTION_RxA`, `RADIO_NextValidList`, `UI_DisplayAudioScopeOverlay` et `UI_DisplayMenu`).
   - Ces 4 fonctions effectuent des modulos ou divisions positives promues en `int` signé à cause de littéraux ou de types d'énumérations.
   - Leur conversion en arithmétique non signée ou en incrément circulaire permet de supprimer totalement `_divsi3.o` et `_dvmd_tls.o`, libérant **472 octets**.

---

## 2. Audit du code source applicatif (`App/`)

### 2.1 Fréquences et synthétiseur RF (`frequencies.c`, `driver/bk4819.c`)
- **Représentation :** Les fréquences sont stockées sous forme d'`uint32_t` en unités de 10 Hz (par exemple `14500000` = 145,00000 MHz).
- **Pas de fréquence (`gStepFrequencyTable`) :** Les pas sont exprimés en multiples de 10 Hz (`STEP_2_5kHz` = 250, `STEP_8_33kHz` = 833, `STEP_0_01kHz` = 1).
- **Programmation des registres BK4819 :**
  - La fonction `scale_freq()` convertit la fréquence audio en mot de commande registre via une virgule fixe Q17 :
    ```c
    return (((uint32_t)freq * 1353245u) + (1u << 16)) >> 17;
    ```
    Cette formule évite toute division et tout calcul en virgule flottante tout en garantissant un arrondi correct.
  - La puissance et le gain RSSI utilisent une arithmétique entière simple :
    ```c
    int16_t BK4819_GetRSSI_dBm(void) {
        uint16_t rssi = BK4819_GetRSSI();
        return (rssi / 2) - 160; // 0.5 dBm par unité, offset de -160 dBm
    }
    ```

### 2.2 Gestion batterie (`helper/battery.c`, `ui/battery.c`)
- **Tension :** Stockée en dixièmes de millivolts / centièmes de volt (`voltage_10mV`, ex. 828 = 8,28 V).
- **Courbes de décharge et conversion en pourcentage (`BATTERY_VoltsToPercent`) :**
  L'interpolation linéaire par morceaux utilise un facteur d'échelle entier fixe `mulipl = 1000` :
  ```c
  const int a = (crv[i - 1][1] - crv[i][1]) * mulipl / (crv[i - 1][0] - crv[i][0]);
  const int b = crv[i][1] - a * crv[i][0] / mulipl;
  const int p = a * voltage_10mV / mulipl + b;
  return MIN(MAX(p, 0), 100);
  ```
  Le calcul est 100% entier.
- **Affichage (`ui/battery.c`) :** Opérations bitmap pures, aucun calcul arithmétique.

### 2.3 Analyseur de spectre (`app/spectrum.c`)
- **Compression d'affichage :** Pour l'affichage non-linéaire du niveau RSSI, une compression par racine carrée est appliquée.
- **Implémentation :** Elle utilise `iSqrt(uint16_t n)` implémentée par l'algorithme de Newton-Raphson entier :
  ```c
  static uint8_t iSqrt(uint16_t n) {
      if (n == 0) return 0;
      uint16_t x = n;
      uint16_t y = (x + 1) >> 1;
      while (y < x) { x = y; y = (x + n / x) >> 1; }
      return (uint8_t)x;
  }
  ```
  Aucun appel à `sqrt()` ou `sqrtf()` de la `libm`.

### 2.4 Audio et tonalités (`audio.c`)
- Les tonalités (`BEEP_Classic_array`) sont des fréquences entières en Hz (400, 500, 600, 880, 1000 Hz) transmises directement à `BK4819_PlayTone()`.
- Aucun calcul flottant.

### 2.5 Aircopy et protocoles numériques (`app/aircopy.c`)
- Manipulation directe d'octets, décalages binaires et calcul CRC16/CRC32 matériel ou tabulé. Aucun flottant.

### 2.6 Formatage et `printf` (`App/external/printf/printf.c`)
- Le firmware intègre la bibliothèque légère `printf` de Marco Paland.
- Dans `App/CMakeLists.txt`, `PRINTF_INCLUDE_CONFIG_H` est défini, ce qui inclut `App/printf_config.h`.
- Ce fichier définit :
  ```c
  #define PRINTF_DISABLE_SUPPORT_PTRDIFF_T
  #define PRINTF_DISABLE_SUPPORT_FLOAT
  ```
- **Conséquence :** Les fonctions `_ftoa()` et `_etoa()` ne sont pas compilées. L'objet `printf.c.obj` ne pèse que **1 766 octets** de code (`.text`).
- Si `PRINTF_DISABLE_SUPPORT_FLOAT` n'était pas activé :
  - `printf.c.obj` grossirait de **+2 014 octets**.
  - L'utilisation de `double` dans `_ftoa`/`_etoa` forcerait l'inclusion des routines soft-float 64 bits de `libgcc.a` (`adddf3`, `muldf3`, `divdf3`), soit **+5 212 octets** supplémentaires.
  - **Surcoût total évité : +7 226 octets (~7,2 KiB) !**

### 2.7 Cas isolé : `MR_GetCacheHitRate` (`App/misc.c`)
- Dans `App/misc.c` (lignes 621-626) :
  ```c
  #ifdef ENABLE_FEAT_F4HWN_DEBUG
  float MR_GetCacheHitRate(void)
  {
      uint32_t total = cache_hits + cache_misses;
      if (total == 0) return 0.0f;
      return (float)cache_hits / (float)total * 100.0f;
  }
  #endif
  ```
- **Statut :** Inactif dans `Fusion` et `FieldOps` car `ENABLE_FEAT_F4HWN_DEBUG` est à `false`.
- **Risque :** Si un développeur active `ENABLE_FEAT_F4HWN_DEBUG` pour du diagnostic, cette unique fonction fait basculer le linker en important `divsf3.o`, `mulsf3.o`, `floatunsisf.o` et `_clzsi2.o`, causant une **inflation Flash immédiate de +1 440 octets**.
- **Alternative proposée :** Remplacer par un calcul en dixièmes de pourcent (ou pourcent entier) :
  ```c
  uint16_t MR_GetCacheHitRateTenths(void)
  {
      uint32_t total = cache_hits + cache_misses;
      if (total == 0) return 0;
      return (uint16_t)(((uint64_t)cache_hits * 1000u + (total / 2u)) / total);
  }
  ```

---

## 3. Cartographie exacte des symboles liés dans les builds de production

Mesures effectuées sur les fichiers ELF et MAP des presets `Fusion` et `FieldOps` :

```
Total Flash utilisé (Preset Fusion)   : 106 976 octets / 120 832 (88,53 %)
Total Flash utilisé (Preset FieldOps) : 113 048 octets / 120 832 (93,56 %)
```

### 3.1 Décomposition par bibliothèque externe

| Archive | Objet lié | Taille Flash | Symboles exportés / utilisés | Rôle |
| :--- | :--- | :---: | :--- | :--- |
| **`libm.a`** | *(aucun)* | **0 B** | *(aucun)* | Bibliothèque mathématique standard (entièrement éliminée par `--gc-sections`) |
| **`libgcc.a`** | `_divsi3.o` | 468 B | `__aeabi_idiv`, `__aeabi_idivmod`, `__divsi3` | Division et modulo signés 32 bits logiciel |
| | `_udivsi3.o` | 276 B | `__aeabi_uidiv`, `__aeabi_uidivmod`, `__udivsi3` | Division et modulo non signés 32 bits logiciel |
| | `_clzsi2.o` | 60 B | `__clzsi2` | Comptage des zéros de tête logiciel |
| | `_thumb1_case_sqi.o` | 20 B | `__gnu_thumb1_case_sqi` | Helper switch case (table d'adresses signée 8-bit) |
| | `_thumb1_case_uqi.o` | 20 B | `__gnu_thumb1_case_uqi` | Helper switch case (table d'adresses non-signée 8-bit) |
| | `_thumb1_case_shi.o` | 20 B | `__gnu_thumb1_case_shi` | Helper switch case (table d'adresses signée 16-bit) |
| | `_thumb1_case_uhi.o` | 20 B | `__gnu_thumb1_case_uhi` | Helper switch case (table d'adresses non-signée 16-bit) |
| | `_dvmd_tls.o` | 4 B | `__aeabi_idiv0`, `__aeabi_ldiv0` | Gestionnaire division par zéro |
| **Sous-total `libgcc.a`** | | **888 B** | | |
| **`libg_nano.a`** | `libc_a-strncpy.o` | 40 B | `strncpy` | Copie de chaîne bornée |
| | `libc_a-memmove.o` | 36 B | `memmove` | Déplacement mémoire avec recouvrement |
| | `libc_a-strchr.o` | 28 B | `strchr` | Recherche de caractère dans chaîne |
| | `libc_a-memcmp.o` | 28 B | `memcmp` | Comparaison mémoire |
| | `libc_a-strcat.o` | 26 B | `strcat` | Concaténation de chaîne |
| | `libc_a-strcmp.o` | 20 B | `strcmp` | Comparaison de chaîne |
| | `libc_a-memcpy-stub.o`| 18 B | `memcpy` | Copie mémoire standard |
| | `libc_a-memset.o` | 16 B | `memset` | Remplissage mémoire |
| | `libc_a-strcpy.o` | 16 B | `strcpy` | Copie de chaîne |
| | `libc_a-strlen.o` | 14 B | `strlen` | Longueur de chaîne |
| | `libc_a-init.o` | 0 B | | Initialisation (vide) |
| **Sous-total `libg_nano.a`**| | **242 B** | | |
| **TOTAL GÉNÉRAL** | | **1 130 B** | | **0,94 % du Flash total disponible** |

---

## 4. Quantification du coût d'introduction du soft-float sur Cortex-M0+

Le cœur ARM Cortex-M0+ (architecture Armv6-M) ne dispose **ni d'unité de calcul flottant matérielle (FPU)**, **ni d'instructions de division matérielle (`UDIV`/`SDIV`)**, **ni d'instruction de comptage de zéros (`CLZ`)**.

Pour évaluer avec une rigueur absolue la pénalité qu'engendrerait l'utilisation de flottants ou de fonctions mathématiques dans le firmware, nous avons compilé et mesuré de manière isolée chaque opération avec les mêmes drapeaux de compilation que le firmware (`-mcpu=cortex-m0plus -mthumb -Os --specs=nano.specs -Wl,--gc-sections -lm`) :

### 4.1 Opérations arithmétiques flottantes simples (simple précision, `float` 32-bit)

| Opération testée | Taille `.text` résultante | Modules importés depuis `libgcc.a` |
| :--- | :---: | :--- |
| *Baseline minimale* (`void _start(void){}`) | 2 B | *(aucun)* |
| `float` addition / soustraction (`a + b`) | **912 B** | `addsf3.o`, `_clzsi2.o` |
| `float` multiplication (`a * b`) | **720 B** | `mulsf3.o`, `_clzsi2.o` |
| `float` division (`a / b`) | **628 B** | `divsf3.o`, `_clzsi2.o` |
| `float` ensemble combiné (`+`, `-`, `*`, `/`) | **2 120 B** | `addsf3.o`, `divsf3.o`, `mulsf3.o`, `_clzsi2.o` |
| `float` comparaisons (`<`, `>`, `==`) | **512 B** | `_arm_cmpsf2.o`, `eqsf2.o`, `gesf2.o`, `lesf2.o` |
| Conversion `int` vers `float` | **228 B** | `floatsisf.o`, `_clzsi2.o` |
| Conversion `float` vers `int` | **84 B** | `fixsfsi.o` |

> **Constat :** L'introduction de simples calculs en `float` (+, -, *, /) engendre une surcharge minimale de **2,1 KiB de Flash**.

### 4.2 Opérations double précision (`double` 64-bit)

| Opération testée | Taille `.text` résultante | Modules importés depuis `libgcc.a` |
| :--- | :---: | :--- |
| `double` addition / soustraction (`da + db`) | **1 928 B** | `adddf3.o`, `_clzsi2.o` |
| `double` multiplication (`da * db`) | **1 520 B** | `muldf3.o`, `_clzsi2.o` |
| `double` division (`da / db`) | **1 908 B** | `divdf3.o`, `_udivsi3.o`, `_dvmd_tls.o`, `_clzsi2.o` |
| `double` ensemble combiné (`+`, `-`, `*`, `/`) | **5 212 B** | `adddf3.o`, `divdf3.o`, `muldf3.o`, `_udivsi3.o`, etc. |

> **Constat :** Les opérations en double précision (souvent introduites par inadvertance via des littéraux comme `1.0` au lieu de `1.0f` ou d'entiers) coûtent plus de **5,2 KiB de Flash**.

### 4.3 Fonctions transcendantes et mathématiques standard (`libm.a`)

| Fonction mathématique appelée | Empreinte Flash totale | Détail des dépendances importées |
| :--- | :---: | :--- |
| `floorf()` / `ceilf()` | **1 708 B** | `libm(sf_ceil.o, sf_floor.o)` + routines soft-float GCC |
| `sqrtf()` | **3 844 B** | `libm(wf_sqrt.o, ef_sqrt.o)` + errno + soft-float GCC |
| `powf()` | **6 372 B** | `libm(wf_pow.o, ef_pow.o, sf_finite.o, sf_scalbn.o)` + soft-float |
| `sinf()` / `cosf()` | **7 032 B** | `libm(sf_sin.o, sf_cos.o, kf_sin.o, kf_cos.o, rem_pio2)` + soft-float |
| `sqrt()` (double) | **8 400 B** | `libm(w_sqrt.o, e_sqrt.o)` + soft-float 64-bit |
| `sin()` / `cos()` (double) | **11 576 B** | `libm(s_sin.o, s_cos.o, k_sin.o, k_cos.o, rem_pio2)` + soft-float 64-bit |
| `pow()` (double) | **12 008 B** | `libm(w_pow.o, e_pow.o, s_finite.o)` + soft-float 64-bit |

> **Constat :** L'appel à une seule fonction trigonométrique `sinf` ou puissance `powf` consomme entre **6 et 7 KiB de Flash**, soit plus de **5,5 % de la mémoire totale du microcontrôleur**.

### 4.4 Formatage de chaînes (`sprintf` / `printf`)

| Fonctionnalité | Empreinte Flash | Commentaire |
| :--- | :---: | :--- |
| `printf.c` standard du repo (entiers uniquement) | **1 766 B** | Actuel : `PRINTF_DISABLE_SUPPORT_FLOAT` activé |
| `printf.c` avec support float `%f` activé | **8 992 B** | **+7 226 B** (+2 014 B dans printf.o, +5 212 B routines double) |
| newlib-nano `sprintf` avec `%f` (`-u _printf_float`) | **21 920 B** | **+21,9 KiB** (inacceptable sur PY32F071) |

---

## 5. Gisements d'optimisation concrets & Quick Wins

Grâce à cet audit, deux gisements d'économies concrets et directs ont été identifiés dans le code actuel.

### 5.1 Quick Win #1 : Correction de la macro `POSITION_VAL` (+256 B de Flash)

#### Origine du problème
Dans `Drivers/CMSIS/Device/PY32F071/Include/py32f0xx.h` (ligne 227) :
```c
#define POSITION_VAL(VAL)     (__CLZ(__RBIT(VAL)))
```
Sur Cortex-M0+, `__RBIT` est défini dans `cmsis_gcc.h` par une fonction inline contenant une boucle d'inversion bit à bit :
```c
__STATIC_FORCEINLINE uint32_t __RBIT(uint32_t value) {
    uint32_t result = value;
    for (value >>= 1U; value != 0U; value >>= 1U) {
        result <<= 1U;
        result |= value & 1U;
        s--;
    }
    result <<= s;
    return result;
}
```
En raison de cette boucle, **GCC en `-Os` ne parvient pas à évaluer l'expression à la compilation**, même lorsque `VAL` est une constante de masque évidente (par exemple `ADC_CR1_RES = 0x03000000`).

Par conséquent :
1. Chaque appel aux macros LL d'initialisation ADC (`LL_ADC_SetResolution`, `LL_ADC_SetDataAlignment`, `LL_ADC_REG_SetTriggerSource`, `LL_ADC_SetChannelSamplingTime` dans `board.c`) génère à l'exécution du code lourd et des branchements.
2. `BOARD_ADC_Init` émet 4 instructions `bl __clzsi2`.
3. Le linker est contraint d'importer l'objet `_clzsi2.o` (60 octets) depuis `libgcc.a`.

#### Solution
Remplacer dans `Drivers/CMSIS/Device/PY32F071/Include/py32f0xx.h` :
```c
#define POSITION_VAL(VAL)     ((uint32_t)__builtin_ctz(VAL))
```
Pour toute constante non nulle, `__builtin_ctz` est résolu à 100 % par le compilateur dès l'étape d'optimisation syntaxique (constant folding).

#### Mesures avant / après
- Taille de `build/Fusion/CMakeFiles/f4hwn.fusion.dir/App/board.c.obj` :
  - Avant : **728 octets**
  - Après : **532 octets** (**-196 octets**)
- Dépendance `_clzsi2.o` dans `libgcc.a` :
  - Avant : **60 octets**
  - Après : **0 octet** (**-60 octets**, éliminé car aucun autre appelant dans le firmware)
- **Gain net total en Flash : 256 octets**.

---

### 5.2 Quick Win #2 : Élimination de `_divsi3.o` (+472 B de Flash)

#### Origine du problème
`libgcc.a` embarque deux implémentations de division 32 bits :
- `_udivsi3.o` (**276 octets**) : Division / modulo non signés (`__aeabi_uidiv`, `__aeabi_uidivmod`).
- `_divsi3.o` (**468 octets**) + `_dvmd_tls.o` (**4 octets**) : Division / modulo signés (`__aeabi_idiv`, `__aeabi_idivmod`).

Une analyse exhaustive des points d'appel du binaire montre que **seules 4 fonctions applicatives** appellent la version signée :

1. **`App/app/action.c` (`ACTION_RxA`) :**
   ```c
   gSetting_set_audio_am = (gSetting_set_audio_am + 1) % 3;
   ```
   `gSetting_set_audio_am` est une variable 0..2. Le littéral `3` (signé) provoque la promotion en `int` et appelle `__aeabi_idivmod`.
   *Remplacement :*
   ```c
   if (++gSetting_set_audio_am >= 3u) gSetting_set_audio_am = 0u;
   ```

2. **`App/radio.c` (`RADIO_NextValidList`) :**
   ```c
   gEeprom.SCAN_LIST_DEFAULT = ((gEeprom.SCAN_LIST_DEFAULT - 2 + MAX_VALUE) % MAX_VALUE) + 1;
   ```
   Le `- 2` signé force une division signée.
   *Remplacement :* Arithmétique non signée explicite `((unsigned)gEeprom.SCAN_LIST_DEFAULT + (MAX_VALUE - 2u)) % MAX_VALUE + 1u`.

3. **`App/ui/main.c` (`UI_DisplayAudioScopeOverlay`) :**
   ```c
   #define SCOPE_SAMPLES 43 // défini comme int signé
   ...
   g_scope_write = (g_scope_write + 1u) % SCOPE_SAMPLES;
   ...
   for (uint8_t i = 0u; i < SCOPE_SAMPLES; i++) {
       const uint8_t idx = (g_scope_write + i) % SCOPE_SAMPLES;
       ...
   }
   ```
   Non seulement `43` est signé, mais la boucle exécute 43 divisions logicielles signées par image (soit plus de 2 150 divisions/s lors de la transmission !).
   *Remplacement :*
   ```c
   #define SCOPE_SAMPLES 43u
   if (++g_scope_write >= SCOPE_SAMPLES) g_scope_write = 0u;
   ...
   uint8_t idx = g_scope_write;
   for (uint8_t i = 0u; i < SCOPE_SAMPLES; i++) {
       ...
       if (++idx >= SCOPE_SAMPLES) idx = 0u;
   }
   ```

4. **`App/ui/menu.c` (`UI_DisplayMenu`) :**
   - Ligne 1033 : `sprintf(String, "%3d.%05u", gSubMenuSelection / 100000, abs(gSubMenuSelection) % 100000);` pour le décalage répéteur (`MENU_OFFSET`). C'est le seul endroit où la valeur peut être négative.
   - Lignes 1078, 1115, 1264, 1286, 1296 : divisions par 60, 1000 de valeurs de menu toujours positives.
   *Remplacement :*
   Extraire le signe manuellement pour `MENU_OFFSET` (`if (val < 0) { sign = '-'; val = -val; }`) et forcer les autres divisions en non signées (`/ 60u`).

#### Bénéfice
En éliminant tout appel à `__aeabi_idiv` et `__aeabi_idivmod`, `_divsi3.o` (468 B) et `_dvmd_tls.o` (4 B) sont complètement exclus du lien.
**Gain potentiel net en Flash : 472 octets.**

---

### 5.3 Nettoyage CMake : Suppression de `-lm`

Dans `cmake/gcc-arm-none-eabi.cmake` :
```cmake
set(TOOLCHAIN_LINK_LIBRARIES "m")
```
Et dans `CMakeLists.txt` :
```cmake
target_link_libraries(${EXE_NAME} ${TOOLCHAIN_LINK_LIBRARIES})
```
Comme démontré, `libm.a` n'est pas utilisé. Supprimer cette déclaration est une mesure d'hygiène logicielle qui formalise l'interdiction des fonctions mathématiques flottantes dans le projet et prévient toute inclusion accidentelle lors de futurs développements.

---

### 5.4 Évaluation du remplacement de `libg_nano.a` par des micro-stubs

L'ensemble des 10 fonctions importées de `libg_nano.a` ne pèse au total que **242 octets** :
- `strlen` : 14 B
- `strcpy` : 16 B
- `memset` : 16 B
- `memcpy` : 18 B
- `strcmp` : 20 B
- `strcat` : 26 B
- `memcmp` : 28 B
- `strchr` : 28 B
- `memmove`: 36 B
- `strncpy`: 40 B

Des micro-stubs spécialisés en assembleur Thumb-1 ou en C minimaliste permettraient au mieux de réduire cette taille à environ 130 octets, soit un **gain théorique maximal de ~110 octets**.
Cependant, newlib-nano bénéficie d'implémentations hautement testées et d'optimisations d'alignement gérées par GCC (builtin expansion). **Le remplacement de ces fonctions n'est donc pas prioritaire**, le ratio gain/risque étant nettement moins favorable que les Quick Wins #1 et #2.

---

## 6. Synthèse des gains et Recommandations

### Tableau récapitulatif des gains identifiés

| Optimisation proposée | Nature du changement | Complexité | Gain Flash mesuré / estimé |
| :--- | :--- | :---: | :---: |
| **Correction macro `POSITION_VAL`** | Remplacer `__CLZ(__RBIT(v))` par `__builtin_ctz(v)` dans `py32f0xx.h` | Trivale (1 ligne) | **+256 octets** *(validé)* |
| **Éradication division signée `_divsi3`** | Remplacer modulos/divisions signées par unsigned / incréments circulaires | Faible (4 fichiers) | **+472 octets** *(estimé)* |
| **Sécurisation `MR_GetCacheHitRate`** | Remplacer le `float` par un entier en dixièmes de pourcent | Trivale (5 lignes) | **+1 440 octets** *(en mode DEBUG)* |
| **Suppression du flag `-lm`** | Nettoyage `cmake/gcc-arm-none-eabi.cmake` | Trivale (1 ligne) | 0 B *(hygiène)* |
| **Micro-stubs string/memory** | Réécrire `strcpy`, `strlen`, `memmove`, etc. | Moyenne | ~110 octets *(non prioritaire)* |
| **TOTAL DES GAINS IMMÉDIATS RECOMMANDÉS** | | | **+728 octets** (production) / **+2 168 octets** (mode debug) |

### Recommandations pour l'équipe Wayfinder
1. **Appliquer immédiatement le Quick Win `POSITION_VAL`** dans un ticket de refactoring dédié : gain immédiat de **256 octets** sans aucune modification de comportement ni risque de régression.
2. **Programmer l'optimisation des 4 points d'appel de division signée** : gain de **472 octets** et suppression de 43 divisions logicielles par image dans l'Audio Scope.
3. **Conserver la configuration `PRINTF_DISABLE_SUPPORT_FLOAT`** : impératif absolu sous peine de saturer immédiatement la Flash restante (+7,2 KiB).
4. **Supprimer `set(TOOLCHAIN_LINK_LIBRARIES "m")`** du fichier toolchain CMake.
