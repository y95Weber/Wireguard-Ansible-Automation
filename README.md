# Wireguard-Ansible-Automation
Das ist das Repo für das 2.te Projekt im Fach Netzwerkbetriebssysteme des Dozenten Oliver Bücher. Manuel Sager und Yves Weber sind die Gründer und Betreiber dieses Repos


## Wichtiges
1. Beim ausführen muss auf der Admin-VM iptabels installiert sein! 
2. ssh-copy-id xy-user@192.168.114.X  Muss nach der hinzufügung im inventory noch je User ausgeführt werden! 

# WireGuard Ansible Automation
Automatisiertes Deployment eines WireGuard-VPN-Setups per Ansible. Ein Playbook
richtet den Server (Admin-VM) ein, deployt VPN-Clients auf beliebig vielen
Ziel-VMs und verknüpft beide Seiten automatisch über Public-Key-Austausch.
Bei erneutem Durchlauf werden bestehende Configs inoriert.

## Funktionsweise
**1. Server-Rolle (`wg-server`)**
Installiert WireGuard, generiert ein Keypair (falls noch keins existiert),
aktiviert IP-Forwarding und startet das Interface `wg0`. Läuft auf der
Admin-VM, lokal via `ansible_connection=local`.

**2. Client-Rolle (`wg-client`)**
Installiert WireGuard, legt den User `VPNUser` an, generiert ein eigenes
Keypair und baut daraus die Client-Config — inklusive Endpoint und Public Key
des Servers (per `hostvars` aus Schritt 1 ausgelesen).

**3. Peer Verknüpfung (im `playbook.yml`, Play 3)**
Nach dem Client-Deployment kennt der Server die Clients noch nicht. Ein
separater Play läuft erneut auf der Admin-VM und trägt für jeden Host in der
Inventory-Gruppe `wg_clients` einen `[Peer]`-Block in die Server-Config ein —
per `blockinfile` mit eindeutigem `marker` pro Host. Das verhindert Duplikate
bei mehrfachem Playbook-Lauf (Idempotenz) und macht das Setup beliebig
erweiterbar: neuer Host im Inventory → neuer Peer-Block, ohne Codeänderung.

**Warum `hostvars`:** Ansible speichert während eines Playbook-Laufs Fakten
zu jedem Host. So kann der Server-Play auf den Public Key zugreifen, den ein
Client-Host in einem früheren Play gesetzt hat — ohne Datei-Transfer oder
manuelles Copy-Paste.

## Modularität

Alle Parameter (IP-Bereiche, Server-Endpoint, Usernamen) liegen in `vars.yml`.
Hosts, IPs und SSH-User sind pro VM frei im `inventory.ini` definierbar
(`ansible_host`, `ansible_user`, `wg_vpn_ip`) — eine neue VM  erfordert nur einen neuen Inventory-Eintrag, kein Code-Änderung.

## Voraussetzungen pro Ziel-VM

- Statische IP im selben Netzwerk wie die Admin-VM
- SSH erreichbar, User mit sudo-Rechten
- SSH-Key der Admin-VM hinterlegt (`ssh-copy-id`)
- Python3 installiert

## Ausführen
```
ansible-playbook -i inventory.ini playbook.yml --ask-become-pass
```