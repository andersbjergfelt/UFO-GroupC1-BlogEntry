# Hvordan man forhindrer NoSQL injection angreb mod MongoDB

### Authors

**Emil Gräs 
&
Anders Bjergfelt**

De fleste kender SQL injections. Det er et velkendt angreb der foregår ved, at en ondsindet bruger tilføjer kode ind via et inputfelt, som ender i en database query, der bliver eksekveret.
Man undgår desværre ikke problemet ved at vælge en NoSQL database. Ens website vil stadig være sårbar for de former for angreb. Et lignede angreb mod en NoSQL database kaldes for en NoSQL injektion og har samme fremgangsmåde og konsekvenser som en SQL injektion. De resulterer oftes i, at data forsvinder eller bliver korrupt. Vælger man dog at tage sine forbehold, er NoSQL injections lette at sikre sig imod. Det kræver blot, at man adresserer disse forbehold og følger dem. Det vil gøre dit website uskadelig for en af de farligste sikkerhedsrisici.  

Dette blogindlæg vil vi kort diskutere og vise, hvordan en NoSQL database kan være sårbar over for NoSQL injektioner og hvordan man kan forhindre dem. Der vil blive fokusere på MongoDB som [den mest populære NoSQL database i øjeblikket] (https://db-engines.com/en/ranking). Begreber som bliver beskrevet i dette blogindlæg er også aktuelle for andre NoSQL databaser.


NoSQL og SQL databer adskiller sig på mange måder fra hinanden. NoSQL databaser har langt færre restriktioner når der fx skal laves skemaer. De bruger heller ikke relationer eller constraints, hvilke i SQL verden er med til at sikre at data er konsistent. "Manglen" på restriktioner er netop en af grundene til hvorfor NoSQL databaser ofte udkonkurerer de mere traditionelle SQL databaser på performance og skalérbarhed. Men det gør dem også mere sårbare overfor angreb. Database injections er i dag stadigvæk et omfattende og alvorligt problem, og det er til trods for at det  oftest ikke kræver meget arbejde at sikre sig imod dem.

Når man laver et databasekald med NoSQL skrives det ofte i det samme sprog som systemet er lavet i. Laver man fx en java applikation, skriver man typisk sine database queries i java. Oftest findes der et hav af APIer til lige netop dette. Disse APIer varierer meget i forhold til hvordan de er lavet og hvad de indeholder. I dag findes der mere end 200 officielle NoSQL databaser (http://nosql-database.org/), og de fleste kan integreres i mange sprog. Da der findes så mange forskellige løsninger kan det være svært at vide hvor sikkert et API er, og om de overhovedet håndtere injections eller andre potentielle sikkerhedsbrister. Det er altså uhyre svært at teste sin applikation for NoSQL injections. Det kræver et godt kendskab til APIet, typisk på et ret teknisk niveau.

## Hvor meget skade kan en injection forårsage?

[OWASP](https://www.owasp.org/index.php/Main_Page) ser et injektion angreb som det mest kritiske webapplikationssikkerhedsrisiko. 

Det tillader ondsindede brugere at manipulere med eksisterende data, tillade fuldstændig offentliggørelse af alle data på systemet, ødelægge data eller gøre det utilgængeligt.

## Hvordan adresserer MongoDB injection?

MongoDB bruger [BSON] (https://docs.mongodb.com/manual/reference/glossary/#term-bson) objekter til forespørgsler og ikke strings.
De fleste biblioteker der er til rådighed for MongoDB ville giver mulighed for at bygge disse objekter og dermed undgå injections.

### Hvordan kan det være at en injection stadig er mulig?
Fordi nogle MongoDB operationer giver dig mulighed for at køre vilkårlige JavaScript-udtryk direkte på serveren. Følgende operationer er:

* $where
* mapReduce
* group

I disse tilfælde skal du være varsom med at bruge de operationer. Hvis du bruger dem skal gøre alt for at forhindre ondsindede brugere i at sende ondsindet JavaScript.

### Test af NoSQL injection sårbarheder i MongoDB

MongoDB forventer BSON objekter. Det forhindrer ikke, at det er muligt at query unserialiseret JSON og JavaScript-udtryk i parametrene.
Operatøren **$where** er det mest almindelige API-opkald, der tillader vilkårlige JavaScript-udtryk.
$where operatøren anvendes normalvis som et filter.

```javascript
db.myCollection.find( { $where: "this.username == this.name" } );
```
Hvis en ondsindet bruger kunne manipulere dataene, der blev sendt til operatøren **$where** og tilførte JavaScript, der skulle evalueres, kunne angrebet være **string '\'; return \ '\' == \ ''** og **$where** vil blive evalueret til **this.name == ''; returnere '' == ''**, hvilket vil resultere i at alle brugere vil blive returneret i stedet for kun dem der matchede **$where**.

Og en anden en:
```javascript
db.myCollection.find( { $where: "this.someID > this.anotherID" } );
```
I dette tilfælde, hvis inputstrengen er '0; return true " ville ens **$where** blive evalueret som *someID > 0;*, returnere sandt og alle brugere vil blive returneret.

Eller en ondsindet bruger kunne give dette **'0; mens (true) {} '** som input og man ville komme ud for et DoS-angreb.

Og en anden velkendt:

Du modtager følgende forespørgsel:

```javascript
{
    "username": {"$ne": null},
    "password": {"$ne": "null"}
}
```
Da **$ne** er "not equal" operatør, vil denne forespørgsel returnere den første bruger uden at kende brugerens navn eller adgangskode.

```javascript
{
    "username": "admin",
    "password": {"$gt": ""}
}
```
I MongoDB vælger **$gt** de dokumenter, hvor værdien af feltet er større end (dvs.>) den angivne værdi. Således overstående sammenligner adgangskode i database med tom streng, hvilket returnerer sandt.

### Hvordan forhindrer man en NoSQL injektion?

NoSQL-injektioner er lette at forhindrer ved at tage disse forholdsregler:

1. Valider input for at beskytte i mod skadelige værdier. I NoSQL-databaser kan du også validere inputtyper mod forventede typer.

2. [Deaktivér serverside JavaScript fuldstændigt via --noscripting.] (Https://docs.mongodb.com/manual/faq/fundamentals/#how-does-mongodb-address-sql-or-query-injection) Operationerne, $where, mapReduce og group bliver ubrugelige

1. Det er vigtigt at sikre, at inputtet fra brugeren, der modtages i APIet, ikke indeholder et tegn, der har en særlig betydning i f.eks
MongoDB

Det er også vigtigt ikke at bruge string concatenation til at opbygge API kald, men at bruge det tilgængelige API'en til at skabe udtrykket.

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

### Opsummering

[Populariteten af NoSQL databaser er stadig høj](https://db-engines.com/en/ranking_trend). NoSQL databaser udkonkurerer ofte de mere traditionelle SQL databaser på performance og skalérbarhed.

Problemet er bare, at mange gode praksisser fra traditionelle SQL-databaser overses. Det medfører, at NoSQL databaser lider under det samme som deres SQL-modpart. Det er vigtigt at tage sine forbehold og derved forhindre et NoSQL angreb. 
Injections er i dag stadigvæk et omfattende og alvorligt problem, og det er til trods for at det oftest ikke kræver meget arbejde at sikre sig imod dem.


Litteratur:
https://ckarande.gitbooks.io/owasp-nodegoat-tutorial/content/tutorial/a1_-_sql_and_nosql_injection.html
https://www.owasp.org/index.php/Testing_for_NoSQL_injection
https://blog.sqreen.io/mongodb-will-not-prevent-nosql-injections-in-your-node-js-app/
https://software-talk.org/blog/2015/02/mongodb-nosql-injection-security/
