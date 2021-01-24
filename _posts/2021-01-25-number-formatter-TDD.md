---
layout: post
title:  "How to create a custom Number Formatter using TDD"
date:   2021-01-25
---

## Presentation of the problem

While working on a project dealing with currency exchange rates I found myself in need of a custom formatter that would return string values following a particular set of rules.

For example, for the following inputs the rules expected the following outputs:

Input | Output
------------ | -------------
0.000032333 | "0.000032"
14.00458 | "14.00"
1.232 | "1.23"
0.000032 | "0.000032"


In this article, I documented how I approached this challenge by test-driving a custom formatter implementation. 
You can find the git repository [here](https://github.com/chrzarma/Formatter).


## Solution of the problem
### First step: Breaking down the requirements

First, I created a list of the rules and grouped possible scenarios that I wanted the formatter to handle.

So I ended up with these cases:

1. Returns string of number with two fraction digits for number with two fraction digits:

    Input | Output
    ------------ | -------------
    1.23 | “1.23”

2. Returns string of number rounded to two fraction digits:
    
    Input | Output
    ------------ | -------------
    14.00023 | “14.00”
    14.008 | “14.01”
    1.2342 | “1.23”
    1.230000 | “1.23”

3. Returns string of number rounded to two significant digits for values less than 1:
    
    Input | Output
    ------------ | -------------
    0.000032 | “0.000032”
    0.0000323343 | “0.000032”
    0.0000328103 | “0.000033”
    0 | “0.00”


### Second step: The part of coding

After drafting the checklist I felt comfortable enough to start test-driving the formatter implementation.

The next step was to create a test assertion for each of the cases I mentioned above, by following the [TDD cycle](https://blog.cleancoder.com/uncle-bob/2014/12/17/TheCyclesOfTDD.html):

* Red state (we want to see a build fail and then a failing test)
* Green state (we want the test to pass with the minimum changes)
* Refactoring (refactor our code to be simple and easy to read if possible)

After writing the first test naturally there was a build error since there's no production code yet.

![2021-01-25-FirstTestBuildFail](https://github.com/chrzarma/chrzarma.github.io/blob/main/images/2021-01-25-FirstTestBuildFail.png?raw=true)

The next step was to write the minimum code possible to make the project build again and pass the test.

```swift
func format(_ value: Decimal) -> String {
    let nf = NumberFormatter()
    nf.maximumFractionDigits = 2
    
    return nf.string(from: value as NSNumber) ?? ""
}
```

However, the test didn’t pass as I initially expected. The problem was that I didn’t take into concideration the device’s locale which was different from the one used in the tests (the locale’s separator was “,” and the locale used in the tests was “.”). But that was good progress as I got to see the test failing.

![2021-01-25-TestFailsPassing](https://github.com/chrzarma/chrzarma.github.io/blob/main/images/2021-01-25-TestFailsPassing.png)

So taking into consideration the devices locale I ended up with the following code: 

```swift
import Foundation

func format(_ value: Decimal, locale: Locale) -> String {
    let nf = NumberFormatter()
    nf.maximumFractionDigits = 2
    nf.locale = locale
    
    return nf.string(from: value as NSNumber) ?? ""
}
```

```swift
@testable import Formatter
 
class FormatterTests: XCTestCase {
  func test_twoFractionDigits() {
    XCTAssertEqual(format(1.23, locale: Locale(identifier: "en_US")), "1.23")
    XCTAssertEqual(format(4.56, locale: Locale(identifier: "en_US")), "4.56")
    XCTAssertEqual(format(7.89, locale: Locale(identifier: "en_US")), "7.89")
    XCTAssertEqual(format(0.12, locale: Locale(identifier: "en_US")), "0.12")
  }
}
```

The test was finally green!

![2021-01-25-TestPasses](https://github.com/chrzarma/chrzarma.github.io/blob/main/images/2021-01-25-TestPasses.png)

At this point there was nothing to refactor so I continued with the next tests following the same pattern for all the cases I had set.

The production code and tests now looked like this:
```swift
public final class Formatter {
  public static func format(_ value: Decimal, locale: Locale) -> String {
    let nf = NumberFormatter()
    nf.maximumFractionDigits = 2
    nf.minimumFractionDigits = 2
    nf.locale = locale

    if value < 1 {
      nf.maximumSignificantDigits = 2
    }

    return nf.string(from: value as NSNumber) ?? ""
  }
}
```

```swift
class FormatterTests: XCTestCase {
  func test_twoFractionDigits() {
    XCTAssertEqual(format(1.23), "1.23")
    XCTAssertEqual(format(4.56), "4.56")
    XCTAssertEqual(format(7.89), "7.89")
    XCTAssertEqual(format(0.12), "0.12")
  }
     
  func test_roundsValuesToTwoFractionDigits() {
    XCTAssertEqual(format(1.234), "1.23")
    XCTAssertEqual(format(1.235), "1.24")
    XCTAssertEqual(format(1.236), "1.24")
    XCTAssertEqual(format(1.2300), "1.23")
    XCTAssertEqual(format(1.00023), "1.00")
  }
   
  func test_roundsValuesToTwoSignificantDigitsWhenLessThanOne() {
    XCTAssertEqual(format(0.000032), "0.000032")
    XCTAssertEqual(format(0.0000323343), "0.000032")
    XCTAssertEqual(format(0.0000328103), "0.000033")
    XCTAssertEqual(format(0.0000325103), "0.000033")
  }

  // MARK: Helpers
   
  func format(_ value: Decimal) -> String {
    Formatter.format(value, locale: Locale(identifier: "en_US"))
  }
}
```

But I wasn’t done yet. When testing functions that handle numbers I’ve found out that it’s  always a good practice to triangulate cases and check if the rounding of the numbers works as you wish.

So I created another test case checking the rounding mode of the formatter.

```swift
func test_roundsHalfUpValuesToTwoFractionDigits() {
    XCTAssertEqual(format(1.005), "1.01")
    XCTAssertEqual(format(2.006), "2.01")
    XCTAssertEqual(format(3.007), "3.01")
    XCTAssertEqual(format(4.008), "4.01")
    XCTAssertEqual(format(5.009), "5.01")
    XCTAssertEqual(format(1.001), "1.00")
    XCTAssertEqual(format(2.002), "2.00")
    XCTAssertEqual(format(3.003), "3.00")
    XCTAssertEqual(format(4.004), "4.00")
}
```

The test failed this time! The reason for that was that the implementation of `Decimal` input in the helper method of the tests works in the following way:

When passing a number as an input and declaring it as a `Decimal` what I’ve found out that happens is that Foundation handles it as a `Double` and then converts it into a `Decimal`. Because `Decimal`s are more precise than `Double`s you may not always get the expected `Decimal` representation.

 For example, if the helper method’s input is `1.005` you end up testing the function for the value `1.004999999` and get a failing test. 

To solve this issue, I changed the implementation of the helper method’s input to `String` literal to get the precision of the `Decimal` input I wanted to test. So the tests now look like this:

```swift
import XCTest
@testable import Formatter

class FormatterTests: XCTestCase {
    func test_twoFractionDigits() {
        XCTAssertEqual(format("1.23"), "1.23")
        XCTAssertEqual(format("4.56"), "4.56")
        XCTAssertEqual(format("7.89"), "7.89")
        XCTAssertEqual(format("0.12"), "0.12")
    }
    
    func test_roundsValuesToTwoFractionDigits() {
        XCTAssertEqual(format("1.234"), "1.23")
        XCTAssertEqual(format("1.235"), "1.24")
        XCTAssertEqual(format("1.236"), "1.24")
        XCTAssertEqual(format("1.2300"), "1.23")
        XCTAssertEqual(format("1.00023"), "1.00")
    }
    
    func test_roundsValuesToTwoSignificantDigitsWhenLessThanOne() {
        XCTAssertEqual(format("0.000032"), "0.000032")
        XCTAssertEqual(format("0.0000323343"), "0.000032")
        XCTAssertEqual(format("0.0000328103"), "0.000033")
        XCTAssertEqual(format("0.0000325103"), "0.000033")
    }
    
    func test_roundsHalfUpValuesToTwoFractionDigits() {
        XCTAssertEqual(format("1.005"), "1.01")
        XCTAssertEqual(format("2.006"), "2.01")
        XCTAssertEqual(format("3.007"), "3.01")
        XCTAssertEqual(format("4.008"), "4.01")
        XCTAssertEqual(format("5.009"), "5.01")
        XCTAssertEqual(format("1.001"), "1.00")
        XCTAssertEqual(format("2.002"), "2.00")
        XCTAssertEqual(format("3.003"), "3.00")
        XCTAssertEqual(format("4.004"), "4.00")
    }
    
    // MARK: Helpers
    
    func format(_ valueString: String) -> String {
        guard let value = Decimal(string: valueString) else { return "" }
        return Formatter.string(from: value, locale: Locale(identifier: "en_US"))
    }
}
```

The test was now failing, however, by adding the following line in the `Formatter` makes it green:

```swift
nf.roundingMode = .halfUp
```

## Summary

So in conclusion, creating a list with all the rules beforehand and grouping possible cases helped me test-drive the custom `Formatter` implementation and don't get lost. 

Another valuable takeaway is to provide the `locale` when creating custom formatters. By using the default `locale` you might end up with values that lead to wrongful assumptions, confusing user experiences, and flaky tests.


You can find this post’s repository on GitHub in the following [here](https://github.com/chrzarma/Formatter)
