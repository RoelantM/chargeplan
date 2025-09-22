# Home Assistant – EPEX Trading with Frank Energie

This repository contains configuration examples and dashboards for integrating **EPEX trading logic** into Home Assistant, using **Frank Energie dynamic pricing**.

---

## 1. Install Frank Energie Integration

We rely on the [Frank Energie Home Assistant integration](https://github.com/bajansen/home-assistant-frank_energie).  
Please follow the instructions in that repository to get the `sensor.current_electricity_price_all_in` entity available in your setup.

---

## 2. Configuration

Add the following to your `configuration.yaml`:

### Input helpers

```yaml
input_number:
  epex_min_spread_eur_kwh:
    name: Min spread (€/kWh)
    min: 0
    max: 0.5
    step: 0.01
    unit_of_measurement: €/kWh
    icon: mdi:chart-bell-curve-cumulative
    initial: 0.08

input_text:
  epex_plan_compact:
    name: EPEX Plan (compact)
    icon: mdi:code-json
    max: 255

input_text:
  epex_plan_proposed_compact:
    name: EPEX Plan proposed (compact)
    icon: mdi:code-json
    max: 255
```

**epex_min_spread_eur_kwh** is the spread that we maintain for building blocks.
**epex_plan_proposed_compact** is automatically built by an automation, which will generate a chargeplan.
**epex_plan_compact** will contain the desired plan. Is used for execution in the "Energy Manager".

### Template sensors
```
- sensor:
    - name: epex_plan_compact_parsed
      state: "{{ states('input_text.epex_plan_compact') }}"
      attributes:
        windows: >-
          {% set entries = states('input_text.epex_plan_compact').split(';') %}
          {% set out = namespace(list=[]) %}
          {% for entry in entries %}
            {% if entry | length >= 12 %}
              {% set y = '20' ~ entry[0:2] %}
              {% set m = entry[2:4] %}
              {% set d = entry[4:6] %}
              {% set start_hour = entry[6:8] %}
              {% set end_hour = entry[9:11] %}
              {% set action = entry[11] %}
              {% set from = y ~ '-' ~ m ~ '-' ~ d ~ 'T' ~ start_hour ~ ':00:00' %}
              {% set till = y ~ '-' ~ m ~ '-' ~ d ~ 'T' ~ end_hour ~ ':00:00' %}
              {% set item = {
                  'from': from,
                  'till': till,
                  'action': 'charge' if action == 'C' else ('discharge' if action == 'D' else ('explicit' if action == 'E'))
              } %}
              {% set out.list = out.list + [item] %}
            {% endif %}
          {% endfor %}
          {{ out.list }}

- sensor:
    - name: epex_plan_proposed_compact_parsed
      state: "{{ states('input_text.epex_plan_proposed_compact') }}"
      attributes:
        windows: >-
          {% set entries = states('input_text.epex_plan_proposed_compact').split(';') %}
          {% set out = namespace(list=[]) %}
          {% for entry in entries %}
            {% if entry | length >= 12 %}
              {% set y = '20' ~ entry[0:2] %}
              {% set m = entry[2:4] %}
              {% set d = entry[4:6] %}
              {% set start_hour = entry[6:8] %}
              {% set end_hour = entry[9:11] %}
              {% set action = entry[11] %}
              {% set from = y ~ '-' ~ m ~ '-' ~ d ~ 'T' ~ start_hour ~ ':00:00' %}
              {% set till = y ~ '-' ~ m ~ '-' ~ d ~ 'T' ~ end_hour ~ ':00:00' %}
              {% set item = {
                  'from': from,
                  'till': till,
                  'action': 'charge' if action == 'C' else ('discharge' if action == 'D' else ('explicit' if action == 'E'))
              } %}
              {% set out.list = out.list + [item] %}
            {% endif %}
          {% endfor %}
          {{ out.list }}
```
These sensors will be used for displaying the plans over the chart from Frank Energy with Apex cards.  

## 3. Value format

Plans are encoded in a compact string format:
25092207-09D;25092212-15C;25092219-21D;

**Format**:  
yymmddHH-hhM  

Where:  
**yy** = year (25 → 2025)  
**mm** = month (09 → September)  
**dd** = day (22)  
**HH** = start hour (24h)  
**hh** = end hour (24h)  
**M** = mode:  
C = Charge  
D = Discharge  
E = Explicit use => should be used in cases where you do not want to wear out batteries, instead, use the net grid for housekeeping.  

So in the example we start charging on 22 sept 2025 at 0700, till 0900. Start another charge session at 22 sept 2025 at 1200, till 1500. Then discharge at 1900, till 2100.


## 4. Automations  
Rebuild the plan automatically when new Frank Energie prices are available:  

```
alias: EPEX – Auto rebuild plan on price update
triggers:
  - trigger: state
    entity_id: sensor.current_electricity_price_all_in
    attribute: prices
conditions:
  - condition: template
    value_template: >
      {{ (state_attr('sensor.current_electricity_price_all_in','prices') | default([])) | count > 0 }}
actions:
  - action: script.epex_build_plan
mode: single
```

**epex_build_plan** is in the repo.

## 5. Charts
We use the [ApexCharts Card](https://github.com/RomRider/apexcharts-card) to visualize prices and actions.  

**Confirmed plan** (epex_plan_compact_parsed)
```
type: custom:apexcharts-card
graph_span: 48h
span:
  start: day
now:
  show: true
  label: Nu
header:
  show: false
  title: Energieprijs per uur (€/kWh)
series:
  - entity: sensor.current_electricity_price_all_in
    name: All in
    type: column
    opacity: 0.3
    stroke_width: 2
    color: "#03b2cb"
    data_generator: |
      return entity.attributes.prices.map((r) => [r.from, r.price]);
  - entity: sensor.epex_plan_compact_parsed
    name: Charge
    type: column
    color: "#8BC34A"
    data_generator: >
      const plan = entity?.attributes?.windows || [];
      const prices = hass.states['sensor.current_electricity_price_all_in']?.attributes?.prices || [];
      const inWin = (ts) => plan.some(w => ts >= new Date(w.from).getTime() && ts < new Date(w.till).getTime() && w.action === 'charge');
      return prices.map(r => inWin(new Date(r.from).getTime()) ? [r.from, r.price] : [r.from, null]);
  - entity: sensor.epex_plan_compact_parsed
    name: Explicit use
    type: column
    color: "#AA"
    data_generator: >
      const plan = entity?.attributes?.windows || [];
      const prices = hass.states['sensor.current_electricity_price_all_in']?.attributes?.prices || [];
      const inWin = (ts) => plan.some(w => ts >= new Date(w.from).getTime() && ts < new Date(w.till).getTime() && w.action === 'explicit');
      return prices.map(r => inWin(new Date(r.from).getTime()) ? [r.from, r.price] : [r.from, null]);
  - entity: sensor.epex_plan_compact_parsed
    name: Discharge
    type: column
    color: "#FF7043"
    data_generator: >
      const plan = entity?.attributes?.windows || [];
      const prices = hass.states['sensor.current_electricity_price_all_in']?.attributes?.prices || [];
      const inWin = (ts) => plan.some(w => ts >= new Date(w.from).getTime() && ts < new Date(w.till).getTime() && w.action === 'discharge');
      return prices.map(r => inWin(new Date(r.from).getTime()) ? [r.from, r.price] : [r.from, null]);
```
<img width="472" height="304" alt="image" src="https://github.com/user-attachments/assets/3a4aaa0b-6f16-4b0e-9649-6186d27286f9" />

**Proposed plan** (epex_plan_proposed_compact_parsed)  
Use the same structure, but replace epex_plan_compact_parsed with epex_plan_proposed_compact_parsed.

**Expex Build Plan**  
Is getting the attributes from Frank Energy:  
<img width="1116" height="802" alt="image" src="https://github.com/user-attachments/assets/b9eac79c-734f-4396-a490-9fb1f950afa3" />  

These values are being used to build pairs. Now we are dividing it into 2 blocks, night (starting 2200) till 10.00, and from 10.00 till 22.00. I was working on fully automatic of getting a better spread, but wasn't properly working. This is WIP and obviously should be used at own risk ;)


## 6. Energy Manager
After the compact value has been created it doesn't work out of the box obviously. I have built my own automation, trigger on /30 seconds. This will probably be different for every user depending on the sensors they have. I will put in mine here (and is already changed 20 times), see EnergyManager.yml. (still WIP, but you get the point)
