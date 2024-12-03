## Minnispunktar eða Notur til að muna hluti sem við erum að gera

### PINNAR

##### Vélmenni esp
48 = Lóðrétt servo hreyfing  ()  
47 = Kjálki servo  (MIN=93, MAX=140)  
45 = Lárétt servo hreyfing  (MIN=0, MAX=185)  
40 = Hægri hönd servo  (MIN=12, MAX=25)  ***BREYTIST
39 = Vinstri hönd servo  (MIN=180, MAX=130)  
38 = Speaker    
[35, 36, 37] = Augu    

##### Sensor esp

Echo(35) Trigger(36) = Sensor left
Echo(18) Trigger(17) = Sensor middle
Echo(16) Trigger(15) = Sensor right
10 = Kista Servo 




### MQTT
Sensor esp sendir skilaboð til Vélmenni esp með MQTT, þessi skilaboð er sett í JSON fyrir sendingu  
Dæmi um hvernig skilaboðinn eru  
*{'scared': False, 'scare': True, 'in_vision': True, 'sensor': 'LMR'}*  
**Scared** lykill segir hvort vélmennið á að vera hrædd
**Scare** lykill segir hvort vélmennið á að hræða eitthvern
**In_vision** segir hvort sensor eru að detecta eitthvern
**Sensor** lykill segir hvaða sensors eru að detecta eitthvern
