# 🔐 Vaultwarden auf Unraid mit HTTPS ohne öffentliche Domain

# 🎯 Motivation

Seit neueren Versionen unterstützen die Browser-Erweiterungen von Vaultwarden bzw. Bitwarden keine unverschlüsselten HTTP-Verbindungen mehr zu selbst gehosteten Passwortservern.

Dadurch können zwar häufig noch Anmeldungen und Lesezugriffe funktionieren, Änderungen an gespeicherten Einträgen werden jedoch verweigert oder die Erweiterung blockiert die Verbindung vollständig.

Typische Symptome:

- ❌ Passwörter können nicht mehr gespeichert werden
- ❌ Einträge können nicht mehr bearbeitet werden
- ❌ Synchronisation schlägt fehl
- ❌ Browser-Erweiterung meldet eine unsichere Verbindung
- ❌ HTTP-URLs werden nicht mehr akzeptiert

Da der Vaultwarden-Server auf einem Unraid-Homeserver ausschließlich im privaten Heimnetz betrieben wird, existiert häufig:

- ❌ keine öffentliche Domain
- ❌ keine öffentliche IP-Adresse
- ❌ kein klassisches Let's-Encrypt-Zertifikat

Trotzdem verlangt die Browser-Erweiterung eine HTTPS-Verbindung.

Die hier beschriebene Lösung verwendet daher:

- 🔑 eine eigene lokale Zertifizierungsstelle (CA)
- 🔒 ein selbst erzeugtes TLS-Zertifikat
- 🏠 ausschließlich interne Kommunikation im Heimnetz

Dadurch erhält Vaultwarden eine vollwertige HTTPS-Verbindung, ohne dass der Server aus dem Internet erreichbar sein muss.

---

# 🎯 Ziel

Der Vaultwarden-Server läuft auf einem Unraid-Homeserver und soll ausschließlich innerhalb des Heimnetzes genutzt werden.

Da aktuelle Bitwarden- und Vaultwarden-Browser-Erweiterungen keine unverschlüsselten HTTP-Verbindungen mehr akzeptieren, wird HTTPS benötigt.

Diese Anleitung beschreibt die Einrichtung einer eigenen Zertifizierungsstelle (CA) mit **mkcert** und die Verwendung eines selbstsignierten Zertifikats für Vaultwarden.

---

# 🏗 Architektur

```text
┌───────────────┐
│ Browser Plugin│
└───────┬───────┘
        │ HTTPS
        ▼
┌─────────────────────┐
│ Unraid Homeserver   │
│ Vaultwarden Docker  │
│ TLS Zertifikat      │
└─────────────────────┘
```

Die Vertrauenskette sieht wie folgt aus:

```text
Eigene Root CA
      │
      ▼
Vaultwarden Zertifikat
      │
      ▼
Browser vertraut Verbindung
```

---

# 📋 Voraussetzungen

- ✅ Unraid Server
- ✅ Vaultwarden Docker Container
- ✅ Zugriff auf die Docker-Konfiguration
- ✅ Windows-PC zur Zertifikatserstellung
- ✅ Feste IP-Adresse für den Homeserver

Beispiel:

| Komponente | Wert |
|------------|------|
| Hostname | `vaultserver.lan` |
| IP-Adresse | `192.168.1.10` |
| Vaultwarden URL | `https://vaultserver.lan` |

---

# 🛠 Schritt 1: mkcert installieren

## Windows

### Chocolatey

```cmd
choco install mkcert
```

### Winget

```cmd
winget install FiloSottile.mkcert
```

---

# 🛠 Schritt 2: Lokale Root-CA installieren

```cmd
mkcert -install
```

Dabei wird:

- 🔑 eine lokale Root-CA erzeugt
- 🔒 die Root-CA in den Windows-Zertifikatsspeicher importiert

Der Browser vertraut anschließend allen Zertifikaten dieser CA.

---

# 🛠 Schritt 3: Zertifikat erzeugen

Beispiel:

```cmd
mkcert vaultserver.lan 192.168.1.10
```

Es entstehen zwei Dateien:

```text
vaultserver.lan+1.pem
vaultserver.lan+1-key.pem
```

---

# 🛠 Schritt 4: Zertifikate nach Unraid kopieren

Empfohlenes Verzeichnis:

```text
/mnt/user/appdata/vaultwarden/ssl/
```

Beispiel:

```text
/mnt/user/appdata/vaultwarden/ssl/
├── vaultserver.lan.pem
└── vaultserver.lan-key.pem
```

---

# 🛠 Schritt 5: Vaultwarden konfigurieren

In den Docker-Variablen folgende Umgebungsvariable ergänzen:

```text
ROCKET_TLS={certs="/data/ssl/vaultserver.lan.pem",key="/data/ssl/vaultserver.lan-key.pem"}
```

Zusätzlich sicherstellen:

```text
WEB_VAULT_ENABLED=true
```

---

# 🛠 Schritt 6: Docker-Pfad-Mapping prüfen

Im Container muss das Verzeichnis eingebunden werden.

Beispiel:

| Host |
|------|
| `/mnt/user/appdata/vaultwarden/ssl` |

↓

| Container |
|------------|
| `/data/ssl` |

---

# 🛠 Schritt 7: Container neu starten

```text
Docker → Vaultwarden → Restart
```

oder

```bash
docker restart vaultwarden
```

---

# 🛠 Schritt 8: FritzBox-Namensauflösung einrichten

Dem Unraid-Server in der FritzBox einen festen Namen geben.

Beispiel:

```text
vaultserver.lan
```

oder

```text
vaultserver.fritz.box
```

Prüfen:

```cmd
ping vaultserver.lan
```

Sollte die korrekte IP-Adresse liefern.

---

# 🛠 Schritt 9: HTTPS testen

Im Browser öffnen:

```text
https://vaultserver.lan
```

Erwartetes Ergebnis:

- ✅ keine Zertifikatswarnung
- ✅ Login-Seite erscheint

---

# 🛠 Schritt 10: Browser-Erweiterung umstellen

In der Bitwarden/Vaultwarden-Erweiterung:

⚙️ Einstellungen → Selbst gehostet

Server URL:

```text
https://vaultserver.lan
```

Anschließend neu anmelden.

---

# 📱 Weitere Geräte einrichten

Damit Smartphones oder Tablets dem Zertifikat vertrauen, muss die Root-CA ebenfalls importiert werden.

Root-CA Speicherort:

```cmd
mkcert -CAROOT
```

Dort befindet sich:

```text
rootCA.pem
```

Diese Datei auf die jeweiligen Geräte übertragen und als vertrauenswürdige Zertifizierungsstelle importieren.

---

# 🔍 Fehlersuche

## Zertifikatswarnung im Browser

Prüfen:

- Root-CA installiert?
- Richtiger Hostname verwendet?
- Zertifikat enthält die verwendete IP-Adresse?

---

## Vaultwarden startet nicht

Logs prüfen:

```bash
docker logs vaultwarden
```

Typische Ursache:

```text
ROCKET_TLS Pfad falsch
```

---

## Browser-Erweiterung verbindet sich nicht

Prüfen:

```text
https://vaultserver.lan
```

muss bereits im Browser funktionieren.

Erst danach funktioniert die Erweiterung.

---

# ✅ Ergebnis

Nach erfolgreicher Einrichtung:

- 🔒 HTTPS für Vaultwarden
- 🔑 Browser-Erweiterung funktioniert wieder
- 🏠 Keine öffentliche Domain erforderlich
- 🌐 Keine Portfreigaben notwendig
- 🛡 Vollständig innerhalb des Heimnetzes nutzbar

---

# 🚀 Spätere Erweiterungen

Falls künftig weitere Dienste abgesichert werden sollen:

- 🌐 Traefik Reverse Proxy
- 🌐 Caddy Reverse Proxy
- 🌐 SWAG (LinuxServer.io)
- 🌐 Eigene interne PKI

Damit können auch Dienste wie:

- Home Assistant
- Nextcloud
- Jellyfin
- Immich
- Paperless-ngx
- IPFS-WebUI

unter einer gemeinsamen HTTPS-Infrastruktur betrieben werden.
