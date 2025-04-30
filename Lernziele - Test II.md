# Docker und Docker Compose: Umfassender Leitfaden

## Inhaltsverzeichnis
1. [Einführung](#einführung)
2. [LZ1: Syntax von Dockerfiles](#lz1-syntax-von-dockerfiles)
3. [LZ2 & LZ3: Dockerfiles nachvollziehen und erstellen](#lz2--lz3-dockerfiles-nachvollziehen-und-erstellen)
4. [LZ4: Einsatzzweck von Docker Compose](#lz4-einsatzzweck-von-docker-compose)
5. [LZ5: Aufbau einer YAML-Datei](#lz5-aufbau-einer-yaml-datei)
6. [LZ6: Docker Compose anwenden](#lz6-docker-compose-anwenden)
7. [LZ7: Umgang mit Secrets in Docker Compose](#lz7-umgang-mit-secrets-in-docker-compose)
8. [Best Practices und erweiterte Konzepte](#best-practices-und-erweiterte-konzepte)

## Einführung

Docker ist eine Plattform zur Entwicklung, Bereitstellung und Ausführung von Anwendungen in isolierten Umgebungen, genannt Containern. Docker Compose erweitert diese Funktionalität, indem es ermöglicht, Multi-Container-Anwendungen zu definieren und zu verwalten. Dieser Leitfaden bietet einen umfassenden Überblick über die wichtigsten Konzepte und Techniken.

## LZ1: Syntax von Dockerfiles

Ein Dockerfile ist eine Textdatei mit einer Reihe von Anweisungen, die beschreiben, wie ein Docker-Image erstellt wird. Diese Anweisungen werden in einer bestimmten Reihenfolge ausgeführt, und jede Anweisung erzeugt einen neuen Layer im Image.

### Grundlegende Dockerfile-Anweisungen

| Anweisung | Beschreibung | Beispiel |
|-----------|-------------|----------|
| `FROM` | Gibt das Basis-Image an | `FROM ubuntu:22.04` |
| `LABEL` | Metadaten zum Image hinzufügen | `LABEL maintainer="name@example.com"` |
| `RUN` | Führt Befehle während des Builds aus | `RUN apt-get update && apt-get install -y nginx` |
| `COPY` | Kopiert Dateien vom Host ins Image | `COPY app/ /app/` |
| `ADD` | Kopiert Dateien und kann Archive entpacken | `ADD archive.tar.gz /app/` |
| `ENV` | Setzt Umgebungsvariablen | `ENV NODE_ENV=production` |
| `WORKDIR` | Setzt das Arbeitsverzeichnis | `WORKDIR /app` |
| `EXPOSE` | Deklariert Container-Ports | `EXPOSE 80 443` |
| `VOLUME` | Definiert ein Volume | `VOLUME /data` |
| `CMD` | Standardbefehl beim Containerstart | `CMD ["nginx", "-g", "daemon off;"]` |
| `ENTRYPOINT` | Hauptprozess des Containers | `ENTRYPOINT ["php", "-S", "0.0.0.0:8000"]` |

### Wichtige Konzepte

#### Unterschied zwischen ADD und COPY
- `COPY` ist einfacher und transparenter, kopiert nur lokale Dateien
- `ADD` hat zusätzliche Funktionen wie Entpacken von Archiven und URL-Downloads
- Best Practice: Verwende `COPY` für einfache Dateioperationen, `ADD` nur wenn die Zusatzfunktionen benötigt werden

#### Unterschied zwischen CMD und ENTRYPOINT
- `CMD` definiert Standardbefehle und -argumente, kann beim `docker run` überschrieben werden
- `ENTRYPOINT` definiert den Hauptprozess, der immer ausgeführt wird; Parameter beim `docker run` werden als Argumente an den ENTRYPOINT-Befehl angehängt

```dockerfile
# Mit CMD
FROM ubuntu:22.04
CMD ["echo", "Hello World"]

# Ausführung: docker run myimage              # Ausgabe: Hello World
# Ausführung: docker run myimage echo Goodbye # Ausgabe: Goodbye (überschreibt CMD)
```

```dockerfile
# Mit ENTRYPOINT
FROM ubuntu:22.04
ENTRYPOINT ["echo", "Hello"]

# Ausführung: docker run myimage         # Ausgabe: Hello
# Ausführung: docker run myimage World   # Ausgabe: Hello World (ergänzt ENTRYPOINT)
```

#### RUN-Befehle verketten
Es ist eine Best Practice, mehrere RUN-Befehle mit `&&` zu verbinden, um die Anzahl der Layers zu reduzieren:

```dockerfile
# Nicht empfohlen: Jeder RUN-Befehl erzeugt einen neuen Layer
RUN apt-get update
RUN apt-get install -y nginx
RUN apt-get clean

# Empfohlen: Alle Befehle in einem einzigen RUN-Statement
RUN apt-get update && \
    apt-get install -y nginx && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

## LZ2 & LZ3: Dockerfiles nachvollziehen und erstellen

### Beispiel-Dockerfile für einen Apache Webserver

```dockerfile
# Basis-Image: Ubuntu in der neuesten Version
FROM ubuntu:latest

# Metadaten zum Image hinzufügen
LABEL maintainer="name@example.com"
LABEL description="Apache Webserver für Entwicklungszwecke"
LABEL version="1.0"

# Umgebungsvariablen setzen
ENV TZ=Europe/Zurich \
    APACHE_RUN_USER=www-data \
    APACHE_RUN_GROUP=www-data \
    APACHE_LOG_DIR=/var/log/apache2

# Zeitzone einstellen und Apache installieren
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && \
    echo $TZ > /etc/timezone && \
    apt-get update && \
    apt-get install -y apache2 && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* && \
    a2enmod rewrite && \
    a2enmod ssl && \
    a2ensite default-ssl

# Standardports für HTTP und HTTPS freigeben
EXPOSE 80 443

# Volume für die Webinhalte definieren
VOLUME /var/www/html

# Apache im Vordergrund starten
CMD ["apache2ctl", "-D", "FOREGROUND"]
```

### Erläuterung der Befehle

- `FROM ubuntu:latest`: Verwendet das offizielle Ubuntu-Image als Basis
- `LABEL`: Fügt Metadaten zum Image hinzu
- `ENV`: Setzt Umgebungsvariablen für Zeitzone und Apache-Konfiguration
- `RUN`: 
  - Konfiguriert die Zeitzone
  - Aktualisiert die Paketlisten
  - Installiert Apache
  - Aktiviert benötigte Module und Konfigurationen
  - Räumt den Paket-Cache auf, um das Image klein zu halten
- `EXPOSE`: Deklariert, dass der Container auf Ports 80 und 443 lauscht
- `VOLUME`: Definiert ein Volume für die Webinhalte
- `CMD`: Startet Apache im Vordergrund (wichtig für Container)

### Schritt-für-Schritt-Anleitung zum Erstellen und Testen eines Dockerfiles

1. **Dockerfile erstellen**:
   ```bash
   mkdir apache-docker
   cd apache-docker
   nano Dockerfile   # Füge den obigen Dockerfile-Inhalt ein
   ```

2. **Build-Kontext vorbereiten**:
   ```bash
   # Erstelle eine einfache HTML-Datei
   mkdir -p html
   echo "<h1>Hello Docker!</h1>" > html/index.html
   ```

3. **Image bauen**:
   ```bash
   docker build -t my-apache:v1.0 .
   ```

4. **Image überprüfen**:
   ```bash
   docker images | grep my-apache
   ```

5. **Container starten**:
   ```bash
   docker run -d -p 8080:80 --name apache-test -v $(pwd)/html:/var/www/html my-apache:v1.0
   ```

6. **Test der Funktionalität**:
   - Öffne einen Browser und navigiere zu `http://localhost:8080`
   - Du solltest die Testseite "Hello Docker!" sehen

7. **Container-Logs überprüfen**:
   ```bash
   docker logs apache-test
   ```

8. **Container-Shell öffnen** (für Debugging):
   ```bash
   docker exec -it apache-test bash
   ```

9. **Aufräumen nach dem Test**:
   ```bash
   docker stop apache-test
   docker rm apache-test
   ```

### Dokumentations-Template für Dockerfiles

```markdown
# Apache Webserver Docker Image

## Beschreibung
Dieses Docker-Image enthält einen Apache Webserver auf Ubuntu-Basis für Entwicklungs- und Testzwecke.

## Funktionen
- Apache 2.4
- SSL-Unterstützung
- URL-Rewriting aktiviert
- Zeitzone auf Europe/Zurich eingestellt

## Verwendung

### Bauen des Images
```bash
docker build -t my-apache:v1.0 .
```

### Starten eines Containers
```bash
docker run -d -p 8080:80 -v /pfad/zu/webinhalten:/var/www/html my-apache:v1.0
```

### Ports
- 80: HTTP
- 443: HTTPS

### Volumes
- `/var/www/html`: Webinhalte

## Konfiguration
Konfigurationsdateien befinden sich in `/etc/apache2/`

## LZ4: Einsatzzweck von Docker Compose

Docker Compose ist ein Tool zur Definition und Ausführung von Multi-Container-Docker-Anwendungen. Mit Docker Compose verwendest du eine YAML-Datei, um die Dienste deiner Anwendung zu konfigurieren und dann alles mit einem einzigen Befehl zu starten.

### Haupteinsatzzwecke

1. **Entwicklungsumgebungen**
   - Schnelles Aufsetzen komplexer lokaler Entwicklungsumgebungen
   - Vermeidung von "Es funktioniert auf meinem Rechner"-Problemen

2. **Automatisierte Tests**
   - Konsistente Testumgebungen für CI/CD-Pipelines
   - Einfache Integration in Testworkflows

3. **Single-Host-Deployments**
   - Einfache Produktionsumgebungen auf einem Host
   - Geeignet für kleine bis mittlere Anwendungen

4. **Service-Orchestrierung**
   - Definition von Abhängigkeiten zwischen Diensten
   - Konfiguration von Netzwerken und Volumes
   - Skalierung von Diensten

5. **Infrastruktur als Code**
   - Versionierbare Anwendungsdefinitionen
   - Reproduzierbare Umgebungen

### Vorteile gegenüber manueller Container-Verwaltung

- **Einfachheit**: Ein Befehl zum Starten aller Dienste
- **Konsistenz**: Gleiche Umgebung für alle Entwickler
- **Wiederverwendbarkeit**: Konfigurationen können in verschiedenen Projekten wiederverwendet werden
- **Isolation**: Jedes Projekt kann seine eigenen Abhängigkeiten haben, ohne Konflikte
- **Dokumentation**: Die Compose-Datei dokumentiert die Abhängigkeiten und Konfiguration

## LZ5: Aufbau einer YAML-Datei

YAML (YAML Ain't Markup Language) ist ein menschenlesbares Datenformat, das häufig für Konfigurationsdateien verwendet wird.

### Grundlegende YAML-Syntax

```yaml
# Dies ist ein Kommentar

# Skalare (einfache Werte)
string: Dies ist ein String
quoted_string: "Dies ist ein zitierter String"
number: 42
float: 3.14159
boolean: true
null_value: null

# Listen
einfache_liste:
  - Eintrag 1
  - Eintrag 2
  - Eintrag 3

# Listen können auch einzeilig geschrieben werden
einzeilige_liste: [Eintrag 1, Eintrag 2, Eintrag 3]

# Verschachtelte Listen
verschachtelte_liste:
  - 
    - Element 1.1
    - Element 1.2
  - 
    - Element 2.1
    - Element 2.2

# Dictionaries (Key-Value-Paare)
person:
  name: Max Mustermann
  alter: 30
  beruf: Entwickler
  hobbys:
    - Programmieren
    - Lesen
    - Wandern

# Dictionary einzeilig
einzeiliges_dictionary: {name: Max, alter: 30}

# Mehrzeilige Strings
beschreibung: |
  Dies ist ein mehrzeiliger String.
  Zeilenumbrüche werden beibehalten.
  Einrückung wird entfernt.

# Mehrzeilige Strings ohne Zeilenumbrüche
zusammengefasst: >
  Dies ist ein mehrzeiliger String.
  Zeilenumbrüche werden durch Leerzeichen ersetzt.
  Nützlich für lange Textzeilen.

# Ankerverweise (wiederverwendbare Elemente)
basis: &basis
  version: 1.0
  sprache: Deutsch

erweiterung:
  <<: *basis  # Übernimmt alle Eigenschaften von 'basis'
  autor: Max Mustermann
```

### Wichtige YAML-Regeln

1. **Einrückung**:
   - Verwendet Leerzeichen (keine Tabs)
   - Konsistente Einrückung (meist 2 oder 4 Leerzeichen)

2. **Doppelpunkte**:
   - Nach einem Schlüssel folgt ein Doppelpunkt und ein Leerzeichen: `schlüssel: wert`

3. **Listen**:
   - Einträge beginnen mit einem Bindestrich und Leerzeichen: `- eintrag`

4. **Anführungszeichen**:
   - Strings müssen in Anführungszeichen gesetzt werden, wenn sie Sonderzeichen enthalten
   - `"doppelte"` oder `'einfache'` Anführungszeichen sind möglich

5. **Escape-Sequenzen**:
   - `\n` für Zeilenumbruch, `\t` für Tab, `\\` für Backslash

6. **Boolesche Werte**:
   - `true`, `false`, `yes`, `no`, `on`, `off` werden als boolesche Werte erkannt

## LZ6: Docker Compose anwenden

### Beispiel: Webserver mit Datenbank

Wir erstellen eine einfache Webanwendung mit Nginx als Webserver und MySQL als Datenbank.

#### Projektstruktur

```
webserver-projekt/
├── docker-compose.yaml
├── nginx/
│   ├── default.conf
│   └── Dockerfile
└── html/
    └── index.html
```

#### 1. HTML-Datei erstellen

```html
<!-- html/index.html -->
<!DOCTYPE html>
<html>
<head>
    <title>Docker Compose Demo</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 40px;
            line-height: 1.6;
        }
        h1 {
            color: #3498db;
        }
    </style>
</head>
<body>
    <h1>Willkommen zur Docker Compose Demo!</h1>
    <p>Diese Seite wird von einem Nginx-Container bereitgestellt.</p>
    <p>Die Daten werden in einer MySQL-Datenbank gespeichert.</p>
</body>
</html>
```

#### 2. Nginx-Konfiguration erstellen

```nginx
# nginx/default.conf
server {
    listen 80;
    server_name localhost;
    
    root /usr/share/nginx/html;
    index index.html;
    
    location / {
        try_files $uri $uri/ =404;
    }
}
```

#### 3. Nginx Dockerfile erstellen

```dockerfile
# nginx/Dockerfile
FROM nginx:alpine

COPY default.conf /etc/nginx/conf.d/default.conf
```

#### 4. Docker Compose-Datei erstellen

```yaml
# docker-compose.yaml
version: '3.8'

services:
  # Webserver-Service
  webserver:
    build: ./nginx
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html
    depends_on:
      - database
    networks:
      - frontend
      - backend
    restart: unless-stopped

  # Datenbank-Service
  database:
    image: mysql:8.0
    volumes:
      - db_data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MYSQL_DATABASE: webappdb
      MYSQL_USER: webapp
      MYSQL_PASSWORD: ${DB_USER_PASSWORD}
    networks:
      - backend
    restart: unless-stopped

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge

volumes:
  db_data:
```

#### 5. Umgebungsvariablen-Datei erstellen

```
# .env (Nicht ins Git-Repository einchecken!)
DB_ROOT_PASSWORD=sicheres_root_passwort
DB_USER_PASSWORD=sicheres_benutzer_passwort
```

#### 6. Docker Compose Befehle

```bash
# Starten der Container
docker compose up -d

# Status anzeigen
docker compose ps

# Logs anzeigen
docker compose logs

# Logs eines bestimmten Services anzeigen
docker compose logs webserver

# Dienste stoppen
docker compose stop

# Dienste starten
docker compose start

# Dienste stoppen und Container entfernen
docker compose down

# Dienste stoppen, Container und Volumes entfernen
docker compose down -v

# Dienste neu erstellen
docker compose up -d --build
```

### Erklärung der Compose-Datei

- **version**: Gibt die Version der Compose-Datei an (3.8 ist die aktuelle stabile Version)
- **services**: Definiert die verschiedenen Container-Dienste
  - **webserver**: Nginx-Container
    - **build**: Baut ein Image aus dem angegebenen Verzeichnis
    - **ports**: Mappt den Host-Port 8080 auf den Container-Port 80
    - **volumes**: Bindet den lokalen html-Ordner in den Container ein
    - **depends_on**: Stellt sicher, dass der Datenbank-Container zuerst gestartet wird
    - **networks**: Verbindet den Container mit den definierten Netzwerken
    - **restart**: Definiert das Restart-Verhalten
  - **database**: MySQL-Container
    - **image**: Verwendet ein fertiges MySQL-Image
    - **volumes**: Persistenter Speicher für Datenbankdaten
    - **environment**: Umgebungsvariablen zur Datenbankkonfiguration
    - **networks**: Verbindet den Container mit dem Backend-Netzwerk
- **networks**: Definiert isolierte Netzwerke für die Dienste
- **volumes**: Definiert persistenten Speicher

## LZ7: Umgang mit Secrets in Docker Compose

Die sichere Verwaltung von Passwörtern, API-Schlüsseln und anderen sensiblen Informationen ist ein wichtiger Aspekt bei der Arbeit mit Docker Compose, besonders wenn das Projekt auf GitHub gehostet wird.

### Methoden zur Verwaltung von Secrets

#### 1. Umgebungsvariablen mit .env-Dateien

Die einfachste Methode, die auch von Docker Compose nativ unterstützt wird:

```yaml
# docker-compose.yaml
services:
  database:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MYSQL_DATABASE: ${DB_NAME}
      MYSQL_USER: ${DB_USER}
      MYSQL_PASSWORD: ${DB_PASSWORD}
```

```
# .env (NICHT ins Git-Repository einchecken!)
DB_ROOT_PASSWORD=sicheres_root_passwort
DB_NAME=meine_datenbank
DB_USER=db_benutzer
DB_PASSWORD=sicheres_benutzer_passwort
```

**Wichtig**: Füge die .env-Datei zu deiner .gitignore hinzu:

```
# .gitignore
.env
```

**Vorteile**:
- Einfache Implementierung
- Native Unterstützung durch Docker Compose
- Konfiguration ist klar von Anwendungscode getrennt

**Nachteile**:
- Gefahr des versehentlichen Commits
- Kein verschlüsselter Speicher
- Variablen könnten in Container-Logs auftauchen

#### 2. Docker Secrets (Docker Compose ab Version 3.1)

Docker Secrets bietet eine sichere Möglichkeit, sensible Daten zu verwalten:

```yaml
version: '3.8'
services:
  database:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: webappdb
      MYSQL_USER: webapp
      # Verweis auf Secrets über Umgebungsvariablen
      # Wir müssen _FILE an den Variablennamen anhängen,
      # damit MySQL die Datei statt der Umgebungsvariable liest
      MYSQL_ROOT_PASSWORD_FILE: /run/secrets/db_root_password
      MYSQL_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_root_password
      - db_password

# Secrets-Definition
secrets:
  db_root_password:
    file: ./secrets/db_root_password.txt
  db_password:
    file: ./secrets/db_password.txt
```

Erstelle die Secret-Dateien:

```bash
mkdir -p secrets
echo "sicheres_root_passwort" > secrets/db_root_password.txt
echo "sicheres_benutzer_passwort" > secrets/db_password.txt
```

**Wichtig**: Füge das secrets-Verzeichnis zu .gitignore hinzu:

```
# .gitignore
secrets/
```

**Vorteile**:
- Besseres Sicherheitsmodell als Umgebungsvariablen
- Dateien statt Umgebungsvariablen (weniger Risiko des Leaks in Logs)
- Klare Trennung von Konfiguration und Secrets

**Nachteile**:
- Nicht alle Images unterstützen Secrets über Dateien
- Funktioniert ohne Docker Swarm nur mit File-basierten Secrets
- Lokale Secret-Dateien müssen trotzdem geschützt werden

#### 3. Externe Secret-Management-Systeme für Produktionsumgebungen

Für anspruchsvollere Anwendungen und Produktionsumgebungen:

- **HashiCorp Vault**: Vollständiges Secret-Management-System
- **AWS Secrets Manager**: Verwaltung von Secrets in AWS
- **Azure Key Vault**: Secret-Management in Azure
- **Google Secret Manager**: Secret-Management in GCP

Beispiel mit HashiCorp Vault:

```yaml
version: '3.8'
services:
  app:
    image: myapp:latest
    environment:
      # Secrets werden zur Laufzeit von einem Script abgerufen
      - DB_PASSWORD=${DB_PASSWORD}
    entrypoint: ["/docker-entrypoint.sh"]
```

```bash
#!/bin/bash
# docker-entrypoint.sh
# Abrufen des Secrets von Vault
export DB_PASSWORD=$(vault read -field=password secret/db/credentials)
# Ausführen des eigentlichen Container-Startbefehls
exec "$@"
```

#### 4. GitHub Actions Secrets (für CI/CD)

Wenn du GitHub Actions für Continuous Integration und Deployment verwendest:

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      
      - name: Set up Docker Compose
        run: |
          docker compose version
          
      - name: Create .env file
        run: |
          echo "DB_ROOT_PASSWORD=${{ secrets.DB_ROOT_PASSWORD }}" > .env
          echo "DB_PASSWORD=${{ secrets.DB_PASSWORD }}" >> .env
          
      - name: Deploy with Docker Compose
        run: docker compose up -d
```

Geheimnisse werden in den GitHub-Repository-Einstellungen gespeichert und dann in Workflows verwendet.

### Best Practices für Secrets Management in GitHub-Projekten

1. **Niemals Secrets direkt in Code oder Compose-Dateien speichern**
   - Verwende immer Variablenersetzung: `${SECRET_NAME}`

2. **Dokumentiere, welche Secrets benötigt werden**
   - Erstelle eine Beispiel-Datei `.env.example` oder `secrets.example`

3. **Füge Secret-Dateien zu .gitignore hinzu**
   ```
   .env
   secrets/
   *.key
   *.pem
   ```

4. **Rotation und Überwachung**
   - Implementiere regelmäßige Rotation der Secrets
   - Überwache Zugriffe auf Secrets

5. **Empfohlen: Verwende eine Beispiel-Umgebungsdatei**

   ```
   # .env.example
   # Kopiere diese Datei zu .env und fülle die Werte aus
   DB_ROOT_PASSWORD=
   DB_NAME=myapp
   DB_USER=myapp
   DB_PASSWORD=
   ```

## Best Practices und erweiterte Konzepte

### Docker-Sicherheit

1. **Ausführen von Containern mit reduziertem Berechtigungsumfang**
   ```yaml
   services:
     app:
       user: "1000:1000"  # Nicht als root ausführen
   ```

2. **Container härten**
   ```dockerfile
   # Im Dockerfile
   # Nicht-root-Benutzer erstellen
   RUN addgroup --system appgroup && \
       adduser --system --ingroup appgroup appuser
   
   # Auf nicht-root-Benutzer wechseln
   USER appuser
   ```

3. **Verwendung von read-only Filesystems**
   ```yaml
   services:
     app:
       read_only: true
       tmpfs:
         - /tmp
         - /var/run
   ```

### Docker Compose Netzwerke

1. **Trennung von Frontend- und Backend-Netzwerken**
   ```yaml
   services:
     webserver:
       networks:
         - frontend
         - backend
     database:
       networks:
         - backend
   
   networks:
     frontend:
       driver: bridge
     backend:
       driver: bridge
       internal: true  # Kein direkter Zugriff von außen
   ```

2. **Netzwerk-Aliase für Service Discovery**
   ```yaml
   services:
     database:
       networks:
         backend:
           aliases:
             - db.internal
   ```

### Skalierung und Hochverfügbarkeit

1. **Replikation von Diensten**
   ```bash
   docker compose up -d --scale webserver=3
   ```

2. **Gesundheitsprüfungen**
   ```yaml
   services:
     app:
       healthcheck:
         test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
         interval: 30s
         timeout: 10s
         retries: 3
   ```

3. **Restart-Richtlinien**
   ```yaml
   services:
     app:
       restart: always  # Alternative: on-failure, unless-stopped
   ```
