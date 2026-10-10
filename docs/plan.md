# Piano: booking-policy-bundle

## Contesto

Più applicazioni gestiscono prenotazioni di natura diversa: turni di lavoro (autoassegnazione, candidature, scambi), posti (scrivanie, sale), lezioni (iscrizioni). Ogni applicazione ha il proprio modulo di dominio, con le proprie entità e invarianti, ma molte **regole di prenotazione** sono le stesse: quando apre e chiude la prenotazione, se serve una conferma ed entro quando, fino a quando si può cancellare, capienza, lista d'attesa.

`fedale/booking-policy-bundle` raccoglie queste regole generiche:
- le rende **configurabili a runtime** su una catena di scope, tramite il nucleo "scoped settings" di `fedale/setting-bundle` (vedi `setting-bundle/docs/plan-scoped-settings.md`);
- le **valuta** con funzioni pure, che i moduli di dominio chiamano dai propri aggregati.

Bundle correlati:
- **setting-bundle**: fornisce definizioni, scope, resolver, store e (in futuro) il bridge Form. È una dipendenza.
- **calendar-bundle** (vedi `calendar-bundle/docs/plan.md`): mostra gli eventi. **Non dipende** da booking-policy, e booking-policy non dipende dal calendario. Gli adapter applicativi traducono le decisioni di booking-policy in `EventInteractions` e `hints` del calendario.

## Principi

- **Nessuna entità di prenotazione.** Il bundle non salva prenotazioni, non conosce le entità dei domini e non ha tabelle proprie (i valori configurati stanno nello store di setting-bundle).
- **Il dominio resta l'autorità.** Il bundle risolve la politica e la valuta su fatti forniti dal chiamante; l'aggregato del modulo di dominio decide e fa rispettare l'invariante.
- **Valutazione pura.** Le funzioni di valutazione ricevono politica, fatti e istante corrente, e restituiscono una decisione. Niente I/O, niente orologio implicito: tutto testabile senza container.
- **Solo regole generiche.** Le regole specifiche di un dominio (riposo minimo legale tra turni, unicità del docente, competenze richieste) restano nei moduli.
- Codice, commenti e documentazione del bundle in inglese.

## 1. Scope

Catena, dal più specifico al più generico:

```
occurrence:<id> → series:<id> → resource:<id> … → eventType:<type> → global
```

- **occurrence**: la singola occorrenza (una lezione del martedì, un turno specifico).
- **series**: la serie ricorrente a cui l'occorrenza appartiene, se esiste.
- **resource**: la risorsa (aula, posto, reparto). Un evento può avere **più risorse**: l'adapter le passa in ordine di priorità (per esempio aula prima del docente) e ognuna diventa uno scope della catena.
- **eventType**: il tipo di evento (`lesson`, `desk`, `shift`…), definito dall'applicazione.
- **global**.

Il tenant è gestito dal nucleo di setting-bundle con la regola "prima il tenant": l'intera catena del tenant corrente, poi l'intera catena del tenant 0 (default di piattaforma).

```php
final readonly class PolicySubject
{
    /** @param list<string> $resourceIds in priority order */
    public function __construct(
        public string $eventType,
        public array $resourceIds = [],
        public ?string $seriesId = null,
        public ?string $occurrenceId = null,
    ) {}

    public function toScopeChain(): ScopeChain;
}
```

## 2. Opzioni

Namespace `booking.policy`. Il valore assente significa "non configurato" e ha una semantica di default dichiarata per ogni opzione (colonna "Se assente").

Legenda applicabilità: ✓ diretta, ~ con interpretazione, – non applicabile. Per i turni "prenotare" significa autoassegnarsi, candidarsi o chiedere uno scambio.

### 2.1 v1

| Gruppo | Chiave | Tipo | Scope | Se assente | Turni | Posti | Lezioni |
|---|---|---|---|---|---|---|---|
| Prenotabilità | `bookable` | Bool | tutti | `false` | ~ | ✓ | ✓ |
| | `opensBefore` | Duration | tutti | sempre aperta | ~ | ✓ | ✓ |
| | `closesBefore` | Duration | tutti | fino all'inizio | ~ | ✓ | ✓ |
| | `opensAt` | DateTime | series, occurrence | — | – | ~ | ✓ |
| | `closesAt` | DateTime | series, occurrence | — | – | ~ | ✓ |
| | `allowAfterStart` | Bool | tutti | `false` | – | ✓ | ✓ |
| Conferma | `confirmation` | Enum: `none`, `manual` | tutti | `none` | ✓ | ✓ | ✓ |
| | `confirmationDeadline` | Duration | tutti | nessuna scadenza | ✓ | ✓ | ✓ |
| | `onConfirmationTimeout` | Enum: `accept`, `reject` | tutti | `reject` | ✓ | ✓ | ✓ |
| | `holdExpiresAfter` | Duration | tutti | nessuna scadenza | – | ✓ | ✓ |
| Cancellazione | `cancellable` | Bool | tutti | `true` | ~ | ✓ | ✓ |
| | `cancellationDeadline` | Duration | tutti | fino all'inizio | ~ | ✓ | ✓ |
| Modifica | `reschedulable` | Bool | tutti | `false` | ✓ | ✓ | ~ |
| | `rescheduleDeadline` | Duration | tutti | fino all'inizio | ✓ | ✓ | ~ |
| Capienza | `capacityMax` | Int | tutti | illimitata | ✓ (organico massimo) | ✓ | ✓ |
| | `capacityMin` | Int | tutti | nessuna soglia | ✓ (copertura) | – | ✓ |
| | `capacityMinReachedBefore` | Duration | tutti | — | ✓ | – | ✓ |
| | `maxUnitsPerBooking` | Int | tutti | 1 | – | ✓ | ~ |
| Lista d'attesa | `waitlist` | Bool | tutti | `false` | ~ | ✓ | ✓ |
| | `waitlistMax` | Int | tutti | illimitata | ~ | ✓ | ✓ |
| | `waitlistAutoPromote` | Bool | tutti | `true` | ~ | ✓ | ✓ |
| Quote | `maxActivePerUser` | Int | eventType, global | illimitata | ~ | ✓ | ✓ |
| Idoneità | `audience` | List | tutti | tutti ammessi | ✓ | ✓ | ✓ |

Note:
- **`confirmation`**: `automatic` non serve come valore, coincide con `none`. Se un domani servirà una conferma automatica con ritardo (per esempio dopo un controllo esterno), si aggiunge un valore.
- **`opensAt`/`closesAt`** sono ammessi solo su series e occurrence: una data assoluta a livello globale o di tipo non ha senso.
- **`maxActivePerUser`** solo su eventType e global: una quota "per risorsa" o "per occorrenza" coincide con `maxUnitsPerBooking`.
- **`audience`**: lista di stringhe opache (gruppi, ruoli). Il confronto con l'utente lo fa un `AudienceMatcherInterface` fornito dall'applicazione; il default confronta con i ruoli Symfony.

### 2.2 v2 (dopo la v1)

| Gruppo | Chiave | Tipo | Turni | Posti | Lezioni |
|---|---|---|---|---|---|
| Quote | `quotaPerPeriod` | Int o Duration + periodo | ✓ (ore/settimana) | ✓ | ✓ |
| Tempi | `durationMin` / `durationMax` / `durationStep` | Duration | ✓ | ✓ | – |
| | `bufferBefore` / `bufferAfter` | Duration | ✓ | ✓ | ✓ |
| Ricorrenza | `bookWholeSeries`: `no`, `optional`, `required` | Enum | ~ | ✓ | ✓ |
| Presenza | `checkIn` / `checkInOpensBefore` / `noShowAfter` | Bool / Duration / Duration | ~ | ✓ | ✓ |
| | `lateCancellationCountsAsNoShow` | Bool | – | ✓ | ✓ |
| Capienza | `overbookingAllowance` | Int | – | ~ | ✓ |
| Visibilità | `showRemainingCapacity` / `showParticipants` | Bool | ~ | ✓ | ✓ |
| Notifiche | `reminders` | List di Duration | ✓ | ✓ | ✓ |

Escluse: pagamenti (possibile estensione futura `requiresPayment`, legata a `holdExpiresAfter`).

## 3. La politica risolta

```php
final readonly class BookingPolicy
{
    // One typed, nullable property per v1 option; null = not configured.
    public function __construct(
        public bool $bookable,
        public ?Duration $opensBefore,
        public ?Duration $closesBefore,
        // ...
        public array $origins,   // key => ResolvedValue origin (scope, tenant), for the admin UI
    ) {}
}

interface BookingPolicyResolverInterface
{
    public function resolve(PolicySubject $subject): BookingPolicy;
}
```

Il resolver chiama `ScopedSettingsResolver::resolve('booking.policy', $subject->toScopeChain(), $subject)` e applica i default della tabella. Le definizioni ricevono il `PolicySubject` come contesto di `isApplicable()`.

## 4. Valutazione

```php
final readonly class BookingFacts
{
    public function __construct(
        public \DateTimeImmutable $start,     // in the timezone of the place where the event happens
        public \DateTimeImmutable $end,
        public int $confirmedUnits,
        public int $pendingUnits,
        public int $waitlistedUnits,
        public int $requestedUnits = 1,
        public int $userActiveBookings = 0,
        public bool $userInAudience = true,
    ) {}
}

final class BookingRules
{
    public function canBook(BookingPolicy $p, BookingFacts $f, \DateTimeImmutable $now): PolicyDecision;
    public function canCancel(BookingPolicy $p, BookingFacts $f, \DateTimeImmutable $now): PolicyDecision;
    public function canReschedule(BookingPolicy $p, BookingFacts $f, \DateTimeImmutable $now): PolicyDecision;
    public function confirmationDueAt(BookingPolicy $p, \DateTimeImmutable $bookedAt): ?\DateTimeImmutable;
    public function holdExpiresAt(BookingPolicy $p, \DateTimeImmutable $bookedAt): ?\DateTimeImmutable;
}
```

`PolicyDecision`:
- `outcome`: `Allowed`, `AllowedPendingConfirmation`, `Waitlisted`, `Denied`;
- `violations`: lista di `PolicyViolation` con un **codice stabile** (`booking.not_open_yet`, `booking.closed`, `booking.full`, `booking.not_in_audience`, `cancellation.too_late`…) e parametri (per esempio `opensAt`). I messaggi sono traduzioni nel dominio `BookingPolicyBundle`.

Punti fermi:
- **Fuso orario**: le durate "N giorni prima" si calcolano nel fuso del luogo dell'evento. Il chiamante passa `start` in quel fuso; `Duration::subtractFrom()` lavora nel fuso dell'istante ricevuto.
- **Orologio**: `$now` è sempre un parametro. Il servizio di comodo che lo legge da `ClockInterface` (PSR-20) sta fuori da `BookingRules`.
- **Concorrenza**: la capienza valutata su `BookingFacts` è corretta solo se i fatti sono coerenti al momento della scrittura. Il dominio deve garantirlo (lock pessimistico, contatore atomico o versione). Va scritto chiaramente nella documentazione.

## 5. Snapshot della politica

Regola:
- le regole valutate **al momento della prenotazione** (finestra, capienza, quote, idoneità) usano la politica **corrente**;
- le regole valutate **più avanti nel ciclo di vita** della prenotazione (cancellazione, modifica, scadenza della conferma, scadenza del blocco provvisorio, e in v2 check-in e no-show) usano lo **snapshot** salvato nella prenotazione.

Così, se un amministratore cambia la scadenza di cancellazione, le prenotazioni già fatte mantengono la regola con cui sono state create.

- `BookingPolicy::toSnapshot(): array` e `BookingPolicy::fromSnapshot(array): self`, con un numero di versione dello schema.
- Il dominio salva lo snapshot nella propria entità (per esempio una colonna JSON). Il bundle non decide come.
- Una chiave sconosciuta nello snapshot viene ignorata; una mancante prende il default della tabella.

## 6. Indicazioni per la visualizzazione

```php
final readonly class BookingHints
{
    public function __construct(
        public bool $bookableNow,
        public ?\DateTimeImmutable $opensAt,
        public ?\DateTimeImmutable $closesAt,
        public ?int $remainingUnits,
        public bool $requiresConfirmation,
        public bool $waitlistOpen,
    ) {}
}
```

`BookingRules::hints(BookingPolicy, BookingFacts, $now): BookingHints` calcola informazioni solo da mostrare. L'adapter applicativo le traduce in `hints` del calendario e ricava `EventInteractions` (per esempio `movable` da `canReschedule`). Nessuno dei due bundle importa classi dell'altro.

## 7. Interfaccia di amministrazione

- Un form per scope, costruito dal bridge Form di setting-bundle (fase 7 di quel piano: booking-policy è il secondo utilizzatore che la sblocca).
- Mostra per ogni opzione il valore ereditato e la sua origine ("ereditato da: tipo lesson, piattaforma"), usando `ResolvedValue`.
- Il bundle fornisce un componente Twig `BookingPolicyForm` da incorporare nelle pagine di amministrazione dell'applicazione (per esempio nella modifica di una lezione, con scope `occurrence:<id>`). L'applicazione decide dove e chi può modificare.

## 8. Dipendenze

- `fedale/setting-bundle` `^1.x` con i tipi `Duration`, `DateTime` e `List` (fase 6 di quel piano): **prerequisito**.
- `psr/clock` per il servizio di comodo.
- Opzionali: `symfony/form` e `symfony/ux-twig-component` per l'amministrazione, `symfony/validator` per i vincoli.
- Nessuna dipendenza da Doctrine, calendar-bundle o API Platform.

## 9. Fasi

0. **Prerequisito**: setting-bundle fase 6 (`Duration`, `DateTime`, `List`).
1. **Definizioni e resolver**: definizioni v1, `PolicySubject` e catena, `BookingPolicy` con i default, `BookingPolicyResolverInterface`. Test sulla precedenza con risorse multiple e sui default.
2. **Valutazione**: `BookingRules`, `PolicyDecision`, codici di violazione e traduzioni. Test su ogni regola, inclusi i cambi dell'ora legale e i confini (esattamente all'apertura, esattamente alla chiusura).
3. **Snapshot**: serializzazione versionata e test di compatibilità.
4. **Indicazioni**: `BookingHints` e un esempio di adapter verso il calendario, nell'applicazione demo.
5. **Amministrazione**: form e componente Twig (dipende dal bridge Form di setting-bundle).
6. **v2**: opzioni della sezione 2.2, a partire da quelle richieste da un'applicazione reale.

## 10. Questioni aperte

- **Valori bloccati**: un valore di piattaforma che il tenant non può sovrascrivere (per esempio un preavviso minimo imposto). Dipende dalla questione analoga aperta in setting-bundle.
- **Ordine tra più risorse**: lo decide l'adapter. Va verificato con i casi reali se basta, o se serve una regola diversa per alcune opzioni (per esempio `capacityMax` come minimo tra le risorse invece del "più specifico vince").
- **Quote nel tempo** (`quotaPerPeriod`, v2): richiede di definire il periodo (settimana di calendario o finestra mobile) e chi conta le prenotazioni (il dominio, passando il conteggio nei fatti).
- **Mesi in `Duration`**: "iscrizioni aperte un mese prima" oggi va espresso in giorni. Da rivalutare se un'applicazione lo richiede.
