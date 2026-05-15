# W25Q Flash OctoSPI Driver (RJ-ELEKTRONIK)

Une bibliothèque C légère, thread-safe et hautement optimisée pour les mémoires Flash Winbond (série W25Qxx, ex: W25Q64) utilisant le périphérique **OctoSPI** des microcontrôleurs STM32 (séries STM32H7, etc.) via l'HAL STMicroelectronics.

---

## 🚀 Fonctionnalités

- **Support Dual Mode** : Bascule facile entre le mode standard SPI (1 ligne) et le mode haute vitesse Quad-SPI (4 lignes).
- **Memory-Mapped Mode** : Configuration du mappage direct en mémoire permettant d'exécuter du code (XIP) ou de lire la Flash comme de la RAM interne (Région `0x90000000` ).
- **Algorithme d'Usure Optimisé (`W25Q_UpdateLive`)** : Inspecte le contenu physique avant écriture. Évite un cycle complet d'effacement (*Erase*) si les bits passent uniquement de 1 à 0, multipliant la durée de vie de la puce.
- **Sécurisation d'état** : Protection automatique de la machine d'état de l'OctoSPI lors des transitions entre le mode indirect et le mode mappé.

---

## 📁 Structure du Projet

```text
Core/
├── Inc/
│   └── w25q_flash_RJ-ELEKTRONIK.h  # Définitions, opcodes Winbond et prototypes
└── Src/
    └── w25q_flash_RJ-ELEKTRONIK.c  # Implémentation du pilote et de la logique
```

---

## ⚙️ Configuration Principale (`.h`)

Pour activer le mode Quad-SPI (4 lignes), assurez-vous que la macro `W25Q_READ_MODE` est configurée ainsi dans votre fichier `w25q_flash_RJ-ELEKTRONIK.h` :

```c
#define W25Q_MODE_1_LINE        0
#define W25Q_MODE_4_LINES       1

// Mode actif : 4 lignes (Quad Mode)
#define W25Q_READ_MODE          W25Q_MODE_4_LINES 
```

*Note sur l'adresse matérielle : Le pilote détecte automatiquement si vous utilisez `OCTOSPI1` (`0x90000000`) ou `OCTOSPI2` (`0x70000000`) sur l'architecture STM32H7.*

---

## 📘 Guide de Référence des Fonctions & Exemples

### 1. `W25Q_Init`
Initialise la mémoire Flash au démarrage du programme.
```c
if (W25Q_Init(&hospi2) == HAL_OK) {
    // La mémoire Flash est prête à être utilisée
} else {
    // Erreur d'initialisation (problème de câblage ou alimentation)
    Error_Handler();
}
```

### 2. `W25Q_ReadJEDECID`
Récupère l'identifiant fabricant et du modèle sous forme d'un entier unique 32-bits.
```c
uint32_t id = W25Q_ReadJEDECID(&hospi2);
// Pour un W25Q64JV, 'id' doit être égal à 0xEF4017
if (id == 0xEF4017) {
    // Composant Winbond W25Q64 identifié avec succès
}
```

### 3. `W25Q_ReadID`
Lit l'identifiant de la puce sous forme brute dans un tableau de 3 octets.
```c
uint8_t id_buffer[3];
W25Q_ReadID(&hospi2, id_buffer);

// id_buffer[0] = Code fabricant (0xEF pour Winbond)
// id_buffer[1] = Type de mémoire
// id_buffer[2] = Capacité de la mémoire
```

### 4. `W25Q_ReadUniqueID`
Lit le numéro de série unique de 64 bits (8 octets) gravé en usine.
```c
uint8_t uid[8];
W25Q_ReadUniqueID(&hospi2, uid);
// 'uid' contient maintenant l'identifiant unique de la puce
```

### 5. `W25Q_Read`
Lit des données brutes en mode indirect à partir d'une adresse spécifique.
```c
uint8_t buffer_lecture[10];
uint32_t adresse_flash = 0x00001000; // Adresse de départ

if (W25Q_Read(&hospi2, adresse_flash, buffer_lecture, 10) == HAL_OK) {
    // Les 10 octets ont été lus et stockés dans 'buffer_lecture'
}
```

### 6. `W25Q_Write`
Écrit des données brutes en gérant automatiquement le découpage par pages de 256 octets (la zone cible doit être préalablement effacée).
```c
uint8_t donnees_a_ecrire[5] = {0x01, 0x02, 0x03, 0x04, 0x05};
uint32_t adresse_flash = 0x00002000;

if (W25Q_Write(&hospi2, adresse_flash, donnees_a_ecrire, 5) == HAL_OK) {
    // Les données ont été écrites physiquement
}
```

### 7. `W25Q_EraseSector`
Efface un secteur complet de 4 Ko (4096 octets). Remet tous les bits à 1 (`0xFF`).
```c
uint32_t adresse_secteur = 0x00000000; // Doit être aligné sur 4Ko (ex: 0x0000, 0x1000, 0x2000...)

if (W25Q_EraseSector(&hospi2, adresse_secteur) == HAL_OK) {
    // Le secteur de 4Ko est entièrement effacé (rempli de 0xFF)
}
```

### 8. `W25Q_EraseBlock32K`
Efface un bloc de 32 Ko (32768 octets). Plus rapide que d'effacer 8 secteurs individuellement.
```c
uint32_t adresse_bloc32 = 0x00000000; // Doit être aligné sur 32Ko

if (W25Q_EraseBlock32K(&hospi2, adresse_bloc32) == HAL_OK) {
    // Le bloc de 32Ko est vide
}
```

### 9. `W25Q_EraseBlock64K`
Efface un gros bloc de 64 Ko (65536 octets). Idéal pour vider rapidement de grandes zones de stockage.
```c
uint32_t adresse_bloc64 = 0x00010000; // Doit être aligné sur 64Ko

if (W25Q_EraseBlock64K(&hospi2, adresse_bloc64) == HAL_OK) {
    // Le bloc de 64Ko est vide
}
```

### 10. `W25Q_EraseChip`
Efface l'intégralité de la puce Flash. Opération bloquante longue (peut prendre plusieurs secondes).
```c
if (W25Q_EraseChip(&hospi2) == HAL_OK) {
    // La totalité de la mémoire contient uniquement des 0xFF
}
```

### 11. `W25Q_EnableQuadMode`
Active le bit QE (Quad Enable) dans le registre de statut 2 de la puce Winbond de façon permanente.
```c
// À appeler une fois lors de la fabrication/premier démarrage si vous utilisez le mode 4 lignes
W25Q_EnableQuadMode(&hospi2);
```

### 12. `W25Q_EnableMemoryMappedMode`
Bascule l'OctoSPI en mode d'accès direct. La mémoire Flash devient lisible via un pointeur d'adresse standard du processeur.
```c
if (W25Q_EnableMemoryMappedMode(&hospi2) == HAL_OK) {
    // Définir un pointeur sur l'adresse de base (0x90000000 ou 0x70000000)
    uint8_t *memoire_directe = (uint8_t *)QSPI_BASE_ADDR;
    
    // Lecture directe d'un octet sans aucune fonction HAL !
    uint8_t premier_octet = memoire_directe[0]; 
}
```

### 13. `W25Q_DisableMemoryMappedMode`
Quitte le mode mappé en mémoire de manière propre pour pouvoir exécuter à nouveau des commandes d'écriture ou d'effacement indirectes.
```c
if (W25Q_DisableMemoryMappedMode(&hospi2) == HAL_OK) {
    // L'OctoSPI est repassé en mode indirect standard
    // Vous pouvez maintenant appeler W25Q_Write ou W25Q_EraseSector
}
```

### 14. `W25Q_UpdateLive`
Modifie des données de manière intelligente : lit le secteur en RAM, vérifie si un effacement est techniquement requis, applique les changements et restaure automatiquement le mode mappé en mémoire.
```c
uint8_t configuration_systeme[4] = {0xAA, 0xBB, 0xCC, 0xDD};
uint32_t adresse_sauvegarde = 0x00005000;

// Gère intelligemment l'usure de la flash et protège l'état matériel
if (W25Q_UpdateLive(&hospi2, adresse_sauvegarde, configuration_systeme, 4) == HAL_OK) {
    // Données enregistrées de façon optimale. Le mode Memory-Mapped est réactivé.
}
```

---

## 💻 Exemple d'Intégration Global (`main.c`)

Voici une structure type d'intégration dans votre boucle principale :

```c
#include "main.h"
#include "w25q_flash_RJ-ELEKTRONIK.h"

extern OSPI_HandleTypeDef hospi2; // Votre instance générée par STM32CubeMX

int main(void)
{
  HAL_Init();
  SystemClock_Config();
  MX_OCTOSPI2_Init();

  // 1. Initialisation de la mémoire Flash W25Q
  if (W25Q_Init(&hospi2) != HAL_OK)
  {
      Error_Handler(); // Erreur matérielle
  }

  // 2. Écriture intelligente (UpdateLive gère l'effacement et repasse en mode mappé)
  uint8_t message[] = "RJ-ELEKTRONIK Flash Driver Active";
  if (W25Q_UpdateLive(&hospi2, 0x00001000, message, sizeof(message)) != HAL_OK)
  {
      Error_Handler();
  }

  // 3. Lecture Directe par Pointeur (Mode Mappé)
  uint8_t *ptr_flash = (uint8_t *)(QSPI_BASE_ADDR + 0x00001000);
  
  if (ptr_flash[0] == 'R') 
  {
      // Succès : Accès direct ultra-rapide validé
  }

  while (1)
  {
      // Votre application principale
  }
}
```

---

## 🛠️ Recommandations STM32CubeMX & Matériel

Pour garantir la stabilité des signaux OctoSPI à haute fréquence (STM32H723) :
1. **Clock Prescaler** : Ajustez la valeur selon l'horloge source de l'OctoSPI pour ne pas dépasser la fréquence maximale de votre puce Winbond (généralement 104 MHz ou 133 MHz).
2. **Sample Shifting** : Activez le paramètre *Sample Shifting* à **Half-cycle** dans CubeMX pour compenser les délais de propagation à haute vitesse.
3. **Pins Drive Strength** : Configurez la vitesse (Speed) des broches GPIO utilisées par l'OctoSPI sur **Very High** dans l'onglet *GPIO Configuration*.

---

## 📝 Informations d'édition
- **Auteur** : RENATYJERRY (RJ-ELEKTRONIK)
- **Version actuelle** : 2.1 (Corrigée et Validée)
- **Date de mise à jour** : 15 mai 2026
