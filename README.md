# Systemanforderungen — Lagerverwaltung

Annahme: **Betrieb auf einem Linux-Server ohne Root-Rechte oder Virtuellen Server**

## 1. Hardware (Mindest- / Empfehlung)

| Ressource | Minimum | Empfohlen |
|-----------|---------|-----------|
| CPU       | 2 Kern  | 4 Kerne   |
| RAM       | 4 GB    | 8 GB      |
| Festplatte| 50 GB   | 100 GB SSD |
| Netzwerk  | 100 MBit| 1 GBit, statische IP |

Speicherbedarf wächst hauptsächlich durch Rechnungs-PDFs und Postgres-WAL/Backups.

## 2. Container-Laufzeit: Rootless Docker oder Podman

Ohne Root-Rechte ist klassisches Docker nicht nutzbar (der Daemon läuft als root).
Es gibt zwei taugliche Optionen — eine davon muss der Administrator bereitstellen:

### Option A — Rootless Docker (empfohlen, weil `docker compose` direkt funktioniert)

Der Administrator richtet **einmalig als Root** ein:

1. Pakete installieren: `docker-ce-rootless-extras`, `uidmap`, `slirp4netns`, `fuse-overlayfs`
2. Subordinate-UIDs/GIDs für die Betriebs-Kennung anlegen, z.B.
   ```
   /etc/subuid:  appuser:100000:65536
   /etc/subgid:  appuser:100000:65536
   ```
3. `loginctl enable-linger appuser` — damit der User-Service-Manager
   nach Reboot ohne Login weiterläuft (Voraussetzung für Auto-Start der Container).
4. Kernel-Parameter freigeben (Distri-abhängig): `kernel.unprivileged_userns_clone=1`,
   ggf. `net.ipv4.ip_unprivileged_port_start=80` falls Binden auf Port 80/443
   ohne Root erlaubt sein soll (sonst nur High-Ports).

Danach durch den Betriebs-User selbst:
```bash
dockerd-rootless-setuptool.sh install
systemctl --user enable --now docker
docker compose up -d
```

### Option B — Podman (wenn Docker nicht bereitgestellt wird)

Podman ist auf Rocky/RHEL 9 und Ubuntu 24.04 standardmäßig rootless-tauglich.
`docker-compose.yml` läuft mit `podman compose` oder `podman-compose`,
gelegentlich mit Anpassungen (Volume-Pfade, Netzwerk-Aliase).

Was der Administrator trotzdem **einmalig als Root** tun muss:
- Paket `podman` (und `podman-compose` oder `podman-docker`) installieren
- Subordinate-UIDs/GIDs eintragen (wie oben)
- `loginctl enable-linger appuser`

## 3. Was der Administrator außerdem **vorher** bereitstellen muss

1. **Persistente Pfade** mit Schreibrechten für den Betriebs-User
   (z.B. `/var/lib/lager/` oder `~/lager-data/` — Volumes für Postgres + Rechnungs-PDFs).
2. **Eintrag in einem vorgeschalteten Reverse-Proxy** (sofern vorhanden):
   `lager.example.org` → `server-intern:8443` (oder beliebiger High-Port).
   Ohne Root kann die App nicht selbst auf 80/443 lauschen — TLS und externer
   DNS gehören in den vorgeschalteten Proxy.
3. **Firewall-Freischaltungen** für die benötigten Ports und ausgehenden
   Verbindungen (SMTP, NTP, ggf. externe Sync-Schnittstellen).
4. **Backup**: System-Backup auf das Datenverzeichnis sowie ein nächtlicher
   `pg_dump`-Cron als User-Cron.
5. **SMTP-Relay** für den Mailversand (Hostname und Port).

## 4. Reverse-Proxy-Optionen ohne Root

| Variante | Geeignet? | Hinweis |
|----------|-----------|---------|
| **Vorgeschalteter Reverse-Proxy auf Infrastrukturebene** | ✅ erste Wahl | TLS-Terminierung, DNS, ggf. SSO inklusive |
| **Caddy** als Container im Stack | ✅ | Auto-HTTPS via DNS-Challenge möglich; auch nur HTTP nach innen |
| **Traefik** als Container im Stack | ✅ | Konfiguration via Labels in `docker-compose.yml` |
| **nginx**/**HAProxy** systemweit installiert | ❌ ohne Root | nur falls der Administrator es bereitstellt |

**Empfehlung:** Wenn ein Infrastruktur-Reverse-Proxy verfügbar ist, diesen
nutzen — kein eigener TLS-Container nötig. Der mitgelieferte `frontend`-Container
(nginx auf Port 80) reicht als interner Endpunkt und wird per Port-Mapping
auf einen High-Port (z.B. 8443) gelegt, auf den der Proxy zeigt.

Steht kein vorgeschalteter Proxy zur Verfügung, übernimmt **Caddy** im
Container-Stack die TLS-Terminierung. Auf Port 80/443 lauschen ist ohne
Root-Rechte nur möglich, wenn der Administrator
`net.ipv4.ip_unprivileged_port_start=80` gesetzt hat — andernfalls High-Ports
(z.B. 8080/8443) verwenden.

## 5. Software (zusätzlich auf dem Server)

- **Linux LTS:** Ubuntu 24.04, Debian 12 oder Rocky 9 — alle drei unterstützen Rootless
- **Git** (für Updates via `git pull`)
- **NTP/chrony** läuft (Token-Ablauf, PDF-Zeitstempel)

## 6. Was die App selbst benötigt

- Erreichbarer SMTP-Server für Mailversand (`SMTP_HOST`/`SMTP_PORT` in `docker-compose.yml`)
- Schreibbares Volume für `rechnungen/` (PDF-Ablage)
- Ausreichend Platz für `pgdata`-Volume

## 7. Update-Prozess

```bash
git pull
docker compose pull
docker compose up -d --build
```

Kein Datenverlust, da `pgdata` und `rechnungen` als Volumes persistiert sind.
Vor größeren Updates: `pg_dump` ziehen.

## 8. Bekannte Einschränkungen im rootless Betrieb

- Ports < 1024 nicht bindbar (deshalb High-Port + vorgeschalteter Proxy).
- Performance-Overhead durch `slirp4netns` (~10–20 % Netzwerkdurchsatz).
  Für diese App irrelevant.
- Manche Images, die `chown` auf systemweite UIDs erwarten, können stocken —
  Postgres- und nginx-Images aus diesem Stack sind getestet.

## 9. Einbettung als iframe in eine bestehende Webseite

Soll das Bestellportal als iframe in eine vorhandene Webseite eingebunden werden
(z.B. unter dem Menüpunkt "Online Katalog"), sind sowohl auf der einbettenden
Seite als auch im Lagerverwaltungs-Stack Anpassungen nötig.

### Auf der einbettenden Webseite

1. **iframe-Element** an gewünschter Stelle:
   ```html
   <iframe
     src="https://lager.example.org/bestellportal"
     title="Online Katalog"
     loading="lazy"
     style="width:100%; height:80vh; border:0;">
   </iframe>
   ```
2. **Content-Security-Policy** der einbettenden Seite muss die Lager-Domain
   in `frame-src` (oder `child-src`) erlauben:
   ```
   Content-Security-Policy: frame-src https://lager.example.org;
   ```
3. **Gleiches Schema (HTTPS) auf beiden Seiten** — sonst blockt der Browser wegen
   Mixed Content. Wenn die Hostseite über HTTPS läuft, muss das Portal ebenfalls
   per HTTPS erreichbar sein.
4. **Optional `sandbox`-Attribut** falls strikte Isolation gewünscht ist —
   muss mindestens `allow-scripts allow-same-origin allow-forms allow-popups`
   enthalten, sonst funktionieren Login und Bestellvorgang nicht.
5. **Referrer-Policy** auf `strict-origin-when-cross-origin` (Standard reicht meist).

### Auf der Lagerverwaltungs-Seite (frontend nginx)

1. **`X-Frame-Options` entfernen oder auf `SAMEORIGIN` belassen ist nicht
   ausreichend** — für eine fremde Host-Domain muss die alte
   `X-Frame-Options`-Header weg und stattdessen via CSP gesteuert werden.
2. **CSP `frame-ancestors`** in `frontend/nginx.conf` setzen:
   ```nginx
   add_header Content-Security-Policy
     "frame-ancestors 'self' https://www.example.org https://*.example.org";
   ```
   Nur die einbettenden Domains aufzählen — kein `*`, sonst Clickjacking-Risiko.
3. **CORS** ist für reine iframe-Einbettung **nicht** nötig (das ist nur ein
   Browser-Frame, kein Cross-Origin-Fetch). CORS wird erst relevant, wenn die
   Hostseite per `fetch()` Daten aus der API zieht.
4. **Cookies / Session:** Diese App hält die Portal-Session in `localStorage`,
   nicht in Cookies — damit greifen Cross-Site-Cookie-Restriktionen
   (`SameSite=None; Secure`, Third-Party-Cookie-Blocker) **nicht**. Sollten
   später Cookies eingeführt werden, müssen sie mit `SameSite=None; Secure`
   gesetzt werden, sonst geht die Session im iframe verloren.
5. **Auto-Login / Magic-Link-Verhalten:** Der Magic-Link öffnet aktuell
   `/bestellportal/verify/<token>` in dem Tab, in dem der Mail-Client den Link
   öffnet — also typischerweise als Top-Level-Tab, **nicht** im iframe.
   Das ist gewollt (sonst würde der Verify-Link in der Mail-App nicht
   funktionieren). Nach erfolgreicher Anmeldung kann der Nutzer in den
   iframe-Tab zurückwechseln; die Session liegt im `localStorage` der
   Portal-Domain und ist im iframe sofort verfügbar.
6. **Höhe/Resize:** iframes haben eine feste Höhe. Bei langen Listen entsteht
   ein innerer Scrollbalken. Optional `postMessage`-basiertes Resize einbauen
   (Portal sendet `{height: …}`, Hostseite passt iframe an) — aktuell nicht
   implementiert.

### Reverse-Proxy-Konfiguration

Der vorgeschaltete Proxy (Infrastruktur oder Caddy/Traefik) muss die
CSP-Header **durchreichen** und nicht überschreiben. Bei nginx davor:
```nginx
proxy_pass_header Content-Security-Policy;
```
Bei Caddy ist das Default-Verhalten.

### Test-Checkliste

- [ ] iframe lädt ohne `Refused to display ... in a frame because it set X-Frame-Options to ...` in der Browser-Konsole
- [ ] Login funktioniert, JWT bleibt nach Reload erhalten
- [ ] Bestellvorgang inkl. Bestätigungs-Mail-Link funktioniert
- [ ] Keine Mixed-Content-Warnungen (alles HTTPS)
- [ ] CSP-Header der Hostseite blockt Portal nicht (in DevTools → Network → Response Headers prüfen)
