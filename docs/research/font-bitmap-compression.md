# Rapport de Recherche : Faisabilité de la Compression RLE, Bit-Packing et Dérivation Procédurale pour Bitmaps et Polices

- **Ticket de référence** : Issue [#4](https://github.com/NomenAK/uv-k1-k5v3-firmware-custom/issues/4)
- **Ticket parent (Map)** : Issue [#1](https://github.com/NomenAK/uv-k1-k5v3-firmware-custom/issues/1)
- **Cible matérielle** : Quansheng UV-K1 / UV-K5 V3 — MCU Puya PY32F071xB (ARM Cortex-M0+, Flash interne 118 KiB / 120 832 octets max, SRAM 16 KiB)
- **Afficheur cible** : Écran LCD monochrome ST7565 (128x64 pixels, contrôleur SPI série, frame buffer VMA 896 octets + 128 octets status line)
- **Toolchain cible** : Arm GNU Toolchain GCC 13.3.rel1 (`arm-none-eabi-gcc`), options `-Os`, `--specs=nano.specs`
- **Statut** : Étude complétée — Décompresseur RLE/bit-packing formellement rejeté (gain net négatif ou insignifiant), dérivations architecturales et déduplications recommandées (+131 B à +695 B net Flash).

---

## 1. Résumé Exécutif

L'objectif de cette étude est d'évaluer la faisabilité technique et la rentabilité en Flash interne d'un algorithme de décompression ultra-léger (RLE, bit-packing 1-bit, tables creuses) ou de dérivations procédurales appliqué aux polices (`App/font.c`) et aux bitmaps (`App/bitmaps.c`) du firmware monolithique UV-K1 / UV-K5 V3.

Les conclusions formelles de l'audit sont les suivantes :

1. **Volume total des données graphiques en Flash** :
   - Polices complètes (`font.c`) : **2 952 octets** (avec `ENABLE_SMALL_BOLD`) ou **2 388 octets** (sans `ENABLE_SMALL_BOLD`).
   - Bitmaps complets (`bitmaps.c`) : **346 octets** au maximum (34 tableaux), dont **219 octets** actifs dans le preset `Fusion` standard et **326 octets** dans `FieldOps` / `Max`.
   - L'ensemble des actifs graphiques représente **~3,2 KiB** (soit seulement 2,7 % de la Flash interne de 118 KiB).

2. **Rejet formel de la décompression à l'exécution (RLE / Bit-packing)** :
   - **Accès aléatoire obligatoire** : Pour afficher un caractère sans décompresser toute la police en RAM (ce qui exigerait ~1,9 KiB de RAM, soit 12 % de la SRAM totale de 16 KiB, un coût intolérable), chaque glyphe doit être adressable en $O(1)$.
   - **Pénalité fatale de la table d'index** : Un adressage direct de glyphes de taille variable nécessite une table d'offsets (`uint16_t offset[94]` = 188 octets). Pour `gFontBig` (1 316 B), le payload compressé RLE est de 1 220 B ; additionné aux 188 B de table et aux ~50 B de décodeur, l'empreinte Flash passe à 1 458 B, soit une **perte nette de 142 octets en Flash** (régression).
   - Pour les bitmaps, les tableaux sont déjà minuscules (moyenne de 10,2 octets par icône). Les compresser individuellement coûte plus cher en pointeurs/code qu'en données brutes.

3. **Rejet de la dérivation procédurale pour `gFontSmallBold` (Invariant UI/UX)** :
   - L'évaluation algorithmique (`bold = norm | (norm << 1)` ou `bold[x] = norm[x] | norm[x-1]`) ne correspond au dessin réel que pour **8,5 % des glyphes** (8/94 en horizontal) et **2,1 %** (2/94 en vertical).
   - En résolution 6x8 pixels, l'épaississement naïf bouche irrémédiablement les contreformes (œilletons intérieurs de 'e', 'a', 's', 'B', '8', '0'), créant des taches noires illisibles qui violent frontalement l'invariant **"Zero UI/UX degradation"**.
   - En revanche, l'activation/désactivation contrôlée par preset de `ENABLE_SMALL_BOLD` permet une récupération propre et immédiate de **564 octets** de Flash sans décodeur ni dégradation non sollicitée.

4. **Découverte majeure : 34 % de `bitmaps.c` sont des doublons textuels de `gFontSmall`** :
   - 10 tableaux de `bitmaps.c` totalisant **118 octets** (`gFontPowerSave`, `gFontMO`, `gFontXB`, `gFontDWR`, `gFontRO`, `gFontVox`, `gFontPttOnePush`, `gFontPttClassic`, `gFontHold`, `gFontS`) ne sont que des chaînes de 2 ou 3 lettres déjà présentes dans `gFontSmall` ("PS", "MO", "XB", "DWR", "RO", "VO", "OP", "CL", "><", "S") pré-rendues avec un décalage d'un pixel.
   - Leur rendu via le moteur de police existant ou l'aliasing direct libère jusqu'à **118 octets** en Flash sans aucun décodeur.
   - La suppression du tableau nul factice `BITMAP_VFO_Empty` (7 octets de zéros) et l'aliasing de `gFontS` (6 octets) offrent un gain immédiat de **13 octets**.

---

## 2. Inventaire et Audit Précis des Actifs Graphiques (`.rodata`)

### 2.1 Polices de caractères (`App/font.c` & `App/font.h`)

| Tableau Symbole | Dimensions déclarées | Glyphes | Format par glyphe | Taille Flash (`.rodata`) | Conditions de compilation |
| :--- | :--- | :---: | :---: | :---: | :--- |
| `gFontBig` | `[95 - 1][16 - 2]` | 94 | 14 octets (7 col $\times$ 2 lignes) | **1 316 octets** | Toujours inclus |
| `gFontBigDigits` | `[11][26 - 6]` | 11 | 20 octets (10 col $\times$ 2 lignes) | **220 octets** | Toujours inclus (Terminus) |
| `gFontSmall` | `[95 - 1][6]` | 94 | 6 octets (6 col $\times$ 1 ligne) | **564 octets** | Toujours inclus |
| `gFontSmallBold` | `[95 - 1][6]` | 94 | 6 octets (6 col $\times$ 1 ligne) | **564 octets** | `#ifdef ENABLE_SMALL_BOLD` |
| `gFont3x5` | `[96][3]` | 96 | 3 octets (3 col $\times$ 5-6 bits) | **288 octets** | Toujours inclus (`#ifdef` commenté) |
| **Sous-total Polices (avec bold)** | — | **389** | — | **2 952 octets** | Presets standard (`Fusion`, `Max`, etc.) |
| **Sous-total Polices (sans bold)** | — | **295** | — | **2 388 octets** | Si `ENABLE_SMALL_BOLD=OFF` |

*Notes d'implémentation historiques* :
- `gFontBig` : Les colonnes vides centrale et finale (0x00) ainsi que le glyphe espace (ASCII 32) avaient déjà été élagués par les mainteneurs DualTachyon/OneOfEleven, ramenant chaque glyphe de 16 à 14 octets (gain historique de $94 \times 2 = 188$ octets).
- `gFontBigDigits` : 6 colonnes vides latérales ont été supprimées (`26 - 6 = 20` octets par chiffre). Seule la police Terminus est compilée (les variantes "original" et "VCR" sont sous `#if 0`).

### 2.2 Bitmaps et Icônes de Statut (`App/bitmaps.c` & `App/bitmaps.h`)

L'audit complet du fichier `App/bitmaps.c` révèle 34 tableaux non commentés actifs dans le code :

| Symbole Bitmap | Dimensions / Nb octets | Rôle Fonctionnel | Condition de compilation |
| :--- | :---: | :--- | :--- |
| `BITMAP_BatteryLevel` | 2 octets | Barres internes jauge batterie | Toujours inclus |
| `BITMAP_BatteryLevel1` | 17 octets | Contour corps de batterie | Toujours inclus |
| `BITMAP_USB_C` | 9 octets | Icône charge USB-C | Toujours inclus |
| `BITMAP_VFO_Default` | 7 octets | Flèche pleine sélection VFO | Toujours inclus |
| `BITMAP_VFO_NotDefault` | 7 octets | Flèche creuse VFO secondaire | Toujours inclus |
| `BITMAP_VFO_Empty` | 7 octets | Zone vide VFO (7 octets 0x00) | Toujours inclus |
| `BITMAP_VFO_Lock` | 7 octets | Cadenas verrouillage canal/VFO | Toujours inclus |
| `BITMAP_compand` | 6 octets | Symbole compresseur audio | Toujours inclus |
| `BITMAP_PowerUser` | 3 octets | Curseur menu utilisateur expert | Toujours inclus |
| `gFontF` | 9 octets | Touche fonction [F] active | Toujours inclus |
| `gFontKeyLock` | 9 octets | Cadenas statut clavier verrouillé | Toujours inclus |
| `gFontLight` | 9 octets | Ampoule rétroéclairage actif | Toujours inclus |
| `gFontLightOff` | 9 octets | Ampoule rétroéclairage coupé | Toujours inclus |
| `gFontMute` | 12 octets | Haut-parleur barré (mute) | Toujours inclus |
| `gFontS` | 6 octets | Indicateur scan / Roger [S] | Toujours inclus |
| `gFontHold` | 10 octets | Flèches `><` double veille hold | Toujours inclus |
| `gFontPowerSave` | 12 octets | Lettres "PS" (Power Save) | Toujours inclus |
| `gFontPttOnePush` | 12 octets | Lettres "OP" (One Push PTT) | Toujours inclus |
| `gFontPttClassic` | 12 octets | Lettres "CL" (Classic PTT) | Toujours inclus |
| `gFontXB` | 12 octets | Lettres "XB" (Cross-Band) | Toujours inclus |
| `gFontMO` | 12 octets | Lettres "MO" (Main Only) | Toujours inclus |
| `gFontDWR` | 18 octets | Lettres "DWR" (Dual Watch Repeater) | Toujours inclus |
| `gFontVox` | 12 octets | Lettres "VO" (VOX actif) | `ENABLE_VOX` |
| `gFontRO` | 12 octets | Lettres "RO" (Rescue Ops) | `ENABLE_FEAT_F4HWN_RESCUE_OPS` |
| `BITMAP_NOAA` | 12 octets | Symbole météo "WX" | `ENABLE_NOAA` |
| `BITMAP_FoxHuntSignal` | 10 octets | Signal sonore balise FoxHunt | `ENABLE_FEAT_F4HWN_FOXHUNT` / `BEACON` |
| `BITMAP_FoxHuntSpeaker` | 10 octets | Haut-parleur station FoxHunt | `ENABLE_FEAT_F4HWN_FOXHUNT` / `BEACON` |
| `BITMAP_FoxHuntUp` | 11 octets | Triangle indicateur tendance + | `ENABLE_FEAT_F4HWN_FOXHUNT` / `BEACON` |
| `BITMAP_FoxHuntDown` | 11 octets | Triangle indicateur tendance - | `ENABLE_FEAT_F4HWN_FOXHUNT` / `BEACON` |
| `BITMAP_FoxHuntFlat` | 11 octets | Barre indicateur tendance = | `ENABLE_FEAT_F4HWN_FOXHUNT` / `BEACON` |
| `BITMAP_FoxHuntBars` | 11 octets | Histogramme S-mètre FoxHunt | `ENABLE_FEAT_F4HWN_FOXHUNT` / `BEACON` |
| `BITMAP_FoxHuntGraph` | 15 octets | Sinusoïde historique signal | `ENABLE_FEAT_F4HWN_FOXHUNT` / `BEACON` |
| `BITMAP_FoxHuntTx` | 16 octets | Icône émission balise TX | `ENABLE_FEAT_F4HWN_FOXHUNT` / `BEACON` |
| `BITMAP_CurrentIndicator` | 8 octets | Flèche menu stock | `!ENABLE_CUSTOM_MENU_LAYOUT` |
| **Sous-total Bitmaps (Max théorique)** | — | **34 tableaux** | — | **346 octets** |

### 2.3 Empreinte selon les Presets de Firmware

| Preset Firmware | Polices Flash | Bitmaps Flash | Total Actifs Graphiques | Part de la Flash interne (118 KiB) |
| :--- | :---: | :---: | :---: | :---: |
| **Custom** | 2 952 B | 219 B | **3 171 B** | 2,62 % |
| **Fusion** | 2 952 B | 219 B | **3 171 B** | 2,62 % |
| **FieldOps** | 2 952 B | 326 B | **3 278 B** | 2,71 % |
| **Labs** | 2 952 B | 231 B | **3 183 B** | 2,63 % |
| **Max** (toutes options) | 2 952 B | 338 B | **3 290 B** | 2,72 % |
| **Max (avec reco SmallBold=OFF)** | 2 388 B | 338 B | **2 726 B** | 2,26 % |

---

## 3. Analyse Statistique : Sparsité, Redondance et Entropie

Une analyse approfondie du contenu binaire des tableaux a été menée pour mesurer la compressibilité théorique et réelle des données.

### 3.1 Distribution de la Sparsité (Octets et Bits Nuls)

| Actif Graphique | Taille Totale | Octets Nuls (0x00) | Ratio Octets Nuls | Bits Nuls ('0') | Ratio Bits Nuls | Entropie Shannon (bits/octet) |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: |
| `gFontBig` | 1 316 B | 311 | 23,6 % | 6 501 / 10 528 | 61,7 % | 4,36 |
| `gFontBigDigits` | 220 B | 18 | 8,2 % | 714 / 1 760 | 40,6 % | 4,38 |
| `gFontSmall` | 564 B | 107 | 19,0 % | 2 756 / 4 512 | 61,1 % | 4,93 |
| `gFontSmallBold` | 564 B | 69 | 12,2 % | 1 873 / 4 512 | 41,5 % | 5,03 |
| `gFont3x5` | 288 B | 22 | 7,6 % | 1 301 / 2 304 | 56,5 % | 4,60 |
| `bitmaps.c` (Global) | 346 B | 41 | 11,8 % | 1 530 / 2 768 | 55,3 % | 4,82 |
| **Ensemble Graphique** | **3 298 B** | **568** | **17,2 %** | **14 675 / 26 384** | **55,6 %** | **4,68** |

### 3.2 Redondance des Colonnes et Asymétrie Spatiale

L'analyse colonne par colonne montre des propriétés structurelles remarquables :

1. **`gFontSmall` (largeur 6 pixels)** :
   - La **colonne 5** (dernière colonne à droite) est égale à `0x00` pour **55 glyphes sur 94** (58,5 %). Les 39 glyphes utilisant la 6e colonne sont les chiffres ('0'-'9'), les majuscules ('A'-'Z'), et quelques symboles larges ('%', '&', '/', '\\', '_', 'q'). Pour la totalité des minuscules 'a' à 'z' (sauf 'q'), la 6e colonne est rigoureusement vide.
   - La **colonne 0** (première colonne à gauche) est égale à `0x00` pour **21 glyphes sur 94** (22,3 %).

2. **`gFontSmallBold`** :
   - L'épaississement des traits réduit les colonnes vides : la colonne 5 n'est nulle que pour 31/94 glyphes (33,0 %), et la colonne 0 pour 18/94 (19,1 %).

3. **`gFontBig` (largeur 7 colonnes $\times$ 2 rangées)** :
   - La rangée inférieure (lignes de pied de caractère) comporte significativement plus de zéros (28,0 %) que la rangée supérieure (19,3 %).
   - Les colonnes latérales 0 et 6 concentrent les zéros : colonne 0 (31,9 % top, 39,4 % bottom), colonne 6 (22,3 % top, 36,2 % bottom).

4. **`gFont3x5` (largeur 3 colonnes $\times$ 5-6 bits)** :
   - **Bits 6 et 7** : Sur les 288 octets du tableau, les bits 6 et 7 valent **0 dans 100 % des cas**.
   - **Bit 5** : N'est utilisé que par **5 glyphes** sur 96 ('g', 'j', 'p', 'q', 'y' — les lettres avec jambage descendant). Les 91 autres glyphes tiennent strictement sur 5 bits verticaux.

---

## 4. Évaluation Approfondie des Algorithmes de Compression

### 4.1 Compression par Plages (RLE / PackBits)

Deux modes d'application du RLE ont été simulés et mesurés directement sur les tableaux extraits du code source.

#### Mode A : Compression en Flux Continu (Streaming)

Dans ce mode, la totalité d'un tableau de police est compressée en un seul flux continu.

| Actif | Taille Originale | RLE PackBits | Gain Brut | RLE Zero-Run | Gain Brut |
| :--- | :---: | :---: | :---: | :---: | :---: |
| `gFontBig` | 1 316 B | 1 199 B | +117 B (8,9 %) | 1 211 B | +105 B |
| `gFontSmall` | 564 B | 535 B | +29 B (5,1 %) | 583 B | -19 B (expansion) |
| `gFontSmallBold` | 564 B | 552 B | +12 B (2,1 %) | 577 B | -13 B (expansion) |
| `gFont3x5` | 288 B | 287 B | +1 B (0,3 %) | 302 B | -14 B (expansion) |
| **Total Polices** | **2 732 B** | **2 573 B** | **+159 B (5,8 %)** | **2 673 B** | **+59 B** |

**Le verdict d'inapplicabilité du Mode A** :
Le gain théorique brut de +159 octets est **inutilisable en pratique**. Pour afficher une chaîne arbitraire (ex: `"145.500"` ou `"MENU"`), le code doit pouvoir adresser n'importe quel caractère à l'indice $c$ en temps constant $O(1)$. Un flux compressé continu impose :
- Soit de décompresser la police complète en RAM au boot : or la RAM totale du MCU n'est que de 16 KiB (avec seulement ~1,5 à 2 KiB de mémoire libre selon les configurations). Consommer **1,88 KiB à 2,73 KiB de RAM** pour y mettre un cache de polices provoquerait un débordement immédiat de pile (Stack Overflow) et le plantage du firmware.
- Soit de reparcourir le flux depuis le début à chaque caractère affiché : la complexité temporelle passerait de $O(1)$ à $O(N \cdot L)$, bloquant l'affichage pendant plusieurs dizaines de millisecondes et violant l'invariant d'absence de régression de timing.

#### Mode B : Compression par Glyphe avec Accès Aléatoire $O(1)$

Pour préserver l'accès $O(1)$ sans utiliser de RAM tampon globale, chaque glyphe $c \in [0..93]$ doit être compressé indépendamment, et une table d'indexation Flash (offsets relatifs) doit être fournie.

| Actif | Taille Originale | Payload RLE cumulé | Table d'Offsets Flash | Total Flash Requis | Bilan Flash Net |
| :--- | :---: | :---: | :---: | :---: | :---: |
| `gFontBig` | 1 316 B | 1 220 B | 188 B (`uint16_t[94]`) | 1 408 B | **-92 B (PERTE)** |
| `gFontSmall` | 564 B | 556 B | 188 B (`uint16_t[94]`) | 744 B | **-180 B (PERTE)** |
| `gFontSmallBold` | 564 B | 557 B | 188 B (`uint16_t[94]`) | 745 B | **-181 B (PERTE)** |
| `gFont3x5` | 288 B | 284 B | 192 B (`uint16_t[96]`) | 476 B | **-188 B (PERTE)** |
| **Total** | **2 732 B** | **2 617 B** | **756 B** | **3 373 B** | **-641 B (PERTE MAJEURE)** |

*Conclusion mathématique incontournable* : Les glyphes d'une police 6x8 ou 14x8 sont beaucoup trop courts (6 à 14 octets) pour amortir le coût d'un métadonnée d'offset (2 octets par glyphe). **La compression RLE par glyphe augmente la taille en Flash de 641 octets au lieu de la réduire.**

### 4.2 Bit-Packing 1-bit / N-bit

#### Architecture d'affichage ST7565 et Bit-Packing vertical natif
L'écran LCD ST7565 est adressé par colonnes et par pages de 8 pixels verticaux. Chaque octet envoyé sur le bus SPI représente une tranche verticale de 8 pixels :
- Bit 0 : pixel supérieur
- Bit 7 : pixel inférieur

Cette organisation spatiale signifie que **les bitmaps et polices du firmware sont DÉJÀ encodés en bit-packing natif 1-bit par pixel dans la dimension verticale**. Il n'y a aucun octet gaspillé pour représenter des pixels individuels (contrairement à un écran RVB où un pixel prendrait 16 ou 24 bits).

#### Cas d'étude : Bit-Packing sur `gFont3x5` (3 colonnes de 5-6 bits)
Comme observé en section 3.2, `gFont3x5` utilise actuellement 3 octets par glyphe (24 bits), dont seuls 5 ou 6 bits par colonne sont significatifs.
- En packant chaque glyphe sur 18 bits (3 colonnes $\times$ 6 bits) de manière continue :
  $$\text{Taille packée} = \frac{96 \times 18 \text{ bits}}{8} = 216 \text{ octets (au lieu de 288 B)}$$
  $$\text{Gain brut sur les données} = 288 - 216 = 72 \text{ octets}$$
- Coût d'implémentation du décodeur :
  Pour extraire un glyphe 18 bits à l'indice $c$ non aligné sur une frontière d'octet :
  ```c
  uint32_t bit_pos = c * 18;
  uint32_t byte_pos = bit_pos >> 3;
  uint32_t shift = bit_pos & 7;
  uint32_t raw = pData[byte_pos] | (pData[byte_pos + 1] << 8) | (pData[byte_pos + 2] << 16);
  uint32_t glyph18 = (raw >> shift) & 0x3FFFF;
  uint8_t col0 = glyph18 & 0x3F;
  uint8_t col1 = (glyph18 >> 6) & 0x3F;
  uint8_t col2 = (glyph18 >> 12) & 0x3F;
  ```
  Le code machine Thumb-1 généré par GCC 13 (`-Os`) pour cette routine nécessite environ **48 octets d'instructions Flash**.
- **Bilan Flash net** : $72 \text{ B (gain données)} - 48 \text{ B (code décodeur)} = \mathbf{+24 \text{ octets}}$.
- **Bilan CPU** : Passer d'un simple `ldrb` (2 cycles d'horloge) à une routine de 35-40 cycles par caractère. Un gain de 24 octets ne justifie en aucun cas cette pénalité de complexité.

#### Cas d'étude : `BITMAP_QR_GitHub_Compressed` dans `welcome.c`
Un exemple de bit-packing déjà présent dans le repo confirme cette analyse :
Dans `App/ui/welcome.c`, les QR codes (33x33 pixels) utilisaient initialement 5 pages de 33 octets ($5 \times 33 = 165$ octets), où la 5e page n'utilisait que le bit 0 (la 33e ligne).
L'auteur a compacté la 5e ligne sous forme de 33 bits consécutifs dans 5 octets (`ceil(33/8)`), ramenant la taille de 165 à 137 octets (**gain net de 28 octets** par QR code, soit 56 octets pour les deux QR codes Wiki et Repo).
Ce bit-packing fonctionne précisément parce que le QR code est une matrice 2D statique affichée en bloc unique à l'écran de démarrage, sans exigence d'accès aléatoire $O(1)$ caractère par caractère.

### 4.3 Tables Creuses (Sparse Glyph Columns)

Pour `gFontSmall`, 55 glyphes sur 94 n'ont que 5 colonnes utiles au lieu de 6.
Peut-on supprimer la 6e colonne des glyphes étroits ?
- Si on stocke une table de largeurs de glyphes (`uint8_t width[94]` = 94 octets) + une table d'offsets (`uint16_t offset[94]` = 188 octets) :
  $$\text{Données colonnes} = (55 \times 5) + (39 \times 6) = 275 + 234 = 509 \text{ octets}$$
  $$\text{Taille totale Flash} = 509 + 94 + 188 = \mathbf{791 \text{ octets}}$$
  Par rapport aux **564 octets** d'origine, cela représente une **perte de 227 octets en Flash**.

---

## 5. Dérivations Procédurales vs Tables Dédiées

La question posée dans l'issue #4 suggère d'examiner si une dérivation algorithmique de la police grasse (ex: `bold = normal | (normal << 1)`) permettrait d'éliminer le tableau `gFontSmallBold` (564 octets).

### 5.1 Confrontation Algorithmique vs Typographie Réelle

Une comparaison bit à bit a été exécutée entre `gFontSmall` et `gFontSmallBold` pour chaque glyphe :

```text
Char '!':  norm: [0x00, 0x00, 0x5E, 0x00, 0x00, 0x00]
           bold: [0x00, 0x00, 0x5E, 0x5E, 0x00, 0x00]   <-- duplication horizontale
Char '+':  norm: [0x08, 0x08, 0x3E, 0x08, 0x08, 0x00]
           bold: [0x18, 0x18, 0x7E, 0x7E, 0x18, 0x18]   <-- expansion 2D (H et V)
Char '-':  norm: [0x00, 0x08, 0x08, 0x08, 0x08, 0x00]
           bold: [0x00, 0x18, 0x18, 0x18, 0x18, 0x00]   <-- épaississement vertical seul
Char '0':  norm: [0x3E, 0x41, 0x41, 0x41, 0x41, 0x3E]
           bold: [0x3E, 0x7F, 0x63, 0x63, 0x7F, 0x3E]   <-- dessin typographique manuel
```

1. **Correspondance avec un OR horizontal (`norm[x] | norm[x-1]`)** :
   - Correspondance exacte : **8 / 94 glyphes** (8,5 %).
   - Pour les 86 autres glyphes, les traits verticaux s'épaississent mais les traits horizontaux restent à 1 pixel. Les lettres étroites comme 'I', 'l', '1' deviennent démesurément larges.

2. **Correspondance avec un OR vertical (`norm[x] | (norm[x] << 1)`)** :
   - Correspondance exacte : **2 / 94 glyphes** (2,1 %).
   - Pour les 92 autres glyphes, les barres horizontales s'épaississent mais les montants verticaux restent fins.

3. **Application combinée (OR horizontal + OR vertical)** :
   - En résolution 6x8 pixels monochrome, un glyphe comporte typiquement un œilleton de 2 à 3 pixels de haut et 2 à 3 pixels de large.
   - Appliquer un OR bidirectionnel remplit entièrement le centre des caractères critiques :
     - '0' (chiffre zéro) devient un rectangle noir plein, impossible à distinguer de '8' ou 'B'.
     - 'e', 'a', 'o', 's' perdent toute lisibilité et deviennent des pavés noirs informes.
     - '%' et '&' deviennent des taches compactes indéchiffrables.

### 5.2 Respect de l'Invariant "Zero UI/UX Degradation"

Le cahier des charges du projet fixe une règle inviolable : **Zero UI/UX degradation**.
Remplacer `gFontSmallBold` par un filtre procédural temps réel produirait une régression visuelle flagrante et immédiatement visible par les utilisateurs dans les menus et l'écran principal. Cette piste algorithmique est donc **rejetée pour non-conformité fonctionnelle**.

### 5.3 L'Alternative Déjà Prévue : `ENABLE_SMALL_BOLD`

Le code source de `App/ui/helper.c` intègre déjà un mécanisme d'exclusion propre et testé :
```c
void UI_PrintStringSmallBold(const char *pString, uint8_t Start, uint8_t End, uint8_t Line)
{
#ifdef ENABLE_SMALL_BOLD
    const uint8_t *font = (uint8_t *)gFontSmallBold;
    const uint8_t char_width = ARRAY_SIZE(gFontSmallBold[0]);
#else
    const uint8_t *font = (uint8_t *)gFontSmall;
    const uint8_t char_width = ARRAY_SIZE(gFontSmall[0]);
#endif
    UI_PrintStringSmall(pString, Start, End, Line, char_width, font);
}
```

Dans `CMakePresets.json`, `ENABLE_SMALL_BOLD` est actuellement positionné à `true` dans tous les presets.
Dans une configuration où la Flash est sous tension critique (notamment pour le **Preset Max** afin de tenir sous les 118 KiB), positionner `"ENABLE_SMALL_BOLD": false` dans ce preset spécifique :
- Supprime immédiatement le tableau `gFontSmallBold` (564 octets en moins en Flash).
- Redirige automatiquement tous les affichages en gras vers `gFontSmall` normal (parfaitement lisible, aucun artefact visuel).
- **Gain net garanti : 564 octets de Flash**, avec un impact visuel minime (texte normal au lieu de gras) et sans ajouter une seule ligne de code décodeur.

---

## 6. Analyse de Performance Temps Réel et Latence CPU (Cortex-M0+ @ 48 MHz)

### 6.1 Coût de Rendu Actuel (Accès Mémoire Direct)

Sur le microcontrôleur Puya PY32F071xB cadencé à 48 MHz (1 cycle = 20,83 ns) :
- Les fonctions de dessin de texte (`UI_PrintString`, `UI_PrintStringSmall`, `UI_DisplayFrequency`) utilisent des copies directes `memcpy` depuis la Flash vers le FrameBuffer en RAM (`gFrameBuffer` / `gStatusLine`).
- En Cortex-M0+ (ARMv6-M), une copie de 6 octets optimisée par GCC (`ldmia` / `stmia` ou séquences de `ldrh`/`strh`) s'exécute en **10 à 15 cycles d'horloge** (~0,25 à 0,31 µs).
- Afficher une ligne complète de texte (ex: 16 caractères) prend environ **200 cycles** (~4,1 µs).

### 6.2 Coût Simulé avec Décompression à la Volée

Si un décodeur RLE ou bit-packing était inséré dans la boucle d'affichage de caractère :
- Accès à la table d'offsets Flash : 6 à 10 cycles.
- Boucle de décodage d'octets / décalage de bits : 25 à 45 cycles par colonne.
- Pour un caractère 14 octets (`gFontBig`) : ~350 à 450 cycles.
- Pour une chaîne de 16 caractères : ~6 400 cycles (~133 µs).

### 6.3 Impact sur le Rafraîchissement ST7565 et les Interruptions RF

1. **Latence globale d'affichage** :
   - Le transfert SPI d'une page LCD complète (128 octets) à 750 kHz (prescaler SPI DIV64 sur APB2 48 MHz) prend :
     $$T_{\text{page}} = \frac{128 \times 8 \text{ bits}}{750 \text{ kHz}} \approx 1,36 \text{ ms}$$
   - Le transfert de l'écran complet (7 pages + status line = 1 024 octets) prend **~10,9 ms**.
   - Un surcoût CPU de 133 µs pour décoder du texte représente seulement **1,2 % du temps de transfert SPI**. D'un point de vue strictement temporel, la décompression n'introduirait pas de scintillement visuel perceptible.

2. **Impact RF et gigue d'interruption (Jitter)** :
   - Les tâches radio critiques (polling BK4819, décodage sous-audible CTCSS/DCS dans `App/action.c`, temporisations squelch) s'exécutent en interruption ou dans la boucle principale.
   - L'allongement de la phase de calcul dans `UI_PrintString` retarde le retour à la boucle de traitement RF.
   - Surtout, ce compromis serait concédé en échange d'un **gain mémoire nul ou négatif**, ce qui rend l'opération doublement injustifiable.

---

## 7. Recommandations Concrètes et Estimations de Gains

Au lieu de décompresser les polices avec une pénalité nette, l'audit approfondi du code a mis en lumière **trois opportunités concrètes et élégantes d'optimisation en Flash interne**.

### 7.1 Opportunité 1 : Déduplication des Bitmaps Multi-Lettres de Statut (Gain : ~118 octets)

Dans `App/bitmaps.c`, 10 tableaux sont des rendus statiques de lettres majuscules déjà présentes dans `gFontSmall` :

| Tableau dans `bitmaps.c` | Taille Flash | Contenu textuel | Équivalent dans `gFontSmall` |
| :--- | :---: | :---: | :--- |
| `gFontPowerSave` | 12 octets | "PS" | `gFontSmall['P'-33]` et `gFontSmall['S'-33]` |
| `gFontPttOnePush` | 12 octets | "OP" | `gFontSmall['O'-33]` et `gFontSmall['P'-33]` |
| `gFontPttClassic` | 12 octets | "CL" | `gFontSmall['C'-33]` et `gFontSmall['L'-33]` |
| `gFontXB` | 12 octets | "XB" | `gFontSmall['X'-33]` et `gFontSmall['B'-33]` |
| `gFontMO` | 12 octets | "MO" | `gFontSmall['M'-33]` et `gFontSmall['O'-33]` |
| `gFontDWR` | 18 octets | "DWR" | `gFontSmall['D']`, `['W']`, `['R']` |
| `gFontRO` | 12 octets | "RO" | `gFontSmall['R']`, `['O']` |
| `gFontVox` | 12 octets | "VO" | `gFontSmall['V']`, `['O']` |
| `gFontHold` | 10 octets | "><" | `gFontSmall['>']`, `['<']` |
| `gFontS` | 6 octets | "S" | Identique bit-à-bit à `gFontSmall['S'-33]` |
| **Total doublons textuels** | **118 octets** | — | — |

**Recommandation d'ingénierie** :
- Pour `gFontS` (6 octets) : Remplacer la définition dans `bitmaps.c` par un alias ou une macro pointant directement sur le caractère 'S' de `gFontSmall` :
  ```c
  #define gFontS (gFontSmall['S' - ' ' - 1])
  ```
  *(Gain immédiat : **6 octets** en Flash, 0 risque)*.
- Pour les indicateurs de statut 2-lettres ("PS", "MO", "XB", etc.) dans `App/ui/status.c` :
  Au lieu de copier un tableau fixe via `memcpy(line + x, gFontPowerSave, 12)`, appeler directement la primitive de rendu existante `UI_PrintStringSmallBufferNormal("PS", line + x)`.
  Le code de la primitive est déjà compilé et amorti dans le binaire ; supprimer ces tableaux dédiés libère jusqu'à **112 octets** nets en Flash.

### 7.2 Opportunité 2 : Élimination du Tableau Factice `BITMAP_VFO_Empty` (Gain : 7 octets)

Dans `App/bitmaps.c:235-244` :
```c
const uint8_t BITMAP_VFO_Empty[7] = {
    0b00000000, 0b00000000, 0b00000000, 0b00000000,
    0b00000000, 0b00000000, 0b00000000
};
```
Utilisé dans `App/ui/main.c:1028-1029` :
```c
for (uint8_t i = 0; i < sizeof(BITMAP_VFO_Empty); i++)
    p_line0[i] = (p_line0[i] & 0x80) | BITMAP_VFO_Empty[i];
```
Puisque `BITMAP_VFO_Empty[i]` est toujours égal à 0, l'expression `(p_line0[i] & 0x80) | 0` équivaut strictement à `p_line0[i] &= 0x80`.
**Recommandation** : Supprimer le tableau `BITMAP_VFO_Empty` de `bitmaps.c` et simplifier la boucle dans `main.c`.
*(Gain : **7 octets** en Flash, code plus clair)*.

### 7.3 Opportunité 3 : Conditionnement de `ENABLE_SMALL_BOLD` sur le Preset Max (Gain : 564 octets)

Le preset `Max` cumule toutes les fonctionnalités de secours, scan, FoxHunt, balises et jeux, ce qui le place sous une pression extrême par rapport à la limite de 118 KiB.
- `ENABLE_SMALL_BOLD` consomme **564 octets** dans `.rodata`.
- En désactivant `ENABLE_SMALL_BOLD` uniquement dans le preset `Max` (`CMakePresets.json`) :
  ```json
  "name": "Max",
  "cacheVariables": {
      "ENABLE_SMALL_BOLD": false
  }
  ```
  Le firmware économise **564 octets instantanément**, tout en conservant une interface parfaitement lisible et opérationnelle grâce au fallback natif de `UI_PrintStringSmallBold`.

---

## 8. Synthèse Finale et Feuille de Route d'Implémentation

### 8.1 Tableau Comparatif des Options Étudiées

| Solution Étudiée | Gain Flash Brut (Données) | Surcoût Code / Métadonnées | Gain Flash Net | Impact CPU / Latence | Statut & Recommandation |
| :--- | :---: | :---: | :---: | :---: | :--- |
| **RLE Streaming Global** | +159 B | +120 B (décodeur) | +39 B | Impossible (exige 1,9 KiB RAM) | **REJETÉ (RAM saturée)** |
| **RLE par Glyphe ($O(1)$)** | +115 B | +756 B (index) + ~100 B code | **-741 B (Régression)** | +30 µs / ligne | **REJETÉ (Perte Flash nette)** |
| **Bit-Packing sur `gFont3x5`** | +72 B | +48 B (décodeur) | **+24 B** | $\times 20$ cycles / char | **NON RETENU (ROI dérisoire)** |
| **Tables Creuses Variable-Width** | +55 B | +282 B (index) | **-227 B (Régression)** | +15 µs / ligne | **REJETÉ (Perte Flash)** |
| **Dérivation procédurale Bold** | +564 B | +40 B code | +524 B | Contreformes bouchées | **REJETÉ (Régression UI/UX)** |
| **Aliasing direct `gFontS`** | +6 B | 0 B | **+6 B** | Aucun (0 cycle) | **RECOMMANDÉ IMMÉDIAT** |
| **Suppression `BITMAP_VFO_Empty`** | +7 B | 0 B (simplifie code) | **+7 B** | Gain cycles CPU | **RECOMMANDÉ IMMÉDIAT** |
| **Déduplication Statut `bitmaps.c`** | +118 B | ~15 B d'appels | **~+103 B** | Négligeable (< 1 µs) | **RECOMMANDÉ PHASE 2** |
| **Désactivation `ENABLE_SMALL_BOLD` (Max)** | +564 B | 0 B | **+564 B** | Aucun (0 cycle) | **RECOMMANDÉ PRESET MAX** |

### 8.2 Plan d'Action Recommandé pour l'Équipe

1. **Aucun investissement en décompresseurs complexes** : Clôturer la piste des algorithmes de compression RLE/bit-packing pour les polices et petits bitmaps. Les mathématiques démontrent que la granularité des glyphes (6 à 14 octets) est trop fine pour amortir le moindre système d'indexation.
2. **Implémentation des micro-gains immédiats (Quick Wins, +13 octets)** :
   - Supprimer `BITMAP_VFO_Empty` et remplacer par masquage direct.
   - Remplacer le tableau `gFontS` par un pointeur/macro vers `gFontSmall['S' - ' ' - 1]`.
3. **Rationalisation des bitmaps de statut (+103 octets)** :
   - Migrer l'affichage des balises 2-lettres ("PS", "MO", "XB", etc.) vers le moteur de police `UI_PrintStringSmall` dans `status.c`.
4. **Levier majeur pour le Preset Max (+564 octets)** :
   - Désactiver `ENABLE_SMALL_BOLD` sur le preset `Max` afin de dégager la marge nécessaire sous les 118 KiB sans sacrifice fonctionnel.
