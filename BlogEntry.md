# Hvordan man forhindrer NoSQL injection angreb mod MongoDB

### Authors

**Emil Gräs 
&
Anders Bjergfelt**


*Man er ikke sikret imod et injektion angreb fordi man vælger en NoSQL database. Ens system vil stadig være sårbar og konsekvenserne er, at din data forsvinder. Prepared Statements fra SQL-verdenen findes ikke i NoSQL, og derfor skal der ses på andre metoder. NoSQL injections er lette at sikre sig imod. Det kræver blot, at man adresserer enkle forbehold og følger dem. Det vil gøre dit system uskadeligt for en af de farligste sikkerhedsrisici.*


## SQL Injection

De fleste kender SQL injections. Det er et velkendt angreb der foregår ved, at en ondsindet bruger tilføjer kode ind via et input felt, som ender i en database query, der bliver eksekveret. 

Man undgår desværre ikke problemet ved at vælge en NoSQL database. Ens website vil stadig være sårbar for de former for angreb. Et lignede angreb mod en NoSQL database kaldes for en NoSQL injektion og har samme fremgangsmåde og konsekvenser som en SQL injektion. De resulterer oftest i, at data forsvinder eller bliver korrupt. 

Dette blogindlæg vil vi kort diskutere og vise, hvordan en NoSQL database kan være sårbar over for NoSQL injektioner og hvordan man kan forhindre dem. Der fokuseres på MongoDB som [den mest populære NoSQL database i øjeblikket](https://db-engines.com/en/ranking). Begreber som bliver beskrevet i dette blogindlæg er også aktuelle for andre NoSQL databaser.

## NoSQL og SQL databaser adskiller sig på mange måder fra hinanden. 

NoSQL databaser har langt færre restriktioner når der fx skal laves skemaer. De bruger heller ikke relationer eller constraints, hvilke i SQL verden er med til at sikre at data er konsistent. "Manglen" på restriktioner er netop en af grundene til hvorfor NoSQL databaser ofte [udkonkurrerer de mere traditionelle SQL databaser på performance og skalérbarhed](https://www.mongodb.com/scale/nosql-vs-relational-databases), men det gør dem også mere sårbare overfor angreb. 

Når man laver et databasekald med NoSQL skrives det ofte i det samme sprog som systemet er lavet i. Laver man fx en Java applikation, skriver man typisk sine database queries i Java. Oftest findes der et hav af API'er til lige netop dette. Disse APIer varierer meget i forhold til hvordan de er lavet og hvad de indeholder. I dag findes der mere end [200 officielle NoSQL databaser](http://nosql-database.org/), og de fleste kan integreres i mange sprog. Da der findes så mange forskellige løsninger kan det være svært at vide hvor sikkert et API er, og om de overhovedet håndterer injections eller andre potentielle sikkerhedsbrister. Det er altså uhyre svært at teste sin applikation for NoSQL injections. Det kræver et godt kendskab til APIet, typisk på et ret teknisk niveau.

## Hvor meget skade kan en injection forårsage?
OWASP ser et injektion angreb som den mest kritiske type af angreb imod en webapplikation.
Det tillader ondsindede brugere at manipulere med eksisterende data, ødelægge data eller gøre det utilgængeligt.

## Hvordan adresserer MongoDB injection?
Et vigtigt sikkerhedsaspekt med MongoDB er, at de bruger [BSON](https://docs.mongodb.com/manual/reference/glossary/#term-bson) objekter til forespørgsler og ikke strenge. De fleste biblioteker der er til rådighed for MongoDB ville giver mulighed for at bygge disse objekter og dermed undgå injections.

## Hvordan kan det være at en injection stadig er mulig?
Fordi nogle MongoDB operationer giver dig mulighed for at køre vilkårlige JavaScript-udtryk direkte på serveren. Følgende operationer er:
* $where
* mapReduce
* group
I disse tilfælde skal du være varsom med at bruge de operationer. Hvis du bruger dem skal gøre alt for at forhindre ondsindede brugere i at sende ondsindet JavaScript.


## Test af NoSQL injection sårbarheder i MongoDB
MongoDB forventer BSON objekter. Det forhindrer ikke, at det er muligt at query serialiseret JSON og JavaScript-udtryk i parametrene. Operatøren *$where* er det mest almindelige API-opkald, der tillader vilkårlige JavaScript-udtryk. $where operatøren anvendes normalvis som et filter.

```javascript db.myCollection.find( { $where: "this.username == this.name" } );
```
Hvis en ondsindet bruger kunne manipulere dataene, der blev sendt til operatøren $where og tilførte JavaScript, der skulle evalueres, kunne angrebet være string ''; return \ '' == \ '' og $where vil blive evalueret til this.name == ''; returnere '' == '', hvilket vil resultere i at alle brugere vil blive returneret i stedet for kun dem der matchede $where.

Og en anden en:

```javascript
db.myCollection.find( { $where: "this.someID > this.anotherID" } );
```
I dette tilfælde, hvis inputstrengen er '0; return true " ville ens $where blive evalueret som someID > 0;, returnere sandt og alle brugere vil blive returneret.
Eller en ondsindet bruger kunne give dette '0; mens (true) {} ' som input og man ville komme ud for et DoS-angreb.

Og en anden velkendt:
Du modtager følgende forespørgsel:
```javascript
{
    "username": {"$ne": null},
    "password": {"$ne": "null"}
}
```
Da $ne er "not equal" operatør, vil denne forespørgsel returnere den første bruger uden at kende brugerens navn eller adgangskode.
```javascript
{
    "username": "admin",
    "password": {"$gt": ""}
}
```

I MongoDB vælger $gt de dokumenter, hvor værdien af feltet er større end (dvs.>) den angivne værdi. Således overstående sammenligner adgangskode i database med tom streng, hvilket returnerer sandt.

## Hvordan forhindrer man en NoSQL injektion?

Hvis du kender lidt til SQL, vil du nok vide at man kan forhindre injektions ved at bruge Prepared Statements. 

Dit første spørgsmål vil nok gå på, om der findes Prepared Statements i NoSQL verdenen. Det gør der ikke. Derfor er man som udvikler nødt til at tage nogle andre forholdsregler, ved at tjekke sit input før man sender en query streng videre til database serveren. 

Der findes heldigvis rigtig gode såkaldte “sanitize” biblioteker, der automatisk fjerner farlige tegn fra fx brugerinputs. 

Her er en liste over de [mest populære javascript sanitize biblioteker](https://libraries.io/search?keywords=sanitization&languages=JavaScript)

### NoSQL-injektioner er lette at forhindrer ved at tage disse forholdsregler:

* Valider input for at beskytte imod skadelige værdier. I NoSQL-databaser kan du også validere inputtyper mod forventede typer.

* [Deaktivér serverside JavaScript fuldstændigt via --noscripting.] (Https://docs.mongodb.com/manual/faq/fundamentals/#how-does-mongodb-address-sql-or-query-injection) Operationerne, $where, mapReduce og group bliver ubrugelige

* Det er vigtigt at sikre, at inputtet fra brugeren, der modtages i APIet, ikke indeholder et tegn, der har en særlig betydning i f.eks MongoDB
* Det er også vigtigt ikke at bruge string concatenation til at opbygge API kald, men at bruge det tilgængelige API'en til at skabe udtrykket.

Her er en hurtig måde at sikre sig i Java målrettet med en MongoDB
```java
ArrayList<String> specialCharsList = new ArrayList<String>() {{
    add("'");
    add("\"");
    add("\\");
    add(";");
    add("{");
    add("}");
    add("$");
}};
@RequestMapping(value = "/{id}", method = RequestMethod.GET)
    public Post getPost(@PathVariable("id") String id){
		    for (String c : specialCharsList) {
             if (id.contains(c)) {
                 return error;
             }
         }
        return repository.findOne(id);
    }
```

Arraylisten indeholder alle tegn, der har en særlig betydning i MongoDB. Det vil forhindre at JavaScript kan blive eksekveret.

## Opsummering

[Populariteten af NoSQL databaser er stadig høj](https://db-engines.com/en/ranking_trend)

NoSQL injections er stadigvæk et stort problem i dag. Til trods for at det er en trussel, der nemt kan overvindes, ved at “rense” eller “sanitize” alt input der sendes fra brugeren. På den måde er DU som udvikler med til at styre hvad der skal / ikke skal sendes til din database server. Yderligere skal man være opmærksom på særlige tegn som $where, mapReduce og group. Queries med disse tegn udføres direkte på serveren, og er derfor særligt følsomme overfor uønskede tegn. 

OWASP skriver at injektions er den farligste form for sårbarhed en application kan have, netop fordi at ondsindede bruger kan have fri adgang til din data. Hvilket i sidste ende kan resultere i at din data går tabt, eller kommer i hænderne på de forkerte folk. 

For at sikre sig imod NoSQL injektioner, skal man huske ALTID at validere ALT input, der kommer fra brugere af systemet. Der findes et hav af gode biblioteker og andre sanitize værktøjer, som kan hjælpe dig. Yderligere kan man overveje helt at deaktivere serverside scripting med MongoDB med --noscripting nøgleordet.

Software sikkerhed har de sidste år været højt på dagsordenen, og vil i de næste mange år fortsætte med at være det indenfor softwareudvikling. Data er i dag i rigtig høj kurs, og derfor er det vigtigere end aldrig før at beskytte sine data imod at falde i de forkertes hænder. Det er simpelthen så vigtigt at man som udvikler, sikre sig ordentligt imod velkendte angreb. Der vil uden tvivl i fremtiden være et langt større fokus på sikkerhed indenfor IT og udvikling. 

Litteratur: https://ckarande.gitbooks.io/owasp-nodegoat-tutorial/content/tutorial/a1_-_sql_and_nosql_injection.htmlhttps://www.owasp.org/index.php/Testing_for_NoSQL_injection https://blog.sqreen.io/mongodb-will-not-prevent-nosql-injections-in-your-node-js-app/ https://software-talk.org/blog/2015/02/mongodb-nosql-injection-security/
