# The Grocery Store analogy

Central to my explanation of hashed collections is the analogy of a grocery store.

Suppose you want to go to the store to pickup some milk, how do you find it?

## Linear search

Do you start at one end of the store, look at *every* item, checking if it's milk, until you find the milk you're looking for?

Of course not. But unfortunately, this is exactly what happens every time we use `Array.contains(_:)`. It loops through the whole array, comparing every element against the element you're looking for, until a match is found.

For an unsorted array with `n` items, on average it will take `n/2` "equality checks" to find an item you're looking for. Some of the items will be near the start, some will be near the end, but on average it'll be near the middle, hence the division by 2.

Because the time this takes scales linearly with the number of elements (e.g. running `contain(_:)` on an array that's 5x bigger, will take 5x longer), the execution time is linearly proportional to the input size, hence this is called "linear search". It get's real slow, real fast. You should avoid it for any non-trivial data set sizes.

## Using a heuristic

So what do you do in the store? You know that milk would be found in the dairy isle. So there's no need to check for milk in the produce section, in the deli section, etc. You're using *heuristic*. A trick that lets you get clever and reduce the amount of 1-by-1 equality checking that you need to do.

This is precisely what `Hashable` is all about. A hash function is used to "summarize" a value, in a way that broadly describes it. Suppose you're searching for an item in a collection, and you know that the item's hash value is `x`. Just like with the grocery isles, you can optimize your search to only check within the set of items that also have a hash value of `x`. If a set of items has some other hash value, we know that we don't need to bother searching within it: our item isn't there, just like how our milk isn't beside the bananas in the produce section.

# In practice

So let's code up a little demonstration of both of these.

Here's our data model:

``` Swift
enum ProductCategory { 
	case bakingNeeds
	case dairy
	case produce
}

struct Product {
	let name: String
	let category: ProductCategory
}
```

## The slow store

Let's make a slow store:

``` Swift
class SlowStore {
	private let products: [Product] = [
		Product(name: "Sugar", category: .bakingNeeds),
		Product(name: "Cheese", category: .dairy),
		Product(name: "Bananas", category: .produce),
		Product(name: "Flour", category: .bakingNeeds),
		Product(name: "Apples", category: .produce),
		Product(name: "Milk", category: .dairy),
	]
	
	func contains(productNamed desiredProductName: String) -> Bool {
		self.products.contains(where: { product in 
			let isFound = desiredProductName == product.name
			
			if isFound { print("Found milk! :)") }
			else { print("\(product.name) isn't \(desiredProductName) :(") }
			
			return isFound
		})
	}
}
```

As you can see, given the `String` name of a product, our store can perform a linear search to see if it contains the product we're looking for. Here's an example usage:

```
let store = SlowStore()
store.contains(productNamed: "Milk") // => true
```

Which prints:

> Sugar isn't Milk :(
>
> Cheese isn't Milk :(
> 
> Bananas isn't Milk :(
> 
> Flour isn't Milk :(
> 
> Apples isn't Milk :(
> 
> Found milk! :) 

As you can see, we got rather unlucky. We had to look through the whole store before we could find our milk. This is one of the worst cases (the worst being when the item doesn't exist, but there's no way to know without *testing them all*).

## The fast store

So let's improve this by taking advantage of the extra information we know about isles.

``` Swift
class FastStore {
	private let bakingNeedsProducts = [
		Product(name: "Sugar", category: .bakingNeeds),
		Product(name: "Flour", category: .bakingNeeds),
	]
	
	private let dairyProducts: [Product] = [
		Product(name: "Cheese", category: .dairy),
		Product(name: "Milk", category: .dairy),
	]
	
	private let produceProducts = [
		Product(name: "Bananas", category: .produce),
		Product(name: "Apples", category: .produce),
	]
	
	func contains(productNamed desiredProductName: String, inIsle isle: ProductCategory) -> Bool {
		let candidateProducts: [Product]
		
		switch isle {
		case .bakingNeeds:
			print("Only searching within the baking needs isle.")
			candidateProducts = self.bakingNeedsProducts
			
		case .dairy:
			print("Only searching within the dairy isle.")
			candidateProducts = self.dairyProducts
			
		case .produce:
			print("Only searching within the produce isle.")
			candidateProducts = self.produceProducts
		}
		
		return candidateProducts.contains(where: { product in 
			let isFound = desiredProductName == product.name
			
			if isFound { print("Found milk! :)") }
			else { print("\(product.name) isn't \(desiredProductName) :(") }
			
			return isFound
		})
	}
}
```

You can that the fast store organizes its products by sorting them into separate isles. To search for a product in the store, you need to know more than just the product name; you also need to know what the appropriate isle is for that product:

``` Swift
let store = FastStore()
store.contains(productNamed: "Milk", inIsle: .dairy) // => true
```

> Only searching within the dairy isle.
> 
> Cheese isn't Milk :(
> 
> Found milk! :)

As you can see, by narrowing down the search to only a single isle, we had to linear search through way fewer products. But since few product isles contain only a single product, some linear search is still necessary (although over smaller data sets, which is what makes this so much faster).

## Limitiations
There are two main limitations to this code:

1. There are a finite number of isles. They're limited both by the enum cases, and by the number of arrays (which are currently hardcoded to be one per case).
2. The concept of "product" and "isle" don't generalize to other domains where we might want to search for things quickly. E.g. searching for streets within cities, people within company departments, movies by genre, etc.

## Allowing a larger number of isles

To overcome limitation #1, we can change the multiple hard-coded arrays into an array of arrays. We'll replace the enum/switch statements with integers and simple array subscripting:

``` Swift
class FastStore2 {
	private let products = [
		[ // Position 0: baking needs
			Product(name: "Sugar", category: .bakingNeeds),
			Product(name: "Flour", category: .bakingNeeds),
		],
		[ // Position 1: dairy products
			Product(name: "Cheese", category: .dairy),
			Product(name: "Milk", category: .dairy),
		],
		[ // Position 2: produce
			Product(name: "Bananas", category: .produce),
			Product(name: "Apples", category: .produce),
		]
	]
	
	func contains(productNamed desiredProductName: String, inIsle isle: Int) -> Bool {
		let candidateProducts = self.products[isle]
		
		return candidateProducts.contains(where: { product in 
			let isFound = desiredProductName == product.name
			
			if isFound { print("Found milk! :)") }
			else { print("\(product.name) isn't \(desiredProductName) :(") }
			
			return isFound
		})
	}
}

let store = FastStore2()
print(store.contains(productNamed: "Milk", inIsle: 1))
```

While the named enum cases were more pleasant to read than abstract isle numbers, the abstract nature of the integers is actually a perk not a drawback: we can generalize this code to search for any kind of items that belong to groups identified by arbitrary integers.

## Generalizing the lookup

"Isle" is a term specific to the grocery store domain. If the domain was employees of a business, the categories might be called "departments" instead of "isles". Instead, we'll use the generalized term: "buckets"

``` Swift

struct ItemCollection<T> {
	let buckets: [[T]]
	
	func contains(where predicate: (T) -> Bool, inBucket bucketNumber: Int) -> Bool {
		return self.buckets[bucketNumber].contains(where: predicate)
	}
}

class FastStore3 {
	private let products = ItemCollection<Product>(buckets: [
		[ // Position 0: baking needs
			Product(name: "Sugar", category: .bakingNeeds),
			Product(name: "Flour", category: .bakingNeeds),
		],
		[ // Position 1: dairy products
			Product(name: "Cheese", category: .dairy),
			Product(name: "Milk", category: .dairy),
		],
		[ // Position 2: produce
			Product(name: "Bananas", category: .produce),
			Product(name: "Apples", category: .produce),
		]
	])
	
	func contains(productNamed desiredProductName: String, inIsle isle: Int) -> Bool {
		return products.contains(where: { product in 
			let isFound = desiredProductName == product.name
			
			if isFound { print("Found milk! :)") }
			else { print("\(product.name) isn't \(desiredProductName) :(") }
			
			return isFound
		}, inBucket: isle)
	}
}
```

Now `ItemCollection<T>` can be reused, such as in a company:

``` Swift
struct Employee {
	let name: String
	let department: Int
}

class Company {
	private let employees = ItemCollection<Employee>(buckets: [
		[ // Position 0: legal department
			Employee(name: "Alice", department: 0),
			Employee(name: "Bob", department: 0),
		],
		[ // Position 1: marketing department
			Employee(name: "Charlie", department: 1),
			Employee(name: "David", department: 1),
		],
		[ // Position 2: customer service department
			Employee(name: "Eve", department: 2),
			Employee(name: "Frank", department: 2),
		]
	])
	
	func contains(employeeNamed desiredEmployeeName: String, inDepartment department: Int) -> Bool {
		return employees.contains(where: { employee in 
			let isFound = desiredEmployeeName == employee.name
			
			if isFound { print("Found David :)") }
			else { print("\(employee.name) isn't \(desiredEmployeeName) :(") }
			
			return isFound
		}, inBucket: department)
	}
}


let company = Company()
company.contains(employeeNamed: "David", inDepartment: 1)
```

But it's still not great. This worked well when we had only a small number of possible "categories", which we numbered to be small numbers counting from 0 onwards.

Imagine instead that we wanted to search for people by their phone 10 digit numbers. There are 10 million possible 10 digit phone numbers (not counting long distance numbers, which can get even longer). Using the technique above, we would need an array of 10 million arrays, each containing all the people who can be reached by a particular phone number. If we only had a few people on our system, then the most of these 10 million arrays would be empty, what a total waste!

This data structure is very sparse, taking up a ton of space only to store a bunch of blanks, and not much real payload.

## Decreasing the sparsity

The sparsity of the previous data structure arrises from there being a one-to-one relationship between the category numbers and the bucket indices within their parent array. To decrease the sparsity, we can squish this relationship into a many-to-one relationship: several categories would be merged together into single buckets, decreasing the count of buckets.

We can do this with the module function. For example, if we have 14 categories, and want only 5 buckets, we can assign each category into a bucket determined as `category % 5`. So categories `0`, `5`, `10` will share bucket `1`, `6`, `11` will share bucket `2`, and so on.

As a result, the buckets will be larger, which means there will be more linear searching (remember that we can quickly jump to a bucket, but within a bucket we resort to linear searching), but there will be less empty buckets, which saves on memory (and also improves performance, by virtue making the processors' cache more effective, because it's not drowned out by as many empty buckets). Indeed, balancing between bucket size and count is one of the key parameters to tuning the performance of this data structure.

And just like that, we've stumbled into the basic building blocks of a hash table (such as Swift's `Dictionary`). The "category numbers" are hash values, as derived using the `Hashable` protocol.

# A knockoff dictionary implementation

Using the knowledge above, we can make our own implementation of dictionary. Here's my quick and dirty take on it:

``` Swift

struct MyDictionary<Key: Hashable, Value> {
	private var buckets: [[(key: Key, value: Value)]]
	
	init(bucketCount: Int) {
		self.buckets = Array(repeating: [], count: bucketCount)
	}
	
	private func getBucketNumber(forKey key: Key) -> Int {
		var hasher = Hasher()
		hasher.combine(key)
		let hashValue = hasher.finalize()
		return abs(hashValue) % self.buckets.count
	}
	
	subscript(key: Key) -> Value? {
		get {
			let correctBucket = self.buckets[getBucketNumber(forKey: key)]
			let keyValuePair = correctBucket.first(where: { $0.key == key })
			return keyValuePair?.value
		}
		set { 
			let bucketNumber = getBucketNumber(forKey: key)
			
			if let existingKeyValuePairIndex = self.buckets[bucketNumber].firstIndex(where: { $0.key == key }) {
				// There's already an existing value for this key
				
				if let newValue = newValue {
					// so we'll overwrite it
					self.buckets[bucketNumber][existingKeyValuePairIndex] = (key: key, value: newValue)
				} else {
					//if nil, we want to delete the key/value pair entirely
					self.buckets[bucketNumber].remove(at: existingKeyValuePairIndex)
				}
			}
			else if let newValue = newValue {
				// There's no value for this key yet, so we'll add it in
				self.buckets[bucketNumber].append((key: key, value: newValue))
			} else {
				// We want to delete the value for that key, but there isn't one, so we don't have to do anything.
			}
		}
	}
}

var dict = MyDictionary<Character, Int>(bucketCount: 10)
dict["a"] = 1
dict["b"] = 2
dict["c"] = 3
print(dict["a"] as Any) // => Optional(1)
print(dict["d"] as Any) // => nil
```

It's missing quite a few things:

1. Initialize to a default number of buckets, and regrow/rearrange as the buckets get too loaded (higher loaded buckets mean more linear searching, and less benefit to hash-based searching)
2. Conformance to `Collection`, `Equatable`, `Hashable`, etc. which allows our data structure to automatically inherit a bunch of built in algorithms from the standard library like `sort`, `map`, `filter`, `reduce`, etc.
3. A conformance to `ExpressibleByDictionaryLiteral`, which allows us to initialize our dictionary using Swift's dictionary literal syntax, like `["a": 1, "b": 2, "c": 3]`.

## `Equatable` and `Hashable`

The sample implementation above demonstrates the key roles `Equatable` and `Hashable` play in a dictionary key lookup.

* The Key's implementation of `Hashable` is used to determine the hash value for the key (this might be implemented to be the isle number of the product, the department number of the employee, etc.), which is used to quickly determine the appropriate bucket to start linear searching in.
* The Key's implementation of `Equatable` is used to determine when a match is or isn't found during the linear searching phase.

Many of the errors and questions around misbehaving sets and dictionaries is caused by a mismatch in the implementations of `Equatable` and `Hashable`, which I'll explore next.

## A mismatch between `Equatable` and `Hashable`

Consider a `Point` struct, with an `x: Int` and `y: Int` field. There are several possible ways the `x` and `y` values can be used to implement `Equatable.==` and `Hashable.hash(into:)`. Here's a table of them, assigning each case a number, which will be elaborated on below

| Case | in `==` | in both  | in `hash(into:)` | Valid? |
|:----:|:-------:|:--------:|:----------------:|:------:|
|   1  |         | `x`, `y` |                  |   ✅   |
|   2  |   `x`   |    `y`   |                  |   ⚠️   |
|   3  |         |    `x`   |       `y`        |   ❌   |
|   4  |   `x`   |          |       `y`        |   ❌   |
|   5  |         |    `x`   |      `foo`       |  ❌❌❌  |

### Case 1 - Correct ✅

All relevant properties are considered in both the implementation of `==` and `hash(into:)`.

This is correct. The hash value is representative of the values being searched for, so the correct bucket will be selected. During the linear searching phase, a matching key is correctly identified, and its associated value is returned.

<details><summary>Example</summary>
<p>

```swift
struct PointV1: Hashable {
	let (x, y): (Int, Int)
	
	static func == (lhs: Self, rhs: Self) -> Bool {
		return lhs.x == rhs.x && lhs.y == rhs.y
	}
		
	func hash(into hasher: inout Hasher) {
		hasher.combine(self.x)
		hasher.combine(self.y)
	}
}

func hash<T: Hashable>(_ hashable: T) -> Int {
	var hasher = Hasher()
	hashable.hash(into: &hasher)
	return hasher.finalize()
}

let p1 = PointV1(x:  1, y: 2)
let p2 = PointV1(x: 99, y: 2)

// The hash values will vary, but will likely be different ✅
hash(p1) // => 719781228487762341
hash(p2) // => -6436957106723858766
p1 == p2 // => false, which is correct ✅
```
</p>
</details>

### Case 2 - Under-hashing ⚠️

All relevant properties are considered in the implementation of `==`, but only a subset of those are also considered in `hash(into:)`.

This is correct, but might be suboptimal. Each property considered in `==` but not in `hash(into:)` can be thought of as a "missed opportunity" to add more "uniqueness" to the hasher. As a result, similar-but-different values might hash the same (where they otherwise might not have, if only `x` was mixed into the hasher), putting them in the same bucket, which necessitates more linear searching than otherwise unnecessary.

Consider the example of `Point(x: 1, y: 2)` and `Point(x: 99, y: 2)`. The hash value of both points would be the same (because it's derived solely for the `y` values, which are the same in this case, both being `2`), but the points are non-equal (`==` returns false, because the `x` values differ). These two points would have colliding hashed, be placed into the same bucket, and require more `==` calls to search for them. If the `x` values were mixed into the hasher, then there's a high likelyhood that the hash values wouldn't collide, and that the points would be placed into different buckets, requiring fewer `==` calls.

There's an unusual case when this might actually be better performing and desirable: if the `x` is really expensive to hash, but really cheap to `==`. In that case, you save yourself the time of hashing `x`, causing more time linear searching, but using the fast `==` implementation. I can't really think of a case when this would happen, but it is *theoretically* possible. Nonetheless, you probably shouldn't do this.

<details><summary>Example</summary>
<p>

```swift
struct PointV2: Hashable {
	let (x, y): (Int, Int)
	
	static func == (lhs: Self, rhs: Self) -> Bool {
		return lhs.x == rhs.x && lhs.y == rhs.y
	}
		
	func hash(into hasher: inout Hasher) {
		// ⚠️ Missed opportunity: `x` isn't mixed into the hasher.
		// hasher.combine(self.x) 
		hasher.combine(self.y)
	}
}

func hash<T: Hashable>(_ hashable: T) -> Int {
	var hasher = Hasher()
	hashable.hash(into: &hasher)
	return hasher.finalize()
}

let p1 = PointV2(x:  1, y: 2)
let p2 = PointV2(x: 99, y: 2)

// The hash values will vary, but will always collide ⚠️
hash(p1) // => 1300649407116756548
hash(p2) // => 1300649407116756548
p1 == p2 // => false, which is correct ✅
```
</p>
</details>


### Case 3 - False matching ❌

All relevant properties are in the implementation of `hash(into:)`, but only a subset of those are also considered in `==`.

This case is broken, and you should never use it. The hash value is representative of he values being searched for, so the correct bucket will be selected. However, during the linear searching phase, a non-matching key might be accidentally selected instead of the true match.

This case is a bit odd to debug, because sometimes it can behave correctly (it's not deterministic). Imagine a bucket contains `p1` and `p2`, and the dictionary is searching for `p1`:

1. If the bucket happens to be ordered `[p1, p2]`, the first check will be `p1 == p1`, which will be true, causing the dictionary to find `p1` correctly,
2. ...but if the ordering of the bucket is `[p2, p1]`, then the first check will be `p1 == p2`, which will erroneously be true, causing the dictionary to find `p2` in a search for `p1`!


<details><summary>Example</summary>
<p>

```swift
struct PointV3: Hashable {
	let (x, y): (Int, Int)
	
	static func == (lhs: Self, rhs: Self) -> Bool {
		// ❌ False matches: points considered equal even if `y` differs
		return lhs.x == rhs.x // && lhs.y == rhs.y
	}
		
	func hash(into hasher: inout Hasher) {
		hasher.combine(self.x) 
		hasher.combine(self.y)
	}
}

func hash<T: Hashable>(_ hashable: T) -> Int {
	var hasher = Hasher()
	hashable.hash(into: &hasher)
	return hasher.finalize()
}

let p1 = PointV3(x: 1, y: 2)
let p2 = PointV3(x: 1, y: 99)

// The hash values will vary, but will likely be different ✅
hash(p1) // => -2102209745080052058
hash(p2) // => -455190567412857877
p1 == p2 // => true, which is incorrect ❌
```
</p>
</details>

### Case 4 - Under hashing with false matching ❌

Only one property is in the implementation of `==`, and only one is in the implementation of `hash(into:)`

This is just a combination of cases 2 and 3, with the downsides of both.

* For equal values (both `x`s and `y`s are the same) you'll select the correct bucket.
* For unequal values (`x`s differ, but `y`s are the same) you'll select the correct bucket.
* For unequal values (`x`s are the same, but `y`s differ), you'll probably select the wrong bucket (unless there's the hashes happen to map to the same bucket).
 
In all 3 scenarios, you might select the wrong value (because of the broken `==` causing a false match).

<details><summary>Example</summary>
<p>

```swift
struct PointV4: Hashable {
	let (x, y): (Int, Int)
	
	static func == (lhs: Self, rhs: Self) -> Bool {
		// ❌ False matches: points considered equal even if `y` differs
		return lhs.x == rhs.x // && lhs.y == rhs.y
	}
		
	func hash(into hasher: inout Hasher) {
		// ⚠️ Missed opportunity: `x` isn't mixed into the hasher.
		// hasher.combine(self.x) 
		hasher.combine(self.y)
	}
}

func hash<T: Hashable>(_ hashable: T) -> Int {
	var hasher = Hasher()
	hashable.hash(into: &hasher)
	return hasher.finalize()
}

let p1 = PointV4(x: 1, y:  2)
let p2 = PointV4(x: 1, y: 99)

// The hash values will vary, but will likely be different ✅
hash(p1) // => 5985996726589812862
hash(p2) // => -5014003295822567750
p1 == p2 // => true, which is incorrect ❌
```
</p>
</details>


### Case 5 - Super duper broken ❌❌❌

The hash value depends on a value that is totally unrepresentative of the value being searched for. The `Point` example doesn't fit as nicely (because there's no reason why a `Point` would contain a totally unrelated field `foo`), so I'll present a different example that I've seen in the wild:

#### `HTTPHeaderField` example
```
struct HTTPHeaderField: Hashable {
	let value: String

	static func == (lhs: Self, rhs: Self) -> Bool {
		// I changed this line to simplify the explanation, the original was
		// `lhs.value.compare(rhs.value, options: .caseInsensitive) == .orderedSame`
		lhs.value.lowercased() == rhs.value.lowercased()
	}

	func hash(into hasher: inout Hasher) {
		value.hash(into: &hasher)
	}
}
```

The author is trying to model HTTP header fields, which are case insensitive. To be compliant to the HTTP spec, upper/lower/mixed case header fields needed to be treated as identically. This implies the following behaviours:

1. A `Set<HTTPHeaderField>` should only allow one instance of `HTTPHeaderField(value: "a b c")`, regardless of its capitalization
2. Similarly, a `Dictionary<HTTPHeaderField, V>` should only allow one key-value mapping with key `HTTPHeaderField(value: "a b c")`, regardless of its capitalization,
3. `HTTPHeaderField(value: "a b c") == HTTPHeaderField(value: "a b c")` should be true, but so should 
	* `HTTPHeaderField(value: "a b c") == HTTPHeaderField(value: "A b c")` and
	* `HTTPHeaderField(value: "A B C") == HTTPHeaderField(value: "A b c")`

#### The issue

This is intentional, and desirable. However, there's a critical mistake: the hash value is derived from `value`, leaving the letter casing intact. Consider that `HTTPHeaderField(value: "a b c")` and `HTTPHeaderField(value: "A b c")` will hash differently (again, the usual caveats apply: it's always possible that these hash values collide, or that they ultimately happen to map to the same buckets). A search for `HTTPHeaderField(value: "a b c")` will likely make you end up in a different bucket than `HTTPHeaderField(value: "A b c")`. This has a bunch of symptoms:

* If you're inserting into a dictionary, the differing hash values can allow two different key value pairs, one for ``HTTPHeaderField(value: "a b c"): foo` and another for `HTTPHeaderField(value: "A b c"): bar`. This breaks the requirement that differently-cased http headers are treated identically.
* If your dictionary happened to contain `HTTPHeaderField(value: "A B C")`, and its bucket maps the same way as the bucket for `HTTPHeaderField(value: "a b c")`, a look up for the value of `HTTPHeaderField(value: "a b c")` might instead find the value for `HTTPHeaderField(value: "A B C")`.

Conceptually, you can think of `value.lowercased()` as a totally different value from `value` (even though it's derived from it, and can happen to be equal, it's usually not), so lets call it `differentValue`. The hash value derives from `value`, but equality checking depends on `differentValue`. This can work correctly when they're equal, but is totally broken when they're not.

### One possible solution

In cases when `==` derives its value from derived values like `differentValue`, you need a `hash(into:)` that also derives its value from `differentValue`. In this case, you can achieve this by hashing `differentValue.hash(into: &hasher)` (a.k.a. `value.lowercased().hash(into: &hasher)`), instead of `value`. Lowercasing is like a form of [canonicalization](https://en.wikipedia.org/wiki/Canonicalization), a process by which all the set of all possible variously cased-values can be reduced 
into a smaller set of lowercase-only values. There's some caveats around localizations, and which lower-casing algorithm is applied (they can be different between different languages), so it's really not so trivial!

In fact, our beloved built-in `Swift.String` itself employs a canonicalization process! You see, semantically equivalent Unicode strings can be represented differently under the hood. For example `"é"` can be represented in two different ways:

1. As a single code point: `U+00E9`, ["Latin Small Letter E with Acute"](https://www.compart.com/en/unicode/U+00E9), written `"\u{00E9}"` in Swift

2. Or as a combination of two code points:

    | Glyph | Unicode code point | Name                                                                  | Swift string literal   |
    |:-----:|:------------------:|:---------------------------------------------------------------------:|:----------------------:|
    |  `e`  |      `U+0065`      | ["Latin Small Letter E"](https://www.compart.com/en/unicode/U+0065)   |  `"e"` or `"\u{00E9}"` |
    |  `◌́`  |      `U+0065`      | ["Combining Acute Accent"](https://www.compart.com/en/unicode/U+0301) |           `"\u{0301}"` |

Swift wants to treat these two representations equivalently. It would be really strange if two identical looking strings with the same semantics compared as unequal according to `==`. As a result, string equality can't be as simple as `strcmp` from C, doing a blind byte-for-byte comparison of the two strings' underlying buffers. Instead, Swift applies one of the [unicode normalization algorithms](https://en.wikipedia.org/wiki/Unicode_equivalence), so that the Strings are massaged into one of the Unicode canonical forms. Once that's done, semantically equivalent strings will have identical byte representations, which can be used to reliably hash and equate strings.
