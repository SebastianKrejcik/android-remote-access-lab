# Android Remote-Access Lab

Aufbau eines SSH-Zugriffs auf ein Android-Gerät (Termux) über ein Hotspot-Netz mit Client-Isolation, gelöst über einen Reverse-SSH-Tunnel.

## Ausgangsproblem

- Android-Smartphone stellt einen WLAN-Hotspot bereit
- PC (Zorin OS) verbindet sich mit dem Hotspot und erhält die IP `10.52.54.46`
- Android-Gerät selbst hat im Hotspot-Netz die IP `10.52.54.55`
- Direkter SSH-Zugriff `ssh -p 8022 10.52.54.55` vom PC aus schlägt fehl (Timeout)

## Analyse

Fehlersuche mit klassischen Linux-Netzwerktools:

```bash
ip route
ssh -vvv -p 8022 10.52.54.55
nc -zv 10.52.54.55 8022
```

**Ergebnis:** 

Der Android-Hotspot isoliert eingehende Verbindungen von Hotspot-Clients zum Hotspot-Host selbst (Client-Isolation). 
Der Port ist lokal auf dem Android-Gerät offen, aber vom Hotspot-Client aus nicht erreichbar.

## Lösung: Reverse-SSH-Tunnel

Da die direkte, eingehende Verbindung durch die Hotspot-Isolation blockiert wird, 
wird stattdessen eine **ausgehende** Verbindung vom Android-Gerät über einen öffentlichen Tunnel-Anbieter (Pinggy) aufgebaut:

Zorin-PC
   |
   | SSH -> oeffentlicher Tunnel-Port
   v
Pinggy (Tunnel-Anbieter)
   |
   | Reverse SSH Tunnel
   v
Android / Termux (u0_a331)
   |
   +-- localhost:8022

Auf dem Android-Gerät (Termux) wird der Tunnel aufgebaut, der eine öffentliche Adresse bereitstellt. 
Der PC verbindet sich dann über diese öffentliche Adresse statt direkt über die (nicht erreichbare) lokale Hotspot-IP.

## Eingesetzte Konzepte

- SSH-Server-Betrieb unter Termux (Linux-Userland auf Android)
- SSH-Authentifizierung (Passwort-basiert)
- NAT und Hotspot-Client-Isolation als Troubleshooting-Fall
- Reverse-SSH-Tunneling (`ssh -R`) über einen öffentlichen Tunnel-Anbieter
- Netzwerkdiagnose mit `ip route`, `ssh -vvv`, `nc`
- Unterscheidung lokale vs. eingehende Erreichbarkeit
- Betrieb ohne Root-Rechte (App-Sandbox-Berechtigungsmodell von Android/Termux)

## Erkenntnis

Android basiert zwar auf dem Linux-Kernel, das Berechtigungs- und Netzwerkmodell der App-Umgebung (Termux) unterscheidet sich aber deutlich von einem klassischen Linux-Server
– u. a. durch fehlende Root-Rechte und die Isolation im Hotspot-Netz. 

Das Problem war nicht SSH-Konfiguration, sondern Netzwerk-Erreichbarkeit – gelöst durch Umkehrung der Verbindungsrichtung (Reverse Tunnel) statt durch Änderungen an der SSH-Konfiguration selbst.

## Hinweis

## Sicherheit

Dieses Setup wurde nur in einer eigenen Lernumgebung getestet:

- eigenes Android-Gerät
- eigener Hotspot
- keine fremden Systeme
- keine produktiven Daten

Der öffentliche Tunnel ist nur für Testzwecke geeignet.
Für produktive Einsätze wären zusätzliche Maßnahmen nötig, z. B.:

- Schlüsselbasierte SSH-Authentifizierung statt Passwort
- zeitlich begrenzte Tunnel
- minimale Berechtigungen
- keine Freigabe sensibler Ports
- idealerweise VPN-Lösungen wie WireGuard/Tailscale statt öffentlichem Tunnel
