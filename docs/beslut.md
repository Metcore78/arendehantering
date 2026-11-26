# Beslut och kända begränsningar

Bestående lärdomar från avslutade uppgifter. Detaljer finns i respektive dokument.

## Kända begränsningar i SDP (från uppgift 01, `sdp-api-rapport.md`)

- SDP On-Premise saknar inbyggd ärendeintegration mellan organisationer. Allt byggs som egen logik.
- Custom Triggers reagerar inte på anteckningar, konversationer eller bilagor.
- Webhooks har inga dokumenterade omförsök eller köer.
- SDP har ingen inkommande webhook. Inkommande går via REST API v3 med teknikerbunden authtoken utan scopes.
- UDF-fält heter `udf_sline_NNN` och skiljer mellan miljöer.
- Den publika on-prem-dokumentationen är ofullständig. Instansens inbyggda API-dokumentation är auktoritativ. Listan i rapportens avsnitt 7.2 är inte verifierad än.

## Arkitekturbeslut (förslag, från uppgift 02, `integrationslager-skiss.md`)

Besluten är skissens förslag och är inte förankrade hos Organisation A.

- **SDP är master.** Leverantörens ärende speglas som en uppgift per leverantörsärende.
- **Integrationslagret är en integration i en delad AIS-plattform** och återanvänder plattformens APIM, Service Bus, nätverk och övervakning.
- **Minimal logik i SDP.** SDP skickar tunna händelser (ärende-ID + händelsetyp) till APIM. Integrationslagret hämtar ärendet via API.
- **Kanonisk ärendemodell och publicera/prenumerera** på Service Bus, med en SDP-adapter och en adapter per leverantör.
- **Korsreferensregistret i integrationslagret** är källan för kopplingen SDP-ärende ↔ leverantörsärende. UDF i SDP är en kopia för teknikerna.
- **Integrationslagret pollar SDP** efter anteckningar och bilagor och stämmer av mot förlorade webhookar. Custom Schedules i SDP används inte.
- **Leverantörer anropar aldrig SDP.** De anropar APIM. En gemensam integrationstekniker i SDP räcker.
- **E-post mot leverantörer utan API** hanteras av integrationslagret via en delad brevlåda per leverantör, inte av SDP:s e-posthantering.
- **Ärenden stängs aldrig automatiskt i SDP** när leverantören löst sin del. IT verifierar.
- **Ekoskydd krävs**, eftersom integrationens egna skrivningar till SDP utlöser SDP:s triggers.

## Antaganden som bär skissen (ej bekräftade)

- Ärendedata får passera och tillfälligt lagras i Azure (EU-region).
- AIS-plattformen har privat förbindelse till on-prem-nätet där SDP står.
- Plattformen följer en typisk AIS-uppsättning (se skissens avsnitt 2).
