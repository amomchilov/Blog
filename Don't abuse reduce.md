# Don't abuse `reduce(_:)`


## First, why `filter(_:)` is so nice

There's an inverse relationship between generality and clarity. APIs that have a single, narrow purpose are very easy to understand from a quick glance. `filter` is one such example. As soon as you see a call to `map`, and without even looking at the sequence it's called on or the closure it's passed, you can instantly know a few things:

1. You know that `input.count <= output.count`
2. You know that the input and output elements will have the same type
3. You know that the output will be a subset of the input, thus there are no elements in the output that weren't in the input

On the other hand, `reduce` is general. Because `reduce` isn't as "limited" as something like `filter`, when you read `reduce` on the screen, you can't automatically infer what that code is doing, unlike with `filter`, where you know that you're just making Array containing a subset of elements.

I suggest that before you use `reduce`, you take a careful look what other APIs are available, and use the most specific one possible for your problem. At the end of this post, I've made a list of common situations I see `reduce` being used, and better alternatives that are available.

## `reduce` is really general.

Generality is good, because it means that `reduce` can be useful in a broad range of cases. While this utility is convenient, it needs to be weighed against the competeing factor, which is potential the loss of readability. Consider that reduce can be used to implement:

1. `count`
2. `isEmpty`
3. `first`
4. `last`
5. `filter(_:)`
6. `map(_:)`
7. `flatMap(_:)`
8. `compactMap(_:)`
9. `prefix(while:)`
10. `prefix(_:)`
11. `suffix(while:)`
12. `suffix(_:)`
13. `dropFirst(_:))`
14. `dropLast(_:)`
15. ... and others

<details><summary>Don't believe me? Here are some sample implementations:</summary>

``` Swift
extension Sequence {
	var shittyFirst: Element? {
		reduce(nil) { acc, element in acc ?? element }
	}
	
	var shittyLast: Element? {
		reduce(nil) { acc, element in element }
	}
	
	var shittyCount: Int {
		reduce(0) { counter, _ in counter + 1 }
	}
	
	var shittyIsEmpty: Bool {
		reduce(true) { _, _ in false }
	}
	
	func shittyFilter(_ predicate: (Element) throws -> Bool) rethrows -> [Element] {
		try reduce(into: []) { acc, element in
			if try predicate(element) {
				acc.append(element)
			}
		}
	}
	
	func shittyMap<T>(_ transform: (Element) throws -> T) rethrows -> [T] {
		try reduce(into: []) { acc, element in acc.append(try transform(element)) }
	}
	
	func shittyFlatMap<SegmentOfResult>(_ transform: (Self.Element) throws -> SegmentOfResult) rethrows -> [SegmentOfResult.Element]
		where SegmentOfResult: Sequence {
		try reduce(into: []) { acc, element in acc.append(contentsOf: try transform(element)) }
	}
	
	func shittyCompactMap<ElementOfResult>(_ transform: (Self.Element) throws -> ElementOfResult?) rethrows -> [ElementOfResult] {
		try reduce(into: []) { acc, element in
			if let element = try transform(element) {
				acc.append(element)
			}
		}
	}
	
	func shittyPrefix(while predicate: (Element) throws -> Bool) rethrows -> ArraySlice<Element> {
		var done = false
		return try reduce(into: []) { acc, element in 
			if done { return }
			if try predicate(element) {
				acc.append(element)
			} else {
				done = true
			}
		}
	}
	
	func shittyPrefix(_ maxLength: Int) -> ArraySlice<Element> {
		var done = false
		var i = 0
		return reduce(into: []) { acc, element in
			if done { return }
			if i == maxLength {
				done = true
			} else {
				acc.append(element)
				i += 1
			}
		}
	}
}

print(Array(1...5).shittyCount)
print(Array(1...5).shittyIsEmpty)
print(Array(1...5).shittyFirst as Any)
print(Array(1...5).shittyLast as Any)
print(Array(1...5).shittyFilter { $0.isMultiple(of: 2) })
print(Array(1...5).shittyMap { $0 + 10 })
print(Array(1...5).shittyFlatMap { [123, $0, 321] })
print([1, nil, 2, nil, 3].compactMap { $0 })
print(Array(1...5).shittyPrefix { $0 < 3 })
print(Array(1...5).shittyPrefix(3))
```
</details>

## Comparison to `forEach`

`reduce` is pretty much exactly as powerful as `forEach`. The `reduce`'s accumulator can be used to maintain state, but you can do that with any old variable, just by capturing in the closure you pass to `forEach`. Neither of these APIs is able to skip iterations or bail early.

In fact, Ruby's `Enumerable` module (their equivalent of equivalent of Swift's `Sequence` protocol) only requires that you implement an `each` method (equivalent of `forEach`), which they use to implement all other APIs within `Enumerable`. However, this is far from perfect, because it means that even simple operations like `count`, `first`, `last`, `isEmpty`, etc. become `O(n)`. So often times, they'll be implemented seperately to take advantage of the implementation details of the particular sequence's implementation, for faster performance.

## Comparison to `for`

`reduce` is not as powerful as `for` loops, however. `for` loops can support `continue`, `break` and `return` (from the parent scope, not merely the `for` loop). You can use `try`/`catch` to mimic a `break` or `return`, but that's kinda cheating :p.

## Common uses of `reduce` and their alternatives

### Group values by a key

``` Swift
let words = ["Apple", "Axe", "Bark", "Bench", "Chair", "Cat"]
let wordsByFirstLetter = words.reduce(into: [:]) { acc, word in acc[word.first!, default: []].append(word) }
print(wordsByFirstLetter)
```

Better alternative: `Dictionary.init(grouping:by:)`

``` Swift
let words = ["Apple", "Axe", "Bark", "Bench", "Chair", "Cat"]
let wordsByFirstLetter = Dictionary(grouping: words, by: { $0.first! })
print(wordsByFirstLetter)
```

### Building up a dictionary from a sequence of keys and values

``` Swift
let keys = ["a", "b", "c"]
let values = [1, 2, 3]
let dict = zip(keys, values).reduce(into: [:]) { acc, pair in acc[pair.0] = pair.1 }
print(dict)
```

Better alternative: `Dictionary.init(uniqueKeysWithValues:)`

``` Swift
let keys = ["a", "b", "c"]
let values = [1, 2, 3]
let dict = Dictionary(uniqueKeysWithValues: zip(keys, values))
```

### Keeping complex state in the accumulator

I can't think of a concrete example right now, but I've seen people trying to wrangle really complex state (usually a tuple) as the accumulator of a reducation, with the aim of having their entire compuration expressed in one expression, like this code to get an array of every other element:

``` Swift
array.reduce(into: (toggler: true, result: []) { pair, element in
    if pair.toggler {
    	pair.result.append(element)
    }
    
    pair.toggler.toggle()
}.result
```

Instead, it's nicer to just use local mutable variables to express your state, and only use the accumulator for what matters towards the final result (though using `filter` would be *even* better for this case):

``` Swift
var toggler = true
array.reduce(into: []) { acc, element in
    if toggler {
    	acc.append(element)
    }
    
    toggler.toggle()
}
```

