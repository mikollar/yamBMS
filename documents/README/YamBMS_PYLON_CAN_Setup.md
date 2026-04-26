# YamBMS — PYLON CAN BMS Setup Guide

> **Verzia dokumentu:** 2026-04-26  
> **Platí pre:** YamBMS core v1.7.0, canbus v2.5.5  
> **Repozitár:** https://github.com/mikollar/yamBMS (fork od Sleeper85/esphome-yambms)

---

## Obsah

1. [Čo je YamBMS](#1-čo-je-yambms)
2. [Architektúra systému](#2-architektúra-systému)
3. [Hardvérové požiadavky](#3-hardvérové-požiadavky)
4. [Štruktúra súborov](#4-štruktúra-súborov)
5. [Konfigurácia — krok za krokom](#5-konfigurácia--krok-za-krokom)
6. [Parametre batérie](#6-parametre-batérie)
7. [CAN bus — dôležité nastavenia](#7-can-bus--dôležité-nastavenia)
8. [Auto funkcie YamBMS](#8-auto-funkcie-yambms)
9. [Fault Registry — indikátor stavu](#9-fault-registry--indikátor-stavu)
10. [Secrets.yaml](#10-secretsyaml)
11. [Časté chyby a riešenia](#11-časté-chyby-a-riešenia)
12. [Príklad konfigurácie](#12-príklad-konfigurácie)

---

## 1. Čo je YamBMS

**YamBMS** (Yet another multi-BMS Merging Solution) je ESPHome firmware pre ESP32, ktorý:

- Číta dáta z jedného alebo viacerých BMS zariadení (PYLONTECH, JK-B, JK-PB, SEPLOS, JBD, ...)
- Kombinuje dáta z viacerých BMS do jedného virtuálneho BMS
- Posiela informácie o batérii invertoru cez **CAN bus** (PYLON 1.2 / PYLON V2 / SMA / Victron / LuxPower protokol) alebo **RS485**
- Integruje sa s **Home Assistant** cez natívne ESPHome API
- Zobrazuje dáta cez **Web Server** (http://\<IP>)

---

## 2. Architektúra systému

```
┌─────────────────────────────────────────────────────────┐
│                     ESP32 (YamBMS)                      │
│                                                         │
│  ┌──────────┐    ┌──────────┐    ┌────────────────────┐ │
│  │  BMS 1   │    │  BMS 2   │    │   YamBMS Core      │ │
│  │PYLONTECH │    │  JK-B    │    │  (combine logic)   │ │
│  │(CAN bus) │    │(RS485)   │    │                    │ │
│  └────┬─────┘    └────┬─────┘    └────────┬───────────┘ │
│       └───────────────┘                   │             │
│                                           ▼             │
│                              ┌────────────────────────┐ │
│                              │  CAN bus to Inverter   │ │
│                              │  (MCP2515 transceiver) │ │
│                              └────────────┬───────────┘ │
└───────────────────────────────────────────┼─────────────┘
                                            │ CAN 500kbps
                                            ▼
                                    ┌───────────────┐
                                    │  Inverter     │
                                    │  (Deye, SMA,  │
                                    │   Victron...) │
                                    └───────────────┘
```

### Tok dát

1. BMS posiela dáta na CAN zbernicu (PYLONTECH protokol)
2. ESP32 číta dáta cez MCP2515 (alebo ESP32 native CAN)
3. `bms_sensors_PYLON_CAN.yaml` parsuje a mapuje hodnoty na YamBMS senzory
4. `bms_combine.yaml` overuje platnosť dát a zaregistruje BMS do kombinátora
5. `yambms_core.yaml` kombinuje všetky BMS do výsledných hodnôt
6. `yambms_canbus.yaml` posiela kombinované hodnoty invertoru

---

## 3. Hardvérové požiadavky

### Minimálna konfigurácia (PYLONTECH CAN)

| Komponent | Popis |
|-----------|-------|
| ESP32 DevKit V1 | alebo iný kompatibilný board |
| MCP2515 transceiver | SPI CAN modul (TJA1050 alebo SN65HVD230) |
| CAN kábel | prepojenie PYLONTECH → ESP32 → Invertor |

### GPIO pre ESP32 DevKit V1

| Funkcia | GPIO pin |
|---------|----------|
| SPI MOSI | 13 |
| SPI MISO | 12 |
| SPI CLK | 14 |
| MCP2515 CS | 15 |
| UART1 TX | 17 |
| UART1 RX | 16 |
| UART2 TX | 19 |
| UART2 RX | 18 |
| UART3 TX | 26 |
| UART3 RX | 25 |
| Status LED | vstavaná LED |

> **Poznámka:** Pre iný board pozri príslušný `packages/board/board_*.yaml` súbor.

### MCP2515 clock

Väčšina MCP2515 modulov používa **8 MHz** oscilátor. Niektoré lacné moduly majú **16 MHz**.  
Skontroluj oscilátor na module a nastav `mcp2515_clock` podľa toho (pozri [sekciu 7](#7-can-bus--dôležité-nastavenia)).

---

## 4. Štruktúra súborov

```
esphome-yambms/
├── packages/
│   ├── base/
│   │   ├── device_base.yaml              # Základ konfigurácie zariadenia
│   │   ├── device_base_wifi.yaml         # WiFi nastavenia
│   │   ├── device_base_ethernet.yaml     # Ethernet nastavenia
│   │   └── device_base_fault_registry.yaml  # Systém sledovania chýb
│   │
│   ├── board/
│   │   ├── board_ESP32_DevKit-V1.yaml    # Board definícia + GPIO piny
│   │   ├── board_options_itf_canbus_mcp2515.yaml  # MCP2515 CAN transceiver
│   │   ├── board_options_itf_canbus_esp32_can.yaml # ESP32 native CAN
│   │   ├── board_options_itf_uart_esp_*.yaml       # UART rozhrania
│   │   └── ...
│   │
│   ├── bms/
│   │   ├── bms_base.yaml                # Základ BMS senzory (povinné ID)
│   │   ├── bms_base_balancer.yaml       # Podpora externého balancera
│   │   ├── bms_combine.yaml             # Logika kombinácie BMS do YamBMS
│   │   ├── bms_sensors_PYLON_CAN.yaml   # Senzory pre PYLONTECH CAN BMS
│   │   ├── bms_combine_PYLON_CAN.yaml   # Kombinovaný package pre PYLON CAN
│   │   ├── bms_sensors_JK_RS485_*.yaml  # Senzory pre JK BMS varianty
│   │   └── ...
│   │
│   ├── shunt/
│   │   ├── shunt_combine_Victron_*.yaml # Victron SmartShunt integrácia
│   │   ├── shunt_combine_Junctek_*.yaml # Junctek KHF shunt integrácia
│   │   └── ...
│   │
│   └── yambms/
│       ├── yambms.yaml                  # Hlavný YamBMS package (include všetkého)
│       ├── yambms_core.yaml             # Jadro — kombinovanie, výpočty
│       ├── yambms_canbus.yaml           # CAN bus output k invertoru
│       ├── yambms_auto_cvl.yaml         # Auto riadenie nabíjacieho napätia
│       ├── yambms_auto_ccl.yaml         # Auto riadenie nabíjacieho prúdu
│       ├── yambms_auto_dcl.yaml         # Auto riadenie vybíjacieho prúdu
│       ├── yambms_auto_float.yaml       # Auto float fáza
│       ├── yambms_auto_eoc.yaml         # End of Charge logika
│       ├── yambms_auto_temp_ccl_dcl.yaml # Teplotná korekcia prúdu
│       └── yambms_auto_soc_limit.yaml   # Obmedzenie SoC
│
├── YamBMS_PYLON_JK_B.yaml               # Príkladový template pre PYLON + JK-B
└── documents/README/
    └── YamBMS_PYLON_CAN_Setup.md        # Tento dokument
```

---

## 5. Konfigurácia — krok za krokom

### Krok 1: Skopíruj template

Skopíruj `YamBMS_PYLON_JK_B.yaml` do `/config/esphome/` a premenuj podľa potreby:

```bash
cp custom_components/esphome-yambms/YamBMS_PYLON_JK_B.yaml \
   /config/esphome/moje-yambms.yaml
```

### Krok 2: Nastav základné substitúcie

```yaml
substitutions:
  friendly_name: 'YamBMS'        # Zobrazovaný názov v HA
  hostname: 'yambms'              # DNS meno / ID zariadenia (musí byť unikátne)
  name: ''                        # Voliteľný prefix (môže byť prázdny)
```

### Krok 3: Vyber board

Odkomentuj riadok s tvojim boardom:

```yaml
packages:
  device_board: !include 'packages/board/board_ESP32_DevKit-V1.yaml'
  # alebo iný board...
```

### Krok 4: Nastav CAN transceiver

```yaml
  # MCP2515 CAN transceiver
  canbus_node_2: !include
    file: packages/board/board_options_itf_canbus_mcp2515.yaml
    vars:
      canbus_node_id: 'canbus_inverter_1'   # ID CAN zbernice k invertoru
      # mcp2515_clock: '16MHz'              # Odkomentuj ak tvoj MCP2515 má 16MHz oscilátor
```

### Krok 5: Pridaj BMS

```yaml
  bms:
    files:
      - path: 'packages/bms/bms_combine_PYLON_CAN.yaml'
        vars:
          bms_id: '1'                         # MUSÍ byť číslo, začínaj od 1
          bms_name: 'PYLONTECH'
          bms_cell_ovp: '3.600'               # V — ochrana pred prebitím bunky
          bms_cell_uvp: '2.800'               # V — ochrana pred vybitím bunky
          bms_balance_trigger_voltage: '0.010' # V — prahové napätie balancera
          bms_cell_max_cycles: '6000.0'       # Maximálny počet cyklov (pre SoH výpočet)
```

> **Viac BMS:** Skopíruj celý blok a zmeň `bms_id: '2'`, `bms_id: '3'` atď.

### Krok 6: Nastav CAN output k invertoru

```yaml
  canbus1: !include
    file: packages/yambms/yambms_canbus.yaml
    vars:
      canbus_id: '1'                      # ⚠️ MUSÍ byť číslo — nie 'canbus1' !
      canbus_name: 'CANBUS 1'
      canbus_node_id: 'canbus_inverter_1' # Musí súhlasiť s canbus_node_id z kroku 4
      canbus_light_id: 'esp_light'
      canbus_link_timer: '5s'
```

---

## 6. Parametre batérie

### Chemické typy batérií

| Hodnota | Typ |
|---------|-----|
| `1` | LFP (LiFePO4) |
| `2` | Li-ion (NMC/NCA) |
| `3` | LTO (Lithium Titanate) |

```yaml
yambms_battery_chemistry: '1'  # LFP
yambms_cell_count: '15'        # Počet buniek v sérii
```

### Napäťové prahy (príklad pre 15S LFP)

| Parameter | Popis | Príklad 15S LFP |
|-----------|-------|-----------------|
| `yambms_bulk_v` | Bulk/Absorpčné napätie | `52.7` V (3.513V/cell) |
| `yambms_float_v` | Float napätie | `51.5` V (3.433V/cell) |
| `yambms_rebulk_v` | Rebulk napätie | `50.0` V (3.333V/cell) |

> Pre 16S LFP: bulk ≈ 55.2V, float ≈ 53.6V, rebulk ≈ 52.8V

### Časovače

| Parameter | Popis | Default |
|-----------|-------|---------|
| `yambms_eoc_timer` | Max. trvanie cut-off fázy (min) | `30` |
| `yambms_cutoff_timer` | Čas splnenia podmienok pre EOC (s) | `60` |
| `bms_cutoff_timer` | Cut-off timer na úrovni BMS (s) | `50s` |

### Prúdové limity

```yaml
yambms_max_requested_charge_current: '90'    # A — maximum pre nabíjanie
yambms_max_requested_discharge_current: '80' # A — maximum pre vybíjanie
```

> Ak máš viac BMS, YamBMS automaticky distribuuje prúd proporcionálne podľa počtu BMS.

---

## 7. CAN bus — dôležité nastavenia

### ⚠️ Kritická chyba: canbus_id musí byť číslo

```yaml
# ✅ SPRÁVNE
canbus_id: '1'

# ❌ NESPRÁVNE — spôsobí chybu kompilácie
canbus_id: 'canbus1'
```

**Dôvod:** `${canbus_id}` sa substituuje priamo do C++ lambda kódu. Reťazec `canbus1` nie je platný C++ identifikátor v danom kontexte.

**Chybová hláška:**
```
error: 'canbus1' was not declared in this scope
```

### MCP2515 clock

Skontroluj oscilátor na svojom MCP2515 module:

```yaml
canbus_node_2: !include
  file: packages/board/board_options_itf_canbus_mcp2515.yaml
  vars:
    canbus_node_id: 'canbus_inverter_1'
    mcp2515_clock: '8MHz'   # alebo '16MHz' pre modul s 16MHz kryštálom
```

### Protokoly pre invertory

Protokol vyberáš priamo v Home Assistant v entite **"YamBMS protocol"**:

| Protokol | Vhodný pre |
|----------|-----------|
| PYLON 1.2 | Väčšina invectorov (Deye, Growatt...) |
| PYLON V2 | Novšie invertory s rozšíreným protokolom |
| SMA | SMA Sunny Island, SMA Sunny Boy |
| Victron | Victron MultiPlus, Quattro |
| LuxPower | LuxPower invertory |

---

## 8. Auto funkcie YamBMS

Všetky auto funkcie sa zapínajú/vypínajú prepínačom v Home Assistant.

### Auto CVL (Charge Voltage Limit)

Automaticky prispôsobuje nabíjacie napätie na základe stavu buniek (PI regulátor).

- **Kp Term** — proporcionálny zosilňovač (default: 0.05)
- **Ki Term** — integračný zosilňovač (default: 0.5)
- Vyššia hodnota = rýchlejšia reakcia, ale riziko oscilácie

### Auto CCL (Charge Current Limit)

Znižuje nabíjací prúd keď sa bunky blížia k `cell_ovp`.

### Auto DCL (Discharge Current Limit)

Znižuje vybíjací prúd keď sa bunky blížia k `cell_uvp`.

### Auto Float

Prepína medzi Bulk a Float napätím podľa stavu nabitia.

### End of Charge (EOC)

Ukončí nabíjanie keď sú splnené podmienky:
1. Bunky nie sú v ekvalizácii (balancovanie)
2. Cut-off napätie dosiahnuté
3. Podmienky splnené po dobu `yambms_cutoff_timer`

### Teplotná korekcia (Auto Temp CCL/DCL)

Automaticky znižuje nabíjací/vybíjací prúd pri nízkych alebo vysokých teplotách.

### SoC Limit

Obmedzuje nabíjanie/vybíjanie podľa nastavených hodnôt SoC.

---

## 9. Fault Registry — indikátor stavu

YamBMS sleduje zdravie každej komponenty pomocou fault registry systému.

### Kategórie

| Kategória | Popis |
|-----------|-------|
| `CAT_SYSTEM` (0) | Systémové chyby |
| `CAT_BMS` (1) | BMS komunikácia |
| `CAT_INV_CAN_BUS` (2) | CAN bus k invertoru |
| `CAT_INV_RS485` (3) | RS485 k invertoru |
| `CAT_SHUNT` (4) | Shunt meranie |
| `CAT_BALANCER` (5) | Externý balancer |
| `CAT_NETWORK` (6) | Sieťová komunikácia |

### LED indikácia

Status LED na ESP32 bliká podľa stavu CAN bus spojenia s invertorom:
- **Bliká** — invertor odpovedá (ACK 0x305 prijatý)
- **Nesvieti** — žiadna komunikácia s invertorom

---

## 10. Secrets.yaml

V hlavnom ESPHome secrets.yaml (`/config/esphome/secrets.yaml`) musíš mať definované:

```yaml
# WiFi
wifi_ssid: MojaWiFiSiet
wifi_password: MojeHeslo
domain: .local          # ⚠️ Toto musí byť definované! Používa sa v board_ESP32_DevKit-V1.yaml

# WiFi AP (voliteľné)
wifi_ap_ssid: YamBMS
wifi_ap_password: MojeHesloAP

# Web Server (voliteľné)
web_server_username: yambms
web_server_password: MojeHesloWeb
```

> **Poznámka:** `domain: .local` je povinné pri použití `board_ESP32_DevKit-V1.yaml`. Bez neho kompilácia zlyhá s chybou `Secret 'domain' not defined`.

---

## 11. Časté chyby a riešenia

### `'canbus1' was not declared in this scope`

**Príčina:** `canbus_id` obsahuje reťazec namiesto čísla.

```yaml
# Oprav na:
canbus_id: '1'
```

### `Secret 'domain' not defined`

**Príčina:** V `/config/esphome/secrets.yaml` chýba kľúč `domain`.

```yaml
# Pridaj do secrets.yaml:
domain: .local
```

### `ID bms1_equalizing redefined`

**Príčina:** `bms_sensors_*.yaml` definuje `bms${bms_id}_equalizing` priamo, ale `bms_base_balancer.yaml` ho tiež definuje ako wrapper.

**Riešenie:** V `bms_sensors_*.yaml` premenuj na `bms${bms_id}_bms_equalizing` (s `_bms_` prefixom).

### `Couldn't find ID 'bms1_bms_equalizing'`

**Príčina:** `bms_base_balancer.yaml` hľadá `bms${bms_id}_bms_equalizing` ale v `bms_sensors_*.yaml` je definované len `bms${bms_id}_equalizing`.

**Riešenie:** Rovnaké ako vyššie — premenuj v sensors súbore na `_bms_` prefix.

### BMS sa nezobrazuje ako "Can be combined"

`bms_combine.yaml` vyžaduje platné hodnoty pre **všetky** povinné senzory. Skontroluj logy či niektorý senzor nemá hodnotu `NaN`.

### Pomalé git operácie

Workspace je na SMB sieťovom zdieľaní — `git status`, `git push` môžu trvať desiatky sekúnd.

---

## 12. Príklad konfigurácie

Kompletná konfigurácia pre ESP32 DevKit V1 + PYLONTECH (CAN) + MCP2515:

```yaml
logger:
  level: INFO

ota:
  - platform: esphome
    password: tvoje_heslo

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  use_address: 192.168.1.100  # Nastav svoju IP

api:
  reboot_timeout: 0s

substitutions:
  friendly_name: 'YamBMS'
  hostname: 'yambms'
  name: ''
  
  yambms_id: 'yambms1'
  yambms_name: 'YamBMS 1'
  yambms_update_interval: '1s'
  yambms_input_number_mode: 'box'
  
  yambms_battery_chemistry: '1'   # LFP
  yambms_cell_count: '15'         # 15S batéria
  yambms_bulk_v: '52.7'
  yambms_float_v: '51.5'
  yambms_rebulk_v: '50.0'
  yambms_eoc_timer: '30'
  yambms_cutoff_timer: '60'
  yambms_max_requested_charge_current: '90'
  yambms_max_requested_discharge_current: '80'
  
  bms_update_interval: '3s'
  bms_combine_interval: '1s'
  bms_cutoff_timer: '50s'
  shunt_update_interval: '3s'
  shunt_combine_interval: '1s'

packages:
  # Board
  device_board: !include 'packages/board/board_ESP32_DevKit-V1.yaml'
  
  # UART 3 pre RS485 (ak potrebuješ)
  # uart_esp_3: !include packages/board/board_options_itf_uart_esp_3.yaml
  
  # MCP2515 CAN transceiver
  canbus_node_2: !include
    file: packages/board/board_options_itf_canbus_mcp2515.yaml
    vars:
      canbus_node_id: 'canbus_inverter_1'
      # mcp2515_clock: '16MHz'  # Odkomentuj pre 16MHz modul

  # BMS — PYLONTECH cez CAN
  bms:
    files:
      - path: 'packages/bms/bms_combine_PYLON_CAN.yaml'
        vars:
          bms_id: '1'
          bms_name: 'PYLONTECH'
          bms_cell_ovp: '3.600'
          bms_cell_uvp: '2.800'
          bms_balance_trigger_voltage: '0.010'
          bms_cell_max_cycles: '6000.0'

  # YamBMS jadro
  yambms: !include packages/yambms/yambms.yaml

  # CAN bus výstup k invertoru
  canbus1: !include
    file: packages/yambms/yambms_canbus.yaml
    vars:
      canbus_id: '1'                       # ⚠️ musí byť číslo
      canbus_name: 'CANBUS 1'
      canbus_node_id: 'canbus_inverter_1'
      canbus_light_id: 'esp_light'
      canbus_link_timer: '5s'

  # Debug info
  device_debug: !include
    file: packages/base/device_debug_ESP32.yaml
    vars:
      debug_name: 'Debug'
      debug_update_interval: '5s'
      debug_psram_size: '0'
```

---

## Podporované BMS

| BMS | Protokol | Package |
|-----|----------|---------|
| PYLONTECH | CAN bus | `bms_combine_PYLON_CAN.yaml` |
| JK-B | RS485 Modbus | `bms_combine_JK_RS485_Modbus_bms_full.yaml` |
| JK-PB | RS485 Modbus | `bms_combine_JK_RS485_Modbus_bms_full.yaml` |
| JK BLE | Bluetooth | `bms_combine_JK_BLE_full.yaml` |
| SEPLOS V1/V2 | RS485 | `bms_combine_SEPLOS_V1_V2_RS485_bms_full.yaml` |
| SEPLOS V3 | RS485 | `bms_combine_SEPLOS_V3_RS485_bms_full.yaml` |
| JBD | UART/BLE | `bms_combine_JBD_UART_full.yaml` |
| BASEN | RS485 | `bms_combine_BASEN_RS485_full.yaml` |
| DEYE CAN | CAN bus | `bms_combine_DEYE_CAN_module_full.yaml` |
| EG4 | RS485 | `bms_combine_EG4_LLV2_RS485_bms_full.yaml` |
| PACE | RS485 | `bms_combine_PACE_RS485_bms_full.yaml` |

---

*Dokumentácia vytvorená na základe kódu v repozitári mikollar/yamBMS (fork Sleeper85/esphome-yambms)*
