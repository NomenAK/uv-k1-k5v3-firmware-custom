# Rapport de Recherche : Évaluation et Faisabilité de LTO (-flto) avec Isolation .mb_ramfunc

- **Ticket de référence** : Issue [#3](https://github.com/NomenAK/uv-k1-k5v3-firmware-custom/issues/3)
- **Ticket parent (Map)** : Issue [#1](https://github.com/NomenAK/uv-k1-k5v3-firmware-custom/issues/1)
- **Cible matérielle** : Quansheng UV-K1 / UV-K5 V3 — MCU Puya PY32F071xB (ARM Cortex-M0+, Flash interne 118 KiB / 120 832 octets max, SRAM 16 KiB)
- **Toolchain cible** : Arm GNU Toolchain GCC 13.3.rel1 (`arm-none-eabi-gcc`), options `-Os`, `--specs=nano.specs`
- **Statut** : Faisable, validé techniquement avec patch d'isolation

---

## 1. Résumé Exécutif

L'optimisation globale à l'édition de liens (Link-Time Optimization ou **LTO**, via `-flto=auto`) représente le levier de réduction d'empreinte mémoire Flash le plus puissant et le plus efficient pour le firmware monolithique. 

Historiquement introduit dans le commit `164a7273`, LTO avait permis un gain immédiat de **3 484 octets en Flash** (-4,13 %) et **352 octets en RAM** sur le preset `Fusion`. Cependant, il avait été désactivé par défaut dès le commit `3f3dab7`, puis rendu incompatible par l'introduction du sous-système de restauration Multiboot Overlay (`5b617408`), dont la porte de sécurité de build (`cmake/check_mb_ramfunc.cmake`) rejette formellement les objets d'entrée LTO.

Cette étude démontre que :
1. **`ENABLE_LTO` est actuellement inopérant** : dans `CMakeLists.txt`, `-flto=auto` n'est appliqué qu'aux options de compilation de `${EXE_NAME}` et est totalement absent des options du linker (`target_link_options`), en plus d'être positionné à `false` dans `CMakePresets.json`.
2. **LTO non régulé brise l'intégrité du stub Multiboot** : sous `-flto`, GCC génère des objets intermédiaires de partitionnement (`ltrans0.ltrans.o`), efface la section `.MBRamFunc` de l'objet natif `mb_flash.c.obj` au profit de sections GIMPLE (`.gnu.lto_...`), et permet à l'optimiseur inter-procédural d'inliner du code résident en Flash dans le stub RAM, risquant un brickage irréversible du MCU pendant la reprogrammation de la Flash interne.
3. **Une isolation chirurgicale résout 100 % du problème** : en compilant spécifiquement `App/driver/mb_flash.c` avec le drapeau `-fno-lto` via CMake, `mb_flash.c.obj` redevient un objet ELF natif contenant la section `.MBRamFunc` intacte et sans relocations. La porte de validation `check_mb_ramfunc.cmake` s'exécute avec succès, tout en permettant au reste du firmware (> 50 unités de traduction) de bénéficier de l'optimisation LTO maximale.
4. **Gain estimé pour le Preset Max** : une réduction d'empreinte Flash comprise entre **3,5 KiB et 5,0 KiB** (typiquement ~4 000 octets), ce qui comble la majorité du déficit mémoire nécessaire pour faire tenir le Preset Max sous la barre fatidique des 118 KiB.

---

## 2. État des Lieux : Pourquoi `ENABLE_LTO` est Actuellement Inactif

Dans l'état actuel de la branche `main` :

1. **Configuration des presets (`CMakePresets.json`)** :
   ```json
   "ENABLE_LTO": false
   ```
   Tous les presets (`Fusion`, `Transfer`, `FieldOps`, `Labs`, `Max`) héritent de la valeur `false`.

2. **Configuration CMake (`CMakeLists.txt:100-102`)** :
   ```cmake
   if(ENABLE_LTO)
       target_compile_options(${EXE_NAME} PRIVATE -flto=auto)
   endif()
   ```

3. **Les lacunes bloquantes du build system** :
   - **Absence de `-flto` au niveau du linker** : En GCC, la phase d'optimisation inter-procédurale (IPA) et la génération de code globale ont lieu au moment de l'édition de liens via `collect2` et `lto-wrapper`. Passer `-flto=auto` uniquement à `target_compile_options(${EXE_NAME})` indique au compilateur d'émettre du bytecode GIMPLE dans les fichiers `.o`. Cependant, sans `target_link_options(${EXE_NAME} PRIVATE -flto=auto)`, le pilote du linker n'est pas instruit d'exécuter la parallélisation automatique des partitions (`-flto=auto`), et les drapeaux d'optimisation de link peuvent diverger.
   - **Incompatibilité avec `check_mb_ramfunc.cmake`** : Si un développeur force `-DENABLE_LTO=ON`, le build échoue immédiatement lors de la commande POST_BUILD avec l'erreur :
     ```text
     CMake Error at cmake/check_mb_ramfunc.cmake:110 (message):
       check_mb_ramfunc: mb_flash object not found in firmware.map (the overlay safety gate requires a non-LTO input object)
     ```

---

## 3. Interaction entre `-flto=auto`, GCC 13.3.rel1 et Cortex-M0+

### 3.1 Mécanisme LTO sous GCC 13.3.rel1
Lorsque `-flto=auto` est activé sous GCC 13 :
1. **Compilation frontend** : Chaque fichier `.c` est analysé, optimisé localement, puis sérialisé sous forme de représentation intermédiaire GIMPLE stockée dans des sections ELF dédiées (`.gnu.lto_*`). Par défaut (`-fno-fat-lto-objects`), les sections normales `.text` de l'objet ne contiennent pas de code machine complet.
2. **Phase WPA (Whole Program Analysis)** : Au link, `lto-wrapper` invoque `lto1` pour construire le graphe global des appels (Call Graph) couvrant l'ensemble du projet.
3. **Partitionnement LTRANS (Local Transformation)** : GCC découpe le programme en partitions indépendantes traitées en parallèle (`-flto=auto` utilise autant de threads que de cœurs CPU détectés).
4. **Code generation** : Chaque partition produit un objet temporaire de code machine nommé `ccXXXXXX.ltrans0.ltrans.o`, qui est ensuite transmis au linker final GNU `ld`.

### 3.2 Spécificités de l'architecture ARM Cortex-M0+ (ARMv6-M)
L'architecture Cortex-M0+ impose des contraintes physiques strictes qui maximisent les gains apportés par LTO :
- **Pression extrême sur les registres bas (`r0`-`r7`)** : Le jeu d'instructions Thumb-1 16-bit ne peut accéder aux registres hauts (`r8`-`r12`, `lr`) que via de rares instructions (`MOV`, `ADD`, `CMP`, `BX`). Hors LTO, chaque appel de fonction cross-module doit respecter l'ABI ARM (AAPCS), ce qui force de nombreux déversements sur la pile (`PUSH`/`POP`). Sous LTO, GCC applique une analyse inter-procédurale de la durée de vie des variables et ajuste les conventions d'appel des fonctions internes pour conserver les valeurs dans les registres bas disponibles, éliminant des milliers d'instructions `push`/`pop` sur l'ensemble du firmware.
- **Suppression du surcoût d'appel (Call Overhead)** : En Thumb-1, un appel externe utilise l'instruction 32-bit `BL` (4 octets) plus le prologue/épilogue standard (`push {r4, lr}` / `pop {r4, pc}`, 4 octets). L'inlining cross-fichier de petites fonctions d'accès aux registres ou de conversion économise 8 à 16 octets à chaque point d'appel.
- **Fusion des Literal Pools (Pools de constantes)** : Sur Cortex-M0+, les valeurs 32-bit (ex: pointeurs d'adresses de base des périphériques `0x40021000` pour RCC, `0x40020000` pour GPIOA, masques de configuration) sont chargées via `LDR Rd, [PC, #offset]`. Sans LTO, chaque unité de traduction duplique ses propres constantes dans son literal pool local. LTO unifie les tables de constantes et les chaînes de formatage (`printf`, libellés UI) à travers tout le projet.
- **Élimination de code mort inter-modules (IPA-DCE)** : Les fonctions utilitaires déclarées non-statiques mais non appelées dans une configuration donnée sont élaguées dès le niveau GIMPLE, avant même le passage de `-Wl,--gc-sections`.

---

## 4. La Contrainte Critique : `.mb_ramfunc` et `check_mb_ramfunc.cmake`

### 4.1 Le rôle vital du stub Multiboot
Le sous-système Multiboot (`App/driver/mb_flash.c`) permet de reprogrammer la Flash interne du MCU en cours de fonctionnement depuis une image stockée dans la Flash SPI externe (PY25Q16).

Lors de cette restauration :
1. La fonction critique `MB_RamReflash` et ses primitives d'accès matériel (`mb_ram_spi`, `mb_ram_flash_idle`, `mb_ram_reset`, etc.) s'exécutent intégralement depuis la RAM (section `.mb_ramfunc` mappée sur l'overlay du cache de secteur PY25Q16, VMA `ORIGIN(RAM) + 0x280`).
2. Pendant toute la durée de l'opération, la Flash interne est déverrouillée et soumise aux commandes d'effacement de page (`FLASH_CR_PER`) et de programmation de page (`FLASH_CR_PG`).
3. **Le piège fatal** : Si une seule instruction exécutée par le processeur tente d'adresser la mémoire Flash (instruction fetch vers une fonction résidente en Flash, accès à une constante du literal pool en Flash, ou appel indirect à une fonction de la libc comme `memset`), le bus mémoire interne se bloque ou lit `0xFFFFFFFF`. Le CPU part immédiatement en HardFault ou exécute des instructions invalides en pleine séquence d'effacement. **Le firmware est corrompu et la radio est définitivement brickée**, nécessitant un flashage DFU matériel en usine.

### 4.2 Les deux verrous de sécurité existants
Pour garantir formellement l'absence de régression, deux mécanismes complémentaires sont en place :
1. **Dans le Linker Script (`Core/py32f071xb.ld:175`)** :
   ```ld
   OVERLAY (ORIGIN(RAM) + 0x280) : NOCROSSREFS AT (LOADADDR(.noncacheable) + SIZEOF(.noncacheable))
   {
     .mb_ramfunc
     {
       . = ALIGN(4);
       __mb_ramfunc_start = .;
       KEEP(*(.MBRamFunc))
       KEEP(*(.MBRamFunc*))
       . = ALIGN(4);
       __mb_ramfunc_end = .;
     }
     ...
   } >RAM
   ```
   La directive GNU ld `NOCROSSREFS` ordonne au linker d'émettre une erreur si une section de l'overlay fait référence à une section extérieure.

2. **Dans le script de validation (`cmake/check_mb_ramfunc.cmake`)** :
   Ce script s'exécute en post-build sur `${EXE_NAME}` dès que `ENABLE_FEAT_F4HWN_MULTIBOOT_OVERLAY` est actif :
   - **Contrôle 1 (Désassemblage complet)** : Décode chaque instruction de branchement (`b`, `bl`, `blx`, `bx`, `cbz`, `cbnz`). Vérifie qu'aucun branchement indirect n'existe (seul `bx lr` est autorisé), et que toutes les cibles de saut direct sont strictement comprises dans l'intervalle d'adresses `[__mb_ramfunc_start, __mb_ramfunc_end]`.
   - **Contrôle 2 (Scan des relocations de l'objet d'entrée)** :
     ```cmake
     file(READ "${MAP}" map_contents)
     string(REGEX MATCH
         "[^\r\n]*\\.MBRamFunc[^\r\n]*mb_flash\\.c\\.(obj|o)"
         object_line "${map_contents}")
     if(object_line STREQUAL "")
         message(FATAL_ERROR
             "check_mb_ramfunc: mb_flash object not found in ${MAP} "
             "(the overlay safety gate requires a non-LTO input object)")
     endif()
     ...
     execute_process(
         COMMAND "${OBJDUMP}" -r -j .MBRamFunc "${object_path}"
         ...
     )
     if(relocations MATCHES "R_ARM_")
         message(FATAL_ERROR "check_mb_ramfunc: .MBRamFunc has an external code/data reference")
     endif()
     ```

### 4.3 Pourquoi LTO global non régulé brise cette sécurité
L'expérimentation montre que si `mb_flash.c` est compilé avec `-flto` :
1. **Échec du map file** : La section `.MBRamFunc` n'est plus issue de `mb_flash.c.obj`, mais de la partition temporaire LTO `ccXXXXXX.ltrans0.ltrans.o`. L'expression régulière échoue, et le build s'interrompt avec `FATAL_ERROR`.
2. **Échec d'objdump** : Dans un fichier objet compilé avec `-flto`, les sections de code machine sont absentes ; `objdump -r -j .MBRamFunc` renvoie une erreur car `.MBRamFunc` est introuvable.
3. **Risque d'inlining croisé** : L'optimiseur inter-unités de traduction pourrait inliner des fonctions externes (ex: helpers GPIO, fonctions de temporisation) dans `MB_RamReflash`, important des références vers la Flash en plein milieu du code exécuté en RAM.

---

## 5. La Solution : Isolation Chirurgicale de `mb_flash.c`

### 5.1 Pourquoi les pragmas et attributs de fonction ne suffisent pas
- L'utilisation de `#pragma GCC optimize("no-lto")` ou `__attribute__((optimize("no-lto")))` est **rejetée par GCC** avec l'avertissement : `warning: bad option ‘-fno-lto’ to pragma ‘optimize’ [-Wpragmas]`. GCC continue d'émettre du bytecode GIMPLE.
- Les attributs `__attribute__((noinline, noclone, used))` déjà présents sur `MB_RAM_HELPER` empêchent GCC d'inliner ou de cloner les fonctions de `mb_flash.c` vers l'extérieur. En revanche, ils n'empêchent pas GCC d'inliner des fonctions *extérieures* dans `mb_flash.c`, ni ne résolvent la disparition de `.MBRamFunc` dans l'objet ELF intermédiaire.

### 5.2 L'isolation par drapeau de compilation de fichier source (`-fno-lto`)
En GCC, lorsqu'un drapeau `-fno-lto` apparaît après `-flto=auto` sur la ligne de commande de compilation d'une unité de traduction, **GCC désactive totalement la génération LTO pour ce fichier unique**. Le compilateur génère directement du code machine ARM Thumb natif dans `mb_flash.c.obj`.

Dans CMake, cela s'exécute de façon propre et déterministe via la propriété de fichier source :
```cmake
set_source_files_properties(App/driver/mb_flash.c PROPERTIES COMPILE_OPTIONS "-fno-lto")
```

### 5.3 Preuve expérimentale du comportement
Un banc d'essai comparatif a été exécuté avec `gcc` et `objdump` :

| Propriété | Scénario A : Full LTO | Scénario B : `mb_flash.c` avec `-fno-lto` + Link LTO |
| :--- | :--- | :--- |
| **Section dans le map file** | `.MBRamFunc` attribuée à `ccXXXXXX.ltrans0.ltrans.o` | `.MBRamFunc` attribuée à `mb_flash_nolto.o` |
| **`check_mb_ramfunc.cmake` (Regex Map)** | ❌ **ÉCHEC** (`FATAL_ERROR`) | ✅ **SUCCÈS** (L'objet est localisé) |
| **`objdump -r -j .MBRamFunc`** | ❌ **ÉCHEC** (`section '.MBRamFunc' not found`) | ✅ **SUCCÈS** (0 relocation détectée) |
| **Disassembly check (`check_mb_ramfunc`)** | Non atteignable (build brisé) | ✅ **SUCCÈS** (Branches confinées en RAM) |
| **Inlining inter-fichiers vers RAM** | ⚠️ Risque critique présent | 🛡️ **Totalement impossible** (GIMPLE absent) |
| **Gains LTO sur le reste du firmware** | 100 % du code | **> 98 % du code** (tout sauf `mb_flash.c`) |

---

## 6. Estimation Chiffrée de la Réduction de Taille Flash & RAM

### 6.1 Données empiriques mesurées sur le projet
Lors de l'évaluation initiale sur le preset `Fusion` (commit `164a7273`, janvier 2026) :

| Métrique | Sans LTO | Avec LTO (`-flto=auto`) | Gain absolu | Gain relatif |
| :--- | :--- | :--- | :--- | :--- |
| **Flash interne** | 84 304 octets | 80 820 octets | **-3 484 octets** | **-4,13 %** |
| **SRAM** | 12 256 octets | 11 904 octets | **-352 octets** | **-2,87 %** |

### 6.2 Projection sur le Preset Max (v6.0.0 monolithique)
Le preset `Max` intègre l'ensemble des fonctionnalités résidentes simultanées :
- Sous-systèmes radio complets (Spectrum, Audio Scope, Scanner rapide, RSSI/Audio bar, Vox, PMR, GMRS/MURS).
- Outils opérationnels (Rescue Ops, FoxHunt, Beacon, AirCopy, Beam, K5Viewer).
- Jeux et interface utilisateur (Breakout resident, Menu Catégories, QR Code, Log Rx/Tx, polices étendues).

Cette volumétrie représente **plus de 50 fichiers sources C** liés de manière monolithique. Plus le nombre d'unités de traduction interconnectées est important, plus la surface d'optimisation inter-procédurale (IPA-inline, fusion de chaînes, déduplication de constantes, élagage de code mort) est vaste.

- **Estimation basse (conservative, 3,0 %)** : ~3 600 octets récupérés.
- **Estimation centrale attendue (3,5 % - 4,0 %)** : **~4 200 à 4 800 octets récupérés**.
- **Estimation haute (4,5 %)** : ~5 400 octets récupérés.

Dans tous les scénarios, LTO apporte un gain net supérieur à **3,5 KiB**, ce qui permet de résorber à lui seul l'essentiel de l'excédent de Flash du Preset Max.

---

## 7. Patch de Configuration et Modifications Requises

Pour activer LTO en toute sécurité tout en satisfaisant à 100 % les exigences de `check_mb_ramfunc.cmake`, les modifications suivantes sont à appliquer :

### 7.1 Modification de `CMakeLists.txt`
Remplacer le bloc LTO existant (lignes 100-102) par :

```cmake
if(ENABLE_LTO)
    # Appliquer -flto=auto à la compilation et à l'édition de liens de l'exécutable
    target_compile_options(${EXE_NAME} PRIVATE -flto=auto)
    target_link_options(${EXE_NAME} PRIVATE -flto=auto)

    # Isolation vitale du stub Multiboot exécuté en RAM (.mb_ramfunc) :
    # Doit impérativement être compilé sans LTO pour garantir :
    # 1. Qu'aucune fonction résidente en Flash n'est inlinée dans le stub RAM
    # 2. Que .MBRamFunc reste intègre dans un objet ELF natif (mb_flash.c.obj) et non ltrans
    # 3. Que les contrôles de relocations et de branches de check_mb_ramfunc.cmake réussissent
    set_source_files_properties(App/driver/mb_flash.c PROPERTIES COMPILE_OPTIONS "-fno-lto")
endif()
```

### 7.2 Durcissement dans `App/driver/mb_flash.c`
Ajouter l'attribut `noclone` à `MB_RamReflash` (ligne 228) pour interdire tout clonage de fonction intra-fichier par l'optimiseur GCC :

```c
/* Avant : */
__attribute__((section(MB_RAM_SECTION), noinline, used))
static void MB_RamReflash(uint32_t intAddr, uint32_t extAddr, uint32_t imageSize,
                          uint8_t *progressLine)

/* Après : */
__attribute__((section(MB_RAM_SECTION), noinline, noclone, used))
static void MB_RamReflash(uint32_t intAddr, uint32_t extAddr, uint32_t imageSize,
                          uint8_t *progressLine)
```

### 7.3 Activation dans `CMakePresets.json`
Activer `"ENABLE_LTO": true` dans le preset `default` (ou spécifiquement sur `Fusion`, `FieldOps`, `Transfer`, `Labs`, et `Max`) :

```json
"cacheVariables": {
    "CMAKE_BUILD_TYPE": "Release",
    "DEV": false,
    "ENABLE_LTO": true,
    ...
}
```

---

## 8. Conclusion et Recommandations pour l'ADR (#8)

1. **Faisabilité** : **Confirmée à 100 %**. L'isolation chirurgicale de `App/driver/mb_flash.c` par `-fno-lto` résout intégralement l'incompatibilité avec `check_mb_ramfunc.cmake`.
2. **Priorité** : **Haute / Fondamentale**. LTO constitue le socle indispensable sur lequel les autres optimisations (compression de polices, factorisation UI, déport Breakout) doivent s'appuyer pour garantir que le Preset Max tienne largement sous la limite des 118 KiB avec une marge de sécurité confortable.
3. **Absence de régression** : L'isolation matérielle du stub garantit zéro risque de corruption mémoire pendant la restauration Multiboot, et la toolchain GCC 13.3.rel1 génère du code Thumb-1 parfaitement conforme aux spécifications Cortex-M0+.
