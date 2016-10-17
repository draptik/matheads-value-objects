## Value Objects, Microtypes und das Specification Pattern
#### (die oft unterschaetzten Bausteine von DDD*)

Patrick Drechsler



Wer praktiziert DDD*?

(*Domain Driven Design)



### Inhalt

- Value Objects: Was ist das - und warum braucht man das?
- Microtypes: Was ist das?
- Persistenz: "...aber mein Framework (zB ORM) mag keine Objekte ohne Id"
- Persistenz: "was mache ich mit Collections?"
- Specification Pattern: Business Regeln (auch zwischen Entitaeten)


...kurzer Ausflug Java vs C#...

(no Java-Bashing, I promise)


```java
// Java
public class Customer {

    private string name;

    public void setName(string value) {
        this.name = value;
    }

    public string getName() {
        return this.name;
    }
}
```

```csharp
// C#
public class Customer 
{
    public string Name { get; set; }
}
```


```java
// Java
public class Customer {

    private string name;    

    public Customer(string name) { this.name = name; }

    public string getName() { return this.name; }
}
```

```csharp
// C#
public class Customer 
{
    public Customer(string name) { this.Name = name; }

    public string Name { get; }
}
```



### Was ist eine Entity?

- Id <!-- .element: class="fragment" data-fragment-index="1" -->
- Lebenszyklus<!-- .element: class="fragment" data-fragment-index="1" -->


### Was ist ein Value Object?

- Objekt ohne Id (immutable)
- hat attributbasierte Vergleichbarkeit
- ist oft "Cohesive" (verbindet z.B. Wert und Einheit)


### Wie erkennt man Value Objects?


#### Typische Kandidaten

- String-Werte, die fuer die Domaene von Bedeutung sind
    - IP Adresse
    - IBAN
- Kombinationen aus Betrag und Waehrung/Einheit
    - Betrag und Waehrung (42 EUR)
    - Wert und Einheit (23.4 kg)
- Adresse


...hat wahrscheinlich jeder schon mal gesehen:
```csharp
public class Customer
{
    public int Id { get; set; } // evtl. in einer Entity-BaseClass 
    //...
    public string EMailAddress { get; set; }
}
```
Probleme: <!-- .element: class="fragment" data-fragment-index="1" -->
- Datentyp 'string' passt nicht wirklich zu EMail Adresse. <!-- .element: class="fragment" data-fragment-index="1" -->
- EMailAddress hat einen public setter. <!-- .element: class="fragment" data-fragment-index="1" -->


```csharp
public class Customer 
{
    //...
    public EMailAddress EMailAddress { get; private set; }
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


```csharp
public class Customer 
{
    //...
    public EMailAddress EMailAddress { get; private set; }
}
```

- Die Klasse Customer kann nur gueltige Email Adresse haben 
- Das klaert nicht die Frage, ob eine Email fuer den Customer verpflichtend ist (dazu spaeter mehr beim Specification Pattern)


```csharp
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


```csharp
public abstract class ValueObject<T> where T : ValueObject<T> 
{
    protected abstract IEnumerable<object> 
        GetAttributesToIncludeInEqualityCheck();
}    
```


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
        int hash = 17;
        foreach (var obj in this.GetAttributesToIncludeInEqualityCheck())
            hash = hash * 31 + (obj == null ? 0 : obj.GetHashCode());
        
        return hash;
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
}
```


#### Vergleichbarkeit fuer Value Objects

Man ueberschreibt die `Equals` und `GetHashCode` Methoden, damit nur die Attribute (aka Properties) verglichen werden.

Bonus: Testen ist sehr einfach<!-- .element: class="fragment" data-fragment-index="1" -->


### Fazit: Value Objects

Wenn der Basistyp (z.B. string) eigentlich ein Konzept der Domaene ist

- Beispiele:
    - EMail Adresse
    - IP Adresse
    - Zahl und Einheit/Waehrung
    - Adresse



### Microtypes


#### Fun fact 

Als ich den Vortrag eingereicht habe, war mein Verstaendnis von Microtypes komplett falsch. 

![fun-faceplam](resources/fun-picard-microtypes-ups.jpg)

Darum sind Vortraege gut!


### Microtypes

Mit Microtypes sind nicht etwa kleinere Einheiten von Value Objects gemeint (dachte ich urspruenglich).


### Microtypes
- Ein Microtype ist eine **Erweiterung** eines bestehenden Value Objects. 
- Aber nicht durch Ableitung, sondern durch Injection:

```csharp
public class InternalEMailAddress 
    : ValueObject<InternalEMailAddress> {

    // ctor
    public InternalEMailAddress(EMAilAddress value) {
        
        if (!IsInternalEMailAddress(value)) { /* throw */ }
        
        Value = value;
    }
}
```


*"Using microtypes is far from an industry standard. In fact, it's quite divisive. Some claim micro types are a precursor to clearer, more composable code, but for others, micro types are too many layers of annoying indirection. It's up to you to decide if you want to use the micro types pattern."*

Scott Millet/Mick Tune in Patterns, Principles and Practices of Domain-Driven Design



### Frameworks und Value Objects


![fun-framework](resources/fun-frameworks-any.jpg)


#### Frameworks...

- Als ich angefangen habe, war .NET "einfacher" als Java weil es weniger Frameworks gab.
- .NET Frameworks haben von anderen gelernt und oft "reifere" Implementierungen nachgeliefert 
- Und jetzt: <!-- .element: class="fragment" data-fragment-index="1" -->
    - .NET Core API: moving target <!-- .element: class="fragment" data-fragment-index="1" -->
    - und ich habe Javascript kennengelernt... <!-- .element: class="fragment" data-fragment-index="2" -->
    - waehrend dieses Talks wurde wahrscheinlich ein neues Javascript-Framework entwickelt <!-- .element: class="fragment" data-fragment-index="2" -->


#### Frameworks and Value Objects

- Value Objects sollten immutable sein
- ORM (Object-Relational-Mapper) <!-- .element: class="fragment" data-fragment-index="1" -->
    - Domaenen Objekte 1-zu-1 mit einem ORM abzubilden <!-- .element: class="fragment" data-fragment-index="1" -->
        - Framework braucht Proxy der Klasse <!-- .element: class="fragment" data-fragment-index="2" -->
            - Jedes Attribut muss einen public setter haben <!-- .element: class="fragment" data-fragment-index="2" -->
            - Constructor muss parameterlos sein <!-- .element: class="fragment" data-fragment-index="2" -->


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
Warum das ComplexType heisst, ist mir auch ein Raetsel...


Alternative: ORM nicht verwenden

(gerade im Umfeld von CQRS und Eventsourcing)


#### Wie speichert man Listen von Value Objects?

DDD: Ist das wirklich eine Collection?<!-- .element: class="fragment" data-fragment-index="1" -->

Kann man die Liste von EMail-Adressen nicht abbilden als:<!-- .element: class="fragment" data-fragment-index="2" -->
- HomeEMailAddress<!-- .element: class="fragment" data-fragment-index="2" -->
- WorkEMailAddress<!-- .element: class="fragment" data-fragment-index="2" -->
- OtherEMailAddress<!-- .element: class="fragment" data-fragment-index="2" -->

Fazit: Die Frage, wie man das technisch loest, wird umgangen.<!-- .element: class="fragment" data-fragment-index="3" -->


#### Wie speichert man Listen von Value Objects?

hier wird es interessant!

serialized string: JSON, XML, ...<!-- .element: class="fragment" data-fragment-index="0" -->

(wenn man eine relationale DB verwendet)<!-- .element: class="fragment" data-fragment-index="0" -->


![fun-collections](resources/fun-picard-serializing-collections.jpg)


##### Die Frage sollte sein:

- Wenn man eine **grosse** Collection von Value Objects hat: 
    - Sind das dann wirklich noch Value Objects und nicht eher Entitaeten?<!-- .element: class="fragment" data-fragment-index="0" -->
    - Ist NoSQL besser geeignet (z.B. MongoDb)?<!-- .element: class="fragment" data-fragment-index="1" -->
    - oder einfach nur Ids/URLs auf einen anderen Storage?<!-- .element: class="fragment" data-fragment-index="2" -->
        - In-Memory (Redis)<!-- .element: class="fragment" data-fragment-index="2" -->
        - Search Engine (Solar, Elastic)<!-- .element: class="fragment" data-fragment-index="2" -->
        - andere Big Data Loesungen<!-- .element: class="fragment" data-fragment-index="2" -->



### Specification Pattern

Business Regeln in eigene Objekte auslagern<!-- .element: class="fragment" data-fragment-index="0" -->


*"A specification pattern outlines a business rule **that is combinable with other business rules**."*

<!-- ![uml-specification-pattern](resources/Specification_UML_v2.png) -->
<span class="small">https://en.wikipedia.org/wiki/Specification_pattern</span>


*"Business rules often do not fit the responsibility of any of the obvious Entities or Value Objects, and their variety and combinations can overwhelm the basic meaning of the domain object. **But moving the rules out of the domain layer is even worse, since the domain code no longer expresses the model.**"*

Eric Evans in Domain Driven Design


*"Specification pattern is a pattern that allows us to **encapsulate some piece of domain knowledge into a single unit** – specification – and reuse it in different parts of the code base."*

Vladimir Khorikov


Also z.B. dem Customer Objekt

```csharp
public class Customer {
    public int Id { get; set; }
    public EMailAddress EMailAddress { get; set; }
}
```


- EMail ist ein Pflichtfeld
- Wie waers mit einer **IsValid** Methode?
```csharp
public class Customer {
    //..
    public bool IsValid() {
        return EMAilAddress != null;
    }
}
```
Aber wenn wir diese Regel auch an anderer Stelle im Projekt brauchen?


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
    private MandatoryStringSpecification mandatoryStringSpec = 
        new MandatoryStringSpecification();

    public bool IsValid()
    {
        if (!mandatoryStringSpec.IsSatisfiedBy(this.EMailAddress.Value))
        {
            throw new MyMandatoryFieldMissingException(nameof(this.EMailAddress));
        }
    }
}
```


Nett, aber das geht noch besser: Regeln koennen auch miteinander kombiniert werden.


Beispiel von M. Fowler & E. Evans (Wikipedia):

- Wenn
    - Rechnung ueberfaellig UND
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
- Rechnung ist ueberfaellig ("OverDue")
- Mahnung wurde verschickt ("NoticeSent")
- Noch nicht bei Inkasso ("InCollection.Not()")


Wie ermoeglicht man die Kombinierbarkeit der Specifications?
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


#### Ist das ein Anti-Pattern?

- Inner-Plattform effect: "And()" implementiert Plattform-Methode "&&"
- Spaghetti-Code: Potentielle Kohaesion wird in eigene Klassen aufgeteilt


Alternative Loesung ohne Specification Pattern:

```csharp
//.. in the service class
var invoiceCollection = Service.GetInvoices();
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


Was ist mit Regeln, die objektuebergreifend sind?


Z.B.: Eine EMail darf nur einem Kunden zugewiesen sein

Anders ausgedrueckt: Das Anlegen eines neuen Kunden soll unterbunden werden, wenn die EMail schon einem anderen Kunden zugewiesen ist. 


Spaetestens jetzt genuegt das einzelne Kundenobjekt nicht mehr. 

Wir muessen eine externe Quelle anzapfen.


Unsere Business Regel kann nicht mehr im Objekt selbst verankert sein.

Die Anti-Pattern Einwaende sind somit erstmal alle hinfaellig...


Lsg: Unsere Service-Schicht verwendet ein neue UniqueEMailSpecification


```csharp
public class Service
{
    private UniqueEMailSpecification _uniqueEmail; // more specifications go here

    public Service(IRepo repo)
    {
        _uniqueEmail = new UniqueEMailSpecification(repo);
    }
    
    public void CreateNewCustomer(Customer customer)
    {
        if (_uniqueEmail.IsSatisfiedBy(customer.EMail)) 
        {
            // save new customer to repo
        }
    }
}

public class UniqueEMailSpecification : CompositeSpecification<T>
{
    private IRepository _repo;
    
    public class UniqueEMailSpecification(IRepository repo)
    {
        _repo = repo;
    }

    public override bool IsSatisfiedBy(string mail)
    {
        return !repo.Customers.Contains(x => x.EMail.Equals(mail));
    }
}
```


### C#/Java haben jetzt Funktionen als First-Class Citizens

...man ist versucht Funktionen zu uebergeben...


```csharp
public abstract class Specification<T>
{
    public abstract Expression<Func<T, bool>> ToExpression();
 
    public bool IsSatisfiedBy(T entity)
    {
        Func<T, bool> predicate = ToExpression().Compile();
        return predicate(entity);
    }
}
```


- Hier koennen Leaky-Abstraction Probleme eine Rolle spielen (Bsp C#):
    - IQueryable (echt fiess: das Interface gibts 2x)
    - IEnumerable
    - IList
    - IReadOnlyCollection

...Potential LSP Violation...


### Fazit: Specification Pattern

- Innerhalb einer Entity: kein Problem
- Von einer Service Klasse: kein Problem
- Wunderbar um Regeln zu kombinieren
- Wenn man ein paar Punkte beachtet: kein Problem



## Take Home Message

- Value Objects: immer
- Microtypes: manchmal
- Specification Pattern: oft



## Fragen?



## Danke



- Buecher
    - E. Evans, Domain-Driven Design (the blue book)
    - V. Vernon, Implementing Domain-Driven Design (the red book)
    - S. Millet & N. Tune, Patterns, Principles and Practices of Domain-Driven Design
    - V. Vernon, Domain-Driven Design Distilled
- Blogs
    - http://enterprisecraftsmanship.com/
- Online trainings
    - https://www.pluralsight.com/courses/domain-driven-design-in-practice

