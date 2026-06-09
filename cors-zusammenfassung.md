# CORS-Zusammenfassung fuer die Praesentation

Stand nach Zweitpruefung des aktuellen Checkouts: Die Spring-API hat aktuell keine klassische Allow-All-CORS-Konfiguration. Trotzdem gibt es CORS-/Origin-relevante Risiken, vor allem ueber Keycloak/OAuth, oeffentliche Datei-Endpunkte und zu breite Demo-Konfigurationen.

Pruefumfang: getrackte Source-/Konfigurationsdateien aus Backend, Frontend, Docker, Nginx und Keycloak-Realm; Build-/DB-Artefakte wurden nicht als Anwendungscode bewertet.

## Kernaussage

- API-CORS aktuell nicht aktiv: keine Treffer fuer `@CrossOrigin`, `CorsConfigurationSource`, `CorsFilter`, `CorsRegistry` oder `http.cors()` in `api/src/main/java`.
- Browser blockiert direkte Cross-Origin-Antworten von `http://localhost:5173` nach `http://localhost:8079`.
- Normale lokale App vermeidet CORS durch Vite-Proxy: `/api` wird an `http://localhost:8079` weitergeleitet.
- Kritisch fuer Demo und Sicherheit: `/files/**` ist komplett `permitAll`.
- Keycloak/OAuth ist zu offen: `webOrigins: ["*"]`, Implicit Flow und Direct Access Grants sind aktiv.
- `web/src/shared/http-client.js` setzt `credentials: 'include'`, obwohl der aktuelle Flow Bearer-Token nutzt.

## Wichtigste Dateien

- API-Security: `api/src/main/java/com/frostbear/scheduler/conf/SecurityConfiguration.java:35-60`
- Oeffentliche Dateien: `SecurityConfiguration.java:48`, `api/src/main/java/com/frostbear/scheduler/admin/controller/FileController.java:30-125`
- Vite-Proxy: `web/vite.config.js:9-17`
- Frontend-Requests: `web/src/shared/http-client.js:164-180`
- Keycloak Realm: `scheduler.json:31,40,855-867`
- Frontend-Keycloak-Config: `web/.env.local:2`
- Nginx ohne `/api`-Proxy: `nginx.conf:8-13`

## Demo-Ablauf fuer Tom

1. Scheduler lokal starten: API `http://localhost:8079`, Frontend `http://localhost:8078`.
2. Lokale Demo-Seite auf anderer Origin starten, z. B. `http://localhost:5173`.
3. Von dort `fetch("http://localhost:8079/session")` oder `/files/planningperiod/1/download` ausfuehren.
4. Sicherer Ist-Zustand: Browser zeigt CORS-Fehler, Antwort ist fuer JavaScript nicht lesbar.
5. `DEV_ONLY` unsichere CORS-Konfiguration aktivieren.
6. Demo erneut ausfuehren: Antwort wird lesbar, Header im Network Tab zeigen.
7. Sichere Allowlist-Konfiguration zeigen.

## Unsichere Demo-Konfiguration

Nur lokal und klar markiert:

```java
// DEV_ONLY_CORS_DEMO: gefaehrlich
config.setAllowedOriginPatterns(List.of("*"));
config.setAllowCredentials(true);
config.setAllowedMethods(List.of("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"));
config.setAllowedHeaders(List.of("*"));
source.registerCorsConfiguration("/**", config);
```

Warum gefaehrlich: Eine fremde Origin koennte API- oder Dateiantworten lesen, wenn Authentifizierung, Tokens oder oeffentliche Endpunkte vorhanden sind.

## Sicherer Fix

```java
config.setAllowedOrigins(List.of("http://localhost:8078"));
config.setAllowCredentials(false);
config.setAllowedMethods(List.of("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"));
config.setAllowedHeaders(List.of("Content-Type", "Authorization"));
config.setExposedHeaders(List.of("Content-Disposition"));
source.registerCorsConfiguration("/session", config);
source.registerCorsConfiguration("/crud/**", config);
source.registerCorsConfiguration("/admin/**", config);
source.registerCorsConfiguration("/lecturers/**", config);
```

Zusaetzlich:

- Keine Origin-Spiegelung ohne Allowlist.
- `Vary: Origin` bei dynamischer Origin-Auswahl.
- `/files/**` nicht global fuer fremde Origins freigeben.
- Datei-Erzeugung und Voll-Exports authentifizieren und autorisieren.
- Keycloak `webOrigins` auf konkrete Origins setzen.
- `implicitFlowEnabled=false`, `directAccessGrantsEnabled=false` fuer Browser-Client.
- `credentials: 'include'` im Frontend auf `same-origin` oder `omit` reduzieren, solange keine Cookie-Session genutzt wird.

## Praesentationssatz

CORS gibt keine Rechte und ersetzt keine Authentifizierung. CORS entscheidet nur, ob fremdes Browser-JavaScript eine Antwort lesen darf. Im Scheduler ist die API aktuell nicht offen per CORS, aber eine falsche Demo-Konfiguration plus `/files/**` oder Tokens macht den Unterschied sofort sichtbar.
