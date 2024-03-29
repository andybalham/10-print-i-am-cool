---
title: "Encapsulation Not Always Desirable"
datePublished: Sun Jul 02 2023 16:06:15 GMT+0000 (Coordinated Universal Time)
cuid: cljlmjsvz000909mg8fhhdhob
slug: encapsulation-not-always-desirable
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/-kr8XPYuSI8/upload/b8542d92ca1f3c28c5c8d48e969aa6bb.jpeg
tags: programming-blogs, programming-tips, learning-journey

---

From my earliest experience with [object-oriented programming](https://en.wikipedia.org/wiki/Object-oriented_programming), I learnt that one of the biggest benefits of the paradigm was the concept of encapsulation. Lately, my experience with developing in JavaScript and creating libraries have led me to come to the conclusion that it is not always desirable.

## What is encapsulation

The following links provide an admirable overview of the concept:

- [Stackify: What is Encapsulation](https://stackify.com/oop-concept-for-beginners-what-is-encapsulation/)
- [Wikipedia: Encapsulation](<https://en.wikipedia.org/wiki/Encapsulation_(computer_programming)#An_information-hiding_mechanism>)

The following in particular sums up the idea:

> Under the definition that encapsulation "can be used to hide data members and member functions", the internal representation of an object is generally hidden from view outside of the object's definition. Typically, only the object's own methods can directly inspect or manipulate its fields. Hiding the internals of the object protects its integrity by preventing users from setting the internal data of the component into an invalid or inconsistent state. A supposed benefit of encapsulation is that it can reduce system complexity, and thus increase robustness, by allowing the developer to limit the interdependencies between software components

## Encapsulation is the default

The majority of my object-oriented programming experience is in Java and C#. In both these languages, the default visibility for class members is non-public. That is, they cannot be access by any class without a special relationship with that class. Such relationships are a sub-class or a class in the same packaging unit. This implicitly pushes the developer to conceal as much as possible.

When I encountered TypeScript, I was surprised to find that the opposite was true. I had to explicitly make any members private. I had to consciously make the decision to hide them from potential users of the class.

## When encapsulation gets in the way

This subtle change made me question my default thinking. This was coupled with my experience of using a third-party component in a way similar to the following hypothetical code:

```TypeScript
const myClient = new ServiceClient({ region: 'eu-west-2'});

// ...

myClient.region; // Not accessible now, although it was provided earlier
```

The component was hiding information that I had provided. What was the point of doing this? How was I going to misuse this information? Now I had to pass around the information that existed inside another parameter that was passed with it.

I also had the experience of trying to subclass a class I had published as part of an `npm` package. I had diligently hidden everything deemed 'not essential', but now had cut off the ability to extend it.

All this caused me to further question how I thought about member visibility.

## Don't throw the baby out with the bathwater

Should we just make everything public? Of course not. Anything public forms part of the contract of your class and you should be committed to honouring that as best you can. Once you have published, then breaking that contract could result in very unhappy clients. If you are using [Domain-driven Design](https://en.wikipedia.org/wiki/Domain-driven_design), then you will also need to implement some business rules to keep your domain objects consistent. Again, encapsulation plays a vary valuable role here.

What I am thinking about here are primarily read-only properties that have been hidden without thought. Perhaps because the language made that the default.

## Sometimes privacy is just a facade

As it turns out, sometimes privacy is just a facade. Take C# for example, where you can define this seemingly well-encapsulated class.

```c#
class MyEncapsulatedClass
{
    private int PrivateProperty { get; set; }

    public MyEncapsulatedClass(int myPropertyValue)
    {
        PrivateProperty = myPropertyValue;
    }
}
```

However, using [reflection](https://www.howtogeek.com/devops/what-is-reflection-in-programming/) you can still access the supposedly-private value.

```c#
var myEncapsulatedInstance = new MyEncapsulatedClass(666);

var myEncapsulatedClass = typeof(MyEncapsulatedClass);
var privateProperty =
    myEncapsulatedClass.GetProperty(
        "PrivateProperty", BindingFlags.NonPublic | BindingFlags.Instance);
var privatePropertyValue =
    privateProperty.GetValue(myEncapsulatedInstance, null);

Console.WriteLine("privatePropertyValue=" + privatePropertyValue);
```

I concede that you have to do some work here to get the value. The point I am trying to make is that you might want to check your language before relying on encapsulation for anything security-related.

## Summary

My experience with TypeScript's public-by-default approach led me to question my default position of hiding everything possible in a class. This was combined with my experience of being frustrated with using libraries that hid information unnecessarily.

Now for utility libraries or packages, I try to keep things as open as possible. Where properties are passed in on construction, I try to make them public and read-only. Internal structures are made private by conscious choice, only if exposing them would make for a fragile contract.

When implementing [Domain-driven Design](https://en.wikipedia.org/wiki/Domain-driven_design) business rules, encapsulation becomes necessary in order to enforce those rules and keep the domain model consistent. Again, this is a conscious choice.

What I also learned, is that trying different programming languages can make you think differently and challenge your own assumptions and habits. This can only be a good thing.
