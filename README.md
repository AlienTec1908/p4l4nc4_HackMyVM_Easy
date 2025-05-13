# p4l4nc4 - HackMyVM (Easy)

![p4l4nc4.png](p4l4nc4.png)

## Übersicht

*   **VM:** p4l4nc4
*   **Plattform:** HackMyVM (https://hackmyvm.eu/machines/machine.php?vm=p4l4nc4)
*   **Schwierigkeit:** Easy
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 2025-01-28
*   **Original-Writeup:** https://alientec1908.github.io/p4l4nc4_HackMyVM_Easy/
*   **Autor:** Ben C.

## Kurzbeschreibung

Das Ziel dieser Challenge war es, Root-Rechte auf der Maschine "p4l4nc4" zu erlangen. Der Weg dorthin begann mit der Entdeckung eines versteckten Verzeichnisses (`/n3gr4/`) auf dem Webserver, das ein PHP-Skript (`m414nj3.php`) mit einer Local File Inclusion (LFI)-Schwachstelle enthielt. Diese LFI wurde mittels einer PHP-Filterkette zu Remote Code Execution (RCE) eskaliert, was zu einer Shell als `www-data` führte. Als `www-data` wurde im Home-Verzeichnis des Benutzers `p4l4nc4` ein passwortgeschützter privater SSH-Schlüssel (`id_rsa`) gefunden. Nach dem Knacken der Schlüssel-Passphrase (`friendster`) mit `ssh2john` und `john` wurde SSH-Zugriff als `p4l4nc4` erlangt. Die finale Rechteausweitung zu Root gelang durch Ausnutzung unsicherer Dateiberechtigungen auf `/etc/passwd` (`-rw-rw-rw-`), was das Hinzufügen eines neuen Benutzers mit UID 0 ermöglichte.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `nmap`
*   `curl`
*   `nikto`
*   `gobuster`
*   `dirb`
*   `dirsearch`
*   `wfuzz`
*   `php_filter_chain_generator.py`
*   `nc` (netcat)
*   Python3 (`pty` Modul)
*   `stty`
*   `ssh2john`
*   `john`
*   `ssh`
*   Standard Linux-Befehle (`vi`/`nano`, `cat`, `find`, `ls`, `id`, `su`, `echo`, `grep`)

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "p4l4nc4" gliederte sich in folgende Phasen:

1.  **Reconnaissance & Web Enumeration:**
    *   IP-Adresse des Ziels (192.168.2.165) mit `arp-scan` identifiziert. Hostname `plpl.hmv` (oder `4ng014.hmv`) in `/etc/hosts` eingetragen. IPv6-Erreichbarkeit wurde ebenfalls bestätigt.
    *   `nmap`-Scan offenbarte Port 22 (SSH, OpenSSH 9.2p1) und Port 80 (HTTP, Apache 2.4.62).
    *   Web-Enumeration mit `gobuster`, `dirb` und `dirsearch` fand `/robots.txt` und `/server-status` (403). Ein Hinweis auf "4ng0l4" (Angola) wurde extern notiert.
    *   Ein `gobuster`-Scan mit einer benutzerdefinierten Wortliste (`1337_format.txt`) fand das versteckte Verzeichnis `/n3gr4/`.
    *   Ein weiterer `gobuster`-Scan auf `/n3gr4/` fand die PHP-Datei `m414nj3.php` (Status 500).

2.  **Initial Access (LFI zu RCE als `www-data`):**
    *   Mittels `wfuzz` wurde der GET-Parameter `page` für `m414nj3.php` gefunden, der anfällig für Local File Inclusion (LFI) war (`m414nj3.php?page=../../.../etc/passwd`).
    *   Der Inhalt von `/etc/passwd` wurde ausgelesen und der Benutzer `p4l4nc4` identifiziert.
    *   Mittels `php_filter_chain_generator.py` wurde eine komplexe PHP-Filterkette für den Payload `<?php system($_GET["cmd"]); ?>` erstellt.
    *   Die Filterkette wurde als Wert für den `page`-Parameter an `m414nj3.php` übergeben (`...?page=php://filter/.../resource=php://temp&cmd=id`), was RCE als `www-data` ermöglichte.
    *   Eine Bash-Reverse-Shell wurde zu einem Netcat-Listener (Port 5555) aufgebaut und stabilisiert. Der Quellcode von `m414nj3.php` bestätigte die LFI (`include($_GET['page']);`).

3.  **User Access (SSH-Key Crack & SSH als `p4l4nc4`):**
    *   Als `www-data` wurde das Home-Verzeichnis `/home/p4l4nc4/` und darin das Unterverzeichnis `.ssh/` mit den Dateien `id_rsa` (privater Schlüssel) und `id_rsa.pub` gefunden.
    *   Der private Schlüssel `id_rsa` wurde auf das Angreifer-System kopiert.
    *   Mittels `ssh2john id_rsa > hash` wurde der Passphrasen-Hash des Schlüssels extrahiert.
    *   `john --wordlist=rockyou.txt hash` knackte die Passphrase: `friendster`.
    *   Erfolgreicher SSH-Login als `p4l4nc4` mit dem privaten Schlüssel und der Passphrase `friendster`.
    *   Die User-Flag (`HMV{6cfb952777b95ded50a5be3a4ee9417af7e6dcd1}`) wurde in `/home/p4l4nc4/user.txt` gefunden.

4.  **Privilege Escalation (von `p4l4nc4` zu `root` via `/etc/passwd` Manipulation):**
    *   Als `p4l4nc4` wurde festgestellt, dass die Datei `/etc/passwd` für alle Benutzer beschreibbar war (`-rw-rw-rw-`).
    *   Die Datei `/etc/passwd` wurde mit `nano` bearbeitet und ein neuer Eintrag für einen Benutzer `fuck` mit UID 0, GID 0 und einem bekannten Passwort-Hash (SHA512crypt) hinzugefügt: `fuck:$6$EZdVo...:0:0:root:/root:/bin/bash`.
    *   Mittels `su fuck` und Eingabe des zugehörigen Passworts (implizit bekannt oder erraten) wurde Root-Zugriff erlangt.
    *   Die Root-Flag (`HMV{4c3b9d0468240fbd4a9148c8559600fe2f9ad727}`) wurde in `/root/root.txt` gefunden.

## Wichtige Schwachstellen und Konzepte

*   **Local File Inclusion (LFI) mit PHP Filter Chains:** Eine LFI-Schwachstelle in einem PHP-Skript wurde durch Verwendung komplexer PHP-Filterketten zu Remote Code Execution (RCE) eskaliert.
*   **Exponierter privater SSH-Schlüssel:** Der private SSH-Schlüssel eines Benutzers (`p4l4nc4`) war für den Webserver-Benutzer lesbar.
*   **Passwort-Cracking (SSH-Key):** Die Passphrase eines verschlüsselten privaten SSH-Schlüssels wurde mit `ssh2john` und `john` geknackt.
*   **Unsichere Dateiberechtigungen (`/etc/passwd`):** Die Datei `/etc/passwd` war für alle Benutzer beschreibbar, was das Hinzufügen eines neuen Benutzers mit Root-Rechten ermöglichte.
*   **Versteckte Verzeichnisse/Dateien:** Entdeckung des Verzeichnisses `/n3gr4/` und der Datei `m414nj3.php` durch gezieltes Fuzzing.

## Flags

*   **User Flag (`/home/p4l4nc4/user.txt`):** `HMV{6cfb952777b95ded50a5be3a4ee9417af7e6dcd1}`
*   **Root Flag (`/root/root.txt`):** `HMV{4c3b9d0468240fbd4a9148c8559600fe2f9ad727}`

## Tags

`HackMyVM`, `p4l4nc4`, `Easy`, `LFI`, `PHP Filter Chain`, `RCE`, `SSH Key Crack`, `/etc/passwd writable`, `Linux`, `Web`, `Privilege Escalation`, `Apache`
