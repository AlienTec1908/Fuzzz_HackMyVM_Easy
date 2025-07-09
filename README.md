# fuzzz - HackMyVM Writeup

![Fuzzz Icon](Fuzzz.png)

## Übersicht

*   **VM:** fuzzz
*   **Plattform:** [HackMyVM](https://hackmyvm.eu/machines/machine.php?vm=fuzzz)
*   **Schwierigkeit:** Easy
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 02. Juli 2025
*   **Original-Writeup:** https://alientec1908.github.io/Fuzzz_HackMyVM_Easy/
*   **Autor:** Ben C.

---

**Disclaimer:**

Dieser Writeup dient ausschließlich zu Bildungszwecken und dokumentiert Techniken, die in einer kontrollierten Testumgebung (HackTheBox/HackMyVM) angewendet wurden. Die Anwendung dieser Techniken auf Systeme, für die keine ausdrückliche Genehmigung vorliegt, ist illegal und ethisch nicht vertretbar. Der Autor und der Ersteller dieses README übernehmen keine Verantwortung für jeglichen Missbrauch der hier beschriebenen Informationen.

---

## Zusammenfassung

Die Box "fuzzz" bot einen interessanten Einstiegspunkt über den Android Debug Bridge (ADB) Dienst. Nach der initialen Enumeration wurde ADB auf Port 5555 als offen identifiziert und erlaubte eine Shell-Verbindung als Benutzer `runner`. Von dieser Shell aus wurde ein lokal laufender Webserver auf Port 80 entdeckt.

Durch den Einsatz von Port Forwarding über einen SSH-Tunnel wurde die lokal gebundene Webanwendung zugreifbar gemacht. Diese Python-Webapp, die als Benutzer `asahi` lief, wies eine Path Injection Schwachstelle auf Endpunkten wie `/lineX` auf. Durch charakterbasiertes Fuzzing der anfälligen URLs konnte ein Base64-kodierter SSH Private Key rekonstruiert werden.

Der rekonstruierte Schlüssel erlaubte den SSH-Login als Benutzer `asahi`. Als `asahi` wurde eine `sudo`-Regel gefunden, die die Ausführung von `/usr/local/bin/lrz` als Root ohne Passwort erlaubte. Diese Binary, Teil des lrzsz-Pakets für ZMODEM-Übertragungen, konnte im Server-Modus gestartet werden und erlaubte das Anhängen von Daten an Dateien. Dies wurde ausgenutzt, um einen neuen Root-Benutzer mit bekanntem Passwort-Hash zu `/etc/passwd` hinzuzufügen und so Root-Zugriff zu erlangen.

## Technische Details

*   **Betriebssystem:** Alpine Linux / Android (basierend auf uname -a und ADB Service)
*   **Offene Ports:**
    *   `22/tcp`: SSH (OpenSSH 9.9)
    *   `5555/tcp`: ADB (Android Debug Bridge)
    *   `80/tcp`: HTTP (Nginx / uWSGI Python Webapp) - lokal gebunden

## Enumeration

1.  **ARP-Scan:** Identifizierung der Ziel-IP (192.168.2.69) im Netzwerk.
2.  **`/etc/hosts` Eintrag:** Hinzufügen von `fuzzz.hmv` zur lokalen hosts-Datei.
3.  **Nmap Scan:** Identifizierung offener Ports 22 (SSH) und 5555 (ADB). Der ADB-Dienst war offen, erforderte aber Token-Authentifizierung (die bei direktem `adb connect` oft automatisch gehandhabt wird). SSH 9.9 hatte bekannte, aber in diesem Fall nicht genutzte CVEs laut Vulners-Script.
4.  **ADB Zugang:** `adb connect 192.168.2.69:5555` war erfolgreich, was auf eine Standard-ADB-Konfiguration hindeutet.
5.  **ADB Shell:** Mittels `adb shell` konnte eine Shell als Benutzer `runner` (UID 1000) erlangt werden.
6.  **Systemerkundung als `runner`:** Netstat zeigte, dass ein Dienst auf `127.0.0.1:80` lauschte. Pspy (`./pspy64`) identifizierte den Prozess `2585 asahi /usr/sbin/uwsgi --plugin python3 --http-socket 127.0.0.1:80 --wsgi-file /opt/webapp/app.py --callable app`, was auf eine lokal gebundene Python-Webapp, laufend als Benutzer `asahi` (UID 1001), hindeutete.

## Initialer Zugriff (SSH Key aus Webapp)

1.  **Port Forwarding:** Da die Webanwendung nur lokal zugänglich war, wurde `chisel` verwendet, um den lokalen Port 80 der Box auf einen Port des Angreifers umzuleiten: `./chisel client 192.168.2.199:9001 R:8080:127.0.0.1:80`.
2.  **Lokale Webapp Enumeration (via Tunnel):** Über den Tunnel (z.B. `http://localhost:8080/`) wurde die Webapp mit feroxbuster und manuellen Tests erkundet. Endpunkte `/line1`, `/line2`, `/line3`, `/line4`, `/line5` lieferten bei GET-Anfragen ohne weiteren Pfad einen 200 OK Status mit Content-Length 0.
3.  **Path Injection Vulnerability:** Es wurde festgestellt, dass diese Endpunkte (`/lineX/`) zusätzliche Pfadsegmente akzeptierten, ohne einen 404-Fehler zu werfen, solange die Zeichen im Pfad aus einem bestimmten Alphabet (Base64-Zeichen) stammten. Längere Pfade oder ungültige Zeichen führten zu Fehlern.
4.  **Extraktion des SSH Keys:** Ein Bash-Skript (`fuzz.sh`) wurde entwickelt, um diese Path Injection zu nutzen. Es iterierte durch Base64-Zeichen und fügte sie schrittweise an den Pfad an (`/lineX/FOUND_STRING`). Wenn die Antwort weiterhin 200 OK und leer war, wurde das Zeichen als korrekt angenommen. Dies ermöglichte die Extraktion von 5 Base64-Strings (jeweils 70 Zeichen lang).
5.  **SSH Key Rekonstruktion:** Die extrahierten Base64-Strings wurden konkateniert und in das Format eines OpenSSH Private Keys gebracht (`-----BEGIN OPENSSH PRIVATE KEY----- ... -----END OPENSSH PRIVATE KEY-----`), um die Datei `idasahi` zu erstellen.

## Lateral Movement (runner -> asahi)

1.  **SSH Login als asahi:** Mit dem rekonstruierten privaten SSH-Schlüssel (`idasahi`) konnte erfolgreich eine SSH-Verbindung als Benutzer `asahi` (UID 1001) hergestellt werden.
2.  **User Flag:** Die User Flag (`user.flag`) wurde im Home-Verzeichnis von asahi (`/home/asahi/`) gefunden.

## Privilegieneskalation (asahi -> root)

1.  **Sudo-Regel für asahi:** Als Benutzer `asahi` wurde `sudo -l` ausgeführt. Eine entscheidende Regel wurde gefunden: `(ALL) NOPASSWD: /usr/local/bin/lrz`. asahi konnte den Befehl `/usr/local/bin/lrz` als jeder Benutzer, einschließlich Root, ohne Passwort ausführen.
2.  **lrz Analyse:** `/usr/local/bin/lrz` ist das ZMODEM-Empfangs-Binary. Es kann im TCP-Server-Modus gestartet werden (`--tcp-server`), um Dateien über das Netzwerk zu empfangen und an eine lokale Datei anzuhängen (`--append`).
3.  **'/etc/passwd' Manipulation:** Die Idee war, `/etc/passwd` zu manipulieren, um einen neuen Root-Benutzer mit einem bekannten Passwort hinzuzufügen. Dazu wurde eine Kopie der aktuellen `/etc/passwd` erstellt, ein neuer Eintrag für einen Benutzer `Dark` mit UID/GID 0 und einem Hash für das Passwort `alpine` (`$6$EZdVo4XckcU2BJJi$IanX1gZA.t1nk2EgRy1SBDPGa69dLrCqv3eOznvqru062GCQ6Eh7VQyXI3lKgsdItq3F/uMWs/VU/TR2E1tzF0`) sowie `/bin/sh` als Shell hinzugefügt.
4.  **Ausnutzung mit lrz & sz:**
    *   Ein lrz-Server wurde als Root auf der Box gestartet, der bereit war, an `/etc/passwd` anzuhängen: `sudo /usr/local/bin/lrz --tcp-server --append /etc/passwd`.
    *   Von der Angreifer-Maschine aus wurde eine Datei, die den neuen Benutzer-Eintrag enthielt, über ZMODEM an den lrz-Server auf der Box gesendet: `sz --tcp-client fuzzz.hmv:<lrz_port> /tmp/passwd`. Der Port wird von lrz beim Start angezeigt.
    *   Der neue Benutzer-Eintrag wurde erfolgreich an `/etc/passwd` angehängt.
5.  **Root-Zugang:** Mit dem Befehl `su Dark` und dem Passwort `alpine` konnte zum Root-Benutzer gewechselt werden.

## Finalisierung (Root Shell)

1.  **Root-Zugriff:** Als Root konnte das Dateisystem vollständig erkundet werden.
2.  **Flag Funde:** Das Root-Flag (`root.flag`) wurde im `/root`-Verzeichnis und die User Flag (`user.flag`) im `/home/asahi`-Verzeichnis gefunden.

## Flags

*   **user.flag:** `flag{da39a3ee5e6b4b0d3255bfef95601890afd80709}` (Gefunden unter `/home/asahi/user.flag`)
*   **root.flag:** `flag{46a0e055d5db8d82eee6e7eb3ee3ccf64be3fca2}` (Gefunden unter `/root/root.flag`)

---
