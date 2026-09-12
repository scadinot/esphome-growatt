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
| Firmware ESP | ESPHome **2026.8.2**, flashé le 12/09/2026 |

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
Vécu une troisième fois, et découvert le 12/09 seulement : sept registres
Ond1 étaient lus à la fois par un select ou un number et par un capteur de
comparaison en `force_new_range`. Sur le firmware d'alors, la réponse au poll
des selects était livrée à la plage du doublon — **quatre selects n'ont
jamais reçu de valeur pendant quinze jours**, sans que rien ne le signale
(ils sont `disabled_by_default`). Le mécanisme exact est décrit en tête de
`gw-comparaison.yaml`, incident n° 2.

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

## Flash du 12/09/2026 — passage en ESPHome 2026.8.2

Quatre changements, un seul flash :

- **Sept capteurs de comparaison Ond1 retirés** (H8, H18, H19, H20, H35, H36,
  H39) : ils doublonnaient les selects et numbers de `gw-ond1-reglages.yaml`
  et privaient quatre selects de toute valeur. Vérifié après flash sur le
  flux `/events` de l'ESP : les quatre ont désormais leur état.
- **Fenêtres horaires Ond2 (H3-H6)** : les adresses étaient croisées par
  rapport au document Growatt et aux numbers d'Ond1. Adresses échangées,
  noms conservés — aucune entité renommée.
- **`command_throttle` → `turnaround_time: 100ms`** dans le bloc `modbus:`.
  L'ancienne clé n'a plus d'effet en 2026.8.2 ; sans valeur explicite le
  défaut de 600 ms aurait triplé la durée du cycle de scrutation.
- **`gw-scan-diff.yaml` porté sur la nouvelle API** du composant
  (`std::span`, `modbus::EntityType`) : la compilation échouait sans cela.

Reste à corriger : `gw-firmware.yaml` affirme en tête que l'outil de lecture
ne sait pas descendre dans `packages/`. C'est faux — l'erreur venait d'un nom
de paramètre incorrect.
