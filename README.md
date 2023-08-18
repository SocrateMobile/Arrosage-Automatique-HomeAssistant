# Arrosage-Automatique-HomeAssistant
Création d un système d arrosage automatique, piloté via Home Assistant
Après quelques années passées sous Jeedom, j’ai migré ma domotique sur HA en mars dernier.
Je vous partage mon projet actuel le plus abouti: l’arrosage automatique.
Le projet vise à remplacer mon système rainbird, dont le dongle wifi à rendu l’âme après deux saisons et demi …
En synthèse: utiliser un ESP8266 pour contrôler une plaque de 8 relais, 5 utilisés dans le montage sur les GPIO D0,D1,D2,D3,D4

capteurs de température Dallas DS18B20 (un intérieur au boîtier, l’autre pour la température extérieure), sur D7
1 capteur d’humidité du sol (yl38), sur A0
deux boutons poussoir pour lancer une automatisation :
5 minutes d’arrosage de toutes les zones sur D5
10 minutes d’arrosage de toutes les zones sur D6
Un troisième bouton sera cablé sur le GPIO RST & GND: un appuie, au reboot toutes les vannes seront éteintes

Et dans un second temps, la création de trois cartes dans HA pour piloter le tout:
