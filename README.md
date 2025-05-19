# ewb Smartmeter Home Assistant

Dieses Repository beschreibt, wie ich einen ewb (Energie Wasser Bern) Smartmeter (Landis + Gyr E450) in Home Assistant integriert habe.

## Schnittstelle

Die Kundeninformationsschnittstelle des Smartmeters muss vorhergehend über den Kundendienst von ewb freigeschaltet werden.
Dabei sollte auch die Dokumentation der Schnittstelle mit den verwendeten OBIS-Codes mitgeliefert werden.
Bei anderen Energieversorgern, welche ebenfalls diesen Zähler verwenden, steht diese Dokumentation öffentlich zur Verfügung, und deckt ähnliche Informationen ab, bspw. [BKW](https://www.bkw.ch/fileadmin/user_upload/03_Energie/03_01_Stromversorgung_Privat-_und_Gewerbekunden/Zaehlerablesung/BKW_faktenblatt_kundenschnittstelle_L_G_E450_E570_def_Web.pdf) ([Archiv](https://web.archive.org/web/20250508173627/https://www.bkw.ch/fileadmin/user_upload/03_Energie/03_01_Stromversorgung_Privat-_und_Gewerbekunden/Zaehlerablesung/BKW_faktenblatt_kundenschnittstelle_L_G_E450_E570_def_Web.pdf)).
Bei der Schnittstelle handelt es sich um einen RJ12-Port, bei welchem die Drähte 3 und 4 für DLMS/COSEM über M-Bus verwendet werden.

Die OBIS-Codes sind weiter unten zusätzlich in einer Tabelle beschrieben.

In meinem Fall wurde die Schnittstelle ohne Verschlüsselung aktiviert.
Wahrscheinlich ist dies das Standardvorgehen, ansonsten kann dies ggf. erwünscht werden.

## Hardware

- Raspberry Pi 5
- USB to MBUS Slave Modul [AliExpress](https://www.aliexpress.com/item/1005005855976536.html)
- RJ12 Kabel, optimalerweise Breakout [AliExpress](https://www.aliexpress.com/item/1005007821256603.html)
- (optional) 3D-gedrucktes Gehäuse für das USB-MBUS Board, siehe Ordner `case` + USB-A-Verlängerungskabel

Dies ist nicht der einzige Weg, wie ein solcher Smartmeter augelesen werden kann.
Insbesondere ist die Auslesung auch mit einfacheren Geräten (bspw. ESP32) und ohne den USB-to-MBUS-Adapter möglich.
In diesem Fall wurde primär Wert auf die Verwaltbarkeit (Zugriff per SSH) und eine einfache Softwarelösung gesetzt - ein kompaktes System war zweitrangig.

## Software

Der M-Bus-zu-USB Adapter präsentiert sich als USB-Serial Gerät unter Linux (`/dev/ttyUSBx`).
Diese serielle Konsole wird mithilfe eines Python-Programms und der `gurux_dlms` Bibliothek eingelesen und schlussendlich über MQTT an Home Assistant übermittelt.
Der Code dafür befindet sich [hier](https://git.rack.farm/jmesserli/mbus-dlms-mqtt).

## Home Assistant Integration

Die per MQTT übermittelten Werte werden anschliessend über die folgenden Sensoren in Home Assistant integriert (`configuration.yaml`):

```yaml
mqtt:
  sensor:
    - name: "Energie Bezug T1"
      state_topic: "meter/energy_meter/1_8_1_bezug_t1"
      device_class: "energy"
      unit_of_measurement: "Wh"
      state_class: "total"
    - name: "Energie Bezug T2"
      state_topic: "meter/energy_meter/1_8_2_bezug_t2"
      device_class: "energy"
      unit_of_measurement: "Wh"
      state_class: "total"
    - name: "Energie Lieferung T1"
      state_topic: "meter/energy_meter/2_8_1_lieferung_t1"
      device_class: "energy"
      unit_of_measurement: "Wh"
      state_class: "total"
    - name: "Energie Lieferung T2"
      state_topic: "meter/energy_meter/2_8_2_lieferung_t2"
      device_class: "energy"
      unit_of_measurement: "Wh"
      state_class: "total"
```

Um die beiden Tarife zu kombinieren habe ich anschliessend noch zwei Template Sensoren definiert, welche beide Tarife aufsummieren.
Diese können unter `Geräte & Dienste > Tab Helfer > + Helfer erstellen` definiert werden.

Template:

```jinja2
{{ (states('sensor.energie_bezug_t1') | int + states('sensor.energie_bezug_t2') | int) / 1000 }}
```

(Dabei wird zudem von Wh zu kWh umgewanelt, da dies etwas üblicher ist).

Masseinheit: `kWh`

Zustandsklasse: `Gesamt`

Für die Lieferung analog, es sind einfach die Sensornamen anzupassen.

## OBIS-Codes

Die folgenden OBIS-Codes stammen aus der Dokumentation.
Ich habe teilweise abweichende OBIS-Codes gesehen, aber zur Orientierung reicht es.
Weiter waren die Werte, welche ich ausgelesen habe, nicht in den angegebenen Einheiten - möglicherweise würde dies über irgend eine Konfiguration, welche gepusht wird, übermittelt. Generell waren es anstelle von Kilowatt einfach Watt und anstelle von Kilowattstunden Wattstunden.

| Anzahl | Zeitintervall | Beschreibung                                                | OBIS-Code     | Einheit |
| ------ | ------------- | ----------------------------------------------------------- | ------------- | ------- |
| 1      | Std.          | Geräteidentifikation 1 (Herstellerseriennummer)             | 0-0:96.1.0;2  |         |
| 1      | Std.          | Uhr                                                         | 0-0:1.0.0;2   |         |
| 1      | Std.          | Konsumenteninformationstext                                 | 0-0:96.13.0;2 |         |
| 1      | Std.          | Konsumenteninformationscode                                 | 0-0:96.13.1;2 |         |
| 5      | sek           | Objektliste Push Einstellungen Verbraucherinformation 1     | 0-8:25.9.0;2  |         |
| 5      | sek           | OBIS Kennziffer Push Einstellungen Verbraucherinformation 1 | 0-8:25.9.0;1  |         |
| 5      | sek           | Geräteidentifikation 1 (Herstellerseriennummer)             | 0-0:96.1.0;2  |         |
| 5      | sek           | Wirkleistung Bezug +P                                       | 1-0:1.7.0;2   | kW      |
| 5      | sek           | Wirkleistung Bezug -P                                       | 1-0:2.7.0;2   | kW      |
| 5      | sek           | Wirkenergie Bezug +A (QI+QIV)                               | 1-1:1.8.0;2   | kWh     |
| 5      | sek           | Wirkenergie Bezug -A (QI+QIV)                               | 1-1:2.8.0;2   | kWh     |
| 5      | sek           | Blindenergie Ri+ Q1                                         | 1-1:5.8.0;2   | kvarh   |
| 5      | sek           | Blindenergie Rc+ Q2                                         | 1-1:6.8.0;2   | kvarh   |
| 5      | sek           | Blindenergie Ri- Q3                                         | 1-1:7.8.0;2   | kvarh   |
| 5      | sek           | Blindenergie Rc- Q4                                         | 1-1:8.8.0;2   | kvarh   |
| 5      | sek           | Blindleistung Q                                             | 1-0:130.7.0   | kvar    |
| 5      | sek           | Strom L1                                                    | 1-0:31.7.0;2  | Amperé  |
| 5      | sek           | Strom L2                                                    | 1-0:51.7.0;2  | Amperé  |
| 5      | sek           | Strom L3                                                    | 1-0:71.7.0;2  | Amperé  |
| 1      | Min           | Objektliste Push Einstellungen Verbraucherinformation 2     | 0-9:25.9.0;2  |         |
| 1      | Min           | OBIS Kennziffer Push Einstellungen Verbraucherinformation 2 | 0-9:25.9.0;1  |         |
| 1      | Min           | Wirkenergie Bezug +A (QI+QIV) Tarif1                        | 1-1:1.8.1;2   | kWh     |
| 1      | Min           | Wirkenergie Bezug +A (QI+QIV) Tarif2                        | 1-1:1.8.2;2   | kWh     |
| 1      | Min           | Wirkenergie Lieferung -A (QI+QIV) Tarif1                    | 1-1:2.8.1;2   | kWh     |
| 1      | Min           | Wirkenergie Lieferung -A (QI+QIV) Tarif2                    | 1-1:2.8.2;2   | kWh     |
| 15     | Min           | Objektliste Push Einstellungen Verbraucherinformation 4     | 0-11:25.9.0;2 |         |
| 15     | Min           | OBIS Kennziffer Push Einstellungen Verbraucherinformation 4 | 0-11:25.9.0;1 |         |
| 15     | Min           | M-Bus Geräte ID 1 Kanal 1                                   | 0-1:96.1.0;2  |         |
| 15     | Min           | M-Bus Geräte ID 1 Kanal 2                                   | 0-2:96.1.0;2  |         |
| 15     | Min           | M-Bus Geräte ID 1 Kanal 3                                   | 0-3:96.1.0;2  |         |
| 15     | Min           | M-Bus Wert 1 Kanal 1                                        | 0-1:24.2.1;2  | m3      |
| 15     | Min           | M-Bus Wert 1 Kanal 2                                        | 0-2:24.2.1;2  | m3/h    |
| 15     | Min           | M-Bus Wert 1 Kanal 3                                        | 0-3:24.2.1;2  | °C      |
| 15     | Min           | M-Bus Gerätetyp Kanal 1                                     | 0-1:24.1.0;9  |         |
| 15     | Min           | M-Bus Gerätetyp Kanal 2                                     | 0-2:24.1.0;9  |         |
| 15     | Min           | M-Bus Gerätetyp Kanal 3                                     | 0-3:24.1.0;9  |         |
| 15     | Min           | Einheit/Skalierung M-Bus Wert 1 Kanal 1                     | 0-1:24.2.1;3  |         |
| 15     | Min           | Einheit/Skalierung M-Bus Wert 1 Kanal 2                     | 0-2:24.2.1;3  |         |
| 15     | Min           | Einheit/Skalierung M-Bus Wert 1 Kanal 3                     | 0-3:24.2.1;3  |         |
