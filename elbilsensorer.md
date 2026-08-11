# Elbil-sensorer — funktionsbeskrivelse

Dette dokument beskriver alle sensorer, hjælpere (helpers), et script og en automatisering, der er bygget til at beregne omkostningen ved at lade elbilen derhjemme, holde styr på hvor stor en andel der kommer fra solceller frem for elnettet, og lade opladning ved eksterne ladestandere indgå i det samlede regnskab.

Alt er defineret i [configuration.yaml](configuration.yaml), [automations.yaml](automations.yaml) og [scripts.yaml](scripts.yaml) på Home Assistant-serveren. Dashboardet, der viser tallene, er ikke beskrevet her.

## Grundlæggende forudsætning

Huset har en Huawei-solcelleinverter, en Huawei "Smart Charger 22" elbillader (tilgået via `fusionsolarplus`-integrationen med enheds-præfiks `fsp_ne_312821708_*`), og en elmåler der viser nettoflowet til/fra elnettet. Der er ikke separate strømmålere på selve elbilladeren isoleret fra resten af huset — så al fordeling mellem "hvor meget af opladningen kom fra sol" og "hvor meget kom fra nettet" er **beregnet**, ikke direkte målt.

**Grundprincippet** (proportional fordeling): i ethvert givet øjeblik antages det, at *alt* forbrug i huset — inklusive elbilen — forsynes med samme procentvise blanding af sol og net, som husets samlede balance viser lige nu. Hvis solcellerne dækker fx 30 % af husets *samlede* belastning i det øjeblik, antages det at de også dækker 30 % af bilens opladning i det samme øjeblik. Der er ingen fysisk måde at vide om *netop* bilens elektroner kom fra solcellerne eller nettet, så dette er den mest retvisende tilnærmelse.

Husets eget batteri er bevidst **udeladt** af denne beregning (se automatiseringen "styr batteri-afladning" nedenfor) — regnestykket antager kun to kilder: sol og net.

---

## 1. Live effekt-sensorer (øjebliksbillede)

Disse sensorer genberegnes hver gang deres kildedata opdaterer, og repræsenterer situationen *lige nu*.

### `sensor.elbil_oplader_ac_effekt` — "Elbil oplader AC effekt"
Laderens faktiske AC-effekt (W), trukket fra husets ledningsnet.

Laderens egen sensor (`sensor.fsp_ne_312821708_output_power`) rapporterer kun **DC-effekten** der leveres til bilens batteri — ikke den AC-effekt laderen reelt trækker fra stikkontakten. Der er et par procents tab i AC→DC-konverteringen. For at få den *rigtige* effekt beregnes den i stedet ud fra laderens tre fasestrømme ganget med husets målte fasespænding:

```
AC-effekt = (V_fase_A × I_A) + (V_fase_B × I_B) + (V_fase_C × I_C)
```

hvor spændingerne kommer fra elmålerens `power_meter_phase_a/b/c_voltage`, og strømmene fra laderens egne `fsp_ne_312821708_phase_a/b/c_current`. Denne AC-effekt er den, alle øvrige beregninger bygger videre på, fordi det er den værdi der reelt matcher det, elmåleren og solcelleinverteren måler på samme "side" af installationen.

### `sensor.elbil_oplader_effekt_fra_nettet` — "Elbil oplader effekt fra nettet"
Den del af laderens AC-effekt, der kommer fra elnettet (W).

```
grid_import      = max(-elmåler_effekt, 0)          # kun hvis der importeres
total_supply      = solcelle_effekt + grid_import
grid_andel        = grid_import / total_supply        # 0 hvis total_supply = 0
effekt_fra_nettet = AC_effekt × grid_andel
```

`sensor.power_meter_active_power` bruges med fortegnskonventionen: **positiv = eksport til nettet, negativ = import fra nettet**.

### `sensor.elbil_oplader_effekt_fra_sol` — "Elbil oplader effekt fra sol"
Resten af laderens AC-effekt:
```
effekt_fra_sol = max(AC_effekt − effekt_fra_nettet, 0)
```

### `sensor.elbil_oplader_solandel` — "Elbil oplader solandel"
Solandelen i procent lige nu:
```
solandel % = (AC_effekt − effekt_fra_nettet) / AC_effekt × 100     (0 hvis AC_effekt = 0)
```

### `sensor.elbil_oplader_dkk_per_time` — "Elbil oplader DKK per time"
Den aktuelle omkostnings-*hastighed* i DKK/time (**ikke** en pris per kWh):
```
DKK/time = (effekt_fra_nettet / 1000) × sensor.el_kob_pris
```
`sensor.el_kob_pris` er husets eksisterende Energi Data Service-sensor med den aktuelle spot-købspris (DKK/kWh, inkl. tariffer og moms). Kun net-andelen prissættes — solenergi er "gratis" i denne sammenhæng.

---

## 2. Akkumulerede levetids-sensorer (Riemann-integration)

Disse `platform: integration`-sensorer integrerer en effekt-sensor (W eller DKK/h) fra afsnit 1 over tid til en akkumuleret energi/pris-sensor (kWh/DKK), på samme måde som husets eksisterende `solar_energy_riemann`. "Sensor" og "Kilde" er altså to forskellige entiteter, ikke den samme — kilden er øjebliksværdien, sensoren er summen af den over tid. De vokser for evigt siden sensoren blev oprettet.

| Sensor (akkumuleret, output) | Kilde (øjebliksbillede fra afsnit 1, input) | Enhed |
|---|---|---|
| `sensor.elbil_oplader_energi_fra_nettet` | `sensor.elbil_oplader_effekt_fra_nettet` | kWh |
| `sensor.elbil_oplader_energi_fra_sol` | `sensor.elbil_oplader_effekt_fra_sol` | kWh |
| `sensor.elbil_oplader_pris_akkumuleret` | `sensor.elbil_oplader_dkk_per_time` | DKK |

Alle tre har `max_sub_interval: 5 minutter` sat — det tvinger sensoren til at genberegne mindst hvert 5. minut, selv hvis kildesensoren ikke ændrer værdi. Uden dette kan integrationssensoren "springe" en stor kunstig værdi ind, hvis kilden ligger stille (fx samme værdi i flere dage) og så pludselig ændrer sig — den antager fejlagtigt at den sidste kendte effekt har været konstant i hele mellemtiden. Det skete faktisk under udviklingen (et engangs-hop på ca. 105 kWh), og blev rettet ved at tilføje `max_sub_interval` og nulstille de påvirkede tællere.

---

## 3. Periodetællere for hjemmeopladning (`utility_meter`)

Tolv `utility_meter`-entiteter tager udgangspunkt i de tre levetids-sensorer ovenfor og nulstiller sig automatisk hver dag/uge/måned/år:

| Formål | Entity_id (dag / uge / måned / år) |
|---|---|
| kWh fra nettet | `sensor.elbil_opladning_kwh_fra_nettet_i_dag` / `_denne_uge` / `_denne_maned` / `_dette_ar` |
| kWh fra sol | `sensor.elbil_opladning_kwh_fra_sol_i_dag` / `_denne_uge` / `_denne_maned` / `_dette_ar` |
| Pris (hjemme) | `sensor.elbil_opladning_pris_i_dag` / `_denne_uge` / `_denne_maned` / `_dette_ar` |

Derudover fire beregnede procent-sensorer (solandel for hver periode), som samme formel som den øjebliksbaserede solandel, blot baseret på periodens akkumulerede kWh i stedet for øjeblikkelig effekt:
```
solandel% = kwh_fra_sol / (kwh_fra_sol + kwh_fra_nettet) × 100
```
→ `sensor.elbil_opladning_solandel_i_dag` / `_denne_uge` / `_denne_maned` / `_dette_ar`

**Bemærk:** Home Assistant genererer selv entity_id ud fra sensorens *navn* (ikke ud fra YAML-nøglen eller `unique_id`) — det er derfor entity_id'erne for "denne måned" og "dette år" ender på `_maned` og `_ar` (uden å), mens de danske navne i UI'en stadig viser "måned"/"år" korrekt.

---

## 4. Ekstern opladning (ladestandere væk fra huset)

Da huset ikke kan måle opladning der sker andre steder, er der bygget en manuel indtastningsmekanisme.

### Input-hjælpere (`input_number`)
- `input_number.elbil_ekstern_kwh_indtastning` — indtastningsfelt: kWh købt ved seneste opladning
- `input_number.elbil_ekstern_pris_indtastning` — indtastningsfelt: pris betalt (DKK)
- `input_number.elbil_ekstern_kwh_total` — skjult "bogholderi"-felt: løbende sum af al ekstern kWh
- `input_number.elbil_ekstern_pris_total` — skjult "bogholderi"-felt: løbende sum af al ekstern pris

### Script: `script.elbil_gem_ekstern_opladning` ("Elbil - gem ekstern opladning")
Når scriptet køres (fx via en knap på dashboardet):
1. Lægger værdien i `elbil_ekstern_kwh_indtastning` til `elbil_ekstern_kwh_total`
2. Lægger værdien i `elbil_ekstern_pris_indtastning` til `elbil_ekstern_pris_total`
3. Nulstiller begge indtastningsfelter til 0, klar til næste registrering

### Sensorer der eksponerer totalerne pænt
- `sensor.elbil_ekstern_opladning_kwh_total` / `sensor.elbil_ekstern_opladning_pris_total` — spejler blot de to "total"-input_numbers som rigtige sensor-entiteter (med `device_class`/`state_class`), så de kan bruges som kilde til `utility_meter`.

### Periodetællere for ekstern opladning
Samme mønster som hjemmeopladningen — otte `utility_meter`-entiteter afledt af de to total-sensorer ovenfor:
- `sensor.elbil_ekstern_opladning_kwh_i_dag` / `_denne_uge` / `_denne_maned` / `_dette_ar`
- `sensor.elbil_ekstern_opladning_pris_i_dag` / `_denne_uge` / `_denne_maned` / `_dette_ar`

---

## 5. Kombinerede totaler (hjemme + ekstern)

Ti sensorer, der lægger hjemme- og eksternopladning sammen — **uden** at det påvirker solandels-beregningerne (som forbliver isoleret til hjemmeopladning, da ekstern strøm per definition har 0 % kendt solandel):

**Levetid:**
- `sensor.elbil_total_kwh_hjemme_og_ekstern` = `elbil_oplader_energi_fra_nettet` + `elbil_oplader_energi_fra_sol` + `elbil_ekstern_kwh_total`
- `sensor.elbil_total_pris_hjemme_og_ekstern` = `elbil_oplader_pris_akkumuleret` + `elbil_ekstern_pris_total`

**Pr. periode** (dag/uge/måned/år), samme mønster:
- `sensor.elbil_total_kwh_i_dag_hjemme_og_ekstern` / `_denne_uge_...` / `_denne_maned_...` / `_dette_ar_...`
- `sensor.elbil_total_pris_i_dag_hjemme_og_ekstern` / `_denne_uge_...` / `_denne_maned_...` / `_dette_ar_...`

Hver af disse lægger simpelthen den tilsvarende hjemme-periode-tæller sammen med den tilsvarende ekstern-periode-tæller.

---

## 6. Automatisering: styring af husbatteriets afladning

**`automation.elbil_oplader_deaktiver_batteri_afladning`** ("Elbil oplader - styr batteri-afladning")

Husets Huawei-batteri kan aflade for at dække husets forbrug, hvilket ville forstyrre sol/net-beregningen ovenfor (batteriets bidrag ville fejlagtigt blive talt som "solenergi" fordi det reducerer den målte netimport). Automatiseringen løser det ved at sætte batteriets tilladte afladningseffekt til 0 W, mens bilen lader:

- **Trigger:** enhver ændring af laderens statussensor (`sensor.fsp_ne_312821708_status`)
- **"Er der ved at blive ladet?"** afgøres ved at statusteksten (case-insensitivt) indeholder ordet "charging", men **ikke** indeholder "waiting" eller "ends" — det dækker alle laderens statusvarianter: `Charging`, `Starting Charging`, `PV Power Charging` (= lader), mens `Timed Charging Waiting`, `PV Power Waiting`, `Charging Ends`, `On Standby` og `No Car Connected` regnes som "lader ikke".
- **Ved overgang til "lader":** gemmer den nuværende `number.battery_maximum_discharging_power`-værdi i `input_number.battery_discharge_power_before_ev_charging`, og sætter derefter afladningseffekten til 0.
- **Ved overgang væk fra "lader":** genopretter den gemte værdi.

Bemærk: dette nulstiller batteriets afladning for **hele huset**, ikke kun elbilens andel — hvilket passer perfekt med at solandels-formlen også regner på husets samlede balance.

---

## Opsummering af beregningskæden

```
fsp_ne_312821708_phase_a/b/c_current  ┐
power_meter_phase_a/b/c_voltage       ┴──► elbil_oplader_ac_effekt (AC, W)
                                             │
power_meter_active_power ──┐                │
inverter_active_power ─────┴──► grid_andel ─┤
                                             ├──► elbil_oplader_effekt_fra_nettet (W)
                                             └──► elbil_oplader_effekt_fra_sol (W)

elbil_oplader_effekt_fra_nettet × el_kob_pris ──► elbil_oplader_dkk_per_time (DKK/h)

effekt-sensorer ──[Riemann-integration]──► energi_fra_nettet / energi_fra_sol / pris_akkumuleret (levetid, kWh/DKK)

levetids-sensorer ──[utility_meter]──► periodetællere (dag/uge/måned/år)

input_number (indtastning) ──[script]──► ekstern total (kWh/DKK) ──[utility_meter]──► ekstern periodetællere

hjemme-periodetællere + ekstern-periodetællere ──► kombinerede total-sensorer (hjemme + ekstern)
```
