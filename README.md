# Scent-a-Tron Project


# ESPHome code for my 'Scent-a-Tron' Project

```yaml
substitutions:
  room: your_room_name_here # Change to your Room
  
  bright_level: "0.1"  # Enter Bright Level
  dark_level: "1.0"    # Enter Dark Level

globals:
  - id: sprays
    type: float
    restore_value: yes
    initial_value: "0"

esphome:
  name: freshmatic_${room}
  platform: ESP8266
  board: d1_mini

  ## Change your Wi-Fi Details here ##
wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

logger:
  logs:
    adc: none
    sensor: none
    light: none
api:
ota:

binary_sensor:
  - platform: gpio
    id: reset
    pin: D4
    filters:
      - invert:
    on_press:
      then:
        - light.turn_on:
            id: led
            effect: flashfast
        - delay: 1s
        - light.turn_off: led
    on_release:
        - light.turn_off: led
    on_click:
      min_length: 900ms
      max_length: 3000ms
      then:
        - lambda: |-
            id(sprays) = 0;

sensor:
  - platform: template
    id: ${room}_can_level
    name: Freshmatic ${room} %
    device_class: pressure
    unit_of_measurement: "Can"
    update_interval: 1s
    lambda: |-
      return ( (2500-id(sprays) )/2500 ) * 100;

  - platform: homeassistant
    entity_id: input_select.freshmatic_${room}
    id: selector
    on_value:
      - if:
          condition:
            - switch.is_on: enable
            - lambda: "return id(selector).state > 1;"
          then:
            - switch.turn_on: spray

  - platform: adc
    id: ${room}_brightness
    pin: A0
    name: Freshmatic ${room} Lux
    update_interval: 2s
    device_class: illuminance
    unit_of_measurement: lx
    
    ### -- Un-Comment these after Calibrating -- ###
    filters:
      # - calibrate_linear:
      #     - ${bright_level} -> 1.0
      #     - ${dark_level} -> 0.0
      # - multiply: 100
    ### -------------[ Use Ctrl+? ]------------- ###
    
    on_value:
      - if:
          condition:
            for:
              time: 20000s
              condition:
                and:
                  - lambda: "return id(${room}_brightness).state < 10;"
                  - switch.is_off: its_dark
          then:
            - switch.turn_on: its_dark
      - if:
          condition:
            and:
              - lambda: "return id(${room}_brightness).state > 10;"
              - switch.is_on: enable
              - lambda: "return id(selector).state < 1;"
              - switch.is_on: its_dark
          then:
            - switch.turn_on: spray
            - delay: 5s
            - switch.turn_off: its_dark

switch:
  - platform: gpio
    name: Freshmatic ${room} Spray
    id: spray
    icon: mdi:spray
    pin: D2
    on_turn_on:
      - light.turn_on: led
      - switch.turn_on: timer
      - delay: 0.25s
      - switch.turn_off: spray
      - lambda: |-
          id(sprays) += 1;

  - platform: template
    id: enable
    name: Freshmatic ${room} Enable
    optimistic: true
    on_turn_on:
      - switch.turn_on: timer

  - platform: template
    id: its_dark
    icon: mdi:brightness-4
    optimistic: true

  - platform: template
    id: timer
    optimistic: true
    on_turn_on:
      - delay: !lambda return id(selector).state* (60000);
      - switch.turn_off: timer
      - if:
          condition:
            - switch.is_on: enable
            - lambda: "return id(selector).state > 1;"
          then:
            - light.turn_on:
                id: led
                effect: pulse
            - delay: 1.0s
            - light.turn_off: led
            - switch.turn_on: spray

light:
  - platform: monochromatic
    output: out_led
    default_transition_length: 0s
    id: led
    effects:
      - pulse:
          transition_length: 0.0s
          update_interval: 0.05s
      - pulse:
          name: flashfast
          transition_length: 0.0s
          update_interval: 0.025s
    on_turn_on:
      - delay: 2s
      - light.turn_off: led

output:
  platform: esp8266_pwm
  pin:
    number: D3
    inverted: true
  id: out_led
```
___
#### 💖 Found this useful, want to say '*Thanks*' and support my efforts. *CHEERS*🍺
| Buy me a Coffee | PATREON |
|-----------------|---------|
| https://www.buymeacoffee.com/3ative | https://www.patreon.com/3ative |
