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

Précision du 13/09 : `force_new_range` sur *tout* item à `skip_updates`
produisait dix-neuf trames d'un registre côté Ond2, tirées ensemble toutes
les 10 min 30. Il n'est nécessaire que face à un voisin contigu *sans*
`skip_updates` ; des voisins au même `skip` se regroupent en une trame.
Les capteurs de comparaison d'Ond2 sont passés de 19 à 8 trames, et c'est
H38 (AC1 Ond2, sans skip) qui porte désormais le `force_new_range`.

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

Même soir, second flash : recherche des données BMS reçues par le CAN. Le
bloc input 90-117 du document Growatt de 2017 est **vide sur l'ES Plus**, et
un balayage de input 90-999 et 3000-3299 n'a trouvé ni SOC, ni tension, ni
cellules — seulement un **miroir des holding registers en input 108-210**.
L'onduleur ne laisse voir du CAN que le SOC (input 18), la limite de courant
(H34) et la tension de charge (H35/H36 = 57,0 V). Compte rendu dans
`gw-bms-can.yaml`, outil de balayage conservé dans `gw-scan-input.yaml`.

13/09, point 7 : le scan comparatif lisait 0/100 côté Ond2 depuis le 28/08.
Un test de neuf lectures une par une (`gw-scan-ond2.yaml`) a montré qu'Ond2
accepte tous les blocs, y compris ceux que le document interdit ; la cause
était l'empilement de dix blocs dans la file, vidée à la première
non-réponse. `gw-scan-diff.yaml` lit désormais trois blocs alignés sur 45
par onduleur, un à la fois, et compare les deux machines en entier.

**Premier balayage complet, 00:47 : 99/99 et 99/99, huit écarts, tous
attendus** — firmware (H11, H14), série (H23-H27), code usine (H80).
**Aucun réglage ne diffère entre les deux onduleurs.** L'écart de 2,1 V entre
les sorties découplées n'a donc aucun réglage pour cause ; il ne reste que le
DSP (101.05 contre 101.07). Au passage, le miroir input/holding d'Ond2 est
décalé de 113 et non de 108 : un firmware, un décalage.

Même nuit : les huit codes énumérés d'Ond2 (H1, H2, H8, H18, H19, H20, H22,
H39) sont devenus des libellés, copiés des `optionsmap` des selects Ond1 —
les deux colonnes du tableau de bord *Synchronisation* se lisent pareil. Le
capteur numérique brut reste en `internal`, le `text_sensor` a repris son nom
et donc son identifiant HA.

13/09, points 3 et 6 : les défauts et avertissements sont lus sur les quatre
registres 40-43 et les libellés tentent les deux lectures — table LCD du
manuel sur le numéro, table de bits du document Growatt sur le champ — en
affichant toujours les valeurs brutes entre crochets, parce que rien n'a
jamais été non nul et que les deux sources se contredisent ; le premier
événement réel figera le décodage. Deux entités **RTC Dérive Ond1 / Ond2**
mesurent chaque horloge contre le SNTP au moment de la lecture (positif = en
avance) ; l'ancien « Écart Ond1-Ond2 » était un artefact d'échantillonnage.
`Work Time Total` retiré (0,0 h depuis le 13/08, registre non implémenté).
