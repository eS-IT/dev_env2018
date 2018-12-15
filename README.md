# Warum und wie phpfarm bei mir abgelöst wurde

Nach dem ich einige Jahre für mein Entwicklungssystem [phpfarm](https://github.com/cweiske/phpfarm) verwendet habe, ist es nun an der Zeit für etwas neues. Zum Einen wird phpfarm nicht mehr betreut, was sich in recht veralteten PHP-Versionen äußert. Zum Anderen gab es einige Probleme, wie immer wieder auftretende Error 500, die sich nicht eingränzen oder beheben ließen. Ich habe zwar schon seit langem auf das Dockerimage [splitbrain/phpfarm](https://hub.docker.com/r/splitbrain/phpfarm) gesetzt. Auf Grund der langen Updatezyklen war dies für mich aber auch nicht mehr die Lösung. Ich hätte gerne schneller die aktuellen PHP-Versionen!

Ich habe mir einige andere Lösungen angesehen und bin zu dem Schluß gekommen, dass sie entweder auch wieder verschiedene Unzulänglichkeiten haben, oder wahnsinnig kompliziert sind.

## Die Lösung

Für mich bestand die Lösung darin, die original [PHP-Images](https://hub.docker.com/_/php) für Docker zu nutzen und an meine Bedürfnisse anzupassen. Ich nutze die Images in der *`-apache`-Version. Die größte Herausforderung war das Installieren der PHP-Erweiterungen. Dies hat mich einige Zeit und Tüftelei bekostet. Aber der Reihe nach.

## Grundüberlegungen

Zunächst hatte ich überlegt, bei dieser Aktion gleich auf einen Container pro Projekt umzustellen. Dies würde bedeuten, dass man für jedes Projekt einen Container starten würde. Da ich in der Regel nur an einer handvoll Projekten pro Tag arbeite, wäre dies durch aus möglich. Zum Einen stört mich aber die Zeit, die es braucht, einen Container zu starten, wenn ein Kunde anruft und nur kurz eine Frage hat, zum Anderen müsste jeder Container einen eigenen Port verwenden, der auf einen Port auf dem Host gemappt wird. Man bräuchte also etwas wie [Traefik](https://traefik.io/). Dies erschien mir dann doch zu kompliziert.


### Name-based Virtual Host

Es bleibt bei mir also bei den [Name-based Virtual Host](https://httpd.apache.org/docs/2.4/vhosts/name-based.html#using), bei denen die URL auf ein Verzeichnis gemappt wird. Die Urls bestehen bei mir aus 4 Teilen:

| Teil der Url | Bedeutung | Beispiel |
| ------------ | --------- | -------- |
| 1 | Gruppe | cus => Customer, <br>dev => Development, <br>... |
| 2 | Name des Kunde | z.B. testkunde |
| 3 | Name des Projekts | z.B. testprojekt |
| 4 | vHosts | z.B. dev4 => Entwicklung in Contao 4, <br>install => Installationstests, <br>... |

Die Nummern werden später in die `conf`-Dateien für den Apache eingetragen. Eine Domain würde dann z.B. so aussehen:

```
http://cus.testkunde.testprojekt.dev4/
```

In den Konfigurationsdateien für den Apache sehe es dann ungefähr so aus:

```conf
VirtualDocumentRoot /mnt/easy.Projekte/%1/%2/%3/vhosts/%4/htdocs/web
```

Die Projekte lieg bei mir unter `/mnt/easy.Projekt/` und werden auch so in den Container eingebungen. Der Rest gibt die Verzeichnisstruktur wieder. Ich gehe später noch näher auf die Konfiguration ein.


## Dateiliste

Bevor ich auf die einzelnen Dateien eingehe, möchte ich hier zunächst einen Überblick über die verwendeten Dateien geben.

__Das ganze Projekt ist auch auf Github zu finden: [https://github.com/eS-IT/dev_env2018](https://github.com/eS-IT/dev_env2018)__

<script src="https://gist.github.com/eS-IT/5131faa2d1c6860b78dd460624e6f8bb.js"></script>

## Variablen

Ich habe die Variablen in die Datei `.secret.env` ausgelagert. Sie sieht wie folgt aus:

<script src="https://gist.github.com/eS-IT/9130556c7e0f458b845f20d7afc3b337.js"></script>

Die Zugangsdaten für den SQL-Server werden automatisch übernommen, so dass hier kein Dockerfile benötigt wird.

## Docker-Compose

In der `docker-compose.yml` wird zunächst ein Container für die MariaDB und dann für die verschiedenen PHP-Versionen erstellt. Da sich die verschiedenen Versionen nur in wenigen Punkten unterscheiden (u.a. Port, Ip, Dockerfile), werde ich hier nur einen Auszug aus der `docker-compose.yml` zeigen. Die ganze Datei kann __[hier](https://gist.github.com/eS-IT/1147f9ce40f2245a45e77947ae1546e0)__ gefunden werden.

<script src="https://gist.github.com/eS-IT/ad0dcb838fa67e4b818a136c7777a6e8.js"></script>

Wichtig ist hier die Variablen `APACHE_RUN_USER` und `APACHE_RUN_GROUP` zu setzen, damit es später keine Probleme mit den Dateiberechtigungen gibt. Sonst wird noch der Name des Dockerfiles gesetzt (`Dockerfile72`). Bei mir liegen die Datei für PHP in dem Unterordner `php` (`context: "php/"`). Dort liegen auch die Dockerfiles. Außerdem wird noch der Port `8072` und das Verzeichnis in dem meine Projekte liegen `/mnt/easy.Projekte` angegeben.

## Dockerfiles

Es gibt für jede PHP-Version ein Dockerfile mit den Anweisungen, wie das Image erstellt weden soll. Dies liegt vor allem an den verschiedene PHP-Erweiterugnen, bzw. daran wie diese genau installiert werden. Ich zeige hier wieder exemplarisch das Dockerfile der Version 7.2.

<script src="https://gist.github.com/eS-IT/f7bf2bc3712a8828b49ae5181dbedfac.js"></script>

In Zeile 4 wählen wir das PHP-Image aus. In Zeile 14 und 15 werden User und Gruppe gesetzt, damit die Dateirechte innerhalb des Containers stimmen. Falls diese nicht in der `docker-compose.yml` gesetzt werden, wird ein Vorgabewert angegeben. In Zeile 19 wird der Benutzer entsprechend modifiziert, dass er die neue Uid und Gid nutzt. Da bei mir beim Build immer die Fehlermeldung kam, dass die Zeitzone nicht gesetzt ist, wird dies in Zeile 24 nachgeholt.

In Zeile 29 werden die nötigen Bibliotheken, sowie weitere Abhängigkeiten installiert. Diese Zeile ist je nach PHP-Version etwas unterschiedlich. Danach werden in Zeile 36 bis 88 die einzelnen PHP-Erweiterungen installiert. Auch hier gibt es je nach PHP-Version kleine Unterschiede. Es hat mich, wie gesagt, einige Zeit und Mühe gekostet dies genau auszutüfteln.

Nun werden die Einstellungen für PHP festgelegt. Es wird zuerst in Zeile 93 die Ini-Datei `php.ini-development` nach `php.ini` verschoben, um die Grundkonfiguration festzulegen. Dann füge ich in Zeile 98 eine weitere `php.ini` mit meine Anpassungen von außen in den Container ein. Die Datei trägt die Nummer der PHP-Version, also z.B. `php-72.ini` und liegt im Unterordner `php/etc`. Auf diese Datei gehe ich weiter unten ein (PHP-INI).

Die Konfigurationsdatei für den Apache wird in Zeile 103 eingefügt. Auch auf diese gehe ich später noch ein (default.conf weiter unten).

Damit die Name-based Virtual Hosts funktionieren, muss in Zeile 108 das entsprechende Modul eingeschaltet werden. In Zeile 113 wird noch das Modul `mod_rewrite` eingeschaltet.

In Zeile 118 wird zum Schluß der Server neu gestartet.

Hier eine Liste zu allen Dockerfiles:

- [/php/Docker56](https://gist.github.com/eS-IT/670e1ca11ec6c93de2517e14fbc1b247)
- [/php/Docker70](https://gist.github.com/eS-IT/35cd47df62ed7a947c4ae47aedbc1c0d)
- [/php/Docker71](https://gist.github.com/eS-IT/d7c7fafbc36798f35ce5dc90c1aa0199)
- [/php/Docker72](https://gist.github.com/eS-IT/f7bf2bc3712a8828b49ae5181dbedfac)
- [/php/Docker73](https://gist.github.com/eS-IT/d404faf4b1f844a2fb253c6eaf6b938a)


## PHP-INI

Hier werden die Änderungen an den PHP-Einstellungen eingefügt. Es können beliebige Werte überschrieben werden. Ich setzte bei allen PHP-Versionen die gleichen Werte, so dass sich die Dateien für die Versionen nur durch den Pfad für Xdbug unterscheiden.

<script src="https://gist.github.com/eS-IT/7117f23f0b4818202b041ff63f952b06.js"></script>

Hier eine Liste zu allen Ini-Dateien:

- [/php/etc/php-56.ini](https://gist.github.com/eS-IT/e712c43fdd6e00bc1f1ecb95f439021d)
- [/php/etc/php-70.ini](https://gist.github.com/eS-IT/0ea8a61a1ef4bda7306953dee878a835)
- [/php/etc/php-71.ini](https://gist.github.com/eS-IT/76795e565d169b057af6bf96c95e0f0b)
- [/php/etc/php-72.ini](https://gist.github.com/eS-IT/7117f23f0b4818202b041ff63f952b06)
- [/php/etc/php-73.ini](https://gist.github.com/eS-IT/87af03e36d8fb33c283a882774c040bd)

## default.conf

Hier werden die Einstellungen für den Server gesetzt. Die Dateien für die einzelnen Container unterscheiden sich nur durch den Port. Sie liegen im Unterordner `php/sites-enabled`.

<script src="https://gist.github.com/eS-IT/3fffb41cc965621519cb0e6d42cf7e47.js"></script>

Zeile 1 ist wichtig, damit das Name-based Virtual Host-Setup funktioniert. In Zeile 7 wird der Port gesetzt und in Zeile 9 startet die Konfiguration des Hosts. Wichtig ist es, auch hier den Port einzutragen. In Zeile 10 wird die Url auf die Pfade gemappt. Der Rest ist Standard.

Hier eine Liste aller conf-Dateien:

- [/php/sites-enabled/8056-default.conf](https://gist.github.com/eS-IT/6ebddcf7901dcbea1e2c2de5d226415e)
- [/php/sites-enabled/8070-default.conf](https://gist.github.com/eS-IT/696c6a8f0c208e361547f194484eae3a)
- [/php/sites-enabled/8071-default.conf](https://gist.github.com/eS-IT/1774667bd30e59797432194f398ad2dd)
- [/php/sites-enabled/8072-default.conf](https://gist.github.com/eS-IT/3fffb41cc965621519cb0e6d42cf7e47)
- [/php/sites-enabled/8073-default.conf](https://gist.github.com/eS-IT/9b75675d5929fb8100136f07a265e0aa)

## Update

Will man ein Update auf die neusten Bugfixverseionen durchführen, reicht es, die Container zu löschen und neu zu erstellen.

## Neue PHP-Version

Es werden standardmäßig immer die aktuellsten Bugfixversionen installiert. Will man einen neue Version (z.B. PHP 7.4) oder eine spezielle Bugfixversion (z.B. 7.2.12) müssen folgenden Schritte befolgt werden:

### docker-compose.yml

- Neuen Abschnitt in der Datei `docker-compose.yml` anlegen
- Containername anpassen [steht oben in Zeile 2]
- Name des Images ändern [steht oben in Zeile 3]
- Dockerfile eintragen [steht oben in Zeile 6]
- Port anpassen [steht oben in Zeile 10]
- Neue IP angeben [steht oben in Zeile 16]

### Dockerfile

- Neues Dockerfile im Unterordner `/php` anlegen
- Gewünschte Grundimage eintragen [Zeile 4]
- Neue `php.ini` für die Version eintragen [Zeile 100]
- Neue `conf`-Datei für die Version eintragen [Zeile 105]

### php.ini

- Neue `php.ini` im Unterordner `/php/etc` anlegen
- Pfad zu Xdebug anpassen [Zeilen 82 und 83]

### conf-Datei

- Neue `conf`-Datei im Unterordner `php/sites-enabled` anlegen
- Port anpassen [Zeilen 7 und 9]

Fertig! Nun wird beim nächsten Start der neue Container mit der gewünschten PHP-Version erstellt.

## Fazit

Mit diesem Setup können leicht Docker-Container für jede beliebige PHP-Version erstellt werden. Es können neue PHP-Versionen direkt ab Release verwendet werden, ohne auf Updates eines externen Projekts, oder anderer Abhängigkeiten warten zu müssen. Durch die Verwendung der Name-based Virtual Hosts, können neue Hosts einfach durch Erstellen von Ordnern (und Eintrag in der `hosts`-Datei) erstellt werden. So ist ein großes Maß an Flexibilität gewährleistet.
