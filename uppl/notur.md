## Minnispunktar eða Notur til að muna hluti sem við erum að gera

### PINNAR

##### Vélmenni esp
48 = Lóðrétt servo hreyfing  
47 = Kjálki servo  
45 = Lárétt servo hreyfing  
40 = Hægri hönd servo  
39 = Vinstri hönd servo  
38 = Speaker  
[35, 36, 37] = Augu  

##### Sensor esp




### MQTT
>Sensor esp sendir skilaboð til Vélmenni esp með MQTT, þessi skilaboð er sett í JSON fyrir sendingu  
>Dæmi um hvernig skilaboðinn eru  
>*{'scared': False, 'scare': True, 'in_vision': True, 'sensor': 'LMR'}*  
>Scared lykill segir hvort vélmennið á að vera hrædd
>Scare lykill segir hvort vélmennið á að hræða eitthvern
>In_vision segir hvort sensor eru að detecta eitthvern
>Sensor lykill segir hvaða sensors eru að detecta eitthvern
