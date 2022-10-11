# inimal
INIM - Allarme

https://community.home-assistant.io/t/inim-alarm/60354/56

Necessaria SmartLanG – credenziali default: admin pass 
Stato dei sensori:	
rest:
  - resource:http://192.165.2.223/cgi-bin/web.cgi?mod=zone&par=now&user=youruser&pass=yourpass&code=yourcode
    scan_interval: 20
    timeout: 20
    sensor:
      - name: "Inim Vol Garage"
        json_attributes_path: "$.zone[0]"
        value_template: >
          {% set vg = value_json.zone[0].st %}
          {% if vg == '1' %}
          Libero
          {% elif vg == '2' %}
          Rilevato
          {% else %}
          ...Non Definito...
          {% endif %}
        json_attributes:
          - lb
          - st
          - tl
      - name: "Inim Sabotaggio Sirena Esterna"
        json_attributes_path: "$.zone[1]"
        value_template: >
          {% set sbs = value_json.zone[1].st %}
          {% if sbs == '1' %}
          Ok
          {% elif sbs == '2' %}
          Sabataggio
          {% else %}
          ...Non Definito...
          {% endif %}
        json_attributes:
          - lb
          - st
          - tl

### il comando zone&par=now serve per recuperare lo stato

 
Stato centrale:
sensor:
  - platform: rest
    name: Stato Inim
    resource: http://192.165.2.223/cgi-bin/web.cgi?mod=sce&par=now&user=youruser&pass=yourpass&code=yourcode
    value_template: >
      {% set s = value_json.lb %}
      {% if s == 'Disinserimento  ' %}
      Disattivato
      {% elif s == 'Totale          ' %}
      - Totale -
      {% elif s == 'Parziale        ' %}
      - Parziale -
      {% elif s == 'Solo Giardino   ' %}
      Solo Giardino
      {% elif s == 'Volumetrici int  ' %}
      Volumetrici Interni
      {% else %}
      ...Non Definito...
      {% endif %}
    scan_interval: 5
    timeout: 4

### il comando sce&par=now serve per recuperare lo stato


Inserimento scenario:
rest_command:
  inim_off: 
    url: "http://192.165.2.223/cgi-bin/web.cgi?mod=do_sce&par=1&user=youruser&pass=yourpass&code=yourcode”
    method: GET
  inim_tot:
    url: "http://192.165.2.223/cgi-bin/web.cgi?mod=do_sce&par=0&user= youruser&pass= ourpass&code=yourcode”
    method: GET
  inim_par: 
    url: "http://192.165.2.223/cgi-bin/web.cgi?mod=do_sce&par=2&user= youruser&pass= yourpass&code=yourcode”
    method: GET

### il comando do_sce&par=0 serve per inserire il primo scenario
### il comando do_sce&par=1 serve per inserire il secondo scenario
### il comando do_sce&par=2 serve per inserire il terzo scenario

 
Inserimento dal modulo allarm di home assistant tramite due automazioni in modo che se l’allarme dovesse essere inserito da altri apparecchi o app il pannello di home assistant venga aggiornato:

automation:
- alias: A_Inim inserimento
  trigger:
    platform: state
    entity_id: alarm_control_panel.inim
  action:
    service: >
      {% if is_state("alarm_control_panel.inim", "disarmed") %} rest_command.inim_off
      {% elif is_state("alarm_control_panel.inim", "armed_home") %} rest_command.inim_par
      {% elif is_state("alarm_control_panel.inim", "armed_away") %} rest_command.inim_tot
      {% endif %}

- alias: A_Inim update panel
  trigger:
    platform: state
    entity_id: sensor.stato_inim
  action:
    service: >
      {% if is_state("sensor.stato_inim", "Disattivato") %} 
      alarm_control_panel.alarm_disarm
      {% elif is_state("sensor.stato_inim", "- Parziale -") %}
      alarm_control_panel.alarm_arm_home
      {% elif is_state("alarm_control_panel.inim", "armed_away") %}
      alarm_control_panel.alarm_arm_away
      {% endif %}
    data:
      code: "code"
    target:
      entity_id: alarm_control_panel.inim
