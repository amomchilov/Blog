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

```
TODO
```
