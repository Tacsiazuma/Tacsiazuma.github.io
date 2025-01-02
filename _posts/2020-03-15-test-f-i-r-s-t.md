---
layout: post
title:  "Test F.I.R.S.T."
author: kpapp
image: assets/images/time-lapse.jpg
featured: false
hidden: false
categories:
    - Intermediate
    - Testing
tags:
    - first
    - guide
    - test
---

Almost anyone can write automated tests. However, writing good tests is not nearly as common. So what makes a test good? Obviously, we could look at test coverage or perhaps how mutation testing can produce impressive metrics, but these by no means guarantee that the test is actually good. Many have tried to define this in some way before, but no one has yet succeeded in an objective, measurable way. I'm not here to present the Holy Grail; rather, I provide a kind of guideline that can help improve our tests.

This is encapsulated in the so-called F.I.R.S.T. acronym, which stands for fast, isolated, repeatable, self-validating, and timely. Translated, these terms mean that tests should be quick, isolated, repeatable, self-validating, and made in time. It's important to note that these principles primarily apply to unit tests, though some may apply at higher levels as well. Let's go through them one by one!

**F - Fast**

It is crucial for our tests to be fast. We discussed mutation testing in a previous blog and saw that those took significantly longer in our case compared to when we simply ran them with PHPUnit, even though we didn't have many tests. It took four and a half seconds for thirty tests, so imagine if we had three thousand tests. It would take approximately seven and a half minutes. Now imagine our typical PHPUnit tests take this long too. Obviously, we won't always run all our tests, but the longer the tests take to run, the fewer times we'll be able to run them in an eight-hour day, and the less inclined we'll feel to run them. If we are doing TDD, we might set up a file watcher that reruns the tests upon changes or set up a shortcut key that gives us feedback in a fraction of a second, as the tests currently run in milliseconds. Let's keep it that way. Of course, as the number of tests grows, this too will increase and could reach 1-2 seconds, or even more in larger projects.

With integration tests, this can be slower, for example, when calling external services and so on, but we don't run those as often. The key is to strive for speed. Are we refactoring? Extracting something, transferring it elsewhere, rerunning the tests; if they're good, we proceed. If this takes seconds, we won't reactivate them. If they become slow, we won't run them because we don't want to wait, or only rarely after we've rewritten the entire code. If something breaks by then, our IDE might not even remember all the steps, and we could be reverting to the last commit, which... when was it?

**I - Isolated**

The next important element is isolation in our tests. This means two things. First, our tests should examine isolated problems. When we look at them, we shouldn't have to think for minutes about what's gone wrong, as the problem should become clear from the test name or the type/text of assertions. If a test is unclear about what the problem is when it breaks and covers multiple separate things, we're better off splitting it into several smaller, more specific tests.

Let's say we're writing a Roman numeral converter that converts Arabic numbers to Roman numerals. It includes a simple validation to only proceed if the parameter passed is a number greater than zero (life is easier with type definitions):

```php
private function validate($arabic) {
    if (!is_numeric($arabic) || $arabic < 1) {
        throw new \InvalidArgumentException();
    }
}
```

We create separate methods for each test case, which include what is expected in each case in the method name.

```php
/**
* @test
* @expectedException InvalidArgumentException
*/
public function it_should_throw_exception_on_negatives() {
    $this->underTest->convert(-1);
}

/**
* @test
* @expectedException InvalidArgumentException
*/
public function it_should_throw_exception_on_zero() {
    $this->underTest->convert(0);
}
/**
* @test
* @expectedException InvalidArgumentException
*/
public function it_should_throw_exception_on_strings() {
    $this->underTest->convert("test");
}
```

What if we bundle them into one test to avoid code duplication, arguing that less duplication makes the code easier to extend?

```php
/**
 * @test
*/
public function test_validation() {
    $params = [-1, 0, "test"];
    $count = 0;
    foreach ($params as $param) {
        try {
            $this->underTest->convert($param);
        } catch(InvalidArgumentException $ex) {
            $count++;
        }
    }
    $this->assertEquals($count, count($params));
}
```

When it breaks, we see this:

```
1) RomanNumberConverterTest::test_validation
Failed asserting that 2 matches expected 3.
```

Neither the assert nor the method name helps much in identifying the exact problem. We change something in the code, rerun the test, and the same method fails again, but it could be another "case" that broke, assuming we catch this from the lines of code. In unit tests, we isolate the elements to be tested as much as possible.

Another aspect is that individual tests do not depend on each other or the order of test execution. There's no state transfer among them that causes a test to break if it is run later than another for some reason. The order of test execution is not guaranteed unless specified (common in UI tests), so we cannot assume they'll run in the correct order. Here's an example!

Although I mentioned the above concerns mainly apply to unit tests, let's look at an integration test now. Suppose we have an `OrderRepository` class responsible for creating and querying orders in the system. Under tests, this runs behind an SQLite database. To kill two birds with one stone, we check if we can create objects in one test and, in another test, query the object created earlier. If the tests run in the correct order, all's well because the database record will be there. But what if the query runs first and creation later? Then it breaks, even though maybe our colleague just alphabetized the method names in the test. Instead, let's ensure each test starts in an isolated environment, in this case beginning with an empty table (which we clear or might as well drop), and solve it in such a way that during each test the necessary tables and data are present. If a certain state is required for all, that could fit into the `before` steps.

**R - Repeatable**

This is quite intuitive. The test should yield the same result anytime, anywhere, no matter how many times it's run. So where can this arise? One scenario we've already mentioned: if our tests depend on each other and the order inadvertently changes, it can cause a problem since running them in different sequences yields different outcomes. But apart from this, we could face issues. The key is the test must not expect any environmental variables and must leave no trace after completion. A great example for the former is time. Creating a test that assumes the year is 2020, but breaks from 2021 onwards, is quite simple. We've already seen examples for the latter, but it also includes external systems. Our tests must not rely on whether the network is currently operational. Unit tests DO NOT test external systems; these are replaced with stubs/mocks as previously covered in other articles.

**S - Self-validating**

The S, or self-validating. Although the name is quite self-explanatory, let's delve in a bit. This means no post-run checks are required to ensure the tests ran error-free. They either succeeded or failed; there's no intermediate state. So we don’t just print the result to the console for later inspection but assert against a value. If it matches, the test passed; otherwise, it failed. We can’t just look at the result and say, “oh, that’s a false positive, it only doesn’t match because the system time switched seconds,” because then there will be 2, 3, or 30 such occurrences in the code, and when we run our tests, we’d have to individually check which ones signify actual problems.

**T - Timely**

Opinions on this vary, but ideally, the test is written even before the code that makes it pass. This appears in both TDD and test-first approaches. Sure, one could argue it doesn't matter when it’s crafted, as long as it gets done, but when we write the test in advance, a lot changes. If it was written afterward, the developer’s initial task would often be to refactor until the code meets the above principles. Consequently, they typically write fewer and larger tests, which ultimately go against the above principles.

To many, all this seems like common sense, but there's a reason they exist, as testing remains a critical component of development where many still grop around.
