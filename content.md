## Value Objects, Microtypes und das Specification Pattern
#### (die oft untersch&auml;tzten Bausteine von DDD*)

Patrick Drechsler



Wer praktiziert DDD*?

(*Domain Driven Design)

Note:
- Wenn wenig Leute antworten: 
    - Wer hat schon mal von Domain-Driven Design gehoert?
    - Wer wuerde gerne DDD einsetzen?



### Inhalt

- Value Objects
    - Was ist das?
    - Microtypes: Was ist das?
    - Persistenz: "mein Framework mag keine Value Objects"
    - Persistenz: "was mache ich mit Collections?"
- Specification Pattern: Business Regeln (auch zwischen Entit&auml;ten)



### Was ist eine Entity?

- Id <!-- .element: class="fragment" data-fragment-index="1" -->
- Lebenszyklus<!-- .element: class="fragment" data-fragment-index="1" -->


### Was ist ein Value Object?

- Objekt ohne Id (immutable)
- hat attributbasierte Vergleichbarkeit
- ist oft "Cohesive" (verbindet z.B. Wert und Einheit)


### Wie erkennt man Value Objects?


#### Typische Kandidaten

- String-Werte, die fuer die Dom&auml;ne von Bedeutung sind
    - IP Adresse
    - IBAN
- Kombinationen wie
    - Betrag und W&auml;hrung (42,13 EUR)
    - Wert und Einheit (23.4 kg)
- Adresse


![funx-winter-value-object](resources/fun-winter-value-object.jpg)


...hat wahrscheinlich jeder schon mal gesehen:
```csharp
public class Customer
{
    public int Id { get; set; } // evtl. in einer Entity-BaseClass 
    //...
    public string EMailAddress { get; set; }
}
```
Problem: <!-- .element: class="fragment" data-fragment-index="1" -->
- Datentyp 'string' passt nicht wirklich zu EMail Adresse. <!-- .element: class="fragment" data-fragment-index="1" -->


```csharp
public class Customer 
{
    //...
    public EMailAddress EMailAddress { get; set; }
}
```

```csharp
public class EMailAddress 
{
    public EMailAddress(string value)
    {
        if (!IsValidEmailAddress(value)) 
            throw new MyInvalidEMailAddressException(value);
        
        Value = value;
    }

    public string Value { get; }

    private bool IsValidEmailAddress(string value)
    {
        try 
        {
            new System.Net.Mail.MailAddress(value).Address = value;
        }
        catch { return false; }
    }
}
```


```txt
[a-z0-9!#$%&'*+/=?^_`{|}~-]+(?:\.[a-z0-9!#$%&'*+/=?^_
`{|}~-]+)*@(?:[a-z0-9](?:[a-z0-9-]*[a-z0-9])?\.)+[a-z0-9]
(?:[a-z0-9-]*[a-z0-9])?
```


Value Object "zu Fu&szlig;" 
```csharp
public class EMailAddress 
{
    public EMailAddress(string value)
    {
        if (!IsValidEmailAddress(value)) { 
            /* throw */ 
        }
        
        Value = value;
    }

    public string Value { get; }

    private bool IsValidEmailAddress(string value) {
        // ...
    }
}
```
- Kein Default-Konstruktor
- Kein public setter fuer Value


```csharp
public class Customer 
{
    //...
    public EMailAddress EMailAddress { get; private set; }
}
```

- Die Klasse Customer kann nur g&uuml;ltige Email Adresse haben 
- Das kl&auml;rt nicht die Frage, ob eine Email f&uuml;r den Customer verpflichtend ist (dazu sp&auml;ter mehr beim Specification Pattern)


```csharp
// within some service class
public void DoSomething(EMailAddress mailOrig, EMailAddress mailNew)
{
    if (mailOrig.Equals(mailNew)) 
    {
        // do foo...
    }
    else 
    {
        // do bar...
    }

    if (mailOrig == mailNew)
    {
        // do baz...
    }
}
```


### Vergleichbarkeit ist attributbasiert


Exkurs Vergleichbarkeit


Equality by reference
![equality-by-reference](resources/equality-reference.png)
<span class="small">http://enterprisecraftsmanship.com/2016/01/11/entity-vs-value-object-the-ultimate-list-of-differences/</span>


Equality by identifier
![equality-by-identifier](resources/equality-identifier.png)
<span class="small">http://enterprisecraftsmanship.com/2016/01/11/entity-vs-value-object-the-ultimate-list-of-differences/</span>


Equality by structure
![equality-by-structure](resources/equality-structural.png)
<span class="small">http://enterprisecraftsmanship.com/2016/01/11/entity-vs-value-object-the-ultimate-list-of-differences/</span>


#### Erstellen einer Value Object Klasse (1/3)

```csharp
public abstract class ValueObject<T> where T : ValueObject<T> 
{
    protected abstract IEnumerable<object> 
        GetAttributesToIncludeInEqualityCheck();
}    
```


#### Erstellen einer Value Object Klasse (2/3)

```csharp
public abstract class ValueObject<T> where T : ValueObject<T> 
{
    protected abstract IEnumerable<object> 
        GetAttributesToIncludeInEqualityCheck();

    public bool Equals(T other) {
        if (other == null) { return false }
        return GetAttributesToIncludeInEqualityCheck().
            .SequenceEqual(other.GetAttributesToIncludeInEqualityCheck());
    }

    public override int GetHashCode() {
        int hash = 17;
        foreach (var obj in this.GetAttributesToIncludeInEqualityCheck())
            hash = hash * 31 + (obj == null ? 0 : obj.GetHashCode());
        
        return hash;
    }
}    
```


#### Erstellen einer Value Object Klasse (3/3)

```csharp
public abstract class ValueObject<T> where T : ValueObject<T> 
{
    protected abstract IEnumerable<object> 
        GetAttributesToIncludeInEqualityCheck();

    public override bool Equals(object other) {
        return Equals(other as T);
    }

    public bool Equals(T other) {
        if (other == null) { return false }
        return GetAttributesToIncludeInEqualityCheck().
            .SequenceEqual(other.GetAttributesToIncludeInEqualityCheck());
    }

    public static bool operator ==(ValueObject<T> left, ValueObject<T> right) {
        return Equals(left, right);
    }

    public static bool operator !=(ValueObject<T> left, ValueObject<T> right) {
        return !(left == right);
    }

    public override int GetHashCode() {
        //...
    }
}
```


```csharp
public class EMailAddress : ValueObject<EMAilAddress> 
{
    public EMailAddress(string value) 
    { 
        if (!IsValidEmailAddress(value)) { /* throw */ }
        Value = value;
    }
    
    public string Value { get; }
    
    public override IEnumerable<object> 
        GetAttributesToIncludeInEqualityCheck() 
    {
        return new object[] { Value };
    }
    //...
}
```


#### Vergleichbarkeit f&uuml;r Value Objects

Man &uuml;berschreibt die `Equals` und `GetHashCode` Methoden, damit nur die Attribute (aka Properties) verglichen werden.

Bonus: Testen ist sehr einfach<!-- .element: class="fragment" data-fragment-index="1" -->


### Fazit: Value Objects

Immer dann, wenn
- der Basistyp (z.B. string) eigentlich ein Konzept der Dom&auml;ne ist (z.B. IP Adresse, Zahl & W&auml;hrung)
-  die Property keinen Lebenszykus hat (z.B. Lieferadresse)



## Microtypes


#### Fun fact 

Als ich den Vortrag eingereicht habe, war mein Verst&auml;ndnis von Microtypes komplett falsch. 


![fun-faceplam](resources/fun-picard-microtypes-ups.jpg)

Note: Darum sind Vortraege gut!


### Microtypes

Mit Microtypes sind nicht etwa kleinere Einheiten von Value Objects gemeint (dachte ich urspr&uuml;nglich).


### Microtypes
- Ein Microtype ist eine **Erweiterung** eines bestehenden Value Objects. 
- Aber nicht durch Ableitung, sondern durch Injection:

```csharp
public class InternalEMailAddress 
    : ValueObject<InternalEMailAddress> {

    public InternalEMailAddress(EMAilAddress value) {
        
        if (!IsInternalEMailAddress(value)) { /* throw */ }
        
        Value = value;
    }
}
```


*"Using microtypes is far from an industry standard. In fact, it's quite divisive. Some claim micro types are a precursor to clearer, more composable code, but for others, micro types are too many layers of annoying indirection. It's up to you to decide if you want to use the micro types pattern."*

Scott Millet/Mick Tune in Patterns, Principles and Practices of Domain-Driven Design



## Frameworks und Value Objects


![fun-framework](resources/fun-frameworks-any.jpg)


### Frameworks...

- Als ich angefangen habe, war .NET "einfacher" als Java weil es weniger Frameworks gab.
- .NET Frameworks haben von anderen gelernt und oft "reifere" Implementierungen nachgeliefert 
- Und jetzt: <!-- .element: class="fragment" data-fragment-index="1" -->
    - .NET Core API: moving target <!-- .element: class="fragment" data-fragment-index="1" -->
    - und ich habe Javascript kennengelernt... <!-- .element: class="fragment" data-fragment-index="2" -->
    - w&auml;hrend dieses Talks wurde wahrscheinlich ein neues Javascript-Framework entwickelt <!-- .element: class="fragment" data-fragment-index="2" --> 


Muss der setter fuer das Attribut wirklich `public` sein? 

Genuegt nicht 
- `internal` oder
- `protected`?


Oder noch besser:

Kann man die Klasse als Value Object beim ORM registrieren?

Ja 

(NHibernate und Entity Framework)


Bsp Entity Framework

```csharp
public class MyDbContext
{
    //...
    protected override void OnModelCreating(DbModelBuilder mb) 
    {
        mb.ComplexType<EMailAddress>();
    }
}   
```
Note: Warum das ComplexType heisst, ist mir auch ein Raetsel...


Alternative: ORM nicht verwenden

(gerade im Umfeld von CQRS und Eventsourcing)


#### Wie speichert man Listen von Value Objects?

DDD: Ist das wirklich eine Collection?<!-- .element: class="fragment" data-fragment-index="1" -->

Kann man die Liste von EMail-Adressen nicht abbilden als:<!-- .element: class="fragment" data-fragment-index="2" -->
- HomeEMailAddress<!-- .element: class="fragment" data-fragment-index="2" -->
- WorkEMailAddress<!-- .element: class="fragment" data-fragment-index="2" -->
- OtherEMailAddress<!-- .element: class="fragment" data-fragment-index="2" -->

Fazit: Die Frage, wie man das technisch l&ouml;st, wird erstmal umgangen.<!-- .element: class="fragment" data-fragment-index="3" -->


#### Wie speichert man Listen von Value Objects?

hier wird es interessant!

serialized string: JSON, XML, ...<!-- .element: class="fragment" data-fragment-index="0" -->

(wenn man eine relationale DB verwendet)<!-- .element: class="fragment" data-fragment-index="0" -->


![fun-collections](resources/fun-picard-serializing-collections.jpg)


##### Die Frage sollte sein:

- Wenn man eine **gro&szlig;e** Collection von Value Objects hat: 
    - Sind das dann wirklich noch Value Objects und nicht eher Entit&auml;ten?<!-- .element: class="fragment" data-fragment-index="0" -->
    - Ist NoSQL besser geeignet (z.B. MongoDb)?<!-- .element: class="fragment" data-fragment-index="1" -->
    - oder einfach nur Ids/URLs auf einen anderen Storage?<!-- .element: class="fragment" data-fragment-index="2" -->
        - In-Memory (Redis)<!-- .element: class="fragment" data-fragment-index="2" -->
        - Search Engine (Solar, Elastic)<!-- .element: class="fragment" data-fragment-index="2" -->
        - andere Big Data L&ouml;sungen<!-- .element: class="fragment" data-fragment-index="2" -->



## Specification Pattern

Business Regeln in eigene Objekte auslagern<!-- .element: class="fragment" data-fragment-index="0" -->


*"A specification pattern outlines a business rule **that is combinable with other business rules**."*

<!-- ![uml-specification-pattern](resources/Specification_UML_v2.png) -->
<span class="small">https://en.wikipedia.org/wiki/Specification_pattern</span>


*"Business rules often do not fit the responsibility of any of the obvious Entities or Value Objects, and their variety and combinations can overwhelm the basic meaning of the domain object. **But moving the rules out of the domain layer is even worse, since the domain code no longer expresses the model.**"*

Eric Evans in Domain Driven Design


*"Specification pattern is a pattern that allows us to **encapsulate some piece of domain knowledge into a single unit – specification – and reuse it in different parts of the code base.**"*

Vladimir Khorikov


Also z.B. dem Customer Objekt

```csharp
public class Customer 
{
    //..
    public EMailAddress EMailAddress { get; set; }
}
```


- EMail ist ein Pflichtfeld
- Wie waers mit einer **IsValid** Methode?
```csharp
public class Customer 
{
    //..
    public bool IsValid() {
        return EMAilAddress != null;
    }
}
```
Aber wenn wir diese Regel auch an anderer Stelle im Projekt brauchen?


Extrahieren wir die IsValid Methode in eine eigene Klasse:

```csharp
public class MandatoryStringSpecification : ISpecification<string>
{
    public bool IsSatisfiedBy(string s)
    {
        return !string.IsNullOrWhitespace(s);
    }
}

public interface ISpecification<T>
{
    bool IsSatisfiedBy(T entity);
}
```
Bonus: Testbarkeit


```csharp
public interface ISpecification<T>
{
    bool IsSatisfiedBy(T entity);
}

public class MandatoryStringSpecification : ISpecification<string>
{
    public bool IsSatisfiedBy(string s)
    {
        return !string.IsNullOrWhitespace(s);
    }
}
```

```csharp
public class Customer
{
    private MandatoryStringSpecification mandatoryString = 
        new MandatoryStringSpecification();

    public bool IsValid()
    {
        if (!mandatoryString.IsSatisfiedBy(this.EMailAddress.Value))
        {
            throw new MyMandatoryFieldMissingException(nameof(this.EMailAddress));
        }
    }
}
```


Nett, aber das geht noch besser: Regeln k&ouml;nnen auch miteinander kombiniert werden.


[Beispiel von M. Fowler & E. Evans](http://www.martinfowler.com/apsupp/spec.pdf) ([Wikipedia](https://en.wikipedia.org/wiki/Specification_pattern)):

- Wenn
    - Rechnung &uuml;berf&auml;llig UND
    - Mahnung verschickt UND
    - Noch nicht bei Inkasso

- Dann
    - Beauftrage Inkasso mit Rechnung


```csharp
// service class
var overDue = new OverDueSpecification();
var noticeSent = new NoticeSentSpecification();
var inCollection = new InCollectionSpecification();

// example of specification pattern logic chaining
var sendToCollection = overDue.And(noticeSent).And(inCollection.Not());

var invoiceCollection = anotherService.GetInvoices();

foreach (var currentInvoice in invoiceCollection) {
    if (sendToCollection.IsSatisfiedBy(currentInvoice))  {
        currentInvoice.SendToCollection();
    }
}
```
- Rechnung ("Invoice")
- Rechnung ist &uuml;berf&auml;llig ("OverDue")
- Mahnung wurde verschickt ("NoticeSent")
- Noch nicht bei Inkasso ("InCollection.Not()")


Wie erm&ouml;glicht man die Kombinierbarkeit der Specifications?
```csharp
public interface ISpecification<T>
{
    bool IsSatisfiedBy(T entity);
    ISpecification<T> And(ISpecification<T> other);
    ISpecification<T> Not();
    //ISpecification<T> AndNot(ISpecification<T> other);
    //ISpecification<T> Or(ISpecification<T> other);
    //ISpecification<T> OrNot(ISpecification<T> other);
}
```


CompositeSpecification

```csharp
public abstract class CompositeSpecification<T> : ISpecification<T>
{
    public abstract bool IsSatisfiedBy(T entity);

    public ISpecification<T> And(ISpecification<T> other)
    {
        return new AndSpecification<T>(this, other);
    }

    public ISpecification<T> Not()
    {
        return new NotSpecification<T>(this);
    }
    //...
}
```
- R&uuml;ckgabewert ist immer vom Typ ISpecification
- Fluent API (aka Builder Pattern)


AndSpecification

```csharp
public class AndSpecification<T> : CompositeSpecification<T>
{
    private readonly ISpecification<T> left;
    private readonly ISpecification<T> right;

    public AndSpecification(ISpecification<T> left, ISpecification<T> right)
    {
        this.left = left;
        this.right = right;
    }

    public override bool IsSatisfiedBy(T candidate)
    {
        return left.IsSatisfiedBy(candidate) && right.IsSatisfiedBy(candidate);
    }
}
```


NotSpecification

```csharp
public class NotSpecification<T> : CompositeSpecification<T>
{
    private readonly ISpecification<T> other;

    public NotSpecification(ISpecification<T> other)
    {
        this.other = other;
    }

    public override bool IsSatisfiedBy(T candidate)
    {
        return !other.IsSatisfiedBy(candidate);
    }
}
```


#### Ist das ein Anti-Pattern (Laut Wikipedia)?
- Cargo-Cult programming: *"...is a style of computer programming characterized by the ritual inclusion of code or program structures that serve no real purpose."*
- **Inner-Plattform effect**: "And()" implementiert Plattform-Methode "&&"
- **Spaghetti-Code**: Potentielle Koh&auml;sion wird in eigene Klassen aufgeteilt


Alternative L&ouml;sung ohne Specification Pattern:

```csharp
//.. in the service class
var invoiceCollection = anotherService.GetInvoices();
foreach (Invoice currentInvoice in invoiceCollection) 
{
    currentInvoice.SendToCollectionIfNecessary();
}

public class Invoice
{
    public bool ShouldSendToCollection 
    { 
        get 
        { 
            return this.OverDue && this.NoticeSent && this.InCollection == false; 
        }
    }

    public void SendToCollectionIfNecessary()
    {
        if (!ShouldSendToCollection) return;
        this.SendToCollection();
    }
}
```


Was ist mit Regeln, die **objekt&uuml;bergreifend** sind?


Z.B.: Eine EMail darf nur einem Kunden zugewiesen sein

Anders ausgedr&uuml;ckt: Das Anlegen eines neuen Kunden soll unterbunden werden, wenn die EMail schon einem anderen Kunden zugewiesen ist. 


Sp&auml;testens jetzt gen&uuml;gt das einzelne Kundenobjekt nicht mehr. 

Wir m&uuml;ssen eine externe Quelle anzapfen.


Unsere Business Regel kann nicht mehr im Objekt selbst verankert sein.

Die Anti-Pattern Einw&auml;nde sind somit erstmal alle hinf&auml;llig...


Lsg: Unsere Service-Schicht verwendet ein neue UniqueEMailSpecification


```csharp
public class Service
{
    private UniqueEMailSpecification _uniqueEmail; // more specifications go here

    public Service(IRepo repo) {
        _uniqueEmail = new UniqueEMailSpecification(repo);
    }
    
    public void CreateNewCustomer(Customer customer) {
        if (_uniqueEmail.IsSatisfiedBy(customer.EMail)) {
            // save new customer to repo
        } else { /* handle error case (ie throw) */ }
    }
}

public class UniqueEMailSpecification : CompositeSpecification<T>
{
    private IRepository _repo;
    
    public class UniqueEMailSpecification(IRepository repo) {
        _repo = repo;
    }

    public override bool IsSatisfiedBy(string mail) {
        return !repo.Customers.Contains(x => x.EMail.Equals(mail));
    }
}
```


### Fazit: Specification Pattern

- Innerhalb einer Entity: kein Problem
- Von einer Service Klasse: kein Problem
- V.a. dann n&uuml;tzlich, wenn die Regeln 
    - an mehreren Stellen verwendet werden und 
    - kombinierbar sein sollen



## Take Home Message

- Value Objects: immer
- Microtypes: manchmal
- Specification Pattern: oft

Alles einsetzbar, ohne DDD

 
 
Beste &Uuml;bersichtsseite: https://github.com/heynickc/awesome-ddd
- B&uuml;cher
    - E. Evans, Domain-Driven Design (the blue book)
    - V. Vernon, Implementing Domain-Driven Design (the red book)
    - S. Millet & N. Tune, Patterns, Principles and Practices of Domain-Driven Design
    - V. Vernon, Domain-Driven Design Distilled
- Blogs
    - http://enterprisecraftsmanship.com/
- Online trainings
    - https://www.pluralsight.com/courses/domain-driven-design-in-practice



## Danke



## Fragen?

Interesse an einem Vortrag &uuml;ber CQRS/Eventsourcing?<!-- .element: class="fragment" data-fragment-index="1" -->