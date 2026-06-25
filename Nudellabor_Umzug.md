# Umzug: Serverprofis-Baukasten → selbst gehostet (Domain & Mail via Strato)

Die Nudellabor-App läuft als **Docker-Container** (Node/Express + SQLite, Login/Shop/Upload)
auf dem eigenen Server, lauscht auf `127.0.0.1:3020` und wird über **Caddy** (Reverse Proxy,
Auto-HTTPS) ausgeliefert.

## Wichtig vorab: Was kann Strato, was nicht

Ein **Webhosting-Minimalpaket** bei Strato (Webspace + PHP + MySQL) kann die App **nicht**
ausführen — sie braucht Node.js/Docker als laufenden Prozess (gibt es nur auf v-Server/
Root-Server). Die App bleibt also auf dem eigenen Server.

Ein kleines Strato-Paket dient nur für:

- **Domain** `nudellabor.com` verwalten (Umzug weg von Serverprofis)
- **DNS** verwalten (Zeiger auf den eigenen Server)
- **E-Mail-Postfächer** `@nudellabor.com` (liegen aktuell vermutlich bei Serverprofis und
  gehen sonst verloren!)

**Merksatz: Strato = Domain + DNS + Mail. Eigener Server = die eigentliche Website.**

---

## Schritte

### 1. Vorbereiten (noch nichts umstellen)
- App auf der Subdomain **komplett testen**: Shop, Login/Admin, Bild-Uploads,
  Bestell-/Kontakt-Mails.
- Öffentliche **Server-IP** notieren: `curl -4 ifconfig.me`.
  Falls nur dynamische IP: DynDNS oder feste IP klären.
- Prüfen, ob bei Serverprofis **E-Mail-Postfächer** auf `@nudellabor.com` liegen.
  Wenn ja → vor der Kündigung Inhalte sichern (IMAP-Export).

### 2. Strato-Paket buchen
- Kleinstes Paket mit **eigener Domain-Verwaltung + E-Mail** wählen.
- Domain `nudellabor.com` als **Umzug/Transfer** angeben (nicht neu registrieren).
- Den Webspace des Pakets einfach ungenutzt lassen.

### 3. Domain transferieren (.com → AuthCode nötig)
- Bei **Serverprofis**: Transfer-Sperre (Domain-Lock) aufheben und **AuthCode/EPP-Code**
  anfordern.
- Bei **Strato**: Transfer-Auftrag mit dem AuthCode starten. Dauer meist 1–7 Tage; die alte
  Seite bleibt währenddessen online.
- Tipp: Vorab bei Serverprofis die **DNS-TTL** auf `300` senken, dann greift die spätere
  Umstellung schneller.

### 4. DNS bei Strato setzen (der eigentliche „Umschalt"-Schritt)
- `A`-Record `@`   → **IP des eigenen Servers**
- `A`-Record `www` → gleiche IP (oder `CNAME` auf die Hauptdomain)
- `MX`-Records     → **Strato-Mailserver** (setzt Strato automatisch beim Anlegen von
  Postfächern) — so funktioniert E-Mail weiter
- Postfächer `@nudellabor.com` bei Strato anlegen.

### 5. HTTPS auf dem eigenen Server (Caddy)
Domain-Block in der Caddyfile ergänzen (analog `Caddyfile.snippet`):

```caddy
nudellabor.com, www.nudellabor.com {
    reverse_proxy 127.0.0.1:3020
}
```

Caddy neu laden → Let's-Encrypt-Zertifikat wird automatisch geholt.
(Klappt erst, sobald die DNS-A-Records auf den eigenen Server zeigen.)

### 6. Prüfen
- `https://nudellabor.com` und `https://www.nudellabor.com` erreichbar, Zertifikat gültig
- Alle Seiten, Bilder, Shop, Login, Test-Bestellung + Mailversand
- Test-Mail an `@nudellabor.com` (Postfach empfängt?)

### 7. Erst danach: Serverprofis kündigen
- Wenn Transfer **abgeschlossen** ist, Seite läuft und Mail funktioniert → Vertrag bei
  Serverprofis kündigen.
- Vorher Kündigungsfrist prüfen, damit kein Übergangsloch entsteht.

---

## Reihenfolge (gegen Ausfälle)

testen → Strato buchen → Transfer → DNS umstellen → Caddy/HTTPS → prüfen → **zuletzt** kündigen
