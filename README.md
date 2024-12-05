# Short King
#### Eftir Ingimar, Marijonas og Valdimar  

## Stutt lýsing
Við hönnuðum vélmenni (animatronic) sem athugar ef eitthvað er nálægt or reynir svo að hræða hlutinn/manneskju sem er fyrir framan það. Það hefur kistu sem er hægt að opna með notkun NodeRed dashboard, ef eitthver opnar hana, þá munn athyglið vélmennið vera á það

## Sensor ESP  
Notar 3 Ultrasonic Sensor hc-sr04 til að sjá ef það er eitthvað nálægt vélmennið, sendir json skrá til vélmenna espið sem munn gera hluti (fer eftir hvað sensors eru að sjá), hefur líka Kistu servo sem opnar og lokar Kistu (fær skilaboð frá NodeRed hvort það á að opna eða loka)

<details>
<summary>Kóði</summary>

```python
#impostra hluti sem kóðinn er að nota
from hcsr04 import HCSR04
from time import sleep_ms, ticks_ms, sleep
import json
from binascii import hexlify
from umqtt.simple import MQTTClient
from machine import unique_id, Pin
from servo import Servo
#---------------------------------VARIABLES--------------------------
#pinnar fyrir sensors
sensor_mid = HCSR04(trigger_pin=17, echo_pin=18, echo_timeout_us=10000)
sensor_left = HCSR04(trigger_pin=36, echo_pin=35, echo_timeout_us=10000)
sensor_right = HCSR04(trigger_pin=15, echo_pin=16, echo_timeout_us=10000)

#breytta sem munn segja hinn esp ef það sér eitthvað
in_vision = False

#segir hvort vélmennið á að hræða eitthvað
scare = False

#kista breytta sem fær upplýsingar frá NodeRed 
kista = False

#þegar kista er kveikt þá á hún að lokast og opnast í smá tíma
kista_stada = False
min_kista = 0
max_kista = 150

#pin á kista
servo_pin = Pin(10) 
kista_servo = Servo(servo_pin)
#----------------------------------MQTT KODI OG NET--------------------------------------

WIFI_SSID = "TskoliVESM"
WIFI_LYKILORD = "Fallegurhestur"

def do_connect():
    import network
    wlan = network.WLAN(network.STA_IF)
    wlan.active(True)
    if not wlan.isconnected():
        print('connecting to network...')
        wlan.connect(WIFI_SSID, WIFI_LYKILORD)
        while not wlan.isconnected():
            pass
    print('network config:', wlan.ifconfig())
    
do_connect()


#fall til að kíka ef esp fékk skilaboð frá NodeRed, eina sem það fær er kista
def fekk_skilabod(topic, skilabod):
    global kista
    kista = skilabod.decode()
    
#allt tengd MQTT
MQTT_BROKER = "broker.emqx.io"
CLIENT_ID = hexlify(unique_id())
TOPIC = b"SensorESP"
mqtt_client = MQTTClient(CLIENT_ID, MQTT_BROKER, keepalive=60)
mqtt_client.connect()
mqtt_client.set_callback(fekk_skilabod)
mqtt_client.subscribe("kista-on")
#------------------------------------- FUNCTIONS --------------------------------------
#Double check hversu langt eitthvað er, ef yfir 2m eða undefined, þá er gefið -1 sem er basically undefined
def of_langt(distance):
    if distance <= 0 or distance > 230:
        distance = -1
    return distance

#Fall fyrir allt, kíkir á sensors, breyttir stöðu kistu og fleira
def main(sensor_left, sensor_mid, sensor_right):
    #
    global in_vision, scare, kista_stada
    
    #breyttur sem munn ég nota seinna
    which_sensor = ""
    #Kíkir á hvað er í gangi með sensors
    distance_mid = of_langt(sensor_mid.distance_cm())
    distance_right = of_langt(sensor_right.distance_cm())
    distance_left = of_langt(sensor_left.distance_cm())
    
    #ef það sér eitthvað
    if any(0 < dist <= 100 for dist in [distance_left, distance_mid, distance_right]):
        in_vision = True
    else:
        in_vision = False
    #Ef það sér og eitthvað kemur of nálægt
    if any(0 < dist <= 40 for dist in [distance_left, distance_mid, distance_right]) and in_vision:
        scare = True
    #annars er enginn nóg nálægt til að hræða
    else:
        scare = False
    
        
    #setjir in í string hvað sensor er að sjá
    if 0 < distance_left <= 100:
        which_sensor += "L"
    if 0 < distance_mid <= 100:
        which_sensor += "M"
    if 0 < distance_right <= 100:
        which_sensor += "R"
        
    #kíkir ef kistan á að opnast
    if kista == 'true':
        
        #hvort kista á að opnast eða lokast
        kista_stada = not kista_stada
        
        #opnar kistuna
        if kista_stada:
            for i in range(min_kista,max_kista):
                kista_servo.write_angle(i)
                sleep(0.003)
                
        #lokar kistuna
        else:
            for i in range(max_kista, min_kista, -1):
                kista_servo.write_angle(i)
                sleep(0.003)
    
    #annars er hún lokuð
    else :
        kista_servo.write_angle(min_kista)
      
    #skilur til baka í dict það sem vélmenni á að sér og hvað á að gera
    return {"sensor":which_sensor,
            "scare":scare,
            "in_vision": in_vision,
            "kista": kista}

#fall til að senda skilaboð
def senda_mqtt_skilabod(mqtt_client_inn, topic, skilabod):
    mqtt_client_inn.publish(topic, skilabod)
    
#loop til að esp virkar endalaust      
while True:
    try:
        #skilaboð sem esp munn senda til hitt
        message = {}
    
        #kíkir á skilaboð
        mqtt_client.check_msg()
        
        #notar funci
        message = main(sensor_left, sensor_mid, sensor_right)
        senda_mqtt_skilabod(mqtt_client, TOPIC, json.dumps(message).encode())
    except:
        pass
    sleep_ms(100)
```
</details>   

## Íhlutalisti
| Hlutur   | Magn    |
| -------- | ------- |
| Ultrasonic Sensor hc-sr04  | 3   |
| Servo | 5     |
| Micro Servo | 1     |
| ESP32    |  2   |
| Hátalari    |  1   |
| DFplayer mini    |  1  |

## Tengingar  
![image](https://github.com/user-attachments/assets/2dc23e35-67e7-4503-8b82-5891b37d700f)  

## Hvering Sena virkar
![image](https://github.com/user-attachments/assets/8d4094f1-b330-4f18-a541-a8c080ad5981)


## Blender Kóróna
![Skjámynd 2024-12-05 093530](https://github.com/user-attachments/assets/9e52fd28-ce83-4dc6-b507-ccb2b9c4ee9f)

## Blender Kista
![Skjámynd 2024-12-05 093518](https://github.com/user-attachments/assets/4793fd09-0602-4759-8b19-40ed2ef38aef)

## Dashboard
![Skjámynd 2024-12-05 112553](https://github.com/user-attachments/assets/76d98e31-c5fa-46ce-989c-139cb3687994)
