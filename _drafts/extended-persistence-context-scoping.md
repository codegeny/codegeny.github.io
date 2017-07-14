---
layout: post
title: Extended persistence context scoping
author: xavier
tag:
  - java
  - jpa
  - cdi
---
//////Years ago, one of the most exciting features of Seam (which later gave birth to CDI) was the introduction of scopes and the ability to use (scoped) `EXTENDED` Persistence Contexts to avoid LazyInitExceptions.


(EJB)Container-managed JPA `EntityManager`s come in two flavors/types:

- `TRANSACTION`, where an `EntityManager` is bound to a transaction (it is open during the transaction and closed when the transaction commits/rolls back).
- `EXTENDED`, where an `EntityManager` is bound to the life-cycle of the `@Stateful` session bean that is holding it (which in turn can have a variety of scopes, thanks to CDI).

Each type has its pros and cons:

`TRANSACTION`: the problem with a `TRANSACTION` `EntityManager` is that... it requires a transaction and transactions are typically short-lived. Once a transaction is completed (either committed or rolled-back), all `EntityManager`s that were bound to that transaction are closed and lazy associations that were not initialized during the transaction will raise a `LazyInitException` if you try to initialize them afterwards.

A workaround is to use some sort of `OpenEntityManagerInViewServletFilter` (or a `PhaseListener` if you're using JSF) to start a transaction and keep it active for the whole duration of the request (including rendering the view where most of the LazyInitExceptions occur).

Another problem comes when your web application holds state on the server-side: entities can be loaded during one request, then *stored* in the http session or some other sub-scope (conversation, flash, JSF viewscope) and then accessed in a subsequent request.

In that case, no ServletFilter or PhaseListener will be able to help as the `EntityManager` that loaded those entities is definitely gone. Accessing uninitialized lazy associations from those entities will give a `LazyInitException`.

The workaround in this case is to reload/merge your entities in a fresh new `EntityManager`. This can be cumbersome depending on how much state you've been holding (you have to replace every entity you held reference to) and this is not really efficient as you are basically retrieving entities from the database you already had. (What's the point of storing them then?).

Note that if you want a wizard-style flow and save your modification only at the end of your flow, you will need to set your intermediates transactions to rollback-only.

Another solution to avoid `LazyInitException` is to load everything beforehand (by using JPA 2.1 `EntityGraph` or joining in your query) but then you need to know exactly what part of your entity graph will be needed (it does not work well with models where you can't know how it will be traversed/navigated).

The only advantage, IMHO, of the `TRANSACTION` `EntityManager`s is that their working model is fairly simple (all modifications to managed entities are automatically flushed to the database at commit-time). `EntityManager`s are managed for you and there is no risk of resource leakage.

//////Using `TRANSACTION` `EntityManager`s is more similar to using raw JDBC: you can load/select data in a request, then update/merge modification on the next request.


`EXTENDED`: the big advantage of an `EXTENDED` `EntityManager` is that you can keep it open as long as you want and avoid `LazyInitException` but then you are in charge of giving it a proper scope so that it does not remain open forever. A big difference with `TRANSACTION` `EntityManager` is that flushing does not occur automatically, you have to call `flush()` yourself inside your transaction (but that can be done really easily with CDI interceptors/events).

In the early days of Seam, you could see examples of JSF backing beans that were `@ConversationScoped` `@Stateful` session beans like this:

```java
@Stateful
@ConversationScoped
public class MyBean {
	
	@PersistenceContext(type = `EXTENDED`)
	private EntityManager manager;
	private MyEntity myEntity;
	private Long id;

	public void initialize() {
		myEntity = manager.createNamedQuery(...).setParameter("id", id).getSingleResult();
		...
	}

	public void setId(Long id) { ... } // id from request params (viewParam)

	public MyEntity getMyEntity() { ... } // the loaded entity, made available to the view

	public void performSomeBusinessLogic() { ... } // the business to perform on the entity

}
```

One thing I didn't like was that my JSF backing beans had direct access to the `EntityManager`. So you couldn't have a nice Repository with properly defined methods, it was even advised to use the `EntityManager` directly injected in the JSF bean as it was considered as a generic Repository and you wouldn't need anything else. I didn't like it because an `EntityManager` with its fluent methods (Query) is harder to mock than a simple interface Repository and conflating business logic + data access in the same class is a design style I don't like much.

So if you wanted to hide your `EntityManager` behind a Repository, you had to make your Repository `@Stateful` as well to be able to hold your `EXTENDED` `EntityManager`.

The problem with that approach comes when you have to combine 2 or more repositories in your business logic. As each Repository would have its own `EntityManager`, they would not share their common entities (first level cache) and there you could have a lot of nasty bugs when mixing entities coming from different EMs.

So you need to find a way to provide a common `EntityManager` for your repositories. That is where Producers come to the rescue. A producer could be responsible to open a single `EntityManager` for the ConversationScope. This (proxied) `EntityManager` would then be injected in all repositories. In return, repositories would not need to be `@Stateful` anymore. So everything seems right at first but it is not: now that your `EntityManager` are ConversationScoped, you are forced to have a Conversation to use them.

In general, this is not a problem because for each request, there is always a conversation (either transient or long-running).

But what if you are doing some batch processes? There is no request and no conversation scopes available at this time.

And what if managing a long-running conversation is too much for your use case? For example, if you are using JSF with `@ViewScoped` beans and want your EM to remain open for AJAX requests that could lead to the initialization of some lazy associations, you now have to begin (and end) a conversation. In that case, it would have been better to have a `@ViewScoped` `EntityManager` where start and ends are implicit instead of being explicit with your conversation.

So, none of those 2 problems are impossible to solve: for the batch process, you could register another `@ConversationScoped` context that is only active during batch processes and for the second one, you could replace the @ViewScoped with `@ConversationScoped` and begin()/end() your Conversation manually (and adding CID parameters everywhere it is needed and such). This is more work for the developer.

Maybe another solution would be to scope the EM in a dynamic context, one that could be `@ConversationScoped`, `@ViewScoped`, `@RequestScoped` or even `@TransactionScoped` depending on the current situation.

Let's name this special context `@UnitOfWorkScoped` and define it as such:

```java
@Retention(RUNTIME)
@NormalScope(passivating = true)
@DynamicScope({
	@DelegateScope(scope = ThreadScoped.class),
	@DelegateScope(scope = ConversationScoped.class, unless = "#{javax.enterprise.context.conversation.transient}"),
	@DelegateScope(scope = ViewScoped.class, unless = "#{facesContext.viewRoot.transient}"),
	@DelegateScope(scope = RequestScoped.class),
	@DelegateScope(scope = TransactionScoped.class)
})
public @interface UnitOfWorkScoped {}
```

This scope will delegate to the first active context in the following order:

- first, it will check if a ThreadContext is available. The ThreadContext is mainly used in batch/asynchronous processes (like an EJB @Schedule or a @MessageDriven), it guarantees that a single EM will be used for that Thread. The ThreadContext must be activated manually through a CDI interceptor (@ThreadContextual) on your method/type. Activating a ThreadContext can also be used when you need to "override" the current EM (when you already have one but you need a new one). (for example, if you have a WS and have a SOAPHandler that logs every request/response, you don't want your business logic to potentially corrupt your @RequestScoped EM and be unable to log your response properly, if you don't specify in your SOAPHandler that you want to use different EM, the same will be used for the whole request: SOAPHandler and business logic).

- then, if no ThreadContext was available, @UnitOfWorkScoped will delegate to the Conversation context only if it is long-running (non-transient). Why make this restriction? Because I think that a ConversationScope is greater than a JSF ViewScope (so it must come before the ViewScope) but as there is always a Conversation available (transient) per request, transient conversation (~ request) would always take precedence over the ViewScope that comes after

- then, if no long-running conversation is available, @UnitOfWorkScoped will check if there is a non-transient @ViewScoped available. Here, the restriction over non-transient is made because storing things in a ViewScope of a transient/stateless view has no point and even gives warnings.

- then, if no ViewScope is available, the EM will stay open for the whole request.

- finally, if no Request is available, the EM will be bound to the current transaction (which gives back the usual way of using an EM, the `TRANSACTION` type)

The scope is marked as passivating as it can potentially delegate to other passivating scopes (conversation, view).

Warning: I've had great success using this mechanism but you need to be sure conditions for evaluating to which scope UnitOfWorkScope will delegate to do not change (unintentionally) during your processing. For example, if you plan on using a long-running conversation, you MUST call Conversation.begin() before any use of the EM (otherwise, you could have multiple EM in different scopes, one for example in Request and one in Conversation after you called begin()).

Remark: I've included a JSF scope (@ViewScoped) but not @FlowScoped, why? The new JSF 2.2 @FlowScoped would theoretically make a good candidate for a delegate scope because it has a definite lifecycle. But the implementation of this scope is problematic and won't work with this @DynamicScope because it is the only scope I know of that has attributes (for example @FlowScoped("MyFlow")), and the Context implementation uses those attributes to keep the scopes for different flows separated. So, if you don't provide those attributes, you will get a NPE. This is real problem that I would like to be addressed in future revision of the JSF specification.



























In an EJB container, managed JPA `EntityManager`s come in two flavors/types:

- `TRANSACTION` (`@PersistenceContext(type = TRANSACTION)`), where an `EntityManager` is bound to a transaction (it is open during the transaction and closed when the transaction commits/rolls back).
- `EXTENDED` (`@PersistenceContext(type = EXTENDED)`), where an `EntityManager` is bound to the lifecycle of the `@Stateful` session bean that is holding it (which in turn can have a variety of scopes, thanks to CDI).

Each type has its pros and cons.

## `TRANSACTION`

The problem with a `TRANSACTION` `EntityManager` is that... it requires a transaction and transactions are typically short-lived. Once a transaction is completed (either committed or rolled-back), all `EntityManager`s that were bound to that transaction are closed and lazy associations that were not initialized during the transaction will raise a `LazyInitException` if you try to initialize them afterwards.

A workaround is to use some sort of `Open`EntityManager`InViewServletFilter` (or a `PhaseListener` if you're using JSF) to start a transaction and keep it active for the whole duration of the request (including rendering the view where most of the `LazyInitException`s occur).

Another problem comes when your web application holds state on the server-side: entities can be loaded during one request, then *stored* in the http session or some other sub-scope (conversation, flash, JSF viewscope) and then accessed in a subsequent request.

In that case, no `ServletFilter` or `PhaseListener` will be able to help as the `EntityManager` that loaded those entities is definitely gone. Accessing uninitialized lazy associations from those entities will give a `LazyInitException`, no matter what.

The workaround in this case is to reload/merge your entities in a fresh new `EntityManager`. This can be cumbersome depending on how much state you've been holding (you have to replace every entity you held reference to) and this is not really efficient as you are basically retrieving entities from the database you already had. (What's the point of storing them then?).

Note that if you want a wizard-style flow and save your modification only at the end of your flow, you will need to set your intermediates transactions to rollback-only.

Another solution to avoid `LazyInitException` is to load everything beforehand (by using JPA 2.1 `EntityGraph` or joining in your query) but then you need to know exactly what part of your entity graph will be needed (it does not work well with models where you can't know how it will be traversed/navigated). You can also transform all your entities to DTOs inside your transaction but you don't always want to use different models in your domain and presentation layers.

The only advantage, IMHO, of the `TRANSACTION` `EntityManager`s is that their working model is fairly simple (all modifications to managed entities are automatically flushed to the database at commit-time). `EntityManager`s are managed for you and there is no risk of resource leakage.

## `EXTENDED`

The big advantage of an `EXTENDED` `EntityManager` is that you can keep it open as long as you want, thus avoiding `LazyInitException`s, but then you are in charge of giving it a proper scope so that it does not remain open forever. A big difference with `TRANSACTION` `EntityManager` is that flushing does not occur automatically, you have to call `flush()` yourself inside your transaction (but that can be done really easily with CDI interceptors/events).

In the early days of Seam, you could see examples of JSF backing beans that were `@ConversationScoped` `@Stateful` session beans like this:

```java
@Stateful
@ConversationScoped
public class MyBean implements MyBeanInterface {
	
	@PersistenceContext(type = `EXTENDED`)
	private EntityManager manager;
	private MyEntity myEntity;
	private Long id;

	public void initialize() {
		myEntity = manager.createNamedQuery(...).setParameter("id", id).getSingleResult();
		...
	}

	public void setId(Long id) { ... } // id from request params (viewParam)
	public MyEntity getMyEntity() { ... } // the loaded entity, made available to the view
	public void performSomeBusinessLogic() { ... } // the business to perform on the entity
}
```

One thing I didn't like was that my JSF backing beans had direct access to the `EntityManager`. So you couldn't have a nice Repository with properly defined methods, it was even advised to use the `EntityManager` directly injected in the JSF bean as it was considered as a generic Repository and you wouldn't need anything else. I didn't like it because an `EntityManager` with its fluent methods (to construct queries) is harder to mock than a simple interface-based Repository and conflating business logic + data access in the same class is a design style I don't like much.

So if you wanted to hide your `EntityManager` behind a Repository, you had to make your Repository `@Stateful` as well to be able to hold your `EXTENDED` `EntityManager`.

The problem with that approach comes when you have to combine 2 or more repositories in your business logic. As each Repository would have its own `EntityManager`, they would not share their common entities (first level cache) and there you could have a lot of nasty bugs when mixing entities coming from different `EntityManager`s.

So you need to find a way to provide a common `EntityManager` for your repositories. That is where CDI Producers come to the rescue. A producer could be responsible to open a single `EntityManager` for the ConversationScope. This (proxied) `EntityManager` would then be injected in all repositories. In return, repositories would not need to be `@Stateful` anymore. So everything seems right at first but it is not: now that your `EntityManager` are `@ConversationScoped`, you are forced to have a Conversation to use them.

In general, this is not a problem because for each request, there is always a conversation (either transient or long-running if you did call `begin()`).

But what if you are doing some batch processes? There is no request and no conversation scopes available at this time.

And what if managing a long-running conversation is too much for your use case? For example, if you are using JSF with `@ViewScoped` beans and want your `EntityManager` to remain open for AJAX requests that could lead to the initialization of some lazy associations, you now have to `begin()` (and `end()`) a conversation. In that case, it would have been better to have a `@ViewScoped` `EntityManager` where start and end are implicit instead of being explicit with your conversation.

So, none of those 2 problems are impossible to solve: for the batch process, you could register another `@ConversationScoped` context that is only active during batch processes and for the second one, you could replace the `@ViewScoped` with `@ConversationScoped` and `begin()`/`end()` your Conversation manually (and adding CID parameters everywhere it is needed and such). This is more work for the developer.

Maybe another solution would be to scope the `EntityManager` in a dynamic context, one that could be `@ConversationScoped`, `@ViewScoped`, `@RequestScoped` or even `@TransactionScoped` depending on the current situation.

Let's name this special context `@UnitOfWorkScoped` and define it as such:

```java
@Retention(RUNTIME)
@NormalScope(passivating = true)
@DynamicScope({
	@DelegateScope(scope = ThreadScoped.class),
	@DelegateScope(scope = ConversationScoped.class, unless = "#{javax.enterprise.context.conversation.transient}"),
	@DelegateScope(scope = ViewScoped.class, unless = "#{facesContext.viewRoot.transient}"),
	@DelegateScope(scope = RequestScoped.class),
	@DelegateScope(scope = TransactionScoped.class)
})
public @interface UnitOfWorkScoped {}
```

This scope will delegate to the first active context in the following order:

- first, it will check if a ThreadContext is available. The ThreadContext is mainly used in batch/asynchronous processes (like an EJB `@Schedule` or a `@MessageDriven`), it guarantees that a single `EntityManager` will be used for that Thread. The ThreadContext must be activated manually through a CDI interceptor (`@ThreadContextual`) on your method/type.

- then, if no ThreadContext was available, `@UnitOfWorkScoped` will delegate to the Conversation context only if it is long-running (non-transient). Why make this restriction? Because I think that a ConversationScope is greater than a JSF ViewScope (so it must come before the ViewScope) but as there is always a Conversation available (transient) per request, transient conversation (~ request) would always take precedence over the ViewScope that comes after

- then, if no long-running conversation is available, `@UnitOfWorkScoped` will check if there is a non-transient `@ViewScoped` available. Here, the restriction over non-transient is made because storing things in a ViewScope of a transient/stateless view has no point and even gives warnings.

- then, if no ViewScope is available, the `EntityManager` will stay open for the whole request.

- finally, if no Request is available, the `EntityManager`will be bound to the current transaction (which gives back the usual way of using an `EntityManager`, the `TRANSACTION` type)

The scope is marked as passivating as it can potentially delegate to other passivating scopes (conversation, view).

## Warning
I've had great success using this mechanism but you need to be sure conditions for evaluating to which scope UnitOfWorkScope will delegate to do not change (unintentionally) during your processing. For example, if you plan on using a long-running conversation, you **must** call `Conversation.begin()` before any use of the `EntityManager` (otherwise, you could have multiple EM in different scopes, one for example in Request and one in Conversation after you called begin()).

## Remark
I've included a JSF scope (`@ViewScoped`) but not `@FlowScoped`, why? The new JSF 2.2 `@FlowScoped` would theoretically make a good candidate for a delegate scope because it has a definite lifecycle. But the implementation of this scope is problematic and won't work with this `@DynamicScope` because it is the only scope I know of that has attributes (for example `@FlowScoped("MyFlow")`), and the Context implementation uses those attributes to keep the scopes for different flows separated. So, if you don't provide those attributes, you will get a NullPointerException. This is real problem that I would like to be addressed in future revision of the JSF specification.