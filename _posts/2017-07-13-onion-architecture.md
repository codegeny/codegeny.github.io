---
layout: post
title: Onion architecture
author: xavier
category: general
tag: java domain-driven
published: false
---


Sometimes, I stumble upon articles praising hexagonal/onion architectures where the domain model is put at the core of the architecture.

I really like that idea but each time I read deeper in the article, there is always something that does not feel quite right.

For example: separating the repository interface (in the domain) from its implementation (in the infrastructure using JPA) seems right but soon I find out that the domain model contains a lot of JPA annotations... where is the separation now?

And that's not the end of it, that kind of domain model always suffers from the technical constraints imposed by JPA like the need for a default constructor or a primary key field (which has no business value).

So if you really want to separate the business domain from its supporting architecture, you should not declare the mapping for your entities in your domain layer and keep it free from all the other constraints. But how then?

So far, I have found 2 ways to do it.

The first is to declare abstract entities in the domain which have concrete business methods but abstract state methods like this:

```java
public abstract class domain.Person {
    
     // abstract state methods
    public abstract UUID getUUID();
    protected abstract LocalDate getBirthDate();

    // really simple business method
    public Period getAgeAt(LocalDate date) {
        return period.of(getBirthDate(), date);
    }
}

public interface domain.PersonRepository {
    
    Optional<Person> findByUUID(UUID uuid);

    Person add(Person person);
}
```

Note: if you are going to need the birthDate outside of the entity, you could make the getBirthDate() method public, it's up to you. By default, I start with protected and increase visibility if needed.

Then I have the following implementation in my infrastructure layer:

```java
@Entity(name = "Person")
public class infrastructure.JpaPerson extends domain.Person {

    @Basic(optional = false)    
    @Column(unique = true)
    UUID uuid;

    @Id
    @GeneratedValue
    Long id;

    @Version
    Integer version;

    @Basic(optional = false)
    LocalDate birthDate;

    // default "technical" constructor for JPA
    JpaPerson() {}

    // business constructor
    JpaPerson(LocalDate birthDate) {
        this.uuid = UUID.randomUUID();
        this.birthDate = birthDate;
    }

    public UUID getUUID() {
        return this.uuid;
    }

    protected LocalDate getBirthDate() {
        return this.birthDate;
    }
}

public class infrastructure.JpaPersonRepository implements domain.PersonRepository {
    
    @PersistentContext
    private EntityManager entityManager;

    public Optional<Person> findByUUID(UUID uuid) {
        try {
            return Optional.of(this.entityManager.createNamedQuery("Person.findByUUID").setParameter("UUID", uuid).getSingleResult());
        } catch (NoResultException noResultException) {
            return Optional.empty();
        }
    }

    public Person add(Person person) {
        return this.entityManager.merge(person);
    }
}
```

This way of doing has several advantages:

* the domain model does not need to be annotated with JPA annotations (only the implementation does)
* the domain model does not need extra id/version fields (only the implementation does)
* the domain model does not need to multiple constructors (technical & business) as it does not have state, so it doesn't need /any/ constructor
* if you have a composite value object, you could decompose it in the implementation class. For example, a value object Address may contain street, box and zipCode. You could store its individual fields directly in the entity as JPA AttributeConverters do not support multi-column mapping ( see https://github.com/javaee/jpa-spec/issues/105)

Why having separate UUID and ID?

* an entity receives its UUID when it's instantiated, that UUID will never change (if you need to implement equals/hashcode, you should do it with UUID)
* an entity receives its ID (database primary key) when it's persisted so it will be null until your entity is persisted. So you should not base your equals/hashcode on this field.
* why not only UUID? only my aggregate roots have an UUID (as they are the only object which will be retrieved through repository, so they are the only ones that must be globally identifiable) but all other sub-entities will only have and ID. In my case, I found the UUID not performant enough and did not want it to be used as foreign key. This is of course an optimization concern and nothing prevent you to use it as the @Id of your entity.

By doing so, the state has been abstracted from the entity.




The other solution is to define some sort of SPI in your domain in charge of the state:

```java
public interface domain.PersonState {
    
    UUID getUUID();

    LocalDate getBirthDate();

    // include setters if your entity is going to change the state
}

public interface domain.PersonStateDao {
    
    Optional<PersonState> findByUUID(UUID uuid);

    PersonState add(PersonState state)
}

public class domain.Person {
    
    final PersonState state;

    public Person(PersonState state) {
        this.state = state;
    }

    public Period getAgeAt(LocalDate date) {
        return period.of(state.getBirthDate(), date);
    }
}

public class domain.PersonRepository {
    
    private final PersonStateDao stateDao;

    public PersonRepository(PersonStateDao stateDao) {
        this.stateDao = stateDao;
    }

    public Optional<Person> findByUUID(UUID uuid) {
        return this.stateDao.findByUUID(uuid).map(Person::new);
    }

    public Person add(Person person) {
        return new Person(this.stateDao.add(person.state));
    }
}
```

Now, in your infrastructure layer, you would have the implementation of the PersonState and the PersonStateDao:

```java
@Entity
public class infrastructure.PersonStateImpl implements domain.PersonState {
    
    @Id
    @GeneratedValue
    Long id;

    @Version
    Integer version;

    @Basic(optional = false)    
    @Column(unique = true)
    UUID uuid;

    @Id
    @GeneratedValue
    Long id;

    @Version
    Integer version;

    @Basic(optional = false)
    LocalDate birthDate;

    // constructors
    // getters
    // setters
}

public class infrastructure.JpaPersonStateDao implements domain.PersonStateDao {
    
    ...
}
```

In this 2nd way, we favor composition over inheritance but there are some drawbacks:

* We end with even more classes than the first method
* What if we want to store a Person entity in a passivating CDI scope like a (Conversation|View|Session)Scope? Then the PersonStateImpl must be serializable (I have no problem with that as the infrastructure is supposed to know about the constraint in the layers underneath) but the Person should be Serializable too (this is not so ideal).
    
    
    
    
    
    
    
    
    
    
    
    

Sometimes, I stumble upon articles praising hexagonal/onion architectures where the domain model is put at the core of the architecture.

I really think this is a good way to keep your domain model free of technical/infrastructure-related constraints but in practice, I rarely see posts
where this is truly achieved in the end. 

Typically, technical concerns are concealed from the domain and implemented in the infrastructure layer... but is it really possible to hide, say the persistence,
and keep the domain free of the constraints imposed by usual frameworks like JPA? 

Let's see if we can identify the problems and do something about it.

I've read a lot of blog posts advocating to put the implementation of your repository (using JPA) in the infrastructure layer.
The problem I have with all those posts is that the domain model was littered with JPA annotations.
So, in the end, if you are going to use JPA, why put the repository implementation in some layer and the mapping in another layer?

The problem here is that the separation was not complete:
I am not saying I am against putting the implementation of the repository in the infrastructure layer but if you do so, you should also free
the domain of the mapping and every other constraint of the persistence framework (like default constructors and id field).

So, I asked myself how I could remove the JPA annotations from my domain and free my model of any restrictions JPA bestows on it.

As we are already separating the repository interface (in the domain layer) from the repository implementation (in the infrastructure layer),
why not do the same with the domain entities as well?    