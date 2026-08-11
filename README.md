# Elbil – Home Assistant

Home Assistant-opsætning der beregner hvor meget det koster at lade en elbil derhjemme, opdelt på hvor meget der kommer fra **solceller** vs. **elnettet**, og som lader manuelt loggede opladninger ved eksterne ladestandere indgå i det samlede regnskab.

## ⚠️ Forbehold

Dette repo deler min personlige opsætning til inspiration — **brug det på eget ansvar**:

- Sensorerne er **beregnede tilnærmelser**, ikke direkte målinger. Der sidder ingen fysisk strømmåler isoleret på selve elbilladeren, så fordelingen mellem sol og net er baseret på en antagelse om, at alt forbrug i huset i et givet øjeblik forsynes med samme sol/net-blanding (se [elbilsensorer.md](elbilsensorer.md) for detaljerne).
- **Ingen garanti for korrekthed.** Tallene er ikke verificeret mod en officiel elregning, og der er ingen garanti for at beregningerne er fejlfri. Brug det som et estimat/overblik, ikke som grundlag for afregning med andre.
- **Entity-navne er specifikke for mit udstyr.** Fx `sensor.fsp_ne_312821708_*` refererer til min konkrete Huawei-lader (device-id'et er unikt per installation). Du skal tilpasse alle entity-referencer til dit eget udstyr, før noget af dette virker.
- YAML-filerne i dette repo er **udtrukne kopier** af det der kører på min Home Assistant-server, ikke filer der kan inkluderes direkte — se afsnittet "Sådan bruges filerne" nedenfor.
- Jeg giver ikke support på dette repo, men du er velkommen til at bruge og tilpasse det frit.

## Dashboards

- [dashboards/elbil.yaml](dashboards/elbil.yaml) — simpelt YAML-mode dashboard (én side, alle tal og rådata samlet)
- [dashboards/elbil_g6.yaml](dashboards/elbil_g6.yaml) — mere avanceret "sections"-dashboard med fem faner (I dag / Uge / Måned / År / Total), Mushroom-cards og ApexCharts-søjlediagrammer. Kræver [Mushroom](https://github.com/piitaya/lovelace-mushroom), [ApexCharts Card](https://github.com/RomRider/apexcharts-card) og [card-mod](https://github.com/thomasloven/lovelace-card-mod) fra HACS.

## Dokumentation

- [elbilsensorer.md](elbilsensorer.md) — fuld funktionsbeskrivelse af alle sensorer, hjælpere, scriptet og automatiseringen: hvad de gør, og præcis hvordan hver beregning er sat sammen.

## Grundlæggende sensorer der kræves

Alt i dette repo er **beregnet ud fra** eksisterende sensorer fra andre integrationer — de er ikke selv en del af dette repo, men skal findes i forvejen for at noget af det virker. Med mit udstyr kommer de fra tre kilder:

### Huawei solcelleinverter + elmåler + batteri ([huawei_solar](https://github.com/wlcrs/huawei_solar/wiki))
| Entity | Bruges til |
|---|---|
| `sensor.inverter_active_power` | Solcelleproduktion (AC, W) |
| `sensor.power_meter_active_power` | Elmålerens nettoflow til/fra nettet (+ = eksport, − = import) |
| `sensor.power_meter_phase_a/b/c_voltage` | Fasespændinger — bruges til at beregne laderens AC-effekt |
| `number.battery_maximum_discharging_power` | Batteriets tilladte afladningseffekt — kun nødvendig hvis du vil bruge batteri-automatiseringen |

### Elbil-lader ([FusionSolarPlus](https://github.com/JortvanSchijndel/FusionSolarPlus) — Huawei FusionSolar-tilknyttet lader)
| Entity | Bruges til |
|---|---|
| `sensor.fsp_ne_XXXXXXXXX_status` | Ladestatus (bruges bl.a. til at afgøre om bilen lader lige nu) |
| `sensor.fsp_ne_XXXXXXXXX_output_power` | Laderens DC-effekt (til bilens batteri) |
| `sensor.fsp_ne_XXXXXXXXX_phase_a/b/c_current` | Laderens fasestrømme — bruges til at beregne den reelle AC-effekt |

`XXXXXXXXX` er et device-id der er unikt for din lader — findes automatisk når integrationen sætter enheden op.

### Elpris ([Energi Data Service](https://github.com/MTrab/energidataservice))
| Entity | Bruges til |
|---|---|
| `sensor.el_kob_pris` | Aktuel spot-købspris (DKK/kWh, inkl. tariffer/moms) — samt attributterne `raw_today`/`raw_tomorrow` (time-for-time priser) til "billigste ladetidspunkt"-kortene i G6-dashboardet |

## Sådan bruges filerne

Filerne i `homeassistant/` er **ikke** noget Home Assistant læser direkte via `!include` — de er udtrukket fra en langt større `configuration.yaml`/`automations.yaml`/`scripts.yaml`, der også indeholder en masse andet, ikke-elbil-relateret husopsætning. For at bruge dem:

1. Kopiér indholdet af `homeassistant/sensors.yaml` ind under dine egne `sensor:`/`template:`-blokke i `configuration.yaml`
2. Tilsvarende for `input_number.yaml` og `utility_meter.yaml`
3. Kopiér automatiseringen fra `homeassistant/automations.yaml` og scriptet fra `homeassistant/scripts.yaml` ind i dine egne `automations.yaml`/`scripts.yaml`
4. **Erstat alle entity-referencer** (`sensor.fsp_ne_312821708_*`, `sensor.inverter_active_power`, `sensor.power_meter_*`, `sensor.el_kob_pris` osv.) med dine egne
5. Genstart Home Assistant og tjek `ha core check` (eller Udvikler-værktøjer → YAML) for fejl
6. Importér et af dashboardene, og opdater entity-navnene der på samme måde
