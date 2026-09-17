# Container-Vorlage: LimeSurvey

LimeSurvey mit PostgreSQL für das mittwald Container-Hosting — als Vorlage im
Format von [mittwald/container-templates](https://github.com/mittwald/container-templates)
und als eigenständige Compose-Datei für den direkten Deploy.

```
limesurvey/                 1:1 in einen Klon von mittwald/container-templates kopierbar
├── docker-compose.yml
├── manifest.yaml
└── icon.svg                aus dashboard-icons (Apache-2.0)
standalone/                 ohne Template-Mechanik, für "mw stack deploy"
├── docker-compose.yml
└── .env.example
```

Die Vorlage enthält **keine** Zugangsdaten. Das Datenbankpasswort erzeugt die
Plattform als `systemInput`, die Admin-Zugangsdaten gibt der Benutzer im
Installations-Assistenten ein.

## Aufbau

| Service | Image | Port | Volume |
|---|---|---|---|
| `limesurvey` | `martialblog/limesurvey:7.0.13-260903-apache` | 8080 → Domain | `limesurvey_upload` |
| `postgres` | `pgautoupgrade/pgautoupgrade:18-alpine` | — | `postgres_data` |

## Entscheidungen, die nicht offensichtlich sind

**Der Image-Tag ist exakt gepinnt, nicht auf den Major.** Die Konvention des
Repos wäre `7-apache`. Build `7.0.15-260910` liefert jedoch eine
`editor/index.html` aus, die Asset-Hashes aus einem älteren Build referenziert;
vier der acht Dateien fehlen im Image, der React-Editor bleibt weiß. Gemeldet
als [martialblog/docker-limesurvey#318](https://github.com/martialblog/docker-limesurvey/issues/318),
noch offen. Sobald ein korrigierter Build erscheint: prüfen und auf den
Major-Tag zurückgehen.

**`PUBLIC_URL` und `HOST_INFO` sind gesetzt.** Vor dem Container terminiert der
Ingress das TLS. Ohne diese beiden Variablen baut LimeSurvey interne Links aus
dem Hostnamen und Port des Containers — Redirects landen dann auf
`http://…:8080`. Beide müssen auf die Adresse zeigen, über die tatsächlich
zugegriffen wird; bei einem Domainwechsel mit anpassen.

**Das `upload`-Verzeichnis wird beim Start angelegt.** `/var/www/html/upload`
enthält Themes, Plugins und Teilnehmer-Uploads und liegt deshalb vollständig im
Volume. Ein neues Volume startet bei mittwald allerdings **leer** — anders als
bei lokalem Docker wird es nicht aus dem Image vorbefüllt, und weder Entrypoint
noch Anwendung legen die Struktur selbst an. Jeder Upload scheitert dann mit
„Could not save file". Die Vorlage legt die sieben Verzeichnisse deshalb per
`command` beim Containerstart an. Das funktioniert ohne Sonderrechte: Das Volume
gehört `www-data`, der Container darf in sein eigenes leeres Volume schreiben.

Derselbe Startbefehl schreibt eine `.htaccess` nach `surveys/`, die den direkten
Web-Zugriff auf Teilnehmer-Uploads (`fu_*`) verbietet — ohne sie liefert Apache
hochgeladene Dateien aus. Sie wird nur angelegt, wenn sie fehlt, eigene
Anpassungen überleben also einen Neustart.

Der Preis: `command` ersetzt das `CMD` des Images, die Vorlage ist damit auf
`apache2-foreground` festgelegt. Ändert der Upstream seinen Startbefehl, muss
das hier nachgezogen werden.

**Der Mailversand hängt an der Delivery-Box, aber nicht automatisch.**
LimeSurvey speichert seine Mail-Einstellungen in der Datenbank, das Image kennt
keine SMTP-Umgebungsvariablen. Die Vorlage deklariert deshalb eine Delivery-Box
und setzt deren Zugangsdaten als `SMTP_*`-Variablen am `limesurvey`-Service —
die Anwendung liest sie nicht, sie existieren nur, damit `help` sie nach der
Installation anzeigen kann. Eintragen muss der Benutzer sie in der Anwendung.
Darauf weist ein `alert` mit Status `warning` hin.

**`ADMIN_NAME` bekommt denselben Wert wie `ADMIN_USER`.** Ein eigenes
Eingabefeld für den Anzeigenamen wäre ein weiteres Pflichtfeld im Assistenten
für einen Wert, den man in der Anwendung in zehn Sekunden ändert.

**Kein `ports`-Eintrag am `postgres`-Service.** Innerhalb eines Stacks
erreichen sich die Container über ihren Service-Namen, ohne dass ein Port
deklariert werden muss. `ports` exponiert einen Dienst im gesamten Projekt —
für eine Datenbank, die nur ihre eigene Anwendung bedient, unnötig.

## Apache-Redirects: ServerName

Ohne expliziten `ServerName` setzt Apache selbstreferenzielle URLs aus seinem
eigenen Hostnamen und Port zusammen — den Ingress davor kennt er nicht.
Verzeichnispfade ohne abschließenden Schrägstrich landeten dadurch auf `http://`,
was der Browser als Mixed Content blockiert:

```
/editor  → 301 → http://<host>/editor/
```

Der Startbefehl schreibt deshalb `/etc/apache2/conf-enabled/servername.conf` mit
`ServerName https://${HOST}` und `UseCanonicalName On`. An einer laufenden
Instanz verifiziert: davor `http://`, danach `https://`.

Dass das ohne root funktioniert, liegt am Image — `/etc/apache2/conf-enabled`
gehört dort `www-data`. Bei einem Domainwechsel entsteht die Datei beim nächsten
Start automatisch neu, weil sie aus `${HOST}` gebaut wird.

## Vorlage einreichen

```bash
git clone https://github.com/mittwald/container-templates
cp -r limesurvey container-templates/
cd container-templates && pnpm install && pnpm validate
```

Der Validator prüft Pflichtdateien, Manifest-Schema und Screenshot-Regeln. Was
noch fehlt, sind die optionalen Katalogbilder: mindestens 1500 px breit, der
Hintergrund im Verhältnis exakt 3:2, erzeugbar mit `pnpm gen:background limesurvey`.
Die Screenshots müssen echte Oberflächen sein, keine Montagen.

## Direkter Deploy ohne Vorlage

```bash
cd standalone
cp .env.example .env    # ausfüllen
mw stack deploy -c docker-compose.yml --env-file .env
```

Anschließend die Domain im mStudio auf Port 8080 des `limesurvey`-Containers
zeigen lassen. Das Backend liegt unter `/index.php/admin`.

## Fallstricke im Betrieb

**Die `POSTGRES_*`-Variablen wirken nur beim allerersten Start.** Ist unter
`PGDATA` bereits ein Cluster vorhanden, ignoriert der Entrypoint sie
kommentarlos — auch nach einer Korrektur. Symptom ist
`FATAL: password authentication failed` mit dem Zusatz
`DETAIL: Role "limesurvey" does not exist`. Der einzige Weg zurück ist, das
Volume zu löschen oder Rolle und Datenbank von Hand anzulegen:

```bash
mw container exec postgres -- psql -U postgres -c "CREATE ROLE limesurvey LOGIN PASSWORD '<passwort>';"
mw container exec postgres -- psql -U postgres -c "CREATE DATABASE limesurvey OWNER limesurvey;"
```

**Nach jeder Reparatur an der Datenbank den App-Container neu starten.** Schema
und Admin-Benutzer legt der Entrypoint nur beim Start an. Ohne Neustart bleibt
die Datenbank leer.

**Vor einem PostgreSQL-Major-Upgrade einen Dump ziehen.** `pgautoupgrade` hebt
das Datenverzeichnis beim Start automatisch auf die Major-Version des Images.
Das erspart `pg_upgrade` von Hand, findet aber in-place statt: Bricht der
Container mitten im Upgrade ab, bleibt das Verzeichnis in einem Zwischenzustand.
Der erste Start nach einem Versionssprung dauert außerdem spürbar länger.
