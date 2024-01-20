# Stop writing getters and setters in Swift

I see this time and time again, and it's about time I write an article in one place to consolidate all my thoughts. If you find yourself writing code that looks like this, listen up:

``` Swift
public class C {
	private var _i: Int = 0
	public var i: Int {
		get {
			return self._i
		}
		set {
			self._i = newValue
		}
	}
}
```


This pattern\* *is completely pointless* in Swift, and I'll explain why, but firstly we need to take a short detour through Java land. Why Java? Because most of the people I run into who write Swift like this have some sort of Java background, it's because either:

1. it was taught in their computer science courses, or
2. they're coming over to iOS development, from Android

## What's the point of getters and setters?

Suppose we have the following class in Java:

``` Swift
public class WeatherReport {
	public String cityName;
	public double temperatureF;

	public WeatherReport(String cityName, double temperatureF) {
		this.cityName = cityName;
		this.temperatureF = temperatureF;
	}
}
```

If you showed this class to any CS prof, they're surely going to bark at you for breaking encapsulation. But what does that really mean? Well, imagine how a class like this would be used. Someone would write some code that looks something like this:


``` Swift
WeatherReport weatherReport = weatherAPI.fetchWeatherReport();
weatherDisplayUI.updateTemperatureF(weatherReport.temperatureF);
```

Now suppose you wanted to upgrade your class to store data in a more sensible temperature unit (beating the imperial system dead horse, am I funny yet?) like Celsius or Kelvin. What happens when you update your class to look like this:

``` Java
public class WeatherReport {
	public String cityName;
	public double temperatureC;

	public WeatherReport(String cityName, double temperatureC) {
		this.cityName = cityName;
		this.temperatureC = temperatureC;
	}
}
```

You've changed the implementation details of your `WeatherReport` class, but you've also made an API breaking change. Because `temperatureF` was public, it was part of this class' API. Now that you've removed it, you're going to cause compilation errors in every consumer that depended on the existence of the `temperatureF` instance variable.

Even worse, you've changed the semantics of the second double argument of your constructor, which *won't* cause compilation errors, but behavioural errors at runtime (as people's old Fahrenheit based values are attempted to be used as if they were Celsius values). However, that's not an issue I'll be discussing in this article.

The issue here is that consumers of this class will be strongly coupled to the implementation details of your class. To fix this, you introduce a layer of separation between your implementation details and your interface. Suppose the Fahrenheit version of our class was implemented like so:

``` Java
public class WeatherReport {
	private String cityName;
	private double temperatureF;

	public WeatherReport(String cityName, double temperatureF) {
		this.cityName = cityName;
		this.temperatureF = temperatureF;
	}

	public String getCityName() {
		return this.cityName;
	}

	public void setCityName(String cityName) {
		this.cityName = cityName;
	}

	public double getTemperatureF() {
		return this.temperatureF;
	}

	public void setTemperatureF(double temperatureF) {
		this.temperatureF = temperatureF;
	}
}
```

The getters and setters are really basic methods that access or update our instance variables. Notice how this time, our instance variables are `private`, and only our getters and setters are `public`. A consumer would use this code, as so:


``` Swift
WeatherReport weatherReport = weatherAPI.fetchWeatherReport();
weatherDisplayUI.updateTemperatureF(weatherReport.getTemperatureF());
```
This time, when we make the upgrade to Celsius, we have the freedom to change our instance variables, and tweak our class to keep it backwards compatible:

``` Java
public class WeatherReport {
	private String cityName;
	private double temperatureC;

	public WeatherReport(String cityName, double getTemperatureC) {
		this.cityName = cityName;
		this.temperatureC = temperatureC;
	}

	public String getCityName() {
		return this.cityName;
	}

	public void setCityName(String cityName) {
		this.cityName = cityName;
	}

	// Updated getTemperatureF is no longer a simple getter, but instead a function that derives
	//  its Fahrenheit value from the Celcius value that actually stored in an instance variable.
	public double getTemperatureF() {
		return this.getTemperatureC() * 9.0/5.0 + 32.0;
	}

	// Updated getTemperatureF is no longer a simple setter, but instead a function
	// that updates the celcius value stored in the instance variable by first converting from Fahrenheit
	public void setTemperatureF(double temperatureF) {
		this.setTemperatureC((temperatureF - 32.0) * 5.0/9.0);
	}

	// Mew getter, for the new temperatureC instance variable
	public double getTemperatureC() { 
		return this.temperatureC;
	}

	// New setter, for the new temperatureC instance variable
	public void setTemperatureC(double temperatureC) {
		this.temperatureC = temperatureC;
	}
}
```

We've added new getters and setters so that new consumers can deal with temperatures in Celsius. But importantly, we've re-implemented the methods that used to be getters and setters for temperatureF (which no longer exists), to do the appropriate conversions and forward on to the Celsius getters and setters. Because these methods still exist, and behave identically as before, we've successfully made out implementation change (storing F to storing C), *without* breaking our API. Consumers of this API won't notice a difference.

## So why doesn't this translate into Swift?

It does. But simply put, it's already done for you. You see, stored properties in Swift are not instance variables. In fact, Swift *does not provide a way for you to create or directly access instance variables*.

To understand this, we need to have a fuller understanding of what properties are. There are two types, stored and computed, and neither of them are "instance variables".

* Stored properties: Are a combination of a compiler-synthesized instance variable (which you never get to see, hear, touch, taste, or smell), and the getter and setter that you use to interact with them.
* Computed properties: Are just a getter and setter, without any instance variable to act as backing storage. Really, they just behave as functions with type `() -> T`, and `(T) -> Void`, but have a pleasant dot notation syntax:


	``` Swift
	print(weatherReport.temperatureC)
	weatherReport.temperatureC = 100
	```

	rather than a function calling syntax:
		
	``` Swift
	print(weatherReport.getTemperatureC())
	weatherReport.setTemperatureC(100)
	```

So in fact, when you write:

``` Swift
class C {
	var i: Int
}
```

`i` is the name of the getter and setter for an instance variable the compiler created for you. Let's call the instance variable `$i` (which is not an otherwise legal Swift identifier). There is no way to directly access `$i`. You can only get its value by calling the getter `i`, or update its value by calling its setter `i`.

So let's see how the `WeatherReport` migration problem looks like in Swift. Our initial type would look like this:

``` Swift
public struct WeatherReport {
	public let cityName: String
	public let temperatureF: Double
}
```

Consumers would access the temperature with `weatherReport.temperatureF`. Now, this looks like a direct access of an instance variable, but remember, that's simply not possible in Swift. Instead, this code calls the compiler-synthesized getter `temperatureF`, which is what accesses the instance variable `$temperatureF`.

Now let's do our upgrade to Celsius. We will first update our stored property:

``` Swift
public struct WeatherReport {
	public let cityName: String
	public let temperatureC: Double
}
```

This has broken our API. New consumers can use `temperatureC`, but old consumers who depended on `temperatureF` will no longer work. To support them, we simply add in a new computed property, that does the conversions between Celsius and Fahrenheit:

``` Swift
public struct WeatherReport {
	public let cityName: String

	public let temperatureC: Double
	public var temperatureF: Double {
		get { return temperatureC * 9/5 + 32 }
		set { temperatureC = (newValue - 32) * 5/9 }
	}
}
```

Because our `WeatherReport` type still has a getter called `temperatureF`, consumers will behave just as before. They can't tell whether a property that they access is a getter for a stored property, or a computed property that derives its value in some other way.

So let's look at the original "bad" code. What's so bad about it?

``` Swift
public class C {
	private var _i: Int = 0
	public var i: Int {
		get {
			return self._i
		}
		set {
			self._i = newValue
		}
	}
}
```

When you call `c.i`, the following happens:

1. You access the getter `i`.
2. The getter `i` accesses `self._i`, which is yet another getter
3. The getter `_i` access the "hidden" instance variable `$i`

And it's similar for the setter. You have two layers of "getterness". See what that would look like in Java:

``` Java
public class C {
	private int i;

	public C(int i) {
		this.i = i;
	}

	public int getI1() {
		return this.i;
	}

	public void setI1(int i) {
		this.i = i;
	}

	public int getI2() {
		return this.getI1();
	}

	public void setI2(int i) {
		this.setI1(i);
	}
}
```

It's silly!

# But what if I want a private setter?

Rather than writing this:

``` Swift
public class C {
	private var _i: Int = 0

	public var i: Int {
		get {
			return self._i
		}
	}
}
```

You can use this nifty syntax, to specify a separate access level for the setter:

``` Swift
public class C {
	public private(set) var i: Int = 0
}
```

Now isn't that clean?
