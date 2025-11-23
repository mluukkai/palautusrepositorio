# Vibe-koodaus

Jostain syystä copilot bugitti siten, että ensimmäisellä yrityksellä se ei luonut UI:ta tuntien odotuksesta huolimatta. Toisella yrityksellä se loi UI:n, mutta ei generoinut muutoksia reviewn jälkeen. Kolmannella yrityksellä kaikki toimi. 

## Päätyikö Copilot toimivaan ja hyvään ratkaisuun
Copilot sai aikaan toimivan ratkaisun, jonka workflow oli melko selkeä. UI:n ulkoasu oli kuitenkin melko alkeellinen. Lomakenäkymässä näytettiin myös liikaa asioita eri väreillä ja se vaikutti sekavalta. Värit olivat räikeitä ja hyppäsivät silmille. UI-painikkeet olivat myös pieniä ja hankalasti käytettäviä. En ole varma olisiko tämä uskottava myyntidemo eli ulkoasua ja käytettävyyttä ainakin pitäisi viilata. UI:sta ei löytynyt Axe-corella kriittisiä käytettävyysongelmia, mutta tason serious ja moderate oli useampia. Review-pyynnöistä huolimatta ulkoasun tyyli ei muuttunut parempaan suuntaan.
Dependency injectionia copilot ei saanut tehtyä kovin helposti. Jouduin ohjeistamaan useiden PR review-pyyntöjen kautta. Eri kokeilujen myötä opin, että promptien ei kannata olla kohteliaita ja laajasti kirjoitettuja pyyntöjä vaan niitä kannattaa kirjoittaa mahdollisimman tiiviisti ja imperatiivimuodossa. Lisäksi opin, että jos kovin monta kertaa pyytää muutoksia, copilot palauttaa helposti aiempia jo kertaalleen korjattuja ratkaisuja alkuperäiseen tilaan. Loppujen lopuksi vibekoodaukseen kulunut aika kaiken säädön jälkeen oli huomattavasti hitaampaa kuin jos olisin alusta asti koodannut kaiken itse. Lisäksi generoitu koodi ei ole kuitenkaan ihan niin tuttua kuin itse koodattu ja jotkut logiikkavirheet tai bugit saattoivat jäädä huomaamatta. Hyvin pienessä mittakaavassa tästä voisi olla hyötyä myös työprojekteissa valmiin boilerplaten luomisessa.

## Oliko koodi selkeää
Koodi oli suhteellisen selkeää, mutta toisteista. Funktiot olivat pitkiä ja ne tekivät turhat monia asioita (single responsibility principleä ei siis noudatettu). Eli oli melko paljon ns. code smelliä. Muuttujien ja luokkien nimet olivat kuitenkin melko selkeitä. 
Copilot luoma warehouse_service-luokka toimi siten, että se loi objektit seuraavasti: 
````
self.warehouses[warehouse_id] = {
    'id': warehouse_id,
    'name': name,
    'varasto': Varasto(tilavuus, alku_saldo)
}
````
Se ei siis edes kapseloinut olioita. 
Reviewissä ehdotin erillisen Warehouse-olion luomista ja sen injektoimista parametrina WarehouseServicelle. 


Annetuista toiveista huolimatta copilot halusi luoda warehouse-objekteja funktioiden sisällä.  
```` 
def create_warehouse(self, name, tilavuus, saldo=0):
        """Create a new warehouse with given name, capacity and initial balance.

        Args:
            name: Name of the warehouse
            tilavuus: Capacity of the warehouse
            saldo: Initial balance (default 0)

        Returns:
            The ID of the created warehouse
        """
        warehouse_id = self._next_id
        self._next_id += 1

        warehouse = Warehouse(warehouse_id, name, tilavuus, saldo)
        self._warehouses[warehouse_id] = warehouse

        return warehouse_id
````

Tein vielä yhden review-kierroksen, missä pyysin copilotia käyttämään DI:tä. 
Tällä kertaa se tekikin näin, mutta loi väliin turhan Warehouse-wrapperin Varasto-oliolle.

Tein vielä yhden interaation copilotin kanssa ja pyysin sitä käyttämään suoraan olemassa olevaa Varasto-oliota Warehouse-olion sijaan. Lopputulos oli tämä:
````
def create_warehouse(self, name, varasto):

    warehouse_id = self._next_id
    self._next_id += 1 

    self.warehouses[warehouse_id] = {
        'id': warehouse_id,
        'name': name,
        'varasto': Varasto(tilavuus, alku_saldo)
    }
````

## Opitko jotain uutta Copilotin tekemää koodia lukiessasi

Copilotin koodi oli toimivaa, mutta sitä oli melko raskasta lukea, koska funktiot tekivät monia asioita. Lisäksi koodi oli heikosti testattavaa eikä copilot osannut yhdistää suomenkielistä koodia englanninkieliseen koodiin ellei sille speksannut vastaavuuksia. Sillä oli myös haasteita ulkoasun tyylien suunnittelussa, kapseloinnissa ja dependency injectionin toteuttamisessa. 

Tietoturvariskejä löytyi myös jonkin verran. Ensinnäkin app-pyöri debug-tilassa, koodissa ei tarkistettu sessionia, poluilla ei ollut CSRF-suojausta, View_warehouse palauttaa varaston rakenteen ilman sanitointia ja suodatusta. Tähän kohtaan kannattaisi lisätä Dto-luokka ja escapettaa tekstikentät. 

Olisi mielenkiintoista nähdä millä tavalla copilot rakentaa tietokannan ja repositoryt ja tekisikö se sellaisia aloittelijan virheitä, kuten sql-injektion. Myöskään minkäänlaista autentikointia/autorisointia ei ollut toteutettu eli sekin pitää osata pyytää ja speksata tarkasti sekä yksinkertaisesti. DoS-suojaus myös puuttui eli tätä API:a olisi mahdollista pommittaa huoletta. 

Olisi mielenkiintoista myös testata copilotia siten, että pyytäisi sitä suojaamaan sovelluksen vähintään OWASP Top 10 mukaisesti ja yrittää löytää sovelluksesta sen jälkeen haavoittuvuuksia.
