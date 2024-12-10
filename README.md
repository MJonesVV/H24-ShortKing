# The Short King Lebron
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
        kista_hinn = True
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
        kista_hinn = False
        kista_servo.write_angle(min_kista)
      
      
    #ef kistan er kveikt, allt annað er slökkt
    if kista_hinn:
        which_sensor = False
        in_vision = False
        scare = False

    #skilur til baka í dict það sem vélmenni á að sér og hvað á að gera
    return {"sensor":which_sensor,
            "scare":scare,
            "in_vision": in_vision,
            "kista": kista_hinn}

#fall til að senda skilaboð
def senda_mqtt_skilabod(mqtt_client_inn, topic, skilabod):
    mqtt_client_inn.publish(topic, skilabod)
    
#loop til að esp virkar endalaust      
while True:
    #try og except til þess að kóðinn slökkvar aldrei
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
    sleep_ms(200)
```
</details>   

## Vélmenna ESP  
Notar 4 mótara og 1 micro mótor. 2 led peru augu. Kóðin tekur við skilaboð frá Sensor esp til þess að geta horft í rétta átt að öðrum sem koma of nálægt. Augun taka við skilaboðum frá Node-Red til þess að vita hvaða lit augun eiga að vera. Þegar Node-Red sendir skilaboð til Sensor ESP er líka sent til Vélmenna ESP til þess að segja vékmenninu að kistan sé opin, þegar kistan er opin á vélmennið að horfa á það með glóandi augu þar til notandin slekkur á kistunni. Í myndbandinu er ekki sínt þegar Node-Red breytir lit á augunum en það virkar.

<details>
<summary>Kóði</summary>

```python
import uasyncio as asyncio  
import json  
from machine import Pin, unique_id, PWM  
from binascii import hexlify  
from umqtt.simple import MQTTClient  
from servo import Servo  
import machine  
import random
from lib.dfplayer import DFPlayer

# ------------ Tengjast WIFI -------------
WIFI_SSID = "TskoliVESM"  #Nafn Wi-Fi netsins
WIFI_LYKILORD = "Fallegurhestur"  #Lykilorð fyrir Wi-Fi netið

def do_connect():
    import network  
    wlan = network.WLAN(network.STA_IF)  
    wlan.active(True)  
    if not wlan.isconnected():  
        print('Tengist neti...')  
        wlan.connect(WIFI_SSID, WIFI_LYKILORD)  
        while not wlan.isconnected():  
            pass
    print('Netstillingar:', wlan.ifconfig())  

do_connect()

# ------------ MQTT ---------------------
MQTT_BROKER = "broker.emqx.io"  #Nafn eða IP-tala MQTT miðlara
CLIENT_ID = hexlify(unique_id())  #Býr til einkvæmt auðkenni fyrir MQTT-klienta
TOPIC = b"SensorESP"  #Skilgreinir umræðuefni fyrir MQTT samskipti
TOPIC_LITUR = b"eye-color"
TOPIC_LITUR_SPEED = b"blink_speed"
TOPIC_ON = b"2806/takki"

# ---------------- Servo-stillingar ----------------
servo_pin_H = machine.Pin(38)  #Hönd - lárétt
servo_pin_V = machine.Pin(39)  #Hönd - lóðrétt
servo_pin_N_U_N = machine.Pin(48)  #Höfuð - upp og niður
servo_pin_N_L_R = machine.Pin(45)  #Höfuð - vinstri og hægri
servo_pin_K = machine.Pin(47)  #Munnur

#Býr til Servo-hluti fyrir hvern pinna
my_servo_H = Servo(servo_pin_H)
my_servo_V = Servo(servo_pin_V)
my_servo_N_U_N = Servo(servo_pin_N_U_N)
my_servo_N_L_R = Servo(servo_pin_N_L_R)
my_servo_K = Servo(servo_pin_K)

#Skilgreinir lágmarks- og hámarksgildi fyrir servó-mótora
minH = 12  #Hönd niður
maxH = 25  #Hönd upp
minV = 110  #Hönd upp
maxV = 180  #Hönd niður
minN_L_R = 5  #Höfuð til hægri
middle_N_L_R = 63  #Höfuð í miðju
maxN_L_R = 120  #Höfuð til vinstri
minN_U_N = 93  #Höfuð niður
maxN_U_N = 140  #Höfuð upp
minK = 93  #Munnur lokaður
maxK = 140  #Munnur opinn

blar = 0
raudur = 0
graen = 0

raudur_pera = PWM(Pin(35, Pin.OUT), freq=500) #Rauður
graen_pera = PWM(Pin(36, Pin.OUT), freq=500) #Grænn
blar_pera = PWM(Pin(37, Pin.OUT), freq=500) #Blár

#Global breytur fyrir stjórnun
on = False
See = {} #dict til að geyma gögn um skynjara og aðgerðir
action_event = asyncio.Event()  #Atburður til að kalla á viðburði
action_in_progress = False  #Flaggað ef aðgerð er í gangi
current_task = None 

#Fall til að vinna með skilaboð frá MQTT miðlara
def fekk_skilabod(topic, skilabod):
    global See, blar, raudur, graen  #Gerir See dict aðgengilega
    print(f"UMRÆÐUEFNI: {topic.decode()}, skilaboð: {skilabod.decode()}")  #Prentar móttekin skilaboð
            
    if topic.decode() == TOPIC.decode():  #Athugar hvort skilaboðin eru á réttu umræðuefni
        try:
            See = json.loads(skilabod.decode())  #Umbreytir JSON skilaboðum í orðabók
            #print(f"Umbreytt See: {See}")  #Prentar umbreytt gögn
            action_event.set()  #Setur atburð í gang ef gögnin eru gild
        except json.JSONDecodeError as e:
            print(f"Villa við JSON umbreytingu: {e}")  #Prentar villu ef JSON umbreyting mistekst
    elif topic.decode() == TOPIC_LITUR.decode(): #Athugar hvort skilaboðin eru á réttu umræðuefni
        Litur = skilabod.decode() #Tekur skilaboðin og setur það í breytu
        blar = int(Litur[4:6], 16) * 4 #Segir hversu blár hann á að vera
        raudur = int(Litur[0:2], 16) * 4 #Segir hversu Rauður hann á að vera
        graen = int(Litur[2:4], 16) * 4 #Segir hversu Grænn hann á að vera
        
#Stillir MQTT klient og tengir við miðlara
mqtt_client = MQTTClient(CLIENT_ID, MQTT_BROKER, keepalive=60)
mqtt_client.set_callback(fekk_skilabod)  #Stillir tilvik til að vinna með móttekin skilaboð
mqtt_client.connect()  #Tengir við MQTT miðlara
mqtt_client.subscribe(TOPIC)  #Skráir sig á umræðuefnið
mqtt_client.subscribe(TOPIC_LITUR)
mqtt_client.subscribe(TOPIC_LITUR_SPEED)
mqtt_client.subscribe(TOPIC_ON)

# ---------------- Aðgerðir ----------------
#Fall til að framkvæma hreyfingu í "idle" ástandi
async def idle_action():
    raudur_pera.duty(raudur) #Breytir lit samkvæmt Node-Red
    graen_pera.duty(graen) #Breytir lit samkvæmt Node-Red
    blar_pera.duty(blar) #Breytir lit samkvæmt Node-Red
    print("Idle: Skannar herbergi.")
    for i in range(minN_L_R, maxN_L_R + 1):  #Hreyfir höfuð frá hægri til vinstri
        my_servo_N_L_R.write_angle(i)
        await asyncio.sleep(0.03)
    for i in range(maxN_L_R, minN_L_R - 1, -1):  #Hreyfir höfuð til baka
        my_servo_N_L_R.write_angle(i)
        await asyncio.sleep(0.03)

#Fall til að framkvæma lag/hljóð
async def lag():
    await df.wait_available()  # optional; making sure DFPlayer finished booting
    print("ok")
    await df.volume(20)
    await df.play(1, 1)  # folder 1, file 1
    await asyncio.sleep_ms(0)  # þarf ekki í þessu tilfelli en má vera

df = DFPlayer(1)  # using UART 
df.init(tx=16, rx=17)  # tx á esp tengist í rx á mp3

# Aðgerð fyrir "kista" (hreyfing)
async def kista_action():
    global action_in_progress
    print("Aðgerð: Kista virk.")
    action_in_progress = True  #Merkja aðgerð í gangi
    See["kista"] = True  #Stillir kista í dict
    
    raudur_pera.duty(0) #Breytir lit í Rauðan
    graen_pera.duty(1023) #Breytir lit í ekkert
    blar_pera.duty(1023) #Breytir lit í ekkert
    
    my_servo_N_U_N.write_angle(200)  #Hreyfir höfuð
    await asyncio.sleep(0.01)
    my_servo_N_L_R.write_angle(160)  #Snýr höfuði
    await asyncio.sleep(0.01)
    
    See["kista"] = False  #Stillir kista af
    action_in_progress = False  #Aðgerð lokið
    
#Aðgerð til að "hræða" (hreyfing)
async def scare_action():
    global action_in_progress
    raudur_pera.duty(raudur) #Breytir lit samkvæmt Node-Red
    graen_pera.duty(graen) #Breytir lit samkvæmt Node-Red
    blar_pera.duty(blar) #Breytir lit samkvæmt Node-Red
    print("Aðgerð: Hrædd viðbrögð virk.")  #Prentar aðgerðarstöðu
    action_in_progress = True  #Merkja aðgerð í gangi

    # Halla sér aftur á bak
    my_servo_N_U_N.write_angle(93)  #Hreyfir höfuð aftur
    await asyncio.sleep(0.01)
    my_servo_H.write_angle(25)  #Lyftir hendi
    await asyncio.sleep(0.01)
    my_servo_V.write_angle(110)  #Setur hendi í lóðrétta stöðu
    await asyncio.sleep(0.01)
    
    # Slembibið áður en "springur fram"
    delay = random.uniform(1, 5)  #Slembibið milli 1 og 5 sekúndur
    await asyncio.sleep(delay)

    # Hreyfist fram
    my_servo_N_U_N.write_angle(140)  #Hreyfir höfuð fram
    await asyncio.sleep(0.01)
    my_servo_K.write_angle(140)  #Opnar munn
    await asyncio.sleep(0.01)
    my_servo_H.write_angle(8)  #Hreyfir hönd
    await asyncio.sleep(0.01)
    my_servo_V.write_angle(180)  #Lætur hönd falla
    await asyncio.sleep(0.01)

    See["scare"] = False  #Stilling fyrir aðgerð lokið
    action_in_progress = False  #Aðgerð lokið

    await asyncio.sleep(3)  #Biður í 3 sekúndur
    my_servo_K.write_angle(93)  #Lokar munni
    await asyncio.sleep(0.01)

#Aðgerð sem framkvæmir hreyfingar miðað við skynjara
async def perform_action(sensor):
    global action_in_progress
    if action_in_progress:  #Athugar hvort önnur aðgerð sé í gangi
        print("Aðgerð er nú þegar í gangi, sleppi.")  #Prentar stöðu
        return
    if See.get("kista", False):  #Athugar hvort "kista" sé í gangi
        print("Kista í gangi, aðrar aðgerðir blokkaðar.")  #Prentar stöðu
        return

    print(See)  #Prentar See dict
    action_in_progress = True  #Merkja aðgerð í gangi
    print(f"Framkvæmi aðgerð fyrir skynjara: {sensor}")  #Prentar skynjara sem triggja aðgerð

    try:
        if See.get("scare", False):  #Athugar hvort scare-aðgerð ætti að keyra
            await scare_action()
        elif sensor == "L":  #Vinstri skynjari
            raudur_pera.duty(raudur) #Breytir lit í Rauðan
            graen_pera.duty(graen) #Breytir lit í ekkert
            blar_pera.duty(blar) #Breytir lit í ekkert
            print("Aðgerð: Vinstri skynjari greindur.")
            my_servo_N_L_R.write_angle(120)  #Snýr höfuði til vinstri
            await asyncio.sleep(0.05)
        elif sensor == "R":  # Hægri skynjari
            raudur_pera.duty(raudur) #Breytir lit í Rauðan
            graen_pera.duty(graen) #Breytir lit í ekkert
            blar_pera.duty(blar) #Breytir lit í ekkert
            print("Aðgerð: Hægri skynjari greindur.")
            my_servo_N_L_R.write_angle(5)  #Snýr höfuði til hægri
            await asyncio.sleep(0.05)
        elif sensor == "LR":  #Vinstri og hægri skynjarar
            raudur_pera.duty(raudur) #Breytir lit í Rauðan
            graen_pera.duty(graen) #Breytir lit í ekkert
            blar_pera.duty(blar) #Breytir lit í ekkert
            print("Aðgerð: Vinstri og hægri skynjarar greindir.")
            for i in range(minN_L_R, maxN_L_R + 1):  #Sveiflar höfuði
                my_servo_N_L_R.write_angle(i)
                await asyncio.sleep(0.01)
            for i in range(maxN_L_R, minN_L_R - 1, -1):
                my_servo_N_L_R.write_angle(i)
                await asyncio.sleep(0.01)
        elif sensor == "LM":  #Vinstri og miðju skynjarar
            raudur_pera.duty(raudur) #Breytir lit í Rauðan
            graen_pera.duty(graen) #Breytir lit í ekkert
            blar_pera.duty(blar) #Breytir lit í ekkert
            print("Aðgerð: Vinstri og miðju skynjarar greindir.")
            for i in range(middle_N_L_R, maxN_L_R):
                my_servo_N_L_R.write_angle(i)
                await asyncio.sleep(0.025)
            for i in range(maxN_L_R, middle_N_L_R, -1):
                my_servo_N_L_R.write_angle(i)
                await asyncio.sleep(0.02)
        elif sensor == "M":  #Miðju skynjari
            raudur_pera.duty(raudur) #Breytir lit í Rauðan
            graen_pera.duty(graen) #Breytir lit í ekkert
            blar_pera.duty(blar) #Breytir lit í ekkert
            print("Aðgerð: Miðju skynjari greindur.")
            my_servo_N_L_R.write_angle(63)  #Setur höfuð í miðju
            await asyncio.sleep(0.05)
        elif sensor == "MR":  #Miðju og hægri skynjarar
            raudur_pera.duty(raudur) #Breytir lit í Rauðan
            graen_pera.duty(graen) #Breytir lit í ekkert
            blar_pera.duty(blar) #Breytir lit í ekkert
            print("Aðgerð: Miðju og hægri skynjarar greindir.")
            for i in range(minN_L_R, middle_N_L_R):
                my_servo_N_L_R.write_angle(i)
                await asyncio.sleep(0.02)
        elif sensor == "LMR":  #Allir skynjarar
            raudur_pera.duty(raudur) #Breytir lit í Rauðan
            graen_pera.duty(graen) #Breytir lit í ekkert
            blar_pera.duty(blar) #Breytir lit í ekkert
            print("Aðgerð: Allir skynjarar greindir.")
            for i in range(minN_L_R, maxN_L_R + 1):
                my_servo_N_L_R.write_angle(i)
                await asyncio.sleep(0.02)
            for i in range(maxN_L_R, minN_L_R, -1):
                my_servo_N_L_R.write_angle(i)
                await asyncio.sleep(0.02)
        else:
            print(f"Engin aðgerð fyrir skynjara: {sensor}")  #Engin sérstök aðgerð
    finally:
        action_in_progress = False  #Aðgerð lokið

#Aðalfallið sem stjórnar hringrásinni
async def main_loop():
    global action_in_progress
    while True:
        action_event.set()  #Kveikir á atburði
        if action_event.is_set() and not action_in_progress:
            sensor = See.get("sensor", "")  # Nær í skynjara úr See
            #print(f"Greindi skynjara í aðalrás: {sensor}")  
            if See.get("kista", False):  #Athugar kista aðgerð
                await kista_action()
            elif See.get("scare", False):  #Athugar scare aðgerð
                await scare_action()
            elif sensor:  #Athugar aðrar aðgerðir fyrir skynjara
                await perform_action(sensor)
            action_event.clear()  # Núllstillar atburð
        elif not action_in_progress:
            await idle_action()  #Fer í idle ástand
        await asyncio.sleep(0.1)

#Fall sem stjórnar MQTT samskiptum
async def mqtt_handler():
    mqtt_client = MQTTClient(CLIENT_ID, MQTT_BROKER, keepalive=60)
    mqtt_client.set_callback(fekk_skilabod)
    mqtt_client.connect()
    mqtt_client.subscribe(TOPIC)
    mqtt_client.subscribe(TOPIC_LITUR)
    mqtt_client.subscribe(TOPIC_LITUR_SPEED)
    print("Tengdur við MQTT og áskrifandi að umræðuefnum.")
    while True:
        mqtt_client.check_msg()  #Athugar skilaboð frá MQTT miðlara
        await asyncio.sleep(0.1)

#Aðalforrit sem keyrir hringrás og MQTT
async def main():
    loop = asyncio.get_event_loop()  #Nær í atburðalykkju
    loop.create_task(mqtt_handler())  #Keyrir MQTT-handler sem bakgrunnsverkefni
    await main_loop()  #Keyrir aðalhringrás

try:
    asyncio.run(main())
except KeyboardInterrupt:
    print("Forritinu var hætt.")
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

## Myndbond
[![Virkni](https://img.youtube.com/vi/FsUnuj2XhRw/0.jpg)](https://www.youtube.com/watch?v=FsUnuj2XhRw)

[![Kista Virkni](https://img.youtube.com/vi/vf3eeeGuTSo/0.jpg)](https://www.youtube.com/watch?v=vf3eeeGuTSo)

## Tengingar  
![image](https://github.com/user-attachments/assets/2dc23e35-67e7-4503-8b82-5891b37d700f)  

## Hvering Sena virkar
![image](https://github.com/user-attachments/assets/8d4094f1-b330-4f18-a541-a8c080ad5981)


## Blender Kóróna
![Skjámynd 2024-12-05 093530](https://github.com/user-attachments/assets/9e52fd28-ce83-4dc6-b507-ccb2b9c4ee9f)

## Blender Kista
![Skjámynd 2024-12-05 093518](https://github.com/user-attachments/assets/4793fd09-0602-4759-8b19-40ed2ef38aef)

## Dashboard
![image](https://github.com/user-attachments/assets/1a527e1f-dce5-473a-aa72-67d31bbd85bc)
## Flow
![image](https://github.com/user-attachments/assets/6eb97e3a-bc80-4d8a-9db0-6e81b50ab248)


