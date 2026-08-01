🇬🇧 [English](#-diy-48v-bidirectional-battery--pcm-ac-coupling) | 🇫🇷 [Français](#-batterie-diy-48v-bidirectionnelle--pcm-ac-coupling)

---

# 🔋 DIY 48V bidirectional battery — PCM AC-coupling

**Bidirectional** (charge / discharge) battery management system, **AC-coupled** to the home grid, driven by a closed-loop **PID controller** running on [ESPHome](https://esphome.io). The system automatically regulates the power exchanged with the battery to keep the active power measured at the grid injection point (meter) close to **0 W** — i.e. absorbing solar surplus to charge, and drawing from the battery to cover consumption peaks.

It is essentially the **DIY** equivalent (home-built, open source, fully customizable) of commercial AC-coupled storage batteries such as **Zendure SolarFlow**, **Anker SOLIX**, **EcoFlow** or **Jackery** — with the added benefit of being able to fine-tune the regulation, the hardware and the software to your own needs.

The regulation loop is **very fast**, especially with the **feed-forward** option enabled: a sudden load step is absorbed within seconds. For example, a **2000W load can be fully compensated in 2 to 3 seconds**.

The system can deliver up to **3600W in both charge and discharge**, and includes an **offgrid/backup output** to power critical loads during a grid outage. ⚠️ This offgrid output requires particular attention to the **earthing system (TT/TN)** and the associated safety devices (RCD/differential breaker, neutral switching where applicable) — this specific part must be carried out or checked by a qualified electrician.

The main controller is an **ESP32-S3**, flashed with a **100% open source [ESPHome](https://esphome.io) firmware** — no dependency on a proprietary cloud, no forced telemetry: all the regulation code is visible, editable and auditable.

Combined with any DIY LFP battery of **16 kWh**, the total cost of the system is around **€2100**.

This repository documents **version 2** of the project, built around a **PCM (Power Conversion Module) 3600W/48V** driven over CAN bus (MCP2515), replacing the R48/HMS pair from V1.

---

## 📐 System overview

```
┌─────────────────┐        ┌──────────────────────────────┐        ┌──────────────┐
│  Energy           │  ──▶   │   ESP32-S3 (XIAO)              │  ──▶   │  PCM 3600W    │
│  meter             │        │   • Active power reading      │  CAN   │  bidirectional│
│  (JSY1039 /        │        │   • PID controller (dualpidpcm)│        │  48V          │
│  Shelly Pro 3EM)    │        │   • CAN control (MCP2515)      │        │              │
└─────────────────┘        └──────────────────────────────┘        └───────┬──────┘
                                       │                                    │
                                       │ Modbus RTU (RS485)                 │ DC 48V
                                       ▼                                    ▼
                              ┌──────────────────┐                ┌─────────────────┐
                              │  JK-PB-BMS BMS      │◀──────────────│  LFP battery     │
                              │  (battery voltage)   │                │  51.2V           │
                              └──────────────────┘                └─────────────────┘
```

The custom ESPHome component [`dualpidpcm`](./components/dualpidpcm) continuously computes the error between the measured active power and the setpoint (0 W), and drives the PCM over CAN bus to charge or discharge the battery accordingly, with:
- a **two-threshold hysteresis** (stop / restart) per direction, to prevent rapid cycling of the PCM's internal relay;
- accounting for the converter's **idle self-consumption** while discharging;
- a **feed-forward** mode (calibrated setpoint jump) to react quickly to large load swings;
- a direct charge ↔ discharge switch-over without cutting the PCM's main power supply whenever possible.

---

## 🎥 Demo

📺 [The bidirectional V2 battery in action](https://youtu.be/oHbPmVrNmeo)

---

## 📷 Photo preview

### PCM 3600W/48V bidirectional
![PCM 3600W/48V bidirectional](images/pcm.png)
*[View product page ↗](https://french.alibaba.com/product-detail/Industrial-Power-Inverter-PCB-Board-AC-1601732066501.html)*

### XIAO ESP32-S3 Plus
![XIAO ESP32-S3 Plus](images/xiao_ESP32_S3.jpg)
*[View product page ↗](https://www.gotronic.fr/art-carte-xiao-esp32s3-plus-47626.htm)*

### XIAO SX1262 (LoRa module)
![XIAO SX1262](images/xiao_sx1262.jpg)
*[View product page ↗](https://www.gotronic.fr/art-shield-xiao-wio-sx1262-47632.htm)*

### V1.3 control PCB (custom)
![V1.3 control PCB](images/pcb_v1_3.jpg)
*Photo to be added — see the [Gerber file](gerber/Gerber_LoRa_PCB_LoRa-mix_2026-05-16_V1_3.zip) in the meantime.*

---

## 🧩 The three communication variants

The project offers **three different measurement chains** depending on the energy meter used and the communication link available. All three share the same base (BMS, PCM, PID controller) and only differ in how the active power is reported back to the controller.

| Variant | YAML file | Meter | Link | Notes |
|---|---|---|---|---|
| **LoRa** | [`bd_pcm_lora.yaml`](code/bd_pcm_lora.yaml) | JSY1039 | LoRa (SX1262, 868 MHz) via `packet_transport` | The meter is remote (e.g. on the main electrical panel), the battery receives the measurement wirelessly. Requires a shared encryption key (`encryption_key`). |
| **Modbus TCP / WiFi** | [`bd_pcm_shellypro3em.yaml`](code/bd_pcm_shellypro3em.yaml) | Shelly Pro 3EM | Modbus TCP over WiFi (external `esphome_tcp` component) | Reads active power on a selectable phase (`phase_select` selector, p1/p2/p3) exposed by the Shelly. No RS485 wiring needed. |
| **Direct Modbus RTU** | [`bd_pcm_jsy1039.yaml`](code/bd_pcm_jsy1039.yaml) | JSY1039 | Modbus RTU over RS485, direct wiring | The meter is wired directly on the same RS485 bus as the BMS (distinct Modbus addresses). Simplest setup, no network dependency. |

All three files reuse the same ESPHome **packages** (loaded from GitHub):

| Package | Role | Source |
|---|---|---|
| `bms` | JK-PB-BMS reading (battery voltage, cells, etc.) | [`pvbrain2/bms/jikong/device_jkpbbms.yaml`](https://github.com/SeByDocKy/pvbrain2) |
| `pcm` | PCM control over CAN bus (MCP2515) | [`code/device_mcp2515_pcm.yaml`](code/device_mcp2515_pcm.yaml) |
| `dualpidpcm` | Bidirectional PID controller (custom component) | [`code/device_dualpidpcm.yaml`](code/device_dualpidpcm.yaml) |
| `powermeter` *(direct RTU variant only)* | Local JSY1039 reading | [`code/device_jsy1039.yaml`](code/device_jsy1039.yaml) |

---

## 🛠️ Hardware (BOM)

### Electronics

- **PCM 3600W / 48V bidirectional** (the only 48V model found so far) — [supplier link](https://french.alibaba.com/product-detail/Industrial-Power-Inverter-PCB-Board-AC-1601732066501.html)
- **1× XIAO ESP32-S3 Plus** — [Gotronic](https://www.gotronic.fr/art-carte-xiao-esp32s3-plus-47626.htm)
- **1× XIAO SX1262 (LoRa module)** *(LoRa variant only)* — [Gotronic](https://www.gotronic.fr/art-shield-xiao-wio-sx1262-47632.htm)
- **1× XIAO ESP32-S3 Meshtastic (ESP32-S3 + LoRa SX1262)** *(integrated alternative for the LoRa variant)* — [Gotronic](https://www.gotronic.fr/art-xiao-esp32s3-mash-lora-40055.htm)
- **1× MCP2515 CAN controller** — [AliExpress](https://fr.aliexpress.com/item/1005006135600010.html)
  *or as an alternative:*
- **1× MCP2518FD CAN controller** - [Reichelt](https://www.reichelt.com/de/en/shop/product/developer_boards_-_can_module_mcp2518-376524)
- **1× dual high-speed converter** — [AliExpress](https://fr.aliexpress.com/item/1005006063419651.html)

### Protection & connectors

- **3× fuses + 3× fuse holders** — [fuse holders](https://fr.aliexpress.com/item/1005009482930402.html) / [fuses](https://fr.aliexpress.com/item/1005007451125243.html)
- **3× 2-pin terminal block** — [AliExpress](https://fr.aliexpress.com/item/1005010629723401.html)
- **JST 2.54mm male/female connectors** — [AliExpress](https://fr.aliexpress.com/item/4000873858801.html)
- **Long 2.54mm connector** — [AliExpress](https://fr.aliexpress.com/item/1005006783391171.html)
- **1× 600A disconnect switch** (DC battery protection) — [AliExpress](https://fr.aliexpress.com/item/1005010347499706.html)
- **1× RJ45 connector** — [AliExpress](https://fr.aliexpress.com/item/1005009200192593.html)

### Metering & power supply

- **1× JSY-MK-1039 energy meter** — [AliExpress](https://fr.aliexpress.com/item/1005007956840686.html)
- **1× 230V → 5V transformer** (logic power supply) — [AliExpress](https://fr.aliexpress.com/item/1005010271595546.html)

> ⚠️ The Shelly Pro 3EM (Modbus TCP variant) is not listed here — it's an off-the-shelf product, available from most home automation/electrical retailers.

---

## 🖨️ PCB

The **V1.3** PCB (control board integrating the ESP32, the LoRa link and the CAN interface) is available in Gerber format, ready to order from a PCB manufacturer (JLCPCB, PCBWay, etc.):

📦 [`Gerber_LoRa_PCB_LoRa-mix_2026-05-16_V1_3.zip`](gerber/Gerber_LoRa_PCB_LoRa-mix_2026-05-16_V1_3.zip)

---

## ⚙️ Installation

### 0. Without installing ESPHome (via Google Colab)

You don't need to install the ESPHome framework on your machine: the firmware can be compiled and flashed directly from your browser via Google Colab. The method is explained in this video:

📺 [Compile and flash ESPHome without installation, via Google Colab](https://youtu.be/cs016LD6Wy8)

### 1. Requirements

- [ESPHome](https://esphome.io/guides/getting_started_command_line.html) ≥ 2025.10 (2026.7 recommended for the LoRa/RTU variants)
- A `secrets.yaml` file at the root of the project containing at least:

```yaml
wifi_ssid: "MyWiFiNetwork"
wifi_password: "MyWiFiPassword"
ap_password: "FallbackPassword"
encryption_key: "..."   # LoRa variant only (packet_transport)
```

### 2. Choosing a variant

Select the YAML file matching your hardware setup:

```bash
# LoRa variant
esphome run code/bd_pcm_lora.yaml

# Modbus TCP variant (Shelly Pro 3EM)
esphome run code/bd_pcm_shellypro3em.yaml

# Direct Modbus RTU variant (wired JSY1039)
esphome run code/bd_pcm_jsy1039.yaml
```

### 3. Parameters to adjust

Each file exposes `substitutions` at the top to adjust for your installation, notably:

- `bms_modbus_address` / `powermeter_modbus_address` / `shelly_modbus_address`: Modbus addresses of the devices on the bus
- `shelly_ip_address` / `shelly_ip_port` *(TCP variant)*: IP address of the Shelly Pro 3EM
- `pcm_min_charging`, `pcm_max_charging`, `pcm_min_discharging`, `pcm_max_discharging`: current bounds (A) sent to the PCM
- `dualpidpcm_current_min_charging` / `dualpidpcm_current_min_discharging`: minimum currents actually usable by the PCM, used by the controller for its hysteresis thresholds

---

## 🎛️ The PID controller (`dualpidpcm`)

The core of the regulation is the custom `dualpidpcm` component. Once flashed, it exposes in Home Assistant (or the ESPHome web interface):

**Switches**
- `activation` — enables/disables the regulation (⚠️ the state is persisted in flash, survives a reboot)
- `feedforward` — enables the calibrated setpoint jump on large load changes
- `allow_charging` / `allow_discharging` — independently allow/forbid charging and discharging
- `reverse`, `pid_mode`, `manual_override` — advanced options

**Numbers (adjustable on the fly)**
- `setpoint` — active power setpoint (typically 0 W)
- `kp`, `ki`, `kd` — PID gains
- `self_consumption` / `discharge_self_consumption` — converter idle self-consumption while discharging (W)
- `delta_idle_charging` / `delta_idle_discharging` — width of the anti-cycling hysteresis between stop and restart, per direction (W)
- `feedforward_threshold` — trigger threshold for the feed-forward jump (W)
- `starting_battery_voltage` / `stopping_battery_voltage` — battery under-voltage protection hysteresis
- `output_min_charging`, `output_max_charging`, `output_min_discharging`, `output_max_discharging` — output bounds per direction (%)

**Diagnostic sensors**
- `pid_error`, `pid_output`, `pid_deadband`, `pid_mode` — to track the regulation in real time (e.g. via Grafana/InfluxDB or the Home Assistant history)

---

## ⚠️ Warnings

- This project involves **230V AC** and **high-voltage DC battery power (48-58V)** — any work must be carried out by someone competent in electrical work, with the power off, and with proper protection devices (fuses, disconnect switch).
- The **offgrid/backup output** requires particular attention to the **earthing system (TT/TN)** and the associated safety devices (RCD, neutral switching) — to be carried out or checked by a professional.
- This repository is provided **as is**, without warranty. The author cannot be held responsible for any material or bodily damage resulting from building this project.
- Verify the compatibility of your BMS and your LFP battery with the configured voltage ranges before any commissioning.

---

## 📄 License

This project is distributed under the **MIT** license.

```
MIT License

Copyright (c) 2026 SeByDocKy

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

The full text is also available in the [LICENSE](LICENSE) file of the repository.

## 👤 Author

[SeByDocKy](https://github.com/SeByDocKy) — [myESPhome repository](https://github.com/SeByDocKy/myESPhome)

## 🔗 Useful links

- [ESPHome — official documentation](https://esphome.io)
- [pvbrain2 — BMS packages](https://github.com/SeByDocKy/pvbrain2)
- [esphome_tcp — external Modbus TCP component](https://github.com/creepystefan/esphome_tcp)

---
---

# 🔋 Batterie DIY 48V bidirectionnelle — PCM AC-coupling

Système de gestion de batterie **bidirectionnelle** (charge / décharge) couplé en **AC** sur le réseau domestique, piloté par un régulateur **PID en boucle fermée** sous [ESPHome](https://esphome.io). Le système régule automatiquement la puissance échangée avec la batterie pour maintenir la puissance active mesurée au point d'injection (compteur) proche de **0 W** — c'est-à-dire consommer le surplus solaire pour charger, et puiser dans la batterie pour couvrir les pics de consommation.

C'est en somme l'équivalent **DIY** (fait maison, open source, personnalisable) des batteries de stockage commerciales couplées AC type **Zendure SolarFlow**, **Anker SOLIX**, **EcoFlow** ou **Jackery** — avec l'avantage de pouvoir ajuster finement la régulation, le matériel et le logiciel à son propre usage.

La boucle de régulation est **très rapide**, en particulier lorsque l'option **feed-forward** est activée : un appel de charge brutal est absorbé en quelques secondes seulement. À titre d'exemple, une charge de **2000W peut être complètement compensée en 2 à 3 secondes**.

Le système peut délivrer jusqu'à **3600W en charge comme en décharge**, et dispose d'une **sortie offgrid/backup** permettant d'alimenter des charges critiques en cas de coupure secteur. ⚠️ Cette sortie offgrid nécessite une attention particulière au **régime de neutre (TT/TN)** et aux organes de protection associés (disjoncteur différentiel, commutation de neutre le cas échéant) — une intervention par une personne compétente en électricité est indispensable pour ce point précis.

Le contrôleur principal est un **ESP32-S3**, flashé avec un firmware **[ESPHome](https://esphome.io) 100% open source** — aucune dépendance à un cloud propriétaire, aucune télémétrie imposée : tout le code de régulation est visible, modifiable et auditable.

Associé à n'importe quelle batterie DIY LFP de **16 kWh**, le coût total du système avoisine les **2100 €**.

Ce dépôt documente la **version 2** du projet, construite autour d'un **PCM (Power Conversion Module) 3600W/48V** piloté par CAN bus (MCP2515), en remplacement du couple R48/HMS de la V1.

---

## 📐 Vue d'ensemble du système

```
┌─────────────────┐        ┌──────────────────────────────┐        ┌──────────────┐
│  Compteur        │  ──▶   │   ESP32-S3 (XIAO)              │  ──▶   │  PCM 3600W    │
│  d'énergie        │        │   • Lecture puissance active  │  CAN   │  bidirection- │
│  (JSY1039 /       │        │   • Régulateur PID (dualpidpcm)│        │  nel 48V      │
│  Shelly Pro 3EM)   │        │   • Pilotage CAN (MCP2515)     │        │              │
└─────────────────┘        └──────────────────────────────┘        └───────┬──────┘
                                       │                                    │
                                       │ Modbus RTU (RS485)                 │ DC 48V
                                       ▼                                    ▼
                              ┌──────────────────┐                ┌─────────────────┐
                              │  BMS JK-PB-BMS     │◀──────────────│  Batterie LFP    │
                              │  (tension batterie) │                │  51.2V           │
                              └──────────────────┘                └─────────────────┘
```

Le composant ESPHome custom [`dualpidpcm`](./components/dualpidpcm) calcule en continu l'erreur entre la puissance active mesurée et la consigne (0 W), et pilote le PCM via CAN bus pour charger ou décharger la batterie en conséquence, avec :
- une **hystérésis à deux seuils** (arrêt / redémarrage) par direction, pour éviter le cyclage rapide du relais interne du PCM ;
- une prise en compte de **l'autoconsommation à vide** du convertisseur en décharge ;
- un mode **feed-forward** (saut de consigne calibré) pour réagir rapidement aux grosses variations de charge ;
- une bascule directe charge ↔ décharge sans coupure de l'alimentation générale du PCM quand c'est possible.

---

## 🎥 Démonstration

📺 [La batterie bidirectionnelle V2 en fonctionnement](https://youtu.be/oHbPmVrNmeo)

---

## 📷 Aperçu photo

### PCM 3600W/48V bidirectionnel
![PCM 3600W/48V bidirectionnel](images/pcm.png)
*[Voir la page produit ↗](https://french.alibaba.com/product-detail/Industrial-Power-Inverter-PCB-Board-AC-1601732066501.html)*

### XIAO ESP32-S3 Plus
![XIAO ESP32-S3 Plus](images/xiao_ESP32_S3.jpg)
*[Voir la page produit ↗](https://www.gotronic.fr/art-carte-xiao-esp32s3-plus-47626.htm)*

### XIAO SX1262 (module LoRa)
![XIAO SX1262](images/xiao_sx1262.jpg)
*[Voir la page produit ↗](https://www.gotronic.fr/art-shield-xiao-wio-sx1262-47632.htm)*

### PCB de contrôle V1.3 (custom)
![PCB de contrôle V1.3](images/pcb_v1_3.jpg)
*Photo à ajouter — voir le [fichier Gerber](gerber/Gerber_LoRa_PCB_LoRa-mix_2026-05-16_V1_3.zip) en attendant.*

---

## 🧩 Les trois variantes de communication

Le projet propose **trois chaînes de mesure** différentes selon le compteur d'énergie utilisé et le lien de communication disponible. Les trois partagent la même base (BMS, PCM, régulateur PID) et ne diffèrent que par la façon dont la puissance active est remontée au contrôleur.

| Variante | Fichier YAML | Compteur | Liaison | Notes |
|---|---|---|---|---|
| **LoRa** | [`bd_pcm_lora.yaml`](code/bd_pcm_lora.yaml) | JSY1039 | LoRa (SX1262, 868 MHz) via `packet_transport` | Le compteur est déporté (ex: sur le tableau électrique), la batterie reçoit la mesure sans fil. Nécessite une clé de chiffrement partagée (`encryption_key`). |
| **Modbus TCP / WiFi** | [`bd_pcm_shellypro3em.yaml`](code/bd_pcm_shellypro3em.yaml) | Shelly Pro 3EM | Modbus TCP over WiFi (composant externe `esphome_tcp`) | Lit la puissance active sur une phase au choix (sélecteur `phase_select`, p1/p2/p3) exposée par le Shelly. Pas de câblage RS485 nécessaire. |
| **Modbus RTU direct** | [`bd_pcm_jsy1039.yaml`](code/bd_pcm_jsy1039.yaml) | JSY1039 | Modbus RTU sur RS485, câblage direct | Le compteur est câblé en direct sur le même bus RS485 que le BMS (adresses Modbus distinctes). Configuration la plus simple, sans dépendance réseau. |

Les trois fichiers réutilisent les mêmes **packages** ESPHome (chargés depuis GitHub) :

| Package | Rôle | Source |
|---|---|---|
| `bms` | Lecture BMS JK-PB-BMS (tension batterie, cellules, etc.) | [`pvbrain2/bms/jikong/device_jkpbbms.yaml`](https://github.com/SeByDocKy/pvbrain2) |
| `pcm` | Pilotage du PCM via CAN bus (MCP2515) | [`code/device_mcp2515_pcm.yaml`](code/device_mcp2515_pcm.yaml) |
| `dualpidpcm` | Régulateur PID bidirectionnel (composant custom) | [`code/device_dualpidpcm.yaml`](code/device_dualpidpcm.yaml) |
| `powermeter` *(variante RTU directe uniquement)* | Lecture JSY1039 en local | [`code/device_jsy1039.yaml`](code/device_jsy1039.yaml) |

---

## 🛠️ Matériel (BOM)

### Carte électronique

- **PCM 3600W / 48V bidirectionnel** (seul modèle trouvé en 48V à ce jour) — [lien fournisseur](https://french.alibaba.com/product-detail/Industrial-Power-Inverter-PCB-Board-AC-1601732066501.html)
- **1× XIAO ESP32-S3 Plus** — [Gotronic](https://www.gotronic.fr/art-carte-xiao-esp32s3-plus-47626.htm)
- **1× XIAO SX1262 (module LoRa)** *(variante LoRa uniquement)* — [Gotronic](https://www.gotronic.fr/art-shield-xiao-wio-sx1262-47632.htm)
- **1× XIAO ESP32-S3 Meshtastic (ESP32-S3 + LoRa SX1262)** *(alternative intégrée pour la variante LoRa)* — [Gotronic](https://www.gotronic.fr/art-xiao-esp32s3-mash-lora-40055.htm)
- **1× contrôleur CAN MCP2515** — [AliExpress](https://fr.aliexpress.com/item/1005006135600010.html)
  *ou en alternative :*
- **1x contrôleur CAN MCP2518FD** - [Reichelt](https://www.reichelt.com/de/en/shop/product/developer_boards_-_can_module_mcp2518-376524)  
- **1× convertisseur high-speed dual** — [AliExpress](https://fr.aliexpress.com/item/1005006063419651.html)

### Protections & connectique

- **3× fusibles + 3× porte-fusibles** — [porte-fusibles](https://fr.aliexpress.com/item/1005009482930402.html) / [fusibles](https://fr.aliexpress.com/item/1005007451125243.html)
- **3× bornier 2 points** — [AliExpress](https://fr.aliexpress.com/item/1005010629723401.html)
- **Connecteurs JST 2.54mm mâle/femelle** — [AliExpress](https://fr.aliexpress.com/item/4000873858801.html)
- **Connecteur 2.54mm long** — [AliExpress](https://fr.aliexpress.com/item/1005006783391171.html)
- **1× sectionneur 600A** (protection DC batterie) — [AliExpress](https://fr.aliexpress.com/item/1005010347499706.html)
- **1× connecteur RJ45** — [AliExpress](https://fr.aliexpress.com/item/1005009200192593.html)

### Mesure & alimentation

- **1× compteur d'énergie JSY-MK-1039** — [AliExpress](https://fr.aliexpress.com/item/1005007956840686.html)
- **1× transformateur 230V → 5V** (alimentation logique) — [AliExpress](https://fr.aliexpress.com/item/1005010271595546.html)

> ⚠️ Le Shelly Pro 3EM (variante Modbus TCP) n'est pas listé ici — c'est un produit du commerce, disponible chez la plupart des revendeurs d'équipement domotique/électrique.

---

## 🖨️ PCB

Le PCB **V1.3** (carte de contrôle intégrant l'ESP32, le lien LoRa et l'interface CAN) est disponible au format Gerber, prêt à commander chez un fabricant de PCB (JLCPCB, PCBWay, etc.) :

📦 [`Gerber_LoRa_PCB_LoRa-mix_2026-05-16_V1_3.zip`](gerber/Gerber_LoRa_PCB_LoRa-mix_2026-05-16_V1_3.zip)

---

## ⚙️ Installation

### 0. Sans installer ESPHome (via Google Colab)

Vous n'avez pas besoin d'installer le framework ESPHome sur votre machine : il est possible de compiler et flasher le firmware directement depuis le navigateur via Google Colab. La méthode est expliquée en vidéo ici :

📺 [Compiler et flasher ESPHome sans installation, via Google Colab](https://youtu.be/cs016LD6Wy8)

### 1. Pré-requis

- [ESPHome](https://esphome.io/guides/getting_started_command_line.html) ≥ 2025.10 (2026.7 recommandé pour les variantes LoRa/RTU)
- Un fichier `secrets.yaml` à la racine du projet contenant a minima :

```yaml
wifi_ssid: "MonReseauWiFi"
wifi_password: "MotDePasseWiFi"
ap_password: "MotDePasseSecours"
encryption_key: "..."   # uniquement pour la variante LoRa (packet_transport)
```

### 2. Choix de la variante

Sélectionnez le fichier YAML correspondant à votre configuration matérielle :

```bash
# Variante LoRa
esphome run code/bd_pcm_lora.yaml

# Variante Modbus TCP (Shelly Pro 3EM)
esphome run code/bd_pcm_shellypro3em.yaml

# Variante Modbus RTU directe (JSY1039 câblé)
esphome run code/bd_pcm_jsy1039.yaml
```

### 3. Paramètres à adapter

Chaque fichier expose des `substitutions` en tête de fichier à ajuster selon votre installation, notamment :

- `bms_modbus_address` / `powermeter_modbus_address` / `shelly_modbus_address` : adresses Modbus des équipements sur le bus
- `shelly_ip_address` / `shelly_ip_port` *(variante TCP)* : adresse IP du Shelly Pro 3EM
- `pcm_min_charging`, `pcm_max_charging`, `pcm_min_discharging`, `pcm_max_discharging` : bornes de courant (A) transmises au PCM
- `dualpidpcm_current_min_charging` / `dualpidpcm_current_min_discharging` : courants minimum réellement exploitables par le PCM, utilisés par le régulateur pour ses seuils d'hystérésis

---

## 🎛️ Le régulateur PID (`dualpidpcm`)

Le cœur de la régulation est le composant custom `dualpidpcm`. Une fois flashé, il expose dans Home Assistant (ou l'interface web ESPHome) :

**Switches**
- `activation` — active/désactive la régulation (⚠️ l'état est mémorisé en flash, persiste après reboot)
- `feedforward` — active le saut de consigne calibré sur les grosses variations de charge
- `allow_charging` / `allow_discharging` — autorisent/interdisent indépendamment la charge et la décharge
- `reverse`, `pid_mode`, `manual_override` — options avancées

**Numbers (réglables à la volée)**
- `setpoint` — consigne de puissance active (typiquement 0 W)
- `kp`, `ki`, `kd` — gains du PID
- `self_consumption` / `discharge_self_consumption` — autoconsommation à vide du convertisseur en décharge (W)
- `delta_idle_charging` / `delta_idle_discharging` — largeur de l'hystérésis anti-cyclage entre l'arrêt et le redémarrage de chaque direction (W)
- `feedforward_threshold` — seuil de déclenchement du saut feed-forward (W)
- `starting_battery_voltage` / `stopping_battery_voltage` — hystérésis de protection sous-tension batterie
- `output_min_charging`, `output_max_charging`, `output_min_discharging`, `output_max_discharging` — bornes de sortie par direction (%)

**Capteurs de diagnostic**
- `pid_error`, `pid_output`, `pid_deadband`, `pid_mode` — pour suivre la régulation en temps réel (par exemple via Grafana/InfluxDB ou l'historique Home Assistant)

---

## ⚠️ Avertissements

- Ce projet manipule du **230V AC** et de la **haute tension DC batterie (48-58V)** — toute intervention doit être réalisée par une personne compétente en électricité, hors tension, avec les protections adéquates (fusibles, sectionneur).
- La **sortie offgrid/backup** implique une vigilance particulière sur le **régime de neutre (TT/TN)** et les organes de sécurité associés (différentiel, commutation de neutre) — à faire réaliser ou vérifier par un professionnel.
- Ce dépôt est fourni **tel quel**, sans garantie. L'auteur ne peut être tenu responsable des dommages matériels ou corporels liés à la réalisation de ce projet.
- Vérifiez la compatibilité de votre BMS et de votre batterie LFP avec les plages de tension configurées avant toute mise en service.

---

## 📄 Licence

Ce projet est distribué sous licence **MIT**.

```
MIT License

Copyright (c) 2026 SeByDocKy

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

Le texte complet est également disponible dans le fichier [LICENSE](LICENSE) du dépôt.

## 👤 Auteur

[SeByDocKy](https://github.com/SeByDocKy) — [dépôt myESPhome](https://github.com/SeByDocKy/myESPhome)

## 🔗 Liens utiles

- [ESPHome — documentation officielle](https://esphome.io)
- [pvbrain2 — packages BMS](https://github.com/SeByDocKy/pvbrain2)
- [esphome_tcp — composant Modbus TCP externe](https://github.com/creepystefan/esphome_tcp)
