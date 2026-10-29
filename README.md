# GLW8 Heizungsregler für einen gemischten Heizkreis – Version 2 (ESP32)

**Firmware-Version 2.6.0** · Arduino IDE · ESP32-WROOM-32E 4-Kanal-Relaisplatine

© 2026 [erich.grassl@glw8.at](mailto:erich.grassl@glw8.at)

Witterungsgeführter Regler für einen gemischten Heizkreis. Aus der Außentemperatur wird über eine Heizkurve der Vorlauf-Sollwert berechnet. Ein PI-Regler verstellt das Mischerventil so, dass die Vorlauftemperatur diesem Sollwert folgt.

Bedient wird der Regler über eine integrierte Weboberfläche, über **MQTT** mit automatischer Einbindung in **Home Assistant** sowie über **Modbus TCP** und **Modbus RTU**. Updates gehen ohne Kabel über den Browser oder direkt aus der Arduino IDE.

> **Version 2** ist der Neubeginn auf dem **ESP32**. Version 1 (ESP8266, Wemos D1 mini, letzter Stand 1.35.0) wurde mit wachsendem Funktionsumfang instabil, weil der Arbeitsspeicher nicht mehr reichte. Version 2 übernimmt alle Funktionen von 1.35.0 und hat rund zehnmal so viel freien Speicher. Die Regelung läuft in einer **eigenen Aufgabe mit Watchdog**, unabhängig von Webserver, MQTT und Modbus (siehe [Aufbau des Quelltexts](#aufbau-des-quelltexts)). Einstellungen aus Version 1 lassen sich übernehmen (siehe [Einstellungen aus Version 1 übernehmen](#einstellungen-aus-version-1-übernehmen)).

> ⚠️ **Sicherheit:** Die Software ist **kein** Sicherheitstemperaturbegrenzer. Besonders bei Fußbodenheizungen muss ein Hardware-STB (Anlegethermostat, z. B. 50–55 °C) die Pumpe unabhängig vom Regler abschalten. Arbeiten an 230 V nur durch eine Elektrofachkraft.

## Inhalt

1. [Funktionen](#funktionen)
2. [Hardware und Verdrahtung](#hardware-und-verdrahtung)
3. [Inbetriebnahme](#inbetriebnahme)
4. [Weboberfläche](#weboberfläche)
5. [Betriebsarten](#betriebsarten)
6. [Regelung im Detail](#regelung-im-detail)
7. [Heizkurve](#heizkurve)
8. [Heizungsprofile](#heizungsprofile)
9. [Zeitprogramm](#zeitprogramm)
10. [Temperaturfühler](#temperaturfühler)
11. [Raumtemperatur](#raumtemperatur)
12. [Kühlen](#kühlen)
13. [Mischer und Regler](#mischer-und-regler)
14. [Relais K4 (Zusatzfunktion)](#relais-k4-zusatzfunktion)
15. [Schutzfunktionen und Störungen](#schutzfunktionen-und-störungen)
16. [Temperaturverlauf](#temperaturverlauf)
17. [MQTT](#mqtt)
18. [Home Assistant](#home-assistant)
19. [Modbus TCP und RTU](#modbus-tcp-und-rtu)
20. [Firmware-Update über WLAN (OTA)](#firmware-update-über-wlan-ota)
21. [REST-API](#rest-api)
22. [Einstellungen speichern und zurücksetzen](#einstellungen-speichern-und-zurücksetzen)
23. [Aufbau des Quelltexts](#aufbau-des-quelltexts)
24. [Modul GlwOta – Firmware-Update für andere Projekte](#modul-glwota--firmware-update-für-andere-projekte)
25. [Versionen](#versionen)

---

## Funktionen

**Regelung**
- Heizkurve aus der gedämpften Außentemperatur: wahlweise **Formel** mit Neigung und Niveau oder **4 frei einstellbare Stützpunkte** mit glattem, monotonem Spline; Vorlauf min/max; in der Grafik per Ziehen einstellbar; Export als CSV-Liste
- PI-Regler für **3-Punkt-Stellantriebe** (Auf/Zu-Impulse) oder **0–10-V-Antriebe** (PWM + Wandler)
- Mischerstellung aus der Motorlaufzeit geschätzt; nach jedem Start fährt der Mischer zur Kalibrierung ganz zu
- Zähler für Mischer-Stellvorgänge (gesamt, AUF, ZU), Pumpenlaufzeit und Pumpenstarts – seit Neustart und als Gesamtzähler über alle Neustarts (im Flash gespeichert)
- Neustart-Zähler mit Neustartgrund (Einschalten, Software, Absturz/Watchdog, Reset-Taste)
- Betriebsarten Aus, Automatik, **Raumtemperatur** (Vorlauf nur nach dem Raum über einen PI-Raumregler), Fester Sollwert, Handbetrieb, **Kühlen** und **Heizen/Kühlen automatisch**
- Kühlen über den Mischer (Kaltwasser von der Wärmepumpe): Kühlkurve, Raumaufschaltung, **Taupunktschutz** mit Raumfeuchte über MQTT oder Modbus
- Zwei Parameterprofile: Radiatorheizung (~50 °C) und Fußbodenheizung (~26–31 °C), dazu das eigene Profil **GLW8**, das die aktuellen Werte speichert und wieder lädt

**Komfort und Schutz**
- Absenkbetrieb nach Zeitprogramm: tägliches Fenster oder Wochenprogramm mit bis zu 3 Heizzeiten je Tag
- Raumaufschaltung und Raumabschaltung mit optionalem Raumfühler, schaltbar
- Rücklaufbegrenzung, Heizgrenze (Sommerbetrieb), Frostschutz, Übertemperaturschutz
- Pumpen-Blockierschutz (180 s Lauf nach 24 h Stillstand)
- **Relais K4** mit wählbarer Funktion: Wärmeanforderung an Kessel/Wärmepumpe, Umschaltung Heizen/Kühlen, Sammelstörmeldung (Ruhestrom), Zirkulationspumpe, Entfeuchter oder frei schaltbar

**Fühler**
- Vier DS18B20 – Vorlauf, Rücklauf, Außen, optional Raum – wahlweise **jeder an einem eigenen Pin** oder **alle an einem gemeinsamen Bus** (Einstellung im Web)
- Jeder Wert alternativ über MQTT (beliebiges Topic, Zahl oder JSON) oder Modbus, auch als automatischer Ersatz bei Fühlerausfall
- Außen und Raum auch von **Bluetooth-Sensoren**: RuuviTag (auch Pro) und **Shelly BLU H&T** (BTHome, auch verschlüsselt) – direkt empfangen, ohne Gateway

**Bedienung und Anbindung**
- Weboberfläche mit Live-Werten, Heizkurvengrafik, Verlauf (1 h bis 24 h) mit CSV-Download und Ereignisprotokoll
- MQTT mit Home-Assistant-Discovery (Thermostat, Sensoren, Schalter, Einstellwerte)
- Modbus TCP über WLAN und Modbus RTU über RS485 mit identischen Registern
- Firmware-Update über WLAN aus der Arduino IDE (ArduinoOTA), als Datei im Browser oder per Knopf **von www.glw8.at** (Prüfung und Download macht der Browser)
- Die Regelung läuft auch ohne WLAN weiter

---

## Hardware und Verdrahtung

### Bauteile

| Bauteil | Hinweis |
|---|---|
| ESP32-WROOM-32E 4-Kanal-Relaisplatine | 4 Relais (SRD-05VDC-SL-C) an GPIO 32/33/25/26, Status-LED GPIO 23, Taster IO0 und EN; Versorgung 5 V DC, 7–30 V DC oder 230-V-Variante; **kein USB** auf der Platine ([Beschreibung](https://prilchen.de/esp32-4-kanal-relais-entwicklungsplatine/)) |
| USB-Seriell-Adapter (3,3-V-Logik) | nur für das erste Aufspielen (GND, RX, TX, 5 V) |
| 3 × DS18B20, wasserdicht | Vorlauf, Rücklauf, Außen |
| 1 × DS18B20 (optional) | Raumfühler im Referenzraum – oder ein Raumthermometer über MQTT (z. B. Zigbee) |
| 4,7 kΩ | Pull-up von DATA nach 3,3 V: einer je Fühler (eigene Pins) bzw. einer für alle (gemeinsamer Bus) |
| PWM→0–10-V-Wandler (optional) | nur bei 0–10-V-Stellantrieb |
| RS485-Modul MAX485, 3,3 V (optional) | nur für Modbus RTU |
| Netzteil 12 oder 24 V DC | an den Eingang 7–30 V der Platine (oder die 230-V-Variante der Platine) |

### Pinbelegung

| GPIO | Funktion |
|---|---|
| 32 (Relais K1) | Mischer **AUF** – fest auf der Platine |
| 33 (Relais K2) | Mischer **ZU** – fest auf der Platine |
| 25 (Relais K3) | **Heizkreispumpe** – fest auf der Platine |
| 26 (Relais K4) | **Zusatzfunktion**, wählbar (Werkseinstellung Wärmeanforderung), Wechsler COM/NO/NC – siehe [Relais K4](#relais-k4-zusatzfunktion) |
| 23 | Status-LED auf der Platine (in 2.0.0 noch ohne Funktion) |
| 4 | DS18B20 **Vorlauf** – bei gemeinsamem Bus: **alle** DS18B20 |
| 13 | DS18B20 **Rücklauf** (nur bei eigenen Pins) |
| 14 | DS18B20 **Außen** (nur bei eigenen Pins) |
| 27 | DS18B20 **Raum**, optional (nur bei eigenen Pins) |
| 19 | PWM 1 kHz, 10 Bit für den 0–10-V-Wandler (optional) |
| 16 (UART2 RX) | RS485 RO – Empfang (nur bei Modbus RTU) |
| 17 (UART2 TX) | RS485 DI – Senden (nur bei Modbus RTU) |
| 18 | RS485 DE + /RE – Richtungsumschaltung (nur bei Modbus RTU) |
| 1 / 3 (UART0) | Programmieren und Meldungen über den USB-Seriell-Adapter |

Die Pins sind oben in `Heizungsregler_ESP32.ino` definiert (`PIN_…`). **Vor dem Verdrahten mit der Beschriftung auf der Rückseite der Platine abgleichen.** Gemieden werden GPIO 6–11 (Flash), 0/2/5/12/15 (Startbedingungen; GPIO 0 ist der Taster IO0) und 34–39 (nur Eingang; frei für spätere Erweiterungen). Frei bleiben außerdem 21/22 (z. B. für ein Display).

### Hinweise zur Verdrahtung

- **Fühler:** Die Anschlussart wählst du im Reiter *Fühler*, siehe [Temperaturfühler](#temperaturfühler).
  - **Eigene Pins:** Jeder DS18B20 braucht einen eigenen **4,7-kΩ-Pull-up** von DATA nach 3,3 V.
  - **Gemeinsamer Bus:** Alle DS18B20 hängen parallel an GPIO 4 (DATA, 3,3 V, GND), mit **einem** 4,7-kΩ-Pull-up. Bei langen Leitungen (mehr als ca. 20 m gesamt) eher 2,2–3,3 kΩ verwenden.
  - Die Fühler werden mit 3,3 V versorgt, nicht mit 5 V.
- **Mischer-Verriegelung:** Die Software schaltet AUF und ZU nie gleichzeitig und hält beim Richtungswechsel 300 ms Pause. Zusätzlich gehört eine **Hardware-Verriegelung** dazu: Phase über COM von K1; NO von K1 → Motor AUF; NC von K1 → COM von K2; NO von K2 → Motor ZU. So liegen nie beide Richtungen gleichzeitig am Motor. Das setzt voraus, dass die Platine COM, NO und NC jedes Relais herausführt.
- **230 V:** Die Relais sind für 10 A bei 250 V~ ausgelegt; bei preiswerten Platinen sind die Abstände zwischen Netz- und Kleinspannung aber oft knapp. Netzleitungen und Fühlerleitungen getrennt führen, Entstörglied (RC-Glied oder Varistor) am Stellantrieb vorsehen. Arbeiten an 230 V nur durch eine Elektrofachkraft.
- **Relaislogik:** Die Relais der Platine schalten bei HIGH. *Regler → Relais Low-aktiv* bleibt deshalb **aus**.
- **0–10 V:** GPIO 19 liefert ein PWM-Signal 0–3,3 V mit 1 kHz. Ein handelsübliches PWM→0–10-V-Modul macht daraus das Stellsignal.
- **Stromlaufplan:** Der Stromlaufplan aus Version 1 gilt für den Wemos D1 mini. Ein Plan für die ESP32-Platine folgt mit der Dokumentation von Version 2.

---

## Inbetriebnahme

1. **Arduino IDE vorbereiten** (Version 2.x empfohlen):
   - *Datei → Einstellungen → Zusätzliche Boardverwalter-URLs:*
     `https://espressif.github.io/arduino-esp32/package_esp32_index.json`
   - *Werkzeuge → Board → Boardverwalter:* **esp32 by Espressif Systems**, Version 3.x, installieren.
   - *Sketch → Bibliothek einbinden → Bibliotheken verwalten:* diese Bibliotheken installieren:

     | Bibliothek | Autor | Version |
     |---|---|---|
     | ArduinoJson | Benoit Blanchon | **7.x** (nicht 6.x) |
     | PubSubClient | Nick O'Leary | ≥ 2.8 |
     | OneWire | Paul Stoffregen | **≥ 2.3.8** (ältere laufen nicht mit dem ESP32-Core 3.x) |
     | DallasTemperature | Miles Burton | aktuell |
     | modbus-esp8266 | Alexander Emelianov | ≥ 4.1 (läuft trotz des Namens auch auf dem ESP32) |
     | NimBLE-Arduino | h2zero | 1.4.x oder ≥ 2.0 (Bluetooth-Empfang von RuuviTag und Shelly BLU H&T; die Version wird beim Übersetzen erkannt) |

     WebServer, Update, LittleFS, ESPmDNS, ArduinoOTA und Ticker gehören zum ESP32-Core.
   - Den **ganzen Ordner** `Heizungsregler_ESP32` in den Sketchbook-Ordner legen: alle `.ino`-Dateien, `web_ui.h` und den Unterordner `src/GlwOta`. Ordnername und Name der Hauptdatei `Heizungsregler_ESP32.ino` müssen gleich sein. Die Arduino IDE öffnet beim Laden der Hauptdatei alle übrigen `.ino`-Dateien als Reiter (siehe [Aufbau des Quelltexts](#aufbau-des-quelltexts)).
   - **Umstieg von einer Version vor 2.4.0:** Die alte Datei `Heizungsregler_ESP32.ino` durch den neuen Ordnerinhalt ersetzen. Bleibt eine alte, einteilige `.ino`-Datei mit anderem Namen im Ordner liegen, meldet die IDE doppelte Definitionen.
   - *Werkzeuge:* Board **ESP32 Dev Module**, Partition Scheme **Minimal SPIFFS (1.9MB APP with OTA/190KB SPIFFS)**, Flash Size 4MB, CPU Frequency 240 MHz. Mit dem Standard-Schema (1,2 MB je Programmbereich) wird es für Updates über WLAN knapp.
2. **Erstes Hochladen über den USB-Seriell-Adapter:** Die Platine hat keinen USB-Anschluss.
   - Adapter an GND, RX, TX und 5 V der Platine (RX des Adapters an TX der Platine und umgekehrt).
   - Taster **IO0** gedrückt halten, kurz **EN** drücken (oder einschalten), IO0 loslassen – der ESP32 wartet jetzt auf die Firmware.
   - In der Arduino IDE den Port des Adapters wählen und hochladen. Danach einmal **EN** drücken.
   - Spätere Updates gehen über WLAN (siehe [Firmware-Update über WLAN](#firmware-update-über-wlan-ota)).
3. **Mit dem Access-Point verbinden:** Beim ersten Start öffnet der Regler den Access-Point **`Heizungsregler-xxxxxx`** mit dem Passwort `heizung123`. Im Browser `http://192.168.4.1` aufrufen.
4. **WLAN eintragen:** Unter *Netzwerk* das WLAN eintragen und speichern. Der Regler startet neu und ist dann unter `http://heizungsregler.local` oder seiner IP-Adresse erreichbar. Wer von Version 1 umsteigt, kann stattdessen die Einstellungen aus Version 1 laden (nächster Abschnitt).
5. **Profil laden:** Unter *Heizkurve* das passende **Heizungsprofil** laden, Radiator- oder Fußbodenheizung.
6. **Fühler einrichten:** Unter *Fühler* die **Anschlussart** wählen (eigene Pins oder gemeinsamer Bus) und kontrollieren, ob alle DS18B20 gefunden wurden. Beim gemeinsamen Bus die Fühler den Rollen zuordnen.
7. **Motorlaufzeit eintragen:** Unter *Regler* die Laufzeit des Stellantriebs eintragen. Sie steht auf dem Typenschild, oft 120 s oder 140 s, und ist für die Stellungsschätzung entscheidend.
8. **Web-Passwort setzen:** Unter *Netzwerk* ein Web-Passwort vergeben, der Benutzer ist `admin`. Es schützt Einstellungen, Bedienung und Firmware-Updates. Das Anlagenschema bleibt als Nur-Anzeige ohne Passwort sichtbar (siehe [Weboberfläche](#weboberfläche)).
9. **Optional:** Zeitprogramm, MQTT, Home Assistant und Modbus einrichten.

### Einstellungen aus Version 1 übernehmen

1. Im **alten Regler** (Version 1) im Browser `http://<alte-ip>/api/config` aufrufen und die angezeigte Seite als Datei speichern (*Rechtsklick → Speichern unter*, Endung `.json`). Version 1 hat noch keinen eigenen Knopf dafür.
2. Im **neuen Regler** (Version 2) im Reiter *System*, Karte **Einstellungen sichern und übertragen**, auf *Einstellungen aus Datei laden* klicken und die Datei wählen.
3. Übernommen wird alles außer **Passwörtern** (stehen in der Datei als `********`) und **Relais Low-aktiv** (hängt vom Relaismodul ab; die ESP32-Platine schaltet bei HIGH). WLAN-Name, Hostname, MQTT und Modbus werden übernommen; ändern sie sich, startet der Regler neu.
4. Danach **Web-Passwort, WLAN-Passwort und MQTT-Passwort neu eintragen** und im Reiter *Fühler* kontrollieren, ob alle Fühler gefunden werden – die Pins sind andere als beim Wemos.

Das eigene Profil **GLW8** und die Gesamtzähler liegen in eigenen Dateien und werden nicht übertragen. Das Profil lässt sich im neuen Regler einfach neu speichern, sobald die Werte übernommen sind.

Mit *Einstellungen herunterladen* sicherst du die Einstellungen von Version 2 jederzeit als Datei (z. B. vor einem Update oder vor *Werkseinstellungen*).

---

## Weboberfläche

Erreichbar unter `http://heizungsregler.local` oder der IP-Adresse. Die Seite funktioniert auch ohne Internet, etwa im Access-Point-Modus. Sie folgt automatisch dem Hell- oder Dunkelmodus des Geräts und ist für Smartphones angepasst.

**Gestaltung:** ruhige Flächen in Grau und Weiß, Orange für Wärme und alles, was man auslösen kann, Blau für Rücklauf und Istwerte. Messwerte stehen in großer, gut lesbarer Schrift mit Dezimalkomma. Die Reiterleiste bleibt beim Scrollen oben stehen, der aktive Reiter ist unterstrichen. Mit der Tastatur bedienbare Elemente zeigen einen deutlichen Fokusrahmen.

**Anlagenschema (Übersicht):** Die Übersicht zeigt den Heizkreis als Bild: Kessel, Dreiwege-Mischer, Pumpe, Haus mit Heizkörper sowie Vor- und Rücklaufleitung. Die Messwerte stehen direkt an der Stelle, an der sie gemessen werden:

- Außentemperatur (mit gedämpftem Wert) links oben
- Mischerstellung und Laufrichtung (▲ öffnet / ▼ schließt / steht) neben dem Mischer; die beiden Ventilwege werden je nach Stellung kräftiger oder blasser
- Vorlauf-Ist groß über der Leitung, darunter der Vorlauf-Soll
- Raumtemperatur mit Raumsoll und Raumkorrektur im Haus
- Rücklauf und Spreizung unter der Rücklaufleitung

Läuft die Pumpe, dreht sich das Pumpensymbol und in den Leitungen ist die Fließrichtung zu sehen; ist sie aus, werden die Leitungen blasser. Wer in den Systemeinstellungen „Bewegung reduzieren“ eingeschaltet hat, sieht keine Animation. Auf dem Smartphone bleibt das Schema als Bild stehen, die Werte erscheinen darunter in einem zweispaltigen Raster in großer Schrift. Unter dem Schema folgen der Balken der Mischerstellung, der Zähler der Stellvorgänge und die Statusanzeigen. Rechts daneben (auf dem Smartphone darunter) wählt man die Betriebsart und stellt Raumsoll, festen Sollwert und Handstellung ein.

Im Kopf steht das **GLW8-Logo**: eine Flamme als Bildmarke und der Schriftzug „GLW8“. Ein Klick darauf führt zur Startseite. Dieselbe Flamme erscheint als Symbol im Browser-Tab. Beides ist direkt in die Firmware eingebettet und braucht keine Internetverbindung. Rechts im Kopf stehen Uhrzeit und IP-Adresse, auf dem Smartphone darunter.

| Reiter | Inhalt |
|---|---|
| **Übersicht** | Anlagenschema mit allen Messwerten, Mischerstellung und Zähler der Stellvorgänge (im Kühlbetrieb türkis mit Schneeflocke); Statusanzeigen (Pumpe mit Laufzeit seit Neustart, Kühlbetrieb, Feuchte und Taupunkt); Betriebsart; Schnelleinstellung von Raumsoll, Raumsoll Kühlen, festem Sollwert und Handstellung; Handschaltung von Mischer und Pumpe |
| **Verlauf** | Temperaturen, Mischerstellung und Pumpe der letzten 1, 6, 12 oder 24 Stunden oder **7 Tage** (Stundenmittel); Download als CSV-Liste |
| **Heizkurve** | Grafik der Heizkurve mit aktuellem Arbeitspunkt, per Maus oder Finger einstellbar; Heizungsprofile, Heizkurve und Grenzwerte, Rücklaufbegrenzung, Raumeinfluss, **Kühlen & Taupunktschutz**, Export der Heizkurve als CSV |
| **Zeitprogramm** | Absenkbetrieb, tägliches Absenkfenster oder Wochenprogramm |
| **Regler** | Stellantrieb (3-Punkt oder 0–10 V), Motorlaufzeit, PI-Parameter, Relais-Polarität, Volumenstrom; Karte **Relais K4** (Funktion, Zeiten, Zustand, bei „Frei schaltbar“ EIN/AUS) |
| **Fühler** | Zustand jedes Fühlers; Anschlussart (eigene Pins / gemeinsamer Bus) mit Zuordnungstabelle; Herkunft (Pin oder MQTT), MQTT-Topic und Korrektur je Fühler; MQTT-Topic der Raumfeuchte für den Taupunktschutz |
| **Netzwerk** | WLAN, Hostname, Web-Passwort, MQTT, Home Assistant, Modbus TCP und RTU, OTA |
| **System** | Kennwerte als Liste: Firmware, Profil, IP, WLAN-Signal, **Programmlaufzeit** (sekundengenau, läuft live mit, z. B. „13 Tage 00:04:16 · gestartet 04.10.2026 04:01“), freier Speicher (mit größtem Block, Zerstückelung und Minimum seit Start), **letzter Absturz** mit Ursache, Pumpenlaufzeit, Stellvorgänge, Modbus RTU, OTA mit letztem Update, Heizkurven-Soll und letzter Stellschritt mit Zeitstempel; Zählerkarte (seit Neustart und gesamt) mit Speichern/Zurücksetzen; Mischer kalibrieren, Firmware-Update, Neustart, Werkseinstellungen; Karte **Gerät und Netzwerk** mit MAC-Adressen, WLAN-Verbindung, IP-Konfiguration, MQTT-Broker, Chip, Flash, Dateisystem und Software-Ständen (siehe [Gerät und Netzwerk](#gerät-und-netzwerk)); Karte **Firmware von glw8.at** (auf neue Firmware prüfen und installieren); Ereignisprotokoll mit den letzten 67 Ereignissen (neueste oben, nummeriert, alle 5 s aufgefrischt) |
| **Hilfe** | Kurzanleitung für die Bedienung (ohne Hinweise zur Arduino IDE) mit den wichtigsten Informationen aus dieser README in aufklappbaren Abschnitten: Sicherheit, erste Inbetriebnahme, Reiter, Betriebsarten, Handschaltung, Heizkurve einstellen (mit Tipps „zu kalt / zu warm“), Heizkurve über 4 Stützpunkte (Einrichten und Feinabstimmung je Punkt), Profile, Zeitprogramm und Raum, Mischerregler, Fühler, Schutzfunktionen und Störungen, Anschlüsse, MQTT/Home Assistant/Modbus, Update und Zurücksetzen. Hostname, MQTT-Basis-Topic und Firmware-Version werden mit den echten Werten des Reglers angezeigt. Funktioniert ohne Internet. |

**Werte in den Diagrammen ablesen:** In allen Diagrammen (Heizkurve, Temperaturverlauf, Mischer/Pumpe) erscheinen die Werte, sobald man mit der Maus darüberfährt oder mit dem Finger darauf tippt. Eine gestrichelte senkrechte Linie markiert die Stelle, auf den Kurven sitzen kleine Kreise, und ein Kästchen zeigt die Zahlen:

- **Heizkurve:** Außentemperatur (in 0,5-°C-Schritten), der Vorlauf-Sollwert laut Kurve und der Raumsollwert, mit dem gerechnet wird. Ist der Wert durch *Vorlauf min/max* begrenzt oder liegt die Außentemperatur über der Heizgrenze, steht das darunter.
- **Verlauf:** Datum und Uhrzeit des nächstgelegenen Messpunkts (bei 1 h mit Sekunden), alle eingeblendeten Temperaturen, Mischerstellung und Pumpe, dazu wie lange das her ist. Die Anzeige gilt für beide Grafiken gleichzeitig, so sieht man direkt, wie der Mischer auf eine Temperatur reagiert hat.

Mit der Maus folgt die Anzeige dem Zeiger und verschwindet beim Verlassen der Grafik. Auf dem Smartphone tippt man auf die Grafik oder wischt waagrecht darüber; die Anzeige bleibt nach dem Loslassen stehen und verschwindet beim Tippen außerhalb. Senkrechtes Wischen scrollt die Seite wie gewohnt. Das Kästchen steht oben in der Grafik, damit der Finger es nicht verdeckt. Die Anzeige bleibt richtig, wenn neue Werte eintreffen.

**Nur-Anzeige ohne Passwort:** Ist ein Web-Passwort gesetzt, zeigt die Startseite zunächst nur das **Anlagenschema** mit allen Messwerten, der Mischerstellung und den Statusanzeigen – zum Anschauen, ohne Passwortabfrage. Die Betriebsart steht als erstes Etikett unter dem Bild, z. B. „Betriebsart: Automatik“. Zusätzlich lässt sich der Reiter **Verlauf** ansehen – mit allen Zeiträumen, Werteanzeige beim Überfahren und CSV-Download. Die übrigen Reiter, Betriebsart, Sollwerte und Handschaltung sind ausgeblendet; darüber steht „Nur Anzeige (Übersicht und Verlauf) – zum Bedienen und für die Einstellungen bitte anmelden“ mit dem Knopf **Anmelden**.
- **Anmelden** fragt Benutzer (`admin`) und Passwort ab und öffnet danach die volle Oberfläche. Der Browser merkt sich die Anmeldung; die volle Oberfläche bleibt in diesem Browser-Tab auch nach dem Neuladen erhalten.
- Die Nur-Anzeige aktualisiert sich alle **5 Sekunden** (angemeldet: alle 2 Sekunden). Das schont den Regler, wenn Seiten länger offen bleiben, etwa auf einem Wandtablett.
- **Hintergrund-Tabs pausieren:** Ist die Seite nicht sichtbar (anderer Tab, Fenster minimiert, Handy gesperrt), fragt sie den Regler gar nicht ab – weder Status noch Protokoll oder Verlauf. Beim Zurückkehren holt sie sofort frische Werte. Das gilt für die Nur-Anzeige und die angemeldete Oberfläche.
- Die Nur-Anzeige holt ihre Werte über `/api/view` und `/api/history`. Beide liefern nur Messwerte und Zustände – keine Einstellungen, keine Passwörter, keine Zähler, keine Netzwerkdaten.
- Alles andere bleibt geschützt: Einstellungen, Befehle, Handschaltung, Heizkurven-Liste, Protokoll, Fühler und Firmware-Update.
- Ohne Web-Passwort gibt es keine Nur-Anzeige; die Seite öffnet direkt die volle Oberfläche.

**Urheberhinweis:** Am unteren Rand jeder Seite steht „© 2026 erich.grassl@glw8.at · GLW8 Heizungsregler · Firmware …“, auf allen Reitern und auf der Update-Seite. Die Mailadresse ist anklickbar. Der Hinweis steht außerdem in beiden CSV-Listen (Heizkurve in der ersten Zeile, Verlauf in der letzten), in der Startmeldung am seriellen Monitor und in Home Assistant im Feld *Hersteller* des Geräts.

Kommt bei einem Fühler der Wert über MQTT, steht hinter seinem Namen „(MQTT)“. Wird der Raumwert nicht für die Regelung verwendet, steht dort „(nur Anzeige)“. Ist der Regler nicht erreichbar, zeigt der Kopf „Keine Verbindung zum Regler“.

---

## Betriebsarten

| Betriebsart | Web | MQTT | Verhalten |
|---|---|---|---|
| Aus | Aus | `off` | Mischer zu, Pumpe aus. Der Frostschutz bleibt aktiv. |
| Automatik | Automatik | `auto` | Vorlauf nach Heizkurve, mit Zeitprogramm, Raumeinfluss und Heizgrenze |
| Raumtemperatur | Raumtemperatur | `room` | Vorlauf **nur nach der Raumtemperatur**: Ein PI-Raumregler hebt bzw. senkt den Vorlauf-Sollwert, bis der Raum sein Soll hält – ohne Heizkurve, Heizgrenze und Außentemperatur. Braucht einen Raumfühler; ohne Raumwert Ersatzbetrieb (siehe [Regelung im Detail](#betriebsart-raumtemperatur-nur-nach-raumtemperatur)) |
| Fester Sollwert | Fester Sollwert | `manual` | Regelt auf einen festen Vorlauf-Sollwert, unabhängig vom Wetter |
| Handbetrieb | Handbetrieb | `hand` | Die Mischerstellung wird direkt in % vorgegeben, etwa zum Testen. Die Pumpe läuft. |
| Kühlen | Kühlen | `cool` | Über der Kühlgrenze Vorlauf nach Kühlkurve mit Raumeinfluss und Taupunktschutz, darunter ruht der Heizkreis (siehe [Kühlen](#kühlen)) |
| Heizen/Kühlen automatisch | Heizen/Kühlen | `heat_cool` | Unter der Heizgrenze heizen wie „Automatik“, über der Kühlgrenze kühlen, dazwischen ruht der Heizkreis |

Der Übertemperaturschutz wirkt in allen Betriebsarten. Wie die Regelung in jeder Betriebsart im Einzelnen arbeitet und welche Messwerte wie wirken, steht im nächsten Kapitel [Regelung im Detail](#regelung-im-detail).

### Handschaltung von Mischer und Pumpe

Für Inbetriebnahme, Wartung und Fehlersuche lassen sich Mischer und Pumpe unabhängig von der Betriebsart direkt von Hand schalten. In der Weboberfläche liegt dafür in der *Übersicht* unter der Betriebsart die Karte **Handschaltung**:

| Knopf | Wirkung |
|---|---|
| Mischer **▲ AUF** | Der Motor fährt durchgehend auf, bis *Stopp* gedrückt wird oder die Endlage erreicht ist (Restweg + 10 % der Motorlaufzeit). |
| Mischer **▼ ZU** | Wie AUF, nur in Richtung ZU. |
| Mischer **■ Stopp** | Der Motor hält sofort an; der Mischer bleibt in dieser Stellung. |
| Mischer **Automatik** | Die Handschaltung endet, der Regler bzw. die Betriebsart übernimmt den Mischer wieder. |
| Pumpe **Ein** / **Aus** | Die Pumpe läuft bzw. steht, unabhängig von der Regelung. |
| Pumpe **Automatik** | Die Regelung schaltet die Pumpe wieder selbst. |

Der hervorgehobene Knopf zeigt den aktuellen Zustand, daneben stehen Laufrichtung bzw. Stellung und die Restzeit. Ist eine Handschaltung aktiv, hat die Karte links einen orangen Rand, in den Statusanzeigen erscheint „Mischer von Hand“ bzw. „Pumpe von Hand“ und im Anlagenschema steht „(Hand)“ beim Mischer.

**Sicherheit:**
- Solange der Mischer von Hand geschaltet ist, verstellt ihn der Regler nicht. Die Regelung beginnt nach der Freigabe neu.
- Bei Übertemperatur endet die Handschaltung des Mischers sofort und er fährt ZU; *AUF* ist dann gesperrt.
- Frostschutz und Übertemperatur halten die Pumpe auch bei *Aus* in Betrieb (Anzeige „läuft trotz ‚Aus‘“). Der Blockierschutz bleibt aktiv.
- Jede Handschaltung endet **nach 24 Stunden** von selbst; jeder neue Handbefehl startet die Zeit neu. So bleibt die Anlage nicht versehentlich dauerhaft im Handbetrieb.
- Die Handschaltung wird nicht gespeichert. Nach einem Neustart regelt der Regler wieder selbst.
- Zwischen AUF und ZU hält der Motor die übliche Pause zum Motorschutz ein.

Bei einem 0–10-V-Antrieb fährt die Stellung mit der eingestellten Motorlaufzeit hoch bzw. herunter, genau wie bei einem 3-Punkt-Antrieb.

Unterschied zum **Handbetrieb** (Betriebsart): Dort gibt man eine feste Mischerstellung in % vor und die Pumpe läuft. Die Handschaltung dagegen fährt den Mischer direkt auf oder zu und schaltet die Pumpe ein oder aus. Sie wirkt in jeder Betriebsart und endet von selbst.

Die Handschaltung geht auch über MQTT, Home Assistant und Modbus (siehe dort).

---

## Regelung im Detail

Dieses Kapitel erklärt, wie der Regler in jeder Betriebsart den Vorlauf-Sollwert bildet, wie der Mischer ihn umsetzt und welche Messwerte dabei wirken. Die Zahlen in den Beispielen sind die Werkseinstellungen (Profil Radiatorheizung).

### Aufbau in zwei Ebenen

Jede Betriebsart besteht aus denselben zwei Ebenen:

```
 Messwerte ──► [1] Sollwertbildung ──► Vorlauf-Soll ──► [2] Vorlaufregelung ──► Mischer AUF/ZU
                   (je Betriebsart)                         (PI-Regler, für           Pumpe EIN/AUS
                                                             alle gleich)
                        ▲                                         ▲
                        └──────────── Schutzfunktionen ───────────┘
                         (Frostschutz, Übertemperatur, Taupunkt, Rücklaufbegrenzung)
```

1. **Sollwertbildung:** Hier unterscheiden sich die Betriebsarten. Sie legen fest, *wie warm* der Vorlauf sein soll, ob die Pumpe läuft und ob überhaupt geregelt wird.
2. **Vorlaufregelung:** Ein PI-Regler vergleicht den gemessenen Vorlauf mit dem Soll und verstellt den Mischer, bis beide übereinstimmen. Das läuft in allen Betriebsarten gleich, nur beim Kühlen mit umgekehrter Wirkrichtung (siehe [Vorlaufregelung](#vorlaufregelung-mischer)).

In der Betriebsart *Raumtemperatur* kommt vor die Vorlaufregelung ein zweiter, langsamer Regler für den Raum. Das ist eine **Kaskade**: Der Raumregler gibt dem Mischerregler den Vorlauf-Sollwert vor.

### Zeitlicher Ablauf

| Was | Wann |
|---|---|
| DS18B20-Fühler messen | alle 2 s (Messung anstoßen, 800 ms später abholen) |
| externe Werte (MQTT, Modbus) | sobald sie eintreffen; ohne neuen Wert nach der Zeitüberwachung verworfen |
| gedämpfte Außentemperatur | bei jeder Messung nachgeführt |
| **Regelzyklus** (Sollwert, Schutz, PI-Schritt, Pumpe) | alle *Abtastzeit* Sekunden (Radiator 10 s, Fußboden 20 s), zusätzlich sofort nach jeder Einstellungsänderung |
| Mischerimpuls | sobald die aufsummierten Stellschritte die *Min. Impulslänge* ergeben und der Antrieb frei ist |

Ein Regelzyklus läuft immer in derselben Reihenfolge ab:

1. **Außentemperatur:** gedämpfter Wert, ersatzweise der Messwert, ohne Außenwert 0 °C.
2. **Absenkung:** Ist laut Zeitprogramm Absenkzeit, gilt *Raumsoll − Absenkung*.
3. **Heizgrenze** und **Frostschutz** auswerten.
4. **Heizkurven-Soll** berechnen (immer, auch zur Anzeige), Kühlgrenze und Kühlkurve, Taupunkt.
5. **Raumeinfluss:** Raumaufschaltung und Raumabschaltung bzw. Raumregler.
6. **Sollwert je Betriebsart** festlegen und entscheiden, ob geregelt wird und ob die Pumpe läuft.
7. **Rücklaufbegrenzung** und **Grenzen** (Vorlauf min/max bzw. Taupunkt-Untergrenze).
8. **Übertemperatur** und **Taupunktalarm** prüfen.
9. **Mischer** ansteuern (Rangfolge siehe [Schutzfunktionen](#vorrang-der-schutzfunktionen)) und **Pumpe** schalten.

### Die Messwerte und was sie bewirken

| Messwert | Aufbereitung | wirkt auf |
|---|---|---|
| **Vorlauf** | direkt | Istwert der Vorlaufregelung (alle Betriebsarten außer Aus/Hand); Übertemperaturschutz; Taupunktalarm beim Kühlen; Anti-Windup des Raumreglers |
| **Rücklauf** | direkt | nur die Rücklaufbegrenzung (wenn *Rücklauf max* > 0, nur beim Heizen); sonst Anzeige und Verlauf |
| **Außen, ungedämpft** | direkt | **Frostschutz** (reagiert sofort) |
| **Außen, gedämpft** | Tiefpass mit *Dämpfung Außentemperatur* (Radiator 45 min, Fußboden 180 min) | **Heizkurve**, **Heizgrenze**, **Kühlkurve**, **Kühlgrenze**; Startwert des Raumreglers |
| **Raum** | direkt (DS18B20 oder extern, z. B. Zigbee über MQTT) | Raumaufschaltung und Raumabschaltung (nur mit *Raumtemperatur berücksichtigen*); **Raumregler** (Betriebsart *Raumtemperatur*, immer); Taupunkt beim Kühlen |
| **Raumfeuchte** | extern (MQTT/Modbus), mit Zeitüberwachung | Taupunkt → Untergrenze des Vorlaufs beim Kühlen |
| **Mischerstellung** | geschätzt aus Laufzeit und Motorlaufzeit (kein Messwert) | Endlagenerkennung; Anti-Windup des Raumreglers |
| **Uhrzeit** | NTP | Zeitprogramm (Absenkung), Feiertage |

**Gedämpfte Außentemperatur:** `neu = alt + (Messwert − alt) · Δt / (τ + Δt)` mit τ = Dämpfung in Sekunden. Ein Kälteeinbruch von 10 K wirkt bei 45 min Dämpfung nach 45 min zu etwa 63 %, nach 2¼ h zu 95 %. So folgt der Regler dem Trend wie das träge Gebäude und nicht jeder Wolke oder jedem Sonnenstrahl auf dem Fühler. Bei Dämpfung 0 gilt der Messwert direkt.

### Betriebsart „Aus“

- **Sollwert:** keiner, der Mischer fährt zu, die Pumpe ist aus.
- **Es wirken nur noch Schutzfunktionen:**
  - **Frostschutz:** Ungedämpfte Außentemperatur unter *Frostschutz* (3 °C) → Pumpe ein, Vorlauf auf *Vorlauf min* regeln. Aus erst über *Frostschutz* + 1 K.
  - **Übertemperatur:** Vorlauf über *Vorlauf max* + 5 K → Mischer zu, Pumpe ein (die Wärme soll abfließen).
- **Messwerte:** Außen (roh) für den Frostschutz, Vorlauf für Übertemperatur und Frostschutz-Regelung. Raum und Rücklauf haben keinen Einfluss.

### Betriebsart „Automatik“ (witterungsgeführt)

Der klassische Betrieb: Die Außentemperatur bestimmt über die Heizkurve den Vorlauf, der Raum korrigiert nur.

```
Raumsoll_wirksam = Raumsoll − Absenkung (in der Absenkzeit)
Kurve            = Heizkurve(Raumsoll_wirksam, Außen_gedämpft)        → begrenzt auf Vorlauf min … max
Raumkorrektur    = Raumaufschaltung · (Raumsoll_wirksam − Raum),       begrenzt auf ± max. Korrektur
Vorlauf-Soll     = Kurve + Raumkorrektur − Rücklaufbegrenzung          → begrenzt auf Vorlauf min … max
```

- **Heizgrenze:** Gedämpfte Außentemperatur über *Heizgrenze* + 0,5 K → Heizkreis aus (Mischer zu, Pumpe aus). Wieder ein unter *Heizgrenze* − 0,5 K.
- **Raumabschaltung** (wenn > 0): Raum um diesen Wert über dem wirksamen Soll → Heizkreis aus, wieder ein 0,5 K darunter.
- **Absenkung:** Sie senkt den Raumsollwert *in der Formel*. Die Kurve sinkt dadurch, bei Kälte stärker als bei mildem Wetter. Zusätzlich korrigiert die Raumaufschaltung auf das abgesenkte Soll.

**Einfluss der Messwerte:**

| Messwert | Einfluss |
|---|---|
| Außen (gedämpft) | **Hauptgröße**: 1 K kälter → bei Neigung 1,0 und 0 °C außen etwa +0,9 K Vorlauf; entscheidet über Heizgrenze |
| Raum | nur, wenn *Raumtemperatur berücksichtigen* an ist: ±*Faktor* K Vorlauf je K Abweichung (Faktor 3: Raum 0,5 K zu kalt → +1,5 K), höchstens ±*max. Korrektur*; Raumabschaltung |
| Rücklauf | nur bei *Rücklauf max* > 0: je K Überschreitung *Faktor* K weniger Vorlauf |
| Vorlauf | Istwert für den Mischer |

**Beispiel:** Außen gedämpft 0 °C, Raumsoll 20 °C, Neigung 1,0, Raum 19,5 °C, Faktor 3:
Kurve 42,3 °C + Raumkorrektur 3 · 0,5 = +1,5 K → **Vorlauf-Soll 43,8 °C**.
In der Absenkzeit (−4 K, Raumsoll 16 °C) mit Raum noch 19,0 °C: Kurve 34,6 °C, Korrektur 3 · (16 − 19) = −9 K → 25,6 °C, also knapp über *Vorlauf min* 25 °C. Das Haus kühlt langsam aus, wie gewünscht.

**Bei Fühlerausfall:**
- Außen: Rechnung mit 0 °C weiter, das Haus bleibt warm. Ohne Außenwert kann der Frostschutz nicht auslösen; ein bereits aktiver Frostschutz bleibt aktiv.
- Raum: Die Raumkorrektur entfällt, die Regelung läuft rein witterungsgeführt.
- Vorlauf: Der Mischer fährt zu (ohne Istwert keine Regelung).

### Betriebsart „Raumtemperatur“ (nur nach Raumtemperatur)

Hier bestimmt **allein der Raum** den Vorlauf. Es gibt keine Heizkurve und keine Heizgrenze, die Außentemperatur spielt keine Rolle. Ein PI-**Raumregler** hebt den Vorlauf-Sollwert, solange der Raum zu kalt ist, und senkt ihn, solange er zu warm ist, bis der Raum sein Soll hält. Das ist sinnvoll ohne Außenfühler, bei Häusern, deren Bedarf stark von Sonne oder Innenwärme abhängt, oder wenn eine Heizkurve schwer zu treffen ist.

```
e            = Raumsoll_wirksam − Raum                       (> 0: Raum zu kalt)
P            = Verstärkung · e                                (sofortige Reaktion)
Grundwert I += Verstärkung · e · Δt / Nachstellzeit           (langsames Nachführen)
Vorlauf-Soll = I + P − Rücklaufbegrenzung                    → begrenzt auf Vorlauf min … max
```

- **Grundwert (I-Anteil):** der Vorlauf, den das Haus bei diesem Wetter braucht. Er stellt sich von selbst ein. Bleibt der Raum dauerhaft 0,3 K zu kalt, steigt er um *Verstärkung* · 0,3 K je *Nachstellzeit*, also bei 12 K/K und 60 min um 3,6 K pro Stunde, bis die Abweichung verschwindet. **Startwert:** das Heizkurven-Soll, wenn ein Außenwert vorliegt, sonst *Vorlauf fest*.
- **Raumanteil (P-Anteil):** reagiert sofort. Bei 12 K/K hebt ein 0,5 K zu kalter Raum den Vorlauf um 6 K. Das bringt schnell Wärme, z. B. nach dem Lüften.
- **Anti-Windup:** Der Grundwert wächst nicht weiter, wenn das Soll schon an *Vorlauf max* liegt oder der Mischer ganz offen ist und der Vorlauf trotzdem mehr als 2 K unter dem Soll bleibt (Kessel zu kalt). Nach unten hört er bei *Vorlauf min* auf. So schießt der Raum später nicht über.
- **Heizpause:** Liegt der Raum um *Heizpause ab* (Radiator 0,5 K, Fußboden 1,0 K) über dem Soll, fährt der Mischer zu und die Pumpe aus. Geheizt wird wieder, sobald der Raum unter das Soll fällt. Der Grundwert bleibt währenddessen stehen. 0 = nie Pause (der Vorlauf sinkt dann höchstens auf *Vorlauf min*).
- **Absenkung:** Sie senkt das Raumsoll, der Regler nimmt den Vorlauf entsprechend zurück oder macht Pause. Am Ende der Absenkzeit hebt der P-Anteil den Vorlauf sofort an.
- **Der Raumwert wird immer verwendet**, auch wenn *Raumtemperatur berücksichtigen* aus ist. Raumaufschaltung und Raumabschaltung sind in dieser Betriebsart abgeschaltet, sie würden doppelt wirken.
- **Ohne Raumwert (Ersatzbetrieb):** Fehlt der Raumwert (Fühler gestört, Zigbee-Batterie leer, Zeitüberwachung abgelaufen), regelt der Regler mit Außenwert wie *Automatik* ohne Raumeinfluss (Heizkurve, Heizgrenze), ohne Außenwert auf *Vorlauf fest*. Das steht als rotes Etikett in der Übersicht und im Ereignisprotokoll. Kommt der Raumwert zurück, macht der Raumregler mit seinem alten Grundwert weiter.
- **Frostschutz und Übertemperatur** wirken wie überall; der Frostschutz braucht dafür einen Außenwert.

**Einstellwerte** (Karte *Raumregelung* im Reiter *Heizkurve*; gehören zu den Profilen):

| Einstellung | Radiator | Fußboden | Wirkung |
|---|---|---|---|
| Verstärkung `rcKp` | 12 K/K | 6 K/K | größer = schnellere Reaktion, aber Neigung zum Pendeln |
| Nachstellzeit `rcTn` | 60 min | 360 min | kürzer = schnelleres Nachführen des Grundwerts; Fußboden braucht wegen des trägen Estrichs viel länger. 0 = nur P-Anteil (dann bleibt eine Abweichung) |
| Heizpause ab `rcOff` | 0,5 K | 1,0 K | Pause bei deutlichem Wärmeüberschuss; bei Fußboden bewusst größer, sonst pendelt der Raum zwischen Pause und Heizen |

Die Werte habe ich an einem einfachen Hausmodell geprüft. Mit ihnen blieb der Raum dort bei gleichem Wetter auf etwa ±0,1 K (Heizkörper) bzw. ±0,2 K (Fußboden), auch nach einem Kälteeinbruch von 0 auf −10 °C. Das echte Haus ist anders. **Einstellen:** Pendelt der Raum mit einer Periode von einigen Stunden, Verstärkung kleiner und Nachstellzeit länger. Kommt er nach einer Störung (Lüften, Absenkung) zu langsam zurück, Verstärkung größer. Immer nur einen Wert ändern und einen Tag beobachten.

**Einfluss der Messwerte:**

| Messwert | Einfluss |
|---|---|
| Raum | **einzige Führungsgröße**: 0,1 K Abweichung → sofort 1,2 K Vorlauf (P), dazu 1,2 K pro Stunde Nachführen (I) |
| Außen | kein Einfluss auf den Sollwert; nur Startwert des Grundwerts, Frostschutz und Ersatzbetrieb |
| Rücklauf | Rücklaufbegrenzung wie bei *Automatik* |
| Vorlauf | Istwert für den Mischer; Anti-Windup |

**Beispiel:** Grundwert 41,0 °C, Raumsoll 20,0 °C, Raum 19,7 °C → e = 0,3 K, P = 12 · 0,3 = 3,6 K → **Vorlauf-Soll 44,6 °C**. Bleibt der Raum eine Stunde bei 19,7 °C, steigt der Grundwert auf 44,6 °C und das Soll auf 48,2 °C. Erreicht der Raum 20,0 °C, ist P = 0 und das Soll gleich dem Grundwert.

**Was diese Betriebsart braucht:** einen zuverlässigen Raumwert aus einem **Referenzraum** (keine Fremdwärme, Thermostatventile dort offen). Ein Raumthermometer, das nur in 0,5-K-Schritten meldet, ist zu grob: Jeder Schritt würde das Soll um 6 K springen lassen. 0,1 K Auflösung ist nötig. Weil der Raum der einzige Maßstab ist, wirken sich Sonne oder ein Kaminofen im Referenzraum auf das ganze Haus aus: Dann bekommen die anderen Räume zu wenig Wärme. In solchen Fällen ist *Automatik* mit Raumaufschaltung die robustere Wahl.

### Betriebsart „Fester Sollwert“

```
Vorlauf-Soll = Vorlauf fest − Rücklaufbegrenzung   → begrenzt auf Vorlauf min … max
```

- Pumpe läuft dauernd, keine Heizgrenze, keine Absenkung, kein Raumeinfluss.
- **Messwerte:** Vorlauf (Istwert), Rücklauf (Begrenzung), Außen nur für den Frostschutz. Der Raum wird nur angezeigt.
- Gedacht für Inbetriebnahme, Estrich-Aufheizprogramme von Hand, Warmwasser-Vorrang-Tests oder als Notbetrieb ohne Außenfühler.

### Betriebsart „Handbetrieb“

- Der Mischer fährt auf *Mischer %*, die Pumpe läuft. Es gibt **keine Regelung**; Vorlauf, Außen und Raum werden nur angezeigt.
- Der **Übertemperaturschutz** hat trotzdem Vorrang und schließt den Mischer.
- Der Frostschutz regelt im Handbetrieb nicht (die Pumpe läuft ohnehin).
- Nicht zu verwechseln mit der **Handschaltung** (Mischer AUF/ZU, Pumpe EIN/AUS von Hand), die in jeder Betriebsart für höchstens 24 h möglich ist.

### Betriebsart „Kühlen“

Voraussetzung: Die Wärmepumpe liefert kaltes Wasser an den Mischer. Mischer **AUF** bedeutet dann *mehr kaltes Wasser* und damit einen *kälteren* Vorlauf; die Wirkrichtung ist umgekehrt.

```
Kühlkurve     = Gerade durch A (22 °C → 22 °C) und B (32 °C → 18 °C) über der gedämpften Außentemperatur
Raumkorrektur = Raumaufschaltung · (Raumsoll Kühlen − Raum)          (Raum zu warm → negativ → kälter)
Untergrenze   = max(Vorlauf min Kühlen, Taupunkt + Abstand)          bzw. Ersatzwert ohne Raumfeuchte
Vorlauf-Soll  = Kühlkurve + Raumkorrektur                           → begrenzt auf Untergrenze … wärmerer Kurvenpunkt
```

- **Kühlgrenze:** gekühlt wird erst über *Kühlgrenze* + 0,5 K (gedämpft), Ende unter *Kühlgrenze* − 0,5 K. Dazwischen ruht der Heizkreis (Mischer zu, Pumpe aus).
- **Keine Absenkung** beim Kühlen, es gilt *Raumsoll Kühlen*.
- **Taupunktschutz:** Der Taupunkt wird aus Raumtemperatur und Raumfeuchte berechnet (Magnus-Formel). Das Soll geht nie unter *Taupunkt + Abstand*. Fällt der gemessene Vorlauf trotzdem unter *Taupunkt + 0,5 K*, schließt der Mischer sofort (Taupunktalarm), die Pumpe läuft weiter.
- **Bei Frost** (ungedämpft unter *Frostschutz*) wird nie gekühlt.

**Einfluss der Messwerte:**

| Messwert | Einfluss |
|---|---|
| Außen (gedämpft) | Kühlkurve: 1 K wärmer → 0,4 K kälterer Vorlauf; Kühlgrenze |
| Raum | Raumaufschaltung (mit *Raumtemperatur berücksichtigen*), Raumabschaltung (Raum um den Wert *unter* dem Soll → Kühlen aus) und **Taupunkt** |
| Raumfeuchte | **Taupunkt** → Untergrenze des Vorlaufs; ohne Feuchtewert gilt die feste Ersatz-Untergrenze (20 °C) |
| Vorlauf | Istwert (umgekehrte Wirkrichtung), Taupunktalarm |
| Rücklauf | kein Einfluss (keine Rücklaufbegrenzung beim Kühlen) |

**Beispiel:** Außen gedämpft 30 °C → Kühlkurve 18,8 °C. Raum 25 °C bei Soll 24 °C → Korrektur 3 · (24 − 25) = −3 K → 15,8 °C. Raumfeuchte 60 % → Taupunkt 16,7 °C, Untergrenze 16,7 + 2 = 18,7 °C → **Vorlauf-Soll 18,7 °C**, Etikett „Vorlauf an Taupunktgrenze“.

### Betriebsart „Heizen/Kühlen automatisch“

Die gedämpfte Außentemperatur entscheidet zwischen drei Zuständen:

| Außen (gedämpft) | Zustand | Regelung |
|---|---|---|
| unter *Heizgrenze* | heizt | wie *Automatik* (Heizkurve, Absenkung, Raumeinfluss) |
| zwischen Heiz- und Kühlgrenze | ruht | Mischer zu, Pumpe aus |
| über *Kühlgrenze* | kühlt | wie *Kühlen* |

Beide Grenzen haben ±0,5 K Hysterese, und die Kühlgrenze liegt immer mindestens 2 K über der Heizgrenze. So kann der Kreis nicht ständig zwischen Heizen und Kühlen wechseln. Bei jedem Wechsel beginnt der Mischerregler neu, weil sich die Wirkrichtung umdreht.

### Vorlaufregelung (Mischer)

Für alle Betriebsarten mit Regelung gleich (Einzelheiten unter [PI-Regler](#pi-regler)):

```
e  = Vorlauf-Soll − Vorlauf-Ist                 (|e| < Totzone → 0)
ΔY = Kp · (e − e_alt) + Kp · Abtastzeit / Tn · e  in %   (Kühlen: ΔY umgekehrt)
```

- **3-Punkt-Antrieb:** ΔY wird zu einem Impuls von `ΔY / 100 · Motorlaufzeit`. Kleine Schritte werden aufsummiert, bis sie die *Min. Impulslänge* ergeben. In der Endlage wird höchstens 20 % der Laufzeit nachgetaktet.
- **0–10-V-Antrieb:** Die Stellung wird direkt ausgegeben.
- Mit Kp 3 %/K und 1 K Abweichung bewegt sich der Mischer pro Abtastschritt von 10 s um etwa 0,2 % (I-Anteil) zusätzlich zum P-Sprung bei Änderungen. Bei 120 s Motorlaufzeit sind das 0,24 s pro Schritt, also wird meist alle 2–3 Schritte ein Impuls von ≥ 400 ms ausgegeben.

### Vorrang der Schutzfunktionen

Der Mischer folgt in jedem Zyklus der **ersten** zutreffenden Zeile:

| Rang | Bedingung | Mischer | Pumpe |
|---|---|---|---|
| 1 | Kalibrierfahrt läuft | fährt zu (abwarten) | wie Betriebsart |
| 2 | **Übertemperatur** (Vorlauf > max + 5 K, aus unter max) | **zu**, beendet Handschaltung | **ein** |
| 3 | **Taupunktalarm** (Kühlen) | **zu**, beendet Handschaltung | ein |
| 4 | Handschaltung des Mischers | bleibt wie von Hand gestellt | – |
| 5 | Betriebsart *Handbetrieb* | auf *Mischer %* | ein |
| 6 | Regelung aktiv | PI-Regler | ein |
| 7 | sonst (Aus, Heizgrenze, Heizpause, Vorlauffühler defekt) | zu | aus |

Der **Frostschutz** wirkt eine Ebene vorher: Wird gerade nicht geregelt (Aus, Sommer, Heizpause) und ist es frostig, schaltet er Regelung und Pumpe ein und regelt auf *Vorlauf min*. Die **Pumpe** läuft bei Übertemperatur und Frost auch dann, wenn sie von Hand ausgeschaltet wurde.

### Übersicht: Welcher Messwert wirkt in welcher Betriebsart?

| Messwert | Aus | Automatik | Raumtemperatur | Fester Sollwert | Hand | Kühlen | Heizen/Kühlen |
|---|---|---|---|---|---|---|---|
| Vorlauf | Schutz | Istwert, Schutz | Istwert, Schutz | Istwert, Schutz | Schutz | Istwert, Taupunktalarm | Istwert, Schutz |
| Rücklauf | – | Begrenzung¹ | Begrenzung¹ | Begrenzung¹ | – | – | Begrenzung¹ (heizen) |
| Außen roh | Frost | Frost | Frost | Frost | – | Frost (kein Kühlen) | Frost |
| Außen gedämpft | – | **Heizkurve**, Heizgrenze | Startwert, Ersatz | – | – | **Kühlkurve**, Kühlgrenze | **Heiz-/Kühlkurve**, beide Grenzen |
| Raum | – | Korrektur² | **Führungsgröße** | – | – | Korrektur², Taupunkt | Korrektur², Taupunkt |
| Raumfeuchte | – | – | – | – | – | **Taupunkt** | Taupunkt (kühlen) |
| Uhrzeit | – | Absenkung | Absenkung | – | – | – | Absenkung (heizen) |

¹ nur wenn *Rücklauf max* > 0 ² Raumaufschaltung und Raumabschaltung, nur mit *Raumtemperatur berücksichtigen*

---

## Heizkurve

Der Regler kennt zwei Kurvenarten: die **Formel** mit Neigung und Niveau (Werkseinstellung, dieser Abschnitt) und **[4 Stützpunkte](#heizkurve-über-4-stützpunkte)**, bei denen du den Vorlauf für vier Außentemperaturen selbst festlegst. Grenzen, Dämpfung, Heizgrenze und Rücklaufbegrenzung gelten für beide.

Formel:

```
d    = T_außen − T_raum
T_VL = T_raum + Niveau − Neigung · d · (1,4347 + 0,021·d + 0,0002479·d²)
```

- **Grenzen:** Das Ergebnis wird auf *Vorlauf min* und *Vorlauf max* begrenzt.
- **Raumtemperatur in der Formel:** Hier steht der Raumsollwert, im Absenkbetrieb um die Absenkung verringert.
- **Neigung:** Sie bestimmt, wie stark der Vorlauf mit sinkender Außentemperatur steigt. Richtwerte: Fußbodenheizung 0,3–0,8, Heizkörper 0,8–1,6.
- **Niveau:** Es verschiebt die ganze Kurve nach oben oder unten, etwa wenn es bei jedem Wetter gleichmäßig zu kalt ist.
- **Gedämpfte Außentemperatur:** Ein Tiefpass glättet die Außentemperatur (Standard 45 min, bei Fußbodenheizung 180 min). So reagiert der Regler auf den Trend wie das träge Gebäude und nicht auf jede Wolke.
- **Heizgrenze:** Liegt die gedämpfte Außentemperatur mehr als 0,5 K über der Heizgrenze, schaltet der Heizkreis ab (Sommerbetrieb). Unter Heizgrenze − 0,5 K schaltet er wieder ein.
- **Rücklaufbegrenzung:** Übersteigt der Rücklauf *Rücklauf max*, wird der Vorlauf-Sollwert um *Faktor × Überschreitung* gesenkt. Das ist bei Fernwärme oder Brennwertkessel sinnvoll. *Rücklauf max* = 0 schaltet die Funktion aus.
- **Außenfühlerfehler:** Der Regler rechnet mit 0 °C weiter, damit das Haus warm bleibt.

### Heizkurve über 4 Stützpunkte

Seit Version 1.34.0 gibt es zwei **Kurvenarten**:

| Kurvenart | Einstellung | Wie die Kurve entsteht |
|---|---|---|
| **Formel** (Werkseinstellung) | `curveMode` = 0 | aus *Neigung* und *Niveau* mit der Formel oben |
| **4 Stützpunkte** | `curveMode` = 1 | Du legst für vier Außentemperaturen den Vorlauf selbst fest; die Kurve läuft glatt durch alle vier Punkte |

#### Wann lohnen sich Stützpunkte?

Die Formel hat nur zwei Stellgrößen. Damit trifft man die meisten Häuser gut, aber nicht alle. Typische Fälle für Stützpunkte:

- **Nur bei mildem Wetter (0 … +10 °C) zu kühl**, bei Kälte passt es. Mit der Formel müsste man das Niveau heben und dafür die Neigung senken; mit Stützpunkten hebst du einfach Punkt 3 an.
- **Nur bei strengem Frost zu kalt**, z. B. weil ein Raum schlecht gedämmt ist. Punkt 1 anheben, der Rest bleibt.
- **Gemischte Anlagen** oder Niedertemperatur-Heizkörper, deren Kennlinie anders gekrümmt ist als die Formel.
- **Eine vorhandene Kurve übernehmen**, z. B. vom alten Kesselregler oder aus einer Heizlastberechnung, die als Wertetabelle vorliegt.

Wer mit Neigung und Niveau zufrieden ist, braucht nichts zu ändern: Die Formel bleibt die Werkseinstellung, und bestehende Anlagen regeln nach einem Update genau wie vorher.

#### Die Einstellungen

Alle Werte stehen in der Karte **Heizkurve über Stützpunkte** im Reiter *Heizkurve*, unter *Heizkurve & Grenzen*.

| Feld | Name in API/MQTT | Bereich | Werkseinstellung | Bedeutung |
|---|---|---|---|---|
| Kurvenart | `curveMode` | 0 / 1 | 0 (Formel) | 0 = Formel, 1 = 4 Stützpunkte |
| Punkt 1: Außen °C | `cpOut1` | −30 … +30 | −15 | kältester Punkt |
| Punkt 1: Vorlauf °C | `cpFlow1` | 10 … 90 | 55,1 | Vorlauf bei Punkt 1, Raumsoll 20 °C |
| Punkt 2: Außen °C | `cpOut2` | −30 … +30 | −5 | |
| Punkt 2: Vorlauf °C | `cpFlow2` | 10 … 90 | 46,6 | |
| Punkt 3: Außen °C | `cpOut3` | −30 … +30 | +5 | |
| Punkt 3: Vorlauf °C | `cpFlow3` | 10 … 90 | 37,6 | |
| Punkt 4: Außen °C | `cpOut4` | −30 … +30 | +15 | wärmster Punkt |
| Punkt 4: Vorlauf °C | `cpFlow4` | 10 … 90 | 26,7 | |
| Raumsoll-Einfluss K/K | `cpRoomK` | 0 … 5 | 1,8 | Vorlaufänderung je K Raumsoll über/unter 20 °C |

Die Werkseinstellungen der Punkte liegen genau auf der Formelkurve mit Neigung 1,0 (Profil Radiatorheizung). Wer nur die Kurvenart umschaltet, ohne Punkte zu ändern, bekommt also fast dieselbe Kurve wie vorher (Abweichung höchstens 0,8 K, siehe Beispiel unten).

**Regeln für die Punkte:**
- Die Außentemperaturen müssen **aufsteigen**: Punkt 1 ist der kälteste, Punkt 4 der wärmste.
- Zwischen zwei Punkten liegen **mindestens 2 K**. Gibt man zu dicht liegende oder vertauschte Werte ein, schiebt der Regler die Punkte selbst zurecht: Jeder Punkt wird auf mindestens „Vorgänger + 2 K“ gesetzt; würde Punkt 4 dabei über +30 °C rutschen, wird von rechts her nach links geschoben. Beispiel: Eingabe −15 / −5 / −6 / +15 ergibt −15 / −5 / **−3** / +15. Nach dem Speichern zeigt das Formular die tatsächlich übernommenen Werte.
- Die Vorlaufwerte müssen nicht fallen. Steigt die Kurve irgendwo an (wärmerer Vorlauf bei wärmerem Wetter), rechnet der Regler das trotzdem sauber, nur ist das für eine Heizung selten sinnvoll.
- Werte außerhalb der Bereiche werden auf die Grenze gesetzt.

#### So rechnet der Regler

```
T_VL = Spline(T_außen) + Raumsoll-Einfluss · (T_raum − 20) + Niveau
```

danach wie immer auf *Vorlauf min* … *Vorlauf max* begrenzt. Im Einzelnen:

1. **Spline zwischen den Punkten.** Zwischen zwei benachbarten Punkten verbindet ein kubisches Kurvenstück die beiden Vorlaufwerte. Die Steigungen an den Punkten wählt der Regler nach dem Verfahren von **Fritsch und Carlson** („PCHIP“, monotone stückweise kubische Hermite-Interpolation):
   - Die Kurve läuft **exakt durch jeden Punkt**.
   - Sie ist **glatt**, hat also an den Punkten keinen Knick.
   - Sie ist **monoton** und **schwingt nicht über**: Zwischen zwei Punkten liegt der Vorlauf immer zwischen deren Werten. Fällt die Kurve von Punkt 1 bis 4, fällt sie überall.
   - Ein Punkt beeinflusst nur die beiden Abschnitte links und rechts von ihm. Verschiebt man Punkt 3, ändert sich die Kurve zwischen Punkt 2 und 4, nicht aber bei −15 °C.

   Das ist der Grund, warum der Regler keine Bézier-Kurve verwendet: Bei Bézier liegen die Kontrollpunkte neben der Kurve, nicht auf ihr, und eine Kurve über der Außentemperatur lässt sich damit nur umständlich ausrechnen.

2. **Außerhalb der Punkte.** Links von Punkt 1 und rechts von Punkt 4 läuft die Kurve **geradlinig** weiter, mit der Steigung, die sie am Randpunkt hat. Bei −20 °C außen und Punkt 1 bei −15 °C wird also verlängert. *Vorlauf max* begrenzt das Ergebnis wie gewohnt.

3. **Raumsoll.** Die Punkte gelten für **Raumsoll 20 °C**. Liegt das wirksame Raumsoll darüber oder darunter, verschiebt sich die ganze Kurve um *Raumsoll-Einfluss × Abweichung*. Das gilt auch für die **Absenkung**: Mit 4 K Absenkung (Raumsoll 16 °C) und 1,8 K/K liegt die Kurve 7,2 K tiefer.
   Bei der Formel wirkt das Raumsoll nicht überall gleich stark, sondern bei Kälte etwas stärker als bei mildem Wetter. Bei den Stützpunkten ist es bewusst ein fester Wert, damit die Wirkung leicht nachvollziehbar bleibt. Richtwerte: Heizkörper 1,5–2 K/K, Fußbodenheizung 1–1,5 K/K. *Aus Formel übernehmen* setzt den passenden Wert für die aktuelle Neigung.

4. **Niveau.** Das Niveau (`shift`) aus *Heizkurve & Grenzen* wirkt zusätzlich und verschiebt die Kurve parallel, genau wie bei der Formel. So kann man die ganze Kurve kurz anheben, ohne die vier Punkte anzufassen.

Danach geht es weiter wie bei der Formel: Begrenzung auf *Vorlauf min/max*, Raumaufschaltung, Rücklaufbegrenzung, Heizgrenze und Frostschutz.

**Die Neigung** (`slope`) hat bei *4 Stützpunkte* keine Wirkung. Das Eingabefeld ist dann gesperrt; der Wert bleibt aber gespeichert und gilt wieder, sobald man auf *Formel* zurückschaltet. Umgekehrt bleiben die Punkte bei *Formel* gespeichert.

#### Beispiel

Werkspunkte (wie Neigung 1,0) und dieselben Punkte mit Punkt 3 um 2,4 K angehoben (37,6 → 40,0 °C), Raumsoll 20 °C, Niveau 0, ohne Begrenzung:

| Außen °C | Formel Neigung 1,0 | Stützpunkte ab Werk | Punkt 3 angehoben | Stützpunkte, Absenkung auf 16 °C |
|---:|---:|---:|---:|---:|
| −20 | 59,7 | 59,2 | 59,8 | 52,0 |
| −15 | 55,1 | **55,1** | **55,1** | 47,9 |
| −10 | 50,8 | 50,9 | 50,6 | 43,7 |
| −5 | 46,6 | **46,6** | **46,6** | 39,4 |
| 0 | 42,3 | 42,2 | 43,5 | 35,0 |
| +5 | 37,6 | **37,6** | **40,0** | 30,4 |
| +10 | 32,5 | 32,4 | 34,3 | 25,2 |
| +15 | 26,7 | **26,7** | **26,7** | 19,5 |
| +20 | 20,0 | 20,8 | 18,4 | 13,6 |

Fett = Stützpunkt. Man sieht: Das Anheben von Punkt 3 wirkt vor allem zwischen 0 und +10 °C; bei −15 °C und darunter bleibt alles gleich. Die letzte Spalte zeigt die Absenkung mit 1,8 K/K (−7,2 K überall).

#### Einrichten Schritt für Schritt

1. Reiter **Heizkurve** öffnen und nach unten zur Karte **Heizkurve über Stützpunkte** scrollen.
2. **Aus Formel übernehmen** klicken. Der Knopf rechnet aus der aktuellen Neigung die Vorlaufwerte an den vier eingetragenen Außentemperaturen aus (gerundet auf 0,5 °C, ohne Niveau), setzt einen passenden *Raumsoll-Einfluss* und stellt die Kurvenart auf *4 Stützpunkte*. Noch ist nichts gespeichert.
3. Bei Bedarf die Außentemperaturen der Punkte anpassen, z. B. −12 / −4 / +4 / +12, wenn es bei euch selten kälter als −12 °C wird. Wer die Außentemperaturen ändert, klickt danach noch einmal *Aus Formel übernehmen*, damit die Vorlaufwerte wieder zur Formel passen.
4. **Speichern.** Ab jetzt regelt der Regler nach den Punkten. Die Grafik oben zeigt die Punkte als kleine Ringe auf der Kurve.
5. Ein bis zwei Tage beobachten, dann gezielt einzelne Punkte nachstellen (Tabelle unten), entweder hier als Zahl oder in der Grafik per Ziehen.

Zurück zur Formel: *Kurvenart* auf *Formel (Neigung / Niveau)* stellen und speichern. Oder ein Profil laden (*Radiator* / *Fußboden* schalten auf die Formel zurück).

#### In der Grafik ziehen

Mit **Heizkurve einstellen** unter der Grafik erscheinen bei der Kurvenart *4 Stützpunkte* vier Griffe **P1 … P4**:

- **Nach oben oder unten ziehen** ändert den Vorlauf des Punkts.
- **Nach links oder rechts ziehen** ändert seine Außentemperatur. Ein Punkt bleibt dabei mindestens 2 K von seinen Nachbarn entfernt und innerhalb der Grafik (−20 … +20 °C); weiter außen liegende Punkte lassen sich als Zahl eingeben.
- Beide Werte rasten auf **0,5 °C**.
- Liegen zwei Punkte dicht beieinander, wird der Punkt genommen, der dem Finger oder Mauszeiger am nächsten liegt.
- **Ziehen an der Kurve selbst** (nicht an einem Punkt) verschiebt sie parallel und ändert nur das Niveau.
- Über jedem Punkt steht, was die Kurve dort liefert, z. B. `P2 −5°: 46,6 °C`. Die Beschriftungen stehen abwechselnd über und unter den Punkten, damit sie sich nicht überdecken. Liegt ein Wert über *Vorlauf max* oder unter *Vorlauf min*, steht dabei, worauf begrenzt wird.
- Unter der Grafik stehen alle vier Punkte bezogen auf **Raumsoll 20 °C** (das sind die Werte, die gespeichert werden), dazu das Niveau.
- Die Beschriftungen in der Grafik zeigen dagegen den Vorlauf beim **gerade wirksamen** Raumsoll mit Niveau. Ist die Absenkung aktiv oder steht das Raumsoll nicht auf 20 °C, unterscheiden sich die beiden Angaben um *Raumsoll-Einfluss × Abweichung*.
- **Übernehmen** speichert Punkte und Niveau, **Zurück auf vorher** stellt den alten Stand her, **Abbrechen** beendet ohne Änderung. Die bisherige Kurve bleibt beim Ziehen grau gestrichelt sichtbar.

#### Feinabstimmung

| Beobachtung | Abhilfe |
|---|---|
| Nur bei strengem Frost (unter −10 °C) zu kalt | Punkt 1 um 1–2 K anheben |
| Bei Frost um −5 °C zu kalt, sonst passend | Punkt 2 anheben |
| Bei mildem Wetter (0 … +10 °C) zu kalt | Punkt 3 anheben |
| In der Übergangszeit (über +10 °C) zu warm | Punkt 4 senken oder Heizgrenze senken |
| Bei jedem Wetter gleich zu kalt / zu warm | Niveau ±1 K (alle Punkte bleiben) |
| Absenkung wirkt zu stark / zu schwach | *Raumsoll-Einfluss* senken / erhöhen |

Wie bei der Formel gilt: kleine Schritte (1–2 K), nur einen Punkt auf einmal, und jeweils ein bis zwei Tage abwarten. Ein Fußboden reagiert noch langsamer.

#### Zusammenspiel mit anderen Funktionen

**Profile:** Die Kurvenart, die vier Punkte und der Raumsoll-Einfluss gehören zu den Profilwerten und stehen in der Vergleichstabelle. *Radiatorheizung* und *Fußbodenheizung* setzen die Kurvenart auf *Formel* und die Punkte auf ihre jeweilige Formelkurve:

| | Radiatorheizung | Fußbodenheizung |
|---|---|---|
| Kurvenart | Formel | Formel |
| Punkte Außen °C | −15 / −5 / +5 / +15 | −15 / −5 / +5 / +15 |
| Punkte Vorlauf °C | 55,1 / 46,6 / 37,6 / 26,7 | 32,3 / 29,3 / 26,2 / 22,3 |
| Raumsoll-Einfluss | 1,8 K/K | 1,3 K/K |

Wer nach dem Laden eines Profils auf *4 Stützpunkte* umschaltet, bekommt also die Profilkurve als Startpunkt. Das Profil **GLW8** speichert Kurvenart, Punkte und Raumsoll-Einfluss mit. Ein GLW8-Profil, das mit einer älteren Version gespeichert wurde, enthält diese Werte noch nicht; beim Laden bleiben sie dann unverändert. Jede Änderung an Kurvenart oder Punkten setzt das Profil auf „angepasst“.

**Weitere Funktionen:**

- **Kühlen:** Die Stützpunkte betreffen nur das Heizen. Beim Kühlen gilt weiter die Kühlkurve mit den Punkten A und B.
- **Fester Sollwert, Handbetrieb, Aus:** Dort wird keine Heizkurve verwendet.
- **CSV-Liste der Heizkurve:** Sie rechnet mit derselben Funktion wie die Regelung. Im Kopf stehen die Kurvenart und bei *4 Stützpunkte* die vier Punkte und der Raumsoll-Einfluss statt der Neigung.
- **Ereignisprotokoll:** Jede Änderung wird mit dem Namen des Felds sowie altem und neuem Wert eingetragen, z. B. *Web: Punkt 3: Vorlauf °C* mit beiden Werten.
- **Werkseinstellungen:** setzen auch die Kurvenart auf *Formel* und die Punkte auf die Werte oben zurück.

#### Über MQTT, REST-API und Home Assistant

Die neuen Werte lassen sich wie alle anderen Parameter setzen:

```
heizung/hk1/set/curveMode   1
heizung/hk1/set/cpFlow3     40
heizung/hk1/set             {"curveMode":1,"cpOut1":-12,"cpFlow1":52,"cpOut2":-4,"cpFlow2":45,"cpOut3":4,"cpFlow3":39,"cpOut4":12,"cpFlow4":30,"cpRoomK":1.8}
```

```
curl -u admin:PASSWORT -X POST http://heizungsregler.local/api/config \
     -H "Content-Type: application/json" -d '{"curveMode":1,"cpFlow3":40}'
```

`GET /api/config` liefert alle zehn Werte. Wer mehrere Außentemperaturen ändert, schickt sie am besten **zusammen** in einem JSON-Objekt: Die Regel „aufsteigend, mindestens 2 K Abstand“ wird nach jeder Nachricht geprüft, und ein einzeln gesendeter Wert kann sonst einen Nachbarpunkt verschieben.

Für Home Assistant und Modbus gibt es bewusst **keine eigenen Entitäten bzw. Register** für die Punkte; die Kurve wird einmal eingerichtet und dann selten geändert. Über das JSON-Topic `…/set` lassen sie sich aber auch aus Home Assistant heraus setzen (z. B. mit `mqtt.publish`).

### Heizkurvengrafik

Die Grafik steht ganz oben im Reiter *Heizkurve*, direkt über den Einstellungen. Sie zeigt die Kurve über der Außentemperatur von −20 °C bis +20 °C, dazu den aktuellen Arbeitspunkt (Sollwert orange, gemessener Vorlauf blau) und die Heizgrenze. Waagrechte Rasterlinien liegen alle 5 °C, beschriftet alle 10 °C.

**Stützpunkte:** Bei der Kurvenart *4 Stützpunkte* sind die vier Punkte in der Grafik markiert: ein Ring auf der Kurve, darüber der Name **P1 … P4** und eine dünne gestrichelte Linie hinunter zur Außentemperatur-Achse. So sieht man auf einen Blick, wo die Kurve festgelegt ist und wo sie dazwischen gerechnet wird. Der Ring sitzt auf dem gerade gültigen Vorlaufwert, also mit wirksamem Raumsoll, Niveau und begrenzt auf *Vorlauf min/max*. Fährt man mit Maus oder Finger in die Nähe eines Punkts (±0,75 °C), springt die Werteanzeige genau auf ihn und nennt ihn, z. B. „Stützpunkt P2 · Außen −5,0 °C“. Liegt ein Punkt außerhalb von −20 … +20 °C, wird er nicht gezeichnet. Bei der Kurvenart *Formel* gibt es keine Markierungen.

**Skala einstellbar:** Rechts über der Grafik wählt man mit *Skala bis* den Höchstwert der Vorlaufachse: **40, 50, 60, 70 oder 80 °C** (Werkseinstellung 60 °C). Für eine Fußbodenheizung passt meist 40 oder 50 °C, für Heizkörper 60 bis 80 °C. Die Auswahl wird sofort im Regler gespeichert (Einstellung `chartMax`). Sie gilt damit auf allen Geräten und bleibt nach einem Neustart erhalten. Sie betrifft nur die Anzeige, nicht die Regelung, und ändert das Heizungsprofil nicht. Über MQTT oder die REST-API lässt sich jeder Wert von 40 bis 80 setzen, z. B. `…/set/chartMax 55`.

Liegt ein Wert über dem gewählten Höchstwert, läuft die Kurve am oberen Rand entlang; die Werteanzeige nennt trotzdem den echten Wert. Beim Einstellen per Ziehen lassen sich die Griffpunkte bis zum gewählten Höchstwert ziehen, mit *Skala bis 80°* also bis 80 °C.

### Heizkurve mit Maus oder Finger einstellen

Im Reiter *Heizkurve* lässt sich die Kurve direkt in der Grafik einstellen, statt Neigung und Niveau als Zahlen einzugeben:

1. Unter der Grafik auf **Heizkurve einstellen** klicken. Die Grafik bekommt einen orangen Rahmen, und auf der Kurve erscheinen zwei runde Griffpunkte bei **−15 °C** und **+10 °C** Außentemperatur. Daneben steht jeweils der Vorlauf-Sollwert.
2. Einen Griffpunkt mit Maus oder Finger **nach oben oder unten ziehen**. Der andere Punkt bleibt dabei stehen. So stellt man z. B. „bei −15 °C 55 °C, bei +10 °C 32 °C“ direkt ein. Neigung und Niveau rechnet der Regler daraus aus.
3. Die Kurve **an einer anderen Stelle ziehen** verschiebt sie parallel nach oben oder unten; dabei ändert sich nur das Niveau.
4. Zum Vergleich bleibt die bisherige Kurve grau gestrichelt sichtbar. Unter der Grafik stehen die neuen Werte für Neigung und Niveau, daneben die bisherigen.
5. **Übernehmen** speichert die neue Kurve. **Zurück auf vorher** stellt die alten Werte wieder her, **Abbrechen** beendet das Einstellen ohne Änderung.

Gerechnet wird mit derselben Formel wie in der Regelung. Neigung wird auf 0,01, Niveau auf 0,1 K gerundet und auf die erlaubten Bereiche begrenzt (Neigung 0,1–4,0, Niveau −15 bis +15 K). Liegt ein Griffpunkt über *Vorlauf max* oder unter *Vorlauf min*, zeigt die Beschriftung, auf welchen Wert dort begrenzt wird. Die Kurve gilt für den aktuell wirksamen Raumsollwert, bei aktiver Absenkung also für den abgesenkten.

**Bei der Kurvenart „4 Stützpunkte“** liegen statt der zwei Griffe vier frei ziehbare Griffe P1 … P4 auf den Stützpunkten; Einzelheiten unter [In der Grafik ziehen](#in-der-grafik-ziehen).

Damit beim Ziehen mit dem Finger die Seite nicht scrollt, nimmt die Grafik nur im Einstellmodus alle Fingerbewegungen auf; sonst lässt sich die Seite wie gewohnt über die Grafik hinweg scrollen. Eine versehentliche Berührung verstellt nichts, weil erst *Übernehmen* speichert. Wie bei jeder Änderung von Neigung oder Niveau steht das Heizungsprofil danach auf „angepasst“.

### Heizkurve als Liste herunterladen

*Heizkurve als Liste herunterladen (CSV)* findest du im Reiter *Heizkurve* direkt unter der Grafik und zusätzlich in der Karte *Heizkurve exportieren* am Ende des Reiters.

**Erzeugt im Regler:** Die Liste entsteht **beim Abruf im Regler selbst** (`/api/curve.csv`).
- **Berechnung:** dieselbe Funktion `heatingCurve()`, mit der der Regler regelt.
- **Werte:** die Parameter, die in diesem Moment wirksam sind.
- **Folge:** Liste und Regelung stimmen immer exakt überein.
- **Dateiname:** enthält Datum und Uhrzeit, z. B. `Heizkurve_2026-10-16_2318.csv`.

**Kopfzeilen:**
- **Einstellungen:** Profil, Raumsollwert Komfort/Absenkung, Kurvenart, Neigung bzw. die vier Stützpunkte mit Raumsoll-Einfluss, Niveau, Vorlauf min/max, Heizgrenze, Raumaufschaltung, Rücklaufbegrenzung.
- **Zustand jetzt:** Betriebsart, Absenkbetrieb aktiv/nicht aktiv, Außentemperatur gemessen und gedämpft, Raumtemperatur und Raumkorrektur, Rücklauf und Rücklaufbegrenzung, Heizbetrieb, Vorlauf-Soll und Vorlauf-Ist.

**Tabelle** für Außentemperaturen von −25 °C bis +25 °C in 1-K-Schritten:

| Spalte | Inhalt |
|---|---|
| Außentemperatur °C | −25 … +25 |
| Vorlauf-Soll Komfort °C | Heizkurve mit Raumsollwert, begrenzt auf Vorlauf min/max |
| Vorlauf-Soll Absenkung °C | Heizkurve mit Raumsollwert − Absenkung, begrenzt |
| Vorlauf-Soll jetzt °C | wie im Regelzyklus: Kurve mit dem gerade gültigen Raumsollwert (Komfort oder Absenkung), begrenzt, plus aktuelle Raumkorrektur (nur in *Automatik*), minus aktuelle Rücklaufbegrenzung, erneut begrenzt |
| Kurve ohne Begrenzung (Komfort) °C | zeigt, wo die Min/Max-Grenzen eingreifen |
| Heizbetrieb | `ja` / `nein (Heizgrenze)`; im Hysteresebereich ±0,5 K der aktuelle Zustand |
| Markierung | `<- aktuelle Außentemperatur` in der Zeile der gerundeten gedämpften Außentemperatur |

**Momentaufnahme:** Die Spalte *jetzt* gibt den Zustand beim Abruf wieder. Raumkorrektur und Rücklaufbegrenzung ändern sich mit den Messwerten, deshalb liefert ein späterer Abruf andere Werte.

**Format:** Semikolon als Trennzeichen, Komma als Dezimalzeichen, UTF-8 mit BOM. Die Datei öffnet sich in einem deutschen Excel oder LibreOffice direkt richtig und eignet sich gut, um zwei Einstellungen zu vergleichen oder die Kurve auszudrucken.

---

## Heizungsprofile

Oben im Reiter *Heizkurve* lassen sich zwei vorbereitete Parametersätze laden. Eine Tabelle vergleicht die aktuellen Werte mit beiden Profilen, übereinstimmende Werte sind grün markiert.

**Erklärungen in der Tabelle:** Die Parameternamen sind gepunktet unterstrichen. Fährt man mit der Maus darüber, tippt mit dem Finger darauf oder steuert sie mit der Tab-Taste an, erscheint ein kleines Kästchen mit einer kurzen Erklärung, z. B. bei *Kp*: „Verstärkung des Reglers: Stelländerung in % je Kelvin Abweichung. Größer = schneller, aber Neigung zum Pendeln.“ Auf dem Smartphone bleibt das Kästchen stehen, bis man woanders hintippt. Dieselben Erklärungen gibt es an den Eingabefeldern der Karten *Heizkurve & Grenzen* und *Raumtemperatur* sowie im Reiter *Regler*, dort zusätzlich für *Rücklauf max*, *Rücklaufbegrenzung*, *Raumtemperatur berücksichtigen*, *Raumabschaltung*, *Stellantrieb* und *Relais Low-aktiv*. Ein Klick auf den Namen zeigt nur die Erklärung und schaltet kein Häkchen um.

| Parameter | Radiatorheizung | Fußbodenheizung |
|---|---|---|
| Vorlauf bei −10 / 0 / +10 °C außen (Raum 20 °C) | ≈ 51 / 42 / 32 °C | ≈ 31 / 28 / 24 °C |
| Neigung / Niveau | 1,0 / 0 K | 0,35 / 0 K |
| Kurvenart | Formel | Formel |
| Stützpunkte Vorlauf bei −15 / −5 / +5 / +15 °C | 55,1 / 46,6 / 37,6 / 26,7 °C | 32,3 / 29,3 / 26,2 / 22,3 °C |
| Raumsoll-Einfluss (Stützpunkte) | 1,8 K/K | 1,3 K/K |
| Vorlauf min / max | 25 / 60 °C | 20 / 40 °C |
| Heizgrenze / Frostschutz | 17 / 3 °C | 16 / 3 °C |
| Fester Vorlauf-Sollwert | 45 °C | 30 °C |
| Dämpfung Außentemperatur | 45 min | 180 min |
| Absenkung | 4 K | 1,5 K |
| Raumaufschaltung / max. Korrektur | 3 K/K / 10 K | 1,5 K/K / 4 K |
| Raumregler: Verstärkung / Nachstellzeit / Heizpause ab | 12 K/K / 60 min / 0,5 K | 6 K/K / 360 min / 1,0 K |
| Kp / Tn | 3 %/K / 150 s | 2 %/K / 300 s |
| Totzone / Abtastzeit / Mindestimpuls | 0,5 K / 10 s / 400 ms | 0,5 K / 20 s / 500 ms |
| Motorlaufzeit (nur wenn angekreuzt) | 120 s | 140 s |

**Warum die Werte unterschiedlich sind:**
- **Heizkörper** reagieren schnell und brauchen höhere Temperaturen. Deshalb gibt es eine steilere Kurve, einen zügigeren Regler und eine Absenkung, die sich lohnt.
- **Fußbodenheizungen** speichern viel Wärme im Estrich. Deshalb gibt es eine flache Kurve, einen langsamen Regler, eine stark gedämpfte Außentemperatur und eine kleine Absenkung.
- **Estrichschutz:** Bei Fußbodenheizung begrenzt *Vorlauf max* 40 °C den Vorlauf, der Übertemperaturschutz greift ab 45 °C. Ein Hardware-STB ist trotzdem Pflicht.

**Was ein Profil nicht ändert:**
- Raumsollwert, Betriebsart, Zeitprogramm, Fühler, Rücklaufbegrenzung und Netzwerk bleiben erhalten.
- Die **Motorlaufzeit** hängt vom eingebauten Stellantrieb ab. Sie wird nur übernommen, wenn das Kästchen angekreuzt ist.

**Nach dem Laden:** Ein Profil ist ein Startpunkt. Ist es zu kalt oder zu warm, passt du Neigung oder Niveau an oder schaltest auf [4 Stützpunkte](#heizkurve-über-4-stützpunkte) um. Sobald ein Profilwert von Hand geändert wird, zeigt der Regler „angepasst“ an. Ab Werk ist das Profil Radiatorheizung geladen.

### Eigenes Profil „GLW8“

Neben den beiden festen Profilen gibt es das eigene Profil **GLW8**. Es nimmt die Werte auf, die du für deine Anlage gefunden hast:

- **Aktuelle Werte als GLW8 speichern** übernimmt alle Profilparameter, die gerade eingestellt sind (Spalte *Aktuell* der Tabelle), dazu die Motorlaufzeit. Ist bereits ein GLW8-Profil gespeichert, fragt die Seite vor dem Überschreiben nach. Danach zeigt der Regler „Aktuell: GLW8“.
- **Profil GLW8 laden** stellt diese Werte wieder her, genau wie bei den anderen Profilen. Die Motorlaufzeit wird nur übernommen, wenn *Motorlaufzeit ebenfalls übernehmen* angekreuzt ist. Solange noch nichts gespeichert wurde, ist der Knopf ausgegraut.
- In der Vergleichstabelle zeigt die Spalte **GLW8** die gespeicherten Werte; Übereinstimmungen mit *Aktuell* sind grün. Unter den Knöpfen steht, wann GLW8 gespeichert wurde.

**Speicherung:** Die Werte stehen in der Datei `/profile_glw8.json` im Flash, getrennt von den Einstellungen. Sie bleiben bei Neustart, Firmware-Update und auch bei **Werkseinstellungen** erhalten. So lassen sich die eigenen Regelwerte nach einem Zurücksetzen mit einem Klick wiederherstellen.

**Typischer Ablauf:** Mit Radiator- oder Fußboden-Profil starten, Heizkurve und Regler ein paar Tage feinjustieren, dann *Aktuelle Werte als GLW8 speichern*. Wer später etwas ausprobiert, kommt mit *Profil GLW8 laden* jederzeit zum bewährten Stand zurück.

**Profil laden über andere Wege** (die Motorlaufzeit bleibt dabei immer unverändert):
- **MQTT:** `heizung/hk1/set/profile` mit `radiator`, `floor` oder `glw8`; `heizung/hk1/set/profileSave` (Nutzlast beliebig) speichert die aktuellen Werte als GLW8
- **Modbus:** Kommando-Register 14 mit Wert `4` (Radiator), `5` (Fußboden), `6` (GLW8 laden) oder `7` (aktuelle Werte als GLW8 speichern)
- **Home Assistant:** Auswahl „Heizungsprofil“ (`radiator` / `floor` / `glw8`, Anzeige `custom` wenn angepasst) und Taste „Aktuelle Werte als Profil GLW8 speichern“

---

## Zeitprogramm

Im Reiter *Zeitprogramm* schaltet der Regler mit „Absenkbetrieb aktiv“ zeitgesteuert um:
- **Komfort:** Es gilt der Raumsollwert.
- **Absenkung:** Es gilt der Raumsollwert minus die eingestellte Absenkung.

| Programmart | Verhalten |
|---|---|
| Tägliches Absenkfenster | Jeden Tag dasselbe Fenster, z. B. 22:00–06:00 Absenkung |
| Wochenprogramm | Je Wochentag bis zu drei Heizzeiten (`HH:MM-HH:MM`); innerhalb Komfort, sonst Absenkung. Zusätzlich eine Zeile **Feiertag** |

Werkseinstellung des Wochenprogramms:

| Tage | Heizzeiten |
|---|---|
| Mo–Fr | 06:00–08:00 und 16:00–22:00 |
| Sa–So | 07:00–22:00 |
| Feiertag | 07:00–22:00 |

**Regeln für die Heizzeiten:**
- **Keine Heizzeit über Mitternacht:** Das Ende muss nach dem Beginn liegen. Für eine Heizzeit über Mitternacht am Folgetag eine weitere ab 00:00 anlegen.
- **Ende 24:00** ist erlaubt.
- **Kopieren:** Schaltflächen übertragen Montag auf Di–Fr, Montag auf Di–So, Samstag auf Sonntag und Sonntag auf Feiertag. Kopiert wird nur in die Tabelle; gespeichert wird mit *Wochenprogramm speichern*.
- **Uhrzeit:** Sie kommt per NTP aus dem Internet (Zeitzone Mitteleuropa mit Sommerzeit). Ohne gültige Uhrzeit wird nicht abgesenkt.

Die Übersicht zeigt den aktuellen Stand, z. B. „Komfort bis 22:00“ oder „Absenkung bis 06:00“, an Feiertagen mit dem Zusatz „· Feiertag“.

### Feiertage

An Feiertagen gelten im Wochenprogramm die Heizzeiten der Zeile **Feiertag** statt der Zeile des Wochentags. Fällt ein Feiertag auf einen Dienstag, heizt der Regler also nach der Feiertagszeile, nicht nach dem Dienstag.

**Gesetzliche Feiertage Österreich** (Kästchen unter der Tabelle, ab Werk ein): Der Regler erkennt sie selbst und braucht dafür kein Internet außer für die Uhrzeit.

| Art | Feiertage |
|---|---|
| fest | Neujahr (1.1.), Heilige Drei Könige (6.1.), Staatsfeiertag (1.5.), Mariä Himmelfahrt (15.8.), Nationalfeiertag (26.10.), Allerheiligen (1.11.), Mariä Empfängnis (8.12.), Christtag (25.12.), Stefanitag (26.12.) |
| beweglich | Ostermontag, Christi Himmelfahrt, Pfingstmontag, Fronleichnam – jedes Jahr aus dem Osterdatum berechnet (Gaußsche Osterformel) |

**Zusätzliche freie Tage:** Im Feld darunter trägst du weitere Tage ein, mit Komma getrennt:
- `TT.MM` gilt jedes Jahr, z. B. `24.12, 31.12` (Heiliger Abend, Silvester).
- `TT.MM.JJJJ` gilt nur einmal, z. B. `2.1.2027` für einen Urlaubs- oder Fenstertag.
- Ungültige Einträge werden rot markiert und nicht gespeichert.

Unter den Feldern steht, ob heute ein Feiertag ist und welcher als Nächstes kommt, z. B. „Nächster Feiertag: 26.10.2026 Nationalfeiertag“. Ist heute ein Feiertag, erscheint in der Übersicht zusätzlich die Statusanzeige „Feiertag: …“.

**Hinweise:**
- Feiertage wirken nur bei der Programmart *Wochenprogramm*, nicht beim täglichen Absenkfenster.
- Ohne gültige Uhrzeit (NTP) erkennt der Regler keine Feiertage.
- Regionale Feiertage einzelner Bundesländer (z. B. Landespatron) sind nicht enthalten; sie lassen sich als freier Tag `TT.MM` eintragen.
- Über MQTT oder die REST-API: `holidayAT` (`true`/`false`), `holidayExtra` (Text) und `week` mit 8 Zeilen (die 8. ist der Feiertag). Ältere Einstellungen mit 7 Zeilen bleiben gültig; die Feiertagszeile hat dann die Werkseinstellung 07:00–22:00.

Per MQTT oder Web-API lässt sich das Wochenprogramm als JSON setzen. Montag kommt zuerst, die 8. Zeile ist der Feiertag (darf fehlen), ein leerer Text bedeutet „Heizzeit frei“:

```
heizung/hk1/set   {"schedMode":1,"week":[["06:00-08:00","16:00-22:00",""],["06:00-08:00","16:00-22:00",""],["06:00-08:00","16:00-22:00",""],["06:00-08:00","16:00-22:00",""],["06:00-08:00","16:00-22:00",""],["07:00-22:00","",""],["07:00-22:00","",""],["07:00-22:00","",""]],"holidayAT":true,"holidayExtra":"24.12, 31.12"}
```

---

## Temperaturfühler

### Anschlussart

Im Reiter *Fühler* wählst du unter **Anschlussart der Fühler**, wie die DS18B20 angeschlossen sind. Die Umschaltung wirkt sofort, ein Neustart ist nicht nötig.

| Anschlussart | Verdrahtung | Zuordnung |
|---|---|---|
| **Eigene Pins je Fühler** (Werkseinstellung) | Vorlauf GPIO 4, Rücklauf GPIO 13, Außen GPIO 14, Raum GPIO 27; je ein 4,7-kΩ-Pull-up | nicht nötig – es zählt der Pin; hängen mehrere an einem Pin, wird der erste verwendet und das angezeigt |
| **Gemeinsamer Bus an GPIO 4** | alle DS18B20 parallel an GPIO 4; ein 4,7-kΩ-Pull-up | über die Seriennummer des DS18B20 |

**Vorteile:**
- **Eigene Pins:** Jeder Fühler ist eindeutig über den Pin bestimmt. Ein Kurzschluss oder defekter Fühler stört die anderen nicht.
- **Gemeinsamer Bus:** weniger Verdrahtung und nur ein Widerstand. GPIO 13, 14 und 27 bleiben frei.

**Zuordnung am gemeinsamen Bus:**
- **Liste:** Bei gemeinsamem Bus erscheint im Reiter *Fühler* eine Liste aller an GPIO 4 gefundenen DS18B20 mit Seriennummer und aktueller Temperatur.
- **Zuordnen:** Über die Auswahl ordnest du jeden Fühler einer Rolle zu (Vorlauf, Rücklauf, Außen, Raum) oder markierst ihn als „nicht verwendet“. Danach *Zuordnung speichern*.
- **Erkennen:** Einen Fühler kurz in der Hand erwärmen, seine Temperatur steigt in der Liste.
- **Automatik:** Rollen ohne Zuordnung werden beim Suchen automatisch in Fundreihenfolge belegt. Ab Werk ist das also schon eine Grundzuordnung, die du nur prüfen musst.
- **Doppelbelegung:** Jede Rolle kann nur einem Fühler zugeordnet werden, die Weboberfläche verhindert Doppelbelegungen.
- **Fühlertausch:** Wird ein Fühler getauscht, hat der neue eine andere Seriennummer. Dann die Zuordnung neu speichern oder die Rolle leeren, damit sie automatisch belegt wird.

### Herkunft je Fühler

Im Reiter *Fühler* stellst du für jeden Fühler einzeln ein, woher der Wert kommt:

| Herkunft | Verhalten |
|---|---|
| Fühler am Pin | nur der DS18B20 am Pin (Werkseinstellung) |
| MQTT / Modbus | nur der externe Wert; der Pin bleibt unbenutzt |
| Pin, bei Ausfall MQTT | der DS18B20 hat Vorrang; fehlt oder versagt er, wird automatisch der externe Wert verwendet |
| Bluetooth (RuuviTag / Shelly BLU) | nur Außen und Raum: nur der Bluetooth-Sensor (siehe [Bluetooth-Sensoren](#bluetooth-sensoren-ruuvitag-shelly-blu-ht)) |
| Bluetooth, bei Ausfall MQTT | nur Außen und Raum: der Bluetooth-Sensor hat Vorrang, sonst der MQTT-/Modbus-Wert |
| Pin, bei Ausfall Bluetooth | nur Außen und Raum: der DS18B20 hat Vorrang, sonst der Bluetooth-Sensor |

Oben im Reiter zeigt eine Tabelle je Fühler den Zustand:
- ob ein DS18B20 gefunden bzw. zugeordnet wurde
- den Pin-Wert
- den MQTT-Wert und wie alt er ist
- welcher Wert gerade verwendet wird (grün)

### Bluetooth-Sensoren (RuuviTag, Shelly BLU H&T)

Der Regler empfängt Funk-Thermometer direkt über das Bluetooth des ESP32 – ohne Home Assistant, ohne Gateway und ohne Kopplung; er hört nur zu (passiver Empfang). MQTT bleibt daneben voll nutzbar. Zwei Sensorarten werden erkannt, beide für **Außen** und **Raum**:

| | RuuviTag (auch Pro) – seit 2.1.0 | Shelly BLU H&T (SBHT-003C) – seit 2.6.0 |
|---|---|---|
| Funkformat | Ruuvi-Herstellerdaten, Formate 5, C5, 3 | BTHome v2 (offener Standard), auch verschlüsselt |
| sendet | etwa jede Sekunde | etwa jede Minute und bei Druck auf die Taste |
| Werte | Temperatur 0,005 °C, Feuchte, Luftdruck, Batterie in V | Temperatur 0,1 °C, Feuchte 1 %, Batterie in % |
| Batteriewarnung | unter 2,5 V | unter 15 % |

**Einrichten:**
1. Im Reiter *Fühler*, Karte **Bluetooth-Sensoren (RuuviTag, Shelly BLU H&T)**, auf *Bluetooth-Suche starten (10 min)* klicken. Es erscheinen alle empfangenen Sensoren mit MAC-Adresse, Art, Temperatur, Feuchte, Luftdruck, Batterie und Empfangsstärke, der stärkste oben. Ein RuuviTag erscheint nach wenigen Sekunden, ein **Shelly erst nach bis zu einer Minute** – schneller geht es mit einem Druck auf seine Taste.
2. Den richtigen Sensor erkennen: kurz in der Hand erwärmen – seine Temperatur steigt (beim Shelly mit bis zu einer Minute Verzögerung).
3. **für Außen** bzw. **für Raum** klicken. Die MAC-Adresse wird beim Fühler eingetragen und die Herkunft auf *Bluetooth (RuuviTag / Shelly BLU)* gestellt (eine schon gewählte Bluetooth-Variante bleibt). Die MAC-Adresse lässt sich auch von Hand im Formular *Herkunft und Korrektur je Fühler* eintragen.
4. Soll ein Ersatz einspringen, unter *Herkunft* „Bluetooth, bei Ausfall MQTT“ oder „Pin, bei Ausfall Bluetooth“ wählen.

**Shelly mit Verschlüsselung:** In der Shelly-App lässt sich für den BLU H&T eine Verschlüsselung einschalten; die App zeigt dann einen Schlüssel aus 32 Hex-Zeichen. Ein solcher Shelly erscheint in der Liste mit „verschlüsselt – Schlüssel beim Fühler eintragen“. So geht es weiter:
1. Den Sensor wie oben mit *für Außen* / *für Raum* zuordnen (die MAC-Adresse steht auch verschlüsselt lesbar in der Liste).
2. Im Formular *Herkunft und Korrektur je Fühler* den Schlüssel bei **Außen – Shelly-Schlüssel** bzw. **Raum – Shelly-Schlüssel** eintragen und speichern.
3. Ab dem nächsten Paket zeigt die Liste die Werte („Shelly BLU, verschlüsselt“). Passt der Schlüssel nicht, steht dort „Schlüssel passt nicht“ und im Ereignisprotokoll eine Meldung.

Der Schlüssel wird wie ein Passwort behandelt: Er wird nur als `********` angezeigt, nicht ins Ereignisprotokoll geschrieben, nicht mit *Einstellungen herunterladen* gesichert und lässt sich nur im Web ändern. Entschlüsselt wird nach dem BTHome-Verfahren (AES-128-CCM); geprüft mit dem Rechenbeispiel von bthome.io. Ohne Verschlüsselung (Werkseinstellung des Shelly) bleibt das Feld leer.

**Im Betrieb:**
- Der Empfang läuft automatisch, sobald Außen oder Raum einen Bluetooth-Sensor verwenden. Er hört seit 2.6.0 alle 160 ms für **80 ms** zu (vorher 40 ms) – der Shelly sendet nur selten, ein kürzeres Fenster würde zu viele seiner Pakete verpassen. Das WLAN läuft dabei weiter flüssig.
- Die Werte werden alle 2 s mit den DS18B20 übernommen. Korrekturwert und **Zeitüberwachung** gelten wie bei MQTT (Außen/Raum einstellbar, Standard 30 min). Für den Shelly nicht unter 10 min einstellen: Er sendet nur einmal pro Minute, und einzelne Pakete gehen im Funk verloren. Meldet sich der Sensor nicht mehr, steht das im Ereignisprotokoll und – bei den Varianten mit Ersatz – springt dieser ein.
- **Raumfeuchte:** Ist beim Raum ein Bluetooth-Sensor gewählt, liefert er auch die Raumfeuchte für den Taupunktschutz beim Kühlen und für einen Entfeuchter an Relais K4.
- **Batterie:** Unter der Grenze (2,5 V bzw. 15 %) steht einmalig eine Warnung im Ereignisprotokoll; in der Liste wird der Wert rot.
- **Datenformate:** RuuviTag 5 (RAWv2), C5 und 3 (RAWv1), geprüft mit den Testdaten von docs.ruuvi.com. BTHome v2 nach bthome.io: ausgewertet werden Temperatur (0x45 und 0x02), Feuchte (0x2E und 0x03), Luftdruck (0x04), Batterie (0x01) und Spannung (0x0C) – damit funktionieren auch andere BTHome-Thermometer, nicht nur der Shelly. Pakete ohne Temperatur (z. B. nur Tastendruck) werden übergangen.

**Reichweite und Montage:** Durch eine Außenwand typisch 5–20 m, Stahlbeton und Metallfassaden dämpfen stark. Die Empfangsstärke in der Liste zeigt es: ab −70 dBm gut, bis −85 dBm mäßig, darunter schwach. Außen wie jeden Außenfühler an der Nord- oder Nordwestseite im Schatten montieren, nicht über Fenstern oder Abluft. Der Shelly BLU H&T hat kein wetterfestes Gehäuse – außen nur geschützt montieren (Wetterschutzgehäuse, ohne direkte Sonne und Regen); bei Frost lässt die Batterie nach.

**Ressourcen:** Bluetooth braucht die Bibliothek **NimBLE-Arduino** (h2zero, 1.4.x oder ≥ 2.0) und belegt zusätzlich etwa 250–300 KB Programmspeicher und 40–50 KB Arbeitsspeicher – mit der Partition *Minimal SPIFFS* kein Problem. Gestartet wird es erst, wenn es gebraucht wird. Für das Entschlüsseln wird die mbedTLS-Bibliothek des ESP32-Cores verwendet (keine Installation nötig).

### Externe Werte einspeisen

Es gibt drei Wege, die sich auch kombinieren lassen:

- **Eigenes MQTT-Topic (empfohlen):** Trage beim Fühler unter *MQTT-Topic* das Topic des Geräts ein, z. B. `zigbee2mqtt/Wohnzimmer`, `shellies/ht-aussen/sensor/temperature` oder `tele/heizung/SENSOR` (Tasmota). Der Regler abonniert es selbst, eine Automation in Home Assistant oder Node-RED ist nicht nötig.
  - Eine **reine Zahl** als Nutzlast (`21.4`) wird direkt übernommen.
  - Bei **JSON** (`{"temperature":21.4,"humidity":48}`) steht unter *JSON-Feld* der Feldname, standardmäßig `temperature`. Verschachtelte Felder werden mit Punkt getrennt, z. B. `DS18B20.Temperature` bei Tasmota.
  - Dasselbe Topic darf bei mehreren Fühlern eingetragen sein, z. B. mit verschiedenen JSON-Feldern.
  - Platzhalter (`+`, `#`) sind nicht erlaubt.
- **MQTT-Befehl:** `heizung/hk1/set/flowTemp`, `…/returnTemp`, `…/outTemp` oder `…/roomTemp`, jeweils mit dem Wert als Nutzlast.
- **Modbus:** Holding-Register 19 (Vorlauf), 20 (Rücklauf), 21 (Außen) und 17 (Raum), jeweils °C × 10.

Externe Werte werden nur verwendet, wenn die Herkunft des Fühlers „MQTT / Modbus“ oder „Pin, bei Ausfall MQTT“ ist.

### Zeitüberwachung

Ein externer Wert muss regelmäßig neu kommen. Sonst wird er verworfen. Der Fühler gilt dann als gestört, bei „Pin, bei Ausfall MQTT“ wird der Pin verwendet.

| Fühler | Zeitüberwachung |
|---|---|
| Vorlauf, Rücklauf | fest 2 Minuten |
| Außen, Raum | einstellbar, Standard 30 Minuten |

**Vorlauf über MQTT** ist nur sinnvoll, wenn der Wert mindestens alle 10 Sekunden kommt. Der Regler arbeitet im 10–20-Sekunden-Takt und braucht aktuelle Werte, sonst schwingt der Vorlauf. Fällt der Vorlaufwert aus, fährt der Mischer zu. Für den Übertemperaturschutz ist ein Fühler direkt am Pin die sicherere Wahl.

### Korrektur

Je Fühler lässt sich ein Korrekturwert von −5 bis +5 K einstellen. Er gilt für Pin- und externe Werte.

---

## Raumtemperatur

Der Raumfühler ist **optional**. Ist an GPIO 27 kein DS18B20 angeschlossen und kommt kein externer Wert, arbeitet der Regler rein witterungsgeführt. Das gilt nicht als Störung.

Die folgenden Einstellungen stehen im Reiter *Heizkurve*, Karte *Raumtemperatur*:

- **Raumtemperatur berücksichtigen:** legt fest, ob der Raumwert die Regelung beeinflusst. Ist der Schalter aus, wird der Raumwert nur angezeigt und aufgezeichnet. Bedienbar über Web, MQTT (`…/set/roomUse` mit `ON`/`OFF`), Modbus (Holding-Register 18) und Home Assistant.
- **Raumaufschaltung:** Der Vorlauf-Sollwert der Heizkurve wird um *Faktor × (Raum-Soll − Raum-Ist)* korrigiert, begrenzt auf ± *max. Vorlaufkorrektur*. Mit Faktor 3 steigt der Vorlauf um 3 K, wenn der Raum 1 K zu kalt ist. So werden Fremdwärme (Sonne, Kamin) und eine nicht ganz passende Heizkurve ausgeglichen. Richtwerte: Heizkörper 2–4, Fußbodenheizung 1–2. Bei 0 wird der Raumwert nur angezeigt.
- **Raumabschaltung:** Liegt der Raum um den eingestellten Wert über dem Sollwert, schaltet der Heizkreis ab (Mischer zu, Pumpe aus). Unter dieser Schwelle minus 0,5 K schaltet er wieder ein. 0 schaltet die Funktion aus. Der Frostschutz bleibt aktiv.

**Hinweise:**
- **Gültigkeit:** Raumaufschaltung und Raumabschaltung wirken in den Betriebsarten *Automatik*, *Kühlen* und *Heizen/Kühlen*. In der Betriebsart *Raumtemperatur* regelt statt dessen der Raumregler (Karte *Raumregelung*), dort wird der Raumwert immer verwendet. Beim Heizen berücksichtigt der Raumsollwert den Absenkbetrieb, beim Kühlen gilt der Raumsollwert Kühlen (siehe [Kühlen](#kühlen)).
- **Referenzraum:** Der Fühler gehört in einen Raum ohne Fremdwärme. Thermostatventile dort ganz öffnen, damit sie nicht gegen den Regler arbeiten.

---

## Kühlen

Mit den Betriebsarten **Kühlen** und **Heizen/Kühlen automatisch** kühlt der Regler über denselben Heizkreis, typisch eine Fußbodenheizung an einer reversiblen Wärmepumpe.

**Voraussetzung:** Die Wärmepumpe (oder eine andere Kältequelle) liefert im Kühlbetrieb selbst kaltes Wasser an den Mischer. Der Regler schaltet die Wärmepumpe nicht um. Er mischt nur und meldet den Kühlbetrieb über MQTT, Home Assistant und Modbus, z. B. für eine Automation, die die Wärmepumpe in den Kühlbetrieb schaltet.

**So regelt der Mischer beim Kühlen:** Mischer AUF bedeutet jetzt „mehr kaltes Wasser“, also einen kühleren Vorlauf. Die Wirkrichtung des PI-Reglers ist deshalb umgekehrt: Ist der Vorlauf zu warm, öffnet der Mischer. Mischer ZU bleibt die sichere Stellung – dann fließt kein kaltes Wasser in den Heizkreis. Beim Wechsel zwischen Heizen und Kühlen beginnt der Regler neu.

### Wann wird gekühlt?

| Betriebsart | gedämpfte Außentemperatur | Heizkreis |
|---|---|---|
| Kühlen | über der Kühlgrenze + 0,5 K | kühlt |
| Kühlen | unter der Kühlgrenze − 0,5 K | ruht (Mischer zu, Pumpe aus) |
| Heizen/Kühlen | unter der Heizgrenze − 0,5 K | heizt wie „Automatik“ (Heizkurve, Zeitprogramm) |
| Heizen/Kühlen | zwischen Heiz- und Kühlgrenze | ruht |
| Heizen/Kühlen | über der Kühlgrenze + 0,5 K | kühlt |

Die Kühlgrenze liegt immer mindestens 2 K über der Heizgrenze, damit der Regler nicht hin und her schaltet. Bei Frost wird nie gekühlt.

### Kühlkurve und Raum

Der Vorlauf-Sollwert beim Kühlen kommt aus einer **Kühlkurve**: einer Geraden durch zwei Punkte. Werkseinstellung:

| | Außentemperatur | Vorlauf-Soll |
|---|---|---|
| Punkt A | 22 °C (und kühler) | 22 °C |
| Punkt B | 32 °C (und wärmer) | 18 °C |

Dazwischen wird linear gerechnet, z. B. 27 °C außen → 20 °C Vorlauf. Je wärmer es draußen ist, desto kühler der Vorlauf.

Ist ein Raumwert vorhanden und *Raumtemperatur berücksichtigen* an, gilt der **Raumsollwert Kühlen** (Standard 24 °C) mit derselben Raumaufschaltung wie beim Heizen: Ist der Raum 1 K zu warm, wird der Vorlauf bei Faktor 3 um 3 K kühler (begrenzt durch *Max. Vorlaufkorrektur*). Die Raumabschaltung wirkt umgekehrt: Liegt der Raum um den eingestellten Wert **unter** dem Kühl-Sollwert, endet das Kühlen. Eine Absenkung nach Zeitprogramm gibt es beim Kühlen nicht.

### Taupunktschutz

Kühlt der Vorlauf unter den Taupunkt der Raumluft, bildet sich Kondenswasser an Boden, Rohren und Verteilern. Der Regler verhindert das in zwei Stufen:

1. **Untergrenze für den Sollwert:** Aus Raumtemperatur und relativer Raumfeuchte berechnet er den Taupunkt (Magnus-Formel, z. B. 26 °C und 60 % → 17,6 °C). Der Vorlauf-Sollwert bleibt mindestens *Abstand zum Taupunkt* (Standard 2 K) darüber und nie unter *Vorlauf min Kühlen* (Standard 16 °C). Fehlt ein aktueller Feuchtewert, gilt stattdessen *Vorlauf min ohne Feuchtewert* (Standard 20 °C).
2. **Taupunktalarm:** Fällt der gemessene Vorlauf trotzdem bis 0,5 K an den Taupunkt (ohne Feuchtewert: 2 K unter die Ersatzgrenze), fährt der Mischer sofort ZU und eine laufende Handschaltung endet. Die Pumpe läuft weiter. Freigabe 1 K darüber.

**Raumfeuchte einspeisen** – eine der drei Möglichkeiten:
- **MQTT-Topic** unter *Fühler → Raumfeuchte – MQTT-Topic*, z. B. `zigbee2mqtt/Wohnzimmer` mit JSON-Feld `humidity`. Es darf dasselbe Topic wie die Raumtemperatur sein.
- **MQTT-Befehl** `…/set/humidity` mit der Feuchte in %, z. B. aus einer Home-Assistant-Automation.
- **Modbus** Holding-Register 24 (Feuchte in % × 10).

Der Feuchtewert unterliegt derselben Zeitüberwachung wie externe Außen- und Raumwerte (*Fühler → Zeitüberwachung*, Standard 30 min).

### Einstellungen

Im Reiter **Heizkurve**, Karte **Kühlen & Taupunktschutz**. Über der Karte stehen die aktuellen Werte: Kühlkurven-Soll, wirksame Untergrenze, Taupunkt, Raumfeuchte und ob der Heizkreis gerade kühlt, heizt oder ruht.

| Einstellung | Standard | Bereich |
|---|---|---|
| Raumsollwert Kühlen | 24 °C | 18–30 °C |
| Kühlgrenze Außen | 22 °C | 15–35 °C (mind. Heizgrenze + 2 K) |
| Kühlkurve Punkt A: Außen / Vorlauf | 22 °C / 22 °C | 15–40 °C / 12–28 °C |
| Kühlkurve Punkt B: Außen / Vorlauf | 32 °C / 18 °C | 20–45 °C (mind. A + 2 K) / 10–28 °C |
| Vorlauf min Kühlen | 16 °C | 10–25 °C |
| Abstand zum Taupunkt | 2 K | 0–10 K |
| Vorlauf min ohne Feuchtewert | 20 °C | 12–28 °C |

Im **Anlagenschema** erscheint der Kühlbetrieb türkis: Vorlauf und Kesselweg des Mischers türkis, Rücklauf gelb (er ist jetzt die wärmere Seite), statt Flamme eine Schneeflocke und „Kälte“ statt „Kessel“. Die Statusanzeigen zeigen „Kühlbetrieb“, Feuchte und Taupunkt sowie „Vorlauf an Taupunktgrenze“ oder „Taupunktschutz – Mischer zu“.

> ⚠️ Der Taupunktschutz des Reglers ersetzt keinen Taupunktwächter in der Anlage, wo der Hersteller der Wärmepumpe oder des Bodens einen vorschreibt. Rohre und Verteiler im Kühlbetrieb gegen Kondenswasser dämmen.

## Mischer und Regler

### Stellantrieb

| Typ | Ansteuerung | Stellung |
|---|---|---|
| 3-Punkt (Auf/Zu) | Relais AUF (D1) bzw. ZU (D2) für eine berechnete Zeit | geschätzt aus der Motorlaufzeit |
| 0–10 V | PWM an GPIO 19 → Wandler → 0–10 V | wird direkt vorgegeben |

**So funktioniert der 3-Punkt-Antrieb:** Er hat keine Rückmeldung, deshalb rechnet der Regler die Stellung über die Laufzeit nach. Bei 120 s Motorlaufzeit entsprechen 12 s Laufzeit 10 % Stellweg.
- **Kalibrierung:** Nach jedem Start fährt der Mischer mit 110 % der Motorlaufzeit ganz zu. Danach steht er sicher auf 0 %. Über *System → Mischer kalibrieren* lässt sich das jederzeit wiederholen.
- **Endlagen:** Für 0 % und 100 % wird 10 % länger gefahren, damit die Endlage sicher erreicht wird. Steht der Mischer bereits in der Endlage, taktet der Regler höchstens 20 % der Laufzeit nach und hört dann auf.
- **Exakte Impulse:** Ein Hardware-Timer beendet jeden Impuls, auch wenn das Programm gerade beschäftigt ist.

### PI-Regler

Der Regler arbeitet in Geschwindigkeitsform: Pro Abtastung berechnet er eine Stelländerung in %, die als Impuls `Δ% / 100 × Motorlaufzeit` ausgegeben wird. Zu kurze Impulse werden aufsummiert, bis die *Min. Impulslänge* erreicht ist.

```
e  = Vorlauf-Soll − Vorlauf-Ist        (Totzone: |e| < Totzone → 0)
ΔY = Kp · (e − e_alt) + Kp · Abtastzeit / Tn · e
```

| Parameter | Standard (Radiator) | Wirkung |
|---|---|---|
| Kp | 3 %/K | größer = schneller, aber Schwingneigung |
| Tn | 150 s | kleiner = stärkerer I-Anteil; 0 = nur P-Regler |
| Totzone | 0,5 K | Ruhe am Antrieb bei kleiner Abweichung |
| Abtastzeit | 10 s | Abstand der Regelschritte |
| Min. Impulslänge | 400 ms | schont Relais und Motor |

**Einstellen:**
1. Mit den Profilwerten starten.
2. Pendelt der Vorlauf, Kp verkleinern und Tn vergrößern.
3. Ist er zu träge, umgekehrt.
4. Bei langen Wegen zwischen Mischer und Vorlauffühler Tn erhöhen.

Im Reiter *System* zeigt **„Letzter Stellschritt“** den letzten echten Stellschritt des Reglers mit Richtung, Datum und Uhrzeit, z. B. „+0,42 % · AUF 17.10. 03:59:57 (vor 2 min)“. Schritte von 0 % (Abweichung innerhalb der Totzone) zählen nicht, deshalb bleibt der letzte wirksame Schritt stehen. Seit dem Neustart noch keiner, steht dort „–“. **„Heizkurven-Soll“** zeigt den Sollwert laut Kurve und wann er zuletzt berechnet wurde (bei jedem Regelzyklus, also alle 10–20 s). Liegt dieser Zeitpunkt deutlich weiter zurück, arbeitet die Regelung nicht mehr.

### Zähler der Stellvorgänge

Der Regler zählt alle Stellvorgänge des Mischers seit dem letzten Neustart, getrennt nach AUF und ZU.
- **3-Punkt-Antrieb:** Jeder Relaisimpuls zählt als ein Stellvorgang, auch die Kalibrierfahrt nach dem Start.
- **0–10-V-Antrieb:** Jede Änderung der Stellung um mindestens 0,1 % zählt.

**Wo der Zähler erscheint:**

| Ort | Anzeige |
|---|---|
| Weboberfläche | Übersicht unter dem Anlagenschema (gesamt) und Reiter *System* (gesamt, AUF, ZU) |
| MQTT | `…/valve_moves` sowie `valveMoves`, `valveOpens`, `valveCloses` im JSON unter `…/state` |
| Home Assistant | Sensor „Mischer-Stellvorgänge“ (Zustandsklasse *total_increasing*, Neustarts werden erkannt) |
| Modbus | Input-Register 14 (oberes Wort) und 15 (unteres Wort), zusammen 32 Bit |

**Wozu der Zähler dient:** Er zeigt, wie stark Antrieb und Relais beansprucht werden. Wie viele Stellvorgänge normal sind, hängt stark von Anlage und Wetter ab. Aussagekräftig ist vor allem der Vergleich: Notiere den Zuwachs über einen Tag und vergleiche ihn nach einer Änderung von Kp, Tn, Totzone oder Mindestimpuls. Steigt die Zahl stark, ohne dass der Vorlauf ruhiger wird, ist der Regler zu „nervös“. Dann Kp verkleinern oder Totzone bzw. Mindestimpuls vergrößern. Der Zähler liegt im Arbeitsspeicher und beginnt nach jedem Neustart bei 0.

### Pumpenlaufzeit und Pumpenstarts

Der Regler zählt seit dem letzten Neustart, wie lange die Heizkreispumpe gelaufen ist und wie oft sie eingeschaltet wurde. Läufe des Blockierschutzes und des Frostschutzes zählen mit.

| Ort | Anzeige |
|---|---|
| Weboberfläche | Übersicht: „Pumpe EIN · 12 h 34 min“; Reiter *System*: Laufzeit, Anteil an der Betriebszeit des Reglers in %, Anzahl Starts |
| MQTT | `…/pump_runtime` (Stunden, 2 Nachkommastellen), `…/pump_starts`; im JSON unter `…/state`: `pumpRunSec`, `pumpRunH`, `pumpStarts` |
| Home Assistant | Sensoren „Pumpenlaufzeit“ (Stunden, Geräteklasse *duration*) und „Pumpenstarts“, beide mit Zustandsklasse *total_increasing* |
| Modbus | Input-Register 16/17: Laufzeit in Sekunden (32 Bit), 18/19: Starts (32 Bit) |

**Dauerbetrieb:** Die Laufzeit wird während des Betriebs jede Minute fortgeschrieben. Dadurch stimmt sie auch bei wochenlangem Dauerbetrieb, wenn der interne Millisekundenzähler des ESP8266 nach etwa 49 Tagen überläuft.

**Neustart:** Beide Zähler liegen im Arbeitsspeicher und beginnen nach jedem Neustart bei 0. Home Assistant erkennt das Zurücksetzen und führt in seiner Statistik eine Gesamtsumme weiter.

### Wärmemengenzählung

Seit 2.3.0 berechnet der Regler die **Wärmeleistung** des Heizkreises und zählt die **Wärmemenge** – ohne Wärmemengenzähler, aus dem Volumenstrom der Pumpe und der Temperaturdifferenz Vorlauf − Rücklauf:

```
P = V̇ · ρ · c · (T_Vorlauf − T_Rücklauf)
V̇ = Volumenstrom (l/min ÷ 60), ρ = Dichte des Wassers (temperaturabhängig, ca. 0,99 kg/l), c = 4180 J/(kg·K)
```

Beispiel: 12 l/min, Vorlauf 45 °C, Rücklauf 38 °C → 0,2 l/s · 0,99 kg/l · 4180 J/(kg·K) · 7 K ≈ **5,8 kW**.

**Einstellen:** Reiter *Regler*, Karte *Mischerregler*, Feld **Volumenstrom Heizkreis l/min** (`flowLpm`, 0 = Zählung aus).

**Volumenstrom ermitteln:** Die Pumpe **Grundfos UPM3 Hybrid 25-70** zeigt keinen Durchfluss an; ihre Leuchtanzeige gibt nur grob die Leistungsaufnahme wieder. Möglichkeiten, von genau bis grob:
1. **Hydraulischer Abgleich / Heizungsbauer:** Dort steht der Volumenstrom des Heizkreises meist im Protokoll.
2. **Wärmemengenzähler oder Wärmepumpe mit Durchflussanzeige** im selben Kreis: Wert ablesen, wenn nur dieser Heizkreis läuft.
3. **Aus Kennlinie und Förderhöhe:** Im Datenblatt der UPM3 Hybrid 25-70 die Kennlinie der eingestellten Betriebsart (Proportionaldruck, Konstantdruck oder Konstantkurve) mit der Förderhöhe bzw. dem Druckverlust der Anlage schneiden.
4. **Aus der Heizlast:** Volumenstrom ≈ Heizlast ÷ (0,7 kW pro l/min bei 10 K) – z. B. 8 kW bei 7 K Spreizung → 8 ÷ (0,07 · 7) ≈ 16 l/min.

Faustregel zum Prüfen: **1 l/min bei 10 K Spreizung ≈ 0,7 kW.** Bei konstant eingestellter Pumpe (Konstantkurve) bleibt der Volumenstrom am stabilsten; bei Proportionaldruck schwankt er mit den Thermostatventilen, die Zählung ist dann ungenauer.

**Gezählt wird** nur bei laufender Pumpe und gültigen Vorlauf- und Rücklaufwerten, alle 2 s. Beim Heizen zählt Vorlauf > Rücklauf als Wärme, beim Kühlen Rücklauf > Vorlauf als **Kälte** (eigener Zähler). Die jeweils andere Richtung – etwa durch Fühlertoleranz bei ruhendem Kreis – zählt nicht.

**Anzeige:**

| Ort | Inhalt |
|---|---|
| Übersicht | Etikett „Wärmeleistung 5,8 kW · heute 18,3 kWh“ (beim Kühlen „Kälteleistung …“); auch in der Nur-Anzeige |
| System → Kennwerte | Wärmeleistung mit Volumenstrom und Spreizung; Wärmemenge heute, gestern, gesamt; Kältemenge (sobald gekühlt wurde) |
| System → Zähler | Wärmemenge (und Kältemenge) seit Neustart und gesamt |
| MQTT (`…/state`) | `heatKw`, `heatKwhToday`, `heatKwhYest`, `heatKwhSess`, `heatKwhTotal`, `coolKw`, `coolKwhToday`, `coolKwhSess`, `coolKwhTotal`, `flowLpm`, `energyValid` |
| Home Assistant | Sensoren „Wärmeleistung“ (kW, Leistung), „Wärmemenge“ und „Wärmemenge heute“ (kWh, Energie, *total_increasing*), „Kälteleistung“, „Kältemenge“ – die Energie-Sensoren lassen sich im **Energie-Dashboard** verwenden |

**Speicherung:** Die Gesamtmengen und die Tageswerte stehen mit den anderen Gesamtzählern in `/counters.json`. Gespeichert wird alle 6 h, um Mitternacht (Tageswechsel) und vor jedem geordneten Neustart. Bei einem Stromausfall gehen höchstens die letzten 6 h der Zählung verloren. *Gesamtzähler zurücksetzen* setzt auch die Wärmemenge auf 0. War der Regler länger als einen Tag aus, beginnt „gestern“ bei 0.

**Genauigkeit:** Sie hängt vor allem am eingetragenen Volumenstrom und an der Temperaturdifferenz: 0,1 K Fühlerabweichung bei 5 K Spreizung sind 2 %. Beide Fühler am besten gleich montieren (Anlegefühler mit Wärmeleitpaste und Isolierung) und bei Bedarf unter *Fühler → Korrektur* angleichen – etwa bei ruhendem, warmem Kreis mit offenem Mischer. Gut zum Vergleichen von Tagen und Einstellungen, nicht für eine Abrechnung.

### Gesamtzähler über alle Neustarts

Zu allen drei Zählern – Mischer-Stellvorgänge (AUF/ZU), Pumpenlaufzeit und Pumpenstarts – führt der Regler zusätzlich **Gesamtzähler** (dazu kommen Neustarts und OTA-Updates, siehe unten bzw. [Firmware-Update](#firmware-update-über-wlan-ota)), die Neustarts, Stromausfälle und Firmware-Updates überstehen. Im Reiter *System* zeigt die Karte **Zähler** beide Werte nebeneinander, dazu das Datum des Zählbeginns und der letzten Speicherung.

**Speicherung:** Die Gesamtzähler stehen in der Datei `/counters.json` im Flash, getrennt von den Einstellungen.
- **Planmäßig:** alle **24 Stunden**.
- **Vor geplanten Neustarts:** etwa nach Einstellungsänderungen, über *Neustart* oder *Werkseinstellungen*.
- **Vor jedem OTA-Update.**
- **Von Hand:** über *Gesamtzähler jetzt speichern*.

Bei einem **Stromausfall** fehlt höchstens der Zuwachs seit der letzten Speicherung, also maximal ein Tag. Mit einem Schreibvorgang pro Tag wird der Flash-Speicher praktisch nicht belastet.

**Zurücksetzen:** *Gesamtzähler zurücksetzen* setzt alle Gesamtzähler auf 0 und startet die Zählung neu, zum Beispiel nach dem Tausch von Pumpe oder Stellantrieb. Das geht absichtlich nur über die Weboberfläche. *Werkseinstellungen* löscht die Gesamtzähler **nicht**.

| Ort | Gesamtzähler |
|---|---|
| Weboberfläche | Reiter *System*, Karte *Zähler* |
| MQTT | `…/pump_runtime_total` (Stunden), `…/pump_starts_total`, `…/valve_moves_total`; im JSON unter `…/state`: `totPumpRunSec`, `totPumpRunH`, `totPumpStarts`, `totValveMoves`, `totValveOpens`, `totValveCloses`, `totSince`, `totSaved` |
| Home Assistant | Sensoren „Pumpenlaufzeit gesamt“, „Pumpenstarts gesamt“, „Mischer-Stellvorgänge gesamt“ |
| Modbus | Input-Register 20/21 Stellvorgänge, 22/23 Pumpenlaufzeit in s, 24/25 Pumpenstarts (je 32 Bit) |

### Neustart-Zähler

Der Regler zählt alle Neustarts über die gesamte Betriebszeit und wertet den Grund aus, den der ESP32 bei jedem Start selbst meldet:

| Grund | Bedeutung |
|---|---|
| Einschalten / Stromausfall | Versorgungsspannung war weg |
| Software-Neustart | gewollter Neustart: Einstellungsänderung, *Neustart*, *Werkseinstellungen* oder Firmware-Update |
| Absturz / Watchdog | Programmabsturz, Watchdog oder Unterspannung; der ESP32 hat sich selbst neu gestartet |
| Reset-Taste | Taster EN auf der Platine oder externer Reset |

**Absturz / Watchdog** sollte im Normalbetrieb 0 bleiben. Steigt der Wert, lohnt sich ein Blick auf Stromversorgung, WLAN-Empfang und die zuletzt geänderten Einstellungen. Den genauen Grund des letzten Absturzes zeigt der Reiter *System* unter **Letzter Absturz**; er bleibt in `/counters.json` gespeichert, auch wenn das Ereignisprotokoll beim Neustart verloren geht.

### Abstürze und Speicher eingrenzen

| Anzeige unter *Letzter Absturz* | Bedeutung, was tun |
|---|---|
| Programmfehler (Absturz) | Der ESP32 ist auf einen Fehler gelaufen. Die serielle Ausgabe (USB-Seriell-Adapter, 115200 Baud) zeigt beim Absturz einen „Backtrace“; der *ESP Exception Decoder* der Arduino IDE ordnet ihn einer Quelltextzeile zu – bitte melden. |
| Task-Watchdog – Regel-Task … blockiert | Der Regel-Task ist 30 s lang nicht durchgelaufen. Das sollte nie vorkommen – bitte mit dem Ereignisprotokoll melden. |
| Hauptschleife hing länger als 3 min | Webserver, MQTT oder Modbus hingen; der Regel-Task hat die Ausgänge abgeschaltet und neu gestartet. |
| Interrupt-Watchdog / Watchdog | Das Programm hing auf unterster Ebene, oft durch elektrische Störungen beim Schalten: Entstörglied (RC-Glied) oder Varistor an den Relaiskontakten, Fühler- und Netzleitungen getrennt führen. |
| Unterspannung (Netzteil zu schwach?) | Die Versorgung ist kurz eingebrochen, meist beim Anziehen der Relais oder beim WLAN-Senden. Stärkeres Netzteil, kurze dicke Zuleitung. |
| Speichermangel – geordneter Neustart | Der Speicherwächter hat neu gestartet, weil 30 s lang weniger als 15 000 Byte frei waren. Mit dem ESP32 nicht zu erwarten – bitte melden. |

Richtwerte für *Freier Speicher*: im Betrieb meist 150–220 KB frei. Unter 40 000 Byte steht eine Warnung im Ereignisprotokoll.

**Speicherung:** Jeder Start wird sofort im Flash gezählt, also ein Schreibvorgang je Neustart. Der Zähler steht zusammen mit den anderen Gesamtzählern in `/counters.json` und wird mit *Gesamtzähler zurücksetzen* ebenfalls auf 0 gesetzt.

| Ort | Anzeige |
|---|---|
| Weboberfläche | Reiter *System*, Karte *Zähler*: Grund des aktuellen Starts, Neustarts gesamt und nach Grund (Abstürze rot, wenn > 0) |
| MQTT | `…/restarts_total`, `…/last_reset` (`power` / `software` / `crash` / `button`); im JSON unter `…/state`: `totBoots`, `bootsPower`, `bootsSoft`, `bootsCrash`, `bootsButton`, `lastReset`, `lastResetText` |
| Home Assistant | Diagnose-Sensoren „Neustarts gesamt“, „Neustarts durch Absturz“, „Letzter Neustartgrund“ |
| Modbus | Input-Register 26/27 Neustarts gesamt (32 Bit), 28 Grund des letzten Starts (0 Einschalten, 1 Software, 2 Absturz, 3 Reset-Taste), 29 Neustarts durch Absturz |

### Gerät und Netzwerk

Die Karte **Gerät und Netzwerk** im Reiter *System* (unter der Karte *System*) zeigt die technischen Daten des Reglers. Sie ist praktisch, um den Regler im Router zu finden, ihm dort eine feste IP-Adresse zu geben, WLAN-Probleme einzugrenzen oder bei einer Anfrage alle Daten auf einmal weiterzugeben.

**Netzwerk**

| Zeile | Inhalt | Wofür |
|---|---|---|
| MAC-Adresse (WLAN) | Hardware-Adresse der WLAN-Schnittstelle, z. B. `5C:CF:7F:A3:41:B2` | Im Router eine **feste IP-Adresse (DHCP-Reservierung)** vergeben oder den Regler in der Geräteliste finden; MAC-Filter |
| MAC-Adresse Access-Point | Adresse des eigenen Access-Points `Heizungsregler-…` | erscheint nur in Geräten, die sich mit diesem Access-Point verbinden (Ersteinrichtung) |
| Hostname | Name im Netzwerk, dazu die mDNS-Adresse, sobald mDNS läuft | Aufruf im Browser über `http://heizungsregler.local` |
| Verbindung | *WLAN-Client*, *Access-Point*, beides oder *nicht verbunden* | |
| WLAN (SSID) | Name des verbundenen WLANs | |
| Router / Access-Point (BSSID) | MAC-Adresse des Routers oder Repeaters, mit dem der Regler gerade verbunden ist | Bei Mesh-Netzen oder Repeatern sieht man, an welchem Gerät der Regler hängt |
| Kanal | WLAN-Kanal (1–13) | Störungen durch Nachbarnetze auf demselben Kanal erkennen |
| Signalstärke | in dBm mit Einordnung: ab −55 *sehr gut*, ab −67 *gut*, ab −75 *mäßig*, darunter *schwach* | Bei *schwach* bricht die Verbindung leicht ab; Antenne/Standort prüfen oder Repeater |
| IP-Adresse, Subnetzmaske, Gateway, DNS-Server | IP-Konfiguration, wie sie der Router per DHCP vergeben hat | Fehlersuche, wenn MQTT oder die Uhrzeit nicht funktionieren |
| Eigener Access-Point | nur wenn aktiv: Name, IP-Adresse und Anzahl verbundener Geräte | |
| MQTT-Broker | Adresse und Port, Zustand (*verbunden* oder *getrennt* mit dem Fehlercode der Bibliothek PubSubClient) und die Client-ID | Fehlersuche bei MQTT; die Client-ID ist im Broker-Protokoll zu sehen |
| Uhrzeit (NTP) | *synchron* oder *noch nicht synchron* | ohne Uhrzeit arbeiten Zeitprogramm und Feiertage nicht |

Fehlercodes MQTT (Auszug): −4 Zeitüberschreitung, −2 Verbindung fehlgeschlagen (Broker nicht erreichbar), 4 Benutzername/Passwort falsch, 5 nicht berechtigt.

**Hardware**

| Zeile | Inhalt |
|---|---|
| Chip-ID | die letzten 6 Hex-Stellen der fest eingebrannten MAC-Adresse des ESP32. Sie steckt auch im Namen des Access-Points, in der MQTT-Client-ID und in der Home-Assistant-Kennung `hzr_…` |
| Prozessor | Chip-Modell, Revision und Anzahl der Kerne mit Taktfrequenz, z. B. „ESP32-D0WD-V3 Rev. 3, 2 Kerne, 240 MHz“ |
| Arbeitsspeicher | freier und gesamter Arbeitsspeicher |
| Flash | verbaute Größe, in der Arduino IDE eingestellte Größe, Takt und Kennung des Flash-Chips. Weichen verbaute und eingestellte Größe ab, erscheint ein roter Hinweis: Dann in der IDE *Flash Size* prüfen (richtig: „4MB (FS:1MB OTA:~1019KB)“) |
| Programm | Größe der Firmware und der freie Platz für das nächste OTA-Update. Ein Update braucht etwa so viel freien Platz, wie die Firmware groß ist |
| Dateisystem | belegter und gesamter Platz im LittleFS (Einstellungen, Zähler, Profil GLW8, Wochenverlauf) |
| Letzter Start | Startgrund, wie ihn der ESP32 meldet: *Einschalten*, *Neustart durch Software*, *Reset-Taste / externer Reset*, *Programmfehler (Absturz)*, *Task-Watchdog*, *Interrupt-Watchdog*, *Unterspannung* |

**Software**

| Zeile | Inhalt |
|---|---|
| Firmware | Version und Übersetzungszeitpunkt (Datum und Uhrzeit, zu denen die Firmware in der Arduino IDE übersetzt wurde) – unterscheidet zwei Dateien mit derselben Versionsnummer |
| ESP32-Core / ESP-IDF | Version des ESP32-Arduino-Cores und des ESP-IDF, mit denen übersetzt wurde |

**Als Text kopieren** legt alle Angaben als einfachen Text in die Zwischenablage, gegliedert wie in der Karte, z. B.:

```
GLW8 Heizungsregler – Gerät und Netzwerk

Netzwerk
  MAC-Adresse (WLAN): 5C:CF:7F:A3:41:B2
  Hostname: heizungsregler (heizungsregler.local)
  …
```

**Laden und Datenschutz:** Die Werte kommen über `GET /api/sysinfo`. Die Weboberfläche fragt sie nur ab, solange der Reiter *System* offen ist: beim Öffnen und dann alle 30 Sekunden (nicht im Hintergrund-Tab). Im Status, der alle 2 Sekunden abgefragt wird, stehen sie bewusst nicht; das hält die häufigen Abfragen klein. Ist ein Web-Passwort gesetzt, ist `/api/sysinfo` geschützt; die Nur-Anzeige zeigt keine MAC- oder Netzwerkdaten.

---

## Relais K4 (Zusatzfunktion)

Relais K4 (GPIO 26) hat einen Wechslerkontakt: **COM** (gemeinsam), **NO** (Schließer – geschlossen, wenn das Relais angezogen ist) und **NC** (Öffner – geschlossen, wenn es abgefallen ist). Seine Funktion stellt man im Reiter *Regler*, Karte **Relais K4 (Zusatzfunktion)** ein. Dort steht auch, was K4 gerade tut, z. B. „Wärmeanforderung – Heizkreis braucht Wärme · Relais angezogen“. Die Karte zeigt nur die Felder, die zur gewählten Funktion gehören.

| Funktion | K4 ist an (Relais angezogen) … | typische Verdrahtung |
|---|---|---|
| **Aus** | nie | – |
| **Wärmeanforderung** (Werkseinstellung) | solange der Heizkreis Wärme braucht, siehe unten | COM/NO an den potenzialfreien Anforderungs- bzw. Freigabeeingang von Kessel oder Wärmepumpe („Raumthermostat“, „externe Anforderung“) |
| **Heizen/Kühlen-Umschaltung** | solange der Heizkreis **kühlt** | COM/NO an den Eingang „Kühlbetrieb“ der Wärmepumpe |
| **Sammelstörmeldung** | solange **keine** Störung vorliegt (Ruhestromprinzip) | COM/NC an Meldeleuchte, Hupe oder Störmeldeeingang: schließt bei Störung |
| **Zirkulationspumpe** | in den Zeitfenstern, im Takt Laufzeit/Pause | COM/NO schaltet die Pumpe – bei höherer Last über ein Schütz |
| **Entfeuchter** | beim Kühlen, wenn es zu feucht wird | COM/NO |
| **Frei schaltbar** | wenn von Hand eingeschaltet (Web, MQTT, Home Assistant, Modbus) | beliebig, z. B. für eine Automation in Home Assistant |

**Wärmeanforderung im Detail:** K4 zieht an, solange der Mischerregler im **Heizbetrieb** arbeitet: Automatik unterhalb der Heizgrenze, fester Sollwert, Raumtemperatur (außer in der Heizpause), Heizen/Kühlen beim Heizen und **Frostschutz**. Im Handbetrieb fordert K4 an, solange der Heizkreis heizen würde. Keine Anforderung gibt es bei Heizgrenze, Raumabschaltung, Heizpause der Raumregelung, Betriebsart *Aus*, beim **Kühlen** und bei **Übertemperatur** im Vorlauf – der Wärmeerzeuger soll dann nicht noch mehr Wärme liefern. Eine Temperatur überträgt das Relais nicht: Der Wärmeerzeuger muss heiß genug liefern, der Mischer regelt herunter.

**Mindestlaufzeit und Mindestpause** (Werkseinstellung je 5 min, 0 = aus) gelten für Wärmeanforderung, Heizen/Kühlen und Entfeuchter. Nach dem Einschalten bleibt K4 mindestens so lange an, nach dem Ausschalten mindestens so lange aus. Das schützt Kessel, Verdichter und Entfeuchter vor kurzem Takten. Die Karte zeigt die Restzeit, z. B. „Mindestlaufzeit noch 3 min“. Nach einem Neustart und nach dem Wechsel der Funktion gilt keine Mindestpause.

**Sammelstörmeldung:** Als Störung zählen Fühlerfehler (Vorlauf, Rücklauf, Außen, Raum), Übertemperatur und Taupunktalarm – dieselben Bedingungen wie bei der Home-Assistant-Entität *Störung*, dazu der Taupunktalarm. Gemeldet wird erst, wenn die Störung **länger als *Störung melden nach*** (Werkseinstellung 60 s) ansteht; kurze Aussetzer eines Fühlers lösen so keinen Alarm aus. Weil das Relais im Ruhestromprinzip arbeitet, meldet es auch Stromausfall, Absturz und einen hängenden Regler. **Auch jeder Neustart und jedes Firmware-Update** unterbricht die Meldung kurz – eine Störmeldezentrale sollte deshalb erst nach etwa 30 s Alarm geben.

**Zirkulationspumpe:** *Zeitfenster* als Text, z. B. `06:00-08:00, 17:00-22:00` (bis zu 4 Fenster, Ende bis 24:00; leer = ganztägig). In den Fenstern läuft die Pumpe *Laufzeit* Minuten und pausiert *Pause* Minuten (Werkseinstellung 5 / 25, also 5 min je halbe Stunde); Laufzeit 0 = durchgehend. Der Takt beginnt mit dem Zeitfenster. Die Funktion braucht die Uhrzeit; ohne Internet-Uhrzeit bleibt die Pumpe aus. Wochentage und Feiertage unterscheidet sie nicht.

**Entfeuchter:** nur während des Kühlbetriebs. An, sobald die Raumfeuchte *Ein ab Raumfeuchte* erreicht (Werkseinstellung 65 %, aus 5 % darunter) **oder** der Taupunktschutz den Vorlauf anhebt bzw. den Mischer schließt. Die Raumfeuchte kommt vom Bluetooth-Sensor des Raums (RuuviTag, Shelly BLU H&T), über MQTT oder Modbus (siehe [Kühlen](#kühlen)).

**Frei schaltbar:** Knöpfe *K4 EIN* / *K4 AUS* in der Karte, `…/set/k4Free ON|OFF` über MQTT, Schalter *Relais K4 frei schalten* in Home Assistant oder Holding-Register 26 über Modbus. Der Zustand bleibt über Neustarts erhalten.

**Ausgang umkehren:** tauscht angezogen und abgefallen (Funktion aus = Relais angezogen). Meist genügt es, statt NO den NC-Kontakt zu verdrahten; die Einstellung hilft, wenn die Verdrahtung schon feststeht. Bei der Sammelstörmeldung hebt sie das Ruhestromprinzip auf. Sie lässt sich nur im Web ändern, nicht über MQTT oder Modbus.

**Anzeige und Zähler:** Die Übersicht zeigt ein Kennzeichen, z. B. „K4 Wärmeanforderung: an“, „K4: Wärmepumpe Kühlen“ oder rot „K4: Störung gemeldet“. Im Reiter *System* steht die Zeile *Relais K4*, unter *Zähler* Laufzeit und Zahl der Einschaltungen seit Neustart und gesamt (bei der Störmeldung: Dauer und Zahl der gemeldeten Störungen). Jeder Wechsel steht im Ereignisprotokoll.

**Schnittstellen:**
- MQTT: `…/k4` (`ON` = Funktion aktiv, bei der Störmeldung = Störung gemeldet), `…/k4_relay` (`ON` = angezogen), im JSON `k4`, `k4Coil`, `k4Mode`, `k4ModeTxt`, `k4Info`, Zähler `k4RunSec`, `k4Starts`, `totK4RunSec`, `totK4Starts`. Einstellen über `…/set/k4Mode` (Zahl 0–6, Name wie `heat_request` oder Anzeigename), `…/set/k4Free`, `…/set/k4MinOn` usw.
- Home Assistant: Binärsensoren *Relais K4* und *Relais K4 angezogen*, Sensor *Relais K4 Zustand* (Klartext), Auswahl *Relais K4 Funktion*, Schalter *Relais K4 frei schalten*.
- Modbus: Holding-Register 25 (Funktion) und 26 (frei schalten), Input-Register 35–39, Statusbits 2 Bit 7/8.

> ⚠️ Die Relaiskontakte vertragen Netzspannung, aber nicht jede Last. Größere Pumpen und Entfeuchter über ein Schütz schalten. Kleinspannungseingänge von Kessel oder Wärmepumpe nie zusammen mit 230 V auf demselben Relais verdrahten. Arbeiten an 230 V nur durch eine Elektrofachkraft.

---

## Schutzfunktionen und Störungen

| Situation | Reaktion |
|---|---|
| Vorlauf > Vorlauf max + 5 K | Mischer ZU, Pumpe EIN (Wärme abführen), bis der Vorlauf unter Vorlauf max sinkt |
| Außen < Frostschutzgrenze | Pumpe EIN, Regelung auf Vorlauf min, auch in „Aus“ und im Sommer; aus bei Frostschutz + 1 K |
| Vorlauffühler defekt / kein Vorlaufwert | Mischer ZU, Pumpe läuft bei Heizbedarf weiter |
| Außenfühler defekt / kein Außenwert | Rechnung mit 0 °C Ersatzwert |
| Fühler am Pin defekt bei „Pin, bei Ausfall MQTT“ | automatische Umschaltung auf den externen Wert |
| externer Wert veraltet | Wert verworfen, Verhalten wie bei Fühlerdefekt |
| Raumfühler defekt / kein Raumwert | rein witterungsgeführt weiter |
| Pumpe 24 h nicht gelaufen | Blockierschutz: 180 s Pumpenlauf |
| WLAN weg | Regelung läuft weiter; nach 30 s öffnet sich zusätzlich der Access-Point |
| Neustart / Stromausfall | Einstellungen bleiben erhalten; Kalibrierfahrt ZU |
| Kühlen: Vorlauf am Taupunkt (+0,5 K) | Mischer ZU, Pumpe läuft weiter; Freigabe 1 K darüber |
| Kühlen: keine aktuelle Raumfeuchte | Untergrenze „Vorlauf min ohne Feuchtewert“ |
| Frost | nie kühlen |

Ein DS18B20 gilt erst nach drei Fehlmessungen in Folge als defekt. Bis dahin bleibt der letzte gültige Wert stehen. Fehlende Fühler sucht der Regler jede Minute neu.

---

## Temperaturverlauf

Im Reiter *Verlauf* zeigt die obere Grafik Vorlauf, Vorlauf-Soll, Rücklauf, Außen- und Raumtemperatur. Einzelne Kurven blendest du über die Legende ein oder aus. Die untere Grafik zeigt die Mischerstellung, die Pumpenlaufzeiten erscheinen als grüne Balken.

Fährt man mit der Maus über eine der Grafiken oder tippt mit dem Finger darauf, erscheinen für diesen Zeitpunkt alle Werte in einem Kästchen (siehe [Weboberfläche](#weboberfläche)).

**Zeiträume und Auflösung:** Der Regler zeichnet in drei Speichern parallel auf.

| Zeitraum | Abstand der Punkte | Datenpunkte |
|---|---|---|
| 1 h | 30 Sekunden | bis 120 |
| 6 h / 12 h / 24 h | 5 Minuten | bis 288 |
| 7 Tage | 1 Stunde (Mittelwert der 5-Minuten-Werte) | bis 168 |

Die 1-Stunden-Ansicht eignet sich, um das Regelverhalten des Mischers zu beurteilen, etwa beim Einstellen von Kp und Tn. Die Ansicht wird alle 30 Sekunden aktualisiert.

**7 Tage:** Jeder Punkt ist der Mittelwert einer vollen Stunde, z. B. 13–14 Uhr. Die senkrechten Linien markieren Mitternacht und sind mit Wochentag und Datum beschriftet (auf dem Smartphone nur der Wochentag). Beim Überfahren steht „Stundenmittel“ im Kästchen; bei der Pumpe „meist ein“ bzw. „meist aus“ (mindestens die halbe Stunde gelaufen oder nicht). Gut geeignet, um Heizkurve, Absenkung und Tagesgang über mehrere Tage zu beurteilen.
- Der Wochenverlauf braucht die Internet-Uhrzeit (NTP). Der erste Punkt erscheint nach der ersten vollen Stunde.
- **Speichern nur per Knopf:** *Wochenverlauf im Regler speichern* (unter dem Diagramm, nur angemeldet) schreibt den Wochenverlauf in den Flash (Datei `/hist7.bin`, ca. 2,4 KB). Automatisch wird nichts gespeichert – das schont den Flash. In der 7-Tage-Ansicht steht unter dem Diagramm, wann zuletzt gespeichert wurde.
- **Nach einem Neustart** lädt der Regler den zuletzt gespeicherten Stand. Was danach bis zum Neustart aufgezeichnet wurde, fehlt, ebenso die Zeit, in der der Regler aus war: beides erscheint als Lücke. Vor einem geplanten Neustart oder Update also kurz speichern.
- Über die REST-API: `POST /api/cmd` mit `{"saveHistory":true}`.
- Die CSV-Liste heißt dann z. B. `Heizungsverlauf_2026-10-24_2310_7d.csv`; die Zeitspalte nennt das Ende der jeweiligen Stunde.

**Download als Liste:** *Angezeigten Zeitraum als Liste herunterladen (CSV)* speichert genau den gewählten Zeitraum als Datei, z. B. `Heizungsverlauf_2026-10-16_2241_1h.csv`.
- **Spalten:** Zeit, Vorlauf, Vorlauf Soll, Rücklauf, Außen, Raum, Mischer %, Pumpe (EIN/AUS).
- **Format für deutsches Excel und LibreOffice:** Semikolon als Trennzeichen, Komma als Dezimalzeichen, UTF-8 mit BOM. Die Datei lässt sich per Doppelklick öffnen, Umlaute stimmen.
- **Fehlende Werte** (Fühler gestört oder nicht vorhanden) bleiben leer.
- **Zeitstempel:** Sie werden nach der Uhr des Geräts berechnet, auf dem die Weboberfläche läuft.

**Speicher:** Die Verläufe 1 h und 24 h liegen nur im Arbeitsspeicher und beginnen nach einem Neustart von vorn; zusammen mit dem Wochenverlauf belegen sie ca. 8 KB RAM. Der erste Punkt wird 1 Minute nach dem Start aufgezeichnet. Für eine dauerhafte Aufzeichnung eignet sich die Historie von Home Assistant.

---

## MQTT

Die Einstellungen stehen unter *Netzwerk*: Broker, Port, Benutzer, Passwort und Basis-Topic. Das Basis-Topic ist standardmäßig `heizung/hk1`.

### Status-Topics (vom Regler, retained)

| Topic | Inhalt |
|---|---|
| `…/status` | `online` / `offline` (Last Will) / `update` (während OTA) |
| `…/state` | alle Werte als JSON |
| `…/flow`, `…/return`, `…/outdoor`, `…/outdoor_filtered` | Temperaturen in °C |
| `…/room`, `…/room_target` | Raumtemperatur und Raumsollwert inkl. Absenkung in °C |
| `…/flow_set` | aktiver Vorlauf-Sollwert in °C |
| `…/valve` | Mischerstellung in % |
| `…/pump` | `ON` / `OFF` |
| `…/k4`, `…/k4_relay` | Relais K4: Funktion aktiv bzw. Relais angezogen, `ON` / `OFF` |
| `…/mode` | `off` / `auto` / `manual` / `hand` / `cool` / `heat_cool` / `room` |
| `…/valve_moves` | Stellvorgänge des Mischers seit Neustart |
| `…/pump_runtime` | Pumpenlaufzeit seit Neustart in Stunden |
| `…/pump_starts` | Pumpenstarts seit Neustart |
| `…/pump_runtime_total`, `…/pump_starts_total`, `…/valve_moves_total` | Gesamtzähler über alle Neustarts |
| `…/restarts_total` | Neustarts gesamt |
| `…/last_reset` | Grund des letzten Starts: `power` / `software` / `crash` / `button` |

Im JSON unter `…/state` stehen zusätzlich die Werte des Kühlens: `cooling` (Kühlbetrieb aktiv), `hcState` (0 ruht, 1 heizt, 2 kühlt), `coolSet` (Kühlkurven-Soll), `coolMin` (wirksame Untergrenze), `dewPoint`, `humidity`, `dewLimit` (Sollwert auf Untergrenze angehoben), `dewAlarm`, `coolRoomSet`, `coolLimit`.

Gesendet wird alle 10 Sekunden und sofort nach jeder Änderung. Ungültige Werte werden in den Einzel-Topics nicht gesendet und stehen im JSON als `null`.

### Steuern (an den Regler)

Das Format ist `…/set/<parameter>` mit dem Wert als Nutzlast. Ja/Nein-Werte verstehen `true`/`false`, `on`/`off`, `ON`/`OFF` und `1`/`0`.

```
heizung/hk1/set/mode          auto      (off, auto, manual, hand, cool, heat_cool, room)
heizung/hk1/set/coolRoomSet   24.5      (Raumsollwert Kühlen)
heizung/hk1/set/humidity      55        (Raumfeuchte in % für den Taupunktschutz)
heizung/hk1/set/targetTemp    21        (Sollwert der Thermostat-Karte: beim Kühlen Kühl-, sonst Raumsollwert)
heizung/hk1/set/roomSet       21.5
heizung/hk1/set/handPos       40
heizung/hk1/set/nightEnabled  ON        (Absenkbetrieb / Zeitprogramm)
heizung/hk1/set/schedMode     1         (0 = tägliches Fenster, 1 = Wochenprogramm)
heizung/hk1/set/nightStart    22:30
heizung/hk1/set/roomUse       OFF       (Raumwert nur anzeigen)
heizung/hk1/set/profile       floor     (radiator / floor)
heizung/hk1/set/roomTemp      21.3      (externe Raumtemperatur)
heizung/hk1/set/outTemp       4.8       (externe Außentemperatur)
heizung/hk1/set/calibrate     1
heizung/hk1/set/valveOpen     1         (Handschaltung: Mischer AUF)
heizung/hk1/set/valveStop     1         (Mischer anhalten)
heizung/hk1/set/valveAuto     1         (Mischer wieder automatisch)
heizung/hk1/set/pumpOff       1         (Pumpe von Hand AUS)
```

Mehrere Werte auf einmal schickst du als JSON an `heizung/hk1/set`, z. B. `{"slope":1.1,"shift":2}`.

**Über MQTT setzbare Parameter:**

| Bereich | Parameter |
|---|---|
| Betrieb | `mode`, `roomSet`, `manualFlowSet`, `handPos` |
| Heizkurve | `slope`, `shift`, `flowMin`, `flowMax`, `heatLimit`, `frostTemp`, `outFilterMin` |
| Stützpunkte | `curveMode`, `cpOut1` … `cpOut4`, `cpFlow1` … `cpFlow4`, `cpRoomK` |
| Rücklauf | `returnMax`, `returnFactor` |
| Zeitprogramm | `nightEnabled`, `schedMode`, `nightStart`, `nightEnd`, `nightReduce`, `week`, `holidayAT`, `holidayExtra` |
| Raum | `roomUse`, `roomFactor`, `roomCorrMax`, `roomCutoff` |
| Raumregler | `rcKp`, `rcTn`, `rcOff` (Betriebsart *Raumtemperatur*) |
| Kühlen | `coolRoomSet`, `coolLimit`, `coolOutA`, `coolFlowA`, `coolOutB`, `coolFlowB`, `coolFlowMin`, `dewMargin`, `coolFlowNoHum` |
| Regler | `kp`, `tn`, `deadband`, `cycleSec`, `travelSec`, `minPulseMs` |
| Fühler | `offFlow`, `offReturn`, `offOut`, `offRoom`, `extTimeout` |
| Externe Werte | `flowTemp`, `returnTemp`, `outTemp`, `roomTemp`, `humidity` (Raumfeuchte %) |
| Befehle | `calibrate`, `restart`, `rescan`, `profile` (`radiator`/`floor`/`glw8`), `profileSave` |
| Handschaltung | `valveOpen`, `valveClose`, `valveStop`, `valveAuto`, `pumpOn`, `pumpOff`, `pumpAuto` (Nutzlast beliebig) |
| Relais K4 | `k4Mode` (0–6 oder `off`/`heat_request`/`heat_cool`/`fault`/`circulation`/`dehumidifier`/`free`), `k4Free` (`ON`/`OFF`), `k4MinOn`, `k4MinOff`, `k4FaultSec`, `k4CircWin`, `k4CircRun`, `k4CircPause`, `k4HumOn` |

**Nur über die Weboberfläche änderbar**, damit sich niemand über das Netzwerk aussperren kann:
- WLAN, Hostname und Web-Passwort
- MQTT- und Modbus-Einstellungen sowie OTA
- Stellantrieb-Typ und Relais-Polarität
- Anschlussart, Zuordnung, Herkunft und MQTT-Topics der Fühler sowie das Feuchte-Topic

Über MQTT geänderte Werte werden gespeichert, sobald 10 Sekunden lang keine weitere Änderung kam. Das schont den Flash-Speicher.

---

## Home Assistant

Voraussetzung ist die MQTT-Integration in Home Assistant mit demselben Broker. Ist *Netzwerk → Home Assistant Discovery* aktiv, legt Home Assistant beim Verbinden automatisch ein Gerät mit diesen Entitäten an:

| Typ | Entitäten |
|---|---|
| Thermostat | „Heizkreis“ (Komponente `climate`), siehe [Thermostat in Home Assistant](#thermostat-in-home-assistant) |
| Sensoren | Vorlauf, Rücklauf, Außen, Außen gedämpft, Raum, Raum-Soll, Vorlauf-Soll, Heizkurven-Soll, Kühlkurven-Soll, Taupunkt, Raumfeuchte, Raumkorrektur, Mischerstellung, Mischer-Stellvorgänge, Pumpenlaufzeit, Pumpenstarts – jeweils seit Neustart und gesamt; Wärmeleistung, Wärmemenge (gesamt, heute), Kälteleistung, Kältemenge (siehe [Wärmemengenzählung](#wärmemengenzählung)); Relais K4 Zustand (Klartext); Diagnose: WLAN-Signal, Neustarts gesamt, Neustarts durch Absturz, letzter Neustartgrund |
| Binärsensoren | Pumpe, Heizbetrieb, Kühlbetrieb, Taupunktschutz, Absenkbetrieb, Frostschutz, Relais K4; Diagnose: Störung (Fühlerfehler oder Übertemperatur), Relais K4 angezogen |
| Auswahl | Betriebsart; Konfiguration: Heizungsprofil (`radiator` / `floor`, Anzeige `custom` wenn angepasst), Relais K4 Funktion |
| Zahlen | Raumsollwert, Raumsollwert Kühlen, Vorlauf fest, Mischer Hand; Konfiguration: Absenkung, Neigung, Niveau, Vorlauf min/max, Heizgrenze, Kühlgrenze, Raumaufschaltung |
| Schalter | Relais K4 frei schalten; Konfiguration: Absenkbetrieb / Zeitprogramm, Raumtemperatur berücksichtigen |
| Taster | Mischer kalibrieren, Neustart; Handschaltung: Mischer AUF / ZU / Stopp / freigeben, Pumpe EIN / AUS / Automatik |

### Thermostat in Home Assistant

Das Thermostat „Heizkreis“ wird per MQTT-Discovery als Komponente **`climate`** angemeldet und erscheint als Thermostat-Karte. Es zeigt Raumtemperatur und Raumfeuchte (falls vorhanden), den Sollwert, die Betriebsart und was der Heizkreis gerade tut.

**Thermostat-Modi (HVAC) und Betriebsarten des Reglers:**

| Modus in Home Assistant | Betriebsart des Reglers | Bedeutung |
|---|---|---|
| `off` | Aus | Mischer zu, Pumpe aus, Frostschutz bleibt aktiv |
| `heat` | **Raumtemperatur** | der Regler heizt auf das Raumsoll (PI-Raumregler) – das typische Thermostat-Verhalten |
| `auto` | Automatik | witterungsgeführt nach Heizkurve (mit Raumeinfluss) |
| `cool` | Kühlen | nach Kühlkurve mit Taupunktschutz |
| `heat_cool` | Heizen/Kühlen automatisch | Heizen unter der Heizgrenze, Kühlen über der Kühlgrenze |

*Fester Sollwert* und *Handbetrieb* haben keinen eigenen Thermostat-Modus; sie werden als `heat` angezeigt.

**Voreinstellungen (Presets):** Alle sieben Betriebsarten sind zusätzlich als Voreinstellung mit deutschem Namen wählbar: *Aus*, *Witterungsgeführt (Heizkurve)*, *Raumtemperatur*, *Fester Vorlauf*, *Handbetrieb*, *Kühlen*, *Heizen/Kühlen automatisch*. Die Karte zeigt so auch, welche Betriebsart hinter `heat` steckt.

**Solltemperatur:**
- in allen Modi außer `heat_cool` **eine** Temperatur: beim Kühlen der Raumsollwert Kühlen, sonst der Raumsollwert (`…/set/targetTemp`)
- bei `heat_cool` ein **Bereich**: unten der Raumsollwert Heizen (`…/set/roomSet`), oben der Raumsollwert Kühlen (`…/set/coolRoomSet`)
- 5–30 °C in Schritten von 0,5 K; die Ist-Temperatur mit 0,1 K

**Aktion:** *Heizen* bzw. *Kühlen*, solange die Pumpe läuft; *Untätig*, wenn der Heizkreis ruht (Heizgrenze, Heizpause der Raumregelung, zwischen Heiz- und Kühlgrenze); *Aus* in der Betriebsart Aus.

> **Geändert in 2.2.0:** Bis 2.1.0 bedeutete `heat` den festen Vorlauf-Sollwert. Automationen, die `climate.set_hvac_mode` mit `heat` verwenden, schalten jetzt auf *Raumtemperatur*. Für den festen Vorlauf `climate.set_preset_mode` mit „Fester Vorlauf“ oder die Auswahl „Betriebsart“ (`manual`) verwenden.

---

## Modbus TCP und RTU

Beide Schnittstellen haben **dieselben Register** und können gleichzeitig aktiv sein. Wird über eine davon ein Wert geschrieben, ist er sofort auch über die andere sichtbar.

| | Modbus TCP | Modbus RTU |
|---|---|---|
| Verbindung | WLAN, Port 502 (einstellbar) | RS485 über MAX485 |
| Adresse | jede Unit-ID | eingestellte Slave-Adresse 1–247 (Standard 1) |
| Werkseinstellung | ein | aus |

### Modbus RTU anschließen

| MAX485 | ESP32-Relaisplatine |
|---|---|
| RO | GPIO 16 (UART2 RX) |
| DI | GPIO 17 (UART2 TX) |
| DE und /RE (gebrückt) | GPIO 18 |
| VCC / GND | 3,3 V / GND |
| A / B | RS485-Bus (A an A, B an B) |

- **Automatische Richtungsumschaltung:** Bei Modulen, die selbst umschalten (ohne DE/RE-Anschluss), die Option *DE/RE-Umschaltung über GPIO 18* ausschalten.
- **Abschlusswiderstände:** Am ersten und letzten Gerät des Busses gehört je ein 120-Ω-Abschluss zwischen A und B.
- **Einstellungen** unter *Netzwerk → Modbus RTU & OTA-Update*, wirksam nach dem automatischen Neustart:
  - **Baudrate:** 1200–115200, Standard 9600.
  - **Datenformat:** 8N1, 8E1 oder 8O1, Standard 8N1. Die Modbus-Norm sieht 8E1 vor, viele Geräte nutzen 8N1. Es muss zum Master passen.

Modbus RTU nutzt die **zweite Hardware-Schnittstelle (UART2)** des ESP32. Die erste Schnittstelle bleibt für Meldungen und das Programmieren frei; das RS485-Modul muss beim Aufspielen nicht abgezogen werden.

### Register

Temperaturen werden als vorzeichenbehaftete 16-Bit-Werte übertragen. **−32768 (0x8000) bedeutet ungültig oder Fühlerfehler.**

**Holding-Register** (FC03 lesen, FC06/FC16 schreiben):

| Adr. | Parameter | Skalierung |
|---|---|---|
| 0 | Betriebsart (0 aus, 1 auto, 2 fester Sollwert, 3 hand, 4 kühlen, 5 heizen/kühlen automatisch, 6 Raumtemperatur) | 1 |
| 1 | Raumsollwert °C | ×10 |
| 2 | Heizkurve Neigung | ×100 |
| 3 | Niveau / Parallelverschiebung K | ×10 |
| 4 | fester Vorlauf-Sollwert °C | ×10 |
| 5 | Mischerstellung Handbetrieb % | ×10 |
| 6 | Vorlauf min °C | ×10 |
| 7 | Vorlauf max °C | ×10 |
| 8 | Heizgrenze °C | ×10 |
| 9 | Absenkbetrieb / Zeitprogramm aktiv (0/1) | 1 |
| 10 | Absenkung K | ×10 |
| 11 | Rücklauf max °C (0 = aus) | ×10 |
| 12 | Kp %/K | ×100 |
| 13 | Tn s | 1 |
| 14 | Kommando: 1 Kalibrieren, 2 Neustart, 3 Fühlersuche, 4 Profil Radiator, 5 Profil Fußboden, 6 Profil GLW8 laden, 7 aktuelle Werte als GLW8 speichern; Handschaltung: 10 Mischer AUF, 11 Mischer ZU, 12 Mischer Stopp, 13 Mischer freigeben, 14 Pumpe EIN, 15 Pumpe AUS, 16 Pumpe Automatik (wird danach auf 0 gesetzt) | – |
| 15 | Raumaufschaltung K/K | ×10 |
| 16 | Raumabschaltung K (0 = aus) | ×10 |
| 17 | externe Raumtemperatur °C | ×10 |
| 18 | Raumtemperatur berücksichtigen (0/1) | 1 |
| 19 | externer Vorlauf °C (spätestens alle 2 min schreiben) | ×10 |
| 20 | externer Rücklauf °C (spätestens alle 2 min schreiben) | ×10 |
| 21 | externe Außentemperatur °C | ×10 |
| 22 | Raumsollwert Kühlen °C | ×10 |
| 23 | Kühlgrenze °C | ×10 |
| 24 | externe Raumfeuchte % (Taupunktschutz) | ×10 |
| 25 | Relais K4 Funktion: 0 Aus, 1 Wärmeanforderung, 2 Heizen/Kühlen, 3 Sammelstörmeldung, 4 Zirkulation, 5 Entfeuchter, 6 frei schaltbar | 1 |
| 26 | Relais K4 frei schalten (0/1, wirkt bei Funktion 6) | 1 |

Die Register 17, 19–21 und 24 sind Eingänge für externe Werte. Register 24 (Feuchte) wird immer übernommen. Sie werden nur verwendet, wenn beim jeweiligen Fühler die Herkunft „MQTT / Modbus“ oder „Pin, bei Ausfall MQTT“ eingestellt ist, und müssen regelmäßig geschrieben werden (siehe [Zeitüberwachung](#zeitüberwachung)).

**Input-Register** (FC04), zusätzlich gespiegelt als Holding-Register 100–139 für Master, die nur FC03 können:

| Adr. | Wert | Skalierung |
|---|---|---|
| 0 | Vorlauf °C | ×10 |
| 1 | Rücklauf °C | ×10 |
| 2 | Außen °C | ×10 |
| 3 | Außen gedämpft °C | ×10 |
| 4 | aktiver Vorlauf-Sollwert °C | ×10 |
| 5 | Heizkurven-Sollwert °C | ×10 |
| 6 | Mischerstellung % | ×10 |
| 7 | Pumpe (0/1) | 1 |
| 8 | Statusbits | – |
| 9 | Betriebsart | 1 |
| 10 | WLAN-Signal dBm | 1 |
| 11 | Laufzeit in Minuten | 1 |
| 12 | Raumtemperatur °C | ×10 |
| 13 | Raumsollwert inkl. Absenkung °C | ×10 |
| 14 | Stellvorgänge seit Neustart, oberes Wort | – |
| 15 | Stellvorgänge seit Neustart, unteres Wort (gesamt = R14 × 65536 + R15) | – |
| 16 | Pumpenlaufzeit seit Neustart in s, oberes Wort | – |
| 17 | Pumpenlaufzeit seit Neustart in s, unteres Wort (gesamt = R16 × 65536 + R17) | – |
| 18 | Pumpenstarts seit Neustart, oberes Wort | – |
| 19 | Pumpenstarts seit Neustart, unteres Wort | – |
| 20 / 21 | Stellvorgänge **gesamt**, oberes / unteres Wort | – |
| 22 / 23 | Pumpenlaufzeit **gesamt** in s, oberes / unteres Wort | – |
| 24 / 25 | Pumpenstarts **gesamt**, oberes / unteres Wort | – |
| 26 / 27 | Neustarts **gesamt**, oberes / unteres Wort | – |
| 28 | Grund des letzten Starts: 0 Einschalten, 1 Software, 2 Absturz/Watchdog, 3 Reset-Taste | – |
| 29 | Neustarts durch Absturz/Watchdog gesamt | – |
| 30 | Kühlkurven-Sollwert °C | ×10 |
| 31 | Taupunkt im Raum °C | ×10 |
| 32 | Raumfeuchte % | ×10 |
| 33 | Heizkreis: 0 ruht, 1 heizt, 2 kühlt | 1 |
| 34 | Statusbits 2: Bit 0 Kühlbetrieb, 1 über Kühlgrenze, 2 Kühl-Soll auf Untergrenze angehoben, 3 Taupunktalarm, 4 gültige Raumfeuchte, 5 Raumregelung Heizpause, 6 Raumregelung Ersatzbetrieb (kein Raumwert), 7 Relais K4 Funktion aktiv, 8 Relais K4 angezogen | – |
| 35 | Relais K4: Bit 0 Funktion aktiv, Bit 1 Relais angezogen, Bit 8–15 Funktion (0–6) | – |
| 36 / 37 | Relais K4: Einschaltungen seit Neustart, oberes / unteres Wort | – |
| 38 / 39 | Relais K4: Laufzeit seit Neustart in s, oberes / unteres Wort | – |

**Statusbits (Register 8):**

| Bit | Bedeutung | Bit | Bedeutung |
|---|---|---|---|
| 0 | Heizbetrieb | 8 | Rücklaufbegrenzung aktiv |
| 1 | Absenkbetrieb gerade aktiv | 9 | Regelung aktiv |
| 2 | Frostschutz | 10 | MQTT verbunden |
| 3 | Fehler Vorlauf | 11 | Access-Point aktiv |
| 4 | Fehler Rücklauf | 12 | Raumwert gestört |
| 5 | Fehler Außen | 13 | Raumabschaltung aktiv |
| 6 | Übertemperatur | 14 | Raumtemperatur wird berücksichtigt |
| 7 | Kalibrierung läuft | 15 | Raumwert kommt aktuell von extern |

---

## Firmware-Update über WLAN (OTA)

Nach dem ersten Upload per USB brauchst du kein Kabel mehr. Die gespeicherten Einstellungen bleiben bei jedem Update erhalten.

### 1. Direkt aus der Arduino IDE (ArduinoOTA)

1. PC und Regler müssen im selben Netz sein, der Regler im WLAN verbunden und nicht im Access-Point-Modus.
2. In der Arduino IDE erscheint unter *Werkzeuge → Port → Netzwerk-Ports* der Eintrag **„heizungsregler at 192.168.x.x“** (Port 3232). Fehlt er, die IDE neu starten. Die Firewall muss mDNS (UDP 5353) und eingehende Verbindungen für die IDE zulassen.
3. Diesen Port wählen und wie gewohnt auf *Hochladen* klicken.
4. Ist ein Web-Passwort gesetzt, fragt die IDE danach.

Während des Uploads (ca. 20–40 s) setzt der Regelzyklus aus (Fühler werden weiter gelesen). Das gilt seit 2.4.0 genauso für das Update über den Browser:
- **Vorher:** Der Mischer wird angehalten, die Pumpe behält ihren Zustand, ungespeicherte Einstellungen werden gesichert.
- **Danach:** Der Regler startet neu und kalibriert den Mischer.

OTA lässt sich unter *Netzwerk → Modbus RTU & OTA-Update* abschalten.

**Letztes OTA-Update:** Im Reiter *System* steht bei *OTA-Update* das Datum des letzten erfolgreichen Updates über WLAN, der Weg und die Version, von der aus aktualisiert wurde, z. B. „letztes Update 17.10.2026 03:12 (aus Arduino IDE, vorher Version 1.23.0)“. Der Regler speichert das in `/counters.json`, deshalb übersteht es den Neustart und auch *Werkseinstellungen*. Hatte der Regler keine Internet-Uhrzeit, steht dort „Zeit unbekannt“.

**Anzahl der OTA-Updates:** Die Karte *Zähler* im Reiter *System* zeigt in der Zeile **OTA-Updates** die Zahl aller erfolgreichen Updates über WLAN, darunter getrennt nach *Arduino IDE* und *Browser*. Eine Spalte „seit Neustart“ gibt es dafür nicht, weil der Regler nach jedem Update neu startet.
- **Arduino IDE und Browser (`/update`):** gezählt, sobald das Update erfolgreich abgeschlossen ist, kurz vor dem Neustart – mit Datum und Uhrzeit (seit 2.4.0 auch beim Browser).
- **Browser-Update aus einer Firmware vor 2.4.0:** Der Regler erkennt beim nächsten Start, dass eine andere Firmware läuft (Version und Übersetzungszeitpunkt) und der Neustart vom Update kam. Datum und Uhrzeit trägt er nach, sobald die Internet-Uhrzeit vorliegt.
- **USB-Kabel:** ist kein OTA-Update und wird nicht gezählt; im Ereignisprotokoll steht „Firmware … per USB aufgespielt“.
- Die Zählung beginnt mit dieser Version. *Gesamtzähler zurücksetzen* setzt auch die OTA-Zähler auf 0.

### 2. Als Datei über den Browser

1. In der Arduino IDE *Sketch → Kompilierte Binärdatei exportieren*.
2. Im Reiter *System* auf *Firmware-Update (Datei hochladen)* klicken (`http://<ip>/update`).
3. Mit **Datei auswählen …** die `.bin`-Datei wählen. Die Seite liest sofort die Versionskennung aus der Datei und zeigt z. B. „Datei Heizungsregler_ESP32.ino.bin (1012 KB): Version 2.4.0 – neuer“ (oder „gleich“, „**älter**“).
4. **Datei hochladen** – nach einer Rückfrage (bei einer älteren Version mit ausdrücklichem Hinweis) zeigt ein Balken die Übertragung. Der Regler startet danach neu; die Seite lädt sich selbst neu, sobald er mit der neuen Version antwortet. Die Einstellungen bleiben erhalten.

**Schutz vor falschen Dateien (seit 2.4.0):** Der Regler nimmt nur Dateien an, die die Kennung `GLW8:HZR3P32:FW=` enthalten. Eine Firmware von Version 1 (ESP8266), ein anderes Programm oder eine beschädigte Datei wird schon beim Auswählen abgewiesen („… ist keine Firmware für dieses Gerät“) und – falls die Seite umgangen wird – vom Regler selbst nach den ersten Kilobyte abgebrochen, bevor etwas gültig wird. Die laufende Firmware bleibt dann unverändert. Ein anderes Programm lässt sich weiterhin per USB oder aus der Arduino IDE aufspielen.

Die Update-Seite ist deutschsprachig und in den Farben der Weboberfläche gestaltet (hell und dunkel), mit *Zurück*-Schaltfläche und Urheberhinweis. Auf ihr steht zusätzlich die Karte **Neue Firmware von www.glw8.at** (siehe 3.). Seite, Prüfung und Schreiben in den Flash stammen aus dem eigenen Modul **GlwOta** (siehe [Modul GlwOta](#modul-glwota--firmware-update-für-andere-projekte)).

Das funktioniert auch im Access-Point-Modus. Mit gesetztem Web-Passwort meldest du dich als `admin` an.

### 3. Neue Firmware von www.glw8.at (Knopf)

Im Reiter *System*, Karte **Firmware von glw8.at** – dieselben Knöpfe stehen auch auf der Seite `/update`:

1. **Auf neue Firmware prüfen:** Der Browser ruft
   `https://www.glw8.at/OTA/Heizungsregler3P/Heizungsregler-ino-esp32.php`
   auf. Das PHP-Skript liefert die aktuelle Firmware-Datei aus (siehe *Einrichtung auf dem Server*). Der Browser liest den Anfang und sucht darin die **Versionskennung**, z. B. `GLW8:HZR3P32:FW=1.32.0;`. Sie steht in jeder Firmware ab 1.32.0 nach etwa 4 KB; danach bricht der Browser den Download ab. Angezeigt wird z. B. „Installiert: 1.32.0 · auf www.glw8.at: 1.33.0 – neuere Version verfügbar“, „aktuell“ oder eine Fehlermeldung.
2. **Jetzt installieren** erscheint nur, wenn die Version auf dem Server **neuer** ist. Nach einer Rückfrage lädt der Browser die ganze Datei, prüft Größe und Versionskennung und schickt sie an `/update` – genau wie beim Hochladen von Hand. Ein Balken zeigt Download und Übertragung. Der Regler schreibt die Firmware in den Flash und startet neu; die Seite lädt sich danach selbst neu, sobald der Regler mit der neuen Version antwortet. Das Update zählt unter *OTA-Updates* als „Browser“.

**Wichtig:** Den Abruf von glw8.at erledigt **der Browser**, nicht der Regler. Der Regler braucht dafür kein HTTPS, keinen zusätzlichen Speicher und keine Internetverbindung – das Gerät mit dem Browser (PC, Handy) schon. Mit gesetztem Web-Passwort muss man angemeldet sein.

**Einrichtung auf dem Server (einmalig):** Browser lesen Daten von einer anderen Adresse nur, wenn der Server das erlaubt (CORS). Ohne diese Freigabe meldet die Prüfung „www.glw8.at nicht erreichbar … oder der Server erlaubt den Abruf nicht (CORS)“.

Seit Version 1.34.2 fragt die Weboberfläche das **PHP-Skript** `Heizungsregler-ino-esp32.php` ab. Es liefert die passende `.bin`-Datei aus. Damit Prüfung und Installation aus dem Browser funktionieren, muss die Antwort des Skripts:

- die Kopfzeile `Access-Control-Allow-Origin: *` enthalten (sonst blockiert der Browser den Abruf),
- den Inhalt der `.bin`-Datei **unverändert** enthalten, ohne zusätzliche Zeichen davor oder danach (die Weboberfläche sucht darin die Versionskennung und prüft die Datei vor dem Installieren),
- möglichst `Content-Length` angeben (für Größenprüfung und Fortschrittsbalken) und nicht zwischengespeichert werden (`Cache-Control: no-cache`), damit eine neue Version sofort gefunden wird.

Der Browser hängt beim Abruf `?t=<Zeitstempel>` an die Adresse, um Zwischenspeicher zu umgehen; das Skript kann diesen Zusatz ignorieren.

**Das Skript** liegt dem Paket als `Server/Heizungsregler-ino-esp32.php` bei und gehört in den Ordner `/OTA/Heizungsregler3P/`. Alle Dateinamen leitet es aus seinem eigenen Namen ab:

| Datei | Zweck |
|---|---|
| `Heizungsregler-ino-esp32.php` | das Skript |
| `glw8_Heizungsregler-ino-esp32_dl.bin` | die Firmware, die ausgeliefert wird (`glw8_` + Skriptname + `_dl.bin`) |
| `Heizungsregler-ino-esp32.log` | Protokoll im Format **JSON Lines**: je Aufruf eine Zeile mit einem JSON-Objekt (legt das Skript selbst an) |
| `Heizungsregler-ino-esp32.log.1` | vorheriges Protokoll: Ab 2 MB wird das Protokoll umbenannt und neu begonnen |

Für ein weiteres Gerät genügt eine Kopie des Skripts unter neuem Namen. Fehlt die Firmware-Datei, antwortet es mit 404 und nennt den erwarteten Namen. Das Skript braucht Schreibrecht im Ordner, sonst fehlt nur das Protokoll; die Auslieferung funktioniert trotzdem.

**Protokoll.** Ab Firmware 1.34.3 schickt die Weboberfläche beim Abruf zusätzliche Angaben mit, damit das Protokoll den ganzen Ablauf zeigt:

| Parameter | Inhalt |
|---|---|
| `a` | `check` = Versionsprüfung, `install` = Download zum Installieren, `report` = Rückmeldung zum Ablauf (es wird nichts ausgeliefert) |
| `v` | auf dem Regler installierte Version |
| `h` | Hostname des Reglers |
| `n` | Version, die installiert werden soll |
| `r` | bei `report`: `uploaded` (Regler hat die Firmware angenommen), `ok` (Regler läuft mit der neuen Version), `timeout` (meldet sich nach 120 s nicht mit der neuen Version), `error` (Fehler bei der Installation), `checkerror` (Fehler bei der Prüfung, z. B. keine Versionskennung) |
| `m` | Fehlermeldung, wie sie die Weboberfläche anzeigt |
| `pin` | Freigabe-PIN (ab 1.34.4), siehe unten |
| `t` | Zeitstempel gegen Zwischenspeicher |

**Format: JSON Lines.** Jede Zeile ist ein vollständiges JSON-Objekt (UTF-8). Alle Zeilen haben **dieselben Felder in derselben Reihenfolge**; was bei einem Aufruf nicht vorkommt, steht als `null`. Zahlen und Ja/Nein-Werte sind echte JSON-Zahlen bzw. `true`/`false`. Damit lässt sich das Protokoll direkt maschinell auswerten (Beispiele unten). Eine abgebrochene Schreiboperation beschädigt höchstens die eine betroffene Zeile.

| Feld | Inhalt |
|---|---|
| `zeit` | Zeitpunkt des Aufrufs, ISO 8601 mit Millisekunden und Zeitzone (Wien), z. B. `2026-10-25T09:24:10.780+01:00` |
| `format` | Formatversion der Zeile (derzeit `1`); ändert sich, falls Felder umgebaut werden |
| `skript` | Name des Skripts ohne `.php` (unterscheidet mehrere Geräte, wenn Protokolle zusammengeführt werden) |
| `ip`, `port` | Adresse und Port des Browsers |
| `weitergeleitet` | Inhalt von `X-Forwarded-For` (ursprüngliche Adresse hinter Proxy oder CDN) |
| `methode`, `protokoll`, `https` | `GET`/`HEAD`/`OPTIONS`, HTTP-Version, `true` bei HTTPS |
| `host`, `adresse` | aufgerufener Host und Pfad mit Parametern (PIN als `****`) |
| `origin` | Adresse der Regler-Weboberfläche im Heimnetz, z. B. `http://192.168.1.50` |
| `referer`, `browser`, `sprache` | Referer (PIN als `****`), Browser-Kennung, bevorzugte Sprachen |
| `aktion` | `check`, `install`, `report` oder `null` (Firmware vor 1.34.3 bzw. direkter Aufruf) |
| `regler`, `installiert`, `ziel` | Hostname des Reglers, installierte Version, Version, die installiert werden soll |
| `pin` | `richtig`, `falsch` oder `fehlt` |
| `datei`, `groesse`, `dateidatum` | ausgelieferte Datei, Größe in Byte, Änderungszeitpunkt (ISO 8601) |
| `version_server` | Versionskennung in der Datei (`null` = keine gefunden) |
| `vergleich` | Server-Version gegenüber der installierten: `neuer`, `gleich`, `aelter` |
| `status` | HTTP-Status der Antwort: `200`, `204`, `403`, `404` |
| `gesendet`, `anteil` | ins Netz gegebene Bytes und Anteil an der Dateigröße in % |
| `ergebnis` | siehe nächste Tabelle |
| `rueckmeldung`, `meldung` | bei `aktion` = `report`: `uploaded`, `ok`, `timeout`, `error`, `checkerror` und die Fehlermeldung der Weboberfläche |
| `dauer_ms` | Bearbeitungsdauer im Skript in Millisekunden (bei abgewiesener PIN inkl. 2 s Wartezeit) |

| `ergebnis` | Bedeutung |
|---|---|
| `vollstaendig` | Firmware ganz gesendet |
| `pruefung_beendet` | Versionsprüfung: Der Browser hat nach der Versionskennung abgebrochen – normal |
| `abgebrochen` | Download vorzeitig abgebrochen (z. B. Seite geschlossen, Verbindung weg) |
| `abgewiesen` | PIN fehlt oder ist falsch, nichts ausgeliefert (403) |
| `datei_fehlt` | Firmware-Datei nicht vorhanden (404) |
| `rueckmeldung` | Rückmeldung der Weboberfläche protokolliert (204) |
| `nur_kopfzeilen` | `HEAD`-Anfrage, nur Kopfzeilen gesendet |
| `cors_vorabanfrage` | `OPTIONS`-Vorabanfrage des Browsers beantwortet |

Beispiel: der Download zum Installieren (Werte erfunden, zur Lesbarkeit umbrochen – im Protokoll steht das Objekt in einer Zeile):

```json
{"zeit":"2026-10-25T09:24:10.780+01:00","format":1,"skript":"Heizungsregler-ino-esp32",
 "ip":"203.0.113.25","port":51830,"weitergeleitet":null,"methode":"GET","protokoll":"HTTP/2.0","https":true,
 "host":"www.glw8.at","adresse":"/OTA/Heizungsregler3P/Heizungsregler-ino-esp32.php?a=install&v=1.34.4&h=heizungsregler&n=1.35.0&pin=****&t=1792916650774",
 "origin":"http://192.168.1.50","referer":null,"browser":"Mozilla/5.0 (Windows NT 10.0; Win64; x64) … Chrome/141.0.0.0 Safari/537.36","sprache":"de-AT,de;q=0.9",
 "aktion":"install","regler":"heizungsregler","installiert":"1.34.4","ziel":"1.35.0","pin":"richtig",
 "datei":"glw8_Heizungsregler-ino-esp32_dl.bin","groesse":512384,"dateidatum":"2026-10-25T08:50:12+01:00",
 "version_server":"1.35.0","vergleich":"neuer","status":200,"gesendet":512384,"anteil":100,
 "ergebnis":"vollstaendig","rueckmeldung":null,"meldung":null,"dauer_ms":1210}
```

Ein vollständiger Ablauf ergibt vier Zeilen: `aktion` `check` (`ergebnis` `pruefung_beendet`) → `install` (`vollstaendig`) → `report` mit `rueckmeldung` `uploaded` → `report` mit `ok`. Fehlt die letzte, ist der Regler nach dem Update nicht mit der neuen Version zurückgekommen oder die Seite wurde vorher geschlossen.

**Auswerten**, zum Beispiel:

```bash
# alle Installationen, die erfolgreich abgeschlossen wurden
jq -c 'select(.rueckmeldung=="ok") | {zeit, regler, ziel}' Heizungsregler-ino-esp32.log
# welche Versionen melden die Regler bei der Prüfung (Anzahl je Version)
jq -r 'select(.aktion=="check" and .pin=="richtig") | .installiert' Heizungsregler-ino-esp32.log | sort | uniq -c
# abgewiesene Zugriffe (falsche oder fehlende PIN)
jq -c 'select(.ergebnis=="abgewiesen") | {zeit, ip, pin, browser}' Heizungsregler-ino-esp32.log
```

In **Excel** öffnet *Daten → Daten abrufen → Aus Datei → Aus JSON* nur ein einzelnes JSON-Dokument. Für JSON Lines geht man über *Aus Text/CSV*, wählt als Trennzeichen „Keines“ und wandelt die Spalte im Power-Query-Editor mit *Transformieren → Analysieren → JSON* um; danach lassen sich alle Felder als Spalten erweitern.

`gesendet` zählt die Bytes, die das Skript ins Netz gegeben hat. Bricht der Browser ab (bei der Prüfung absichtlich), liegt der Wert wegen der Puffer von Webserver und Netz meist deutlich über dem, was der Browser tatsächlich gelesen hat. Aufrufe mit `aktion` = `null` stammen von älteren Firmware-Versionen oder von direkten Aufrufen im Browser.

**Freigabe-PIN.** Seit 1.34.4 liefert das Skript nur aus, wenn die Adresse die richtige PIN enthält (`pin=2202`). Die Weboberfläche schickt sie bei jedem Abruf und jeder Rückmeldung automatisch mit (Konstante `FW_PIN` in `web_ui.h`); im Skript steht sie in der Konstante `PIN`. Beide Werte müssen übereinstimmen.

- **Ohne oder mit falscher PIN** antwortet das Skript nach 2 Sekunden Wartezeit (bremst das Durchprobieren) mit **403** und dem Text „Zugriff verweigert: PIN fehlt oder ist falsch“. Es liefert nichts aus. Im Protokoll steht `"pin":"fehlt"` bzw. `"pin":"falsch"` mit `"ergebnis":"abgewiesen"`. Die Weboberfläche meldet „Der Server verweigert die Firmware (403) – PIN fehlt oder ist falsch.“
- **Mit richtiger PIN** steht `"pin":"richtig"` im Protokoll, und alles läuft wie oben beschrieben.
- **Die PIN selbst erscheint nie im Protokoll**: In *Adresse* und *Referer* wird sie durch `****` ersetzt. In den Zugriffsprotokollen des Webservers (z. B. Apache `access.log`) steht die Adresse allerdings vollständig.
- **PIN ändern:** in `web_ui.h` (`FW_PIN`) und im Skript (`PIN`) gleich ändern. Da die Regler die PIN mit der Firmware erhalten, erst eine Firmware mit der neuen PIN verteilen und danach das Skript umstellen – sonst kommen ältere Regler nicht mehr an das Update.
- **Schutzwirkung:** Die PIN hält zufällige Abrufe, Suchmaschinen und Fremde ohne die Adresse fern. Sie ist aber kein starker Schutz: Sie steht im Klartext in der Weboberfläche jedes Reglers (Seitenquelltext) und in jeder Firmware-Datei. Firmware vor 1.34.4 sendet keine PIN und bekommt kein Update mehr über den Knopf; solche Regler einmal von Hand über *Firmware-Update (Datei hochladen)* aktualisieren.

**Datenschutz:** Das Protokoll enthält IP-Adressen. Es liegt im selben Ordner wie das Skript und wäre ohne Schutz über `https://www.glw8.at/OTA/Heizungsregler3P/Heizungsregler-ino-esp32.log` abrufbar. Bei Apache sperrt eine `.htaccess` im Ordner den Zugriff:

```apache
<FilesMatch "\.log(\.1)?$">
  Require all denied
</FilesMatch>
```

Bei nginx entsprechend `location ~ \.log(\.1)?$ { deny all; }`.

```php
<?php
// ============================================================================
// GLW8 Heizungsregler – Firmware-Auslieferung mit Protokoll
// ============================================================================
// Abgerufen von der Weboberfläche des Reglers (Reiter System, Karte
// "Firmware von glw8.at"). Der Abruf kommt vom BROWSER, nicht vom Regler.
//
// Dateien (alle im selben Ordner, Namen aus dem Namen dieses Skripts):
//   Heizungsregler-ino-esp32.php           dieses Skript
//   glw8_Heizungsregler-ino-esp32_dl.bin   Firmware, die ausgeliefert wird
//   Heizungsregler-ino-esp32.log           Protokoll im Format JSON Lines:
//                                            je Aufruf eine Zeile mit einem JSON-Objekt
//   Heizungsregler-ino-esp32.log.1         vorheriges Protokoll (ab 2 MB)
// Für ein weiteres Gerät das Skript einfach unter neuem Namen kopieren.
//
// Parameter, die die Weboberfläche ab Firmware 1.34.3 mitschickt:
//   a = check    Versionsprüfung (Browser liest nur den Anfang und bricht ab)
//       install  Download zum Installieren (ganze Datei)
//       report   Rückmeldung zum Ablauf, es wird nichts ausgeliefert:
//                r = uploaded | ok | timeout | error | checkerror, m = Meldung
//   v = installierte Version, h = Hostname des Reglers,
//   n = Version, die installiert werden soll, t = Zeitstempel (gegen Caches)
//   pin = Freigabe-PIN (ab Firmware 1.34.4). Ohne richtige PIN antwortet das
//         Skript mit 403 und liefert nichts aus; der Versuch wird protokolliert.
//         Im Protokoll steht die PIN nie im Klartext.
//
// Wichtig: vor dem PHP-Starttag darf nichts stehen, und am Ende steht bewusst
// kein PHP-Endtag – sonst gelangen zusätzliche Zeichen in die Firmware-Datei.
// (Das Endtag darf auch in keinem Kommentar vorkommen: PHP beendet dort den Code.)

$t0   = microtime(true);
date_default_timezone_set('Europe/Vienna');
$name = basename(__FILE__, '.php');
$bin  = 'glw8_' . $name . '_dl.bin';
$f    = __DIR__ . '/' . $bin;
$log  = __DIR__ . '/' . $name . '.log';
const LOG_MAX = 2 * 1024 * 1024;           // ab 2 MB wird in .log.1 umbenannt
const PIN     = '2202';                     // Freigabe-PIN; muss zu FW_PIN in web_ui.h passen
const LOG_FMT = 1;                          // Formatversion der Protokollzeilen (Feld "format")

ignore_user_abort(true);                    // auch nach Abbruch des Browsers zu Ende loggen
@set_time_limit(0);
while (ob_get_level()) ob_end_clean();      // ohne Ausgabepuffer: Bytes gehen sofort raus

// Text auf eine Länge begrenzen und Steuerzeichen entfernen; leer -> null
function txt($v, $len) {
    $v = preg_replace('/[\x00-\x1F\x7F]/', ' ', (string)$v);
    $v = mb_substr($v, 0, $len);
    return $v === '' ? null : $v;
}
function par($k, $len = 40) { return txt(isset($_GET[$k]) ? $_GET[$k] : '', $len); }
function srv($k, $len = 300) { return txt(isset($_SERVER[$k]) ? $_SERVER[$k] : '', $len); }
// PIN in Adressen fürs Protokoll unkenntlich machen
function noPin($u) { return $u === null ? null : preg_replace('/([?&]pin=)[^&]*/i', '$1****', $u); }

$act  = par('a', 12);
$inst = par('v', 20);

// ----------------------------------------------------------------------------
// Protokollzeile: immer dieselben Felder in derselben Reihenfolge (nicht
// vorhandene Werte = null). Bedeutung der Felder siehe README.
// ----------------------------------------------------------------------------
$https = !empty($_SERVER['HTTPS']) && $_SERVER['HTTPS'] !== 'off';
$L = [
    'zeit'           => null,     // wird in writeLog() gesetzt (ISO 8601, Millisekunden, mit Zeitzone)
    'format'         => LOG_FMT,
    'skript'         => $name,
    // Verbindung
    'ip'             => srv('REMOTE_ADDR', 60),
    'port'           => isset($_SERVER['REMOTE_PORT']) ? (int)$_SERVER['REMOTE_PORT'] : null,
    'weitergeleitet' => srv('HTTP_X_FORWARDED_FOR', 200),   // bei Proxy/CDN: ursprüngliche IP(s)
    'methode'        => srv('REQUEST_METHOD', 10),
    'protokoll'      => srv('SERVER_PROTOCOL', 20),
    'https'          => $https,
    'host'           => srv('HTTP_HOST', 100),
    'adresse'        => noPin(srv('REQUEST_URI', 400)),
    'origin'         => srv('HTTP_ORIGIN', 200),            // Adresse der Regler-Weboberfläche, z. B. http://192.168.1.50
    'referer'        => noPin(srv('HTTP_REFERER', 300)),
    'browser'        => srv('HTTP_USER_AGENT', 400),
    'sprache'        => srv('HTTP_ACCEPT_LANGUAGE', 100),
    // Regler (Parameter der Weboberfläche)
    'aktion'         => $act,                              // check | install | report | null (ältere Firmware / direkter Aufruf)
    'regler'         => par('h', 40),
    'installiert'    => $inst,
    'ziel'           => par('n', 20),
    'pin'            => null,                              // richtig | falsch | fehlt
    // Firmware-Datei
    'datei'          => null,
    'groesse'        => null,                              // Byte
    'dateidatum'     => null,
    'version_server' => null,
    'vergleich'      => null,                              // neuer | gleich | aelter (Server gegenüber installiert)
    // Ablauf
    'status'         => null,                              // HTTP-Status der Antwort
    'gesendet'       => null,                              // Byte, die das Skript ins Netz gegeben hat
    'anteil'         => null,                              // gesendet in % der Dateigröße
    'ergebnis'       => null,                              // siehe README (vollstaendig, pruefung_beendet, abgebrochen, …)
    'rueckmeldung'   => null,                              // bei aktion=report: uploaded | ok | timeout | error | checkerror
    'meldung'        => null,                              // Fehlermeldung der Weboberfläche
    'dauer_ms'       => null,
];
function L($k, $v) { global $L; $L[$k] = $v; }
function writeLog() {
    global $L, $log, $t0;
    $L['zeit']     = date('Y-m-d\TH:i:s', (int)$t0) . sprintf('.%03d', (int)(fmod($t0, 1) * 1000)) . date('P', (int)$t0);
    $L['dauer_ms'] = (int)round((microtime(true) - $t0) * 1000);
    $line = json_encode($L, JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES | JSON_INVALID_UTF8_SUBSTITUTE) . "\n";
    if (@filesize($log) > LOG_MAX) @rename($log, $log . '.1');
    @file_put_contents($log, $line, FILE_APPEND | LOCK_EX);
}

header('Access-Control-Allow-Origin: *');                     // Abruf aus der Weboberfläche erlauben (CORS)
header('Cache-Control: no-cache, no-store, must-revalidate'); // neue Version sofort sichtbar

// Vorabanfrage des Browsers (kommt normalerweise nicht, schadet aber nicht)
if ($L['methode'] === 'OPTIONS') {
    header('Access-Control-Allow-Methods: GET, HEAD, OPTIONS');
    http_response_code(204);
    L('status', 204); L('ergebnis', 'cors_vorabanfrage');
    writeLog(); exit;
}

// Freigabe-PIN prüfen (gilt für Auslieferung und Rückmeldungen)
$pin = isset($_GET['pin']) ? (string)$_GET['pin'] : '';
if (!hash_equals(PIN, $pin)) {
    sleep(2);                               // bremst das Durchprobieren von PINs
    http_response_code(403);
    header('Content-Type: text/plain; charset=utf-8');
    echo 'Zugriff verweigert: PIN fehlt oder ist falsch';
    L('pin', $pin === '' ? 'fehlt' : 'falsch');
    L('status', 403); L('ergebnis', 'abgewiesen');
    writeLog(); exit;
}
L('pin', 'richtig');

// Rückmeldung der Weboberfläche zum Ablauf der Installation: nur protokollieren
if ($act === 'report') {
    L('rueckmeldung', par('r', 20));
    L('meldung', par('m', 200));
    http_response_code(204);
    L('status', 204); L('ergebnis', 'rueckmeldung');
    writeLog(); exit;
}

// --- Firmware-Datei ---
L('datei', $bin);
if (!is_file($f) || !is_readable($f)) {
    http_response_code(404);
    header('Content-Type: text/plain; charset=utf-8');
    echo 'Firmware nicht gefunden: ' . $bin;
    L('status', 404); L('ergebnis', 'datei_fehlt');
    writeLog(); exit;
}
$size = filesize($f);
L('groesse', $size);
L('dateidatum', date('Y-m-d\TH:i:sP', filemtime($f)));
// Version aus der Kennung "GLW8:HZR3P32:FW=x.y.z;" (steht in den ersten KB)
$head = (string)@file_get_contents($f, false, null, 0, 131072);
$sv = preg_match('/GLW8:HZR3P32:FW=([0-9]+(?:\.[0-9]+){1,3});/', $head, $m) ? $m[1] : null;
L('version_server', $sv);
if ($sv !== null && $inst !== null && preg_match('/^[0-9]+(\.[0-9]+)*$/', $inst)) {
    $c = version_compare($sv, $inst);
    L('vergleich', $c > 0 ? 'neuer' : ($c === 0 ? 'gleich' : 'aelter'));
}

header('Content-Type: application/octet-stream');
header('Content-Length: ' . $size);                           // für Größenprüfung und Fortschrittsbalken
header('Content-Disposition: attachment; filename="' . $bin . '"');
http_response_code(200);
L('status', 200);

if ($L['methode'] === 'HEAD') { L('ergebnis', 'nur_kopfzeilen'); writeLog(); exit; }

// In Stücken senden und mitzählen, wie viel tatsächlich hinausgeht.
// Bei der Versionsprüfung bricht der Browser nach den ersten KB absichtlich ab.
$sent = 0; $aborted = false;
$h = fopen($f, 'rb');
while (!feof($h)) {
    $buf = fread($h, 8192);
    if ($buf === false || $buf === '') break;
    echo $buf; flush();
    if (connection_aborted()) { $aborted = true; break; }
    $sent += strlen($buf);
}
fclose($h);

L('gesendet', $sent);
L('anteil', $size ? (int)round($sent * 100 / $size) : null);
if (!$aborted && $sent === $size) L('ergebnis', 'vollstaendig');
elseif ($act === 'check')         L('ergebnis', 'pruefung_beendet');   // normal: Browser bricht nach der Versionskennung ab
else                              L('ergebnis', 'abgebrochen');
writeLog();
```

Vor `<?php` darf nichts stehen (auch keine Leerzeile), und am Ende steht bewusst kein `?>`: Jedes zusätzliche Zeichen würde in die ausgelieferte Firmware geraten, und Prüfung oder Installation schlagen fehl. Auch in Kommentaren darf `?>` nicht vorkommen – PHP beendet den Code dort, selbst mitten in einer `//`-Kommentarzeile.

**Neue Version bereitstellen:**
1. In `Heizungsregler.ino` die Versionsnummer `FW_VERSION` erhöhen, z. B. von `"1.32.0"` auf `"1.33.0"`. Installiert wird nur eine höhere Nummer.
2. In der Arduino IDE *Sketch → Kompilierte Binärdatei exportieren*. Die exportierte Datei `Heizungsregler_ESP32.ino.bin` in **`glw8_Heizungsregler-ino-esp32_dl.bin`** umbenennen.
3. Diese Datei nach `/OTA/Heizungsregler3P/` auf www.glw8.at hochladen (vorhandene ersetzen). Das Skript liefert ab sofort die neue Datei aus.

---

## REST-API

Alle Endpunkte liefern JSON. Ist ein Web-Passwort gesetzt, ist HTTP-Basic-Auth mit dem Benutzer `admin` nötig – außer für `/`, `/api/view` und `/api/history`.

| Methode | Pfad | Inhalt |
|---|---|---|
| GET | `/` | die Weboberfläche selbst (HTML, aus `web_ui.h`) – **ohne Passwort**, die Daten kommen über die API |
| GET | `/api/view` | Nur-Anzeige für das Anlagenschema – **ohne Passwort**: Temperaturen, Sollwerte, Mischer, Pumpe, Betriebsart, Zeitprogramm, Störungen, Uhrzeit, Firmware; `auth` = Passwort gesetzt |
| GET | `/login` | fragt das Passwort ab und leitet dann zur vollen Oberfläche weiter |
| GET | `/api/status` | Live-Werte, Zustände, Systeminfos, Anzahl der Protokolleinträge (`logCount`); Handschaltung: `manValve`, `manPump` (−1 Automatik, 0 aus, 1 ein), Restzeiten `manValveLeft` / `manPumpLeft` in s, `manPumpForced`; letzter Stellschritt `stepDY` und Alter `stepAge` in s, Alter des Heizkurven-Solls `curveAge` in s, letztes OTA-Update `lastOta` / `lastOtaFrom` / `lastOtaVia` (`ide`/`web`); OTA-Updates gesamt `totOta`, davon `totOtaIde` / `totOtaWeb`; Speicher `heap`, `heapBlock`, `heapFrag`, `heapMin`, `heapBlockMin`; letzter Absturz `crash`, `crashAt`; Kühlen `cooling`, `hcState`, `coolSet`, `coolMin`, `dewPoint`, `humidity`, `dewLimit`, `dewAlarm` |
| GET | `/api/config` | alle Einstellungen (Passwörter als `********`) |
| POST | `/api/config` | Einstellungen ändern; es genügen die zu ändernden Felder, z. B. `{"roomSet":21}` |
| POST | `/api/cmd` | `{"calibrate":true}`, `{"restart":true}`, `{"rescan":true}`, `{"factoryReset":true}`, `{"saveTotals":true}`, `{"resetTotals":true}`, `{"saveHistory":true}` (Wochenverlauf speichern), `{"bleScan":true}` (Bluetooth-Suche 10 min), `{"profile":1,"withTravel":false}` (1 Radiator, 2 Fußboden, 3 GLW8), `{"saveProfile":"glw8"}`; Handschaltung: `{"valve":"open"}` (bzw. `close`, `stop`, `auto`), `{"pump":"on"}` (bzw. `off`, `auto`) |
| GET | `/api/log` | Ereignisprotokoll als Text, bis 67 Zeilen, neueste zuerst |
| GET | `/api/sysinfo` | Gerät und Netzwerk: `mac`, `apMac`, `hostname`, `mdns`, `wifiMode`, `ssid`, `bssid`, `channel`, `rssi`, `ip`, `mask`, `gw`, `dns`, `dns2`, im Access-Point-Betrieb `apSsid`, `apIp`, `apClients`; `mqttHost`, `mqttState`, `mqttId`, `ntp`; `chipId`, `flashId`, `cpuMHz`, `flashReal`, `flashSize` (Byte), `flashMHz`, `sketch`, `sketchFree`, `fsTotal`, `fsUsed` (Byte), `resetReason`; `fw`, `build`, `core`, `sdk` |
| GET | `/api/profiles` | Werte aller Profile (Radiator, Fußboden, GLW8), aktuelles Profil, ob und wann GLW8 gespeichert wurde |
| GET | `/api/curve.csv` | Heizkurve als CSV-Liste, zur Laufzeit im Regler erzeugt (siehe [Heizkurve als Liste](#heizkurve-als-liste-herunterladen)) |
| GET | `/api/sensors` | je Fühler: Pin, Herkunft, gefundener DS18B20, Pin-Wert, externer Wert mit Alter, verwendeter Wert, Störung; bei gemeinsamem Bus zusätzlich alle gefundenen DS18B20 mit Rolle ; je Fühler zusätzlich `bleMac`, `bleT`, `bleAge`; dazu `bleOn` (Empfang läuft), `bleDiscover` (Restdauer der Suche in s) und `ble` = alle empfangenen RuuviTags mit `mac`, `t`, `h`, `p` (hPa), `mv`, `rssi`, `fmt`, `age`, `roles` |
| GET | `/api/history` | **ohne Passwort** – Verlauf 24 h in 5 min, mit `?res=fine` 1 h in 30 s, mit `?res=week` 7 Tage als Stundenmittel (`iv` 3600, `age` = Sekunden seit Ende der letzten vollen Stunde): `{"iv":300,"age":s,"d":[[vorlauf,soll,rücklauf,außen,raum,mischer,pumpe],…]}`; Werte ×10, −32768 = ungültig |
| GET / POST | `/update` | Seite zum Hochladen der Firmware / Empfang der `.bin`-Datei |

---

## Einstellungen speichern und zurücksetzen

- **Speicherort:** Alle Einstellungen liegen in der Datei `/config.json` im Flash-Dateisystem (LittleFS) und überstehen Stromausfall und Firmware-Updates.
- **Sofort gespeichert:** Änderungen über die Weboberfläche.
- **Verzögert gespeichert:** Änderungen über MQTT und Modbus, nach 10 Sekunden ohne weitere Änderung. Das schont den Flash.
- **Neustart nötig:** Änderungen an WLAN, Hostname, Web-Passwort, Modbus und OTA lösen automatisch einen Neustart aus.
- **Gesamtzähler:** Sie liegen getrennt in `/counters.json` und werden alle 24 h sowie vor Neustarts und Updates gespeichert (siehe [Gesamtzähler](#gesamtzähler-über-alle-neustarts)).
- **Werkseinstellungen** (*System → Werkseinstellungen*): löscht **alle** Einstellungen einschließlich WLAN, aber nicht die Gesamtzähler und nicht das Profil GLW8. Danach startet der Regler im Access-Point-Modus wie beim ersten Start, mit dem Profil Radiatorheizung.
- **Update von älteren Versionen:**
  - Bestehende Einstellungen bleiben erhalten. Die Raum-Einstellungen aus Version 1.3–1.5 (`roomSource`, `roomTopic`, `roomJsonKey`, `roomExtTimeout`) werden automatisch übernommen.
  - Wer von Version 1.5 oder älter kommt, hat die Fühler noch gemeinsam an D5. Der Regler erkennt die gespeicherte Zuordnung über Seriennummern und stellt automatisch auf **Gemeinsamer Bus an D5**. Es muss nichts umverdrahtet werden.
  - Wer von Version 1.6 kommt, behält die Anschlussart **Eigene Pins**.

---

## Aufbau des Quelltexts

Seit 2.4.0 ist der Sketch auf mehrere Dateien verteilt:

```
Heizungsregler_ESP32/
├── Heizungsregler_ESP32.ino     Hauptdatei: Übersicht, Bibliotheken, Version, Abschnitte 1–4
├── A050_Hilfsfunktionen.ino     5   Uhrzeit, Zeitprogramm, Feiertage, Ereignisprotokoll
├── A060_Konfiguration.ino       6   /config.json laden, speichern, ändern
├── A070_Mischer_Zaehler.ino     7   Mischer, Pumpe, Handschaltung, Zähler, Wärmemenge, Neustarts
├── A080_Fuehler.ino             8   DS18B20, externe Werte (MQTT/Modbus)
├── A085_Bluetooth.ino           8b  RuuviTag (NimBLE 1.4.x und 2.x)
├── A090_Regelung.ino            9   Heizkurve, Kühlkurve, PI-Regler, Schutzfunktionen
├── A095_Relais_K4.ino           9b  Relais K4: Wärmeanforderung, Störmeldung, Zirkulation …
├── A100_Status.ino              10  Status als JSON
├── A105_Profile.ino             10b Parameterprofile
├── A110_Modbus.ino              11  Modbus TCP und RTU
├── A115_Verlauf.ino             11b Temperaturverlauf
├── A120_MQTT.ino                12  MQTT, Home Assistant
├── A130_Webserver.ino           13  REST-API
├── A135_Firmware_Update.ino     13b Anbindung des Moduls GlwOta
├── A140_WLAN.ino                14  WLAN, Access-Point
├── A150_Setup_Loop.ino          15  setup(), loop(), Regel-Task
├── web_ui.h                     Weboberfläche
├── src/GlwOta/GlwOta.h, .cpp    Modul Firmware-Update (wiederverwendbar)
└── Server/                      PHP-Skript für die Auslieferung auf glw8.at
```

**Wie die Arduino IDE die Dateien zusammensetzt:** Jede `.ino`-Datei erscheint als eigener Reiter. Beim Übersetzen hängt die IDE zuerst die Hauptdatei und dann die übrigen `.ino`-Dateien in **alphabetischer** Reihenfolge aneinander und übersetzt das Ganze als ein Programm. Deshalb:

- Die Nummer im Dateinamen ist die Abschnittsnummer mal 10 (8b → `085`). Damit bleibt die Reihenfolge genau die der früheren einteiligen Datei, auch wenn die IDE Groß- und Kleinschreibung unterschiedlich sortiert. Neue Dateien bekommen eine passende Nummer dazwischen.
- **Datentypen und globale Variablen stehen alle in der Hauptdatei.** Eine Variable, die erst in einer späteren Datei angelegt wird, ist in den Dateien davor unbekannt. Funktionen dürfen dagegen in jeder Datei stehen – die IDE meldet sie selbst überall an.
- Dateien im Unterordner `src/` übersetzt die IDE dagegen **getrennt** als eigene C++-Dateien. Dort liegt das Modul `GlwOta`, das bewusst nichts vom Regler weiß; eingebunden wird es mit `#include "src/GlwOta/GlwOta.h"`.

Dazu kommt im Ordner `Server/` das Skript für die Firmware-Auslieferung auf glw8.at.

**Aufgaben (FreeRTOS) in Version 2:**

| Aufgabe | Kern / Priorität | Inhalt |
|---|---|---|
| **Regel-Task** `controlTask` | Kern 1, Priorität 3 | alle 20 ms: Fühler lesen, Mischerstellung nachführen, Handschaltung, Verlauf, Pumpenlaufzeit; im Takt *Abtastzeit* der Regelzyklus. Watchdog 30 s. Überwacht zusätzlich die Hauptschleife (Neustart, wenn sie 3 min hängt). |
| **Hauptschleife** `loop()` | Kern 1, Priorität 1 | WLAN, Webserver, MQTT, Modbus, OTA, Gesamtzähler, Speicherwächter, Speichern der Einstellungen |
| WLAN-Stack, Timer | Kern 0 | vom ESP32-Core; der Mischerimpuls wird von einem Hardware-Timer beendet |

Gemeinsame Daten (Einstellungen `cfg`, Zustand `st`) schützt ein Mutex: Der Regel-Task hält ihn während jedes Durchlaufs (wenige Millisekunden), Webserver-Handler, MQTT-Empfang und Modbus halten ihn, solange sie Daten lesen oder ändern (Klasse `Guard`). Das Ereignisprotokoll hat einen eigenen Mutex. Alle sind durchgehend auf Deutsch kommentiert: Jede Datei beginnt mit einer Übersicht, jeder Abschnitt hat eine Überschrift, jede Funktion eine Beschreibung, und schwierigere Stellen (Heizkurvenformel, PI-Regler, Impulsansteuerung des Mischers, Ringpuffer des Verlaufs, Speicherung der Zähler) sind Schritt für Schritt erklärt.

### Die Abschnitte der Firmware

| Abschnitt | Inhalt |
|---|---|
| 1. Hardware | Pinbelegung, Rollen der Fühler |
| 2. Konfiguration | `struct HeizConfig` mit allen einstellbaren Werten und ihren Grundeinstellungen |
| 3. Laufzeitzustand | Messwerte, Zustände des Reglers, Zähler |
| 4. Globale Objekte | Webserver, MQTT, Modbus, OneWire-Busse, Ticker |
| 5. Hilfsfunktionen | Zeit, Ereignisprotokoll, Zahlenformate |
| 6. Konfiguration | Laden und Speichern von `/config.json`, Übernahme geänderter Werte |
| 7. Mischer | 3-Punkt-Impulse bzw. 0–10 V, Kalibrierung, Zähler für Stellvorgänge, Pumpenlaufzeit und Neustarts, Gesamtzähler in `/counters.json` |
| 8. Temperaturfühler | eigene Pins oder gemeinsamer Bus, Zuordnung, externe Werte, Zeitüberwachung |
| 9. Regelung | Heizkurve (Formel oder Stützpunkt-Spline `curvePoints()`), Kühlkurve, Taupunkt, Absenkung, Heiz- und Kühlgrenze, Raumeinfluss, Rücklaufbegrenzung, PI-Regler (beim Kühlen umgekehrt), Pumpe, Schutzfunktionen |
| 9b. Relais K4 | Zusatzfunktion: Wärmeanforderung, Heizen/Kühlen, Sammelstörmeldung, Zirkulation, Entfeuchter, frei schaltbar; Mindestlaufzeit/-pause, Zähler |
| 10. Status | alle Werte als JSON (für Web, MQTT und CSV) |
| 10b. Profile | Parametersätze Radiator- und Fußbodenheizung |
| 11. Modbus | Register für TCP und RTU |
| 11b. Verlauf | Ringpuffer 24 h (5 min) und 1 h (30 s) |
| 12. MQTT | Topics, Befehle, Home-Assistant-Discovery |
| 13. Webserver | REST-API, Gerätedaten (`/api/sysinfo`), CSV-Ausgabe der Heizkurve |
| 13b. Firmware-Update | Einstellungen für das Modul GlwOta (Server, PIN, Kennung) und was der Regler vor und nach einem Update tut |
| 14. WLAN | Verbindung, Access-Point, Start des Netzwerk-Ports für die Arduino IDE |
| 15. `setup()` / `loop()` | Start und Hauptschleife; Ausgabe der Versionskennung |

### Stromlaufplan

Für Version 2 folgt ein eigener Stromlaufplan für die ESP32-Relaisplatine. Bis dahin gelten die Pinbelegung und die Hinweise unter [Hardware und Verdrahtung](#hardware-und-verdrahtung).

### `web_ui.h` – die Weboberfläche

Die komplette Bedienseite (HTML, CSS und JavaScript) steht als ein einziger Text im Flash (`PROGMEM`) und wird von `/` ausgeliefert. Sie lädt nichts aus dem Internet. Die Datei ist in drei Teile gegliedert:

- **Gestaltung (CSS):** Alle Farben stehen als Variablen am Anfang (`--acc` Orange, `--blue` Blau, `--paper` Hintergrund …). Wer die Farben ändern will, ändert nur diese Zeilen; der Dunkelmodus hat einen eigenen Block direkt darunter. Danach folgen, jeweils kommentiert, Kopfzeile, Reiter, Karten, Anlagenschema mit Animation, Smartphone-Raster, Formulare, Tabellen und die Anpassungen für schmale Bildschirme.
- **Seitenaufbau (HTML):** Kopfzeile, Reiterleiste und je Reiter ein Abschnitt. Beim Anlagenschema ist erklärt, wie das Koordinatensystem (520 × 300) und die Leitungspfade aufgebaut sind und wie die Werte über `data-v` an die richtige Stelle kommen.
- **Programmlogik (JavaScript):** Eine Übersicht am Anfang beschreibt den Ablauf. Die Einstellungsformulare entstehen aus einer Tabelle (`F`): Für ein neues Eingabefeld genügt eine Zeile dort, sofern die Firmware den gleichnamigen JSON-Wert kennt. Danach folgen Wochenprogramm, Profile, Live-Anzeige (`poll()` alle 2 s, Nur-Anzeige 5 s, pausiert bei unsichtbarer Seite), Fühler, Heizkurvengrafik, Verlauf und CSV-Download.

Da die Kommentare in `web_ui.h` Teil der Seite sind, werden sie mit an den Browser geschickt. Die Seite ist dadurch rund 78 KB groß; sie wird direkt aus dem Flash gesendet und belegt keinen Arbeitsspeicher.

### Speicherbedarf (Version 2.0.0, ESP32)

| | Belegung |
|---|---|
| Programm im Flash | mit Partition *Minimal SPIFFS* stehen je Programmbereich 1,9 MB zur Verfügung – genug Reserve für Updates über WLAN; aktuelle Größe und freier Platz in der Karte *Gerät und Netzwerk* |
| Arbeitsspeicher | rund 320 KB; im Betrieb typisch 150–220 KB frei (Version 1: 10–20 KB) |
| Dateisystem (LittleFS, 190 KB) | `/config.json`, `/counters.json`, `/profile_glw8.json`, `/hist7.bin` – zusammen wenige KB |

## Modul GlwOta – Firmware-Update für andere Projekte

Das Firmware-Update ist seit 2.4.0 ein eigenes Modul im Ordner `src/GlwOta` (`GlwOta.h`, `GlwOta.cpp`). Es kennt den Heizungsregler nicht und lässt sich unverändert in jedes ESP32-Projekt mit dem Webserver des ESP32-Cores übernehmen. Es bringt mit:

- **Update-Seite** `/update`: Datei auswählen, Kennung und Version in der Datei prüfen, mit Fortschrittsbalken hochladen, nach dem Neustart selbst neu laden. Hell und dunkel, für Handy und PC. Name, Version, Fußzeile und Server kommen aus den Einstellungen – die Seite selbst enthält nichts Projektbezogenes.
- **Prüfen und Installieren von einem eigenen Server** (z. B. glw8.at) über ein PHP-Skript mit PIN und Protokoll – auf der Update-Seite und auf Wunsch in jeder eigenen Seite (Skript `/ota.js`).
- **Netzwerk-Port für die Arduino IDE** (ArduinoOTA) mit verständlichen Fehlermeldungen.
- **Versionskennung** in der Firmware (Makro `GLWOTA_MARKER`) und **Schutz vor falschen Dateien** beim Hochladen.
- **Rückmeldungen** `onStart`, `onEnd`, `onError`, `onProgress` – dort hängt das eigene Projekt seine Aufgaben an (Ausgänge abschalten, Daten sichern, protokollieren).

Das Modul liegt dem Paket zusätzlich als fertige **Arduino-Bibliothek** `GlwOta.zip` bei (mit Beschreibung, Beispiel-Sketch und allgemeinem PHP-Skript). Zwei Wege, es in einem anderen Projekt zu verwenden:

1. **Als Bibliothek (empfohlen für mehrere Projekte):** *Sketch → Bibliothek einbinden → .ZIP-Bibliothek hinzufügen …* und `GlwOta.zip` wählen. Danach `#include <GlwOta.h>`. Ein Beispiel steht unter *Datei → Beispiele → GlwOta → Minimal*.
2. **Als Kopie im Sketch:** den Ordner `src/GlwOta` in den Ordner des anderen Sketches kopieren und `#include "src/GlwOta/GlwOta.h"` schreiben. Der Sketch ist dann ohne zusätzliche Installation übersetzbar (so macht es der Heizungsregler).

Kleinstes Beispiel:

```cpp
#include <WiFi.h>
#include <WebServer.h>
#include <GlwOta.h>

#define FW_VERSION "1.0.0"
GLWOTA_MARKER("FIRMA:PRODUKT:FW=", FW_VERSION);   // Kennung, einmal im Programm

WebServer server(80);
GlwOta ota;

void setup() {
  WiFi.begin("WLAN", "Passwort");
  while (WiFi.status() != WL_CONNECTED) delay(200);
  GlwOtaConfig c;
  c.product   = "Mein Gerät";
  c.version   = FW_VERSION;
  c.markerKey = "FIRMA:PRODUKT:FW=";
  // optional: c.serverUrl = "https://www.example.at/ota/mein-geraet.php"; c.serverPin = "1234";
  ota.setAuth("admin", "geheim");          // optional: Anmeldung für /update
  ota.begin(server, c);
  server.begin();
  ota.startIde("mein-geraet");             // Netzwerk-Port für die Arduino IDE
}

void loop() {
  server.handleClient();
  ota.loop();
}
```

Wichtig für jedes Projekt:

- Partitionsschema **mit OTA** wählen (z. B. *Minimal SPIFFS* oder *Default 4MB with spiffs*). Ohne zweite Programm-Partition meldet das Modul „keine Update-Partition“.
- Die **Kennung** (`markerKey`) muss je Produkt eindeutig sein. Nur Dateien mit genau dieser Kennung nimmt das Gerät über den Browser an (`strictMarker`, abschaltbar).
- `ota.loop()` muss regelmäßig laufen; es startet nach einem erfolgreichen Browser-Update neu.
- Für den Server-Weg das PHP-Skript aus der Bibliothek (`extras/server/firmware.php`) unter einem passenden Namen auf den Webserver legen und die Firmware als `glw8_<Skriptname>_dl.bin` daneben. Die Weboberfläche des Geräts muss die PIN kennen (`serverPin`).

Die vollständige Beschreibung aller Einstellungen, Adressen und Rückmeldungen steht in `GlwOta.h` und in der `README.md` der Bibliothek.

---

## Versionen

### Version 2 (ESP32)

| Version | Änderungen |
|---|---|
| 2.6.0 | **Shelly BLU H&T (SBHT-003C) als Bluetooth-Sensor** für Außen und Raum, neben dem RuuviTag: Format BTHome v2, auch **verschlüsselt** (Schlüssel je Fühler, wie ein Passwort behandelt; geprüft mit dem Beispiel von bthome.io); Temperatur, Feuchte (auch Raumfeuchte für Taupunktschutz und Entfeuchter), Batterie in %; Liste der Bluetooth-Sensoren mit Art und Schlüsselzustand; Bluetooth-Empfangsfenster 80 statt 40 ms je 160 ms. Auch andere BTHome-v2-Thermometer werden erkannt |
| 2.5.0 | **Relais K4 mit wählbarer Funktion:** Wärmeanforderung an Kessel/Wärmepumpe (Werkseinstellung), Umschaltung Heizen/Kühlen, Sammelstörmeldung im Ruhestromprinzip, Zirkulationspumpe mit Zeitfenstern und Takt, Entfeuchter beim Kühlen, frei schaltbar; Mindestlaufzeit und Mindestpause, Ausgang umkehrbar; Karte im Reiter Regler, Kennzeichen in der Übersicht, Zeile im System, Zähler (seit Neustart und gesamt); MQTT `…/k4`, Home-Assistant-Entitäten, Modbus-Register 25/26 und 35–39; Abschnitt im Reiter Hilfe. Zuvor blieb K4 immer aus |
| 2.4.0 | **Quelltext auf mehrere Dateien verteilt** (Reiter je Abschnitt, siehe [Aufbau des Quelltexts](#aufbau-des-quelltexts)). **Firmware-Update als eigenes, wiederverwendbares Modul GlwOta** (auch als Arduino-Bibliothek `GlwOta.zip`): Update-Seite mit Dateiprüfung (Version wird vor dem Hochladen angezeigt, ältere Version mit Warnhinweis), Fortschrittsbalken und automatischem Neuladen; Regler nimmt über den Browser nur Dateien mit der Kennung `GLW8:HZR3P32:FW=` an; „Auf neue Firmware prüfen“ jetzt auch auf der Seite `/update`; während eines Browser-Updates ruht die Regelung wie beim Update aus der Arduino IDE (Mischer angehalten, Daten gesichert); Browser-Updates werden sofort mit Uhrzeit gezählt; Fehlermeldungen beim Update im Klartext. **Heizkurvengrafik:** Stützpunkte P1 … P4 markiert (Ring, Name, Lotlinie zur Außentemperatur), Werteanzeige springt auf den Stützpunkt |
| 2.3.2 | Bluetooth: übersetzt jetzt mit **NimBLE-Arduino 1.4.x und 2.x** (die installierte Version wird automatisch erkannt). Behebt die Fehler „expected class-name before '{'“, „marked 'override', but does not override“ und „discards qualifiers“ mit NimBLE 1.x |
| 2.3.1 | Übersicht: **Mischerstellung farbig** von kalt (blau, zu) nach warm (orange, offen) – Zahl im Anlagenschema, Zahl im Smartphone-Raster und Balken (Verlauf über die ganze Skala); beim Kühlen umgekehrt, weil „offen“ dort mehr kaltes Wasser bedeutet |
| 2.3.0 | **Wärmemengenzählung** aus Volumenstrom (neues Feld *Volumenstrom Heizkreis l/min* im Reiter Regler) und Vorlauf/Rücklauf: Wärmeleistung in der Übersicht und im Reiter System, Wärmemenge heute/gestern/gesamt und seit Neustart, getrennte Kältemenge beim Kühlen; MQTT-Werte und Home-Assistant-Sensoren für das Energie-Dashboard; Gesamtzähler werden jetzt alle 6 h statt alle 24 h gespeichert |
| 2.2.0 | **Home Assistant – Thermostat (`climate`) ausgebaut:** Modus `heat` = Betriebsart *Raumtemperatur* (bisher fester Vorlauf); alle Betriebsarten zusätzlich als Voreinstellungen (Presets) mit deutschem Namen; Temperaturbereich (Raumsoll Heizen/Kühlen) im Modus `heat_cool`; Anzeigegenauigkeit 0,1 K; Symbol Heizkörper |
| 2.1.0 | **Bluetooth-Sensoren (RuuviTag, auch Pro)** als Quelle für Außen und Raum: neue Herkünfte *Bluetooth*, *Bluetooth, bei Ausfall MQTT* und *Pin, bei Ausfall Bluetooth*; Karte *Bluetooth-Sensoren* mit Suche (10 min), Liste (Temperatur, Feuchte, Druck, Batterie, Empfang) und Zuordnung per Klick; Raumfeuchte vom Raum-RuuviTag für den Taupunktschutz; Zeitüberwachung und Batteriewarnung; Datenformate 5, C5 und 3; Spalte *Wert Bluetooth* in der Fühlertabelle; Hinweis „Ersatzwert“ in der Übersicht für alle Ersatzquellen. Neue Bibliothek NimBLE-Arduino |
| 2.0.2 | Fehlalarm „Warnung: Regel-Task seit 5 s ohne Durchlauf“ behoben: Der Vergleich der Zeitstempel von Regel-Task und Hauptschleife lief ohne Vorzeichen über, sobald die Hauptschleife kurz auf den Mutex warten musste. Die Regelung war davon nicht betroffen. Gleiche Absicherung bei der Überwachung der Hauptschleife |
| 2.0.1 | MQTT: Fehlercodes des Brokers im Klartext (Karte *Gerät und Netzwerk* und Ereignisprotokoll), z. B. Code 5 „nicht berechtigt – Broker verlangt Benutzer und Passwort“; nach dem Laden einer Einstellungsdatei Hinweis, welche Passwörter neu einzutragen sind |
| 2.0.0 | **Neubeginn auf ESP32** (ESP32-WROOM-32E 4-Kanal-Relaisplatine) mit allen Funktionen von 1.35.0. Regelung in eigenem **Regel-Task** (Kern 1, Watchdog 30 s), Netzwerk in der Hauptschleife, gemeinsame Daten mit Mutex geschützt; Überwachung der Hauptschleife (Neustart nach 3 min Hängen). Neue Pinbelegung (Relais 32/33/25/26, Fühler 4/13/14/27, 0–10 V an 19 mit Hardware-PWM, RS485 an UART2 16/17/18 – serielle Meldungen bleiben frei). Neustartgründe des ESP32 im Klartext (u. a. Unterspannung, Task-Watchdog). Speicherwächter-Grenzen an den ESP32 angepasst. Karte *Gerät und Netzwerk* mit Chip-Modell und Arbeitsspeicher. Neue Karte **Einstellungen sichern und übertragen** (herunterladen, aus Datei laden – auch aus Version 1). Firmware-Prüfung über eigenes Skript `Heizungsregler-ino-esp32.php` und eigene Kennung `GLW8:HZR3P32:FW=…`, damit ESP8266- und ESP32-Firmware nie verwechselt werden. K4 und Status-LED folgen in der nächsten Version. |

### Version 1 (ESP8266, Wemos D1 mini)

| Version | Änderungen |
|---|---|
| 1.35.0 | Neue Betriebsart **Raumtemperatur** (`room`, Modbus 6): Vorlauf-Sollwert aus einem PI-Raumregler (Kaskade) ohne Heizkurve und Außentemperatur; Verstärkung, Nachstellzeit und Heizpause einstellbar (Karte *Raumregelung*, Profilwerte); Anti-Windup an Vorlauf max bzw. bei offenem Mischer; Ersatzbetrieb ohne Raumwert (Heizkurve bzw. fester Sollwert); Status `rcSet`, `rcI`, `rcP`, `rcIdle`, `rcFallback`, Modbus-Statusbits 2 Bit 5/6; neues Kapitel **Regelung im Detail** mit allen Betriebsarten und dem Einfluss der Messwerte |
| 1.34.4 | Abruf von glw8.at mit Freigabe-PIN (`pin`, Konstante `FW_PIN`); das Server-Skript liefert nur mit richtiger PIN aus (sonst 403 nach 2 s), protokolliert PIN fehlt/falsch/richtig und ersetzt die PIN im Protokoll durch `****`; Protokoll des Skripts im Format JSON Lines (je Aufruf ein JSON-Objekt mit festen Feldern); verständliche Meldung bei 403 in der Weboberfläche |
| 1.34.3 | Abruf von glw8.at mit Angaben für das Protokoll des Server-Skripts (`a` Aktion, `v` installierte Version, `h` Hostname, `n` Zielversion) und Rückmeldungen zum Ablauf der Installation (`uploaded`, `ok`, `timeout`, `error`, `checkerror`); Server-Skript schreibt je Aufruf eine Zeile in `Heizungsregler-ino-esp32.log` |
| 1.34.2 | Prüfung auf neue Firmware fragt jetzt das PHP-Skript `https://www.glw8.at/OTA/Heizungsregler3P/Heizungsregler-ino-esp32.php` ab, das die Datei `glw8_Heizungsregler-ino-esp32_dl.bin` ausliefert; Skript als Vorlage unter `Server/` |
| 1.34.1 | Reiter *System*: neue Karte **Gerät und Netzwerk** mit MAC-Adresse (WLAN und Access-Point), Hostname/mDNS, SSID, BSSID, Kanal, Signalstärke mit Einordnung, IP/Subnetz/Gateway/DNS, MQTT-Broker mit Zustand und Client-ID, NTP, Chip-ID, Prozessor, Flash (mit Warnung bei falscher Größe), Programm- und Update-Platz, Dateisystem, Startgrund, Übersetzungszeitpunkt, Core/SDK; Knopf *Als Text kopieren*; neuer Endpunkt `/api/sysinfo` (nur bei offenem Reiter, alle 30 s) |
| 1.34.0 | Heizkurve wahlweise über **4 frei einstellbare Stützpunkte** (Außen- und Vorlauftemperatur je Punkt) mit glattem, monotonem Spline; neue Karte *Heizkurve über Stützpunkte* mit Knopf *Aus Formel übernehmen*; in der Grafik vier frei ziehbare Griffe; Raumsoll-Einfluss einstellbar; Kurvenart und Punkte in Profilen, Konfiguration und CSV-Liste |
| 1.33.2 | Prüfung auf neue Firmware sucht jetzt `https://www.glw8.at/OTA/Heizungsregler3P/Heizungsregler-ino-esp32.bin` |
| 1.33.1 | Wochenverlauf wird nicht mehr stündlich, sondern **nur noch per Knopf** *Wochenverlauf im Regler speichern* im Flash gesichert; Anzeige des letzten Speicherzeitpunkts; Befehl `saveHistory` |
| 1.33.0 | Verlauf über **7 Tage**: Stundenmittel der 5-Minuten-Werte, 168 Punkte, stündlich im Flash gesichert (übersteht Neustarts), Lücken für Zeiten ohne Strom; Ansicht „7 Tage“ mit Tageslinien, CSV-Download, `/api/history?res=week` |
| 1.32.4 | Ereignisprotokoll: Änderungen an Einstellungen (Web und MQTT) erscheinen einzeln mit altem und neuem Wert, z. B. „Web: Raumsollwert °C 20 → 21.5“; Passwörter nur als „geändert“ |
| 1.32.3 | Aktiver Reiter deutlich hervorgehoben: Schrift in Akzentfarbe und fett, getönte Fläche, 3-px-Balken unten |
| 1.32.2 | Reiter *Hilfe*: alle Hinweise zur Arduino IDE entfernt (Update aus der IDE, Binärdatei exportieren, OTA-Eintrag) |
| 1.32.1 | Pumpen-Blockierschutz läuft 180 s statt 30 s |
| 1.32.0 | Knopf **Auf neue Firmware prüfen** im Reiter System: Der Browser liest die Version der Datei `Heizungsregler_ESP32.ino.bin` auf www.glw8.at und installiert eine neuere Version nach Rückfrage über `/update`; Versionskennung `GLW8:HZR3P32:FW=…;` wieder in der Firmware; der Regler selbst baut keine Internetverbindung auf |
| 1.31.0 | **Kühlen**: Betriebsarten *Kühlen* und *Heizen/Kühlen automatisch* (Heiz- und Kühlgrenze), Kühlkurve aus zwei Punkten, Raumsollwert Kühlen mit Raumaufschaltung, umgekehrte Regelrichtung des Mischers, Taupunktschutz (Untergrenze Taupunkt + Abstand, Mischer ZU bei Taupunktalarm) mit Raumfeuchte über MQTT-Topic, `…/set/humidity` oder Modbus-Register 24; Anlagenschema im Kühlbetrieb türkis; Home Assistant: Modi `cool` und `heat_cool`, Raumfeuchte, Kühl-/Taupunkt-Entitäten; Modbus: Register 22–24 und 30–34 |
| 1.30.0 | Automatisches Firmware-Update von www.glw8.at ersatzlos entfernt: Schalter, *Jetzt nach Update suchen*, Zähler „automatisch“, Versionskennung in der Firmware und die Datei `update_certs.h` entfallen; Programm rund 135 KB kleiner. Update weiterhin aus der Arduino IDE oder als Datei im Browser |
| 1.29.2 | Hintergrund-Tabs pausieren: Ist die Seite nicht sichtbar (anderer Tab, Fenster minimiert, Handy gesperrt), fragt sie den Regler nicht ab; beim Zurückkehren sofort frische Werte |
| 1.29.1 | Nur-Anzeige (ohne Passwort) fragt nur alle 5 s statt alle 2 s ab |
| 1.29.0 | Stabilität: Absturzursache (Exception, Adresse, Watchdog) wird gespeichert und unter *System → Letzter Absturz* angezeigt; Speicherwächter mit Minimum, größtem Block und geordnetem Neustart bei Speichermangel; JSON-Antworten und MQTT-Nachrichten werden direkt gesendet statt im RAM zusammengebaut; Ereignisprotokoll mit festen Zeilen (keine Heap-Zerstückelung); Weboberfläche wird vom Browser zwischengespeichert (ETag, 304); MQTT-Wiederverbindung mit wachsender Pause (10 s bis 5 min, 2 s Verbindungs-Timeout); automatisches Update prüft den Speicher vorher und setzt nach einem Absturz 24 h aus; Fühler am gemeinsamen Bus werden nur einmal je Messung gelesen |
| 1.28.1 | Handschaltung endet erst nach 24 Stunden statt nach 60 Minuten; Restzeit in Stunden und Minuten; neue Datei `Stromlaufplan.svg` (Verdrahtung) |
| 1.28.0 | Automatisches Update von www.glw8.at: nach dem Start und alle 7 Tage, nur neuere Versionen (Versionskennung in der Firmware), HTTPS mit Zertifikatsprüfung; Schalter und *Jetzt nach Update suchen* unter *Netzwerk → Modbus RTU & OTA-Update*; neue Datei `update_certs.h` |
| 1.27.1 | Nur-Anzeige: auch der Reiter *Verlauf* ist ohne Anmeldung sichtbar (`/api/history` ohne Passwort) |
| 1.27.0 | Zähler der OTA-Updates (gesamt, davon Arduino IDE / Browser) in der Karte *Zähler*; *letztes Update* jetzt auch für Updates über den Browser, mit Angabe des Wegs |
| 1.26.1 | Nur-Anzeige: Betriebsart als Etikett unter dem Anlagenschema |
| 1.26.0 | Anlagenschema ohne Passwort als Nur-Anzeige sichtbar (`/api/view`), Knopf *Anmelden* (`/login`) für die volle Oberfläche; Einstellungen und Befehle bleiben geschützt |
| 1.25.3 | Kurzerklärungen auch im Reiter *Regler* (Stellantrieb, Motorlaufzeit, Kp, Tn, Totzone, Abtastzeit, Mindestimpuls, Relais Low-aktiv) |
| 1.25.2 | Kurzerklärungen auch an den Eingabefeldern der Karten *Heizkurve & Grenzen* und *Raumtemperatur* |
| 1.25.1 | Profiltabelle: Kurzerklärung zu jedem Parameter beim Überfahren mit der Maus, Antippen oder per Tastatur |
| 1.25.0 | Wochenprogramm: Zeile **Feiertag**; gesetzliche Feiertage Österreich automatisch (inkl. beweglicher Feiertage), zusätzliche freie Tage einstellbar; Anzeige heutiger/nächster Feiertag; neue Kopierknöpfe *Montag → Di–So* und *Sonntag → Feiertag* |
| 1.24.1 | Reiter *System*: Zeile *Programmlaufzeit* sekundengenau mit Startzeitpunkt (ersetzt die grobe Zeile *Laufzeit*); Laufzeit wird jetzt in 64 Bit gezählt – vorher begann sie nach ca. 49,7 Tagen wieder bei 0 |
| 1.24.0 | Reiter *System*: Zeitstempel bei *Letzter Stellschritt* (letzter echter Schritt mit Richtung) und *Heizkurven-Soll* (letzte Berechnung); bei *OTA (Arduino IDE)* Datum des letzten OTA-Updates mit vorheriger Version |
| 1.23.0 | Eigenes Heizungsprofil **GLW8**: aktuelle Werte (inkl. Motorlaufzeit) speichern und wieder laden; eigene Datei im Flash, übersteht Werkseinstellungen; Spalte GLW8 in der Vergleichstabelle; über Web, MQTT, Home Assistant und Modbus |
| 1.22.0 | Neuer Reiter *Hilfe* mit Kurzanleitung (aufklappbare Abschnitte, echte Werte des Reglers eingesetzt) |
| 1.21.1 | Knopf *Heizkurve als Liste herunterladen (CSV)* wieder direkt unter der Heizkurvengrafik |
| 1.21.0 | Höchstwert der Heizkurvengrafik einstellbar (*Skala bis* 40/50/60/70/80 °C), im Regler gespeichert (`chartMax`) |
| 1.20.2 | Heizkurvengrafik (mit Einstellen per Maus/Finger) aus der Übersicht in den Reiter *Heizkurve* verschoben, ganz nach oben |
| 1.20.1 | Heizkurvengrafik: Vorlaufachse 10–60 °C statt 10–80 °C, höhere Grafik (PC 360 px, Smartphone 300 px), Rasterlinien alle 5 °C |
| 1.20.0 | Ereignisprotokoll mit 67 statt 15 Einträgen; eigenes, nummeriertes Fenster im Reiter System; Abfrage über `/api/log` statt im Status |
| 1.19.1 | Anlagenschema: Vorlauf- und Außenanzeige wieder an der ursprünglichen Stelle (waren in 1.19.0 versehentlich verschoben) |
| 1.19.0 | Heizkurve direkt in der Grafik mit Maus oder Finger einstellen: Griffpunkte bei −15 °C und +10 °C, Parallelverschiebung durch Ziehen der Kurve, Vergleich mit der bisherigen Kurve, Speichern erst mit „Übernehmen“ |
| 1.18.0 | Handschaltung: Mischer AUF / ZU / Stopp / Automatik und Pumpe Ein / Aus / Automatik, in der Weboberfläche sowie über MQTT, Home Assistant und Modbus; Schutzfunktionen haben Vorrang, Ende nach 60 min |
| 1.17.0 | Werteanzeige in allen Diagrammen beim Überfahren mit der Maus oder Antippen mit dem Finger (Hilfslinie, Punkte auf den Kurven, Kästchen mit allen Werten) |
| 1.16.1 | Urheberhinweis „© 2026 erich.grassl@glw8.at“ auf allen Seiten (Fußzeile), in den CSV-Listen, in der seriellen Startmeldung und in Home Assistant; eigene deutsche Firmware-Update-Seite; Korrektur für die Arduino IDE (Datentyp des Verlaufs vor der ersten Funktion) |
| 1.16.0 | Weboberfläche neu gestaltet: Anlagenschema mit Messwerten an der Messstelle und laufender Pumpe, ruhigere Farben, segmentierte Auswahl der Betriebsart, Statusanzeigen als Etiketten, Smartphone-Raster, Dezimalkomma, verbesserter Dunkelmodus; Quelltext beider Dateien ausführlich deutsch kommentiert |
| 1.15.0 | Heizkurven-Liste wird zur Laufzeit im Regler mit den wirksamen Parametern erzeugt; zusätzliche Spalte „Vorlauf-Soll jetzt“ und Ist-Zustand |
| 1.14.0 | Heizkurve als CSV-Liste herunterladen (Vorlauf-Soll je Außentemperatur, Komfort und Absenkung) |
| 1.13.0 | GLW8-Logo im Kopf der Weboberfläche und als Browser-Tab-Symbol |
| 1.12.0 | Neustart-Zähler mit Neustartgrund (Einschalten, Software, Absturz/Watchdog, Reset-Taste) |
| 1.11.0 | Gesamtzähler über alle Neustarts für Stellvorgänge, Pumpenlaufzeit und Pumpenstarts; Speicherung im Flash alle 24 h und vor Neustart/Update |
| 1.10.0 | Pumpenlaufzeit und Pumpenstarts seit Neustart (Web, MQTT, Home Assistant, Modbus) |
| 1.9.0 | Zähler der Mischer-Stellvorgänge seit Neustart (Web, MQTT, Home Assistant, Modbus) |
| 1.8.0 | Verlauf zusätzlich 1 h mit 30-s-Auflösung; Download des angezeigten Zeitraums als CSV-Liste |
| 1.7.0 | Anschlussart der Fühler wählbar: eigene Pins oder gemeinsamer Bus an D5 mit Zuordnung über die Seriennummer |
| 1.6.0 | Jeder Fühler an eigenem Pin (D5, D7, D3, D4); Herkunft je Fühler: Pin, MQTT/Modbus oder Pin mit MQTT-Ersatz; Modbus RTU auf Hardware-Schnittstelle (RX/TX) |
| 1.5.0 | Update aus der Arduino IDE (ArduinoOTA); Modbus RTU über RS485 |
| 1.4.0 | Heizungsprofile Radiator- und Fußbodenheizung |
| 1.3.0 | Raumtemperatur über beliebiges MQTT-Topic; Raumeinfluss schaltbar |
| 1.2.0 | Wochenprogramm, Temperaturverlauf, erweiterte Home-Assistant-Discovery |
| 1.1.0 | Raumtemperaturfühler mit Raumaufschaltung und Raumabschaltung |
| 1.0.0 | Grundversion: Heizkurve, PI-Mischerregelung, Weboberfläche, MQTT, Modbus TCP |

---

© 2026 [erich.grassl@glw8.at](mailto:erich.grassl@glw8.at) · GLW8 Heizungsregler
