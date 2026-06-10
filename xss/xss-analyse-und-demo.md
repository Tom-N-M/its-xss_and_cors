# XSS-Analyse und Demo fuer das Projekt `scheduler`

Diese Unterlage ist ausschliesslich fuer die lokale Demo-Anwendung `scheduler` gedacht. Die unsicheren Varianten sind bewusst als `DEV_ONLY` markiert und sollen nicht in produktive Branches, echte Systeme oder gegen echte Nutzer eingesetzt werden.

## 1. Einstieg fuer Linus

Cross-Site Scripting (XSS) bedeutet, dass fremder JavaScript- oder HTML-Code in einer Webanwendung ausgefuehrt wird, obwohl die Anwendung ihn eigentlich nur als normale Nutzereingabe behandeln sollte. Im lokalen Scheduler waeren typische Eingaben zum Beispiel Modulnamen, Veranstaltungsnamen, Gruppeninformationen, Kommentare oder Annotationen in Kalenderterminen.

Stored XSS entsteht, wenn eine manipulierte Eingabe gespeichert wird und spaeter bei jedem Aufruf wieder ausgegeben wird. Reflected XSS entsteht, wenn eine Eingabe direkt aus der Anfrage, zum Beispiel aus einem Suchparameter, in die Antwort gerendert wird. DOM-based XSS entsteht komplett im Browser, wenn JavaScript Daten aus URL, Storage oder API-Antworten unsicher in das DOM schreibt.

XSS ist gefaehrlich, weil der Code im Kontext der echten Anwendung laeuft. In diesem Projekt bedeutet das: Ein Skript koennte im Browser eines eingeloggten Demo-Nutzers sichtbare Daten auslesen, UI-Elemente manipulieren, lokale Requests an die Scheduler-API senden oder auf den im Frontend verfuegbaren Keycloak-Token zugreifen. Der Scheduler speichert den Token nicht dauerhaft in `localStorage`, aber er liegt waehrend der Sitzung in JavaScript-Objekten wie `window.$keycloak` und im Vuex-Store.

Typische Ursachen sind ungefilterte Benutzereingaben, unsicheres Rendering von HTML, direkte Nutzung von `innerHTML`, Vue `v-html`, React `dangerouslySetInnerHTML`, fehlendes Escaping, unsichere Markdown- oder HTML-Verarbeitung und unescaped Template-Ausgabe. Das Grundprinzip fuer sichere Web-UIs ist: Nutzereingaben werden als Text behandelt, nicht als HTML.

Was ein Angreifer theoretisch mit XSS erreichen koennte:

- Aktionen im Namen des eingeloggten Nutzers ausfuehren
- Session- oder Token-Diebstahl, wenn Tokens im Frontend erreichbar sind
- Phishing innerhalb der Anwendung einblenden
- Kalender, Tabellen oder Formulare optisch manipulieren
- sichtbare Daten aus der Seite auslesen
- Requests an die eigene Scheduler-API im Kontext des eingeloggten Nutzers senden

## Projektueberblick

- Backend: Spring Boot 3.0.1, Java 17, Spring Security, OAuth2 Resource Server, JDBC/Postgres.
- Frontend: Vue 3 mit Vite, Vue Router, Vuex, Bootstrap, `fetch`.
- Authentifizierung: Keycloak lokal auf `http://localhost:8080`, API auf `http://localhost:8079`, Frontend auf `http://localhost:8078`.
- Lokale Entwicklung: Das Frontend ruft `/api/...` auf. Vite proxyt diese Requests an `http://localhost:8079`, siehe `web/vite.config.js:9-17`.
- API-Authentifizierung: Bearer-JWT im Header `Authorization`, gesetzt in `web/src/shared/http-client.js:164-180`.
- Keycloak-Token: wird in `web/src/App.vue:118-124` und `web/src/shared/stores/session.store.js:25,73-74` im JS-Kontext gehalten.
- Backend-Security: `api/src/main/java/com/frostbear/scheduler/conf/SecurityConfiguration.java:35-60` deaktiviert CSRF, schaltet Form Login/Logout aus, erlaubt Swagger, `/files/**` und `/public`, und verlangt sonst Authentifizierung.
- Es gibt keine gefundene Verwendung von `innerHTML`, `v-html`, `dangerouslySetInnerHTML` oder `insertAdjacentHTML` in `web/src`.
- Es gibt keine explizite Content Security Policy in `web/index.html` oder `nginx.conf`.

Relevante Datenfluesse:

- CRUD-Formulare: `web/src/shared/components/fbFormFields.vue`, `fbInput.vue`, `fbTextArea.vue`
- CRUD-Tabellen: `web/src/shared/components/fbTable.vue`, `fbTable/fbTableColumn.vue`, `fbTable/fbArrayColumn.vue`, `fbTable/fbLinkArrayColumn.vue`
- Generische CRUD-API: `api/src/main/java/com/frostbear/scheduler/shared/controller/CrudController.java`
- Suche: `web/src/shared/components/fbSearch.vue:205-226`, `api/src/main/java/com/frostbear/scheduler/shared/controller/CrudController.java:81-96`
- Kalender/Scheduler-Anzeige: `web/src/admin/views/data-admin/PlanningPeriodCrud/ScheduleOverview.vue`
- Annotationen: Frontend `ScheduleOverview.vue:647-650`, Backend `ScheduleController.java:92-95`, DTO `AnnotateCommand.java:8-11`
- Modul-Kommentar: Frontend `ModuleCrudView.vue:67,100-106`, Backend `ModuleCrud.java` mit `ModulePayload.comment`
- Gruppeninfo: Frontend `GroupCrudView.vue:132-136,206-210`, Backend `GroupCrud.java` mit `GroupPayload.groupInfo`
- Veranstaltungsname: Frontend `LectureTemplateCrudView.vue:37,59-64`, Backend `LectureTemplateCrud.java`
- Dozentenfelder: Frontend `TeacherCrudView.vue`, Backend `TeacherCrud.java`
- HTML-E-Mail-Templates: `api/src/main/resources/emailTemplates/*`, Einsetzen per `.formatted(...)` in `api/src/main/java/com/frostbear/scheduler/mail/*Mail.java`

Kurze Einordnung nach Abgleich mit `SECURITY_REPORT.md`:

- Tatsaechlich vorhandene Browser-XSS-Schwachstelle: In eigener Vue-Logik wurde zwar keine direkte Verwendung von `innerHTML` oder `v-html` gefunden, aber die Drittbibliothek `vue-cal` rendert Event-Titel und Event-Inhalte im Default-Rendering per `innerHTML`. Kalenderansichten ohne eigenen sicheren Event-Slot sind deshalb kritisch.
- Potenziell riskante Stellen: gespeicherte Freitextfelder wie `name`, `comment`, `groupInfo`, `annotation`, `lectureInfo`, Dozentennamen und Titel. Sie sind riskant, sobald sie unsicher als HTML ausgegeben werden.
- Bereits sichere Stellen: zentrale Vue-Interpolation in Tabellen, Notifications, Links und in `ScheduleOverview.vue` mit eigenem `VueCal`-Slot. Diese Stellen escapen Text aktuell korrekt.

## 2. Konkrete XSS-Analyse des Projekts

### Befund 0: Kritisch - Stored DOM-XSS ueber `vue-cal` Default-Rendering

- Beschreibung

  Das Projekt nutzt `vue-cal` fuer Kalenderansichten. Diese Bibliothek rendert im Default-Event-Template `event.title` und `event.content` per `innerHTML`. In mehreren Projektansichten werden datenbankbasierte Werte als `title` oder `content` an `VueCal` uebergeben, ohne einen eigenen sicheren Event-Slot zu definieren.

- Betroffene Dateien / Funktionen / Routen

  - Bibliothek: `web/node_modules/vue-cal/dist/vue-cal.es.js:962` enthaelt `innerHTML` fuer Event-Titel und Event-Inhalte.
  - Teacher-Kalender: `web/src/teacher/views/AssignmentView.vue:106-116` setzt `title` aus `schedule.lecture.lectureName` und `content` aus `schedule.group.name`; `AssignmentView.vue:140-146` erweitert `content`.
  - Admin-Reservierungskalender: `web/src/admin/views/data-admin/PlanningPeriodCrud/ReserveTimeslot.vue:346-351` und `398-402` setzen `title` aus `blockedBy`, `lectureName` oder `label`.
  - `ReserveTimeslot.vue:504-522` rendert `<VueCal :events="events">` ohne eigenen `event`-Slot.
  - Admin-Ansicht fuer unvollstaendige Verfuegbarkeiten: `web/src/admin/views/data-admin/PlanningPeriodCrud/EditIncompleteStateAvailabilities.vue:99-109` setzt `content` aus `timeslot.groupName` oder absichtlich auf ein Icon-HTML-Fragment; `EditIncompleteStateAvailabilities.vue:458-477` rendert ohne eigenen `event`-Slot.
  - Teacher-Verfuegbarkeitskalender: `web/src/teacher/views/TimeSlotCalendarView.vue:895-900` nutzt aktuell nur statisches Icon-HTML in `content`, rendert aber ebenfalls ueber `VueCal` Default-Rendering bei `TimeSlotCalendarView.vue:1095-1118`. Das ist eher eine riskante Pattern-Stelle als eine direkte Freitext-XSS-Stelle.
  - Positives Gegenbeispiel: `web/src/admin/views/data-admin/PlanningPeriodCrud/ScheduleOverview.vue:1008-1013` nutzt einen eigenen Slot und Vue-Interpolation.

- Warum ist das problematisch?

  Die Anwendung kann Namen und Labels speichern, die aus Formularen, Stammdaten oder Importen kommen. Wenn ein solcher gespeicherter Wert HTML enthaelt, wird er in den betroffenen `vue-cal`-Ansichten nicht als Text behandelt, sondern als HTML in das DOM geschrieben. Das ist Stored DOM-XSS.

- Wie kann man es in der Demo ausnutzen?

  Lokal einen harmlosen Demo-Wert in einem Feld speichern, das spaeter als Kalender-Event-Titel erscheint, zum Beispiel Veranstaltungsname, Gruppenname oder Reservierungslabel:

  ```html
  XSS Demo <img src=x onerror="alert('XSS Demo lokal ueber vue-cal')">
  ```

  Danach die betroffene Kalenderansicht oeffnen:

  - Teacher: `/teacher/assignment/:id`, wenn der manipulierte Veranstaltungs- oder Gruppenname dort als Event erscheint.
  - Admin: `/admin/data-admin/planningperiods/:id/reserve-timeslots/:idGroup`, wenn ein manipuliertes Reservierungslabel oder ein manipulierter Veranstaltungstitel dort als Event erscheint.

- Was koennte ein Angreifer theoretisch damit erreichen?

  Ein Skript laeuft im Origin der Scheduler-App. Es koennte sichtbare Planungsdaten auslesen, UI-Inhalte veraendern, API-Requests ausloesen oder den im Browserkontext vorhandenen Keycloak-Token aus `window.$keycloak.token` beziehungsweise dem Session-Store lesen.

- Sichere Gegenmassnahme

  Fuer alle `VueCal`-Instanzen eigene `event`-Slots verwenden und alle Werte ausschliesslich per Vue-Interpolation ausgeben. Keine HTML-Strings in `event.title` oder `event.content` speichern oder an die Bibliothek uebergeben.

- Konkreter Fix im Projekt

  Fuer `AssignmentView.vue` und `ReserveTimeslot.vue` jeweils einen sicheren Slot ergaenzen:

  ```diff
  - <VueCal
  -     :events="events"
  -     ...
  - ></VueCal>
  + <VueCal
  +     :events="events"
  +     ...
  + >
  +     <template #event="{ event }">
  +         <div class="vuecal__event-content">
  +             <strong>{{ event.title }}</strong>
  +             <div>{{ event.content }}</div>
  +         </div>
  +     </template>
  + </VueCal>
  ```

  Falls `event.content` aktuell zusammengesetzte Texte enthaelt, sollten diese als getrennte Felder modelliert werden, zum Beispiel `event.groupName`, `event.lectureCounter`, `event.status`, und im Slot einzeln per `{{ ... }}` gerendert werden.

- Demo-Idee: unsichere Variante vs. sichere Variante

  Unsichere Ist-Variante: `vue-cal` Default-Rendering fuehrt den gespeicherten Demo-Payload aus. Sichere Variante: eigener Event-Slot zeigt denselben gespeicherten Wert als Text an.

### Befund A: Eigene Vue-Ausgabe ist ueberwiegend sicher escaped

- Beschreibung

  Die zentralen eigenen Tabellen- und einige Kalenderausgaben verwenden Vue-Interpolation mit `{{ ... }}`. Vue escaped diese Werte standardmaessig als Text. Beispiele:

  - Standard-Tabellenzelle: `web/src/shared/components/fbTable/fbTableColumn.vue:102-106`
  - Link-Text in Tabellen: `web/src/shared/components/fbTable/fbTableColumn.vue:95-100`
  - Kalendertermin: `web/src/admin/views/data-admin/PlanningPeriodCrud/ScheduleOverview.vue:1008-1013`
  - Toast-/Notification-Text: `web/src/shared/components/fbNotificationBar.vue:44-53`

- Betroffene Dateien / Funktionen / Routen

  - `web/src/shared/components/fbTable/fbTableColumn.vue`
  - `web/src/shared/components/fbTable/fbArrayColumn.vue`
  - `web/src/shared/components/fbTable/fbLinkArrayColumn.vue`
  - `web/src/admin/views/data-admin/PlanningPeriodCrud/ScheduleOverview.vue`
  - Daten kommen unter anderem aus `/crud/{name}/search`, `/admin/schedule/...`, `/lecturers/schedule/...`

- Warum ist das problematisch?

  Diese konkrete Stelle ist aktuell nicht problematisch, solange Vue-Interpolation erhalten bleibt. Sie ist aber sicherheitskritisch, weil fast alle gespeicherten Freitextdaten durch diese Komponenten laufen. Eine spaetere Umstellung auf `v-html` oder eine eigene HTML-Renderfunktion wuerde sofort viele gespeicherte Felder riskant machen.

- Wie kann man es in der Demo ausnutzen?

  In der sicheren Variante gar nicht: Wird zum Beispiel ein Modulname mit HTML-Zeichen gespeichert, sollte der Browser die Zeichen sichtbar als Text anzeigen und nichts ausfuehren.

- Was koennte ein Angreifer theoretisch damit erreichen?

  Wenn diese Stelle unsicher auf HTML-Rendering geaendert wuerde, waere Stored XSS moeglich. Ein Payload in einem Modulnamen, Veranstaltungstitel, Gruppennamen oder einer Annotation wuerde beim Oeffnen der Tabelle oder des Kalenders im Browser eines anderen Nutzers laufen.

- Sichere Gegenmassnahme

  Weiterhin Vue-Interpolation nutzen. Keine Nutzereingaben mit `v-html`, `innerHTML` oder Bootstrap-HTML-Tooltips rendern.

- Konkreter Fix im Projekt

  Der aktuelle Code in `fbTableColumn.vue` ist fuer normale Textausgabe richtig:

  ```vue
  <template v-else>{{
      column.render !== undefined && column.render !== null && typeof column.render === 'function'
          ? column.render(fieldValue, record)
          : fieldValue
  }}</template>
  ```

- Demo-Idee: unsichere Variante vs. sichere Variante

  Sichere Variante: Payload als Text speichern und zeigen, dass nichts passiert. Unsichere Variante: lokal und nur `DEV_ONLY` diese zentrale Ausgabe mit `v-html` ersetzen und denselben gespeicherten Wert erneut anzeigen.

### Befund B: Stored-XSS-Risiko durch gespeicherte Freitextfelder, falls unsicher gerendert

- Beschreibung

  Das Projekt speichert viele Nutzertexte in der Datenbank. Beispiele sind Modulnamen, Modulbeschreibungen, Veranstaltungsnamen, Gruppennamen, Gruppeninfo, Dozentennamen, Titel und Kalender-Annotationen. Serverseitig gibt es Validierung auf Laenge und Typ, aber keine HTML-Sanitization. Das ist in Ordnung, solange die Ausgabe escaped erfolgt. Es ist riskant, wenn spaeter irgendwo HTML erlaubt oder mit `v-html` ausgegeben wird.

- Betroffene Dateien / Funktionen / Routen

  - Module:
    - Frontend: `web/src/admin/views/data-admin/ModuleCrudView.vue:67,100-106`
    - Backend: `api/src/main/java/com/frostbear/scheduler/shared/service/crud/ModuleCrud.java`
    - API: `POST /crud/module`, `PUT /crud/module/{id}`, `POST /crud/module/search`
  - Veranstaltungen:
    - Frontend: `web/src/admin/views/data-admin/LectureTemplateCrudView.vue:37,59-64`
    - Backend: `api/src/main/java/com/frostbear/scheduler/shared/service/crud/LectureTemplateCrud.java`
    - API: `POST /crud/lecturetemplate`, `PUT /crud/lecturetemplate/{id}`, `POST /crud/lecturetemplate/search`
  - Gruppen:
    - Frontend: `web/src/admin/views/data-admin/GroupCrudView.vue:35-56,132-136,206-210`
    - Backend: `api/src/main/java/com/frostbear/scheduler/shared/service/crud/GroupCrud.java`
    - API: `POST /crud/group`, `PUT /crud/group/{id}`, `POST /crud/group/search`
  - Scheduler-Annotationen:
    - Frontend: `web/src/admin/views/data-admin/PlanningPeriodCrud/ScheduleOverview.vue:647-650,1013-1022`
    - Backend: `api/src/main/java/com/frostbear/scheduler/admin/controller/ScheduleController.java:92-95`
    - DTO: `api/src/main/java/com/frostbear/scheduler/admin/command/AnnotateCommand.java:8-11`
    - API: `POST /admin/schedule/annotate/{schedule_id}`

- Warum ist das problematisch?

  Stored XSS ist besonders gut fuer eine Demo geeignet: Der Wert wird einmal gespeichert und spaeter an anderer Stelle wieder ausgegeben. Wenn diese Ausgabe unsicher ist, wird der Payload nicht nur beim Erstellen, sondern bei jedem spaeteren Anzeigen aktiviert.

- Wie kann man es in der Demo ausnutzen?

  Lokal eine Demo-Eingabe in einem Freitextfeld speichern, zum Beispiel im Modulnamen oder in der Modulbeschreibung. In der aktuellen sicheren Variante wird der Wert als Text dargestellt. In der `DEV_ONLY` unsicheren Variante wird er als HTML ausgefuehrt.

- Was koennte ein Angreifer theoretisch damit erreichen?

  Im Browser eines eingeloggten Admins koennte ein Skript sichtbare Planungsdaten auslesen, die UI manipulieren, API-Requests mit dem vorhandenen Bearer-Token ausloesen oder den Token aus `window.$keycloak.token` bzw. `session.state.token` lesen. In der Demo wird nur eine harmlose lokale Meldung gezeigt.

- Sichere Gegenmassnahme

  HTML aus diesen Feldern nicht erlauben. Felder serverseitig auf sinnvolle Laenge begrenzen und clientseitig nur fuer UX validieren. Beim Rendern konsequent escaped Text verwenden.

- Konkreter Fix im Projekt

  Fuer `groupInfo` und Annotationen fehlen beziehungsweise wirken keine klaren Laengenregeln. Empfehlenswert:

  ```diff
  // api/src/main/java/com/frostbear/scheduler/shared/service/crud/GroupCrud.java
   public static class GroupPayload {
       ...
+      @Length(max = 2000)
       public String groupInfo;
   }
  ```

  ```diff
  // api/src/main/java/com/frostbear/scheduler/admin/command/AnnotateCommand.java
+ import org.hibernate.validator.constraints.Length;

   public class AnnotateCommand {
+      @Length(max = 500)
       protected String name;
   }
  ```

  Damit ist XSS nicht allein geloest, aber die gespeicherte Payload-Groesse wird reduziert. Die eigentliche XSS-Grenze bleibt Output Escaping im Frontend.

- Demo-Idee: unsichere Variante vs. sichere Variante

  Unsicher: gespeicherte Werte mit `v-html` rendern. Sicher: denselben Wert mit `{{ ... }}` rendern und zeigen, dass die Zeichenfolge sichtbar bleibt.

### Befund C: `DEV_ONLY` Demo-Schwachstelle ueber zentrale Tabellenzelle

- Beschreibung

  Fuer die Praesentation kann lokal eine absichtlich unsichere Variante in der generischen Tabellenzelle eingebaut werden. Dadurch reicht ein einzelnes Freitextfeld, zum Beispiel `Module > Modulname`, um Stored XSS sichtbar zu machen.

- Betroffene Dateien / Funktionen / Routen

  - Datei: `web/src/shared/components/fbTable/fbTableColumn.vue`
  - Sichere Ausgabe: `fbTableColumn.vue:102-106`
  - Datenquelle fuer Demo: `POST /crud/module`, Anzeige ueber `POST /crud/module/search`
  - Demo-Seite: `/admin/data-admin/lecturetemplates` oder, falls die Modulroute wieder aktiviert wird, `/admin/data-admin/modules`

- Warum ist das problematisch?

  `v-html` interpretiert den String als HTML. Wenn der String aus der Datenbank kommt, wird gespeicherter Nutzerinhalt zu aktivem DOM.

- Wie kann man es in der Demo ausnutzen?

  Nur lokal, nur fuer die Demo:

  ```diff
  <!-- web/src/shared/components/fbTable/fbTableColumn.vue -->
  - <template v-else>{{
  -     column.render !== undefined && column.render !== null && typeof column.render === 'function'
  -         ? column.render(fieldValue, record)
  -         : fieldValue
  - }}</template>
  + <!-- DEV_ONLY_XSS_DEMO: unsicher, nur lokal fuer die Praesentation -->
  + <span
  +     v-else
  +     v-html="
  +         column.render !== undefined && column.render !== null && typeof column.render === 'function'
  +             ? column.render(fieldValue, record)
  +             : fieldValue
  +     "
  + />
  ```

  Danach einen harmlosen lokalen Payload als Veranstaltungsname oder Modulname speichern:

  ```html
  XSS Demo <img src=x onerror="alert('XSS Demo lokal: gespeicherter Wert wurde als HTML ausgefuehrt')">
  ```

- Was koennte ein Angreifer theoretisch damit erreichen?

  Statt `alert(...)` koennte ein Skript sichtbare Daten aus der Seite lesen, lokale API-Requests ausloesen oder den im JS-Kontext vorhandenen Keycloak-Token auslesen. Fuer die Demo werden keine externen Endpunkte und keine echten Daten verwendet.

- Sichere Gegenmassnahme

  Diese `v-html`-Aenderung nicht committen. Zentrale Tabellenzellen und Kalenderinhalte weiter mit Vue-Interpolation rendern.

- Konkreter Fix im Projekt

  Nach der Demo die sichere Variante wiederherstellen:

  ```diff
  - <span v-else v-html="..." />
  + <template v-else>{{
  +     column.render !== undefined && column.render !== null && typeof column.render === 'function'
  +         ? column.render(fieldValue, record)
  +         : fieldValue
  + }}</template>
  ```

- Demo-Idee: unsichere Variante vs. sichere Variante

  Erst unsicher zeigen: Meldung erscheint. Dann Fix zurueck auf `{{ ... }}` zeigen: Derselbe gespeicherte Wert erscheint als Text, inklusive `<img ...>`, ohne Ausfuehrung.

### Befund D: Kalender-Annotation als alternative Stored-XSS-Demo

- Beschreibung

  Kalendertermine bauen Event-Daten aus API-Antworten und zeigen `event.title`, `event.content`, `event.dozent` und `event.annotation` an. Aktuell ist die Ausgabe escaped.

- Betroffene Dateien / Funktionen / Routen

  - Event-Aufbau: `web/src/admin/views/data-admin/PlanningPeriodCrud/ScheduleOverview.vue:280-290`
  - Annotation speichern: `ScheduleOverview.vue:647-650`
  - Annotation anzeigen: `ScheduleOverview.vue:1010-1013`
  - Backend: `api/src/main/java/com/frostbear/scheduler/admin/controller/ScheduleController.java:92-95`
  - DTO: `api/src/main/java/com/frostbear/scheduler/admin/command/AnnotateCommand.java:8-11`
  - API: `POST /admin/schedule/annotate/{schedule_id}`

- Warum ist das problematisch?

  Annotationen sind ein klassisches Stored-XSS-Feld: Ein Admin kann Text an einem Termin speichern, ein anderer Admin sieht spaeter denselben Termin. Aktuell verhindert Vue-Escaping die Ausfuehrung.

- Wie kann man es in der Demo ausnutzen?

  `DEV_ONLY` die Anzeige der Annotation unsicher machen:

  ```diff
  - <div v-if="event.annotation && !clickedAnnotate">{{ event.annotation }}</div>
  + <!-- DEV_ONLY_XSS_DEMO: unsicher -->
  + <div v-if="event.annotation && !clickedAnnotate" v-html="event.annotation"></div>
  ```

  Harmloser lokaler Testwert:

  ```html
  <strong>Demo-Hinweis</strong><img src=x onerror="console.log('XSS Demo lokal: Annotation ausgefuehrt')">
  ```

- Was koennte ein Angreifer theoretisch damit erreichen?

  Ein gespeicherter Termintext koennte beim Oeffnen des Planungszeitraums im Browser anderer Nutzer laufen und API-Aktionen im Kontext dieser Sitzung ausfuehren.

- Sichere Gegenmassnahme

  Annotationen als Text anzeigen. Wenn Formatierung wirklich noetig ist, eine kleine erlaubte Markup-Sprache oder Sanitizer mit strenger Allowlist einsetzen.

- Konkreter Fix im Projekt

  Sichere Anzeige beibehalten:

  ```vue
  <div v-if="event.annotation && !clickedAnnotate">{{ event.annotation }}</div>
  ```

  Zusaetzlich DTO validieren:

  ```diff
  +import org.hibernate.validator.constraints.Length;

   public class AnnotateCommand {
  +    @Length(max = 500)
       protected String name;
   }
  ```

- Demo-Idee: unsichere Variante vs. sichere Variante

  Unsicher: `v-html` auf Annotation, Payload wird ausgefuehrt. Sicher: Interpolation, Payload bleibt Text.

### Befund E: HTML-E-Mail-Templates sind kein Browser-XSS, aber HTML-Injection-relevant

- Beschreibung

  Das Backend verwendet HTML-Mail-Templates unter `api/src/main/resources/emailTemplates/*`. Werte wie Namen und Planungsperioden werden in mehreren `*Mail.java`-Klassen mit `.formatted(...)` in HTML eingefuegt, zum Beispiel `AccountCreatedMail.java:29-34` oder `PlanningStartedMail.java:30-40`.

- Betroffene Dateien / Funktionen / Routen

  - `api/src/main/resources/emailTemplates/*.html`
  - `api/src/main/java/com/frostbear/scheduler/mail/*Mail.java`
  - `api/src/main/java/com/frostbear/scheduler/mail/service/impl/EmailService.java`

- Warum ist das problematisch?

  Das ist nicht direkt XSS in der Webanwendung, weil es nicht im Scheduler-Browserkontext laeuft. Trotzdem koennen unescaped Werte HTML in E-Mails beeinflussen, wenn Namen oder Periodentitel aus Nutzereingaben stammen.

- Wie kann man es in der Demo ausnutzen?

  Fuer die XSS-Demo nicht empfohlen, weil E-Mail-Clients Skripte oft blockieren und das Thema vom Web-XSS ablenkt. Als Hinweis kann man zeigen, dass HTML-Templates ebenfalls Encoding brauchen.

- Was koennte ein Angreifer theoretisch damit erreichen?

  HTML-Injection in E-Mails, optische Manipulation, Phishing-artige Inhalte in einer Benachrichtigung. JavaScript-Ausfuehrung ist in modernen Mailclients meist blockiert.

- Sichere Gegenmassnahme

  Werte vor Einsetzen in HTML-Kontext encoden, zum Beispiel mit Apache Commons Text `StringEscapeUtils.escapeHtml4(...)`, oder eine Template-Engine mit automatischem HTML-Escaping verwenden.

- Konkreter Fix im Projekt

  Beispielhaft:

  ```diff
  +import org.apache.commons.text.StringEscapeUtils;

  -var fullName = "%s %s".formatted(parameters.getFirstName(), parameters.getLastName());
  +var fullName = StringEscapeUtils.escapeHtml4(
  +    "%s %s".formatted(parameters.getFirstName(), parameters.getLastName())
  +);
  ```

- Demo-Idee: unsichere Variante vs. sichere Variante

  Nicht als Hauptdemo verwenden. Nur als Transferfrage: "Auch ausserhalb der Vue-UI muessen HTML-Kontexte escaped werden."

### Befund F: Keine erkennbare CSP oder Security-Header als XSS-Zusatzschutz

- Beschreibung

  Im aktuellen Projekt wurde keine Content Security Policy und keine explizite Security-Header-Konfiguration gefunden. Das Backend setzt in `SecurityConfiguration.java` keine Header-Policy, `web/index.html` enthaelt keine CSP-Meta-Policy, und `nginx.conf` liefert nur statische Dateien aus.

- Betroffene Dateien / Funktionen / Routen

  - `api/src/main/java/com/frostbear/scheduler/conf/SecurityConfiguration.java:35-62`
  - `web/index.html:18-30`
  - `nginx.conf:1-16`
  - Relevant fuer alle Browserseiten der Scheduler-App

- Warum ist das problematisch?

  CSP behebt keine XSS-Ursache, begrenzt aber die Wirkung. Ohne CSP kann eine erfolgreiche XSS-Stelle leichter Inline-JavaScript, Event-Handler und externe Ressourcen verwenden. Im Projekt ist das besonders wichtig, weil der Keycloak-Token im JavaScript-Kontext verfuegbar ist.

- Wie kann man es in der Demo ausnutzen?

  Bei der lokalen `vue-cal`-Demo laeuft der harmlose Demo-Payload ohne zusaetzliche Browser-Blockade. Mit einer wirksamen CSP ohne `unsafe-inline` wuerde ein Inline-Event-Handler wie `onerror="..."` zusaetzlich blockiert.

- Was koennte ein Angreifer theoretisch damit erreichen?

  Eine echte XSS koennte Requests im Browser des eingeloggten Nutzers ausloesen, sichtbare Daten lesen oder den JS-verfuegbaren Token angreifen. Eine CSP wuerde das nicht vollstaendig verhindern, aber viele einfache Payloads und externe Nachladeversuche stoppen.

- Sichere Gegenmassnahme

  CSP und weitere Security-Header im Reverse Proxy oder Backend setzen. Fuer die Produktion sollten mindestens `default-src 'self'`, eingeschraenkte `script-src`, `object-src 'none'`, `base-uri 'self'` und `frame-ancestors 'none'` geprueft werden.

- Konkreter Fix im Projekt

  Fuer nginx als statische Frontend-Auslieferung:

  ```nginx
  add_header Content-Security-Policy "default-src 'self'; object-src 'none'; base-uri 'self'; frame-ancestors 'none'; script-src 'self' http://localhost:8080; connect-src 'self' http://localhost:8079 http://localhost:8080" always;
  add_header X-Content-Type-Options "nosniff" always;
  add_header Referrer-Policy "strict-origin-when-cross-origin" always;
  ```

  Fuer Production muessen die `localhost`-Quellen durch echte erlaubte Origins ersetzt und die Policy mit Keycloak getestet werden.

- Demo-Idee: unsichere Variante vs. sichere Variante

  Unsicher: keine CSP, Demo-Payload laeuft in der `vue-cal`-Ansicht. Sicher: Event-Slot-Fix verhindert die Ursache; CSP blockiert zusaetzlich einfache Inline-Payloads als zweite Schutzschicht.

## 3. XSS-Demo fuer Arvid

### Demo-Ziel

Die Demo zeigt Stored XSS lokal im Scheduler: Ein gespeicherter Wert wird zuerst unsicher als HTML interpretiert und nach dem Fix wieder harmlos als Text angezeigt.

### Vorbereitung

1. Lokale Infrastruktur starten:

   ```powershell
   docker-compose up
   ```

2. Backend im Profil `dev` starten:

   ```powershell
   cd api
   .\gradlew bootRun --args='--spring.profiles.active=dev'
   ```

   Alternativ wie im README: `ApiApplication` in IntelliJ mit Profil `dev` starten.

3. Frontend starten:

   ```powershell
   cd web
   npm install
   npm run dev
   ```

4. Login im Browser unter `http://localhost:8078`:

   - Admin + Teacher: `rainerzufall` / `password`
   - Admin only: `annaadmin` / `password`
   - Teacher only: `damiandozent` / `password`

### Empfohlenes Szenario 1: echte `vue-cal`-Schwachstelle

Dieses Szenario benoetigt lokale Kalenderdaten, in denen der manipulierte Wert als Event-Titel oder Event-Inhalt erscheint. Es zeigt die echte Schwachstelle ohne absichtliche Code-Aenderung.

Sichere lokale Beispiel-Payload:

```html
XSS Demo <img src=x onerror="alert('XSS Demo lokal ueber vue-cal')">
```

Moegliche Datenquellen:

- Veranstaltungsname: `/admin/data-admin/lecturetemplates`
- Gruppenname oder Gruppeninfo: `/admin/data-admin/groups`
- Reservierungslabel: `/admin/data-admin/planningperiods/:id/reserve-timeslots/:idGroup`

Schritte:

1. Den Demo-Wert lokal in einem passenden Feld speichern.
2. Die betroffene Kalenderansicht oeffnen:
   - Teacher Assignment Calendar: `web/src/teacher/views/AssignmentView.vue`
   - Admin Reserve Timeslot Calendar: `web/src/admin/views/data-admin/PlanningPeriodCrud/ReserveTimeslot.vue`
3. Browser-Verhalten beobachten.
4. Danach den sicheren Event-Slot aus Befund 0 einbauen.
5. Dieselbe Ansicht erneut oeffnen.

Erwartetes Verhalten vor dem Fix:

- `vue-cal` schreibt `event.title` oder `event.content` per `innerHTML` ins DOM.
- Die lokale Demo-Meldung erscheint.

Erwartetes Verhalten nach dem Fix:

- Derselbe gespeicherte Wert erscheint als Text.
- Keine JavaScript-Ausfuehrung.

Code zum Zeigen:

- `web/node_modules/vue-cal/dist/vue-cal.es.js:962`
- `web/src/teacher/views/AssignmentView.vue:106-116`
- `web/src/admin/views/data-admin/PlanningPeriodCrud/ReserveTimeslot.vue:346-351,398-402,504-522`
- `web/src/admin/views/data-admin/PlanningPeriodCrud/EditIncompleteStateAvailabilities.vue:99-109,458-477`
- Sicheres Gegenbeispiel: `web/src/admin/views/data-admin/PlanningPeriodCrud/ScheduleOverview.vue:1008-1013`

### Backup-Szenario 2: Veranstaltungsname in der Tabelle mit `DEV_ONLY`-Patch

Falls die Kalenderdaten fuer die Live-Demo schwer vorzubereiten sind, kann die zentrale Tabellenzelle lokal absichtlich unsicher gemacht werden. Falls die Modulroute in `web/src/router.js` auskommentiert bleibt, ist `Veranstaltungen` einfacher zu erreichen: `/admin/data-admin/lecturetemplates`.

Sichere Eingabe zum Testen:

```html
XSS Demo <img src=x onerror="alert('XSS Demo lokal')">
```

#### Variante 1: sicherer Tabellenzustand

1. Im Frontend als Admin einloggen.
2. Zu `Veranstaltungen` wechseln.
3. Neue Veranstaltung mit dem obigen Wert als `Veranstaltungsname` anlegen.
4. Tabelle aktualisieren oder Suche ausloesen.

Erwartetes Verhalten:

- Der String wird sichtbar angezeigt.
- Die Zeichen `<img ...>` werden nicht als Bild/Script ausgefuehrt.
- Kein Alert erscheint.

Code zum Zeigen:

- `web/src/admin/views/data-admin/LectureTemplateCrudView.vue:37,59-64`
- `web/src/shared/components/fbTable/fbTableColumn.vue:102-106`
- `api/src/main/java/com/frostbear/scheduler/shared/controller/CrudController.java:115-128`

#### Variante 2: unsichere `DEV_ONLY`-Version

1. Lokal `web/src/shared/components/fbTable/fbTableColumn.vue` wie in Befund C auf `v-html` aendern.
2. Frontend neu laden.
3. Dieselbe Tabelle oeffnen.

Erwartetes Verhalten:

- Browser interpretiert die gespeicherte Zeichenfolge als HTML.
- Der lokale Alert erscheint.
- In DevTools sieht man, dass HTML im DOM gelandet ist.

#### Variante 3: Fix zeigen

1. `v-html` wieder entfernen.
2. Die sichere `{{ ... }}`-Ausgabe wiederherstellen.
3. Frontend neu laden.
4. Tabelle erneut oeffnen.

Erwartetes Verhalten:

- Derselbe gespeicherte Payload ist noch in der Datenbank.
- Er wird aber nur als Text angezeigt.
- Keine JavaScript-Ausfuehrung.

### Alternative Demo: Kalender-Annotation

Diese Demo passt besonders gut, wenn im lokalen Datenbestand bereits ein Planungszeitraum mit Terminen existiert.

1. In `ScheduleOverview.vue` `event.annotation` nur lokal mit `v-html` rendern.
2. In einem Termin eine Annotation mit Demo-Payload speichern.
3. Terminansicht erneut oeffnen.
4. Danach `v-html` wieder entfernen.

Beispielwert:

```html
<strong>Lokale XSS-Demo</strong><img src=x onerror="console.log('XSS Demo: Annotation')">
```

Erwartung vor dem Fix:

- Bei unsicherer `v-html`-Variante wird HTML interpretiert.

Erwartung nach dem Fix:

- Der komplette String wird als Text angezeigt.

## 4. XSS-Schutzmassnahmen

### Output Encoding / Escaping

Alle Werte aus Datenbank, API, URL, Storage und Formularen muessen im Zielkontext escaped werden. Fuer HTML-Text ist Vue-Interpolation (`{{ value }}`) richtig. Fuer Attribute sind Vue-Bindings in der Regel sicherer als Stringverkettung, muessen aber trotzdem auf erlaubte Protokolle oder Zieltypen begrenzt werden.

### Kein direktes Rendern von HTML aus Benutzereingaben

Im Scheduler sollten Freitextfelder wie `name`, `comment`, `groupInfo`, `annotation`, `lectureInfo`, `firstName`, `lastName`, `title` keine HTML-Felder sein. Es gibt keinen Produktgrund, daraus aktives HTML zu machen.

### Sichere Template-Mechanismen verwenden

Vue-Komponenten sollen Text mit `{{ ... }}` oder Props anzeigen. Backend-HTML-E-Mails sollten eine Template-Engine mit Auto-Escaping oder explizites HTML-Encoding verwenden.

### `innerHTML`, `v-html`, `dangerouslySetInnerHTML` vermeiden

Im gefundenen Vue-Code gibt es keine direkte Nutzung von `innerHTML` oder `v-html`. Das sollte so bleiben. Falls eine Komponente HTML zwingend braucht, muss sie als Ausnahme dokumentiert und mit Sanitizing abgesichert werden.

### Sanitizing nur wenn HTML wirklich erlaubt sein muss

Wenn in Zukunft formatierte Beschreibungen erlaubt werden, nicht selbst mit Regex filtern. Besser eine gepflegte Bibliothek wie DOMPurify nutzen und eine enge Allowlist definieren:

```js
import DOMPurify from 'dompurify';

const cleanHtml = DOMPurify.sanitize(userInput, {
  ALLOWED_TAGS: ['b', 'strong', 'i', 'em', 'br', 'ul', 'ol', 'li'],
  ALLOWED_ATTR: []
});
```

### Content Security Policy als Zusatzschicht

CSP verhindert nicht die Ursache, reduziert aber die Wirkung vieler XSS-Fehler. Fuer das Projekt muss Keycloak beachtet werden, weil `web/src/App.vue:110-115` das Keycloak-SDK von `http://localhost:8080/js/keycloak.js` laedt.

Beispiel fuer eine produktionsnaehere Spring-Security-Konfiguration:

```java
http.headers(headers -> headers.contentSecurityPolicy(csp -> csp.policyDirectives(
    "default-src 'self'; " +
    "script-src 'self' http://localhost:8080; " +
    "connect-src 'self' http://localhost:8080; " +
    "style-src 'self' 'unsafe-inline'; " +
    "img-src 'self' data:; " +
    "object-src 'none'; " +
    "base-uri 'self'; " +
    "frame-ancestors 'none'"
)));
```

Fuer Vite-Development kann eine lockerere Dev-CSP noetig sein. Fuer Production sollte die CSP enger sein und keine unnoetigen externen Quellen erlauben.

### Cookie-Flags

Die Scheduler-API arbeitet aktuell nicht mit eigener serverseitiger Session-Cookie-Authentifizierung, sondern mit Bearer-JWTs. Keycloak nutzt eigene Cookies auf `localhost:8080`. Falls die Scheduler-API spaeter Cookies verwendet:

- `HttpOnly`: verhindert Zugriff per JavaScript.
- `Secure`: nur ueber HTTPS.
- `SameSite=Lax` oder `Strict`: reduziert CSRF-Risiko.

### Tokens nicht unnoetig in `localStorage` speichern

Der aktuelle Token wird in Vuex-State gehalten und bei Keycloak aktualisiert. `web/src/shared/PersistedStorage.js` kapselt `localStorage`, wird aber nicht als Token-Speicher verwendet. Das ist besser als dauerhafte Token in `localStorage`, denn XSS koennte `localStorage` direkt auslesen.

Trotzdem gilt: Jeder XSS-Code kann im laufenden Browserkontext auf `window.$keycloak.token` und `session.state.token` zugreifen. Deshalb ist XSS-Schutz hier besonders wichtig.

### Serverseitige Validierung

Clientseitige Validierung in `web/src/shared/validation.js` ist nur UX-Hilfe. Sicherheitsrelevant ist die serverseitige Validierung in DTOs, zum Beispiel `@Length` in CRUD-Payloads. Fehlende Laengenbegrenzungen sollten ergaenzt werden.

Konkrete Projekt-Fixes:

```diff
// api/src/main/java/com/frostbear/scheduler/shared/service/crud/GroupCrud.java
+import org.hibernate.validator.constraints.Length;

 public static class GroupPayload {
     ...
+    @Length(max = 2000)
     public String groupInfo;
 }
```

```diff
// api/src/main/java/com/frostbear/scheduler/admin/command/AnnotateCommand.java
+import org.hibernate.validator.constraints.Length;

 public class AnnotateCommand {
+    @Length(max = 500)
     protected String name;
 }
```

### Clientseitige Validierung nur als UX-Hilfe

Die Frontend-Definitionen in `ModuleCrudView.vue`, `GroupCrudView.vue`, `LectureTemplateCrudView.vue` und `TeacherCrudView.vue` sind hilfreich, aber nicht ausreichend. Ein Angreifer kann die API direkt aufrufen. Deshalb muessen alle relevanten Regeln im Backend gespiegelt sein.

## 5. Praesentationsstruktur fuer XSS

### Linus: Einstieg und Grundlagen

- Was ist XSS?
- Stored, Reflected und DOM-based XSS unterscheiden.
- Warum XSS im Scheduler-Kontext kritisch waere: eingeloggter Nutzer, API-Token, sichtbare Planungsdaten.
- Typische Ursache zeigen: aus Text wird HTML.

### Arvid: Live-Demo unsicher

- Hauptdemo: sicheren Payload als Kalenderwert speichern, der in `vue-cal` als Event-Titel oder Event-Inhalt erscheint.
- Betroffene Ansicht oeffnen, zum Beispiel Teacher-Kalender oder Reserve-Timeslot-Kalender.
- Zeigen: `vue-cal` Default-Rendering interpretiert den gespeicherten Wert als HTML.
- Lokale Demo-Meldung erscheint.
- Backup, falls Kalenderdaten schwer vorzubereiten sind: `DEV_ONLY` Tabellen-Demo mit `v-html`.

### Arvid: Ursache im Code erklaeren

- Unsichere echte Stelle: `vue-cal` Default-Rendering in `web/node_modules/vue-cal/dist/vue-cal.es.js:962` nutzt `innerHTML`.
- Datenfluss: gespeicherter Name oder Inhalt -> API -> Kalender-Event -> `VueCal`.
- Projektstellen: `AssignmentView.vue:106-116`, `ReserveTimeslot.vue:346-351,398-402,504-522` und `EditIncompleteStateAvailabilities.vue:99-109,458-477`.
- Sichere Gegenbeispiele: Vue-Interpolation in Tabellen und eigener `VueCal`-Slot in `ScheduleOverview.vue:1008-1013`.

### Arvid: Fix zeigen

- In allen `VueCal`-Ansichten mit Datenbankwerten eigenen `#event`-Slot verwenden.
- Event-Titel und Event-Inhalte mit `{{ event.title }}` und `{{ event.content }}` rendern.
- Keine HTML-Ausgabe aus Scheduler-Freitextfeldern erlauben, solange es keinen expliziten Sanitizer gibt.
- Optional serverseitige Laengenvalidierung fuer Namen, Annotationen und `groupInfo` zeigen.

### Arvid: sichere Variante demonstrieren

- Seite neu laden.
- Derselbe Payload bleibt gespeichert.
- Er wird sichtbar als Text angezeigt.
- Kein Script laeuft.
