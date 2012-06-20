---
layout: post
title: "Use delegation to write map/filter in Java"
date: 2010-03-01 19:32
comments: true
categories: java, functional programming, design patterns 
---

## The problem

In Java, imagine you have a list of `User` objects, each encapsulates the user's `id`, `first name`, `last name` and `age`. Then you want to call a web service `UserService.deleteUsersByIds(List<Integer> userIds)` to delete the users from your data store. It doesn't sound too hard, does it? All you need to do is to transform you `List<User>` to `List<Integer>`. So you go ahead and write the following code:

```java
List<Integer> ids = new ArrayList<Integer>(users.size());
for (User user : users) {
  ids.append(user.getId());
}
```

Then you go ahead and use your `ids` list, and everything is fine and dandy.

However, two minutes later, you find yourself having to provide another API method with a list of user's names in String. So, again, you exercise your *CSC101* skill:

```java
List<String> names = new ArrayList<String>(users.size());
for (User user : users) {
  names.append(new StringBuilder(user.getFirstName()).append(" ").append("user.getLastName()));
}
```

Now, something else comes along and you need to write a piece of code that returns a list of names that belong to people who are under 21 years of age in the list...You get the idea. Well, things get boring pretty quickly.

As it turns out, these are two very important functions in [functional programming](http://en.wikipedia.org/wiki/Functional_programming) map and filter.

* `map(coll, f)` "loops" over the collection, calls the function f on each element, and add the return of the `f(element)` to the return collection.
*  `filter(coll, f)` "loops" over the collection, calls `f(element)`, and only add element to the return list when `f(element)` returns `true`

## Use delegation for generic-ity

Now we take our first step in designing our generic map function:

```java
<FromType, ToType> List<ToType> map(ArrayList<FromType> list) {
  List<ToType> retval = new ArrayList<ToType>(list.size());
  for (FromType item : list) {
    [...]
  }
  return retval;
}
```

What we left out in the above code snippet is how the input is mapped to the output. This is where delegates come in. Unfortunately, Java doesn't have the language-level delegate. We need to design an interface for this delegate.

```java
interface MapDelegate<FromType, ToType> {
  ToType map(FromType obj);
}
```

The delegate is parameterized (to provide more type safety) with `FromType` and `ToType`. `FromType` is the type of the objects in the original list, and `ToType` is the type of objects in the mapped list. Now we need to change our method signature to incorporate the delegate.

```java
<FromType, ToType> List<ToType> map(ArrayList<FromType> list, MapDelegate<FromType, ToType> mapDelegate) {
  List<ToType> retval = new ArrayList<ToType>(list.size());
  for (FromType item : list) {
    retval.add(mapDelegate.map(item));
  }
  return retval;
}
```

Now the client code will look like this:

```java
List<User> users = getUserListFromSomeWhere();
List<String> ids = map(users, new MapDelegate<User,String>() {
  public String map(User obj) {
    return new StringBuilder(user.getFirstName()).append(" ").append("user.getLastName()).toString();
  }
});
```

Similarly, we can write a filter function:
```java
<T> List<T> filter(List<T> list, FilterDelegate<T> filterDelegate) {
  List<T> retval = new ArrayList<T>(list.size());
  for (T item : list) {
    if (filterDelegate.filter(item)
      retval.add(item);
  return retval;
}
```

```java
interface FilterDelegate<T> {
  boolean filter(T item);
}
```

## What about return value creation?

Use delegation, we can separate the parts of an algorithm in terms of their interfaces and leave the implementation to the caller. However, given the above filter and map methods, what if I don't want the return type to be `ArrayList`? What if I want a `LinkedList` or a `HashSet`? Doesn't the statement

```java
  List<T> retval = new ArrayList<T>(list.size());
```

an implementation by itself?

Absolutely! For more flexibility, the "new" statement in the implementation body has to be delegated out as well. We introduce a `ReturnDelegate` interface:

```java
interface ReturnDelegate<R extends Collection<?>> {
  R createReturnCollection();
}
```

and plug in the return delegate to the map method:
```java
<FromType, ToType, R extends Collection<?>> R map(Collection<FromType> coll, MapDelegate<FromType, ToType> mapDelegate, ReturnDelegate<R> returnDelegate) {
  R retval = returnDelegate.createReturnCollection();
  for (FromType item : list) {
    retval.add(mapDelegate.map(item));
  }
  return retval;
}
```

Now the actual implementation has been completely separated. I know you can probably achieve flexibility without return delegate with the use of reflection, but on some systems (like GWT, which is what I'm working on and what this code is originally designed for), reflection is off limits.

