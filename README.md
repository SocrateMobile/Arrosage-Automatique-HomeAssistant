# Arrosage-Automatique-HomeAssistant

Création d un système d arrosage automatique, piloté via Home Assistant
Après quelques années passées sous Jeedom, j’ai migré ma domotique sur HA en mars dernier.
Je vous partage mon projet actuel le plus abouti: l’arrosage automatique.
Le projet vise à remplacer mon système rainbird, dont le dongle wifi à rendu l’âme après deux saisons et demi …
En synthèse:
utiliser un ESP8266 pour contrôler:
- une plaque de 8 relais, 5 utilisés dans le montage sur les GPIO D0,D1,D2,D3,D4
- 2 capteurs de température Dallas DS18B20 (le premier intérieur au boîtier, l’autre pour la température extérieure), sur D7 et le GRD
- 1 capteur d’humidité du sol (yl38), sur A0 et sur le GRD
- deux boutons poussoir pour lancer une automatisation :
      5 minutes d’arrosage de toutes les zones sur D5 et sur le GRD
      10 minutes d’arrosage de toutes les zones sur D6 et sur le GRD
- Un troisième bouton sera cablé sur le GPIO RST & GND: un appuie, au reboot toutes les vannes seront éteintes

Et dans un second temps, la création de trois cartes dans HA pour piloter le tout:

![alt text](https://github.com/SocrateMobile/Arrosage-Automatique-HomeAssistant/blob/main/3cartes.jpeg)

la première carte est a utiliser pour soit lancer une séquence d’arrosage de 5mn par zone, soit 10mn, et au cas ou… un stop
la deuxième, sert visualiser les zones d arrosage, et a les démarrer / arrêter manuellement
la troisième est à mon sens la plus intéressante: le temps d arrosage de chaque zone peut être ajusté entre 1 et 20mn, et on lance l enchainement des séquences

Commencons...

dans HA il faut créer 5 entrées nommées tempo_temps1 à tempo_temps5 ( 8 si vous en avez 8 vannes)

![alt text](https://github.com/SocrateMobile/Arrosage-Automatique-HomeAssistant/blob/main/5entrées.JPG)


maintenant, nous allons définir les variables pour le temps d'arrosage :
un timmer propre a chaque zone pour un mode personnalisé, 
Ce code est à coller dans un fichier nommé "input_number.yaml", à mettre dans le même répertoire que le fichier configuration.yaml
Ensuite, dans le fichier configuration.yaml il faut ajouter : input_number: !include input_number.yaml

```
  tempo_temps1:
    name: tempo_temps1
    initial: 1
    min: 1
    max: 20
    step: 1
    unit_of_measurement: "min"

  tempo_temps2:
    name: tempo_temps2
    initial: 1
    min: 1
    max: 20
    step: 1
    unit_of_measurement: "min"
    
  tempo_temps3:
    name: tempo_temps3
    initial: 1
    min: 1
    max: 20
    step: 1
    unit_of_measurement: "min"
    
  tempo_temps4:
    name: tempo_temps4
    initial: 1
    min: 1
    max: 20
    step: 1
    unit_of_measurement: "min"
    
  tempo_temps5:
    name: tempo_temps5
    initial: 1
    min: 1
    max: 20
    step: 1
    unit_of_measurement: "min"
```

A cette étape, il vous faut éditer ou créer le fichier secret.yaml DANS le répartoir config / esphome de votre Home Assistant
et y intégrer :
```
key_arrosage: "votre token API de votre ESP"
ota_arrosage: "votre token OTA"
wifi_ssid: "le nom de votre réseau WIFI de la maison"
wifi_password: "la clé de votre réseau wifi de la maison"
mdp_hotspot: "la clé du hotspot de votre ESP en cas de plantage"
server_username: le login que vous souhaitez pour accéder à l interface WEB de votre ESP
server_password: le MDP que vous souhaitez pour accéder à l interface WEB de votre ESP
```

Voici maintenant le code pour l ESP8266, à mettre dans ESPHome.
C est mon premier code, il est donc certainement optimisable
je l’ai créé a partir de la doc ESPHome, de divers projets glanés sur le net, mais également en tâtonnant avec ChatGPT que je découvre en même temps
(il peut donner des pistes, mais rarement une solution a appliquer directement, mais cela m a aidé)

```
# ok
# code v3 ARROSAGE AUTOMATIQUE
# Controle d'une plaque de 8 relais, 5 utilisés dans le montage, GPIO D0,D1,D2,D3,D4
# 2 capteurs de température Dallas DS18B20 (un interrieur au boitier, un température exterieure), sur D7
# 1 capteur d'humidité du sol (yl38), sur A0
# deux boutons poussoir pour lancer une automatisation :
  # 5 minutes d'arrosage de toutes les zones sur D5
  # 10 minutes d'arrosage de toutes les zones sur D6
# Un troisieme bouton sera cablé sur le GPIO RST & GND: un appuie, au reboot toutes les vannes seront eteintes
# ici, on commence par configurer les différentes variables, cela facilite la duplication ;-)

substitutions:
# nom des Zones d'arrosage
  switch1_name: 1🎍Les Bambous
  switch2_name: 2🌱Le Jardin côté droit
  switch3_name: 3🌱Le Jardin côté gauche
  switch4_name: 4🌹Les Bordures
  switch5_name: 5🍅 Le Potager
  switch6_name: 6 Libre
  switch7_name: 7 Libre
  switch8_name: 8 Libre

# ici, on donne les PIN du GPIO 8266 qui seront utilisés
  # Pour les relais
  pin_switch1: D0
  pin_switch2: D1
  pin_switch3: D2
  pin_switch4: D3
  pin_switch5: D4
  pin_bouton1: D5
  pin_bouton2: D6
  # Pour le capteur de température interieur & exterieur:
  pin_dallas: D7
  # Pour le capteur d'humidité du sol:
  pin_moisure: A0

# true ou flase en fonction de votre relay et des boutons, afin d aligner l état remonté sur l'état physique ( allumé/éteint )
  is_inverted: "true"
  button_inverted: "true"

# Configuration de l IP fixe, ici en "2"
  ip_device: 192.168.1.2
  ip_gateway: 192.168.1.254
  ip_subnet:  255.255.255.0

# Nom de l ESP qui sera vible sur votre box (en minuscules)
  esphome_name: gpioarrosagev3

# Frendly name qui sera visible dans Home Assistant
  devicename: GPIO Arrosage V3

# Timer des automatismes (ici, 5 minutes par Zone pour le premier et 10 minutes pour le second)
  Temps1: "5"
  Temps2: "10"

# Nom des deux boutons possoirs qui feront démarrer deux automatismes
  bouton1_name: ${Temps1} minutes d'arrosage de toutes les zones 
  bouton2_name: ${Temps2} minutes d'arrosage de toutes les zones

# ici, on indique le nom des icones à utiliser pour que cela soit joli dans home assistant
  switch_icon: sprinkler-variant
  switch_remote: remote
  switch_reboot: restart
  pushbutton1: pushbutton1
  pushbutton2: pushbutton2
# ici, on donne le Numéro de série des capteurs de température dallas DS18B20
  dallas_int: "0x64041722a8XXXXXX" #adresse de votre DS18B20 interieur
  dallas_ext: "0xcb0214814YYYYYYY" #adresse de votre DS18B20 exterieure

# dans le reste du code, tout est variabilisé sur les labels de subsitution

esphome:
  name: ${esphome_name}
  friendly_name: ${devicename}
  on_boot:
    priority: -10
    then:
        - switch.turn_off: relay1
        - switch.turn_off: relay2 
        - switch.turn_off: relay3   
        - switch.turn_off: relay4
        - switch.turn_off: relay5
        
esp8266:
  board: nodemcuv2

# Enable logging
logger:

# Enable Home Assistant API
# créez ici dans le repertoir EPSHome un fichier secret.yaml qui comportera les valeurs sensibles comme vos mots de passe
# le déclaratif est simple à réaliser , dans le fichier secret.yaml, ajouter une ligne key_arrosage: la clé de votre API, 
# ota_arrosage: l'OTA de votre arrosage etc

api:
  encryption:
    key: !secret key_arrosage

ota:
  - platform: esphome
    password: !secret ota_arrosage

# Ici, on lance la connexion de l'ESP8266 à votre WIFI, 
# les paramètres sont à remplir plus haut, et dans le fichier sercet.ymal

wifi:
  ssid: !secret wifi_ssid_arrosage
  password: !secret wifi_password_arrosage
  manual_ip:
    static_ip: ${ip_device}
    gateway: ${ip_gateway}
    subnet: ${ip_subnet}

# Enable fallback hotspot (captive portal) in case wifi connection fails
# En cas de plantage, un point d'acces est créé
  ap:
    ssid: "${devicename} Hotspot"
    password: !secret mdp_hotspot

# Ici, on active le serveur WEB du ESP8266, en allant sur l IP du ESP8266, on tombe sur une interface pour piloter les relais
web_server:
  port: 80
  auth:
    username: !secret server_username
    password: !secret server_password
    
mqtt:
  broker: 192.168.1.5
  username: !secret server_username
  password: !secret mqtt_password
  discovery: False # disable entity discovery
  discover_ip: True # enable device discovery


captive_portal:
# Enable Home Assistant API

text_sensor:
- platform: version
  name: "${devicename} ESPHome Version"
  hide_timestamp: true

- platform: wifi_info
  ip_address:
    name: "${devicename} IP Address"

# Example configuration entry
# Ici , c est la récupération de la température interieure et exterieure 

one_wire:
  - platform: gpio
    pin: ${pin_dallas}  # Replace with your actual GPIO pin

globals:
  - id: temps1
    type: int
    restore_value: no 
    initial_value: '0' 

  - id: temps2
    type: int
    restore_value: no
    initial_value: '0' 

  - id: temps3
    type: int
    restore_value: no 
    initial_value: '0' 

  - id: temps4
    type: int
    restore_value: no
    initial_value: '0' 

  - id: temps5
    type: int
    restore_value: no 
    initial_value: '0' 


# Individual sensors
sensor:
  - platform: dallas_temp
    address: ${dallas_int}
    name: "Temperature Interieure"
  - platform: dallas_temp
    address: ${dallas_ext}
    name: "Temperature Exterieure"
    
  - platform: homeassistant
    id: tempo_temps1
    entity_id: input_number.tempo_temps1
    on_value:
      then:
        - globals.set:
            id: temps1
            value: !lambda 'return int(x);'
  
  - platform: homeassistant
    id: tempo_temps2
    entity_id: input_number.tempo_temps2
    on_value:
      then:
        - globals.set:
            id: temps2
            value: !lambda 'return int(x);'
  - platform: homeassistant
    id: tempo_temps3
    entity_id: input_number.tempo_temps3
    on_value:
      then:
        - globals.set:
            id: temps3
            value: !lambda 'return int(x);'
  
  - platform: homeassistant
    id: tempo_temps4
    entity_id: input_number.tempo_temps4
    on_value:
      then:
        - globals.set:
            id: temps4
            value: !lambda 'return int(x);'

  - platform: homeassistant
    id: tempo_temps5
    entity_id: input_number.tempo_temps5
    on_value:
      then:
        - globals.set:
            id: temps5
            value: !lambda 'return int(x);'




  # Ici , c est la récupération de l humidité du sol 
  - platform: adc
    pin: ${pin_moisure}
    name: "Humidité du sol"
    id: hum_sol
    update_interval: 60s
    unit_of_measurement: "%"
    filters:
    - multiply: 100
 #   - median:
 #       window_size: 7
 #       send_every: 4
 #       send_first_at: 1
 #   - calibrate_linear:
 #       - 0.85 -> 0.00
 #       - 0.44 -> 100.00
 #   - lambda: if (x < 1) return 0; else if (x > 100) return 100; else return (x);
    accuracy_decimals: 2


  - platform: wifi_signal
    name: "${devicename} WiFi signal"

####################################################################################################################
#                                                                                                                  #
#                           Configuration de 5 relais correspondants aux 5 vannes d'arrosage                       #
#                                                                                                                  #
#                                                 # # S W I T C H # #                                              #
#                                                                                                                  #
####################################################################################################################

switch:
  - platform: gpio
    name: "${switch1_name}"
    id : relay1
    icon: "mdi:${switch_icon}"
    pin: ${pin_switch1}
    on_turn_on:
    - logger.log: "L'arrosage de la zone ${switch1_name} est en cours."
    on_turn_off:
    - logger.log: "L'arrosage de la zone ${switch1_name} est terminé."
    inverted: ${is_inverted}
  - platform: gpio
    name: "${switch2_name}"
    id : relay2
    icon: "mdi:${switch_icon}"
    pin: ${pin_switch2}
    on_turn_on:
    - logger.log: "L'arrosage de la zone ${switch2_name} est en cours."
    on_turn_off:
    - logger.log: "L'arrosage de la zone ${switch2_name} est terminé."
    inverted: ${is_inverted}
  - platform: gpio
    name: "${switch3_name}"
    id : relay3
    icon: "mdi:${switch_icon}"
    pin: ${pin_switch3}
    on_turn_on:
    - logger.log: "L'arrosage de la zone ${switch3_name} est en cours."
    on_turn_off:
    - logger.log: "L'arrosage de la zone ${switch3_name} est terminé."
    inverted: ${is_inverted}
  - platform: gpio
    name: "${switch4_name}"
    id : relay4
    icon: "mdi:${switch_icon}"
    pin: ${pin_switch4}
    on_turn_on:
    - logger.log: "L'arrosage de la zone ${switch4_name} est en cours."
    on_turn_off:
    - logger.log: "L'arrosage de la zone ${switch4_name} est terminé."
    inverted: ${is_inverted}
  - platform: gpio
    name: "${switch5_name}"
    id : relay5
    icon: "mdi:${switch_icon}"
    pin: ${pin_switch5}
    on_turn_on:
    - logger.log: "L'arrosage de la zone ${switch5_name} est en cours."
    on_turn_off:
    - logger.log: "L'arrosage de la zone ${switch5_name} est terminé."
    inverted: ${is_inverted}
  - platform: restart
    name: "🛑Reboot ${devicename}🛑"
    id: reboot
    icon: "mdi:${switch_reboot}"
####################################################################################################################
#                                                                                                                  #
# Ici on fait deux switch virtuels, pour lancer l'un ou l'autre des deux scipts                                    #
#                                                                                                                  #
####################################################################################################################

  - platform: template
    name: "🕐Lancer ${Temps1} mn d'arrosage"
    icon: "mdi:${switch_remote}"
    turn_on_action:
        - script.execute: script_${Temps1}

#    turn_off_action:
  - platform: template
    name: "🕑Lancer ${Temps2} mn d'arrosage"
    icon: "mdi:${switch_remote}"
    turn_on_action:
        - script.execute: script_${Temps2}
  - platform: template
    name: "🕑Lancer l'arrosage manuel"
    icon: "mdi:${switch_remote}"
    turn_on_action:
        - script.execute: tempo_manuelle

####################################################################################################################
#                                                                                                                  #
#                                                                                                                  #
# Configuration de 2 boutons poussoirs, le premier servira à lancer un cycle d'arrosage de toutes les zones 10mn   #
# le second lancera un cycle de 10mn pour la pellouse et 20 minutes pour les plantes                               #
#                                                                                                                  #
#                                                                                                                  #
####################################################################################################################

binary_sensor:
  - platform: gpio
    device_class: motion
    icon: "mdi:${switch_remote}"
    pin:
      number: ${pin_bouton1}
      mode: INPUT_PULLUP
      inverted: ${button_inverted}
    name: "${bouton1_name}"
    id: pb1
    on_press:
      then:
        - script.execute: script_${Temps1}

  - platform: gpio
    device_class: motion
    icon: "mdi:${switch_remote}"
    pin:
      number: ${pin_bouton2}
      mode: INPUT_PULLUP
      inverted: ${button_inverted}
    name: "${bouton2_name}"
    id: pbd
    on_press:
      then:
        - script.execute: script_${Temps2}

####################################################################################################################
#                                                                                                                  #
#                                                                                                                  #
# Ancienne configuration, via button, elle a été remplacée par les deux switchs virtuels                           #
#                                                                                                                  #
#                                                                                                                  #
####################################################################################################################

# button:
#  - platform: template
#    name: Programme de ${Temps1}
#    id: Bouton5
#    icon: "mdi:${switch_remote}"
#    on_press:
#    - script.execute: script_${Temps1}

#  - platform: template
#    name: Programme de ${Temps2}
#    id: Bouton10
#    icon: "mdi:${switch_remote}"
#    on_press:
#    - script.execute: script_${Temps2}

####################################################################################################################
#                                                                                                                  #
#                                                                                                                  #
# Voici les scripts qui seront lancés en fonction du switch de commande qui sera activé par home assistant         #
# ou par le fait de presse un des deux boutons poussoir (Le 3eme etant directement relié au GPIO RST)              #
#                                                                                                                  #
#                                                                                                                  #
####################################################################################################################

script:
  - id: script_${Temps1}
    mode: restart
    then:
        - logger.log: "Programme de ${Temps1} minutes"
####################################################################################################################
# Sscript qui suit la temporisation N1                                                                                                                 #
# On part d'un état ou toutes les vannes sont éteintes
# puis 3 actions sont enchainées, pour chacunes des vannes:
# on allume la vanne de la premiere zone,
# on écrit dans le log que l arrosage de la zone 1 a déuté,
# on patiente un délai égale à Temps1 (ici 5mn)
# on ferme la vanne de la zone 1
# on ouvre la vanne de la zone 2 et ainsi de suite                                                                                                                #
####################################################################################################################
        - switch.turn_off: relay1
        - switch.turn_off: relay2 
        - switch.turn_off: relay3   
        - switch.turn_off: relay4
        - switch.turn_off: relay5
        - switch.turn_on: relay1
        - delay: ${Temps1}min
        - logger.log: "Arrosage ${switch1_name} terminé "
        - switch.turn_off: relay1    
        - switch.turn_on: relay2
        - delay: ${Temps1}min
        - logger.log: "Arrosage ${switch2_name} terminé "
        - switch.turn_off: relay2    
        - switch.turn_on: relay3
        - delay: ${Temps1}min
        - logger.log: "Arrosage ${switch3_name} terminé "
        - switch.turn_off: relay3    
        - switch.turn_on: relay4
        - delay: ${Temps1}min
        - logger.log: "Arrosage ${switch4_name} terminé "
        - switch.turn_off: relay4    
        - switch.turn_on: relay5
        - delay: ${Temps1}min
        - logger.log: "Arrosage ${switch5_name} terminé "
        - switch.turn_off: relay5    
# idem, on en profite pour vérifier si tous les autres relais sont bien eteints
        - switch.turn_off: relay1
        - switch.turn_off: relay2 
        - switch.turn_off: relay3   
        - switch.turn_off: relay4
        - switch.turn_off: relay5

  - id: script_${Temps2}
    mode: restart
    then:
        - logger.log: "Programme de ${Temps2} minutes"
# idem que pour le premier scipt, avec un délai égale au temps 2
        - switch.turn_off: relay1
        - switch.turn_off: relay2 
        - switch.turn_off: relay3   
        - switch.turn_off: relay4
        - switch.turn_off: relay5
        - switch.turn_on: relay1
        - delay: ${Temps2}min
        - logger.log: "Arrosage ${switch1_name} terminé "
        - switch.turn_off: relay1    
        - switch.turn_on: relay2
        - delay: ${Temps2}min
        - logger.log: "Arrosage ${switch2_name} terminé "
        - switch.turn_off: relay2    
        - switch.turn_on: relay3
        - delay: ${Temps2}min
        - logger.log: "Arrosage ${switch3_name} terminé "
        - switch.turn_off: relay3    
        - switch.turn_on: relay4
        - delay: ${Temps2}min
        - logger.log: "Arrosage ${switch4_name} terminé "
        - switch.turn_off: relay4    
        - switch.turn_on: relay5
        - delay: ${Temps2}min
        - logger.log: "Arrosage ${switch5_name} terminé "
        - switch.turn_off: relay5    
# idem, on en profite pour vérifier si tous les autres relais sont bien eteints
        - switch.turn_off: relay1
        - switch.turn_off: relay2 
        - switch.turn_off: relay3   
        - switch.turn_off: relay4
        - switch.turn_off: relay5

  - id: tempo_manuelle
    mode: restart
    then:
        - logger.log: "Programme manuel lancé"
####################################################################################################################
# Sscript qui suit la temporisation N1                                                                                                                 #
# On part d'un état ou toutes les vannes sont éteintes
# puis 3 actions sont enchainées, pour chacunes des vannes:
# on allume la vanne de la premiere zone,
# on écrit dans le log que l arrosage de la zone 1 a déuté,
# on patiente un délai égale à Temps1 (ici 5mn)
# on ferme la vanne de la zone 1
# on ouvre la vanne de la zone 2 et ainsi de suite                                                                                                                #
####################################################################################################################
        - switch.turn_off: relay5
        - switch.turn_off: relay4 
        - switch.turn_off: relay3   
        - switch.turn_off: relay2
        - switch.turn_off: relay1
# l'odre des déclenchements des relai peut changer 

        - switch.turn_on: relay2
        - logger.log: id(temps2) minutes d'arrosage pour ${switch2_name}
        - delay: !lambda 'return id(temps2) * 60000;'  # Convertit la valeur en minutes
        - logger.log: "Arrosage ${switch2_name} terminé "
        - switch.turn_off: relay2    
        
        - switch.turn_on: relay3
        - logger.log: id(temps3) minutes d'arrosage pour ${switch3_name}
        - delay: !lambda 'return id(temps3) * 60000;'  # Convertit la valeur en minutes
        - logger.log: "Arrosage ${switch3_name} terminé "
        - switch.turn_off: relay3    
        
        - switch.turn_on: relay5
        - logger.log: id(temps5) minutes d'arrosage pour ${switch5_name}
        - delay: !lambda 'return id(temps5) * 60000;'  # Convertit la valeur en minutes
        - logger.log: "Arrosage ${switch5_name} terminé "
        - switch.turn_off: relay5

        - switch.turn_on: relay4
        - logger.log: id(temps4) minutes d'arrosage pour ${switch4_name}
        - delay: !lambda 'return id(temps4) * 60000;'  # Convertit la valeur en minutes
        - logger.log: "Arrosage ${switch4_name} terminé "
        - switch.turn_off: relay4    
       
        - switch.turn_on: relay1
        - logger.log: id(temps1) minutes d'arrosage pour ${switch1_name}
        - delay: !lambda 'return id(temps1) * 60000;'  # Convertit la valeur en minutes
        - logger.log: "Arrosage ${switch1_name} terminé "
        - switch.turn_off: relay1    
        
# idem, on en profite pour vérifier si tous les autres relais sont bien eteints
        - switch.turn_off: relay5
        - switch.turn_off: relay4 
        - switch.turn_off: relay3   
        - switch.turn_off: relay2
        - switch.turn_off: relay1

        # Enjoy #
        # 
        # La prochaine évolution, variabiliser le temps pour n'avoir qu un seul script
        # 
        # si vous souhaitez faire évoluer ce code et partager vos amélioration, je suis preneur ;-)
        #  socratemobile@protonmail.com
        #
```

Il ne reste plus qu'à créer les 3 cartes pour gérer l affichage dans Home Assistant :

![alt text](https://github.com/SocrateMobile/Arrosage-Automatique-HomeAssistant/blob/main/3cartes.jpeg)

Le code des trois cartes:

carte 1:

```
type: picture-elements
image: \local\plans\Perso\Arrosage.png
elements:
  - type: image
    entity: switch.gpio_arrosage_v2_lancer_5_mn_d_arrosage
    tap_action:
      action: toggle
    hold_action:
      action: 'on'
    image: \local\plans\Perso\5MN.png
    state_filter:
      'on': brightness(130%) saturate(1.5) drop-shadow(0px 0px 10px gold)
      'off': brightness(80%) saturate(0.8)
    style:
      top: 60%
      left: 16%
      width: 30%
      padding: 30%
  - type: image
    entity: switch.gpio_arrosage_v2_lancer_10_mn_d_arrosage
    tap_action:
      action: toggle
    hold_action:
      action: more-info
    image: \local\plans\Perso\10MN.png
    state_filter:
      'on': brightness(130%) saturate(1.5) drop-shadow(0px 0px 10px gold)
      'off': brightness(80%) saturate(0.8)
    style:
      top: 60%
      left: 85%
      width: 30%
  - type: image
    entity: switch.gpio_arrosage_v2_reboot_gpio_arrosage_v2
    tap_action:
      action: toggle
    hold_action:
      action: more-info
    image: \local\plans\Perso\stop.png
    state_filter:
      'on': brightness(130%) saturate(1.5) drop-shadow(0px 0px 10px gold)
      'off': brightness(80%) saturate(0.8)
    style:
      top: 87%
      left: 50%
      width: 12%
      padding: 30%
```

Le code de la carte 2

```
type: entities
entities:
  - entity: switch.gpio_arrosage_v2_1bambous
    name: 1 Bambous
  - entity: switch.gpio_arrosage_v2_2jardin_cote_droit
    name: 2 Jardin côté droit
  - entity: switch.gpio_arrosage_v2_3jardin_cote_gauche
    name: 3 Jardin côté gauche
  - entity: switch.gpio_arrosage_v2_4bordures
    name: 4 Bordures
  - entity: switch.gpio_arrosage_v2_5_potager
    name: 5 Potager
  - entity: switch.gpio_arrosage_v2_lancer_10_mn_d_arrosage
    name: Lancer 10 mn d'arrosage
  - entity: switch.gpio_arrosage_v2_lancer_5_mn_d_arrosage
    name: Lancer 5 mn d'arrosage
  - entity: switch.gpio_arrosage_v2_lancer_l_arrosage_manuel
title: GPIO Arrosage V2
header:
  type: picture
  image: \local\plans\Perso\arrosage_bandeau2.png
  tap_action:
    action: none
  hold_action:
    action: none

```

et enfin, le code de la carte 3

```
type: vertical-stack
cards:
  - type: entities
    entities:
      - entity: input_number.tempo_temps1
        name: Bambous
        icon: mdi:timer-sand-complete
      - entity: input_number.tempo_temps2
        name: Côté droit
        icon: mdi:timer-sand-complete
      - entity: input_number.tempo_temps3
        name: Côté Gauche
        icon: mdi:timer-sand-complete
      - entity: input_number.tempo_temps4
        name: Bordures
        icon: mdi:timer-sand-complete
      - entity: input_number.tempo_temps5
        name: Potager
        icon: mdi:timer-sand-complete
      - entity: switch.gpio_arrosage_v2_lancer_l_arrosage_manuel
        name: Lancer l'arrosage manuel
    header:
      type: picture
      image: \local\plans\Perso\arrosage_bandeau2.png
      tap_action:
        action: none
      hold_action:
        action: none

```
