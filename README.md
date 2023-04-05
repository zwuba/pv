# Tom's PV Dashboard
Scripts und Configs um ein PV Dashboard wie dieses zu machen:
![Dashboard Januar 2023](https://github.com/thomhug/pv/blob/main/pv%20dashboard%202023-01-13.PNG)
Neue Version mit Batterieanzeige:
![Dashboard Februar 2023](https://github.com/thomhug/pv/blob/main/pv%20dashboard%202023-02-21.jpg)

Grobe Funktionsweise:

- Auf einem RaspberryPi im LAN/Wifi mit einem Fronius Wechselrichter, Shelly 3EM's, myStrom Steckdosen und anderen Geräten läuft ein json_exporter (https://github.com/prometheus-community/json_exporter). Die Config vom json_exporter beschreibt, was wo ist in den jeweiligen APIs von den Geräten.
- Mittels einem SSH Tunnel expose ich den Port vom json_exporter (default 7979) auf einem Jumphost (normaler Cloudserver)
- Auf meinem Prometheus/Grafana Server läuft ebenfalls ein SSH Tunnel auf den Jumphost um den Port lokal abrufbar machen zu können (so braucht weder jemand von aussen Zugriff auf den RaspberryPi via SSH, noch braucht der Raspi Zugriff auf den Prometheus/Grafana Server
- Ein Prometheus holt die Metriken durch die Tunnels vom json_exporter ab und speichert sie lokal in seiner TimeSeriesDB
- Grafana stellt alles dar via Prometheus Datasource
- Um den Sonnenstand darzustellen verwende ich das Sun and Moon Plugin für Grafana (https://grafana.com/grafana/plugins/fetzerch-sunandmoon-datasource/)

Konzept Metriken- & Labelnamen

- In/Out aus Sicht Haus: 
  - Grid in = Bezug (>0), grid out = Einspeisung (<0)
  - PV in = Produktion (>0) 
  - Batt in = Laden (>0)
   
```
    sensor:
      location: <standort>
      type: <temp, hum, percent, status>
      position: <wohnzimmer, boiler, outdoor>
      kind: <indoor, outdoor, boiler, battery, heizung>   <- optional
    meter:
      location: <standort>
      position: <generator, grid, battery, device, genstring> 
      type: <counter, gauge>
      direction: <in, out>   <- nur für counter, gauge wird negativ, bei PV leer
      description: <optional ... z.B. Wärmepumpe, WR Name>
    tarif:
      location: <standort>
      type: <bezug, einspeisung>
```

Nächste Schritte

- Solarprognose von https://twitter.com/_solarmanager speichern und als ist-soll Vergleich visualisieren.
- Langzeitspeicher 'Problem' lösen, indem Tagessummen in einer SQL DB gespeichert werden (gibt es bessere Ideen?)

Fragen & Feedback: https://twitter.com/tomdawon


Client Setup How-2
==================

On your Fronius Inverter
------------------------
- Make sure the inverter has a fixed IP-Adress 
- Log into your inverters webInterface and goto "Communication / ModBus"
  - activate "Slave as Modbus TCP"
  - set "SunSpec Model Type" to float



On your "client system" (eg Raspberry Pi)
-----------------------------------------
Install Python:
$ sudo apt install python3

Install required packages:
$ pip3 install pymodbus pyModbusTCP

Create a SSH key (NO passphrase!) (this example is based on the file-name 'mysshkey'): 
$ ssh-keygen -t rsa

Check if remote connection is working: 
$ ssh <username>@cloud-tom-07.nine.ch

Test if there's a nice answer from the inverter by doing (remember to put in your inverter IP):
$ curl -s "http://192.168.1.30/solar_api/v1/GetPowerFlowRealtimeData.fcgi"
/solar_api/v1/GetMeterRealtimeData.cgi
/solar_api/v1/GetStorageRealtimeData.cgi
/solar_api/v1/GetInverterRealtimeData.cgi?DataCollection=CommonInverterData&Scope=Device

Create python-script file that is reading your inverter data 
$ TODO

Test script:
$ python3 testScript.py

Output should look like:
DCW: 1
cham
gen24
0  --> this will be your actual PV power but at night it will be zero

Install webserver: 
$ sudo apt-get lighttpd

Create file /etc/lighttpd/conf-enabled/11-python.conf
***************
server.modules += ( "mod_cgi" )
cgi.assign = ( ".py" => "/usr/bin/python3")
***************

Perform:
$ lighttpd reload

Copy testScript.py to webroot

Get json_exporter from github:
$ git clone http://github.com/prometheus-community/json_exporter

replace /json_exporter/config.yml from TomHug-git-repo
$ git clone https://github.com/tomhug/pv

Install Go: 
$ sudo apt-get install golang

then compile json_exporter: 
/json_exporter$ sudo make build 


Create file: /etc/lighttpd/conf-enabled/11-python.conf
$ TODO

Create file: /etc/systemd/system/json_exporter.service
$ TODO

Create file: /etc/systemd/system/http_forward.service
(Remember to put the correct SSH-Key in here)
$ TODO

Test if json_exporter is running:
$ curl localhost:7979 (is expected to return a 404 page not found)

$ curl "localhost:7979/probe?module=gen24&target=http://192.168.1.30/solar_api/v1/GetPowerFlowRealtimeData.fcgi"