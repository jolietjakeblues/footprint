# Pand-oppervlakte-kaart

Eén statisch HTML-bestand (`pand-oppervlakte-kaart.html`) waarmee je:

- op adres zoekt en de footprint-oppervlakte van een pand berekent uit de BAG-geometrie
- op rijksmonumentnummer zoekt en direct naar het bijbehorende pand springt
- ziet of een pand een rijksmonument is (schildje), inclusief link naar het register
- een snelkoppeling krijgt naar het energielabel op EP-Online

Geen backend, geen build step, geen dependencies om te installeren. Alle logica draait client-side in de browser.

## Snel starten

Open het bestand lokaal in een browser, of host het als statisch bestand (GitHub Pages, Netlify Drop, Vercel, een eigen webserver). Er is geen server-side component nodig.

## Architectuur

```
adres/nummer invoer
        │
        ▼
┌───────────────────┐     ┌──────────────────────┐
│ PDOK Locatieserver │────▶│  BAG WFS (pand)      │
│ (geocoding)        │     │  service.pdok.nl     │
└───────────────────┘     └──────────────────────┘
        │                          │
        │                          ▼
        │                 shoelace-formule (JS)
        │                 → footprint m²
        │                          │
        ▼                          ▼
┌───────────────────┐     ┌──────────────────────┐
│ RCE linked data    │     │  RCE ps-ch WFS       │
│ (Speedy SPARQL)    │     │  (rijksmonumenten)   │
│ nummer → adres(sen)│     │  punt-in-polygon     │
└───────────────────┘     └──────────────────────┘
        │                          │
        └──────────┬───────────────┘
                    ▼
         kaartweergave (Leaflet) +
         resultaatpaneel + monumentschildje
```

## Databronnen en endpoints

| Bron | Endpoint | Gebruikt voor |
|---|---|---|
| PDOK Locatieserver | `api.pdok.nl/bzk/locatieserver/search/v3_1/free` | Adres → RD-coördinaat (autocomplete + geocoding). Ook gebruikt met `fq=adresseerbaarobject_id:{vbo-id}` om een BAG-verblijfsobject direct op te zoeken. |
| BAG WFS | `service.pdok.nl/lv/bag/wfs/v2_0` | Pandgeometrie ophalen via een bbox-query op `bag:pand`. Footprint wordt zelf berekend uit de polygon (shoelace-formule), niet uit een BAG-attribuut — dat bestaat niet op pandniveau. |
| RCE rijksmonumenten (WFS) | `service.pdok.nl/rce/ps-ch/wfs/v1_0` | Featuretype `rce_inspire_points`, gefilterd op `namespace='nlps-rijksmonumenten'`. Gebruikt om te bepalen of een pand een rijksmonument is, via de dichtstbijzijnde monumentpunt binnen de pand-bbox + marge. |
| RCE linked data (Speedy SPARQL) | `api.linkeddata.cultureelerfgoed.nl/datasets/rce/cho/services/cho/sparql` | Rijksmonumentnummer → alle bijbehorende adressen en BAG-verblijfsobject-ID's. Een monument kan meerdere verblijfsobjecten hebben (bijv. een pand met meerdere huisnummers); de query haalt ze allemaal op. |
| EP-Online | `www.ep-online.nl/Energylabel/Search` | Geen API-aanroep, alleen een uitgaande link met postcode+huisnummer alvast op het klembord. Zie "Waarom geen live energielabel" hieronder. |

Alle bovenstaande endpoints staan CORS open (`Access-Control-Allow-Origin: *`) en zijn getest vanuit een browsercontext.

### Waarom Speedy voor het rijksmonumentnummer, en niet de REST-query?

Er bestaat ook een vaste "rest-api-rijksmonumenten" query (`api.linkeddata.cultureelerfgoed.nl/queries/rce/rest-api-rijksmonumenten/run?rijksmonumentnummer=...`). Die retourneert altijd maar **één** BAG-relatie, ook als een monument er meerdere heeft. Voor monumenten met meerdere verblijfsobjecten (bijv. een pand gesplitst in meerdere huisnummers) gaf dat onvolledige of verkeerde resultaten. De Speedy SPARQL-query haalt daarom rechtstreeks alle `heeftBAGRelatie`-relaties op:

```sparql
PREFIX ceo: <https://linkeddata.cultureelerfgoed.nl/def/ceo#>
PREFIX graph: <https://linkeddata.cultureelerfgoed.nl/graph/>
SELECT ?datum ?adres ?vbo WHERE {
  GRAPH graph:instanties-rce {
    ?s ceo:rijksmonumentnummer "{nummer}" ;
       ceo:heeftBasisregistratieRelatie ?o .
    OPTIONAL { ?s ceo:datumInschrijvingInMonumentenregister ?datum }
    ?o ceo:heeftBAGRelatie ?bagrel .
    ?bagrel ceo:volledigAdres ?adres ;
            ceo:heeftVerblijfsobject ?vbo .
  }
}
```

Resultaten worden gededupliceerd op verblijfsobject-ID, want sommige objecten (bijv. ligplaatsen) hebben meerdere adresvarianten voor hetzelfde ID.

### Waarom het BAG-verblijfsobject-ID en geen adresstring

RCE-adresgegevens per BAG-relatie zijn niet altijd compleet: soms ontbreekt de straatnaam en/of postcode, en staat alleen een (soms merkwaardig geformatteerd) huisnummer erbij. Adresstrings zelf samenstellen en daarmee geocoderen bleek onbetrouwbaar.

In plaats daarvan gebruikt de app het BAG-verblijfsobject-ID rechtstreeks: PDOK Locatieserver indexeert dit veld als `adresseerbaarobject_id`, en filteren daarop (`fq=adresseerbaarobject_id:{id}`) geeft een exacte match, ongeacht hoe onvolledig de RCE-adresvelden zijn.

CQL_FILTER op de BAG WFS zelf (bijv. filteren op `identificatie`) bleek **niet betrouwbaar** — het endpoint negeert dat filter soms stilzwijgend en geeft een ongefilterde paginaresultaat terug. Vandaar de omweg via Locatieserver in plaats van een directe WFS-filter op ID.

## Belangrijkste berekeningen

**Footprint-oppervlakte**: shoelace-formule toegepast op de RD-coördinaten (EPSG:28992) van de pandgeometrie (`ringOppervlakte` / `berekenOppervlakte` in de code). RD is al een projected CRS in meters, dus geen omrekening nodig — in tegenstelling tot een aanpak met `turf.area()`, die geografische (WGS84) coördinaten verwacht en op RD-meters een verkeerde uitkomst zou geven.

**RD → WGS84**: alleen voor de kaartweergave, via `proj4.js` met de standaard RD-parameters (Bessel-ellipsoïde, 7-parameter Helmert-transformatie naar WGS84).

**Rijksmonument-detectie bij adreszoekopdrachten**: puntafstand tussen het pand-centroïde en het dichtstbijzijnde RCE-monumentpunt binnen de pand-bbox + 15 m marge. Als de afstand < 40 m is, wordt het als match beschouwd. Dit is een benadering: RCE-monumentpunten kunnen een paar meter van het BAG-pandcentroïde afwijken door verschillen in geocodering.

## Bekende beperkingen

- **Footprint ≠ gebruiksoppervlakte.** De grote cijferweergave is de voetprint van het gebouw (uit de geometrie), niet de NEN2580-gebruiksoppervlakte die in BAG per verblijfsobject staat. Beide worden getoond, expliciet naast elkaar.
- **40 m-marge voor monumentdetectie bij adreszoekopdrachten** kan in dichtbebouwde binnensteden een naastgelegen pand als monument aanmerken, of een echt monument missen. Bij zoeken op rijksmonumentnummer speelt dit niet: daar is de monumentstatus al zeker.
- **Geen live energielabel.** EP-Online's publieke API vereist een API-key. Omdat deze app volledig client-side draait, zou een ingebakken key voor iedereen zichtbaar zijn via "bekijk paginabron" — dat is een beveiligingsrisico voor de key-eigenaar. In plaats daarvan opent de app EP-Online's zoekpagina met postcode+huisnummer alvast op het klembord.
- **Enkele monumenten hebben geen BAG-koppeling** (met name archeologische monumenten en sommige ligplaatsen). Die tonen een doorkliklink naar het register in plaats van een kaartresultaat.
- **Geen rate limiting / caching.** Bij veel gelijktijdig gebruik kunnen PDOK- of RCE-endpoints throttelen. Voor breed/publiek gebruik: navragen bij de bronhouders wat een verantwoorde gebruikslast is.

## Attributie

Bij gebruik van de rijksmonumentendata: naamsvermelding aan de Rijksdienst voor het Cultureel Erfgoed (RCE) verplicht (CC-BY). BAG-data via PDOK/Kadaster, eveneens open data met bronvermelding.

## Techstack

Eén HTML-bestand, geen bundler. Externe libraries via CDN (cdnjs):

- [Leaflet](https://leafletjs.com/) 1.9.4 — kaartweergave
- [proj4js](http://proj4js.org/) 2.9.2 — RD ↔ WGS84 transformatie
- [Turf.js](https://turfjs.org/) 6.5.0 — alleen `booleanPointInPolygon` (planaire test, werkt ook correct op RD-coördinaten; `turf.area`/`turf.distance` worden bewust *niet* gebruikt omdat die geografische aannames maken die op RD-meters verkeerde uitkomsten geven)

Lettertype: IBM Plex Sans / IBM Plex Mono (Google Fonts).