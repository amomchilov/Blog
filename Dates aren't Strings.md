# MOVED ➡️

This content has been moved to [my site](http://momchilov.ca/2020/07/01/dates_arent_strings.html).

# Dates aren't Strings.

At least seveal times a week, I see StackOverflow questions related to the user struggling to do some simple date processing on strings. The solution is always the same, and it's always simple: don't store dates as strings.

Numbers aren't strings. Dates aren't strings. URLs aren't strings. Just beacuse all of these things can be **expressed** as strings, doesn't mean they should **be** strings in your systems. All those things could also be encoded in binary (indeed, at bottom, all data is binary encoded in memory), a PNG image of hand-written French cursive writing, or a video clip of a sailor signaling using [semaphores](https://en.wikipedia.org/wiki/Semaphore_(programming)).

But that would be nonsense. Use `Int` (or `Double`/`Decimal`), `Date`, and `URL`, respectively.

## Common currency types

Swift has a core set of types known as "common currency types". The set is loosely defined, but consists of common types we can use to transact with other peices of code. They're limited enough that everybody should be able to have access to them all, but varied enough that they're expressive of most basic needs. In Swift, these types are:

| Type                            | Purpose                                                                        |
| ------------------------------- | ------------------------------------------------------------------------------ |
| `Int`                           | Whole numbers (even if they don't need to negative, just for convenience [^1]) |
| `Double`                        | Non-whole numbers when you don't mind decimal imprecision                      |
| `Foundation.Decimal`            | Base-10 numbers with up to 32 digits of precision                              |
| `String`                        | Unformatted text                                                               |
| `Foundation.NSAttributedString` | Formatted text                                                                 |
| `Foundation.Date`               | Human dates (with no implied timezone)                                         |
| `Foundation.URL`                | A URL to the web, or a local file reference                                    |
| `Bool`                          | Modeling `true`/`false`                                                        |
| `Optional<Wrapped>`             | Modeling the absence of a value (as `nil`)                                     |
| `Result<Success, Failure>`      | Modeling a computation that can succeed or fail                                | 
| `Array<Element>`                | An ordered collection of `Element`                                             |
| `Set<Element>`                  | An unordered collection unqie `Element`s, with fast `contains` look-ups        |
| `Dictionary<Key, Value>`        | An unordered association between `Key`s and `Value`s                           |

If push comes to shove, you could find a way to express all of these as strings, but that doesn't mean you should. You won't see a date-arithemtic API in `Foundation` that deals with `String`. Imagine if you had an algorithm that expected an array, but took it as a common seperated string like `"1, 2, 3"` instead of `[1, 2, 3]`. It just doesn't make any sense.

Interestingly, there's no standard type for human curreny/money. I often see `Int`/`Double`/`Decimal` abused for the purpose, by being used directly, rather than making a `Currency` struct that contains one of those. It would be nice if that was standardized, once and for all.

## The problem

`Foundation.Date` is only available within your Swift program. It's not a standardized data type, so all external dates (such as from network calls, databases, files read from disk, etc.) that come into your system will come in as some other format (usually `String` or `Data`).

Rather than letting this implementation detail of your external dependancy (e.g. database) metastisize throughout your code base, isolate it and abstract it.

## The solution

***At the earliest possible moment*** in your database layer, network layer, etc., parse the string date into a proper `Date`. 

Likewise, when you need to export data into external systems, serialize your dates into the necessary formats (e.g. `String`), ***at the last possible moment***.

This way, the rest of your app can be a wonderful walled garden, where dates are dates, and operating on them is simple and wonderful.

[^1]: > Use the Int type for all general-purpose integer constants and variables in your code, even if they’re known to be nonnegative. Using the default integer type in everyday situations means that integer constants and variables are immediately interoperable in your code and will match the inferred type for integer literal values.
  >
  > Use other integer types only when they’re specifically needed for the task at hand, because of explicitly sized data from an external source, or for performance, memory usage, or other necessary optimization. Using explicitly sized types in these situations helps to catch any accidental value overflows and implicitly documents the nature of the data being used.
  
  [The Swift Programming Language / The Basics / Numeric Type Conversion](https://docs.swift.org/swift-book/LanguageGuide/TheBasics.html#ID324)