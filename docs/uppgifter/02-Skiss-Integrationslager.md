# 02 – Skiss på integrationslagret (AIS)

## Uppgiften

Skissa på hur integrationslagret skulle se ut. Antag att integrationslagret kommer vara en del i en generell integrationsplattform baserad på Azure Integrations Service (AIS) som hanterar flera andra integrationer mot andra system (som inte är kopplade till ärendehantering) och som hanterar andra integrationsmönster än just detta. Utgå från en antagen AIS arkitektur, så som en AIS plattform vanligen är upplagd.

## Hur uppgiften utfördes

- Utgick från `docs/sdp-api-rapport.md`: användningsfallen UC1–UC12, bristerna B1–B14 och den rekommenderade arkitekturen i avsnitt 7.1.
- Antog en AIS-plattform enligt Microsofts referensarkitekturer: hub-and-spoke med privat förbindelse till on-prem, delat API Management bakom Application Gateway/WAF, delat Service Bus, Logic Apps Standard och Functions, Key Vault, Log Analytics/Application Insights.
- Ingen ny verifiering gjordes mot SDP eller mot en verklig plattform. Skissen bygger på antaganden som listas i dess avsnitt 2 och öppna frågor i avsnitt 11.

## Resultat

Skissen finns i `docs/integrationslager-skiss.md`. Den innehåller antagen plattform, designprinciper, översiktsbild, komponenter (delade och egna), köer och ämnen, korsreferensregister, ekoskydd, flöden per användningsfall, utkast till kanonisk ärendemodell, nätverk och säkerhet, felhantering och övervakning, etappindelning, öppna frågor samt en tabell över hur varje brist B1–B14 hanteras.

## Ändringar i tidigare arbete

`docs/sdp-api-rapport.md`, avsnitt 7.1: en hänvisning till skissen har lagts till, med de tre punkter där skissen preciserar rapportens rekommendation:

1. Pollning efter anteckningar och bilagor görs av integrationslagret, inte med Custom Schedules i SDP.
2. E-post till och från leverantörer utan API går via integrationslagrets brevlådor, inte via SDP:s e-posthantering.
3. En gemensam integrationstekniker används i stället för en per leverantör, eftersom leverantörer aldrig anropar SDP direkt.

Rapportens övriga innehåll är oförändrat.

## Lärdomar

Överförda till `docs/beslut.md`.
