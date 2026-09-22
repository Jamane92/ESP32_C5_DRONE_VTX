# VTX ESP32-C5 — Air unit FPV (caméra OV5640 + enregistrement SD + WiFi 5 GHz)

Firmware ESP-IDF pour un ESP32-C5 qui capture le flux JPEG d'une caméra OV5640
en DVP, le diffuse en direct via WiFi 5 GHz (softAP), l'enregistre en option
sur carte micro-SD, et expose une interface web de réglage (résolution,
qualité JPEG, netteté, etc.).

## Sommaire

- [Matériel et brochage](#matériel-et-brochage)
- [Pourquoi VSYNC/HREF ne sont pas câblés (et comment on s'en passe)](#pourquoi-vsynchref-ne-sont-pas-câblés-et-comment-ons-en-passe)
- [PARLIO : le périphérique qui reçoit le bus caméra](#parlio--le-périphérique-qui-reçoit-le-bus-caméra)
- [PSRAM : ce qui y vit et pourquoi](#psram--ce-qui-y-vit-et-pourquoi)
- [Modèle de tâches / concurrence](#modèle-de-tâches--concurrence)
- [Serveur HTTP et endpoints](#serveur-http-et-endpoints)
- [Structure du dépôt](#structure-du-dépôt)
- [Limitations connues](#limitations-connues)

## Matériel et brochage

- MCU : **ESP32-C5** (RISC-V, WiFi 2,4/5 GHz, un seul cœur pour les tâches applicatives)
- Caméra : **OV5640** (5 MP, DVP 8 bits, JPEG matériel), objectif M12 3,6 mm fixe
- Stockage : carte micro-SD en **SPI** (pas SDMMC 4 bits)

| Signal caméra | GPIO | Remarque |
|---|---|---|
| XCLK | 26 | Horloge fournie à la caméra (24 MHz) |
| SDA / SCL | 25 / 24 | I2C (SCCB) pour la configuration du capteur |
| D0-D7 | 7,9,10,8,6,27,4,5 | Bus de données parallèle 8 bits |
| PCLK | 23 | Horloge pixel — **seul signal de synchro réellement utilisé** |
| PWDN / RESET | -1 / -1 | Non câblés (tirés au bon niveau sur le module caméra) |
| **VSYNC** | **-1** | **Non câblé** |
| **HREF / HSYNC** | **-1** | **Non câblé** |

## Pourquoi VSYNC/HREF ne sont pas câblés (et comment on s'en passe)

Deux raisons se cumulent, une contrainte matérielle et un choix de brochage :

1. **VSYNC n'est de toute façon pas exploitable par le périphérique PARLIO
   de l'ESP32-C5.** Ce n'est pas qu'un manque de broches disponibles : même
   configuré, le driver logue *"VSYNC signal can not be used, ignoring the
   assigned pin"* — le contrôleur PARLIO RX de ce SoC n'a pas d'entrée
   dédiée à un signal de trame comme VSYNC.
2. **HREF/HSYNC auraient pu être câblés** (PARLIO sait s'en servir comme
   signal de "valid" matériel pour délimiter chaque ligne), mais l'ESP32-C5
   dispose de très peu de GPIO comparé à un ESP32/ESP32-S3 classique. Une
   fois XCLK, I2C, PCLK, les 8 lignes de données et les 4 broches SPI de la
   carte SD posées, il ne restait plus de broches à sacrifier pour HREF —
   d'où le choix de ne pas le câbler non plus.

**La solution retenue : laisser PARLIO tourner en mode "délimiteur
logiciel"** (`parlio_rx_soft_delimiter`), sans aucun signal de synchro
matériel. Le composant caméra échantillonne en continu les 8 lignes de
données sur chaque front de PCLK, produisant un flux d'octets brut, sans
notion native de "début/fin d'image".

Ça ne fonctionne que parce que **le capteur est configuré en sortie JPEG
matérielle** (`pixel_format = ESP_CAM_IO_PARL_PIXFORMAT_JPEG`). Le JPEG
est un format auto-délimité : chaque image commence par le marqueur SOI
(`0xFFD8`), puis un marqueur SOS (`0xFFDA`) avant les données entropiques,
et se termine par EOI (`0xFFD9`). Le driver fait tourner une petite machine
à états (`IDLE → HEADER → ENTROPY → IDLE`) sur le flux brut pour repérer
ces marqueurs et reconstituer les images, **entièrement en logiciel**, sans
avoir besoin de savoir où commence/finit une ligne ou une trame au niveau
matériel.

Cette astuce a une contrepartie importante : **elle ne marche qu'en JPEG.**
Passer le capteur en sortie brute (YUV/RGB, sans compression) ne
fonctionnerait pas dans cette configuration, faute de marqueurs à détecter
dans le flux — il faudrait alors câbler au minimum HREF (voire VSYNC sur un
autre SoC qui le supporte) pour cadrer les lignes/trames matériellement.

## PARLIO : le périphérique qui reçoit le bus caméra

PARLIO (*Parallel IO*) est un périphérique récent d'Espressif (ESP32-C5,
C6, H2, P4...) qui remplace l'ancien contrôleur "I2S caméra" utilisé sur
ESP32/ESP32-S3. Ici il est configuré en réception (RX) sur 8 lignes de
données avec une horloge externe (`PARLIO_CLK_SRC_EXTERNAL`) calée sur le
PCLK de la caméra — c'est le capteur qui pilote le rythme d'échantillonnage,
pas l'ESP32. Les octets reçus sont livrés par paquets via un callback
(`on_partial_receive`), consommés par la machine à états JPEG décrite
ci-dessus, puis remis à disposition de l'application dans une file
(`esp_cam_io_parl_receive()`), une image complète à la fois.

## PSRAM : ce qui y vit et pourquoi

L'ESP32-C5 n'a que quelques centaines de Ko de RAM interne, largement
insuffisants pour stocker plusieurs images JPEG + les buffers HTTP de front.
Tout ce qui est volumineux ou peut varier en taille est donc alloué
explicitement en PSRAM (`MALLOC_CAP_SPIRAM`), jamais sur la pile :

- Le buffer d'image partagé entre la capture et le streaming
  (`shared_frame.buf`, jusqu'à 256 Ko)
- Les buffers de rendu HTML de la page principale (`head`/`tail` dans
  `root_get_handler`)

Ce dernier point vient d'un vrai bug corrigé en cours de route : ces
buffers HTML étaient initialement des variables locales (donc sur la pile
de la tâche du serveur HTTP, ~4 Ko par défaut) — largement dépassées par
une page de plusieurs Ko, ce qui provoquait un dépassement de pile et un
redémarrage en boucle dès qu'un client se connectait. Les mettre en PSRAM
(avec libération systématique, y compris sur les chemins d'erreur) a réglé
le problème sans dépendre de la taille de pile allouée au serveur.

## Modèle de tâches / concurrence

Une seule tâche, `capture_task`, est l'unique lectrice du flux caméra.
À chaque image reçue, dans l'ordre :

1. Copie vers `shared_frame` (protégée par mutex) — consommée par `/stream`
   et `/capture`.
2. Si un enregistrement est en cours, écriture bloquante (`fwrite`) sur la
   carte SD.

Conséquence assumée : **un enregistrement SD actif ralentit légèrement le
flux live**, puisque l'écriture SD (SPI, largement plus lente que de la
RAM) retarde la lecture de l'image suivante par la même tâche. C'est
volontairement simple (une seule source de vérité, pas de double
tamponnage) plutôt que découplé sur deux tâches + une file — un choix qui
pourrait évoluer si le compromis devient gênant.

## Serveur HTTP et endpoints

| Endpoint | Rôle |
|---|---|
| `GET /` | Page principale : flux vidéo, REC start/stop, tous les réglages caméra |
| `GET /pov` | Flux vidéo seul, pivoté pour un affichage en paysage |
| `GET /stream` | Flux MJPEG (`multipart/x-mixed-replace`), port 81 |
| `GET /capture` | Une image JPEG unique |
| `GET /quality?q=` | Qualité JPEG (0-63) |
| `GET /settings?...` | Luminosité, contraste, saturation, effet, netteté, débruitage, correction d'objectif |
| `GET /resolution?f=` | Change la résolution à chaud (QQVGA → HD) |
| `GET /status` | JSON : FPS capture/stream, état d'enregistrement |
| `GET /record/start`, `/record/stop` | Pilotage de l'enregistrement SD |

## Structure du dépôt

Le projet avance par étapes validées une à une, chaque dossier reprenant
le précédent en y ajoutant une brique :

```
01_sd_card_test/          Montage SD seul
02_wifi_5ghz_test/        SoftAP WiFi 5 GHz seul
03_camera_validate/       Caméra OV5640 seule (une capture)
04_camera_stream_5ghz/    Flux MJPEG en direct via WiFi
05_camera_record_sd/      + enregistrement SD, tâche de capture unique
06_camera_quality_tuning/ + réglages avancés, page /pov, résolution à chaud
```

## Limitations connues

- **Plafond de gain (gainceiling) volontairement absent.** Le driver OV5640
  utilisé (identique sur ce point au driver officiel Espressif
  `esp32-camera`) écrit l'indice brut (0-6) dans les registres `0x3A18`/
  `0x3A19`, alors que l'OV5640 y attend un code de gain sur ~10 bits. Le
  résultat plafonne le gain réel bien en dessous de 1x — l'image devient
  très sombre et ne récupère plus tant que le capteur n'est pas réinitialisé.
  Le contrôle a été retiré de l'interface et de l'API tant que cette
  fonction n'est pas réécrite avec les vraies valeurs de registre.
- **Objectif à mise au point fixe**, sans autofocus (pas de moteur voice-coil
  câblé) — la netteté au loin dépend uniquement de la résolution choisie et
  du réglage mécanique de l'objectif.
- **Le champ "Latence" affiché à l'écran mesure le temps d'aller-retour de
  l'appel `/status`**, pas la latence bout-en-bout réelle du flux vidéo
  (qui inclut l'encodage JPEG, la mise en file PARLIO et le rendu côté
  navigateur). À prendre comme un indicateur de réactivité du WiFi, pas
  comme une mesure de latence vidéo absolue.
