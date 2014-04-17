#1 Elasticsearch

Elasticsearch predstavlja fleksibilan, **open-source** , distribuiran **engine** za ril-tajm   
pretragu i analizu podataka.

Ovo je najkraca formulacija onoga sto elasticsearch stvarno jeste i koje funkcionalnosti pruza. 
Prvo cemo proci kroz instalaciju elasticsearch-a.

Treba dodati i reci da u srzi elasticsearch-a se nalazi software projekat pod imenom Lucene.  
Tako da elasticsearch u osnovi je najlakse shvatiti kao dio infrastrukture izgradjen oko  
Lucene engine-a koji kao zasebna cjelina nece biti predmet ovog tutoriala.

###1.2 Instalacija  

Da bi instalirali elasticsearch potrebno je instalirati java-u

sudo apt-get update  
sudo apt-get install openjdk-7-jre-headless -y

I sledece dvije komande da bi skinuli i instalirali elasticsearch

wget https://download.elasticsearch.org/elasticsearch/elasticsearch/elasticsearch-1.1.0.deb  
sudo dpkg -i elasticsearch-1.1.0.deb

Sada imamo instaliran elasticsearch i server mozemo pokrenuti sljedecom komandom:  

sudo /etc/init.d/elasticsearch start

Ova komanda ima jos i opcije stop , restart , force-reload i status.  

Nakon pokretanja servera mozemo provjeriti da sve radi kako treba tako sto u browser-u  
posjetimo http://localhost:9200 . Kao odgovor bi trebali da vidimo stranicu koja izgleda  
slicno navedenom primjeru:  

    {
      "ok": true,
      "status": 200,
      "name": "Psyche",
      "version": {
        "number": "0.90.0",
        "snapshot_build": false
      },
      "tagline": "You Know, for Search"
    }

Prije nego pocnemo sa samom strukturom podataka i komandi u elastisearch-u par rijeci o konfiguracionim fajlovima.

Nezavisno od toga koji smo paket skinuli (u nasem primjeru debian paket), svaki paket dolazi  
sa default konfiguracionim fajlom koji se nalazi u /etc/default/elasticsearch. U ovom  
fajlu se nalaze podesavanja a parametre o kojima nesto vise mozete procitati (kao i potpun  spisak parametara) na sljedecem [linku](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/setup-service.html)

Sto se tice podesavanja za elastic search u folderu /etc/elasticsearch se nalaze 2 fajla  
elasticsearch.yml(za podesavanje razlicitih ES modula) i logging.yml( za podesavanje ES logginga). Detaljnije o ovim podesavanjima na [linku](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/setup-configuration.html)  

###1.3 Struktura podataka u Elasticsearch-u

Osnovna jedinica podataka u elasticsearch-u je polje (field). Polje ima definisan tip i moze  
sadrzati jednu ili vise vrijednosti tog tipa. 

Dokument (Document) je kolekcija polja i predstavlja osnovnu jedinicu skladistenja u  
elasticsearch-u.

###1.3.1 JSON, jezik Elasticsearch-a

Primarni format za podatke koji koristi elasticsearch je JSON. Jednostavan dokument izgleda  
kao u navedenom primjeru:

    {
      "_id": 1,
      "handle": "ron",
      "age": 28,
      "hobbies": ["hacking", "the great outdoors"],
      "computer": {"cpu": "pentium pro", "mhz": 200}
    }

Vidimo iz primjera da ovaj dokument sadrzi i slozene tipove kao sto su niz za "hobbies"  
i **object** dok su ostala polja string i numericke vrijednosti.

Elasticsearch rezervise odredjena polja za specijalnu upotrebu. Iz prethodnog primjera se vidi  
polje _id. id dokumenta je jedinstven i po automatizmu se kreira.

###1.3.2 Osnove o tipovima

Svaki dokument u elasticsearch-u mora odgovarati nekom korisnicki definisanom mapiranju tipa  (type mapping) analogno shemi baze podataka. Mapiranje tipa definise tip polja (integer,string)  
Treba napomenuti da vise dokumenata u indexu mogu imati iste id-eve dok su razlicitog tipa.
Tipovi su definisani sa Mapping API-jem koji pridruzuje imena tipova sa definicijom property-ja.

    {
      "user": {
          "properties": {
              "handle": {"type": "string"},
              "age": {"type": "integer"},
              "hobbies": {"type": "string"},
              "computer": {
                  "properties": {
                      "cpu": {"type": "string"},
                      "speed": {"type": "integer"}}}}}}

Svako polje u property sekciji mapiranja je povezan sa nekim osnvonim(core) tipom. Slijedi  
spisak **core** tipova u elasticsearch-u.

Type  -----  	Definition  
string ------- Text  
integer ------ 32 bit integers  
long ------- 64 bit integers  
float -------	IEEE float  
double ------	Double precision floats  
boolean -------	true or false  
date 	UTC ------ Date/Time (JodaTime)  
geo_point -------	Latitude / Longitude  

Kompleksni JSON tipovi su takodje podrzani u upotrebi je notacija niza i objekta. Dodatno jos  
elasticsearch dokumenti mogu da upravljaju slozenim parent/child relacijama kao i specijalnim  
**nested** tipovima dokumenata.

Sto se tice nizova, jednostavno su realizovani u elasticsearch-u. Kao sto je vec receno svako  
polje u dokumentu moze sadrzat jednu ili vise vrijednosti. Nije potrebno deklarisati poseban  
tip za nizove u mapiranju. Jedino treba zapamtiti da nizovi u elasticsearchu ne mogu sadrzati  
razlicite tipove vrijednosti(ako je polje deklarisano kao integer moguce je skladistiti jedan  
ili vise (niz) integera nikako mix tipova)

U prethodnom primjeru mapiranja vidimo da je moguce opisati objekte koji sadrze druge objekte  
kao sto je **computer** polje u konkretnom primjeru. Kljucna stvar koju treba zapamtiti je  
da su njihove osobine sadrzane u **properties** polju u mapiranju i da se ne navodi njegov tip.

###1.3.3 Indexi, osnove 

Najveca jedinica podataka u elasticsearch-u je **index**.Indexi su logicke i fizicke particije  
dokumenata. Dokumenti i tipovi dokumenata su jedinstveni na nivou indexa. Indexi nemaju znanje  
o podacima sadrzanim u drugim indexima.

Svaki index samim kreiranjem definise broj shard-ova na koji se dijeli i koji je po default-u  
5(primary) i 1 replica shard. Shard predstavlja neki oblik namespace-a. Osim  mogucnosti definisanja broja shardova koje index treba da ima, direktno shardove ne treba referencirati. Elasticsearch vrsi distribuciju shardova medju svim cvorovima(nodes) u klasteru,
i moze da pomjera shardove iz jednog cvora u drugi automatski u slucaju otkaza nekog cvora.

Svak dokument je smjesten u primarni shard. Kada indexiramo dokument, indexiran je prvo u primarni shard pa onda u sve replica shardove primarnog sharda.

Svaki primarni shard ima 0 ili vise replica. Replica je kopija primarnog sharda i ima 2 svrhe:

+ Ako primarni otkaze replica moze biti promovisana u primarni
+ Povecanje performansi: get ili search zahtjevima se moze rukovati od strane primarnog  
ili replica sharda.Po default-u svaki primarni shard ima jedan replica shard, ali broj replica shardova se moze dinamicki mijenjati na postojecem indexu. Replica shard nikad nece biti startovan na istom cvoru kao primarni shard

Nakon definisanja svih potrebnih pojmova mozemo obnoviti analogiju sa bazama podataka  
radi lakseg shvatanja.

Index predstavlja bazu podataka, tip predstavlja tabelu u bazi, dokument predstavlja jedan  
zapis.

###1.4 Komande 

U ovoj sekciji cemo reci nesto vise o komandama pomocu kojij mozemo kreirati, brisati,  
dohvatati indexe tipove i dokumente u njima.

Slijedi primjer ilustrativne prirode par jednostavnih komandi

    // Create a type called 'hacker'
    PUT /planet/hacker/_mapping
    {
      "hacker": {
        "properties": {
          "handle": {"type": "string"},
          "age": {"type": "long"}}}}

    // Create a document
    PUT /planet/hacker/1
    {"handle": "jean-michel", "age": 18}

    // Retrieve the document
    GET /planet/hacker/1

    // Update the document's age field
    POST /planet/hacker/1/_update
    {"doc": {"age": 19}}

    // Delete the document
    DELETE /planet/hacker/1

    // Create an index named 'planet'
    	PUT /planet


Na ovom primjeru mozemo vidjeti kreiranje indexa, kreiranje dokumenta u indexu sa  
odgovarajucim tipom kao i akcije za dohvatanje, brisanje i update dokumenata, kao  
i na pocetku primjera prikaz mapping komande tj. kreiranje mapiranja za odgovarajuci  
tip. Detaljniji primjeri komandi u nastavku.

Radi lakseg koriscenja komandi koje slijede preporucujem upotrebu alata koji pruza graficki  
interfejs i pruza znatne olaksice u odnosu na terminal i mozete ga naci na sledecem linku  
[ElasticHammer](http://elastichammer.exploringelasticsearch.com/)

Komande ce biti u originalnoj sintaksi i rasporedjeni po tome sto je glavni predmet komande.

Osnovna komanda za dohvatanje dokumenta  

    curl -XGET 'http://localhost:9200/twitter/tweet/1'

Rezultat ove komande bi bio 

    {
        "_index" : "twitter",
        "_type" : "tweet",
        "_id" : "1",
        "_version" : 1,
        "found": true,
        "_source" : {
            "user" : "kimchy",
            "postDate" : "2009-11-15T14:12:12",
            "message" : "trying out Elasticsearch"
        }
    }

Ova komanda dohvata JSON dokument iz indexa twitter, tipa tweer sa id-jem 1.  
Ovaj tip komande je pod okriljem GET API-ja koji je zaduzen za dohvatanje dokumenata  
po njihovim id-u.
Ovaj API dozvoljava i provjeru egzistencije dokumenta koriscenjem HEAD kao npr:

    	curl -XHEAD -i 'http://localhost:9200/twitter/tweet/1'

Da bi kreirali dokument u nekom indexu koristimo komandu prikazanu sljedecim primjerom:

    curl -XPUT 'http://localhost:9200/twitter/tweet/1' -d '{
        "user" : "kimchy",
        "post_date" : "2009-11-15T14:12:12",
        "message" : "trying out Elasticsearch"
    }'

Rezultat ove komande 

    {
        "_index" : "twitter",
        "_type" : "tweet",
        "_id" : "1",
        "_version" : 1,
        "created" : true
    }


Sledeca komanda vrsi brisanje JSON dokumenta iz indexa twitter pod tipom tweet sa id-jem  1:

    curl -XDELETE 'http://localhost:9200/twitter/tweet/1'

Moguce je vrsiti update nad odredjenim dokumentima. To se realizuje sljedecom komandom:  

    curl -XPOST 'localhost:9200/test/type1/1/_update' -d '{
        "script" : "ctx._source.counter += count",
        "params" : {
            "count" : 4
        }
    }'


Sada cemo nesto vise reci o samom kreiranju indexa i navesti propratne komande.  

    curl -XPUT 'http://localhost:9200/twitter/'

Ovo je osnovna komanda za kreiranje indexa, medjutim mogu se definisati dodatne opcije  
kao npr. broj shardova i replica shardova:


    curl -XPUT 'http://localhost:9200/twitter/' -d '
    index :
        number_of_shards : 3
        number_of_replicas : 2
    '
Ovo definisanje opcija je upotreba YAML, takodje se mogu definisati opcije upotrebom JSON:

    curl -XPUT 'http://localhost:9200/twitter/' -d '{
        "settings" : {
            "index" : {
                "number_of_shards" : 3,
                "number_of_replicas" : 2
            }
        }
    }'


Index se moze izbrisati sljedecom komandom:

    curl -XDELETE 'http://localhost:9200/twitter/'

ova komanda vrsi brisanje indexa twitter. Moguce je izbrisati vise od jednog indexa  
kao i sve indexe stavljanjem _all na mjesto imena indexa.

Mapiranje tipa je moguce definisati na sljedeci nacin  

    curl -XPUT 'http://localhost:9200/twitter/tweet/_mapping' -d '
    {
        "tweet" : {
            "properties" : {
                "message" : {"type" : "string", "store" : true }
            }
        }
    }
    '

Moguci su konflikti ako vec postoji mapiranje za odredjeni tip, te dvije definicije,  
prva vec definisana i nova definicija se merge-uju. 
U ovom primjeru prikazano je kreiranje mapiranja(type) tweet u twitter indexu.
Odredjeno mapiranje je moguce dohvatiti komandom:

    curl -XGET 'http://localhost:9200/twitter/tweet/_mapping'

Ako zelimo da dohvatimo sva mapiranja svih indexa i tipova sljedece dvije komande su  
ekvivalentne: 

    curl -XGET 'http://localhost:9200/_all/_mapping'

    curl -XGET 'http://localhost:9200/_mapping'

I na kraju primjer komande za brisanje mapiranja:

    curl -XDELETE 'http://localhost:9200/{index}/{type}/_mapping'


###1.5 Search 

Search API se koristi dodavanjem _search na kraju komande(putanje u komandi) i to na /index  
/index/type putanje. Za pretragu na nivou indexa bi izgledalo /my_index/_search, a za odredjeni 
tip dokumena /my_index/my_type/_search. Posao Search API-ja je da pokrene upit sa odgovarajucim 
aprametrima. Takodje postoji mogucnost Faceting-a i Filtering-a o cemu ce biti rijeci.

Slijedi prikaz jednostavnog upita koji ima samo size i query parametre postavljene, bez facet-a 
i filter-a. _search radi sa GET i POST HTTP metodima.

    {
      "size": 3,
        "query": {
            "match": {"hobbies": "skateboard"}
         }}

Osim odredjivanja koji dokumenti zadovoljavaju odredjeni kriterijum, vrsi se i uredjenje po  
njihovom **similarity-ju** koji predstvarlja mjeru koja odredjuje koliko je dobro odredjen  
dokument zadatim upitom i ova vrijednost se oznacava kao **score**.

Slijedi primjer rezultata jednog jednostavnog upita u elasticsearch-u.

    {
      "took": 2, "timed_out": false,
      "_shards": {"total": 5, "successful": 5, "failed": 0},
      "hits": {
        "total": 1, "max_score": 0.15342641,
        "hits": [
          {
            "_index": "planet", "_type": "hacker",
            "_id": "2", "_score": 0.15342641,
            "_source": {
              "handle": "gondry",
              "hobbies": ["writing reddit comments", "skateboarding"]}}]}}


Navodimo neke od mogucih opcija pretrage tj upita.

Primjer trazenja kompletne fraze:

    POST /planet/_search
    {"query": {"match": {"hobbies": {"query": "writing reddit comments", "type": "phrase"}}}}
    // Matches gondry who does indeed like to write reddit comments

Jos jedan od interesantnih tipova je fuzzy query. Ovaj tip upita "boduje" rezultate shodno  
njihovom Levenstajnovom distancom. Naveden je primjer pretrage za nepravilno napisanu rijec  
"skateboaring".

    POST /planet/_search
    {"query": {"fuzzy": {"hobbies": "skateboarig"}}}
    // Matches gondry who has "skateboarding" listed as a hobby.

Sada ce biti nesto vise rijeci o samom procesu analize teksta.

###1.5.1 Analiza

Elasticsearch prilicno velik skup alatki za razne oblike obrade teksta sa ciljem da budu  
efikasno pretrazivane.

Za pocetak cemo razmotriti analizu engleskih rijeci sa snowball analyzer-om. Snowball  
analyzer je pogodan za nalazenje 'korjena' rijeci. Za primjer uzimamo par oblika rijeci  
'rollerblading' koju propustamo kroz snowball analyzer.

Rollerblading  
Rollerblader 			 
Rollerbladed 

Dobijamo kao rezultat analyzer-a rijec rollerblad.

Kada se pretraga vrsi na analizirano polje sami upit je analiziran, vrseci poklapanje  
sa dokumentiama koji su analizirani prilikom dodavanja u bazu podataka. Redukcija rijeci  
na ove skracene tokene normalizuje tekst cime je omoguceno brze i efikasnije pretrazivanje.
Kad god pretrazujete 'rollerblading' u bilo kom obliku initerno trazimo 'rollerblad'.

Sljedece pitanje koje se namece je gdje i kada se desava analiza, prilikom skladistenja  
podataka u dokumente prvi put i jos po jednom na svaki dolazeci upit saglasno sa pravilima  
analize polja sa kojim se poklapa. Slijedi opis procesa analize dokumenata:

- Update ili create dokumenta
- Vrijednosti polja dokumenta prolaze kroz analyzer koji pretvara vrijednosti u tokene
- Ovako tokenizirane vrijednosti se smjestaju u index, pokazujuci na originalnu verziju  
dokumenta

Na ovaj nacin je kreiran invertovani index 

Slijedi jednostavan pimrjer komande koja direktno poziva analyzer na serveru:  

    GET '/_analyze?analyzer=snowball&text=candles%20candle&pretty=true'

U ovom slucaju se vrsi analiza stringa "candles candle " radi ilustracije kako se slicne  
rijeci analiziraju. Rezultat izgleda ovako:  

    {
      "tokens" : [ {
        "token" : "candl",
        "start_offset" : 0,
        "end_offset" : 7,
        "type" : "<ALPHANUM>",
        "position" : 1
      }, {
        "token" : "candl",
        "start_offset" : 8,
        "end_offset" : 14,
        "type" : "<ALPHANUM>",
        "position" : 2
      } ]
    }

U nekim situacijama nisu dovoljni default analyzer-i, to su obicno situacije gdje postoji  
potreba za nekim ne-default opcijama i alternativnim kombinacijama tokenizer-a i filter-a.  

Analazyer se sastoji od 3 faze:  

+ Character Filtering: filtriranje ulaznog stringa na nivou karaktera
+ Tokenization: Pretvara filtriran string u niz tokena 
+ Token Filtering: Vrsi post-procesiranje filtriranih tokena 

Treba naglasiti da su koraci filtriranja opcioni i da se od analyzer-a samo zathijeva da  
pretvori string u niz tokena.

###1.5.2 Facets

Agregatna statistika predstavlja znacajan dio elasticsearch-a. Facet-i su uvijek ukljuceni  
u neki query upit, dozvoljavajuci da se ovi statisticki podaci vrate uporedo sa rezultatom  
pretrage. Razmotrimo korisnika koji pretrazuje filmove po nazivu . Koriscenjem facet-a moguce  
je vratiti npr. broj razlicitih zanrova u okviru rezultujuceg skupa pretrage.

Facet-i se mogu konfigurisati i mogu se koristiti za slozenija racunanja.

Slijedi primjer za koji je dato mapiranje: 

    GET /movie_db/movie/_mapping?pretty=true
    {
      "movie": {
        "properties": {
          "actors": {"type": "string", "analyzer": "standard",
                     "position_offset_gap": 100},
          "genre": {"type": "string", "index": "not_analyzed"},
          "release_year": {"type": "integer", "index": "not_analyzed"},
          "title": {"type": "string", "analyzer": "snowball"} }}},
          "description": {"type": "string", "analyzer": "snowball"} }}}


Faceting se vrsi po polju "genre" . Query je postavljen da trazi opis koji sadrzi rijec  
"hacking", 	i rezultat ce vratiti listu facet-a koji prikazuje koji zanrovi imaju opis  
koji sadrzi "hacking" i broj filmova toga zanra sa tim opisom:

    POST /movie_db/_search
    {"query": {"match": {"description": "hacking"}},
     "facets": {
       "genre": {
         "terms": {"field": "genre"},
         "size": 10}}}


U reluztatu pod **hits** bi trebao da bude spisak filmova dok facet sekcija bi trebala da izgleda kao na sljedecem primjeru.

     "facets": {
        "genre": {
          "_type": "terms",
          "missing": 0,
          "total": 5,
          "other": 0,
          "terms": [
            {"term": "Crime", "count": 2},
            {"term": "Action", "count": 2},
            {"term": "Drama", "count": 1}
          ] }}}


###1.5.4 Filtering

Filteri su znacajan dio elastisearch pretrage jer imaju uticaj na rezulutujuci set pretrage.
Sljedece su tri najznacajnije faze u elasticsearch filtriranju:

+ Upiti **filtered** tipa. Ugnjezdeni u query polje. Ovaj tip filtera ce uticati na rezultat  
pretrage i na broj facet-a.
+ Top-level filter element. Definisanjem filter elementa u 'korjenu' pretrage, ce vrsiti  
filtriranje samo query-a , nece uticati na facet-e.
+ Facet-i sa facet_filter opcijom. Dodatna opcija za filtriranje facet-a utice samo na  
rezultujuci skup facet-a ne i na query.

Za situacije gdje je potrebno filtriranje i **facet** i **query** rezultata najcesce se koristi  **filtered** element. Filter se prvo izvrsava pa zatim upit i facet, i u opticaju  
je povecanje brziine pretrage ako se izvrsi dovoljna restrikcija rezultujuceg skupa.

    POST /products/_search
    {
      "facets": {"department": {"terms": {"field": "department_name"}}},
      "query": {
        "filtered": {
          "query": {"match": {"name": "fake"}},
          "filter": {"term": {"department_name": "Books"}}}}}

Filtered upit se sastoji od 2 argumenta : ugnjezdeni query i filter koji treba primijeniti.


Filter element se smatra najkonfuznijim elementom u elasticsearh-u. konfuzija nastaje s obirom  
na to da **filtered** upit je ugnjezden unutar top-level query elementa i utice na query i facet-e, dok filter element je smjesten u samom korjenu u ravni query i facet elementa a ipak utice  
samo na query dio.

    POST /products/_search
    {
      "facets": {"department": {"terms": {"field": "department_name"}}},
      "query": {"match": {"description": "tcp"}},
      "filter": {"term": {"department_name": "Books"}}}


Facet filteri su suprotni filter elementu , vrsi se filtriranje samo jednog facet-a i nema uticaja na query. Vrsi se normalan facet-ing samo sto se ignorisu dokumenti koji ne zadovoljavaju filter.

    POST /products/_search
    {
      "facets": {
        "department": {
          "terms": {"field": "department_name"},
           "facet_filter": {
             "term": {"department_name": "Books"}}}
      },
      "query": {"match_all": {}}}


Filter facet radi drugaciju stvar od facet filtera, koristi se za broj dokmenata koji  
zadovoljavaju i query i dodatni filter. Razlika imedju ova dva filtera na facetima je  
sto filter_facet uvijek vrace jedan broj, dok facet_filter moze da vrati vise ako vise  
termina se poklopi sa filterom. 

    POST /products/_search
    {
      "facets": {
        "books_and_housewares": {
         1"filter": {
            "terms": {"department_name": ["Books", "Housewares"]}}}},
      "query": {"match_all": {}}}