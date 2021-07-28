# Energiemessung Dokumentation

**Abteilung**: Industrial Internet of Things

**Betreuung**: Michael Schulze Weppel, Peer Ketterle, Dr. Daniel Kliewe

**Entwickler**: Florian Hoxha

## Einleitung
Der Intention von diesem Projekt war es dem Kunden zu ermöglichen, eine Überwachung diverser elektrischer Größen durchführen zu können. Es wurden zuvor die Werte ausgesucht, welche für den Kunden wichtig sein können. Für die Ermittlung dieser Werte wurde ein Messgerät des Herstellers `Camille Bauer Metrawatt AG` ausgewäht. Auf das Messgerät wird anhand einer `TCP-Verbindung` von der Entwicklungsumgebung in `Node-RED` zugegriffen und im Nachhinein in der Benutzeroberfläche angezeigt.  

## Node-RED Anforderung

Folgende Bibliotheken wurden für die Entwicklung verwendet:
* node-red
* node-red-contrib-modbus
* node-red-dashboard

## Setup
Zunächst wird eine Verbindung zum Messgerät hergestellt, welche wir anhand der `modbus-client node` erreichen können.


![modbus_clientnode_nodered](\images\modbus_clientnode_nodered.png)

In dieser Node müssen wir einige Parameter eintragen, um eine erfolgreiche Verbindung zu erstellen. Diese lauten wie folgt:

* `Type`: Auswahl der Verbindungsart, TCP/Serial/Serial Expert. In unserem Falle ist es `TCP`.
* `Host`: Die IP-Adresse des Messgeräts. In unserem Falle `172.30.16.222`.
* `Port`: Zuordnung der TCP-Verbindung. In unserem Falle `502`.

Nach dieser Konfiguration kann man mit der Erstellung des [Flows](#Flow) beginnen.

## Flow

In diesem Projekt wurden zwei Flows erstellt um von den elektrischen Größen (wie Leistung, Spannung, Stromstärke, usw.) und den harmonischen Schwingungen der Spannungsanteile, zu unterscheiden. Die Flows kann man in der Entwicklungsumgebung folgendermaßen wiederfinden:

* Energiemessung:

![Energiemessung_nodered](\images\Energiemessung_nodered.png)

* Oberschwingungen:

![Energiemessung_nodered](\images\Oberschwingungen_nodered.png)

Damit diese Flows uns auch den Wert ausgeben den wir suchen, brauchen wir einige Nodes um dies zu erreichen, und zwar:

* [FC3 Funktion](#FC3-Funktion)
* [Get Values](#Get-Values)
* [Convert Payload to Float Funktion](#Convert-Payload-to-Float-Funktion)


### FC3 Funktion  


Hierbei handelt es sich um eine normale `function-node` welche im Zusammenhang mit der `modbus-flex-getter-node` arbeitet. 
In der Funktion werden anhand weniger Zeilen von Code die Input Parameter, für die abzufragenden Werte, eingetragen. 


```node.js
var fc = 3
var id = '/HBIIoT/D2C/L1/rExampleX'
var address = 100
var quantity = 2

msg.payload =   {
                'value': msg.payload,
                'fc': fc, 
                'unitid': id,
                'address': address ,
                'quantity': quantity 
                }
return msg;
```

* `fc` = die Function Code (1:4). In unserem Falle `3` (Read Holding Registers)
* `id` = die ID die wir der abgefragten Variable vergeben.
* `address` = die Adresse des abgefragten Wertes. _siehe [Modbus-Schnittstelle SINEAX AM1000](https://www.gmc-instruments.de/media/doku/me/sineax-am-series/sineax-am1000-3000-modbus-sb_d.pdf)_
* `quantity` = die Anzahl der Register die von der Anfangsadresse ausgelesen werden. Meistens sind es 2 oder 4.

### Get Values

Nachdem wir erfolgreich im Abschnitt [FC3 Funktion](#FC3-Funktion) die Parameter eingetragen haben, müssen wir nur noch anhand der `modbus-flex-getter-node` auf die Adressen zugreifen.

![flexgetter_nodered](flexgetter_nodered.png)

In dieser Node müssen wir lediglich die zuvor erstellte [TCP-Verbindung](#Setup) auswählen und dieser Schritt ist vollendet.

### Convert Payload to Float

Wir sind bereits an dem Punkt angekommen wo wir die jeweiligen Werte nun auslesen können, jedoch stoßen wir an ein Problem. Die Ausgabe die wir nach dem Auslesen bekommen, ist für uns unbrauchbar. Es wird in diesem Falle ein `Data Array` ausgegeben, welches wir in eine Gleitkommazahl konvertieren um mit dem Wert arbeiten zu können.

```node.js
const buf = Buffer.from(msg.payload.buffer);
const value = parseFloat(buf.readFloatBE().toFixed(2));

switch (msg.modbusRequest.address){
    case xxx:
    var msgy = {payload : value};
    
}
return msgx;
```

* `buf` = hier wird der Buffer definiert, aus dem wir unser `Data Array` bekommen. In unserem falle `msg.payload.buffer`.
* `value` = hier wird der Wert in eine Gleitkommazahl konvertiert und im Anschluss auf eine Nachkommazahl von 2 gerundet.
* `switch` = hier wird ein `Switch Statement` aufgerufen, welche sich auf die `msg.modbusRequest.address` bezieht.
* `case` = bei dem `xxx` wird die Adresse eingetragen die wir Filtern wollen.
* `msgx` = hier wird die Gleitkommazahl aufgerufen. _Das `y` kann abhängig von der Anzahl der Adressen, kontinuierlich steigen._

Bei den `Oberschwingungen` haben wir eine weiter Ausgabe die wir hinzufügen müssen.
```node.js
switch (msg.modbusRequest.address){
    case xxx:
    var msgy = {payload : value,topic : z};
```
* `topic` = hierbei handelt es sich um die Beschriftung, welche im späteren Schritt im Balkendiagramm angezeigt wird.

Neben dem ganzen Code müssen wir noch auf die Anzahl der Ausgänge dieser Funktion acht geben. Hierbei kommt es darauf an, wie viele `Cases` wir in unserem `Switch Statement` haben.

![convertpayloadtofloat_nodered](\images\convertpayloadtofloat_nodered.png)

## Benutzeroberfläche

Nachdem wir die Ermittlung der Werte vollständig durchgeführt haben, besteht die Möglichkeit eine `Benutzeroberfläche` hinzuzufügen, welche die Anzeige der Werte erleichtert. In unserem Projekt wurden die folgenden `Nodes` dafür verwendet:

* [Button Node](#Button-Node)
* [Text Node](#Text-Node)
* [Gauge Node](#Gauge-Node)
* [Chart Node](#Chart-Node)

### Button Node

Diese Node ist dafür da, um das Ermitteln der aktuellen Werte zu Starten oder zu Stoppen. Diese werden wie folgt konfiguriert:

* `Start Funktion`: hier werden lediglich die Felder `Beschriftung` (welcher Name im User Interface angezeigt werden soll) und `Gruppe` (zuordnung wo diese Nodes angezeigt werden), konfiguriert.

![startbutton_ui_func_nodered](\images\startbutton_ui_func_nodered.png)

* `Stop Funktion`: hierbei wird neben den Feldern wie bei der [Start Funktion](#Start-Funktion), noch der `payload`, konfigurtiert. Beim `payload` wird nur ein `false` eingetragen, welches für das Anhalten des `Abfrageintervalls` da ist.

![stopbutton_ui_func_nodered](\images\stopbutton_ui_func_nodered.png)

* `Trigger Node`: diese ist dafür da um eine Abfrage in einem bestimmten Intervall zu konfigurieren. In dieser Node werden die Felder `dann` (wo die `Zeit` eingetragen wird) und das Feld `msg.payload ist gleich` (wo `false` eingetragen wird, um das Intervall nach einem false anzuhalten), konfiguriert.

![trigger_nodered](\images\trigger_nodered.png)

### Text Node

Diese Node ist dafür da, um die ermittelnden elektrischen Größen in der Benutzeroberfläche, darstellen zu können. Hierbei werden nur drei Werte konfiguriert, und zwar:

* `Label`: hier wird die Beschriftung, die in der Benutzeroberfläche angezeigt wird, eingetragen.
* `Value format`: hierbei wird lediglich in den geschweiften Klammern unser Wert eingetragen und die Einheit welche neben dem Wert angezeigt wird.

![value_ui_dashboard_nodered](\images\value_ui_dashboard_nodered.png)

### Gauge Node

Diese Node unterscheidet sich in der Konfiguration nicht gewaltig von der [Text Node](#Text-Tode). Der einzige Unterschied besteht darin, dass die Einheit in einem separaten Feld eingetragen wird, `Units`.

![gauge_ui_nodered](\images\gauge_ui_nodered.png)

### Chart Node

Das Chart Node verwenden wir für die Darstellung unserer `harmonischen Schwingungen`, da wir verschiedene Spannungsanteile haben die gleichzeitig dargestellt werden müssen. Bei dem Balkendiagramm müssen lediglich nur zwei Werte konfiguriert werden, diesen sind die folgenden:

* `Typ`: hier müssen wir das normale `Balkendiagramm` auswählen, damit wir die Werte in der X-Achse eintragen.
* `Y-Achse`: hierbei wird das `minimum` auf `0` und das `maximum` auf `2.5` gesetzt, damit wir einen Wertebereich von `0-2.5%` haben. 

![harmonic_osc_chart_nodered](\images\harmonic_osc_chart_nodered.png)
