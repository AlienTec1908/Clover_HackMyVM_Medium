# Clover - HackMyVM (Medium)

![Clover Icon](Clover.png)

## Übersicht

*   **VM:** Clover
*   **Plattform:** [HackMyVM](https://hackmyvm.eu/machines/machine.php?vm=Clover)
*   **Schwierigkeit:** Medium
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 26. März 2021
*   **Original-Writeup:** https://alientec1908.github.io/Clover_HackMyVM_Medium/
*   **Autor:** Ben C.

## Kurzbeschreibung

Die virtuelle Maschine "Clover" von HackMyVM (Schwierigkeitsgrad: Medium) bot einen mehrstufigen Weg zur Kompromittierung. Der Einstieg begann mit der Entdeckung von Hinweisen durch Steganographie in Web-Bilddateien und einer SQL-Injection-Schwachstelle (vermutlich in einem ColdFusion-Admin-Panel), die zur Offenlegung von Benutzer-Credentials führte. Der initiale Zugriff erfolgte via SSH mit einem dieser Credentials. Die Privilegienerweiterung zu Root wurde durch die Ausnutzung eines SUID-gesetzten Lua-Skripts (`deamon.sh`) erreicht, das die Ausführung von Shell-Befehlen mit Root-Rechten erlaubte.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `nmap` (impliziert)
*   `gobuster`
*   `curl`
*   `nikto` (impliziert oder nicht direkt im Log)
*   `stegsnow`
*   `cat`
*   Burp Suite (für Request Capturing)
*   `sqlmap`
*   `hydra`
*   `ssh`
*   `vi`
*   `ssh-keygen`
*   Standard Linux-Befehle (`ls`, `whoami`, `id`, `cd`, `su`, `bash`)

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Clover" erfolgte in mehreren Schritten:

1.  **Reconnaissance & Web Enumeration:**
    *   Identifizierung der Ziel-IP (`192.168.2.115`) via `arp-scan`.
    *   `gobuster` auf Port 80 fand `/index.html`, `/status` und `/robots.txt`.
    *   `robots.txt` (impliziert) verwies auf das Verzeichnis `/webmaster/`.
    *   Im Verzeichnis `/webmaster/` wurden Bilddateien (`christina-wocintechchat-com-unsplash-1.jpg`, `-2.jpg`) gefunden.

2.  **Steganographie & Informationsleck:**
    *   Mittels `stegsnow` wurden aus den Bilddateien die Zeichenketten `s0sy` und `tsst` extrahiert.
    *   Eine Datei `locale.txt` (Fundort unklar im Log, vermutlich im Web-Root) enthielt die Base64-kodierte Zeichenkette `cGluZyBwb25n` (dekodiert: `ping pong`) und die Namen `monet`, `tiffany`, `jen`.

3.  **SQL Injection:**
    *   Ein anfälliger Endpunkt (vermutlich `/CFIDE/Administrator/` oder eine ähnliche ColdFusion-Funktion) wurde identifiziert. Ein Request wurde mit Burp Suite abgefangen und in `clover.sql` gespeichert.
    *   `sqlmap -r clover.sql --batch --dbs` listete Datenbanken auf, darunter `clover`.
    *   `sqlmap -r clover.sql --batch --tables -D clover` zeigte die Tabelle `users`.
    *   `sqlmap -r clover.sql --batch --dump -T users -D clover` extrahierte Benutzerdaten:
        *   `0xBush1do`:`33a41c7507cy5031d9tref6fdb31880c`
        *   `asta`:`asta$$123` (Klartextpasswort oder sqlmap Artefakt)
        *   `0xJin`:`92ift37507ad7031d9decf98setf4w0c`

4.  **Initial Access (SSH):**
    *   Trotz eines nicht erfolgreichen `hydra`-Versuchs im Log wurde eine SSH-Verbindung als Benutzer `sword` mit dem Passwort `P4SsW0rD4286` hergestellt. Die genaue Herkunft dieses Passworts ist aus den Logs nicht eindeutig ersichtlich, könnte aber durch weitere, nicht dokumentierte Enumeration oder als separater Hinweis bekannt geworden sein. (Alternativ hätte `asta:asta$$123` getestet werden können).

5.  **Privilege Escalation Enumeration (als sword/asta):**
    *   Als `sword` wurde die User-Flag (`34f35ca9ea7febe859be7715b707d684`) gefunden (in `local2.txt` und später als `asta` in `local.txt`).
    *   Ein Wechsel zum Benutzer `asta` erfolgte (vermutlich mit `su asta` und dem Passwort `asta$$123`).
    *   Im Verzeichnis `/usr/games/clover/` wurde das Skript `deamon.sh` entdeckt.

6.  **Privilege Escalation (deamon.sh - Root):**
    *   Das Skript `deamon.sh` war ein Lua-Skript, das die Ausführung von Code über die `-e` Option erlaubte.
    *   Das Skript hatte das SUID-Bit gesetzt und gehörte `root`.
    *   Durch Ausführung von `./deamon.sh -e 'os.execute("/bin/sh")'` als Benutzer `sword` wurde eine Shell mit effektiver UID (`euid`) von `0` (root) erlangt.
    *   Die Root-Flag wurde aus `/root/proof.txt` gelesen.

## Wichtige Schwachstellen und Konzepte

*   **Steganographie:** Versteckte Informationen in Bilddateien (`stegsnow`).
*   **SQL Injection:** Ausnutzung einer SQLi-Schwachstelle (vermutlich in ColdFusion) zum Auslesen von Benutzerdaten.
*   **Informationslecks:** Preisgabe von potenziellen Benutzernamen und Hinweisen in Textdateien.
*   **Schwache/Kompromittierte Passwörter:** Ermöglichten SSH-Zugriff.
*   **SUID-Exploitation:** Ein SUID-gesetztes Skript (`deamon.sh`), das die Ausführung von beliebigem Code (hier Lua, das dann Shell-Befehle ausführt) mit Root-Rechten erlaubte.
*   **Lua Code Injection:** Durch die `-e` Option des `deamon.sh`-Skripts.

## Flags

*   **User Flag (für `asta`, gefunden als `sword` und `asta`):** `34f35ca9ea7febe859be7715b707d684`
*   **Root Flag (`/root/proof.txt`):** `974bd350558b912740f800a316c53afe`

## Tags

`HackMyVM`, `Clover`, `Medium`, `Steganography`, `SQL Injection`, `ColdFusion`, `SSH`, `SUID`, `Lua`, `Privilege Escalation`, `Web`, `Linux`
