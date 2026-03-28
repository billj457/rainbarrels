# rainbarrels
220 gallon rain barrel array with level sensor and 2 zone watering automation (MQTT, Home Assistant)

esphome:
  name: "rain-barrel-monitor-d7ba38"
  friendly_name: Rain Barrel Monitor
  min_version: 2025.11.0
  name_add_mac_suffix: false

esp32:
  board: esp32dev
  framework:
    type: arduino

logger:
  level: INFO

api:

ota:
  - platform: esphome

uart:
  id: uart_bus
  tx_pin: GPIO16
  rx_pin: GPIO17
  baud_rate: 9600

sensor:
  - platform: "a02yyuw"
    name: "Rain Barrel Distance"
    id: distance_mm
    uart_id: uart_bus
    unit_of_measurement: "mm"
    accuracy_decimals: 0
    filters:
      - filter_out: 0
      - median:
          window_size: 7
          send_every: 1
      - throttle: 60s

  - platform: template
    name: "Rain Barrel Level Percent"
    id: barrel_percent
    unit_of_measurement: "%"
    state_class: measurement
    device_class: battery
    lambda: |-
      float air_gap = id(distance_mm).state;
      if (std::isnan(air_gap)) return {}; 

      float dist_empty = 847.7; 
      
      // Change '50.8' to whatever the air gap is when water hits the overflow
      float dist_full = 50.8; // 2 inches of air gap at full

      float pct = ((dist_empty - air_gap) / (dist_empty - dist_full)) * 100.0;

      if (pct < 0) return 0;
      if (pct > 100) return 100;
      return pct;

  - platform: template
    name: "Rain Barrel Gallons"
    unit_of_measurement: "gal"
    state_class: measurement
    lambda: |-
      if (std::isnan(id(barrel_percent).state)) return {};
      // 220 gallons total capacity
      return (id(barrel_percent).state / 100.0) * 220.0;

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
