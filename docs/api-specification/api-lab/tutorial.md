
# Tutorial notificeren

In deze tutorial configureren we de referentieimplementaties van de algemene voor het versturen en ontvangen van notificaties via de generieke 
Notificaties API.

De tutorial is hands-on - onderaan vind je verdere referenties en bronnen indien je meer wil lezen.

De volgende componenten zijn meest relevant:

* NRC: voor het routeren van notificaties, kanaal- en abonnementbeheer
* PRC: voor het versturen van notificaties


## Wat zijn de vereisten voor deze tutorial?

* API-key voor authorisatie

* `optioneel`
  * `docker` en `docker-compose` om lokaal op je (ontwikkelmachine) de componenten te hosten. Tijdens het API-Lab kan de installatie en werking van het docker-image niet ondersteund worden.

* Familiariteit met webhooks is een plus

## Aan de slag

### Ontvangen en versturen van notificaties

Het PRC verstuurt notificaties naar het NRC. Het NRC distribueert deze vervolgens naar de abonnees.

De notificaties zijn inzichtelijk gemaakt op het NRC - ga naar [https://notificaties-api.test.vng.cloud/] en klik op de knop 'Logviewer'.

Er zijn twee perspectieven:

* [Notificaties publiceren](#ik-wil-als-bron-notificaties-publiceren)
* [Notificaties ontvangen](#ik-wil-als-consumer-notificaties-ontvangen)

#### Ik wil als bron notificaties publiceren

Het registreren van de domeinen is een noodzakelijke stap om notificaties te kunnen publiceren.

Eenvoudigweg operaties uitvoeren op de PRC API zal ervoor zorgen dat notificaties gepubliceerd worden. Je kan bijvoorbeeld via de API een zaak
aanmaken, wijzigen of statussen toevoegen op een zaak om dit in actie te zien. De token om in het API-Lab te gebruiken, zal worden verspreid en biedt alle autorisaties.

1. Bepaal de naam van het domein. Voor bijvoorbeeld zaken is dit `nl.vng.zaken`.

2. Zorg dat het domein bekend is bij het NC. Je kan dit controleren door eerst de lijst met domeinen op te vragen:
 
   ```http
   GET https://notificaties-api.test.vng.cloud/api/v1/domains HTTP/1.0
   Authorization: Bearer abcd1234
   ```
   Het resultaat ziet er als onderstaand uit:
   
   {
        "name": "nl.vng.zaken",
        "documentationLink": "https://github.com/VNG-Realisatie/zaken-api/",
        "filterAttributes": [],
        "url": "https://notificaties-api.test.vng.cloud/api/v1/domains/3afb22e1-92f8-469f-9b50-35748642c82b"
    }
   
   Als het domein bestaat, kun je aan de hand van de UUID van het domein, de details van het domein opvragen (in dit geval zou dat dan 3afb22e1-92f8-469f-9b50-35748642c82b zijn)
   
   ```http
   GET https://notificaties-api.test.vng.cloud/api/v1/domains/UUID HTTP/1.0
   Authorization: Bearer abcd1234
   ```

3. Registreer het domein (indien het nog niet bestaat)

    ```http
    POST https://notificaties-api.test.vng.cloud/api/v1/domains HTTP/1.0
    Authorization: Bearer abcd1234
    Content-Type: application/json

    {
      "naam": "nl.vng.zaken",
      "documentatieLink": "https://github.com/VNG-Realisatie/zaken-api/",
      "filters": [
        "bronorganisatie",
        "zaaktype",
        "vertrouwelijkheidaanduiding"
      ]
    }
    ```

    De documentatielink hoort te documenteren welke kenmerken relevant zijn
    voor een domein, en welke events gepubliceerd worden. Dit helpt consumers
    om te bepalen waarop ze willen abonneren.

4. Verstuur een bericht

    ```http
    POST https://ref.tst.vng.cloud/nrc/api/v1/notificaties HTTP/1.0
    Authorization: Bearer abcd1234
    Content-Type: application/json

    {
      "kanaal": "zaken",
      "hoofdObject": "https://ref.tst.vng.cloud/zrc/api/v1/zaken/ddc6d192",
      "resource": "status",
      "resourceUrl": "https://ref.tst.vng.cloud/zrc/api/v1/statussen/44fdcebf",
      "actie": "create",
      "aanmaakdatum": "2019-03-27T10:59:13Z",
      "kenmerken": {
        "bronorganisatie": "224557609",
        "zaaktype": "https://ref.tst.vng.cloud/ztc/api/v1/catalogussen/39732928/zaaktypen/53c5c164",
        "vertrouwelijkheidaanduiding": "openbaar"
      }
    }
    ```

#### Ik wil als consumer notificaties ontvangen

Je dient de scope `notificaties.scopes.consumeren` in het JWT te hebben
voor deze acties. Je kan de [tokentool][token-generator] gebruiken om een
JWT te genereren.

1. Voorzie een endpoint om notificaties te ontvangen. Een eenvoudige manier om
   deze te inspecteren is met de [webhook-site](https://webhook.site). Voor het
   vervolgen gebruiken we `https://webhook.site/ea216914-fc38-462e-a24c-7dc7e969d873`
   als voorbeeld-URL waarop notificaties bezorgd worden.

2. Vraag op welke kanalen beschikbaar zijn:

   ```http
   GET http://<nrc-ip>:8004/api/v1/kanaal HTTP/1.0
   Authorization: Bearer abcd1234
    ````

3. Registreer je abonnement bij het NRC:

   ```http
   POST http://<nrc-ip>:8004/api/v1/abonnement HTTP/1.0
   Authorization: Bearer abcd1234
   Content-Type: application/json

   {
     "callbackUrl": "https://webhook.site/ea216914-fc38-462e-a24c-7dc7e969d873",
     "auth": "dummy",
     "kanalen": [
       {
         "naam": "zaken",
         "filters": {
           "bronorganisatie": "224557609"
         }
       }
     ]
   }
   ```

    * `callbackUrl` is de volledige URL naar je _eigen_ endpoint waar je
      notificaties wenst op te ontvangen

    * `auth` is de waarde van de `Authorization` header om je _eigen_ endpoint
      te kunnen benaderen. Deze waarde wordt gebruikt door het NRC om berichten
      af te leveren. Voor `webhook.site` kan een dummy waarde gebruikt worden.

    * `kanalen` is een lijst van kanalen waarop je wenst te abonneren, met de
      relevante filters. De beschikbare kenmerken waarop gefilterd kan worden
      horen gedocumenteerd te zijn op de kanalen die opgevraagd zijn in stap 2.

    * `filters` zijn optioneel. Indien je een filter weglaat, dan geldt dit als
      wildcard.

4. Berichten worden nu naar je eigen endpoint gestuurd met een POST request

    Hieronder staat een verzoek zoals dat gedaan wordt door het NRC. Je kan dit
    verzoek uiteraard ook zelf sturen voor test doeleinden:

   ```http
   POST https://webhook.site/ea216914-fc38-462e-a24c-7dc7e969d873 HTTP/1.0
   Content-Type: application/json
   Authorization: Token abcde12345

   {
     "kanaal": "zaken",
     "hoofdObject": "https://zaken-api.vng.cloud/api/v1/zaken/ddc6d192",
     "resource": "status",
     "resourceUrl": "https://zaken-api.vng.cloud/api/v1/statussen/44fdcebf",
     "actie": "create",
     "aanmaakdatum": "2019-03-27T10:59:13Z",
     "kenmerken": {
       "bronorganisatie": "224557609",
       "zaaktype": "https://catalogi-api.vng.cloud/api/v1/zaaktypen/53c5c164",
       "vertrouwelijkheidaanduiding": "openbaar"
     }
   }
   ```

    Merk op dat de `Authorization` header hier verschilt van de `Authorization`
    naar het NRC. De notificatie wordt naar jouw eigen endpoint verstuurd,
    en bij het abonneren heb je aangegeven wat de `Authorization` header
    hiervoor moet zijn.

[token-generator]: https://zaken-auth.vng.cloud


## Achtergrondinformatie

De [technische notificaties achtergrond](/gemma-zaken/themas/achtergronddocumentatie/notificaties) bevat de
design-standpunten en onderkent de limitaties en risico's van deze aanpak.

[Hier](./_assets/notificeren.pptx) is de presentatie te vinden die gegeven is op het
API-lab.