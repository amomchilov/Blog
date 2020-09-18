# The bad ways

Avoid all of these. If you get linked to one of these sections, read the section, follow the chain of "slightly improved" variants until you end up at the best technique. That way, you can see what every incremental improvement does, and what makes the "best technique" so good.

## `while ...`

There are many loops that can only be structured using a `while` loop, such as reading from a queue until it's empty (whilst concurrently adding to the queue). But if you're using `while` loops and manual counters as a means of iterating all elements of an array, you're doing it wrong. For example:

``` Swift
var counter = 0
while counter < array.count {
    print(counter, array[counter])
    counter += 1
}
```

Why it's bad:
* Complex, obscures the real meaning
* Easy to make a mistake, such as incrementing the counter too early and causing off-by-one errors/crashes.
* Doesn't work for sliced collections, where the indices don't start from zero.
* Doesn't work for collections whose indices aren't `Int`.

Slightly improved: `for i in 0..<array.count {`

## `for i in 0...array.count-1 { }`

Why it's bad:
* If `array` is empty, this will try to form the range `0...(-1)`, which will crash:

	> Fatal error: Can't form Range with upperBound < lowerBound
* Too easy to forget the `-1`.
* Doesn't work for sliced collections, where the indices don't start from zero.
* Doesn't work for collections whose indices aren't `Int`.

Slightly improved: `for i in 0..<array.count {`

## `for i in 0..<array.count {`

Improvements over the previous:

* Support empty arrays gracefully (by forming an empty range)
* Can't forget the `-1`

Why it's bad:

* Too easy to accidentally type `0...array.count`, causing an off-by-one error (by trying to access `array[array.count]`)
	* You might think you're immune, but believe me, I've seen tens of StackOverflow questions about this.
* Doesn't work for sliced collections, where the indices don't start from zero.
* Doesn't work for collections whose indices aren't `Int`.

Improved: `for i in array.indices { }`

# "Proper" techniques

## Iterating over the indices

This is probably *not* what you need. I can't think of many situations in which you'd be interested in the indices of a collection, but not in its elements.

### Using indices when you don't need to

Iterating the indexes of an array when you don't need to, just introduces unecessary complexity. For example in this code:

``` Swift
let array = ["A", "B", "C" ]
for i in array.indices {
	let char = array[i]
	print(char)
}
```

The index `i` isn't actually required. It's only used as a means to an end, to get to `char`. Instead, we can just directly iterate over the elements of the array, skipping the middle man `i`:

``` Swift
let array = ["A", "B", "C" ]
for char in array {
	print(char)
}
```

However, there are times when you *do* also need the index, such as when you want to consume it (e.g. print it for debug information) or use it to mutate the original array. For example:


``` Swift
var array = ["A", "B", "C" ]
for i in array.indices {
	let char = array[i]
	print(char)
	array[i] = " "
}
```

But there are better ways:

* If you need both the elements and the indices, see [Iterating over the indices and the elements](#Iterating-over-the-indices-and-the-elements).
* If you need both the elements and a counter, see [Iterating over the elements, and maintaining a counter](Iterating-over-the-elements,-and-maintaining-a-counter).


### Best practice: `for i in array.indices { }`

If you really do need this, the best way to do this is with the built in [`Collection.indices`](https://developer.apple.com/documentation/swift/collection/1641719-indices) computed property. It's defined on `Collection`, so it's not limited to not just `Array`. Instead it's available to any `Collection`, like `String`, `Set`, `Dictionary`, etc.

In the case of `Array`, it simply returns `0..<self.count`. However, other types implement this computed property differently, so that it's always correct. For example, slices start and end at the correct index into their parent container.

``` Swift
let array = ["A", "B", "C" ]
for i in array.indices {
	print(i)
}
```

Improvements over the "bad ways":

* No off-by-one errors
* Works for slices
* Works for non-integer indexed collection
* Very expressive, reads like English.

## Iterating over the elements, and maintaining a counter

### Best practice: `for (offset, char) in array.enumerated() { }`

Use [`Sequence.enumerated()`](https://developer.apple.com/documentation/swift/sequence/1641222-enumerated). It iterates the elements of a sequence, while maintaining a counter for you and providing it to you via its iterator. This counter is an offset from `0`, but ***it is not an index***. It can only be used as an index for collections with zero-based integer indices. That is not the case for many collections, such as `ArraySlice`, `String`, `Set`, etc.

``` Swift
let array = ["A", "B", "C" ]
for (offset, char) in array.enumerated() {
	print(offset, char)
    _ = array[offset] // ❌ Do not use the offset as an index
}
```

## Iterating over the indices and the elements

### Best practice: `for (index, char) in zip(array.indices, array) { }`

Unlike `Sequence.enumerated()`, this technique provides you with indices that are guaranteed to be valid for use with the collection.

``` Swift
let array = ["A", "B", "C" ]
for (index, char) in zip(array.indices, array) {
	print(index, char)
	_ = array[index] // ✔️ index is a real index
}
```


## Iterating over the elements

### Best practice: `for element in array { }`
Keep it simple:

``` Swift
let array = ["A", "B", "C" ]
for char in array {
	print(char)
}
```

# Summary

There are several ways to iterate sequence/collection in a `for` loop, depending on whether you need only the indices, only the elements, or both the indices and the elements.

It's actually exceptionally rare that you need to iterate over *only* the indices of a collection, and not the associated values. Thus, you'll almost always be using the right column of this table.

|                     | Elements not needed | Elements needed                                                  |
|---------------------|---------------------|------------------------------------------------------------------|
| Indices not needed  | Just don't iterate anything lol | `for element in array { }`                           |
| Counter needed      | `for i in 0... { } `         | `for (offset, element) in array.enumerated() { }`       |
| Indices needed      | `for i in array.indices { }` | `for (index, element) in zip(array.indices, array) { }` |
