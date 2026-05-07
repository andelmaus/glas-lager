# SSH/Ping nicht erreichbar – Troubleshooting Uni-Netz

Annahme: lokale Firewall am Ziel-PC ist aus, IP + Hostname sind vergeben, Rechner ist eingeschaltet.

## 1. Direkt am Ziel-PC (Tastatur/Monitor)

```bash
ip a                          # Interface up? IP korrekt?
ip route                      # Default-Gateway gesetzt?
ping -c2 134.176.9.40         # Gateway erreichbar?
ping -c2 134.176.5.113        # ein Uni-Server (z.B. DNS)
cat /etc/resolv.conf          # DNS gesetzt?

systemctl status ssh          # läuft der Dienst?
ss -tlnp | grep :22           # lauscht er auf 22?
sudo journalctl -u ssh -n 50  # Fehler im SSH-Log?
```

```bash
sudo apt update                                                                                                                                                                                                                                                               
sudo apt install openssh-server         
sudo systemctl enable --now ssh                     
sudo systemctl status ssh        # sollte "active (running)" zeigen
ss -tlnp | grep :22    
```

Wenn Gateway nicht pingbar → Kabel / Switchport / VLAN / IP-Konfig falsch.
Wenn `ss` Port 22 nicht zeigt → `sudo apt install openssh-server && sudo systemctl enable --now ssh`.

## 2. Vom Client aus

```bash
ping <ip>                     # ICMP (Uni blockt das oft – nicht aussagekräftig!)
nslookup <hostname>           # löst DNS auf die richtige IP?
nmap -Pn -p22 <ip>            # open / filtered / closed ?
ssh -vvv user@<ip>            # verbose-Output gibt Hinweis
```

Interpretation `nmap`:
- **open** → SSH läuft, Problem liegt woanders (Login, Keys)
- **filtered** → Firewall (zentrale!) verwirft Pakete still
- **closed** → Paket kommt an, aber kein Dienst lauscht

## 3. Test innerhalb desselben Subnetzes

Wenn möglich, von einem anderen Rechner im **gleichen VLAN/Subnetz** testen
(z.B. Nachbarrechner im Büro). Wenn das geht, aber von außen nicht → zentrale
Firewall / VLAN-Trennung der Uni ist die Ursache.

## 4. Zentrale Firewall (wahrscheinlichste Ursache)

Im Uni-Netz ist eingehender SSH von außen meist per Default geblockt.
Beim **Rechenzentrum / Netzbetrieb** muss die IP für Port 22 freigeschaltet werden.

Anfrage typisch mit:
- Hostname + IP
- gewünschter Port (22/tcp)
- Quell-Netze (z.B. nur Uni-VPN, oder bestimmte Subnetze)
- Verantwortlicher + Begründung

Bis zur Freischaltung: per **Uni-VPN** einwählen, dann ist der Rechner meist
aus dem internen Netz heraus erreichbar.

## 5. Schnelle Differenzialdiagnose

| Symptom                            | wahrscheinliche Ursache                  |
|------------------------------------|------------------------------------------|
| Ping geht, SSH nicht               | sshd aus / nicht installiert / Port zu   |
| Ping nicht, SSH geht               | normal im Uni-Netz (ICMP geblockt)       |
| Beides nicht, intern aber ok       | zentrale Firewall / VLAN-Trennung        |
| Beides nicht, auch intern nicht    | IP/Routing/Gateway am Host falsch        |
| `nmap` zeigt filtered              | Firewall im Pfad                         |
| `nmap` zeigt closed                | Dienst lauscht nicht                     |
| DNS löst falsche IP auf            | DNS-Eintrag veraltet / falsch            |

## 6. Logs zum Mitschicken (falls Ticket nötig)

Vom Ziel-PC:
```bash
ip a; ip route; ss -tlnp | grep :22; systemctl status ssh
```
Vom Client:
```bash
nmap -Pn -p22 <ip>; ssh -vvv user@<ip> 2>&1 | head -40
```
