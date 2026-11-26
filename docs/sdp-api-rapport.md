# ServiceDesk Plus (On-Premise) – förmåga att hantera ärenden mot leverantörer via API

**Organisation:** Organisation A – IT-avdelningens service desk
**Produkt:** ManageEngine ServiceDesk Plus (SDP), On-Premise, Enterprise Edition
**Datum:** 2026-11-17
**Status:** Utkast för granskning

---

## 1. Sammanfattning

ServiceDesk Plus On-Premise har ett REST-API (**v3**) som täcker de centrala objekten i ett ärende: själva ärendet, anteckningar, uppgifter, bilagor, lösning, avslut, tilldelning och sammanslagning. SDP har också flera mekanismer för **utgående** anrop: *Custom Triggers*, *Webhooks*, *Custom Functions* i skriptspråket Deluge, *Execute Script* och *Custom Schedules*. Med dem kan SDP skapa och uppdatera ärenden i en leverantörs ärendehanteringssystem.

SDP saknar däremot en **inbyggd funktion för ärendeintegration mellan organisationer** (e-bonding eller ticket federation). Varje leverantörskoppling måste byggas som skräddarsydd logik:

- mappning av fält och status
- lagring av extern referens
- synkronisering åt båda håll
- felhantering och omförsök.

Logiken byggs antingen i SDP (Deluge eller skript) eller i ett mellanlager (iPaaS/integrationsplattform).

**De viktigaste bristerna:**

1. **Leverantörsärendet är inget förstaklassobjekt.** SDP har inget fält eller objekt för "externt ärende hos leverantör X med status Y". Det måste modelleras med egna fält (UDF), uppgifter eller anteckningar.
2. **Custom Triggers reagerar inte på anteckningar eller konversationer.** Att ett ärende skapas eller redigeras kan trigga synkronisering, men inte en ny anteckning eller ett nytt e-postsvar. Kommentarssynk i realtid kräver därför schemalagd polling eller manuella steg.
3. **Webhooks saknar dokumenterade omförsök, kö och leveransgaranti.** Ett misslyckat anrop till leverantören kan gå förlorat om integrationen inte själv hanterar fel.
4. **Leverantörsavtal (underpinning contracts) kan inte kopplas till ärendens SLA.** Modulen *Contracts* hanterar avtal kopplade till tillgångar och leverantörer, men leverantörens åtagande (till exempel en svarstid på 4 timmar) kan inte mätas automatiskt per ärende.
5. **Inkommande uppdateringar kräver att leverantören anropar SDP:s API.** Det förutsätter att on-premise-servern exponeras utåt, eller ett mellanlager. Autentiseringen sker med en teknikerbunden *authtoken* utan OAuth-scopes, vilket ger grov behörighetsstyrning.
6. **Den publika on-premise-dokumentationen för v3-API:t är ofullständig.** Flera operationer är dokumenterade för molnversionen men inte för on-premise, bland annat worklogs, länkning av ärenden, kontrakt och leverantörer. Dokumentationen motsäger sig ibland själv, till exempel kring länkning av ärenden via API. Allt måste därför verifieras mot den egna instansens version.

**Rekommendation:** Bygg leverantörsintegrationen enligt mönstret *"SDP som master + integrationslager"*. SDP skickar händelser utåt (Webhook eller Custom Function), och ett integrationslager översätter till respektive leverantörs API eller e-post och skriver tillbaka via SDP:s REST-API. För leverantörer utan API används e-post med strikt ärende-ID i ämnesraden. Se avsnitt 7.

---

## 2. Bakgrund, avgränsning och källor

### 2.1 Scenariot

Organisation A:s verksamhet anmäler ärenden till IT-avdelningens service desk. När IT behöver hjälp av en leverantör för att åtgärda hela eller delar av problemet ska service desk kunna **skapa och uppdatera ärenden i leverantörens ärendehanteringssystem** och få tillbaka information om leverantörens arbete.

Typiska leverantörskategorier:

| Kategori | Exempel på ärenden |
|---|---|
| **Hårdvara** (servrar, laptops, plattor inkl. OS och Office) | Hårdvarufel, garantiärende eller RMA, utbyte, OS-image, licensproblem |
| **Systemleverantörer** | Buggar, incidenter i verksamhetssystem, ändringsbeställningar, konfigurationsfrågor |
| **Telefoni** | Driftstörning, nummerflytt, abonnemangsändringar, hårdvara som telefoner och växel |

### 2.2 Avgränsning

- Rapporten gäller **SDP On-Premise, Enterprise Edition** och **REST API v3**. Det äldre servlet-API:t (`/sdpapi/...`) stöds enligt ManageEngine inte längre och tas inte upp ([REST API – help.servicedeskplus.com](https://help.servicedeskplus.com/api/rest-api.html)).
- Molnversionen (SDP Cloud / On-Demand) har ett delvis annorlunda och mer omfattande v3-API. Molndokumentation refereras bara där on-premise-dokumentationen saknas, och det markeras då uttryckligen.
- Rapporten beskriver förmågor och brister, inte en färdig lösningsdesign.

### 2.3 Om källorna

- **Primär källa:** ManageEngines publika API-dokumentation för On-Premise v3,
  `https://www.manageengine.com/products/service-desk/sdpop-v3-api/` (nedan kallad **[OP-API]**). Den äldre adressen `ui.servicedeskplus.com/APIDocs3/` vidarebefordras dit.
- **Sekundär källa:** SDP:s hjälpdokumentation för on-premise, `https://help.servicedeskplus.com/` (nedan **[HELP]**).
- **Kompletterande källa:** ManageEngines dokumentation för SDP MSP och SDP Cloud. Den används där on-premise-dokumentationen saknas och markeras **[MSP]** respektive **[CLOUD]**.

> ⚠️ **Viktigt:** Den publika on-premise-dokumentationen [OP-API] innehåller bara modulerna *Requests* (inkl. Notes, Tasks, Drafts, Archive), *Changes*, *Projects*, *Releases*, *Admin/User* och *General Tasks*. Varje on-premise-installation har även en inbyggd API-dokumentation under **Admin → API / Developer Space** som speglar den installerade versionen ([Online API documentation – Pitstop](https://pitstop.manageengine.com/portal/en/kb/articles/online-api-documentation)). Den inbyggda dokumentationen är den auktoritativa källan för Organisation A:s byggnummer och bör användas för att verifiera allt som markerats med "verifiera" i rapporten.

---

## 3. Integrationsmönster – hur SDP kan prata med leverantörens system

Det finns fyra grundmönster. I praktiken kombineras de per leverantör.

| # | Mönster | Riktning | SDP-mekanism | Lämpligt när |
|---|---|---|---|---|
| A | **E-post** | Båda | Utgående notifieringar och svar, inkommande e-posthämtning, *Email Command* | Leverantören saknar API, eller ärendevolymen är låg |
| B | **SDP anropar leverantörens API** | Ut | Custom Trigger → Webhook / Custom Function (Deluge `invokeurl`) / Execute Script | Leverantören har REST-API (t.ex. ServiceNow, Jira SM, Freshservice, egen portal med API) |
| C | **Leverantören eller ett mellanlager anropar SDP:s API** | In | REST API v3 med authtoken | Leverantören kan skicka webhooks eller uppdateringar, eller ett integrationslager pollar leverantören |
| D | **SDP pollar leverantören** | In | *Custom Schedules* med Deluge-funktion | Leverantören har API men inga utgående webhooks, och SDP ska inte exponeras utåt |

Referenser:

- Webhooks: [HELP – Webhooks](https://help.servicedeskplus.com/webhooks)
- Custom Functions: [HELP – Request Custom Functions](https://help.servicedeskplus.com/request-custom-function), [HELP – Deluge](https://help.servicedeskplus.com/using-deluge), [HELP – Global Functions](https://help.servicedeskplus.com/global-function)
- Custom Schedules: [HELP – Custom Schedules Function](https://help.servicedeskplus.com/custom-schedule-function)
- Custom Triggers: [HELP – Custom Triggers FAQ](https://help.servicedeskplus.com/configurations/helpdesk/customtriggers_faqs.html), [MSP – Custom Triggers for Requests](https://www.manageengine.com/products/service-desk-msp/help/adminguide/configurations/helpdesk/custom-trigger-for-requests.html)
- Email Command: [HELP – Email Command](https://help.servicedeskplus.com/email-command)
- Exempel på dubbelriktad integration (SDP on-prem ↔ Jira): [Pitstop KB – Two-way integration SDP On-Premise and Jira](https://pitstop.manageengine.com/portal/en/kb/articles/two-way-integration-between-manageengine-servicedesk-plus-on-premise-and-jira)

---

## 4. API-grunder (REST API v3, On-Premise)

| Aspekt | Beskrivning | Referens |
|---|---|---|
| **Bas-URL** | `http(s)://<server>:<port>/api/v3/<modul>` | [HELP – REST API](https://help.servicedeskplus.com/api/rest-api.html) |
| **Autentisering** | HTTP-headern `authtoken: <nyckel>`. Nyckeln genereras per tekniker med inloggningsbehörighet, av en administratör (*Generate Authtoken*), med eller utan utgångsdatum. Nyckeln raderas om teknikerns inloggning inaktiveras. | [OP-API – Introduction](https://www.manageengine.com/products/service-desk/sdpop-v3-api/index.html), [HELP – REST API](https://help.servicedeskplus.com/api/rest-api.html) |
| **Behörighet** | API-nyckeln ärver **teknikerns roll**. Det finns inga separata API-scopes. | [HELP – REST API](https://help.servicedeskplus.com/api/rest-api.html) |
| **Format** | Headern `Accept: application/vnd.manageengine.sdp.v3+json`. Data skickas som parametern `input_data` med JSON (form-kodad). | [OP-API – Input Data](https://www.manageengine.com/products/service-desk/sdpop-v3-api/getting-started/input-data.html) |
| **Listning och sökning** | `list_info` med `row_count` (max 100), `start_index`, `sort_field`, `sort_order`, `get_total_count`, `search_criteria` (fält, villkor, AND/OR, nästlade `children`) och `fields_required` | [OP-API – Search Criteria](https://www.manageengine.com/products/service-desk/sdpop-v3-api/getting-started/search-criteria.html), [OP-API – Pagination](https://www.manageengine.com/products/service-desk/sdpop-v3-api/getting-started/pagination.html) |
| **Statuskoder** | Standardiserade svarskoder i `response_status` | [OP-API – Status Codes](https://www.manageengine.com/products/service-desk/sdpop-v3-api/getting-started/common-error-code.html) |
| **Formatändringar** | v3-formatet har ändrats mellan byggen. Exempelvis bytte `request_note` namn till `note`, bilagornas URL:er ändrades och `stopclock` blev `stop_timer`. | [HELP – Request API (V3-ändringar)](https://help.servicedeskplus.com/request-api) |

**Brister i API-grunderna:**

- **Ingen OAuth 2.0 eller klientuppgifter med scopes** i on-premise-dokumentationen, bara teknikerbunden authtoken. En integration för en leverantör måste därför ha en egen "integrationstekniker" vars roll begränsar åtkomsten.
- **Ingen dokumenterad hastighetsbegränsning (rate limit)** för on-premise. Det är bra för volymen men ger inget skydd mot en felande integration.
- **Ingen OpenAPI/Swagger-specifikation** publiceras, vilket försvårar kodgenerering och import i integrationsplattformar ([Pitstop – Swagger documentation needed](https://pitstop.manageengine.com/portal/en/community/topic/swagger-documentation-for-servicedesk-plus-needed)).
- **Formatet har ändrats mellan byggen**, vilket kräver regressionstest av integrationer vid uppgradering ([HELP – Request API](https://help.servicedeskplus.com/request-api)).

---

## 5. Användningsfall

Varje användningsfall beskrivs med **behov**, **SDP-förmåga och API-referens** samt **brister**.

### UC1 – Eskalera ett ärende till leverantör (skapa externt ärende)

**Behov:** En tekniker konstaterar att leverantören behöver involveras. Ett ärende ska skapas i leverantörens system med relevant information från SDP-ärendet: beskrivning, påverkan, prioritet, berörd tillgång, kontaktuppgifter och bilagor.

**SDP-förmåga:**

| Steg | Mekanism | Referens |
|---|---|---|
| Starta eskaleringen | Teknikern sätter ett fält (t.ex. UDF "Eskalera till leverantör" = *Dell*) eller status *Väntar på leverantör*. En **Custom Trigger** med villkor på fältet startar åtgärden. Alternativt används en *Custom Menu* (manuell knapp) som kör samma funktion. | [MSP – Custom Triggers for Requests](https://www.manageengine.com/products/service-desk-msp/help/adminguide/configurations/helpdesk/custom-trigger-for-requests.html), [HELP – Custom Triggers FAQ](https://help.servicedeskplus.com/configurations/helpdesk/customtriggers_faqs.html) |
| Anropa leverantören | **Webhook** (GET/POST/PUT/PATCH/DELETE, egna headers, `$`-variabler från ärendet, för- och efterbearbetning med Deluge) eller **Custom Function** (Deluge `invokeurl`) eller **Execute Script** (t.ex. Python eller PowerShell som får `$COMPLETE_V3_JSON_FILE` med hela ärendet inkl. egna fält) | [HELP – Webhooks](https://help.servicedeskplus.com/webhooks), [HELP – Request Custom Functions](https://help.servicedeskplus.com/request-custom-function), [HELP – Custom Triggers FAQ](https://help.servicedeskplus.com/configurations/helpdesk/customtriggers_faqs.html) |
| Leverantör utan API | E-postnotifiering till leverantörens supportadress via trigger-åtgärden *Notification*, eller svar från ärendet (konversation) | [MSP – Custom Triggers for Requests](https://www.manageengine.com/products/service-desk-msp/help/adminguide/configurations/helpdesk/custom-trigger-for-requests.html) |
| Läsa ärendedata | `GET /api/v3/requests/{id}` | [OP-API – Request](https://www.manageengine.com/products/service-desk/sdpop-v3-api/requests/request.html) |

**Brister:**

- **Ingen inbyggd koppling mellan leverantör och tjänst eller kategori.** SDP vet inte vilken leverantör som ansvarar för en viss tjänst, kategori eller tillgång. Routingen måste kodas i triggervillkor eller i en mappningstabell i integrationen.
- **Ingen inbyggd fältmappning.** Mappning av prioritet, kategori och status till leverantörens värden måste skrivas för hand per leverantör.
- **Begränsningar i triggerkonstruktionen:** högst ett If-If-block per trigger (10 block × 5 åtgärder), och Custom Functions och skript kan inte kombineras i samma block ([MSP – Custom Triggers for Requests](https://www.manageengine.com/products/service-desk-msp/help/adminguide/configurations/helpdesk/custom-trigger-for-requests.html)). Med många leverantörer blir triggerkonfigurationen svår att överblicka.
- **Webhooks har ingen dokumenterad hantering av omförsök eller köer.** Om leverantörens API inte svarar finns ingen garanti för att anropet görs igen. Felet syns bara i ärendets historik.
- **Execute Script** kräver filer och runtime (t.ex. Python) på SDP-servern. Det ger drift- och säkerhetsansvar och fungerar sämre i en klustrad eller hårdnad servermiljö.

---

### UC2 – Lagra och spåra leverantörens ärendereferens

**Behov:** SDP-ärendet ska innehålla leverantörens ärendenummer och gärna en länk, så att tekniker och integration kan hitta rätt externt ärende. Det ska gå att söka fram alla ärenden som väntar på en viss leverantör.

**SDP-förmåga:**

- **Egna fält (Additional Fields / UDF)** på ärendemallen, t.ex. `Leverantör`, `Leverantörens ärende-ID`, `Leverantörsstatus` och `Eskalerad datum`. De skrivs via `PUT /api/v3/requests/{id}` i `udf_fields` ([OP-API – Request](https://www.manageengine.com/products/service-desk/sdpop-v3-api/requests/request.html)). Jira-exemplet från ManageEngine använder exakt detta mönster: Jira-nyckeln sparas i ett UDF-fält ([Pitstop KB – SDP ↔ Jira](https://pitstop.manageengine.com/portal/en/kb/articles/two-way-integration-between-manageengine-servicedesk-plus-on-premise-and-jira)).
- **Sökning** på UDF-fältet med `search_criteria` i `GET /api/v3/requests`, t.ex. `udf_fields.udf_sline_xxx` *is* `INC0012345` ([OP-API – Search Criteria](https://www.manageengine.com/products/service-desk/sdpop-v3-api/getting-started/search-criteria.html)). Detta behövs när leverantören skickar en uppdatering som bara innehåller deras eget ID.

**Brister:**

- **Endast ett externt ID per fält.** Om flera leverantörer är inblandade i samma ärende (UC8) behövs flera fältuppsättningar, eller att varje leverantör modelleras som en *uppgift* (UC7). Det finns ingen generisk tabell för externa referenser.
- **UDF-fälten heter internt `udf_sline_NNN`** och skiljer sig mellan miljöer (test och produktion). Integrationen måste konfigureras per miljö.
- **Ingen unikhetskontroll** på UDF-fält, så dubblettskydd måste byggas i integrationen.

---

### UC3 – Skicka uppdateringar till leverantören (kommentarer, ny information, statusändring)

**Behov:** När service desk eller verksamheten tillför ny information, till exempel svar på leverantörens fråga, nya loggar eller ändrad prioritet, ska den föras över till leverantörens ärende.

**SDP-förmåga:**

- **Fältändringar** (prioritet, status, UDF) fångas av en Custom Trigger på *Request edited* och kan skickas ut via Webhook eller Custom Function ([MSP – Custom Triggers for Requests](https://www.manageengine.com/products/service-desk-msp/help/adminguide/configurations/helpdesk/custom-trigger-for-requests.html)). Webhookens förfunktion får en `diffJson` med vad som ändrats ([HELP – Webhooks](https://help.servicedeskplus.com/webhooks)).
- **Anteckningar** kan läsas via `GET /api/v3/requests/{id}/notes` ([OP-API – Request Note](https://www.manageengine.com/products/service-desk/sdpop-v3-api/requests/request_note.html)).

**Brister:**

- ❗ **Anteckningar, svar och konversationer räknas inte som en redigering av ärendet och kan därför inte trigga en Custom Trigger** ([HELP – Custom Triggers FAQ](https://help.servicedeskplus.com/configurations/helpdesk/customtriggers_faqs.html)). Detta är den **största enskilda bristen** för ärendekommunikation med leverantörer. Kommentarssynk måste lösas på något av följande sätt:
  - teknikern sätter samtidigt ett fält (t.ex. "Skicka till leverantör = Ja") som triggar synken, vilket är manuellt och lätt att glömma
  - en *Custom Menu*-knapp ("Skicka anteckning till leverantör")
  - schemalagd polling (*Custom Schedules*) av nya anteckningar, vilket ger fördröjning och kräver att integrationen håller reda på vilka anteckningar som redan skickats.
- **Ingen inbyggd skillnad mellan intern anteckning och anteckning till leverantören.** Anteckningar har `show_to_requester` men inget motsvarande för en extern part. Integrationen måste införa en konvention, till exempel ett prefix eller en separat uppgift per leverantör.

---

### UC4 – Ta emot uppdateringar från leverantören

**Behov:** När leverantören kommenterar, begär information, ändrar status eller löser sitt ärende ska det synas i SDP-ärendet. Helst ska det gå automatiskt, och vid behov ska ärendets status ändras (t.ex. från *Väntar på leverantör* till *Pågående*).

**SDP-förmåga:**

| Operation | API | Referens |
|---|---|---|
| Hitta rätt ärende utifrån leverantörens ID | `GET /api/v3/requests` med `search_criteria` på UDF | [OP-API – Search Criteria](https://www.manageengine.com/products/service-desk/sdpop-v3-api/getting-started/search-criteria.html) |
| Lägga till leverantörens kommentar | `POST /api/v3/requests/{id}/notes` (`description`, `show_to_requester`, `add_to_linked_requests`) | [OP-API – Request Note](https://www.manageengine.com/products/service-desk/sdpop-v3-api/requests/request_note.html) |
| Uppdatera status och fält | `PUT /api/v3/requests/{id}` (`status`, `udf_fields` …) | [OP-API – Request](https://www.manageengine.com/products/service-desk/sdpop-v3-api/requests/request.html) |
| Polling från SDP | *Custom Schedules* med Deluge-funktion som anropar leverantörens API | [HELP – Custom Schedules Function](https://help.servicedeskplus.com/custom-schedule-function) |
| E-post från leverantören | Inkommande e-post med `##<ärende-ID>##` i ämnet läggs till som konversation i befintligt ärende. *Email Command* (`EditRequest`, `CloseRequest`) kan ändra fält via e-post. | [HELP – Email Command](https://help.servicedeskplus.com/email-command), [Pitstop – Same case](https://pitstop.manageengine.com/portal/en/community/topic/how-servicedesk-plus-consider-incoming-emails-as-same-case) |

**Brister:**

- **SDP saknar en inkommande webhook-endpoint.** Leverantörens system måste anropa SDP:s generella REST-API med en authtoken. Det kräver att:
  - SDP-servern på intern on-premise-miljö **exponeras mot internet** (reverse proxy, WAF, IP-filtrering), eller
  - ett **mellanlager** tar emot leverantörens webhooks och anropar SDP internt.
  ManageEngines egen Jira-guide påpekar att en on-premise-SDP bakom NAT behöver reverse proxy eller tunnel ([Pitstop KB – SDP ↔ Jira](https://pitstop.manageengine.com/portal/en/kb/articles/two-way-integration-between-manageengine-servicedesk-plus-on-premise-and-jira)).
- **Leverantören kan inte uppdatera ärendet utan en teknikernyckel.** API-nycklar finns bara för tekniker med inloggning, så en leverantörsintegration tar en teknikerlicens i anspråk och får teknikerns rollbehörigheter.
- **E-post är bräckligt mellan två ärendesystem.** När två ärendesystem mejlar varandra innehåller ämnesraden ofta båda systemens ärende-ID. Det leder till fel trådning, nya dubblettärenden eller loopar av automatiska svar ([Pitstop – mail between two SD Plus systems](https://pitstop.manageengine.com/portal/en/community/topic/mail-sent-from-two-different-sd-plus-systems-communication-breaks-down), [Pitstop – handling e-mails from other ITSM systems](https://pitstop.manageengine.com/portal/en/community/topic/how-are-others-handling-incoming-emails-messages-from-other-itsm-systems)).
- **Email Command kräver att avsändaren är en användare i SDP med inloggning.** Ändringar via Email Command hamnar dessutom i *Historik*, inte i *Konversationer* ([HELP – Email Command](https://help.servicedeskplus.com/email-command)). Funktionen passar därför dåligt för leverantörers automatiska utskick.
- **Polling via Custom Schedules** har en minsta intervall och belastar SDP-servern. Integrationen måste själv hålla reda på senaste synkade tidpunkt.

---

### UC5 – Bilagor (loggar, skärmdumpar, felrapporter, diagnosfiler)

**Behov:** Bilagor ska kunna skickas till leverantören, t.ex. en loggfil, en diagnostikrapport från en server eller en skärmdump av ett fel. Leverantörens bilagor, som felrapporter, åtgärdsprotokoll och RMA-dokument, ska kunna tas emot.

**SDP-förmåga:**

| Operation | API | Referens |
|---|---|---|
| Ladda upp bilaga till ärende | `PUT /api/v3/requests/{id}/upload`, multipart med parametern `input_file`, **en fil per anrop** | [MSP – V3 Attachment API Changes](https://www.manageengine.com/products/service-desk-msp/help/adminguide/general-features/v3-attachment-API-changes.html) (on-prem-dokumentationen [OP-API] beskriver inte uppladdning, så **verifiera** i instansens API-dokumentation) |
| Lista bilagor | Fältet `attachments` i `GET /api/v3/requests/{id}` (id, namn, `content_url`, storlek) | [OP-API – Request](https://www.manageengine.com/products/service-desk/sdpop-v3-api/requests/request.html) |
| Hämta bilaga | `GET /api/v3/requests/{id}/attachments/{attachment_id}/download` | [HELP – Request API (V3-ändringar)](https://help.servicedeskplus.com/request-api) |
| Bilagor på uppgifter | `GET/DELETE /api/v3/requests/{id}/tasks/{task_id}/attachments/{att_id}` | [OP-API – Request Task](https://www.manageengine.com/products/service-desk/sdpop-v3-api/requests/request_task.html) |

**Brister:**

- **Bilagor kan inte kopplas till anteckningar.** Anteckningsobjektet har inga bilagor ([OP-API – Request Note](https://www.manageengine.com/products/service-desk/sdpop-v3-api/requests/request_note.html)), så kopplingen mellan "kommentar + bifogad fil" går förlorad. Leverantörssystem som ServiceNow och Jira bär ofta bilagor per kommentar.
- **Uppladdning är dåligt dokumenterad för on-premise.** Forumtrådar rapporterar att uppladdningen ger HTTP 200 men att filen inte syns på ärendet ([Pitstop – SDP API attachment missing](https://pitstop.manageengine.com/portal/en/community/topic/sdp-ondemand-api-problem-attachment-successfully-added-mut-missing-from-request), [Pitstop – API Adding Attachments](https://pitstop.manageengine.com/portal/en/community/topic/api-adding-attachments)).
- **Bilagor triggar inte Custom Triggers.** Precis som anteckningar (UC3) kräver utgående bilagesynk polling eller manuellt steg.
- **Webhooks kan inte skicka binära filer** enligt dokumentationen. Utgående bilagor kräver Custom Function eller skript som först hämtar filen.
- **Storleksgränser** är inte dokumenterade för API:t. För e-post är standardgränsen 10 MB, och den kan ändras ([MSP – Custom Triggers for Requests](https://www.manageengine.com/products/service-desk-msp/help/adminguide/configurations/helpdesk/custom-trigger-for-requests.html)).

---

### UC6 – Vänta på leverantör: status, SLA och leverantörens åtagande

**Behov:** När ärendet ligger hos leverantören ska SLA-klockan mot verksamheten hanteras enligt avtal (oftast pausas, eller fortsätta men med separat uppföljning). Leverantörens egna åtaganden, som svarstid, åtgärdstid och on-site inom X timmar, ska kunna följas upp.

**SDP-förmåga:**

- **Egna statusar med "stoppa timer".** Man kan skapa en status som *Väntar på leverantör* med `stop_timer` (tidigare `stopclock`), som pausar SLA ([HELP – Request API, statusfält](https://help.servicedeskplus.com/request-api), [HELP – Start/Stop Request Timer](https://help.servicedeskplus.com/requests/request-on-hold.html)). Statusen sätts via `PUT /api/v3/requests/{id}` ([OP-API – Request](https://www.manageengine.com/products/service-desk/sdpop-v3-api/requests/request.html)).
- **SLA-fält** som `due_by_time`, `first_response_due_by_time`, `ola_due_by_time` och `sla` kan läsas i ärendets JSON ([OP-API – Request](https://www.manageengine.com/products/service-desk/sdpop-v3-api/requests/request.html)).
- **OLA** (Operational Level Agreement) per supportgrupp finns i Enterprise. Den kan användas som approximation om varje leverantör modelleras som en supportgrupp.
- **Uppgifter (tasks)** har `scheduled_start_time`, `scheduled_end_time` och `estimated_effort` ([OP-API – Request Task](https://www.manageengine.com/products/service-desk/sdpop-v3-api/requests/request_task.html)) och kan användas för att följa leverantörens förväntade leveranstid.

**Brister:**

- ❗ **Det finns inget stöd för Underpinning Contracts (UC) kopplade till ärenden.** SDP:s *Contracts*-modul hanterar avtal kopplade till leverantör och tillgång (förnyelse, kostnad, giltighet) ([HELP – Contracts](https://help.servicedeskplus.com/contracts/service-desk-contracts.html)). Avtalens servicenivåer (t.ex. "Next Business Day on-site") driver dock inga timers på ärenden. Leverantörens SLA-uppfyllelse kan inte mätas automatiskt.
- **OLA är gjort för interna grupper**, inte externa parter. Leverantören arbetar inte i SDP, så OLA-mätningen blir beroende av att integrationen sätter rätt status och tidpunkt.
- **Timern stoppas för hela ärendet, inte per del.** Om leverantören bara löser en del av ärendet (t.ex. byte av hårddisk medan IT återställer data) kan SLA:n inte pausas för den delen.
- **Forumtrådar rapporterar inkonsekvent SLA-beräkning** kring On Hold i vissa byggen ([Pitstop – SLA Timer (On Hold Status)](https://pitstop.manageengine.com/portal/en/community/topic/sla-timer-on-hold-status-10-7-2020)). Detta bör testas.

---

### UC7 – Delegera delar av ärendet till leverantör(er) med uppgifter

**Behov:** Ett ärende kan kräva arbete från både IT och en eller flera leverantörer. Varje leverantörsinsats ska kunna följas separat.

**SDP-förmåga:**

| Operation | API | Referens |
|---|---|---|
| Skapa uppgift på ärende | `POST /api/v3/requests/{id}/tasks` | [OP-API – Request Task](https://www.manageengine.com/products/service-desk/sdpop-v3-api/requests/request_task.html) |
| Uppdatera, stänga, tilldela | `PUT .../tasks/{task_id}`, `PUT .../_close`, `PUT .../_assign` | [OP-API – Request Task](https://www.manageengine.com/products/service-desk/sdpop-v3-api/requests/request_task.html) |
| Beroenden mellan uppgifter | `POST/GET/DELETE .../task_dependencies` | [OP-API – Request Task](https://www.manageengine.com/products/service-desk/sdpop-v3-api/requests/request_task.html) |
| Tidsåtgång (worklog) | `POST /api/v3/requests/{id}/worklogs`. Dokumenterat för Cloud och MSP, inte i [OP-API], men listat i on-prem-hjälpens operationsöversikt. | [HELP – REST API](https://help.servicedeskplus.com/api/rest-api.html), [CLOUD – Request Worklog](https://www.manageengine.com/products/service-desk/sdpod-v3-api/requests/request_worklog.html) |
| Custom Function på uppgifter | Deluge-funktioner kan köras för uppgifter | [HELP – Task Custom Function](https://help.servicedeskplus.com/task-custom-function) |

En rimlig modell är **en uppgift per leverantörsärende**. Uppgiftens titel och egna fält bär leverantör och externt ID, och uppgiftens status speglar leverantörens status.

**Brister:**

- **Uppgifter har ingen kommentarstråd** i on-premise-API:t. Leverantörens kommentarer måste läggas som anteckningar på ärendet, inte på uppgiften.
- **Uppgiftens ägare måste vara en tekniker i SDP.** Leverantören kan inte vara ägare utan att ha en teknikerlicens.
- **Ingen dokumenterad trigger på uppgiftsändring** i on-premise-dokumentationen för Custom Triggers, som gäller ärenden. **Verifiera** om Task Custom Functions kan anropas vid ändring.

---

### UC8 – Flera leverantörer, länkade ärenden och storstörning

**Behov:** En storstörning, till exempel ett telefoniavbrott, kan generera många ärenden från verksamheten men bara ett ärende hos leverantören. Ibland är flera leverantörer inblandade, t.ex. en systemleverantör och en hårdvaruleverantör.

**SDP-förmåga:**

| Funktion | API | Referens |
|---|---|---|
| Länka ärenden (huvudärende och länkade ärenden) | `POST /api/v3/requests/{id}/link_requests`, motsvarande DELETE. **Se brist nedan.** | [OP-API – Request](https://www.manageengine.com/products/service-desk/sdpop-v3-api/requests/request.html), [Pitstop KB (CN) – länka ärenden via API](https://support.manageengine.cn/portal/zh/kb/articles/%E8%AF%B7%E6%B1%82%E7%9B%B8%E5%85%B3-%E9%93%BE%E6%8E%A5%E8%AF%B7%E6%B1%82) |
| Anteckning till alla länkade ärenden | `add_to_linked_requests: true` vid `POST .../notes` | [OP-API – Request Note](https://www.manageengine.com/products/service-desk/sdpop-v3-api/requests/request_note.html) |
| Slå ihop ärenden | `PUT /api/v3/requests/{id}/merge_requests` | [OP-API – Request](https://www.manageengine.com/products/service-desk/sdpop-v3-api/requests/request.html) |
| Koppla till problem | `POST/GET/DELETE /api/v3/requests/{id}/problem` | [OP-API – Request](https://www.manageengine.com/products/service-desk/sdpop-v3-api/requests/request.html) |
| Koppla till ändring | `.../request_initiated_change`, `.../request_caused_by_change` | [OP-API – Request](https://www.manageengine.com/products/service-desk/sdpop-v3-api/requests/request.html) |

Rekommenderad modell: **ett huvudärende** (eller ett problem) bär leverantörsreferensen. Övriga ärenden länkas dit, och leverantörens uppdateringar sprids med `add_to_linked_requests`.

**Brister:**

- ❗ **Dokumentationen motsäger sig själv om länkning via API.** Hjälpdokumentationen säger uttryckligen *"Requests cannot be linked through API or E-Mail Command features"* ([HELP – Linking Requests](https://help.servicedeskplus.com/requests/linking-requests.html)), medan API-dokumentation och kunskapsbasartiklar beskriver endpointen `link_requests`. Detta **måste verifieras** i Organisation A:s byggnummer.
- **Ett ärende kan bara ha ett huvudärende.** Ett ärende som berör två leverantörer kan alltså inte länkas till två separata huvudärenden.
- **Ingen inbyggd hantering av flera externa referenser per ärende** (se UC2).
- **Problemmodulens API saknas i den publika on-premise-dokumentationen** och finns bara som associering från ärende. **Verifiera** fullständigt problem-API i instansens dokumentation.

---

### UC9 – Lösning och avslut i båda system

**Behov:** När leverantören meddelar att åtgärden är klar ska IT kunna verifiera, dokumentera lösningen och stänga sitt ärende. Det ska också gå åt andra hållet: om verksamheten bekräftar lösningen eller ärendet stängs i SDP ska leverantörens ärende stängas.

**SDP-förmåga:**

| Operation | API | Referens |
|---|---|---|
| Lägga till eller läsa lösning | `POST/GET /api/v3/requests/{id}/resolutions` | [OP-API – Request](https://www.manageengine.com/products/service-desk/sdpop-v3-api/requests/request.html) |
| Stänga ärende (med `closure_code` och `closure_comments`) | `PUT /api/v3/requests/{id}/close` | [OP-API – Request](https://www.manageengine.com/products/service-desk/sdpop-v3-api/requests/request.html) |
| Stänga uppgift | `PUT .../tasks/{task_id}/_close` | [OP-API – Request Task](https://www.manageengine.com/products/service-desk/sdpop-v3-api/requests/request_task.html) |
| Stänga hos leverantören vid avslut i SDP | Custom Trigger (villkor: status = Stängd/Löst) → Webhook eller Custom Function | [HELP – Webhooks](https://help.servicedeskplus.com/webhooks) |

**Brister:**

- **Det finns inget inbyggt tillstånd för "leverantören har löst sin del".** Det modelleras med uppgiftsstatus eller UDF.
- **Ingen kontroll av att öppna leverantörsärenden stängts.** SDP kan stänga ärendet trots att leverantörens ärende fortfarande är öppet, om inte en *Business Rule*, *Request Life Cycle* eller Custom Function blockerar det. Enterprise har Request Life Cycle, men det måste konfigureras.
- **Återöppning** hos leverantören eller i SDP kräver egen logik för att spegla tillbaka.

---

### UC10 – Hårdvara: garanti, RMA, tillgångar och kontrakt

**Behov:** Vid hårdvarufel ska tekniker se vilken leverantör och vilket garanti- eller serviceavtal som gäller för den berörda tillgången, och skicka serienummer, modell och avtalsnummer till leverantören (t.ex. ett garantiärende hos Dell, HP eller Lenovo).

**SDP-förmåga:**

- **Tillgångar kopplade till ärendet** finns i fältet `assets` på ärendet ([OP-API – Request](https://www.manageengine.com/products/service-desk/sdpop-v3-api/requests/request.html)). SDP Enterprise har inbyggd tillgångshantering (CMDB och tillgångar).
- **Kontrakt per tillgång** kan hämtas via `GET /api/v3/assets/{asset_id}/contracts` ([Pitstop KB – contract details per asset](https://pitstop.manageengine.com/portal/en/kb/articles/to-pull-out-the-contract-details-to-which-an-asset-is-associated-through-api)). Endpointen dokumenteras för Cloud ([CLOUD – Contract](https://www.manageengine.com/products/service-desk/sdpod-v3-api/contracts/contract.html)), men **inte** i [OP-API], så den måste **verifieras**.
- I en Custom Function kan man alltså slå upp tillgång → kontrakt → leverantör och använda det för att routa och berika leverantörsärendet.

**Brister:**

- **Ingen inbyggd integration mot tillverkarnas garanti- eller RMA-API:er** (Dell TechDirect, HP, Lenovo). Detta måste byggas.
- **Leverantörer (Vendors) saknar dokumenterat v3-API** i on-premise-dokumentationen. Leverantörsregistret används främst i inköp och kontrakt och är inte kopplat till ärenden.
- **Inget fält för leverantörens RMA- eller fraktnummer** och inget arbetsflöde för retur. Det modelleras med UDF och uppgifter.

---

### UC11 – Ändringar som utförs av leverantören

**Behov:** Leverantören ska genomföra en ändring i organisationens miljö, t.ex. en systemuppgradering eller en ändring i telefoniväxeln. Ändringen ska planeras, godkännas (CAB) och följas upp i SDP.

**SDP-förmåga:**

- **Ändringsmodulen** med godkännanden, nivåer och uppgifter ([OP-API – Change](https://www.manageengine.com/products/service-desk/sdpop-v3-api/changes/change.html), [Change Approval](https://www.manageengine.com/products/service-desk/sdpop-v3-api/changes/change_approval.html), [Change Task](https://www.manageengine.com/products/service-desk/sdpop-v3-api/changes/change_task.html)).
- **Koppling ärende → ändring** via `request_initiated_change` ([OP-API – Request](https://www.manageengine.com/products/service-desk/sdpop-v3-api/requests/request.html)).
- **Custom Triggers och Custom Functions** finns även för ändringar ([HELP – Change Custom Functions](https://help.servicedeskplus.com/change-custom-function)). ManageEngines Jira-exempel visar en dubbelriktad integration på ändringsnivå ([Pitstop KB – SDP ↔ Jira](https://pitstop.manageengine.com/portal/en/kb/articles/two-way-integration-between-manageengine-servicedesk-plus-on-premise-and-jira)).

**Brister:**

- **Ingen dokumenterad endpoint för anteckningar eller worklogs på ändringar** i [OP-API] (bara för releaser). Leverantörens framstegsrapportering kan behöva ske via uppgifter.
- Samma begränsningar som för ärenden när det gäller extern referens och triggers.

---

### UC12 – Uppföljning av leverantörsprestation

**Behov:** IT vill följa upp antal ärenden per leverantör, ledtid hos leverantören, återöppnade ärenden och SLA-uppfyllelse som underlag för leverantörsmöten och avtalsförhandling.

**SDP-förmåga:**

- **Listning med filter** via `GET /api/v3/requests` och `search_criteria` på UDF "Leverantör", status och datum ([OP-API – Search Criteria](https://www.manageengine.com/products/service-desk/sdpop-v3-api/getting-started/search-criteria.html)).
- **Inbyggda rapporter** (Query Reports) i SDP på UDF-fält.
- **Ärendesammanfattning:** `GET /api/v3/requests/{id}/summary` ([OP-API – Request](https://www.manageengine.com/products/service-desk/sdpop-v3-api/requests/request.html)).

**Brister:**

- **Tid hos leverantören mäts inte automatiskt.** Den måste beräknas från statushistorik (tid i *Väntar på leverantör*) eller från tidsstämplar i UDF som integrationen sätter.
- **API:t saknar en dokumenterad endpoint för ärendets historik och statusövergångar** i [OP-API], vilket försvårar extern analys.
- **Max 100 rader per anrop** gör det långsamt att ta ut större datamängder. Rapporter eller databasexport kan vara bättre för analys.

---

## 6. Sammanställning av brister

| # | Brist | Påverkar | Allvarlighet | Möjlig kompensation |
|---|---|---|---|---|
| B1 | Ingen inbyggd e-bonding eller koppling mellan ärendesystem | Alla | Hög | Integrationslager (iPaaS) eller egenutvecklade Deluge-funktioner per leverantör |
| B2 | Custom Triggers reagerar inte på anteckningar, konversationer eller bilagor | UC3, UC5 | Hög | Custom Menu-knapp, fält som triggar, eller polling med Custom Schedules |
| B3 | Webhooks saknar dokumenterade omförsök, kö och larm | UC1, UC3, UC9 | Hög | Låt SDP anropa ett internt integrationslager med kö, i stället för leverantören direkt |
| B4 | Ingen inkommande webhook-endpoint. Leverantören måste anropa SDP:s REST-API med teknikernyckel. | UC4 | Hög | Mellanlager i DMZ, eller polling från SDP |
| B5 | Authtoken är bunden till tekniker och roll, utan OAuth eller scopes | UC4, säkerhet | Medel | Dedikerad integrationstekniker med minimal roll och nyckel med utgångsdatum |
| B6 | Inga Underpinning Contracts eller leverantörs-SLA kopplade till ärenden | UC6, UC12 | Medel–Hög | UDF och tidsstämplar, status med stoppad timer, OLA per "leverantörsgrupp", egen rapport |
| B7 | Ingen generisk struktur för externa referenser (en leverantör per fält) | UC2, UC8 | Medel | En uppgift per leverantörsärende, eller flera UDF-uppsättningar |
| B8 | Bilagor kan inte kopplas till anteckningar, och uppladdning är dåligt dokumenterad för on-prem | UC5 | Medel | Konvention: anteckning anger filnamn. Verifiera uppladdning i testmiljö. |
| B9 | Motsägelsefull dokumentation om länkning av ärenden via API | UC8 | Medel | Verifiera i instansens API-dokumentation, eller använd problem som huvudobjekt |
| B10 | Den publika on-prem-dokumentationen saknar flera moduler (worklog, kontrakt, leverantörer, problem, historik) | Alla | Medel | Använd instansens inbyggda API-dokumentation (Admin → API) |
| B11 | Inget leverantörsregister kopplat till tjänster, kategorier eller ärenden | UC1, UC10 | Medel | Mappningstabell i integrationslagret eller i en Deluge-funktion |
| B12 | E-postintegration mellan ärendesystem är bräcklig (trådning, loopar, Email Command kräver SDP-användare) | UC4 | Medel | Fasta ämnesformat, separata brevlådor per leverantör, filter mot autosvar |
| B13 | v3-formatet ändras mellan byggen | Drift | Låg–Medel | Regressionstest av integrationer före uppgradering |
| B14 | Execute Script kräver runtime och skriptfiler på SDP-servern | Drift, säkerhet | Låg–Medel | Använd Deluge eller Webhook i stället, eller flytta logiken till integrationslagret |

---

## 7. Rekommenderad arkitektur och nästa steg

### 7.1 Rekommenderad arkitektur

```
 Verksamhet ──► SDP (On-Prem) ──Webhook/Custom Function──► Integrationslager ──► Leverantör A (REST-API)
                    ▲                                        (kö, mappning,      ──► Leverantör B (REST-API)
                    │                                         omförsök, logg)    ──► Leverantör C (e-post)
                    └────────── SDP REST API v3 ◄─────────────────┘ ◄── webhooks/polling från leverantörer
```

1. **SDP är master** för verksamhetens ärende. Leverantörens ärende speglas, i första hand som en **uppgift per leverantörsärende** med extern referens i UDF.
2. **Utgående:** Custom Trigger (vid skapande eller ändring) och Custom Menu (vid anteckningar och bilagor) → Webhook till **integrationslagret**, inte direkt till leverantören. Lagret har kö, omförsök, loggning och fältmappning per leverantör.
3. **Inkommande:** Integrationslagret tar emot leverantörens webhooks eller pollar deras API, och skriver till SDP via REST API v3 (anteckningar, status, uppgifter, bilagor). SDP behöver då inte exponeras mot internet.
4. **Status:** Inför statusen *Väntar på leverantör* (stoppad timer) och UDF för leverantör, externt ID, extern status och tidsstämplar.
5. **Leverantörer utan API:** E-post från en dedikerad brevlåda per leverantör med fast ämnesformat (`[SDP ##<id>##] <ärende>`) och regler mot autosvarsloopar.

> **Tillägg 2026-11-26:** Integrationslagret är skissat i [`integrationslager-skiss.md`](integrationslager-skiss.md), som en del av en plattform på Azure Integration Services. Skissen preciserar punkterna ovan på tre sätt:
>
> - Pollning efter anteckningar och bilagor görs av integrationslagret, inte med *Custom Schedules* i SDP (punkt 2, UC3, UC4).
> - E-post till och från leverantörer utan API går via integrationslagrets brevlådor, inte via SDP:s e-posthantering (punkt 5).
> - En gemensam integrationstekniker räcker, eftersom leverantörer aldrig anropar SDP direkt (jfr B5 och licenspunkten i 7.2).

### 7.2 Att verifiera i Organisation A:s installation

Verifiera med instansens API-dokumentation under Admin → API / Developer Space, i en testmiljö:

- [ ] Byggnummer och om v3-formatet med `note` och `stop_timer` gäller
- [ ] Bilageuppladdning: `PUT /api/v3/requests/{id}/upload` med `input_file`, och att filen syns på ärendet
- [ ] `link_requests` via API (motsägelse i dokumentationen)
- [ ] Worklog-, problem-, kontrakt- och leverantörsendpoints
- [ ] Om det finns en trigger eller custom function på ändringar av uppgifter
- [ ] Webhookens autentiseringstyper (Basic, OAuth, API-nyckel i header), timeout och felbeteende
- [ ] Om Deluge-*Connections* finns i on-prem för säker hantering av leverantörers inloggningsuppgifter
- [ ] Minsta intervall för Custom Schedules
- [ ] SLA-beteende vid statusen *Väntar på leverantör* (stoppad timer) i kombination med OLA
- [ ] Licenspåverkan av integrationstekniker (en teknikerlicens per integration)

### 7.3 Frågor till leverantörerna

För varje leverantör bör man ta reda på:

- Har de ett API för att skapa, uppdatera och läsa ärenden?
- Skickar de utgående webhooks?
- Vilka autentiseringsmetoder stöder de?
- Vilka fält är obligatoriska?
- Hur hanterar de bilagor?
- Vilka statusar har de?
- Vilka SLA-nivåer gäller enligt avtal?

Svaren avgör om mönster A, B, C eller D (avsnitt 3) ska användas per leverantör.

---

## 8. Referenser

### ServiceDesk Plus On-Premise – REST API v3 [OP-API]

- Introduktion och autentisering: https://www.manageengine.com/products/service-desk/sdpop-v3-api/index.html
- Input data: https://www.manageengine.com/products/service-desk/sdpop-v3-api/getting-started/input-data.html
- Search criteria: https://www.manageengine.com/products/service-desk/sdpop-v3-api/getting-started/search-criteria.html
- Pagination: https://www.manageengine.com/products/service-desk/sdpop-v3-api/getting-started/pagination.html
- Status codes: https://www.manageengine.com/products/service-desk/sdpop-v3-api/getting-started/common-error-code.html
- Request: https://www.manageengine.com/products/service-desk/sdpop-v3-api/requests/request.html
- Request Note: https://www.manageengine.com/products/service-desk/sdpop-v3-api/requests/request_note.html
- Request Task: https://www.manageengine.com/products/service-desk/sdpop-v3-api/requests/request_task.html
- Change: https://www.manageengine.com/products/service-desk/sdpop-v3-api/changes/change.html
- General Task: https://www.manageengine.com/products/service-desk/sdpop-v3-api/tasks/task.html

### ServiceDesk Plus On-Premise – hjälpdokumentation [HELP]

- REST API (översikt, authtoken): https://help.servicedeskplus.com/api/rest-api.html
- Request API – V3-formatändringar: https://help.servicedeskplus.com/request-api
- Webhooks: https://help.servicedeskplus.com/webhooks
- Custom Triggers FAQ: https://help.servicedeskplus.com/configurations/helpdesk/customtriggers_faqs.html
- Request Custom Functions: https://help.servicedeskplus.com/request-custom-function
- Task Custom Function: https://help.servicedeskplus.com/task-custom-function
- Change Custom Functions: https://help.servicedeskplus.com/change-custom-function
- Global Functions: https://help.servicedeskplus.com/global-function
- Custom Schedules Function: https://help.servicedeskplus.com/custom-schedule-function
- Deluge i ServiceDesk Plus: https://help.servicedeskplus.com/using-deluge
- Callback Functions: https://help.servicedeskplus.com/callback-functions
- Linking Requests: https://help.servicedeskplus.com/requests/linking-requests.html
- Email Command: https://help.servicedeskplus.com/email-command
- Start/Stop Request Timer: https://help.servicedeskplus.com/requests/request-on-hold.html
- Contracts: https://help.servicedeskplus.com/contracts/service-desk-contracts.html

### Kompletterande källor (MSP och Cloud)

- [MSP] Custom Triggers for Requests: https://www.manageengine.com/products/service-desk-msp/help/adminguide/configurations/helpdesk/custom-trigger-for-requests.html
- [MSP] V3 Attachment API Changes: https://www.manageengine.com/products/service-desk-msp/help/adminguide/general-features/v3-attachment-API-changes.html
- [CLOUD] Request Worklog: https://www.manageengine.com/products/service-desk/sdpod-v3-api/requests/request_worklog.html
- [CLOUD] Contract: https://www.manageengine.com/products/service-desk/sdpod-v3-api/contracts/contract.html

### ManageEngine Pitstop (kunskapsbas och forum)

- Two-way integration SDP On-Premise ↔ Jira: https://pitstop.manageengine.com/portal/en/kb/articles/two-way-integration-between-manageengine-servicedesk-plus-on-premise-and-jira
- Online API documentation: https://pitstop.manageengine.com/portal/en/kb/articles/online-api-documentation
- Contract details per asset via API: https://pitstop.manageengine.com/portal/en/kb/articles/to-pull-out-the-contract-details-to-which-an-asset-is-associated-through-api
- E-post mellan två SDP-system: https://pitstop.manageengine.com/portal/en/community/topic/mail-sent-from-two-different-sd-plus-systems-communication-breaks-down
- E-post från andra ITSM-system: https://pitstop.manageengine.com/portal/en/community/topic/how-are-others-handling-incoming-emails-messages-from-other-itsm-systems
- Trådning av inkommande e-post: https://pitstop.manageengine.com/portal/en/community/topic/how-servicedesk-plus-consider-incoming-emails-as-same-case
- Bilaga via API syns inte: https://pitstop.manageengine.com/portal/en/community/topic/sdp-ondemand-api-problem-attachment-successfully-added-mut-missing-from-request
- API Adding Attachments: https://pitstop.manageengine.com/portal/en/community/topic/api-adding-attachments
- SLA Timer (On Hold): https://pitstop.manageengine.com/portal/en/community/topic/sla-timer-on-hold-status-10-7-2020
- Swagger-dokumentation saknas: https://pitstop.manageengine.com/portal/en/community/topic/swagger-documentation-for-servicedesk-plus-needed
