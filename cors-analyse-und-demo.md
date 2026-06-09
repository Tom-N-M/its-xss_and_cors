# CORS-Analyse und Demo fuer das Projekt `scheduler`

Diese Unterlage ist ausschliesslich fuer die lokale Demo-Anwendung `scheduler` gedacht. Unsichere CORS-Konfigurationen sind bewusst als `DEV_ONLY` markiert und duerfen nicht in produktive Systeme uebernommen werden. Die Demo bleibt lokal auf `localhost` und nutzt keine fremden echten Systeme oder produktiven Daten.

## 1. Einstieg fuer Ole

Die Same-Origin Policy ist eine Browser-Schutzregel. Ein Skript von `http://localhost:5173` darf zwar grundsaetzlich einen Request an `http://localhost:8079` senden, aber es darf die Antwort nicht lesen, wenn die Zielseite das nicht per CORS erlaubt. Eine Origin besteht aus Schema, Host und Port. `http://localhost:8078` und `http://localhost:8079` sind deshalb verschiedene Origins.

CORS steht fuer Cross-Origin Resource Sharing. Damit kann ein Server ausdruecklich sagen, welche fremden Origins seine Antworten im Browser lesen duerfen. CORS ist also keine Authentifizierung, sondern eine browserseitige Lesefreigabe fuer Cross-Origin-Antworten.

Wichtige Header:

- `Access-Control-Allow-Origin`: Welche Origin darf die Antwort lesen? Sicher ist eine feste Allowlist, nicht pauschal jede Origin.
- `Access-Control-Allow-Credentials`: Duerfen Cookies, HTTP-Auth oder TLS-Client-Credentials bei Cross-Origin-Requests verwendet werden? Nur aktivieren, wenn wirklich noetig.
- `Access-Control-Allow-Methods`: Welche Methoden sind erlaubt, zum Beispiel `GET`, `POST`, `PUT`, `DELETE`.
- `Access-Control-Allow-Headers`: Welche Request-Header darf der Browser senden, zum Beispiel `Authorization` oder `Content-Type`.
- `Access-Control-Expose-Headers`: Welche Response-Header darf JavaScript lesen, zum Beispiel `Content-Disposition`.
- `Vary: Origin`: Wichtig bei dynamischer Origin-Auswahl, damit Caches Antworten nicht fuer die falsche Origin wiederverwenden.

Einfache Requests sind zum Beispiel bestimmte `GET`- oder `POST`-Requests ohne besondere Header. Sobald ein Request zum Beispiel `Authorization: Bearer ...` oder `Content-Type: application/json` nutzt, sendet der Browser vorher einen Preflight Request mit `OPTIONS`. Der Server muss diesen Preflight korrekt beantworten, sonst blockiert der Browser den eigentlichen Request.

CORS ersetzt keine Authentifizierung. Wenn ein API-Endpunkt vertrauliche Daten liefert, muss er weiterhin pruefen, wer der Nutzer ist und welche Rechte er hat. Falsches CORS ist gefaehrlich, weil eine fremde Webseite die API-Antworten im Browser lesen kann, wenn der Browser gueltige Credentials oder einen gueltigen Token mitsendet.

## Projektueberblick

- Backend: Spring Boot 3.0.1, Spring Security, OAuth2 Resource Server, Keycloak/JWT.
- Frontend: Vue 3 mit Vite.
- Ports laut `README.md:22-29`: Keycloak `8080`, API `8079`, Web `8078`, Postgres `5432`.
- Vite-Proxy: `web/vite.config.js:9-17` leitet lokale `/api`-Requests an `http://localhost:8079` weiter. Dadurch sieht der Browser im normalen Frontend keinen Cross-Origin-API-Aufruf.
- API-Client: `web/src/shared/http-client.js:164-180` setzt `Authorization: Bearer ...`, `credentials: 'include'` und ruft relativ `/api/...` auf.
- Keycloak-Token: `web/src/App.vue:118-124` initialisiert Keycloak und speichert den Token im Frontend-State.
- Keycloak/OAuth-Konfiguration: `scheduler.json:31` setzt `sslRequired` auf `none`, `scheduler.json:40` deaktiviert Brute-Force-Schutz, und der Web-Client nutzt bei `scheduler.json:859-867` `webOrigins: ["*"]`, `implicitFlowEnabled: true` und `directAccessGrantsEnabled: true`.
- Backend Security: `api/src/main/java/com/frostbear/scheduler/conf/SecurityConfiguration.java:35-60`
- Backend-Port: `api/src/main/resources/application.properties:4`, Dev-Issuer `api/src/main/resources/application-dev.properties:9`
- Oeffentliche API-Ausnahme: `SecurityConfiguration.java:48` erlaubt `/files/**` ohne Authentifizierung.
- Datei-Endpunkte: `api/src/main/java/com/frostbear/scheduler/admin/controller/FileController.java:30-125`
- Es gibt keine gefundene `@CrossOrigin`-Annotation und keine `CorsConfigurationSource`, `CorsFilter` oder `CorsRegistry` in `api/src/main/java`.

## 2. Aktuelle CORS-Situation im Projekt

### Konfiguration A: CORS ist aktuell nicht konfiguriert

- Beschreibung

  Im Backend gibt es keine explizite CORS-Konfiguration. Spring Security baut zwar eine Security-Filter-Chain, aber dort wird CORS nicht aktiviert. Die API gibt deshalb bei direkten Cross-Origin-Requests keine `Access-Control-Allow-Origin`-Header aus.

- Betroffene Dateien / Funktionen / Routen

  - `api/src/main/java/com/frostbear/scheduler/conf/SecurityConfiguration.java:35-60`
  - Keine Treffer fuer `@CrossOrigin`, `CorsConfiguration`, `CorsFilter`, `CorsRegistry`
  - Alle API-Routen, zum Beispiel:
    - `GET /session`
    - `POST /crud/{name}/search`
    - `GET /admin/schedule/group/{id}/planningperiod/{planningperiod_id}`
    - `GET /files/planningperiod/{id}/download`

- Warum ist das problematisch?

  Das ist fuer Cross-Origin-Lesen erst einmal sicher, weil der Browser Antworten fremder Origins blockiert. Es ist aber wichtig fuer die Demo: Direkt von `http://localhost:5173` nach `http://localhost:8079` sieht man im Browser einen CORS-Fehler.

- Wie kann man es in der Demo ausnutzen?

  Nicht als Angriff, sondern als sichere Baseline: Eine lokale Demo-Seite auf `http://localhost:5173` versucht `fetch("http://localhost:8079/session")`. Ohne CORS-Freigabe blockiert der Browser den Zugriff auf die Antwort.

- Was koennte ein Angreifer theoretisch damit erreichen?

  Mit dieser aktuellen CORS-Situation allein kann eine fremde Webseite die API-Antworten nicht lesen. Sie kann Requests zwar ausloesen, aber ohne gueltige Authentifizierung und ohne CORS-Lesefreigabe nicht sinnvoll auswerten.

- Sichere Gegenmassnahme

  CORS weiterhin nur bewusst und gezielt aktivieren. Standard: blockieren. Nur benoetigte Origins und Routen freigeben.

- Konkreter Fix im Projekt

  Kein Fix noetig, wenn das Frontend immer ueber denselben Origin oder einen Reverse Proxy ausgeliefert wird. Falls echtes Cross-Origin-Frontend noetig ist, sichere Allowlist wie in Abschnitt 5 verwenden.

- Demo-Idee: unsichere Variante vs. sichere Variante

  Sichere Variante: kein `Access-Control-Allow-Origin`, Browser blockiert. Unsichere Variante: Origin-Spiegelung oder breite Allowlist aktivieren, Browser kann Antwort lesen.

### Konfiguration B: Vite-Proxy vermeidet CORS in der normalen lokalen App

- Beschreibung

  Das Vue-Frontend ruft nicht direkt `http://localhost:8079/...` auf, sondern relativ `/api/...`. Vite leitet diese Requests an die API weiter. Fuer den Browser ist der Request dadurch same-origin zu `http://localhost:8078`.

- Betroffene Dateien / Funktionen / Routen

  - `web/vite.config.js:9-17`
  - `web/src/shared/http-client.js:180`
  - Alle normalen Frontend-API-Aufrufe

- Warum ist das problematisch?

  Das ist nicht problematisch, aber wichtig fuer die Praesentation: Im normalen Frontend sieht man kein CORS, weil der Browser nur `localhost:8078` sieht. Die CORS-Demo braucht deshalb eine separate Origin, zum Beispiel `localhost:5173`.

- Wie kann man es in der Demo ausnutzen?

  Normale App auf `http://localhost:8078` starten. Separaten Demo-Client auf `http://localhost:5173` starten. Nur der zweite Client erzeugt echte Cross-Origin-Requests an `http://localhost:8079`.

- Was koennte ein Angreifer theoretisch damit erreichen?

  Der Proxy selbst gibt einem Angreifer nichts. Er erklaert nur, warum CORS im normalen Entwicklungsmodus nicht sichtbar wird.

- Sichere Gegenmassnahme

  Fuer lokale Entwicklung ist ein Proxy sinnvoll. Fuer Produktion sollte ein Reverse Proxy die API und das Frontend unter einer kontrollierten Origin ausliefern oder eine enge CORS-Allowlist verwenden.

- Konkreter Fix im Projekt

  Wenn Production ueber nginx laufen soll, muss `/api` dort ebenfalls an das Backend weitergeleitet werden. Die aktuelle `nginx.conf:8-13` liefert nur statische Dateien und hat keine `/api`-Proxy-Regel.

- Demo-Idee: unsichere Variante vs. sichere Variante

  Sicher: Frontend nutzt `/api` ueber Vite-Proxy. Cross-Origin-Demo: separater Port ohne Proxy.

### Konfiguration C: `/files/**` ist oeffentlich erlaubt

- Beschreibung

  Spring Security erlaubt alle Requests unter `/files/**` ohne Authentifizierung. Der Controller liefert unter anderem Planungsdateien und ICS-Dateien aus. Einige davon nutzen Token im Pfad, andere wie `GET /files/planningperiod/{id}/download` liegen ebenfalls unter dieser oeffentlichen Regel.

- Betroffene Dateien / Funktionen / Routen

  - Security-Ausnahme: `api/src/main/java/com/frostbear/scheduler/conf/SecurityConfiguration.java:48`
  - Datei-Controller: `api/src/main/java/com/frostbear/scheduler/admin/controller/FileController.java:30-125`
  - Beispiele:
    - `GET /files/planningperiod/{id}/download`
    - `GET /files/token/{token}`
    - `GET /files/token/{token}/teacher/{id}/planningperiod/{planningperiod_id}/Stundenplan.ics`
    - `POST /files/get-or-create-ics-token`

- Warum ist das problematisch?

  Das ist prima fuer eine CORS-Demo, weil ein oeffentlicher Endpunkt ohne Bearer-Token getestet werden kann. Sicherheitstechnisch sollte aber geprueft werden, ob wirklich alle `/files/**`-Routen oeffentlich sein sollen. CORS schuetzt nicht vor direktem Download, es schuetzt nur davor, dass fremdes JavaScript die Antwort lesen darf.

- Wie kann man es in der Demo ausnutzen?

  Demo-Seite auf `localhost:5173` ruft `http://localhost:8079/files/planningperiod/1/download` auf. Ohne CORS kann JavaScript die Antwort nicht lesen. Mit unsicherem CORS kann die fremde Seite Blob-Groesse und erlaubte Header sehen.

- Was koennte ein Angreifer theoretisch damit erreichen?

  Wenn ein Endpunkt vertrauliche Dateien ohne Authentifizierung liefert und CORS breit erlaubt ist, koennte eine fremde Webseite diese Daten per JavaScript auslesen. Auch ohne CORS koennte ein direkter Link einen Download ausloesen; CORS betrifft aber das Lesen durch JavaScript.

- Sichere Gegenmassnahme

  Authentifizierung und Autorisierung fuer nicht-oeffentliche Dateien erzwingen. Tokenisierte Datei-Links nur fuer bewusst oeffentliche, zeitlich oder logisch begrenzte Downloads verwenden. CORS nicht global auf `/files/**` fuer jede Origin freigeben.

- Konkreter Fix im Projekt

  Security-Regel enger machen, falls nicht alle Dateien oeffentlich sein sollen:

  ```diff
  - http.authorizeHttpRequests().requestMatchers("/files/**").permitAll();
  + http.authorizeHttpRequests()
  +     .requestMatchers("/files/token/**").permitAll()
  +     .requestMatchers("/files/**").fullyAuthenticated();
  ```

  Je nach gewuenschter Fachlogik sollte zusaetzlich geprueft werden, ob der angemeldete Nutzer die konkrete Datei oder Planungsperiode sehen darf.

- Demo-Idee: unsichere Variante vs. sichere Variante

  Unsicher: CORS global erlauben und eine Datei von fremder Origin lesen. Sicher: CORS blockiert oder Route verlangt Authentifizierung und passende Rechte.

### Konfiguration D: Keycloak/OAuth-Origin-Konfiguration ist riskant

- Beschreibung

  Neben der Spring-API gibt es eine zweite browserrelevante Origin-Konfiguration: die Keycloak-Realm-Datei. Der Web-Client in `scheduler.json` erlaubt `webOrigins: ["*"]`, aktiviert den Implicit Flow und erlaubt Direct Access Grants. Das ist nicht dieselbe CORS-Schicht wie `Access-Control-Allow-Origin` im Spring-Backend, aber fuer die Demo wichtig: Auch der Identity Provider entscheidet, von welchen Browser-Origins Auth-Requests und Token-Flows erlaubt sind.

- Betroffene Dateien / Funktionen / Routen

  - Realm-Grundschutz: `scheduler.json:31` mit `"sslRequired": "none"`
  - Login-Schutz: `scheduler.json:40` mit `"bruteForceProtected": false`
  - Web-Client: `scheduler.json:859-867`
    - `"webOrigins": ["*"]`
    - `"implicitFlowEnabled": true`
    - `"directAccessGrantsEnabled": true`
  - Frontend-Initialisierung: `web/src/App.vue:118-124`

- Warum ist das problematisch?

  Eine Wildcard bei Web Origins ist fuer OAuth/OIDC-Clients zu breit. Sie erhoeht das Risiko, dass lokale oder versehentlich erlaubte fremde Origins in Login- und Token-Flows einbezogen werden. Der Implicit Flow ist fuer moderne SPAs unguenstig, weil Tokens direkt im Browser-Fragment landen koennen. Direct Access Grants erlauben Passwort-Grant-Flows, die fuer normale Browser-Apps in der Regel nicht gebraucht werden.

- Wie kann man es in der Demo ausnutzen?

  Fuer die lokale Demo kann Tom zeigen, dass Keycloak im Realm-Export keine enge Origin-Liste nutzt. Dazu `scheduler.json` oeffnen und die Stelle `webOrigins: ["*"]` zeigen. Dann im Browser erklaeren: Eine fremde lokale Origin wie `http://localhost:5173` ist damit aus Sicht des Clients nicht grundsaetzlich ausgeschlossen. Die Demo bleibt lokal und nutzt keine fremden Domains.

- Was koennte ein Angreifer theoretisch damit erreichen?

  Bei zusaetzlichen Fehlern wie XSS, unsicheren Redirect-URIs oder Token-Ablage im Browser koennte eine zu breite OAuth-Origin-Konfiguration Token-Flows und Browser-Interaktion erleichtern. CORS oder Web Origins ersetzen keine Authentifizierung und keine serverseitige Autorisierung.

- Sichere Gegenmassnahme

  Feste Origins statt `*`, keine unnoetigen OAuth-Flows, HTTPS ausserhalb lokaler Entwicklung, Brute-Force-Schutz aktivieren und Redirect-URIs eng halten. Fuer SPAs bevorzugt Authorization Code Flow mit PKCE statt Implicit Flow.

- Konkreter Fix im Projekt

  Fuer die lokale Demo kann `localhost:8078` erlaubt bleiben. Produktive Werte muessen aus der Deployment-Umgebung kommen und duerfen nicht pauschal `*` sein.

  ```diff
  - "sslRequired": "none",
  + "sslRequired": "external",

  - "bruteForceProtected": false,
  + "bruteForceProtected": true,

  - "webOrigins": [
  -   "*"
  - ],
  + "webOrigins": [
  +   "http://localhost:8078"
  + ],

  - "implicitFlowEnabled": true,
  + "implicitFlowEnabled": false,

  - "directAccessGrantsEnabled": true,
  + "directAccessGrantsEnabled": false,
  ```

- Demo-Idee: unsichere Variante vs. sichere Variante

  Unsicher: Realm-Export mit `webOrigins: ["*"]` und Implicit Flow zeigen. Sicher: konkrete lokale Origin eintragen, Implicit Flow deaktivieren und erklaeren, dass nur bekannte Frontends am Auth-Flow teilnehmen sollen.

## 3. Unsichere CORS-Konfiguration fuer Demo-Zwecke

### Variante A: Origin dynamisch spiegeln ohne Allowlist (`DEV_ONLY`, gefaehrlich)

- Beschreibung

  Der Server akzeptiert jede Origin und gibt sie als `Access-Control-Allow-Origin` zurueck. Zusammen mit `Access-Control-Allow-Credentials: true` ist das die gefaehrlichste Demo-Variante, weil eine fremde Webseite Antworten lesen kann, wenn der Browser gueltige Credentials mitsendet.

- Betroffene Dateien / Funktionen / Routen

  - Neue Demo-Konfiguration in `api/src/main/java/com/frostbear/scheduler/conf/SecurityConfiguration.java`
  - Alle Routen, wenn global registriert

- Warum ist das problematisch?

  Jede beliebige Webseite kann im Browser des Nutzers Cross-Origin-Requests an die API stellen und die Antworten lesen, wenn Authentifizierung vorhanden ist. Bei Cookie-basierten Sessions waere das besonders kritisch. In diesem Projekt nutzt die API Bearer-Tokens; eine fremde Webseite bekommt diesen Token nicht automatisch. Wenn der Token aber durch XSS verfuegbar wird oder fuer die Demo manuell eingetragen wird, kann sie Antworten lesen.

- Wie kann man es in der Demo ausnutzen?

  Lokale Demo-Seite auf `http://localhost:5173` ruft die API auf. Nach Aktivierung der unsicheren Konfiguration sind im Network Tab `Access-Control-Allow-Origin: http://localhost:5173` und bei Preflight `Access-Control-Allow-Headers: Authorization` sichtbar.

- Was koennte ein Angreifer theoretisch damit erreichen?

  API-Daten auslesen, Dateiantworten lesen, Response Header wie `Content-Disposition` lesen, und bei gueltigen Credentials API-Aktionen vorbereiten. CORS gibt keine Rechte, aber es hebt die Browser-Lesesperre auf.

- Sichere Gegenmassnahme

  Nie ungeprueft die Origin spiegeln. Immer eine feste Allowlist verwenden und Credentials nur erlauben, wenn sie wirklich gebraucht werden.

- Konkreter Fix im Projekt

  Unsichere `DEV_ONLY`-Demo-Konfiguration:

  ```java
  // DEV_ONLY_CORS_DEMO: gefaehrlich, nicht committen
  import org.springframework.context.annotation.Bean;
  import org.springframework.security.config.Customizer;
  import org.springframework.web.cors.CorsConfiguration;
  import org.springframework.web.cors.CorsConfigurationSource;
  import org.springframework.web.cors.UrlBasedCorsConfigurationSource;

  import java.util.List;

  @Bean
  CorsConfigurationSource corsConfigurationSource() {
      var config = new CorsConfiguration();
      config.setAllowedOriginPatterns(List.of("*"));
      config.setAllowCredentials(true);
      config.setAllowedMethods(List.of("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"));
      config.setAllowedHeaders(List.of("*"));
      config.setExposedHeaders(List.of("Content-Disposition"));

      var source = new UrlBasedCorsConfigurationSource();
      source.registerCorsConfiguration("/**", config);
      return source;
  }
  ```

  Und in `configureHttp`:

  ```diff
   public SecurityFilterChain configureHttp(HttpSecurity http) throws Exception {
+      http.cors(Customizer.withDefaults());
       http
               .csrf().disable();
  ```

- Demo-Idee: unsichere Variante vs. sichere Variante

  Unsicher: `localhost:5173` kann die Antwort lesen. Sicher: nur erlaubte Origins stehen in der Allowlist oder es gibt gar keine CORS-Freigabe.

### Variante B: `Access-Control-Allow-Origin: *`

- Beschreibung

  `*` erlaubt jeder Origin das Lesen von Antworten, aber nicht zusammen mit Credentials. Browser akzeptieren `Access-Control-Allow-Origin: *` nicht fuer Credential-Requests.

- Betroffene Dateien / Funktionen / Routen

  - CORS-Konfiguration im Backend
  - Besonders riskant fuer oeffentliche, nicht authentifizierte API-Endpunkte wie `/files/**`

- Warum ist das problematisch?

  Fuer oeffentliche Endpunkte kann jede fremde Webseite die Antwort lesen. Bei vertraulichen Daten ist das fatal. Bei credentialed APIs wirkt `*` nicht wie erwartet und fuehrt oft zu verwirrenden Fehlern.

- Wie kann man es in der Demo ausnutzen?

  Fuer `GET /files/planningperiod/1/download` reicht `*`, weil kein Token und keine Cookies noetig sind. Die Demo-Seite kann Blob-Daten lesen.

- Was koennte ein Angreifer theoretisch damit erreichen?

  Alle oeffentlich erreichbaren API-Antworten per JavaScript auswerten. Wenn oeffentliche Endpunkte vertrauliche Inhalte liefern, werden diese Inhalte browserseitig auslesbar.

- Sichere Gegenmassnahme

  `*` nur fuer bewusst oeffentliche, nicht vertrauliche, nicht credentialed Ressourcen verwenden. Fuer APIs mit Nutzerdaten: feste Allowlist.

- Konkreter Fix im Projekt

  Nicht verwenden:

  ```java
  config.setAllowedOrigins(List.of("*"));
  ```

  Stattdessen:

  ```java
  config.setAllowedOrigins(List.of("http://localhost:8078"));
  ```

- Demo-Idee: unsichere Variante vs. sichere Variante

  Unsicher: jede Origin kann public API lesen. Sicher: nur Scheduler-Frontend-Origin darf lesen oder gar keine Cross-Origin-Freigabe.

### Variante C: Zu breite Methoden und Header

- Beschreibung

  `allowedMethods("*")` und `allowedHeaders("*")` erlauben fuer Preflight praktisch alles. Das ist bequem, aber unnoetig breit.

- Betroffene Dateien / Funktionen / Routen

  - Globale CORS-Konfiguration, falls in `SecurityConfiguration.java` ergaenzt
  - API-Methoden wie `POST /crud/{name}`, `PUT /crud/{name}/{id}`, `DELETE /crud/{name}/{id}`

- Warum ist das problematisch?

  Je breiter die Freigabe, desto mehr fremde Frontends koennen komplexe Requests mit eigenen Headern stellen. Bei einem spaeteren Auth-Fehler oder falsch gesetzten Credentials wird der Schaden groesser.

- Wie kann man es in der Demo ausnutzen?

  Demo-Client sendet `Authorization` und `Content-Type: application/json`. Mit breiter Header-Freigabe klappt der Preflight. Ohne passende Freigabe blockiert der Browser vor dem eigentlichen Request.

- Was koennte ein Angreifer theoretisch damit erreichen?

  Authentifizierte API-Endpunkte leichter aus fremden Origins ansprechen, wenn der Token oder Cookie verfuegbar ist.

- Sichere Gegenmassnahme

  Nur benoetigte Methoden und Header erlauben.

- Konkreter Fix im Projekt

  ```java
  config.setAllowedMethods(List.of("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"));
  config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
  ```

- Demo-Idee: unsichere Variante vs. sichere Variante

  Unsicher: Preflight akzeptiert alles. Sicher: Preflight akzeptiert nur die fuer das Scheduler-Frontend noetigen Header.

## 4. CORS-Demo fuer Tom

### Ziel

Die Demo zeigt drei Zustaende:

1. Aktuell/sicher: Browser blockiert fremde Origin.
2. Unsicher: API spiegelt jede Origin, Demo-Seite kann Antwort lesen.
3. Sicherer Fix: Nur erlaubte Origins duerfen lesen.

### Startbefehle

Scheduler-Infrastruktur:

```powershell
docker-compose up
```

Backend:

```powershell
cd api
.\gradlew bootRun --args='--spring.profiles.active=dev'
```

Frontend:

```powershell
cd web
npm install
npm run dev
```

Demo-Angreifer-Seite auf anderer Origin. Der HTML-Demo-Client aus dem naechsten Abschnitt wird dabei als `cors-demo/index.html` verwendet:

```powershell
mkdir cors-demo
cd cors-demo
python -m http.server 5173
```

Dann `http://localhost:5173` im Browser oeffnen.

### Lokaler Demo-Client als HTML/JS-Snippet

Diese Seite bleibt lokal. Der Token ist optional und nur fuer die lokale Demo gedacht. Fuer `/files/...` ist im aktuellen Projekt wegen `permitAll` kein Token noetig. Fuer `/session` wird ein Bearer-Token benoetigt.

```html
<!doctype html>
<html lang="de">
<head>
  <meta charset="utf-8">
  <title>Lokale CORS Demo</title>
  <style>
    body { font-family: system-ui, sans-serif; margin: 2rem; max-width: 900px; }
    label { display: block; margin-top: 1rem; font-weight: 600; }
    input, textarea, button { width: 100%; box-sizing: border-box; margin-top: .35rem; padding: .55rem; }
    button { cursor: pointer; }
    pre { background: #111; color: #eee; padding: 1rem; overflow: auto; min-height: 10rem; }
  </style>
</head>
<body>
  <h1>Lokale CORS Demo</h1>

  <label>
    API URL
    <input id="url" value="http://localhost:8079/session">
  </label>

  <label>
    Bearer Token optional, fuer /session noetig
    <textarea id="token" rows="4" placeholder="Nur lokalen Demo-Token einfuegen"></textarea>
  </label>

  <label>
    <input id="includeCredentials" type="checkbox">
    credentials: "include"
  </label>

  <button id="fetchJson">Fetch scheduler data</button>
  <button id="fetchFile">Fetch public scheduler file demo</button>

  <pre id="out">Noch kein Request.</pre>

  <script>
    const out = document.querySelector("#out");

    function log(value) {
      out.textContent = typeof value === "string" ? value : JSON.stringify(value, null, 2);
    }

    async function fetchWithOptions(url, asBlob) {
      const token = document.querySelector("#token").value.trim();
      const includeCredentials = document.querySelector("#includeCredentials").checked;
      const headers = {};

      if (token) {
        headers.Authorization = "Bearer " + token;
      }

      const response = await fetch(url, {
        method: "GET",
        headers,
        credentials: includeCredentials ? "include" : "omit"
      });

      const result = {
        ok: response.ok,
        status: response.status,
        type: response.type,
        url: response.url,
        contentType: response.headers.get("content-type"),
        contentDisposition: response.headers.get("content-disposition")
      };

      if (asBlob) {
        const blob = await response.blob();
        result.blobSize = blob.size;
      } else {
        result.body = await response.text();
      }

      return result;
    }

    document.querySelector("#fetchJson").addEventListener("click", async () => {
      try {
        log(await fetchWithOptions(document.querySelector("#url").value, false));
      } catch (err) {
        log("Browser hat den Zugriff blockiert oder Request ist fehlgeschlagen:\n" + err);
      }
    });

    document.querySelector("#fetchFile").addEventListener("click", async () => {
      try {
        log(await fetchWithOptions("http://localhost:8079/files/planningperiod/1/download", true));
      } catch (err) {
        log("Browser hat den Zugriff blockiert oder Request ist fehlgeschlagen:\n" + err);
      }
    });
  </script>
</body>
</html>
```

Falls `planningperiod/1` im lokalen Datenbestand nicht existiert, fuer die Datei-Demo eine existierende Planungsperioden-ID einsetzen. Fuer den CORS-Effekt reicht aber auch ein Fehlerstatus: Entscheidend ist, ob JavaScript die Antwort wegen CORS lesen darf.

### Schrittfolge

#### Zustand 1: CORS deaktiviert / sicherer Ist-Zustand

1. Scheduler-Backend auf `http://localhost:8079` starten.
2. Demo-Seite auf `http://localhost:5173` starten.
3. In der Demo-Seite `Fetch public scheduler file demo` klicken.
4. Optional fuer `/session`: In der echten Scheduler-App einloggen, DevTools oeffnen, lokalen Demo-Token aus `window.$keycloak.token` lesen und in die Demo-Seite eintragen. Dann `Fetch scheduler data` klicken.

Erwartetes Ergebnis:

- Console: CORS-Fehler wie "No 'Access-Control-Allow-Origin' header..."
- Network Tab: Request an `localhost:8079`, aber JavaScript kann die Antwort nicht lesen.
- Bei `Authorization`-Header sieht man einen `OPTIONS` Preflight, der nicht die noetigen CORS-Header bekommt.

#### Zustand 2: Unsichere `DEV_ONLY` CORS-Konfiguration

1. Unsichere Konfiguration aus Abschnitt 3 Variante A lokal aktivieren.
2. Backend neu starten.
3. Demo-Seite neu laden.
4. Erneut `Fetch public scheduler file demo` oder mit Token `Fetch scheduler data` klicken.

Erwartetes Ergebnis:

- JavaScript sieht die Antwort.
- Network Tab zeigt:
  - `Access-Control-Allow-Origin: http://localhost:5173`
  - optional `Access-Control-Allow-Credentials: true`
  - bei Preflight `Access-Control-Allow-Headers` mit `Authorization`
  - `Access-Control-Expose-Headers: Content-Disposition`, falls gesetzt

#### Zustand 3: Sichere Konfiguration

1. Unsichere Origin-Spiegelung entfernen.
2. Allowlist nur fuer benoetigte Origins setzen.
3. Backend neu starten.
4. Demo-Seite von `http://localhost:5173` testen.

Erwartetes Ergebnis:

- Wenn `localhost:5173` nicht in der Allowlist steht: Browser blockiert.
- Wenn `localhost:5173` nur fuer die Demo erlaubt ist: Browser erlaubt genau diese Origin.
- Eine andere Origin, zum Beispiel ein anderer Port, wird blockiert.

### DevTools-Hinweise

- Console:
  - CORS-Fehler erklaeren: Request ging raus, Antwort darf nicht gelesen werden.
- Network Tab:
  - `OPTIONS` Request ist der Preflight.
  - Beim eigentlichen Request Response Headers ansehen.
  - `Access-Control-Allow-Origin` muss zur Origin passen.
  - Bei dynamischer Allowlist muss `Vary: Origin` vorhanden sein.
  - `Access-Control-Allow-Credentials: true` ist nur mit konkreter Origin sinnvoll, nicht mit `*`.

## 5. Sichere CORS-Konfiguration

### Sichere Konfiguration: feste Allowlist und blockierende Defaults

- Beschreibung

  CORS soll nur fuer bekannte Frontend-Origins aktiviert werden. In diesem Projekt ist fuer die normale lokale App keine Cross-Origin-Freigabe noetig, weil Vite `/api` proxyt. Wenn ein separates Frontend oder eine Demo-Origin erlaubt werden soll, muss diese explizit in die Allowlist.

- Betroffene Dateien / Funktionen / Routen

  - `api/src/main/java/com/frostbear/scheduler/conf/SecurityConfiguration.java`
  - optional neue Properties:
    - `api/src/main/resources/application.properties`
    - `api/src/main/resources/application-dev.properties`

- Warum ist das problematisch?

  Ohne klare Allowlist neigt CORS dazu, "kurz fuer die Entwicklung" zu breit aktiviert zu werden. Dann bleibt eine gefaehrliche Produktionskonfiguration zurueck.

- Wie kann man es in der Demo ausnutzen?

  Erst `http://localhost:5173` in die Allowlist aufnehmen und zeigen, dass es geht. Danach Origin entfernen oder anderen Port verwenden und zeigen, dass der Browser blockiert.

- Was koennte ein Angreifer theoretisch damit erreichen?

  Bei falscher Allowlist kann eine beliebige fremde Seite API-Antworten lesen, sobald sie gueltige Credentials oder einen Token hat. Bei korrekter Allowlist bleibt das auf kontrollierte Frontends beschraenkt.

- Sichere Gegenmassnahme

  Feste Allowlist, keine Origin-Spiegelung, keine globale Freigabe fuer alles, konkrete Methoden und Header, `Vary: Origin` bei dynamischer Entscheidung, getrennte Development- und Production-Werte.

- Konkreter Fix im Projekt

  Beispiel fuer eine sichere Konfiguration:

  ```java
  package com.frostbear.scheduler.conf;

  import org.springframework.beans.factory.annotation.Value;
  import org.springframework.context.annotation.Bean;
  import org.springframework.context.annotation.Configuration;
  import org.springframework.security.config.Customizer;
  import org.springframework.security.config.annotation.web.builders.HttpSecurity;
  import org.springframework.security.web.SecurityFilterChain;
  import org.springframework.web.cors.CorsConfiguration;
  import org.springframework.web.cors.CorsConfigurationSource;
  import org.springframework.web.cors.UrlBasedCorsConfigurationSource;

  import java.util.List;

  @Configuration
  public class SecurityConfiguration {
      @Value("${scheduler.cors.allowed-origins:}")
      private List<String> allowedOrigins;

      @Bean
      public SecurityFilterChain configureHttp(HttpSecurity http) throws Exception {
          http.cors(Customizer.withDefaults());
          http.csrf().disable();

          // bestehende authorizeHttpRequests- und oauth2ResourceServer-Konfiguration beibehalten

          return http.build();
      }

      @Bean
      CorsConfigurationSource corsConfigurationSource() {
          var config = new CorsConfiguration();
          config.setAllowedOrigins(allowedOrigins);
          config.setAllowCredentials(false);
          config.setAllowedMethods(List.of("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"));
          config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
          config.setExposedHeaders(List.of("Content-Disposition"));
          config.setMaxAge(3600L);

          var source = new UrlBasedCorsConfigurationSource();
          source.registerCorsConfiguration("/**", config);
          return source;
      }
  }
  ```

  Development-Properties:

  ```properties
  # api/src/main/resources/application-dev.properties
  scheduler.cors.allowed-origins=http://localhost:8078,http://localhost:5173
  ```

  Production-Properties:

  ```properties
  # api/src/main/resources/application.properties
  scheduler.cors.allowed-origins=https://planr.ai.frostbear.com
  ```

  Hinweis: `credentials` ist fuer den aktuellen Bearer-Token-Flow nicht noetig. `Authorization` ist kein Browser-Credential im CORS-Sinn, sondern ein expliziter Header. Wenn spaeter Cookie-basierte API-Sessions eingefuehrt werden, muss `allowCredentials(true)` nur mit konkreter Origin und zusaetzlichem CSRF-Schutz verwendet werden.

  Bei Spring CORS werden `Vary`-Header typischerweise durch die CORS-Verarbeitung gesetzt. Wenn eine eigene CORS-Header-Logik statt `CorsConfigurationSource` gebaut wird, muss mindestens `Vary: Origin` explizit gesetzt werden; bei Preflight zusaetzlich `Vary: Access-Control-Request-Method` und `Vary: Access-Control-Request-Headers`.

- Demo-Idee: unsichere Variante vs. sichere Variante

  Unsicher: `allowedOriginPatterns("*")`, `allowCredentials(true)`, alle Methoden/Header. Sicher: `allowedOrigins` aus Properties, `allowCredentials(false)`, konkrete Methoden/Header, Demo-Origin nur in `dev`.

### Sichere Routenbegrenzung: CORS nicht global fuer alles

- Beschreibung

  CORS muss nicht zwingend fuer jede Route gelten. Falls nur bestimmte API-Bereiche von einem separaten Frontend gebraucht werden, kann die Registrierung auf diese Pfade begrenzt werden.

- Betroffene Dateien / Funktionen / Routen

  - `UrlBasedCorsConfigurationSource`
  - Beispielrouten:
    - `/crud/**`
    - `/session`
    - `/admin/**`
    - `/lecturers/**`
    - `/files/**`

- Warum ist das problematisch?

  Globale CORS-Freigaben machen auch unerwartete oder versehentlich oeffentliche Endpunkte aus fremden Origins lesbar.

- Wie kann man es in der Demo ausnutzen?

  CORS nur fuer `/session` registrieren. Die Demo zeigt: `/session` klappt, `/files/planningperiod/1/download` bleibt blockiert.

- Was koennte ein Angreifer theoretisch damit erreichen?

  Bei globalem CORS koennte eine fremde Webseite mehr API-Oberflaeche untersuchen und lesen. Bei routenbegrenztem CORS wird die Angriffsoberflaeche kleiner.

- Sichere Gegenmassnahme

  CORS nur fuer benoetigte API-Pfade registrieren und oeffentliche Download-Endpunkte gesondert bewerten.

- Konkreter Fix im Projekt

  ```java
  source.registerCorsConfiguration("/session", config);
  source.registerCorsConfiguration("/crud/**", config);
  source.registerCorsConfiguration("/admin/**", config);
  source.registerCorsConfiguration("/lecturers/**", config);
  // /files/** nur freigeben, wenn diese Antworten bewusst von fremden Frontends gelesen werden sollen.
  ```

- Demo-Idee: unsichere Variante vs. sichere Variante

  Unsicher: `source.registerCorsConfiguration("/**", config)`. Sicher: nur die konkret benoetigten API-Pfade.

### Sichere Credentials-Entscheidung

- Beschreibung

  Das Frontend setzt aktuell `credentials: 'include'` in `web/src/shared/http-client.js:164-170`. Fuer den aktuellen Bearer-JWT-Flow ist das nicht erforderlich.

- Betroffene Dateien / Funktionen / Routen

  - `web/src/shared/http-client.js:164-170`
  - CORS-Konfiguration im Backend, falls `allowCredentials(true)` gesetzt wird

- Warum ist das problematisch?

  `credentials: include` kann spaeter unbeabsichtigt Cookies an Cross-Origin-APIs senden. Wenn dann CORS zu breit ist und Cookies fuer API-Auth genutzt werden, entsteht eine riskante Kombination.

- Wie kann man es in der Demo ausnutzen?

  Demo-Client mit Checkbox `credentials: "include"` verwenden. In der aktuellen API bringt das fuer Bearer-Auth keinen Vorteil. Die Erklaerung: Fuer Cookie-basierte Sessions waere genau diese Kombination kritisch.

- Was koennte ein Angreifer theoretisch damit erreichen?

  Bei Cookie-Session plus falschem CORS koennte eine fremde Seite Antworten im Kontext des eingeloggten Nutzers lesen.

- Sichere Gegenmassnahme

  Credentials nur aktivieren, wenn sie wirklich benoetigt werden. Bei Bearer-Token-API `credentials` im Frontend auf `omit` oder `same-origin` reduzieren.

- Konkreter Fix im Projekt

  ```diff
  // web/src/shared/http-client.js
   const payload = {
       method: req.method,
       headers: {
           Authorization: 'Bearer ' + session.state.token,
       },
  -    credentials: 'include',
  +    credentials: 'same-origin',
   };
  ```

  Falls die API wirklich Cookie-Credentials braucht, dann stattdessen CORS eng konfigurieren:

  ```java
  config.setAllowedOrigins(List.of("https://planr.ai.frostbear.com"));
  config.setAllowCredentials(true);
  ```

- Demo-Idee: unsichere Variante vs. sichere Variante

  Unsicher: fremde Origin plus Credentials plus breite Origin-Freigabe. Sicher: keine Credentials fuer Bearer-Flow oder Credentials nur fuer exakt erlaubte Origins.

## 6. Praesentationsstruktur fuer CORS

### Ole: Grundlagen

- Same-Origin Policy mit `localhost:8078` vs. `localhost:8079` erklaeren.
- CORS als ausdrueckliche Lesefreigabe des Servers erklaeren.
- Header und Preflight kurz an einem Network-Tab-Beispiel zeigen.
- Klarstellen: CORS ist keine Authentifizierung.

### Ole: Problem im Projektkontext

- Normale App nutzt Vite-Proxy, daher kein sichtbares CORS im Alltag.
- Direkter API-Aufruf von anderer Origin wird aktuell blockiert.
- `/files/**` ist oeffentlich und daher ein guter Demo-Endpunkt, aber auch ein Security-Punkt zur Pruefung.
- Keycloak-Realm zeigt zusaetzlich eine riskante OAuth-Origin-Konfiguration mit `webOrigins: ["*"]`.

### Tom: Live-Demo sichere Baseline

- Scheduler starten.
- Demo-Seite auf `http://localhost:5173` starten.
- `Fetch public scheduler file demo` klicken.
- Browser blockiert Zugriff wegen fehlendem `Access-Control-Allow-Origin`.

### Tom: Unsichere Konfiguration zeigen

- `DEV_ONLY` CORS mit `allowedOriginPatterns("*")` und `allowCredentials(true)` aktivieren.
- Backend neu starten.
- Demo-Seite erneut ausfuehren.
- Antwort wird lesbar.
- Im Network Tab CORS-Header und Preflight zeigen.

### Tom: Sichere Konfiguration und Fazit

- Feste Allowlist aus Properties zeigen.
- Keycloak-Web-Origins ebenfalls auf konkrete Frontend-Origins begrenzen.
- `localhost:5173` nur im Dev-Profil erlauben oder entfernen.
- Konkrete Methoden/Header setzen.
- Credentials nur falls noetig.
- Fazit: Auth prueft Rechte, CORS regelt nur, welche fremden Browser-Origins Antworten lesen duerfen.
