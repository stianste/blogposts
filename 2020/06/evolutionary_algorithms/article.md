# «Survival of the fittest» – En intuitiv introduksjon til evolusjonære algoritmer

## Notater

- Skriver i førsteperson og refererer til leseren som "deg/du"

## Introduksjon

Det kan ofte virke som maskinlæring og "AI" krever en master i lineær algebra,
en PhD i statistikk og 10 års erfaring med Python.
I denne artikkelen skal jeg vise deg en liten snarvei,
og heller bruke det vi allerede vet fungerer fra biologien – Evolusjonære Algoritmer (EA).
EA bruker konsepter som reproduksjon, mutasjon og «fitness»
til å gjøre alt fra å lære roboter å gå, til å løse reelle optimaliseringsproblemer hos NASA.
Dette gjør de i tillegg på en intuitiv måte som alle kan forstå, uten noen forkunnskap.

I denne artikkelen skal jeg gå gjennom de grunnleggende stegene i en evolusjonær algoritme,
og gi et praktisk eksempel i Kotlin på hvordan vi kan bruke disse til å
approksimere Traveling Salesperson problemet.

**Disclaimer**: _Jeg har aldri skrevet Kotlin før, så det blir nok ikke den ryddigste koden du har sett._
I tillegg er det et par variabler rundt omkring i koden som er definert utenfor scopet som er vist, men
som ikke defineres eksplisitt i artikkelen for å gjøre ting litt ryddigere.
F.eks. `random` fra Kotlin, konstanter som `POPULATION_SIZE`, støttefunksjoner for å skrive ut løsninger og lignenende.

Den endelige koden kan du finne [her](https://github.com/stianste/evolutionary-TSP)

## Problemet

For å forstå hvordan vi kan bruke EA til å approksimere TSP, kan det være greit å ta en rask gjennomgang av hva
problemet faktisk er. [Wikipedia](https://en.wikipedia.org/wiki/Travelling_salesman_problem) beskriver problemet
på følgende vis:

> Given a list of cities and the distances between each pair of cities,
> what is the shortest possible route that visits each city and returns to the origin city?

Så et eksempel kan formuleres som: «Gitt at vi starter i Oslo, hva er den korteste ruten vi kan ta for å besøke
alle byene i Norge og havne tilbake i Oslo?»

De som har hatt fag som Algoritmer og Datastrukturer eller tilsvarende, husker kanskje at dette er et
NP-Hardt problem. Uten å gå i for mye detaljer vil det si at problemet er jævlig vanskelig å løse.
Det betyr også at om jeg gir deg en løsning og
sier «hei, her har du den korteste ruten for å besøke alle byer i Norge», så
kan ikke du engang verifsere at min foreslåtte løsningen stemmer (i polynomisk tid alle fall).

Om dette ikke gir 100% mening trenger du ikke uroer deg. Det viktiste vi må vite er at problemet, så vidt
vi vet idag i alle fall, ikke lar seg løse. Vi må derfor ta til takke med å _approksimere_ en løsning.

I denne artikkelen kommer vi derfor til å ta for oss noen mindre eksempler som vi allerede vet løsningen på, slik at vi kan
verfisere at approkismeringen vår er ok. Mer spesifikt fant jeg [denne siden](https://people.sc.fsu.edu/~jburkardt/datasets/tsp/tsp.html)
med noen mindre datasett. Her finner vi blant annet FIVE problemet som som har korteste løsning på 19. Den har en distansematrise som
ser slik ut:

```
0.0  3.0  4.0  2.0  7.0
3.0  0.0  4.0  6.0  3.0
4.0  4.0  0.0  5.0  8.0
2.0  6.0  5.0  0.0  6.0
7.0  3.0  8.0  6.0  0.0
```

Det vil si at distansen mellom by 1 og by 2 er 3,0. Om dette er 3,0 km eller 3,0 mil spiller egentlig ingen
rolle. Tilsvarende er distansen mellom by 1 og by 3, 4,0. Distansen mellom by 2 og by 4 er 6.0 – også videre.
Legg merke til at matrisen er symmetrisk og at diagonalen er 0.0 for alle verdier. Dette er naturligvis forventet ettersom
lengden fra by 1 til by 3 vil være den samme som fra by 3 til by 1, og lengden fra en by til seg selv vil være 0.

Om vi kikker på løsningen for problemet så er den 1 -> 3 -> 2 -> 5 -> 4. Altså at vi starter i by 1, drar til by 3, så til by 2,
etterfulgt av by 5, og tislutt by 4. Om vi leser av distansematrisen får vi da `4.0 + 4.0 + 3.0 + 6.0 = 17`.
I tillegg må vi, slik problemet er beskrevet, tilbake til by 1. Distansen fra 4 til by 1 er 2.0, så da lander vi på `19` som
er den optimale løsningen for dette problemet.

Vi skal bruke dette eksempelet for å verifisere at algoritmen kommer fram til en sti av lengde 19. I tillegg skal vi bruke FRI26 problemet
fra samme side for å få et litt større problem. Dette problemet består av 26 byer, og har en optimal løsning på `973`.

For å gi litt perspektiv kan det være verdt å merke seg hvor stort «search space» hvert problem har, altså hvor mange mulige løsninger
det finnes.
Om kombinatorikken min fortsatt stemmer vil det med 5 byer være 5 måter å velge den første byen på. Deretter gjenstår det 4 byer vi kan
velge å besøke. Etter at vi har valgt neste by vil det være 3 valg gjennstående, osv. Vi ender da med `5*4*3*2*1 = 5! = 120` potensielle
løsninger. Det er ikke for ille, og gjør egentlig problemet enkelt fordi vi bare kan prøve alle 120 løsningene, og velge den med kortest
lengde. Om vi tar 26 byer dog, blir det en litt annen historie. Med 26 får vi `26! = 4.0329146e+26` mulige løsninger – også kjent som flere løsninger
enn vi klarer å fatte.

Vi starter raskt med å definere denne matrisen i programmet vårt i Kotlin:

```kotlin
val exampleDistanceMatrix = arrayOf(
    doubleArrayOf(0.0,  3.0,  4.0,  2.0,  7.0),
    doubleArrayOf(3.0,  0.0,  4.0,  6.0,  3.0),
    doubleArrayOf(4.0,  4.0,  0.0,  5.0,  8.0),
    doubleArrayOf(2.0,  6.0,  5.0,  0.0,  6.0),
    doubleArrayOf(7.0,  3.0,  8.0,  6.0,  0.0)
)
```

## Byggesteinene i en Evolusjonær Algoritme

Så, nå som vi har fått en forståelse for problemet vi skal løse, kan vi se nærmere på verktøyet vi skal bruke.
En evolusjonær algoritme består av et par grunnleggende konsepeter, samt noen enkle steg i selve algrotimen.

### Representasjon

Det første vi trenger er en form for **representasjon** av en potensiell løsning, noe også kan se på som **genomet** til løsningen.
I vårt tilfelle vet vi både hvor lang en potensiell løsning vil være (antall byer i problemet), samt de mulige verdiene byene kan ha.
Derfor vil en fornuftig representasjon i vårt tilfelle være en vektor. Andre problemer kan kreve andre representasjoner som matriser eller trær,
men det skal ikke vi fokusere på. Om vi velger å representere løsninger via vektorer kan en mulig løsning for FIVE problemet være `[1,2,3,4,5,1]`,
altså å bare besøke alle byene i rekkefølge. Den optimale løsningen vil da være vektoren `[1,3,2,5,4,1]` (som vist i forrige seksjon). Siden vi dog
vet at vi må begynne og slutte i første by, kan vi sløyfe `1` ettersom det er implisitt. Altså at representasjonen for den optimale løsningen
i programmet vårt blir `[3,2,5,4]`.

### Fitness

I en EA er vi også avhengig av å kunne vurdere hvor god en løsning er. Til dette bruker vi noe som vi kaller «fitness».
Fitnessen til en løsning vil påvirke sannsynligheten for at denne løsningen «overlever» og får lov til å reprodusere seg og lage nye løsninger –
akkurat som i evolusjonen! Vi må derfor definere en fitnessfunksjon som er en funksjon som tar inn en mulig løsning, or returnerer
en score på hvor god løsningen er. Denne funksjonen vil ha stor innvirkning på hvor godt algoritmen vil være i stand til å tilpasse
løsninger.

I vår tilfelle er fitnessfunksjonen relativt enkel, men ikke triviel.
Vi ønsker en funksjon som straffer løsninger hvor den resulterende ruten er lang,
og belønne korte løsninger. La oss si at værst
tenkelige løsning har en lengde på 100. Da vil vi helst at en funksjon hvor en løsning med lengde 100 har en fitness som nærmer seg 0,
mens en kort løsning har en langt høyere høy fitness.
Dessverre vet vi hverken verste eller beste mulige løsning før vi begynner problemet.
Vi må derfor _estimere_ dette «taket» på funksjonen.
Vi kan f.eks. si at den verste tenkelige løsnigen vil være den største distansen mellom uansett by for alle byer. Så f.eks.
for FIVE problemet, hvor vi skal innom fem byer, og den lengste distansen i matrisen er 8.0 (mellom by 3 og 5), så er den verst tenkelige
distansen `8*5 = 40`. Det er verdt å merke seg at dette faktisk er en verre løsning en den verste mulige løsningen, fordi å gå mellom by
3 og 5 fem ganger ikke er en gyldig løsning. Dette er derfor vi kaller det et estimat, for å finne den lengste ruten faktisk vil være like
vanskelig å finne som den korteste. Vi kan dermed definere fitness funksjonen vår slik:

```kotlin
fun scoreFitness(solution: IntArray, distanceMatrix: Array<DoubleArray>, maxFitness: Double): Double {
    var routeLength = distanceMatrix[0][solution[0]]
    for ( i in 1 until solution.size) {
        routeLength += distanceMatrix[solution[i-1]][solution[i]]
    }

    routeLength += distanceMatrix[solution.last()][0]

    return maxFitness - routeLength
}
```

Her tar vi inn `solution`, som er den foreslåtte løsningen som vi ønsker å gi en fitness-score. Vi starter med å sette `routeLength`
til å være distansen fra by 1 til den første byen i `solution`. Deretter ittererer vi bare over og legger til distansene til den totale
lengden, før vi tilslutt legger til distansen fra den siste byen i løsningen tilbake til by 1. Deretter returnerer vi `maxFitness - routeLength`,
som diskutert over.

Så for eksempel vil fitness scoren til den trivielle løsningen hvor vi besøker byene i rekkefølge være
`scoreFitness([2,3,4,5], exampleDistanceMatrix, maxFitness = 40.0) = 40.0 - (3.0 + 4.0 + 5.0 + 6.0 + 7.0) = 15.0`, mens den optimale løsningen vil
ha en fitness scorep på
`scoreFitness([3,2,5,4], exampleDistanceMatrix, maxFitness = 40.0) = 40.0 - 19.0 = 21.0`. Her ser vi at forskjellen mellom den trivielle løsningen
og den optimale ikke veldig stor. Det er definitivt rom for forbedring i taket og fitnessfunksjonen vår, men som et eksempel duger den godt.

### Grunnleggende Steg

Vi skal nå gå gjennom de grunleggende stegene i en EA. Stjålet skamløst fra Wikipedia så er basisstegene:

1. Generate the initial population of individuals randomly. (First generation)

2. Evaluate the fitness of each individual in that population (time limit, sufficient fitness achieved, etc.)

3. Repeat the following regenerational steps until termination:

   - Select the best-fit individuals for reproduction. (Parents)
   - Breed new individuals through crossover and mutation operations to give birth to offspring.
   - Evaluate the individual fitness of new individuals.
   - Replace least-fit population with new individuals. -->

#### Steg 1 – Generere den initielle populasjonen

Steg 1 er enkelt – vi trenger rett og slett bare et sted å starte. Vi lager en funksjon for å generere en tilfeldig løsning:

```kotlin
fun generateRandomSolution(totalNumberOfCities: Int): IntArray {
    // Start at 1 to omit the origin city
    return (1 until totalNumberOfCities).toList().shuffled(random).toIntArray()
}
```

Dette er bare en enkel funksjon som genererer en liste fra 1 til antall byer (så i FIVE eksempelet `[2,3,4,5]`), stokker den i en tilfeldig rekkefølge basert på `random`
(ekstrahert for å kunne sette seed selv), og returnerer listen.

Deretter kan vi bestemme hvor sto populasjonen vår skal være (altså hvor mange løsninger vi ønsker å ha i hver) generasjon, og generere potensielle løsninger:

```kotlin

var currentPopulation = Array(POPULATION_SIZE) { generateRandomSolution(distanceMatrix.size)}
currentPopulation.take(3).forEach ( ::printPopulationMember )

```

som for FIVE vil skrive ut noe som:

```
[4, 3, 2, 5, ]
[5, 3, 4, 2, ]
[5, 3, 4, 2, ]
```

Vi har nå generert den første generasjonen vår med løsninger

#### Steg 2 – Evaluere den poplasjonen

Neste steg blir så å sjekke fitnessen til den nåværende generasjonen av løsninger.

## Future Work
