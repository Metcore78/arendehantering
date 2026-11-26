# Ärendehanteringsystem

# Instruktioner för Claude
Utför uppgifterna nedan. Utgå från projektkontexten tillsammans med de arbete som gjorts tidigare (rapporter, dokumentation, kod etc.) och relevant research som behöver göras för att utföra uppgiften. Modifiera tidigare arbetet om det under utförande av en ny uppgift upptäcker felaktigheter eller ändrade förutsättningar från tidigare arbete. Om ändringar görs i tidigare arbete delge detta tydligt när uppgiften är slutförd.

## Hur uppgifter ges i detta repo
- Uppgifter ges antingen direkt i prompten (webbgränssnittet) eller som
  md-filer under `docs/uppgifter/`.
- Avsnittet **Aktuell uppgift** nedan pekar på den uppgift som ska utföras,
  eller innehåller ordet `Ingen`.
- Avsnittet **Avslutade uppgifter** finns av historiska skäl, för att ge
  kontext. Utför aldrig något därifrån.

## Vad du ska göra vid sessionsstart
1. Om prompten innehåller en uppgift: utför den. Prompten går före
   Aktuell uppgift.
2. Annars, om Aktuell uppgift pekar på en uppgiftsfil: läs den och utför den.
3. Annars (Aktuell uppgift är `Ingen` och prompten saknar uppgift):
   gör ingenting. Ändra, skapa eller committa inga filer. Svara kort att
   det inte finns någon uppgift och avsluta.

## När en uppgift är klar
- Arkivera uppgiften i mappen "uppgifter" som en markdown fil döpt enligt konventionen nedan. Sammanfatta hur uppgiften utfördes och hur resultatet av det arbetet sparats i projektet. Ta även med de ändringar som gjorts i tidigare arbete.
- Flytta raden från Aktuell uppgift till Avslutade uppgifter och markera
  den (klar). Sätt Aktuell uppgift till `Ingen`.
- För över bestående lärdomar till `docs/beslut.md`.

# Projektkontext
## Syfte
Integration mot ServiceDesk Plus (SDP) för där Organisation A ska byta ut sitt nuvarande ärendehanteringssystem mot SDP, vilket innebär ombyggnad av befintliga integrationer. Det kommer vara en on-prem lösning. Systemet ska användas av organisationens IT-avdelnings service desk, och det kommer användas för att interagera med leverantörer till IT-avdelningen via ärenden. Service desk har då behov av att kunna skapa och modifiera ärenden i dess leverantörers ärendehanteringssystem i de fall där IT-avdelningen behöver hjälp av leverantören att åtgärda hela eller delar av det problem som är orsak till att Organisation A:s verksamhet skapade ärendet hos dess IT-avdelning. Det kan vara organisationens leverantör av hårdvara (servrar, laptops, plattor etc. med tillhörande grundläggande mjukvara (OS, Office etc)) eller organisationens systemleverantörer eller dess leverantör av telefonutrustning etc. 


## Viktig kontext
- En analys av SDP:s API finns i `docs/sdp-api-rapport.md`. Läs den innan du
  arbetar med något som rör API:t. Men behövs ytterligare information finns SDP API dokumentation öppet på nätet.
- Beslut och kända begränsningar finns i `docs/beslut.md`.

## Aktuell uppgift
Ingen

## Avslutade uppgifter
- 01: API-rapport (klar) → `docs/uppgifter/01-api-rapport.md`
- 02: Skissa på hur integrationslagret skulle se ut. Antag att integrationslagret kommer vara en del i en generell integrationsplattform baserad på Azure Integrations Service (AIS) som hanterar flera andra integrationer mot andra system (som inte är kopplade till ärendehantering) och som hanterar andra integrationsmönster än just detta. Utgå från en antagen AIS arkitektur, så som en AIS plattform vanligen är upplagd. (klar) → `docs/uppgifter/02-Skiss-Integrationslager.md`, resultat i `docs/integrationslager-skiss.md`

## Konventioner
- Döp slutförda uppgifter enligt: Löpnummer-BeskrivandeOrd-BeskrivandeOrd.md (ex: 02-Stubba-Integration.md), upptill 3 beskrivande ord.
