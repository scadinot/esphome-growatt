# Growatt — deux onduleurs SPF 6000 ES Plus

Configuration ESPHome instrumentant deux onduleurs **Growatt SPF 6000 ES Plus**
en Modbus RTU sur une paire RS485 partagée.

**La référence est le serveur Home Assistant**, dans `/config/esphome/`.
Ce dépôt est une copie versionnée. Les modifications partent du serveur.

---

## Matériel

| | |
|---|---|
| Carte | Waveshare ESP32-S3-ETH |
| Adresse IP | 192.168.0.70 |
| Onduleur 1 | adresse Modbus 1, série `KHM8F3X0XD`, firmware 100.06 / DSP 101.05 |
| Onduleur 2 | adresse Modbus 2, série `UYN8G3V04A`, firmware 100.08 / DSP 101.07 |
| Liaison | RS485 9600 8N1, TX GPIO17 / RX GPIO16 |

Les deux firmwares **diffèrent** — deux unités achetées ensemble, fabriquées à
des dates différentes. C'est la seule différence matérielle documentée entre
elles, et la piste retenue pour l'écart de régulation de 2,1 V mesuré sorties
découplées.

## Organisation

Treize paquets dans `packages/`. Le découpage date du 23/08/2026 : le fichier
monolithique avait atteint 86 000 caractères et n'était plus réécrivable d'un
bloc par l'outillage.

```
gw-ond1-mesures    capteurs Ond1
gw-ond1-libelles   libellés texte et alarmes Ond1
gw-ond1-reglages   switch / select / number, synchro AC1, fenêtres horaires
gw-ond1-divers     relevés de config en lecture seule
gw-seuils-soc      Prog 12, 13 et 21 — le cycle batterie
gw-ond2            tout l'onduleur 2
gw-horloges        les deux RTC et leur écart
gw-horloge-set     mise à l'heure depuis l'ESP
gw-pv-off          PV et générateur, commentés en attente de raccordement
gw-firmware        versions firmware des deux onduleurs
gw-comparaison     réglages comparés Ond1 / Ond2
gw-comparaison-b   complément : H39, H8, H94, H20, H3-H6, H22
gw-scan-diff       balayage H0-H99 et diff des deux
```

## Deux pièges, à lire avant toute modification

**`skip_updates` appartient à la PLAGE Modbus, pas à l'entité.** ESPHome
regroupe les registres voisins en une seule commande ; un item porteur de
`skip_updates` affame tous ses voisins de plage. `force_new_range: true`
protège l'item qui le porte — **mais pas les voisins de la plage qu'il vient
d'ouvrir**. Vécu deux fois : le 22/08 sur les selects, le 28/08 sur les quatre
fenêtres horaires d'Ond1.

Corollaire : ne jamais dupliquer un registre déjà lu par une autre entité.

**`Duplicate modbus command found` ne signale pas un doublon d'entité.** Le
critère est *quand* le message apparaît. En continu pendant la scrutation,
c'est une saturation — allonger l'intervalle. Une ligne isolée après une
écriture, c'est la relecture de contrôle normale.

## Ce qui n'est pas ici

`secrets.yaml` — WiFi, clé d'API, mot de passe OTA. Il vit uniquement sur le
serveur, et `.gitignore` en interdit l'ajout. Sans lui, cette configuration ne
compile pas telle quelle.

## Rafraîchir depuis le serveur

Par le serveur MCP ESPHome, chemins relatifs à `/config/esphome/` :

```
esphome_pull_files(filenames=["growatt.yaml", "packages/gw-ond2.yaml", ...])
```

Le paramètre s'appelle `filenames`. Omis, l'outil ne rend que la racine.

Puis `git diff` : vide, la sauvegarde est à jour ; sinon il montre exactement
ce qui a changé.

## À faire au prochain flash

Compilé avec ESPHome **2026.7.4**. L'add-on est passé en **2026.8.2**, qui
signale :

```
'command_throttle' no longer has any effect and will be removed in 2027.2.0.
Command spacing is handled by the 'modbus' component — use 'turnaround_time'.
```

Deux commentaires à corriger également : `gw-firmware.yaml` et
`gw-comparaison-b.yaml` affirment que l'outil de lecture ne sait pas descendre
dans `packages/`. C'est faux — l'erreur venait d'un nom de paramètre incorrect.
