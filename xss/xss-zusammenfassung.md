# XSS-Zusammenfassung fuer die Praesentation

Stand nach Zweitpruefung des aktuellen Checkouts: Das Projekt ist nicht vollstaendig sicher. Die relevante echte XSS-Stelle liegt nicht im eigenen Vue-Code, sondern in `vue-cal`.

Pruefumfang: getrackte Source-/Konfigurationsdateien aus Backend, Frontend, Docker, Nginx und Keycloak-Realm; Build-/DB-Artefakte wurden nicht als Anwendungscode bewertet. `node_modules` wurde nur fuer die konkrete `vue-cal`-Stelle geprueft.

## Kernaussage

- Echte Schwachstelle: Stored DOM-XSS ueber `vue-cal` Default-Rendering.
- Im eigenen `web/src` wurde keine direkte Nutzung von `v-html`, `innerHTML`, `insertAdjacentHTML`, `eval` oder `dangerouslySetInnerHTML` gefunden.
- Vue-Interpolation in Tabellen, Notifications und eigenen Slots ist ueberwiegend sicher.
- Risiko bleibt, weil gespeicherte Scheduler-Daten an `vue-cal` gehen und dort per `innerHTML` gerendert werden.
- Es gibt keine erkennbare CSP in `web/index.html`, `nginx.conf` oder `SecurityConfiguration.java`.

## Wichtigste Dateien

- Bibliothek: `web/node_modules/vue-cal/dist/vue-cal.es.js:962`
- Betroffen: `web/src/teacher/views/AssignmentView.vue:106-116,769-781`
- Betroffen: `web/src/admin/views/data-admin/PlanningPeriodCrud/ReserveTimeslot.vue:346-351,398-402,504-516`
- Betroffen: `web/src/admin/views/data-admin/PlanningPeriodCrud/EditIncompleteStateAvailabilities.vue:99-109,458-468`
- Riskantes Pattern: `web/src/teacher/views/TimeSlotCalendarView.vue:895-900,1095-1106`
- Sicheres Gegenbeispiel: `web/src/admin/views/data-admin/PlanningPeriodCrud/ScheduleOverview.vue:1008-1013`
- Token im JS-Kontext: `web/src/App.vue:118-124`, `web/src/shared/http-client.js:155-169`

## Demo-Payload

Nur lokal verwenden:

```html
XSS Demo <img src=x onerror="alert('XSS Demo lokal ueber vue-cal')">
```

## Demo-Ablauf fuer Arvid

1. Manipulierten Namen oder Inhalt speichern, der in einer `vue-cal`-Ansicht erscheint.
2. Betroffene Kalenderansicht oeffnen.
3. Zeigen: `vue-cal` interpretiert `event.title` oder `event.content` als HTML.
4. Ursache im Code zeigen: Bibliothek nutzt `innerHTML`, Projekt uebergibt gespeicherte Werte.
5. Fix zeigen: eigener `#event`-Slot mit Vue-Interpolation.
6. Dieselbe Eingabe erneut anzeigen: Payload bleibt Text, kein Script laeuft.

## Konkreter Fix

Alle `VueCal`-Instanzen mit Datenbankwerten bekommen einen eigenen Slot:

```vue
<VueCal :events="events">
  <template #event="{ event }">
    <div class="vuecal__event-content">
      <strong>{{ event.title }}</strong>
      <div>{{ event.content }}</div>
    </div>
  </template>
</VueCal>
```

Zusaetzlich:

- Kein HTML in `title` oder `content` speichern.
- Statische Icons nicht als HTML-String speichern, sondern im Slot anhand von Statusfeldern rendern.
- Serverseitige Laengenvalidierung fuer Freitextfelder ergaenzen.
- CSP im Reverse Proxy oder Backend setzen.
- Token nicht unnoetig im globalen JS-Kontext sichtbar machen.

## Praesentationssatz

XSS entsteht hier nicht, weil Vue unsicher waere, sondern weil eine Kalenderbibliothek gespeicherte Daten als HTML rendert. Der sichere Fix ist nicht ein anderer Payload-Filter, sondern Text-Rendering ueber eigene Slots plus CSP als Zusatzschutz.
