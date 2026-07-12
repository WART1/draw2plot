# draw2plot

Interface et plotter graphique automatisé open-source (2 axes X/Y + stylo) piloté depuis un PC. Budget matériel estimé : **moins de 30€**.

![Status](https://img.shields.io/badge/status-en%20développement-yellow) ![Licence](https://img.shields.io/badge/licence-MIT-blue) ![Budget](https://img.shields.io/badge/budget%20matériel-%3C30€-green)

> Tu dessines à la souris sur une interface processing, le robot trace sur papier — avec pause stylo, reprise exacte et communication sécurisée par checksum.

Projet inspiré par : 
- [mertArduino miniCNC plotter](https://www.instructables.com/Arduino-Mini-CNC-Plotter/)
- [Niklas Roy Graffomat](https://www.niklasroy.com/graffomat/)
---

## Table des matières

1. [Concept](#concept)
2. [Matériel requis](#matériel-requis)
3. [Câblage électronique](#câblage-électronique)
4. [Protocole de communication](#protocole-de-communication)
5. [Algorithme de tracé (Bresenham)](#algorithme-de-tracé-bresenham)
6. [Interface graphique](#interface-graphique)
7. [Machine à états](#machine-à-états)
8. [Installation](#installation)
9. [Contribuer](#contribuer)

---

## Concept

DRAW2PLOT sépare l'intelligence logicielle de la gestion mécanique en deux environnements distincts qui communiquent via USB :

| Composant | Technologie | Rôle |
|---|---|---|
| **Interface (Master)** | Processing (Java) | Dessin, chargement `.PLT`, pilotage |
| **Contrôleur (Slave)** | Arduino C++ | Moteurs pas-à-pas, servo stylo, Bresenham |
L'Arduino renvoie un signal de validation (ACK) à Processing **uniquement lorsque le mouvement est physiquement terminé** — ce qui permet de mettre l'impression en pause pour changer de stylo et de reprendre exactement là où elle s'était arrêtée.

---

## Matériel requis

| Composant | Qté | Rôle | Note technique |
|---|---|---|---|
| **Arduino Uno r3** (ou Nano) | 1 | Microcontrôleur principal | Exécute le code C++ et pilote les pins |
| **Moteurs pas-à-pas 28BYJ-48** | 2 | Déplacement axes X et Y | Moteurs économiques avec réducteur intégré |
| **Drivers ULN2003A** | 2 | Cartes de puissance moteurs | Souvent fournis avec le moteur |
| **Micro-Servo SG90** | 1 | Mécanisme de levée du stylo (Z) | Course rapide 0–180° pour lever/poser |
| **Alimentation externe 5V (2A)** | 1 | Source d'énergie mécanique | ⚠️ Ne pas alimenter via le port USB Arduino |

---

## Câblage électronique

> **Règle fondamentale :** sépare l'alimentation logique (USB) de l'alimentation de puissance (5V externe) pour protéger ton ordinateur.

### Connexions drivers ULN2003A → Arduino

| Axe | Broches driver | Pins Arduino |
|---|---|---|
| **Moteur X** | IN1, IN2, IN3, IN4 | 2, 3, 4, 5 |
| **Moteur Y** | IN1, IN2, IN3, IN4 | 6, 7, 8, 9 |
| **Servo (axe Z)** | Signal (orange/jaune) | 10 |

> ⚠️ **Point critique — masse commune :** relie impérativement la borne négative (−) de ton alimentation externe 5V à la broche **GND** de l'Arduino. Sans cette liaison, les signaux n'ont pas de référence et les moteurs agissent de manière erratique.

---
## Protocole de communication

La communication série est **asynchrone avec acquittement (ACK)**, sécurisée par checksum. Chaque message fait exactement **6 octets** :

| Octet | Nom | Plage | Description |
|---|---|---|---|
| 1 | Header | `0` | Début de message |
| 2 | Commande | `1–4` | 1 = ligne, 2 = déplacement rapide, 3 = HOME |
| 3 | Position X | `0–255` | Coordonnée cible axe X |
| 4 | Position Y | `0–255` | Coordonnée cible axe Y |
| 5 | État stylo | `1` ou `255` | 1 = posé (dessin), 255 = levé |
| 6 | Checksum | `1–250` | Somme de contrôle |

```
Processing  →  [Header | Commande | X | Y | Stylo | Checksum]  →  Arduino
Processing  ←  [ACK = octet 6]                                  ←  Arduino
```

Processing attend l'ACK avant d'envoyer le point suivant — aucun mouvement ne peut être perdu.

### Calcul du checksum (Processing)

```java
int v1 = command & 0xFF;
int v2 = xPos   & 0xFF;
int v3 = yPos   & 0xFF;
int v4 = pen    & 0xFF;
int ichksm = ((v1 + v2 + v3 + v4) % 250) + 1;
byte chksm = byte(ichksm);
```

---

## Algorithme de tracé (Bresenham)

Les moteurs pas-à-pas ne connaissent que des déplacements discrets (pas entiers). Pour tracer une ligne diagonale fluide entre deux points sans saccade, l'Arduino utilise l'**algorithme de Bresenham**.

À chaque pas, il évalue en temps réel quelle direction (X, Y, ou les deux simultanément) se rapproche le plus de la droite théorique idéale — sans division ni virgule flottante, ce qui le rend parfaitement adapté à un microcontrôleur. Si l'écart dépasse un seuil, l'axe secondaire avance d'un pas pour corriger la trajectoire.

---

## Interface graphique

| Fonctionnalité | Description |
|---|---|
| **Clic gauche** | Ajoute un segment de tracé standard |
| **Clic droit** | Déplacement sans tracer (sauter d'une forme à l'autre) |
| **Show/Hide Grid** | Affiche ou masque la grille de repères |
| **Grid + / Grid −** | Ajuste l'échelle de la grille |
| **PRINT** | Sauvegarde obligatoire du `.PLT` puis lancement automatique de l'impression |
| **PAUSE** | Interrompt proprement après le segment en cours |
| **PLAY** | Reprend sans décalage depuis le point de pause |
| **STOP** | Annule définitivement et ramène le stylo à l'origine |

L'interface utilise des boutons avec icônes PNG transparentes et des infobulles dynamiques centralisées (variable `hoverLabel` globale, rendue une seule fois en fin de `draw()`).

---

## Machine à états

L'impression est pilotée par la variable `printStatus` dans la boucle `draw()` de Processing :

```
0 — STOP    : arrêt d'urgence, stylo levé immédiatement
1 — PLAY    : envoi séquentiel des points du buffer
2 — PAUSE   : gel propre après le segment en cours (idéal changement de stylo)
3/4 — HOME  : retour automatique en (1,1) en fin de tracé
```

Le **buffer d'impression** (`printXP`, `printYP`…) est figé au clic sur PRINT — l'utilisateur peut continuer à dessiner ou tout effacer sans perturber le tracé en cours.

---
## Installation

### Processing (interface graphique)
1. Installer [Processing 4](https://processing.org/download)
2. Ouvrir `DRAW2PLOT_Processing/DRAW2PLOT_Processing.pde`
3. Sélectionner le bon port série dans le sketch
4. Lancer avec ▶

### Arduino (firmware)
1. Installer l'[IDE Arduino](https://www.arduino.cc/en/software)
2. Ouvrir `DRAW2PLOT_Arduino/DRAW2PLOT_Arduino.ino`
3. Vérifier les pins moteurs et servo dans le fichier
4. Téléverser sur la carte

### Environnement recommandé (optionnel)
VS Code avec le Workspace fourni permet de compiler et lancer Processing via `Ctrl + Shift + B` (nécessite `processing-java` dans le PATH).

---
## Contribuer

Les contributions sont les bienvenues ! Quelques pistes identifiées :

- Optimisation de l'ordre de tracé
- Ajout d'un bouton Stop hardware
---

## Licence

Ce projet est distribué sous licence **MIT** — libre d'utilisation, de modification et de redistribution.
