# Avancerad produktionsguide: Alarmo + Zigbee i ditt hem (Home Assistant 2026)

> Fokus: **stabilt, lokalt, produktionsklart** med din nuvarande setup (HA installerat, ZBT-2, Hue Bridge, Aqara T1, Frient, Eufy, Samsung Family Hub, LG ThinQ, Roborock).

---

## 1) Målarkitektur (så vi bygger rätt från start)

- **Primärt radiospår:** Zigbee via **ZBT-2** (lokalt).
- **Larmmotor:** Alarmo (HACS).
- **Larmzoner:**
  - `Entrézon` = huvuddörr.
  - `Nedervåning` = altandörr + fönster.
  - **Ingen övervåningszon** (enligt krav).
- **Brand:** Frient används för rökdetektion och brandautomation.
- **Inbrottssiren:** Frient kan användas om din modell exponerar siren-entity; annars separat Zigbee-siren (fallback).

---

## 2) Exakta steg: Installation av Alarmo (UI-klick)

1. Gå till **HACS → Integrations → Explore & Download Repositories**.
2. Sök `Alarmo`.
3. Klicka **Alarmo → Download**.
4. Vänta tills installation är klar.
5. Gå till **Settings → System → Restart** och starta om HA.
6. Gå till **Settings → Devices & Services → Add Integration**.
7. Sök `Alarmo` och klicka **Add**.
8. Ge larmet namn: `Hemlarm`.
9. PIN:
   - Välj **Require code to arm/disarm**.
   - Ange minst 6 siffror.
10. Slutför wizard.

### Rekommenderade grundvärden i Alarmo

- Entry delay: **30 s**
- Exit delay: **45 s**
- Trigger time: **180 s**
- Arming modes: `Away`, `Night`

---

## 3) Enhetsmodellering före zoner (viktigt för driftsäkerhet)

Gör först tydliga namn och areas:

1. **Settings → Devices & Services → Entities**.
2. Byt namn enligt nedan:
   - `binary_sensor.aqara_t1_huvuddorr`
   - `binary_sensor.aqara_t1_altandorr`
   - `binary_sensor.aqara_t1_fonster_vardagsrum`
   - `binary_sensor.frient_smoke_entre`
   - `binary_sensor.frient_smoke_overvaning`
3. Lägg i area:
   - Entré/hall: huvuddörr + frient entré
   - Vardagsrum: altandörr + fönster
   - Övervåning hall: frient övervåning

---

## 4) Alarmo-zoner exakt enligt planritning

## 4.1 Entrézon

1. **Settings → Devices & Services → Alarmo → Configure**.
2. **Sensors → Add sensor**.
3. Välj `binary_sensor.aqara_t1_huvuddorr`.
4. Sensor type: `Door`.
5. Arm modes: `Away`, `Night`.
6. Delay: `Use entry delay`.
7. Zone name: `Entrézon`.

## 4.2 Nedervåning (huvudzon)

1. **Sensors → Add sensor**.
2. Lägg till `binary_sensor.aqara_t1_altandorr`.
   - Type: `Door`
   - Arm modes: `Away`, `Night`
   - Delay: `Instant`
   - Zone: `Nedervåning`
3. Lägg till `binary_sensor.aqara_t1_fonster_vardagsrum`.
   - Type: `Window`
   - Arm modes: `Away`, `Night`
   - Delay: `Instant`
   - Zone: `Nedervåning`

> Ingen övervåningszon skapas.

---

## 5) Frient brandvarnare: rök + siren

## 5.1 Rökdetektor i Alarmo/automation

- Behåll båda Frient som separata `smoke`-entiteter.
- Dessa ska **inte** vara inbrottssensorer, utan brandtriggers.

## 5.2 Siren vid inbrott (modellberoende)

1. Kontrollera om Frient exponerar `siren.turn_on`:
   - **Developer Tools → States**
   - Sök på `frient` och kontrollera om någon `siren.` entity finns.
2. Om JA: använd den i automationen “Inbrott → Siren”.
3. Om NEJ: fallback till Hue-ljusblink + mobilnotis + köp separat Zigbee-siren.

### YAML: inbrott utlöst → siren (med fallback)

```yaml
alias: Alarmo - Inbrott siren och larmbelysning
mode: single
trigger:
  - platform: state
    entity_id: alarm_control_panel.hemlarm
    to: triggered
action:
  - choose:
      - conditions:
          - condition: template
            value_template: "{{ states('siren.frient_entre') not in ['unknown','unavailable',''] }}"
        sequence:
          - service: siren.turn_on
            target:
              entity_id: siren.frient_entre
    default: []
  - service: light.turn_on
    target:
      entity_id:
        - light.hue_lampa_1
        - light.hue_lampa_2
        - light.hue_lampa_3
        - light.hue_lampa_4
    data:
      flash: long
      brightness_pct: 100
      rgb_color: [255, 0, 0]
```

---

## 6) Placering enligt din planritning (säkerhet + täckning)

## 6.1 Brandvarnare

- **Frient 1 (entrévåning):** tak i passagen mellan hall/trappa och vardagsrum, minst 50 cm från vägg.
- **Frient 2 (övervåning):** tak i hall utanför sovrum.
- Undvik direkt placering i kök/badrum.

## 6.2 Sensorer för larmzon

- Huvuddörrkontakt på dörrblad + karm i överkant.
- Altandörrkontakt på den dörr som används mest (minimerar missar).
- Fönsterkontakt på mest åtkomligt fönster nedervåning (vardagsrum).

## 6.3 Zigbee mesh med Hue plug router

- Placera Hue smart plug i **entrévåning mittzon**, gärna vid trapp/hall-kant mot vardagsrum.
- Mål: korta hopp mellan ZBT-2 → router (plug) → battery-sensorer.
- Kör alltid ZBT-2 på USB-förlängare med avstånd från metall/chassi.

---

## 7) Presence-larm (2 telefoner) – exakt implementation

## 7.1 Koppla personer till device_trackers

1. Se till att båda mobilerna har HA Companion App.
2. På varje mobil: aktivera platsbehörighet `Always` + background updates.
3. I HA: **Settings → People → [Ditt namn] → Track device** och välj mobilens `device_tracker`.
4. Upprepa för partner.

## 7.2 Automation: båda borta → Arm Away

UI-väg:
- **Settings → Automations & Scenes → Create Automation → Start with empty automation**
- Namn: `Alarmo Auto Arm Away`
- Trigger: `State` på `person.du` till `not_home`
- Trigger 2: `State` på `person.partner` till `not_home`
- Condition 1: `person.du == not_home`
- Condition 2: `person.partner == not_home`
- Condition 3: `alarm_control_panel.hemlarm == disarmed`
- Action: `alarmo.arm` eller `alarm_control_panel.alarm_arm_away`

YAML:

```yaml
alias: Alarmo Auto Arm Away
mode: single
trigger:
  - platform: state
    entity_id: person.du
    to: not_home
  - platform: state
    entity_id: person.partner
    to: not_home
condition:
  - condition: state
    entity_id: person.du
    state: not_home
  - condition: state
    entity_id: person.partner
    state: not_home
  - condition: state
    entity_id: alarm_control_panel.hemlarm
    state: disarmed
action:
  - service: alarm_control_panel.alarm_arm_away
    target:
      entity_id: alarm_control_panel.hemlarm
```

## 7.3 Automation: någon kommer hem → Disarm

```yaml
alias: Alarmo Auto Disarm Home Arrival
mode: single
trigger:
  - platform: state
    entity_id: person.du
    to: home
  - platform: state
    entity_id: person.partner
    to: home
condition:
  - condition: not
    conditions:
      - condition: state
        entity_id: alarm_control_panel.hemlarm
        state: disarmed
action:
  - service: alarm_control_panel.alarm_disarm
    target:
      entity_id: alarm_control_panel.hemlarm
    data:
      code: !secret alarmo_code
```

> Om du inte vill lägga kod i YAML: använd Alarmos trusted users i stället.

---

## 8) Night mode (endast nedervåning)

1. **Settings → Automations & Scenes → Create automation**.
2. Trigger: tid `23:00` (justera efter rutin).
3. Conditions:
   - minst en person hemma (`person.du == home OR person.partner == home`)
   - larm inte redan armed.
4. Action: `alarm_control_panel.alarm_arm_night`.

YAML:

```yaml
alias: Alarmo Auto Night
mode: single
trigger:
  - platform: time
    at: "23:00:00"
condition:
  - condition: or
    conditions:
      - condition: state
        entity_id: person.du
        state: home
      - condition: state
        entity_id: person.partner
        state: home
  - condition: state
    entity_id: alarm_control_panel.hemlarm
    state: disarmed
action:
  - service: alarm_control_panel.alarm_arm_night
    target:
      entity_id: alarm_control_panel.hemlarm
```

---

## 9) Altandörr-automation 20:00–01:00

```yaml
alias: Altandörr kvällsljus
mode: restart
trigger:
  - platform: state
    entity_id: binary_sensor.aqara_t1_altandorr
    to: "on"
    id: opened
  - platform: state
    entity_id: binary_sensor.aqara_t1_altandorr
    to: "off"
    id: closed
condition:
  - condition: time
    after: "20:00:00"
    before: "01:00:00"
action:
  - choose:
      - conditions:
          - condition: trigger
            id: opened
        sequence:
          - service: light.turn_on
            target:
              entity_id: light.hue_lampa_entre
      - conditions:
          - condition: trigger
            id: closed
        sequence:
          - delay: "00:10:00"
          - condition: state
            entity_id: binary_sensor.aqara_t1_altandorr
            state: "off"
          - service: light.turn_off
            target:
              entity_id: light.hue_lampa_entre
```

---

## 10) Brandautomation (rök = ljus + notis + siren)

```yaml
alias: Brandlarm - full respons
mode: single
trigger:
  - platform: state
    entity_id:
      - binary_sensor.frient_smoke_entre
      - binary_sensor.frient_smoke_overvaning
    to: "on"
action:
  - service: light.turn_on
    target:
      entity_id: all
    data:
      brightness_pct: 100
      color_temp_kelvin: 6500
  - service: notify.mobile_app_din_telefon
    data:
      title: "BRANDLARM"
      message: "Rök detekterad. Kontrollera huset omedelbart."
  - service: notify.mobile_app_partner_telefon
    data:
      title: "BRANDLARM"
      message: "Rök detekterad. Kontrollera huset omedelbart."
  - choose:
      - conditions:
          - condition: template
            value_template: "{{ states('siren.frient_entre') not in ['unknown','unavailable',''] }}"
        sequence:
          - service: siren.turn_on
            target:
              entity_id: siren.frient_entre
```

---

## 11) Dashboarddesign (4 dashboards)

## 11.1 Huvuddashboard

Sektioner (Sections view):
- **Larmstatus**: Alarmo card + status-chip.
- **Lampor**: 4 Hue lampor + scenknappar.
- **Snabbkommandon**: Arm Away, Arm Night, Disarm, Dammsug nu.

## 11.2 Säkerhetsdashboard

- Alarmo panel högst upp.
- Grid med sensorer: huvuddörr, altandörr, fönster, smoke x2.
- Kamerarad: Eufy streams/snapshots (via integration/camera proxy).
- Incidentlogg: senaste 20 larmhändelser.

## 11.3 Mobil-dashboard

- 1 vy, 3 block:
  - Alarmo status + 3 stora knappar.
  - Viktiga lampor (entré, vardagsrum).
  - Kamera snabbtitt.

## 11.4 Family Hub kyl-dashboard (touchvänlig)

Layout:
- 2 kolumner, stora tile-kort.
- Övre rad: Alarmo status (stor badge), Arm/Disarm-knappar.
- Mitten: två kamerakort (entré + ute).
- Nedre: 4 stora ljusknappar + “Allt av”.

Praktiskt:
- Skapa separat dashboard: **Settings → Dashboards → Add Dashboard**.
- Namn: `Kylskärm`.
- Kryssa i `Show in sidebar` av (öppna via direktlänk på Family Hub browser).

---

## 12) Salus-termostater utan gateway: tydligt beslut

- **Fungerar utan gateway endast om exakt modell är Zigbee-stödd i ZHA/Zigbee2MQTT.**
- Om inte stödd: köp Salus gateway **eller** byt till Zigbee-termostater med bekräftat HA-stöd.

### Rekommenderad väg

1. Inventera exakt modellnummer på varje Salus-enhet.
2. Testpara en enhet mot ZBT-2.
3. Om intervju (temperatur + setpoint + hvac mode) inte exponeras stabilt inom 24–48 h: avbryt och välj gateway/byte.

---

## 13) Produktionshärdning (måste innan “skarpt läge”)

- Kör larmtest 2 gånger/vecka i 2 veckor.
- Sätt batterivarning på alla batterisensorer (<20%).
- Automation: “sensor unavailable > 30 min” → notis.
- Dokumentera fysisk placering + entity-id i ett markdown-dokument.
- Backup före varje större ändring.

---

## 14) Fallbacks när något inte fungerar

- Alarmo syns inte efter HACS-installation: restart HA + browser hard refresh.
- Presence opålitligt: aktivera både app GPS + router presence (composite via person).
- Frient siren saknas: använd separat Zigbee-siren för inbrott.
- Eufy livevideo instabil: använd stillbilder i säkerhetsdashboard, live först på mobil.

---

## 15) Prioriterad körordning (praktisk)

1. Namnsätt entiteter + areas.
2. Installera Alarmo.
3. Skapa två zoner exakt enligt ovan.
4. Lägg presence-arm/disarm.
5. Lägg night mode.
6. Lägg altandörr-ljus.
7. Lägg brandautomation.
8. Bygg 4 dashboards.
9. Kör stresstest 14 dagar.

Klart: då har du ett lokalt, robust och driftsäkert system utan överdesign.
