---
title: "Creating Overloaded Methods in TypeScript"
datePublished: Sun Apr 16 2023 08:13:04 GMT+0000 (Coordinated Universal Time)
cuid: clgj4qou6000p09ie0yuycd57
slug: creating-overloaded-methods-in-typescript
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/Oxl_KBNqxGA/upload/4231fa3b0caf9d8851baf98d53002ece.jpeg
tags: programming-blogs, typescript, programming-languages, programming-tips

---

For the majority of my journey in software development, I have used languages that have supported method overloading. I have found this to be useful, so I have been wondering how to implement something similar in [TypeScript](https://www.typescriptlang.org/). How can we have a method can be declared multiple times with different parameters in a language which does not support method overloading in the traditional way?

> After posting, it was brought to my attention that there is another way. I have added an update at the end to cover this approach.

## The Problem

I wanted to create a class to build queries for [DynamoDB](https://aws.amazon.com/dynamodb/). DynamoDB is a NoSQL database that indexes each item by two keys. A partition key and a sort key. When querying a DynamoDB table you always supply a partition key, and you optionally supply a sort key along with an operator such as 'greater than'. Another option is to supply two sort key values to provide a range.

## The C# Solution

In [C#](https://dotnet.microsoft.com/en-us/languages/csharp), we could define as follows:

```c#
enum SortKeyOperator
{
    EQUALS,
    LESS_THAN,
    LESS_THAN_OR_EQUAL,
    GREATER_THAN_OR_EQUAL,
    GREATER_THAN,
    BEGINS_WITH,
}

class QueryBuilder
{
    public void Build(string partitionKeyValue)
    {
    }

    public void Build(
        string partitionKeyValue,
        string sortKeyValue)
    {
    }

    public void Build(
        string partitionKeyValue,
        SortKeyOperator sortKeyOperator,
        string sortKeyValue)
    {
    }

    public void Build(
        string partitionKeyValue,
        string sortKeyFromValue,
        string sortKeyToValue)
    {
    }
}
```

This is allowed, as the combination of parameters means that each method signature is unique, even though the name of the method is not. When using the Visual Studio [IDE](https://www.freecodecamp.org/news/what-is-an-ide-in-programming-an-ide-definition-for-developers/) the intellisense prompts as follows.

![An example of how overloaded methods appear in Visual Studio](https://github.com/andybalham/blog-source-code/blob/master/blog-posts/images/creating-overloaded-methods-in-typescript/csharp-basic-overload-dropdown-1.png?raw=true)

This allows the user to scroll through the various overloaded versions of the method.

![An example of how overloaded methods appear in Visual Studio](https://github.com/andybalham/blog-source-code/blob/master/blog-posts/images/creating-overloaded-methods-in-typescript/csharp-basic-overload-dropdown-2.png?raw=true)

By adding documentation to the methods, you can clearly communication the intended use of each overload.

## TypeScript Attempt No.1 - Separate Methods

The simplest way I could think of to try to replicate method overloading is to have separate methods that share a common prefix. In this case, `buildWith`. The result is as follows:

```typescript
class QueryBuilder {
  buildWithPartitionKeyOnly(partitionKeyValue: string) {...}

  buildWithSortKey(partitionKeyValue: string, sortKeyValue: string) {...}

  buildWithComparison(
    partitionKeyValue: string,
    sortKeyOperator: SortKeyOperator,
    sortKeyValue: string
  ) {...}

  buildWithRange(
    partitionKeyValue: string,
    sortKeyFromValue: string,
    sortKeyToValue: string
  ) {...}
}
```

This would result in the following prompt when using VS Code:

![An example of how a named method appears in VS Code](https://github.com/andybalham/blog-source-code/blob/master/blog-posts/images/creating-overloaded-methods-in-typescript/typescript-basic-overload-1.png?raw=true)

I actually think this approach has some merit. The explicit naming provides some level of self-documentation. A downside is that the underlying implementation might need either some duplication in the separate methods, or some common code outside them.

## TypeScript Attempt No.2 - Optional Parameters

Another approach is to use optional parameters and [deconstructed parameters](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring_assignment). We can define a single method with a single object parameter, and we can make the sort key parameters all optional. The result is as follows:

```typescript
build({
  partitionKeyValue,
  sortKeyValue,
  sortKeyComparison,
  sortKeyRange,
}: {
  partitionKeyValue: string;
  sortKeyValue?: string;
  sortKeyComparison?: {
    operator: SortKeyOperator;
    value: string;
  };
  sortKeyRange?: {
    fromValue: string;
    toValue: string;
  };
}) {
  if (sortKeyValue) {
    // Handle case where we match by value equality
  } else if (sortKeyComparison) {
    // Handle case where we match by comparison
  } else if (sortKeyRange) {
    // Handle case where we match by range
  } else {
    // Handle case where we match just by primary key
  }
}
```

Whilst this works, it isn't obvious to the caller what combination of parameters should be used to get the various outcomes. For example, can `sortKeyRange` be used with `sortKeyValue`? The only way to know this, is to look inside the method. Not ideal. Can we do better?

## TypeScript Attempt No.3 - Naive Discriminated Types

TypeScript allows you to define that a value can be one of a set of types, for example:

```typescript
var v: number | string;
```

Can we take advantage of this to give the callers of the method a set of exclusive choices, so that they do not need to look inside the method to work out how to use it?

Below was my first effort:

```typescript
build({
  partitionKeyValue,
  sortKeyCriteria,
}: {
  partitionKeyValue: string;
  sortKeyCriteria?:
    | {
        value: string;
      }
    | {
        comparison: {
          operator: SortKeyOperator;
          value: string;
        };
      }
    | {
        range: {
          fromValue: string;
          toValue: string;
        };
      };
}) {
  if (sortKeyCriteria) {
    if ('value' in sortKeyCriteria) {
      // Handle case where we match by value equality
    } else if ('comparison' in sortKeyCriteria) {
      // Handle case where we match by comparison
    } else if ('range' in sortKeyCriteria) {
      // Handle case where we match by range
    } else {
    }
  } else {
    // Handle case where we match just by primary key
  }
}
```

Here we use the `in` operator to work out which of the types has been specified. This all seemed to be working as I expected until I tried the following:

```typescript
queryBuilder.build({
  partitionKeyValue: "pk",
  sortKeyCriteria: {
    value: "sortKeyValue",
    range: {
      fromValue: "sortKeyValue1",
      toValue: "sortKeyValue2",
    },
    comparison: {
      operator: SortKeyOperator.GREATER_THAN,
      value: "sortKeyValue",
    },
  },
});
```

I was expecting a compiler error, as I had specified all three options. However, clearly TypeScript does not work that way. What I had I done wrong?

## TypeScript Attempt No.4 - Discriminated Types Done Properly

The solution came from an [example in the TypeScript playground](https://www.typescriptlang.org/play#example/discriminate-types). What I needed to do was define a value that would discriminate the types. The result is as follows:

```typescript
build({
  partitionKeyValue,
  sortKeyCriteria,
}: {
  partitionKeyValue: string;
  sortKeyCriteria?:
    | {
        type: 'value';
        value: string;
      }
    | {
        type: 'comparison';
        operator: SortKeyOperator;
        value: string;
      }
    | {
        type: 'range';
        fromValue: string;
        toValue: string;
      };
}) {
  if (sortKeyCriteria?.type === 'value') {
    // Handle case where we match by value equality
  } else if (sortKeyCriteria?.type === 'comparison') {
    // Handle case where we match by comparison
  } else if (sortKeyCriteria?.type === 'range') {
    // Handle case where we match by range
  } else {
    // Handle case where we match just by primary key
  }
}
```

Now when using the class in VS Code, when you select the `type` you get the corresponding options. For example, when using `range` you get prompted for the relevant `fromValue` and `toValue` values.

![An example showing how using the type filters the options shown](https://github.com/andybalham/blog-source-code/blob/master/blog-posts/images/creating-overloaded-methods-in-typescript/typescript-discriminated-type-prompt.png?raw=true)

Now we have a single method that presents a set of mutually exclusive choices. The caller cannot get the parameters wrong and doesn't need to look inside the method.

## Documentation

One downside to using deconstructed parameters is that I could not find a way to document them well using [JSDoc](https://en.wikipedia.org/wiki/JSDoc). The best I could come up with was the following.

```TypeScript
/**
 * Builds a query based on the key criteria supplied.
 * @param param0 Key criteria
 */
build({ partitionKeyValue, sortKeyCriteria }: {...}) {...}
```

This resulted in the following prompt in VS Code, which does give some indication of the options via the `type` values.

![An example showing basic documentation of the method with deconstructed parameters](https://github.com/andybalham/blog-source-code/blob/master/blog-posts/images/creating-overloaded-methods-in-typescript/typescript-deconstructed-jsdoc.png?raw=true)

I may have been missing something, but for this reason I found myself quite liking the solution with separate methods. That approach was easy to document and also somewhat documented itself with the verbose names.

![An example of how a named method appears in VS Code](https://github.com/andybalham/blog-source-code/blob/master/blog-posts/images/creating-overloaded-methods-in-typescript/typescript-overload-1.png?raw=true)

Here we can see that the individual parameters can be documented in the same way that they can be in the C# example, as shown below.

![An example of how overloaded methods appear in Visual Studio](https://github.com/andybalham/blog-source-code/blob/master/blog-posts/images/creating-overloaded-methods-in-typescript/csharp-overload-dropdown-2.png?raw=true)

## Summary

In this post we looked at various ways that we can implement some form of method overloading in TypeScript and compared these with an equivalent in C#. My feeling is that for internal libraries, I would favour the discriminated type approach. However, for external libraries, I feel that the ability to fully document means that the simple, multi-named approach would be better. Behind the scenes, these methods may map onto a single discriminated type method, in order to keep functionality together.

## Update

After posting this, it was pointed out to me that there is another way that we can implement method overloading in TypeScript.

The way we can do it is by defining an `args` parameter with the [JavaScript spread syntax](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_syntax) and a list of possible parameters. In our example this would be the following:

```TypeScript
build(
  ...args:
    | [partitionKeyValue: string]
    | [partitionKeyValue: string, sortKeyValue: string]
    | [
        partitionKeyValue: string,
        sortKeyOperator: NumericSortKeyOperator,
        sortKeyValue: string
      ]
    | [
        partitionKeyValue: string,
        sortKeyFromValue: string,
        sortKeyToValue: string
      ]
) {...}
```

In VS Code, this gives an experience very similar to that we had for C#:

![VS Code showing the list of overloaded methods](https://github.com/andybalham/blog-source-code/blob/master/blog-posts/images/creating-overloaded-methods-in-typescript/arg-list-overload-dropdown.png?raw=true)

We still have the problem of how to know which overload is being called. I solved this by building up a signature string from the arguments types and switching on the result.

```TypeScript
const signature = args
  .map((arg) => typeof arg)
  .reduce((accumulator, argType) => `${accumulator}${argType}|`, '|');

switch (signature) {
  case '|string|':
    // Handle case where we match by partition key only
    break;
  case '|string|string|':
    // Handle case where we match by compound key
    break;
  case '|string|string|string|':
    // Handle case where we match by range
    break;
  case '|string|number|string|':
    // Handle case where we match by comparison
    break;
  default:
    throw new Error(`Unhandled signature`);
}
```

One thing I did have to change was the type of `enum`. Originally, it was a set of strings, but this would cause a clash of signatures. I changed it for a set of integers and this avoided the issue.

I used array destructuring to access the values as follows:

```TypeScript
case '|string|number|string|':
  // Handle case where we match by comparison
  {
    const [partitionKeyValue, sortKeyOperator, sortKeyValue] = args;
    // Call the method implementation
  }
  break;
```

Whilst TypeScript does infer the types, it does not discriminate. So it can only assert that some values are one of a set:

![VS Code showing the inferred types](https://github.com/andybalham/blog-source-code/blob/master/blog-posts/images/creating-overloaded-methods-in-typescript/arg-list-inferred-typing.png?raw=true)

This approach suffers from the documentation issue that other advanced approaches do. My feeling overall is that, although it gives a similar intellisense experience, it falls down when implementing the underlying functionality and I would still be tempted to go down the explicit naming route with discriminated types underneath.
