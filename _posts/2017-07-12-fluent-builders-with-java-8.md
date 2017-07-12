---
layout: post
title: Fluent builders with java 8
author: xavier
category: general
tag: java
published: true
scripts:
  - https://tabatkins.github.io/railroad-diagrams/railroad-diagrams.js
styles:
  - https://tabatkins.github.io/railroad-diagrams/railroad-diagrams.css 
---
I have always liked fluent builders in java but, sometimes, they can be really tedious to implement.
 
But that might not longer be the case with java 8 lambda's.
 
Before I begin, let's distinguish 2 types of builders:
* the lenient builders
* the restrictive builders
 
An example of lenient builders is this one:

```java 
public class PersonBuilder {

    private String firstName;
    private String lastName;
    private LocalDate birthDate;

    public PersonBuilder setFirstName(String firstName) {
        this.firstName = firstName;
        return this;
    }

    public PersonBuilder setLastName(String lastName) {
        this.lastName = lastName;
        return this;
    }

    public PersonBuilder setBirthDate(LocalDate birthDate) {
        this.birthDate = birthDate;
        return this;
    }

    public Person build() {
        return new Person(firstName, lastName, birthDate);
    }
}
```

With a usage like this:

```java 
Person person = new PersonBuilder().setFirstName("Sherlock").setLastName("Holmes").build();
```

<script>
ComplexDiagram(
   ZeroOrMore(
    Choice(0,
      NonTerminal('setFirstName'),
      NonTerminal('setLastName'),
      NonTerminal('setBirthDate')
    )
  ),
  NonTerminal('build')
).addTo();
</script> 
 
I don't find that type of builders appealing for the following reasons: 

* you could just do new `PersonBuilder().build();` and it would compile fine as there is no indication of what fields are mandatory and nothing forces you to enter complete data
* you can call a setter multiple times (but why would you do that and why would the builder let you do that?)

In general, a lenient builder will not guide you nor help you instantiating an object in a valid state (and you should always have your objects in a valid state, right?).
 
So, personally, I prefer to have builder which will guide me and prevent me to create invalid objects.

But creating those builders require a little effort.

This is how I do it.

```java
@FunctionalInterface
public interface WithFirstName<T> {
    T withFirstName(String firstName);
}

@FunctionalInterface
public interface WithLastName<T> {
    T withLastName(String lastName);
}

@FunctionalInterface
public interface WithBirthDate<T> {
    T withBirthDate(LocalDate birthDate);
}

public interface PersonBuilder extends WithFirstName<WithLastName<WithBirthDate<Person>>> {
    static PersonBuilder newPerson() {
        return firstName -> lastName -> birthDate -> new Person(firstName, lastName, birthDate);
    }
}
```

Which is used like this:

```java
Person person = PersonBuilder.newPerson()
    .withFirstName("Sherlock")
    .withLastName("Holmes")
    .withBirthDate(LocalDate.of(1854, 1, 6));
```

<script>
ComplexDiagram(
  NonTerminal('withFirstName'),
  NonTerminal('withLastName'),
  NonTerminal('withBirthDate')
).addTo();
</script> 
 
Some problem may arise if you have too much fields resulting in constructors having dozens of parameters.
 
When that happens, I like to create a contract representing the state:

```java
public interface PersonState {
    String getFirstName();
    String getLastName();
    LocalDate getBirthDate();
    ... // even more fields
}

public interface PersonBuilder extends WithFirstName<WithLastName<WithBirthDate<WithEvenMoreFields<Person>>>> {
    static PersonBuilder newPerson() {
        return firstName -> lastName -> birthDate -> even more fields -> new Person(new PersonState() {
            public getFirstName() {
                return firstName:
            }
            public getLastName() {
                return lastName;
            }
            public getBirthDate() {
                return birthDate;
            }
            ...
        });
    }
}

// implementing PersonState is not mandatory but can yield some benefit
public class Person implements PersonState {
    public Person(PersonState state) {
        this.firstName = state.getFirstName();
        this.lastName = state.getLastName();
        this.birthDate = state.getBirthDate();
        ...
    }

    public Person clone() {
        // possible only if Person implements PersonState
        return new Person(this);
    }
}
```

Okay okay, that was easy. Now let's see how to build collections.

```java
public interface WithStreet<T> { ... }
public interface WithBox<T> { ... }
public interface WithCity<T> { ... }

public interface AddressBuilder<T> extends WithStreet<WithBox<WithCity<T>>> {
    static <T> AddressBuilder<T> newAddress(Function<? super Address, T> chain) {
        return street -> box -> city -> chain.apply(new Address(street, box, city));
    }
}

public class AddressesBuilder<T> {

    private final Function<? super Iterable<? extends Address>, T> chain;
    private final List<Address> addresses = new LinkedList<>();

    public AddressesBuilder(Function<? super Iterable<Address>, T> chain) {
        this.chain = chain;
    }

    public AddressBuilder<AddressesBuilder<T>> add() {
        return AddressBuilder.newAddress(this::add);
    }

    public AddressesBuilder<T> add(Address address) {
        this.addresses.add(address);
        return this;
    }

    public T end() {
        return chain.apply(this.addresses);
    }
}

public interface WithAddresses<T> {

    T withAddresses(Iterable<Addresses> addresses);

    default AddressesBuilder<T> withAddresses() {
        return new AddressesBuilder<>(this::withAddresses);
    }

    default T withoutAddresses() {
        return withAddresses(Collections.emptyList());
    }
}

public interface PersonBuilder extends WithFirstName<WithLastName<WithBirthDate<WithAddresses<Person>>>> {
    static PersonBuilder newPerson() {
        return firstName -> lastName -> birthDate -> addresses -> new Person(
            firstName,
            lastName,
            birthDate,
            addresses
        );
    }
}

Person person = PersonBuilder.newPerson()
    .withFirstName("Sherlock")
    .withLastName("Holmes")
    .withBirthDate(LocalDate.of(1854, 1, 6))
    .withAddresses()
        .add().withStreet("Baker Street").withBox("221B").withCity("London")
        .end(); // addresses
```

<script>
ComplexDiagram(
  Stack(
    NonTerminal('withFirstName(firstName)'),
    NonTerminal('withLastName(lastName)'),
    NonTerminal('withBirthDate(birthDate)'),
    Choice(0,
      NonTerminal('withoutAddresses()'),
      NonTerminal('withAddresses(addresses)'),
      Sequence(
        NonTerminal('withAddresses()'),
        ZeroOrMore(
          NonTerminal('add(address)'),
          Sequence(
            NonTerminal('add()'),
            Stack(
              NonTerminal('withStreet(street)'),
              NonTerminal('withBox(box)'),
              NonTerminal('withCity(city)')
            )
          )
        ),
        NonTerminal('end()')
      )
    )
  )
).addTo();
</script> 
 
So where do I use such builders? I use them when I need to convert from an external model to my internal model (at my work, we receive a lot of information in XML files and those must be transformed in objects in our model).
 
By doing so, I am sure no property is forgotten and if I add something in my builder and forget to adapt the client code, it won't compile anymore.
 
I am also using those builders in my tests but you have to be careful: if your model is changing everyday, maintaining tests can be a nightmare... that is why, you have to factorize as much as possible your test data. Make some factory method which will create test data for common scenari and use that data instead of recreating it for each test (or worse like copy/pasting it).

```java 
public class PersonFixtures {
    public static Person newWorkerPerson() { ... }
    public static Person newStudentPerson() { ... }
    public static Person newRetiredPerson() { ... }
}
```
 
Then adapt/derive one common scenario for your test.