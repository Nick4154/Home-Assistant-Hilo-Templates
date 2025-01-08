# Home-Assistant-Hilo-Templates
Collection de templates et automations pour Home Assistant et Hilo. Inspiré de [https://github.com/dvd-dev/hilo](https://github.com/dvd-dev/hilo)

## Prérequis Lovelace
- [lovelace-mushroom](https://github.com/piitaya/lovelace-mushroom)
- [hilo par dvd-dev](https://github.com/dvd-dev/hilo)

- Une entité doit être créée (input_select.hvac_mode) avec 3 différentes options:
  - ete
  - hiver
  - defi-hilo-en-cours

Je me sert de cette entité pour allumer/éteindre le chauffage, l'air climatisé, ainsi que désactiver les automations pour entrée/sorties de zones pendant les défis Hilo.

## Cartes Lovelace
Voici les différentes cartes que j'ai créé. Un template complet pour la page est disponible sur ([lovelace-hilo.yaml](https://github.com/Nick4154/Home-Assistant-Hilo-Templates/blob/main/lovelace-hilo.yaml))

__Attention__: Toutes mes cartes utilisent des conditions pour l'affichage, selon l'état des défis et d'autres entités personnalisées. Libre à vous d'enlever les conditions mais les cartes risquent de mal fonctionner.
J'ai laissé les conditions sur place pour donner une idée de comment elles sont affichées. Je vais aussi laisser les différentes automations que j'utilisent pour les défis Hilo éventuellement, car elles sont très 
personnalisée pour mon train de vie/travail et ne seront pas utiles pour la plupart des utilisateurs.

### Mode de chauffage/ventilation
Cette carte affiche le mode de ventilation que Home Assistant suit (ex: en mode hiver, l'air clim ne sera jamais allumé. En mode été, les thermostats sont forcés à 5C et en mode Défi Hilo, les automations de HVAC sont désactivées
pour empêcher de changer les thermostats lorsque je part/j'arrive à la maison, etc)
```yaml
- type: custom:mushroom-template-card
  primary: >-
    {% if states('input_select.hvac_mode') == 'hiver'%}Mode de
    chauffage{% endif %}

    {% if states('input_select.hvac_mode') == 'ete'%}Mode de
    ventilation{% endif %}

    {% if states('input_select.hvac_mode') ==
    'defi-hilo-en-cours'%}Mode de chauffage{% endif %}
  secondary: >-
    {% if states('input_select.hvac_mode') == 'hiver'%}Hiver{% endif
    %}

    {% if states('input_select.hvac_mode') == 'ete'%}Été{% endif %}

    {% if states('input_select.hvac_mode') ==
    'defi-hilo-en-cours'%}Défi Hilo{% endif %}
  icon: mdi:home-thermometer
  fill_container: false
  layout: horizontal
  icon_color: >-
    {% if states('input_select.hvac_mode') == 'hiver'%}blue{% endif %}

    {% if states('input_select.hvac_mode') == 'ete'%}green{% endif %}

    {% if states('input_select.hvac_mode') ==
    'defi-hilo-en-cours'%}purple{% endif %}
- type: markdown
  content: >

    # Défi Hilo #{{ state_attr('sensor.defi_hilo',
    'next_events')[0]['event_id'] }}
  visibility:
    - condition: or
      conditions:
        - condition: state
          entity: sensor.defi_hilo
          state: scheduled
        - condition: state
          entity: sensor.defi_hilo
          state: appreciation
        - condition: state
          entity: sensor.defi_hilo
          state: pre_heat
        - condition: state
          entity: sensor.defi_hilo
          state: reduction
        - condition: state
          entity: sensor.defi_hilo
          state: recovery
```

### Carte pour date limite
Cette carte affiche la date/heure limite pour participer au prochain défi. Je l'affiche avec une entité "Participer au défi?" que je peux désactiver si je ne veux pas qu'Home Assistant joue avec les thermostats pendant un défi (ce qui va fort probablement échouer le défi au niveau Hilo)
```yaml
- type: custom:mushroom-template-card
  primary: Date limite pour participer
  secondary: >
    {% set next_event = state_attr('sensor.defi_hilo',
    'next_events')[0] %}

    {% set phases = next_event['phases'] %}

    {% set date_now = now().timestamp() %}

    {% set preheat_start = as_timestamp(phases['preheat_start']) %}


    {% if as_timestamp(state_attr('sensor.defi_hilo',
    'next_events')[0]['phases']['settings_deadline']) ==
    as_timestamp(states('sensor.date')) %}aujourd'hui à

    {% else %}

    demain à

    {% endif %}

    {{ preheat_start | timestamp_custom('%Hh') }}
  icon: ''
  visibility:
    - condition: or
      conditions:
        - condition: state
          entity: sensor.defi_hilo
          state: scheduled
- type: tile
  entity: input_boolean.defi_hilo_participation
  name: Participer au défi?
  visibility:
    - condition: or
      conditions:
        - condition: state
          entity: sensor.defi_hilo
          state: scheduled
```

### Carte phase en cours
Cette carte permet d'afficher la phase du défi:
- Programmé
- Appréciation 3 heures avant la phase de préchauffage
- Préchauffage
- Réduction
- Reprise
```yaml
- type: custom:mushroom-template-card
  primary: Phase en cours
  secondary: >-
    {% set next_event = state_attr('sensor.defi_hilo',
    'next_events')[0] %}

    {% set phases = next_event['phases'] %}

    {% set date_now = now().timestamp() %}

    {% set reduction_start = as_timestamp(phases['reduction_start'])
    %}

    {% set appreciation_start = as_timestamp(phases['preheat_start'])
    - (3 * 3600) %}

    {% set preheat_start = as_timestamp(phases['preheat_start']) %}

    {% set recovery_start = as_timestamp(phases['recovery_start']) %}

    {% set recovery_end = as_timestamp(phases['recovery_end']) %}

    {% set deadline = as_timestamp(phases['settings_deadline']) %}


    {% if date_now >= appreciation_start and date_now <= preheat_start
    %}
      Appréciation jusqu'à {{ preheat_start | timestamp_custom ('%Hh') }}
    {% elif date_now <= appreciation_start %}
      Programmé pour {{ appreciation_start | timestamp_custom('%Hh') }} (Appréciation)
    {% elif date_now >= preheat_start and date_now <= reduction_start
    %}
      Préchauffage jusqu'à {{ reduction_start | timestamp_custom('%Hh') }}
    {% elif date_now >= reduction_start and date_now <= recovery_start
    %}
      Réduction jusqu'à {{ recovery_start | timestamp_custom('%Hh') }}
    {% elif date_now >= recovery_start and date_now <= recovery_end %}
      Reprise jusqu'à {{ recovery_end | timestamp_custom('%Hh') }}
    {% endif %}
  icon: mdi:thermometer-chevron-down
  fill_container: false
  layout: horizontal
  icon_color: >-
    {% set next_event = state_attr('sensor.defi_hilo',
    'next_events')[0] %}

    {% set phases = next_event['phases'] %}

    {% set date_now = now().timestamp() %}

    {% set reduction_start = as_timestamp(phases['reduction_start'])
    %}

    {% set appreciation_start = as_timestamp(phases['preheat_start'])
    - (3 * 3600) %}

    {% set preheat_start = as_timestamp(phases['preheat_start']) %}

    {% set recovery_start = as_timestamp(phases['recovery_start']) %}

    {% set recovery_end = as_timestamp(phases['recovery_end']) %}

    {% set deadline = as_timestamp(phases['settings_deadline']) %}


    {% if date_now >= appreciation_start and date_now <= preheat_start
    %}
      yellow
    {% elif date_now <= appreciation_start %}
      grey
    {% elif date_now >= preheat_start and date_now <= reduction_start
    %}
      orange
    {% elif date_now >= reduction_start and date_now <= recovery_start
    %}
      blue
    {% elif date_now >= recovery_start and date_now <= recovery_end %}
      green
    {% endif %}
  grid_options:
    rows: 1
    columns: 12
  visibility:
    - condition: or
      conditions:
        - condition: and
          conditions:
            - condition: state
              entity: input_boolean.defi_hilo_participation
              state: scheduled
            - condition: state
              entity: input_select.hvac_mode
              state: defi-hilo-en-cours
        - condition: state
          entity: sensor.defi_hilo
          state: recovery
```

### Consommation du compteur
Cette carte permet d'afficher la consommation rapporté par Hilo.
Il y a aussi des thresholds pour l'icone, c'est à modifier selon le nombre de Watts et la couleur que vous souhaitez.
```yaml
- type: custom:mushroom-template-card
  primary: Consommation actuelle
  secondary: >-
    {{ (states('sensor.meter00_power') | int / 1000) | round(1,
    "ceil", default)}} kW
  icon: mdi:meter-electric
  icon_color: >-
    {% if states('sensor.meter00_power') | int < 1000 %}

    green

    {% elif states('sensor.meter00_power') | float >= 1000 and
    states('sensor.meter00_power') | float < 2000 %}

    yellow

    {% elif states('sensor.meter00_power') | float >= 2000 and
    states('sensor.meter00_power') | float < 3500%}

    orange

    {% elif states('sensor.meter00_power') | int >= 3500 %}

    red

    {% endif %}
  grid_options:
    columns: 9
    rows: 1
  fill_container: false
  multiline_secondary: false
  tap_action:
    action: more-info
  entity: sensor.meter00_power
  visibility:
    - condition: state
      entity: input_select.hvac_mode
      state: defi-hilo-en-cours
```

### Cartes récompenses et Consommation défi
Ces deux cartes affichent le montant estimé que le défi vous donnera (le montant commence élevé et baisse au fur et à mesure que la partie allouée du défi est consommée), ainsi que la consommation utilisé/totale du défi en pourcentage et en kWh
```yaml
- type: custom:mushroom-template-card
  primary: Récompense
  secondary: >-
    {{(( state_attr('sensor.defi_hilo',
    'next_events')[0]['allowed_kWh'] - state_attr('sensor.defi_hilo',
    'next_events')[0]['used_kWh']) * 0.55) | round(2) }} $
  icon: mdi:cash
  grid_options:
    columns: 5
    rows: 1
  visibility:
    - condition: or
      conditions:
        - condition: state
          entity: sensor.defi_hilo
          state: reduction
        - condition: state
          entity: sensor.defi_hilo
          state: recovery
    - condition: state
      entity: input_boolean.defi_hilo_participation
      state: 'on'
  icon_color: green
- type: custom:mushroom-template-card
  primary: Consommation défi
  secondary: >-
    {{ state_attr('sensor.defi_hilo',
    'next_events')[0]['used_percentage']}}% -  

    {{ state_attr('sensor.defi_hilo',
    'next_events')[0]['used_kWh']}}/{{ state_attr('sensor.defi_hilo',
    'next_events')[0]['allowed_kWh']}} kWh
  icon: mdi:lightning-bolt
  icon_color: >-
    {% if state_attr('sensor.defi_hilo',
    'next_events')[0]['used_percentage'] < 20 %}

    green

    {% elif state_attr('sensor.defi_hilo',
    'next_events')[0]['used_percentage'] <= 20 and
    state_attr('sensor.defi_hilo',
    'next_events')[0]['used_percentage'] < 30 %}

    yellow

    {% elif state_attr('sensor.defi_hilo',
    'next_events')[0]['used_percentage'] <= 30 and
    state_attr('sensor.defi_hilo',
    'next_events')[0]['used_percentage'] < 40 %}

    orange

    {% elif state_attr('sensor.defi_hilo',
    'next_events')[0]['used_percentage'] <= 50 %}

    red

    {% endif %}
  grid_options:
    columns: 7
    rows: 1
  visibility:
    - condition: or
      conditions:
        - condition: state
          entity: sensor.defi_hilo
          state: reduction
        - condition: state
          entity: sensor.defi_hilo
          state: recovery
    - condition: state
      entity: input_boolean.defi_hilo_participation
      state: 'on'
```

### Carte des défis
Carte des défis par dvd-dev, avec un peu de modifications, que j'affiche lorsqu'il n'y a pas de défis en cours et que l'entité "season" est en mode "winter"
```yaml
- type: grid
  cards:
    - type: markdown
      content: |-
        {% 

        set data = state_attr('sensor.recompenses_hilo','history')[0] 

        %}
        {% if data is defined %}

          <h2><center>Défis Hilo - Saison  {{data.season}}-{{data.season + 1}}</center></h2>  
          <ha-alert alert-type="success"><h3>
            Nb de défis: {{data.events | count}}<br>
            Récompense totale: {{"%.2f" | format(data.totalReward) }}$<br>
            Récompense moyenne: {{"%.2f" | format(data.totalReward /(data.events | count)) }}$<br>
            Énergie sauvée: {{ (state_attr('sensor.recompenses_hilo', 'history')[0]['totalEnergySavingsWh'] | float / 1000) | round(1, "ceil", default)}} kWh
            
          </h3></ha-alert>      
          <br>

          <table width=100%>
            <tr>
              <th>#</th>
              <th>Date</th>
              <th>Plage</th>
              <th>Alloué</th>
              <th>Utilisé</th>
              <th>Prime</th>
            </tr>  

            {%- for event in data.events|reverse -%}    
              <tr>
              <td align=center>{{ "%02i"|format(loop.revindex) }}</td>
              <td align=center>{{ event.phases.reduction_start.timestamp() | timestamp_custom('%b %d (%a)') }}</td>
              <td align=center>{{ "Matin" if (event.period =="am") else "Soir" }}</td>
              <td align=center>{{ "%.1f"|format(event.allowed_kWh) }} kWh</td> 
              <td align=center>{{ "%.1f"|format(event.used_kWh) }} kWh</td>      
              <td align=center>{{ "%.2f"|format(max(0,event.allowed_kWh - event.used_kWh) * 0.55) }}$</td>
              </tr>
            {%- endfor -%}

          </table>

        {% else %}
        aucune donnée disponible
        {% endif %}
  visibility:
    - condition: state
      entity: sensor.season
      state: winter
    - condition: and
      conditions:
        - condition: state
          entity: sensor.defi_hilo
          state_not: pre_heat
        - condition: state
          entity: sensor.defi_hilo
          state_not: reduction
        - condition: state
          entity: sensor.defi_hilo
          state_not: recovery
```

# Changelog
- 07/01/2025 - Ajout des cartes pour le groupe Facebook
