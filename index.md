# Building Good Tests

This is written from the perspective of `pytest`. So some needs may change based on the language and testing framework used. But these principles should extend to all automated testing in some capacity.

### A fixture should only do or provide a single thing

They should also be named after the thing the do or provide. They can be made to do both at the same time, but only if the name you would give to the thing it provides also adequately describes the thing it does.

A good measure to go buy when trying to figure out if you should break something out into a standalone fixture, is very similar to doing the same for functions. If a chunk of the code in your fixture can be described with a simple phrase, break it out into its own fixture. Fixtures can request other fixtures just like tests can, so we can (and should) abstract these operations and resources just like we can (and should) with normal code.

#### Why?

* More modular (they can be used in several places).
* Easily parameterizable.
* Individual actions/resources can be easily overridden based on context and scope for select tests without impacting others.
* Easily documented, and easily named.
* Easier to understand what a given test requires, and how it's set up/torn down.
* Easier to manage setups and teardown. 
* Easier to debug.
* Keeps fixtures shorter and more concise.

### Do not test the setups. Test the end result

Your test functions are for the end result, not the steps to get there, so don't bother asserting those went well in the test. 

#### Why?

Steps in the setup process can be broken out into standalone fixtures with their own assert statements to make sure they're alright. If those asserts fail, they will show up as an error, rather than a failure in the test. When a test has an error, it should represent that the desired state for that test could not be reached. If a test fails, it should represent that the state was reached, but something about the state is wrong. Additionally, if a fixture has a problem, then every test that exists in the scope of that fixture will automatically show the problem, and you''ll have a cascading failure that happens quickly (performance boost!).

This lets us know at a quick glance which tests are actually failing according to what they should be testing, and which ones are just being impacted by problems in some earlier step of the workflow. For every step in the setup process, there should be an associated test scenario that tests this state. 

Those fixtures should also have their own tests anyway, and it's not the responsibility of this test to make sure those went well.

### For every fixture that represents a change in the state, there should be a test for that state

For every fixture that changes something about the state of the SUT, there should be an alternate version of this fixture that doesn't have an assert in it, and that fixture should be used to make the actual test for that state.

#### Why?

In combination with the previous point, this creates a system where you can quickly and easily identify where the breakdowns are in the flows of the STU, and how widely impacting certain problems are. Failures will tell you exactly where a problem is, while errors tell you how much impact that problem has.

### 1 `assert` per test function/method and nothing else

The only thing that should be in a test function is a single `assert` statement. It should be simple, concise, and easily readable.

If you're working with multiple `assert` statements because your test subject is very complex and there are many things about it that need to be validated, you can use a class, module, or even an entire package as the scope for your test scenario. Your fixtures can be made to work around this scope, just like they can for the function-level scope. This gives you room to breath so you can give each package/module/class/function a meaningful name and you can maintain the rule of only 1 `assert` statement per test function/method.

You can also rely on magic comparison methods (e.g. `__eq__`) along with special state comparison objects to create more intuitive `assert` statements that can check more than one thing at once while providing more readable failure messages. 

#### Why?

* If you have a test function with more than one `assert` statement, and an earlier one fails, you won't know if the later ones would pass or fail because test execution stops on the first failure, and this prevents you from getting a complete understanding about what is wrong.
* It's less cluttered.
* A single `assert` should be the test, because that is thing you're *asserting*, so more than one `assert` means you're asserting more than one thing, and thus you are testing something different. 
* If they're bundled into one test function, they can't be targeted and ran individually, which makes debugging specific asserts more difficult.
* Doesn't prevent parallelization.
* Doesn't prevent parameterization.
* It's easily avoided in `python` and `pytest` by having multiple test methods in a test class/module/package, or by utilizing magic comparison methods like `__eq__`.

<https://testing.googleblog.com/2018/06/testing-on-toilet-keep-tests-focused.html>

<https://docs.microsoft.com/en-us/dotnet/core/testing/unit-testing-best-practices#avoid-multiple-asserts>

##### Bad:

Code:

```python
@pytest.fixture
def apple():
    return Apple()

def test_apple(apple):
    assert apple.is_sweet()
    assert apple.color == "Red"
    assert len(apple.worms) == 0
```

Output:

```
_______________________________ test_apple ________________________________

apple = <test_fruit.Apple object at 0x7f36857b5cf8>

    def test_apple(apple):
>       assert apple.is_sweet()
E       assert False

test_fruit.py:17: AssertionError
```

Was the apple red? Did it have any worms? 

##### Good:

Code:

```python
@pytest.fixture(scope="class")
def apple():
    return Apple()

class TestApple():
    def test_is_sweet(self, apple):
        assert apple.is_sweet()
    def test_color_is_red(self, apple):
        assert apple.color == "Red"
    def test_has_no_worms(self, apple):
        assert len(apple.worms) == 0
```

Output:

```
_________________________ TestApple.test_is_sweet _________________________

self = <test_fruit.TestApple object at 0x7f6a9b48db70>
apple = <test_fruit.Apple object at 0x7f6a9b471b70>

    def test_is_sweet(self, apple):
>       assert apple.is_sweet()
E       assert False

test_fruit.py:20: AssertionError
_______________________ TestApple.test_color_is_red _______________________

self = <test_fruit.TestApple object at 0x7f6a9b48da20>
apple = <test_fruit.Apple object at 0x7f6a9b471b70>

    def test_color_is_red(self, apple):
>       assert apple.color == "Red"
E       AssertionError: assert 'Green' == 'Red'
E         - Green
E         + Red

test_fruit.py:22: AssertionError
_______________________ TestApple.test_has_no_worms _______________________

self = <test_fruit.TestApple object at 0x7f6a9b487400>
apple = <test_fruit.Apple object at 0x7f6a9b471b70>

    def test_has_no_worms(self, apple):
>       assert len(apple.worms) == 0
E       assert 1 == 0

test_fruit.py:24: AssertionError
```

### Use standard `assert` statements, instead of the `unittest.TestCase` assert methods

Don't use the assert methods provided by the `unittest.TestCase` class. Instead, use the standard `assert` keyword.

#### Why?

* It's more idiomatic.
* `pytest` will show context-sensitive comparisons automatically for failed tests.
* `pytest` will do assertion introspection to give more information surrounding a failed test.
* Avoids dependency on `unittest.TestCase`, which is essential for the next point.

### Don't inherit from `unittest.TestCase` in test classes (either directly, or indirectly)

#### Why?

This causes conflicts within `pytest` with regards to how it executes tests and fixtures. As a result, it heavily limits the functionality of `pytest` that is available to you (e.g. parameterization is not possible). 

### A test should only involve the resources it needs

A test should not involve any superfluous resources. If you don't need to involve some resource in order to test some behavior, then there's no reason to have the test require it.

#### Why?

* It's inherently faster.
* It's easier to manage.
* It avoids unrelated problems in as many parts of the stack as possible.
* It can be run by more people, because they will need less of a local sandbox established to get it working.

When a test utilizes less resources, it will inherently run faster. The faster a test is, the more often it can be run, and the less time a developer is waiting for it.

<https://www.youtube.com/watch?reload=9&v=AJ7u_Z-TS-A>

### Test behavior, not implementation

Design your tests around the behavior you want to test, not the implementation you ended up with. If you are changing behavior, you should be thinking about how you can test that behavior. You should try to find ways to test it that aren't dependent on the implementation. 

#### Why?

* Reduces the need to change/update tests whenever the implementation changes.
* Makes sure we're actually testing behavior, and not just that our code works in general.
* Reduces the number of tests, due to focusing on the tests that matter.
* Encourages sufficient levels of abstraction.

<https://testing.googleblog.com/2013/08/testing-on-toilet-test-behavior-not.html>

### Only verify state-changing method calls

Note: this does not mean don't ever use non-state-changing method calls in your tests.

#### Why?

* Non-state-changing method calls will often be tested through other means, such as using them to check that state-changing method calls did what they should.
* It adds unnecessary fluff to the test suite.

<https://testing.googleblog.com/2017/12/testing-on-toilet-only-verify-state.html>

### Utilize fixture scope levels to optimize your tests

You can utilize classes, modules, and even entire packages to hold the tests for a single state. All it requires is creating fixtures for those scopes.

##### Example:

```python
@pytest.fixture(scope="class")
def apple():
    return Apple()

class TestApple():
    def test_is_sweet(self, apple):
        assert apple.is_sweet()
    def test_color_is_red(self, apple):
        assert apple.color == "Red"
    def test_has_no_worms(self, apple):
        assert len(apple.worms) == 0
```

In this example, only a single `Apple` instance is created. But the state is represented by the entire class, so you have the opportunity to test individual attributes in their own test methods.

### Test the result, not the process

This is more of an organizational point, but what the test subject is can often be confusing. To simplify this, you should always consider your test subject to be the end result of the process, rather than the process itself.

For example, let's say you want to test that logging in to your website works, and you're going to do this with an end-to-end test. Is this a test for the log in page? the log in process? or the page you land on after logging in? Because your tests are running while you're on the landing page, and the landing page is the only thing you can use to test that logging in worked, that means the landing page is your test subject.

#### Why?

* Establishes a consistent means of determining what state the test subject should be in while the tests for that scenario are running.
* The test subject reflects the desired behavior after a certain process is performed (e.g. I should be on the landing page after logging in).
* You can group tests more effectively (see next point). 

### Group tests based on what they're testing

Tests should be grouped together based on the method/unit/component they are testing, and the scenario under which they're being tested. For example, all the test scenarios for the home page should be grouped together in a common space, and all the tests for the home page after logging out should be put in the same scope together so they can be run at the same time.

#### Why?

*  Tests are easily targeted according to the test subjects they apply to.
* Less redundant code due to having common fixtures only having to be defined once, and only as high up as they need to be.
* Common fixtures that need to be overridden due to the tests being run only have to be overridden once, and can be defined at a much lower level so they don't conflict with things they shouldn't.
* Naming tests becomes much easier (see next point).

### The names of the test package, module, class, and method/function should combine to completely describe the unit/component being tested, what scenario it's being tested under, and what behavior specifically is being tested, without any redundancy

The name of the outermost surrounding scopes should combine to describe what unit/component is being tested, while the innermost surrounding scope describes the scenario, and the test function/method itself describes the behavior being tested. The only exception is if you're using a state comparison object to check multiple things at once in a single test function/method, each thing being checked is self-explanatory, and if the test fails, the failure output will very clearly describe each issue that was found.

##### Bad:

```
 tests.py::test_product_is_good FAILED
```

##### Bad:

```
 tests/test_website/test_website_landing_page.py::TestWebsiteLandingPageAfterLogIn::t
 est_website_landing_page_after_login_header_is_displayed FAILED
```

##### Good:

```
 tests/website/test_landing_page.py::TestAfterLogIn::test_header_is_displayed PASSED
 tests/website/test_landing_page.py::TestAfterLogIn::test_header_text FAILED
 tests/website/test_landing_page.py::TestAfterLogIn::test_header_tag_name FAILED
```

Failure Messages:

```
=================================== FAILURES ======================================
_________________________ TestAfterLogIn.test_header_text _________________________

self = <tests.website.test_landing_page.TestAfterLogIn object at 0x7f8d20e0fd30>
page = <tests.website.test_landing_page.LandingPage object at 0x7f8d20e0f860>

    def test_header_text(self, page):
>       assert page.header.text == "My Header"
E       AssertionError: assert 'Something else' == 'My Header'
E         - Something else
E         + My Header

website/test_landing_page.py:25: AssertionError
_______________________ TestAfterLogIn.test_header_tag_name _______________________

self = <tests.website.test_landing_page.TestAfterLogIn object at 0x7f8d20e0f550>
page = <tests.website.test_landing_page.LandingPage object at 0x7f8d20e0f860>

    def test_header_tag_name(self, page):
>       assert page.header.tag_name == "h1"
E       AssertionError: assert 'div' == 'h1'
E         - div
E         + h1

website/test_landing_page.py:28: AssertionError
```

##### Good:

```
 tests/website/test_landing_page.py::TestAfterLogIn::test_header FAILED
```

Failure Message:

```
==================================== FAILURES =====================================
___________________________ TestAfterLogIn.test_header ____________________________

self = <tests.website.test_landing_page.TestAfterLogIn object at 0x7faf95aee240>
page = <tests.website.test_landing_page.LandingPage object at 0x7faf95aaad30>

    def test_header(self, page):
>       assert page.header == State(
            IsDisplayed(),
            Text("My Header"),
            TagName("h1"),
        )
E       AssertionError: assert Comparing Header State:
E             Text: 'Something else' is not 'My Header'
E             TagName: 'div' is not 'h1'

website/test_landing_page.py:22: AssertionError
```

<https://testing.googleblog.com/2014/10/testing-on-toilet-writing-descriptive.html>

<https://docs.microsoft.com/en-us/dotnet/core/testing/unit-testing-best-practices#naming-your-tests>

### The order tests are run in shouldn't matter (including parameterized tests)

The order pytest runs tests in is not deterministic, and can change at any time, especially when the tests may be run in parallel. 

#### Why?

* It helps make sure tests aren't dependent on other tests.
* It helps make sure tests don't conflict with other tests.
* It makes parallelization easier.

### Every test should be able to be run directly (including parameterized tests)

Every test is given a `nodeid` by pytest, and that nodeid can be passed as an argument to pytest on the command line so that it is run directly.  

#### Why?

* You can save time by targeting specific tests so only they run.
* It helps make sure tests aren't dependent on other tests.

### Parameterized tests should have unique, readable names

When parameterizing a test, or scope, pytest will automatically use the `repr` for each object of the set to give a unique element to the test `nodeid`s so you can see which specific set being used for a given test, and also so you can directly target one or more tests directly for them to run individually.  You can override this behavior using the `ids` argument when parameterizing. This accepts either a iterable of strings that should map 1:1 for each parameter set, or a callable so you can rely on some logic to generate the name for a given parameter set. Whichever approach is used, it should be something that is unique and adequately describes what that set is being used to test, i.e. it should provide insight as to what exactly it's being used to test, and why the other parameter sets wouldn't cover the same thing.

#### Why?

* It tells you what behavior is being tested.
* It's more readable.
* It makes targeting tests easier.

<https://testing.googleblog.com/2014/10/testing-on-toilet-writing-descriptive.html>

### Every test should be able to be run in parallel with any other test

Some tests should not be allowed to be split up across multiple threads, simply because it would be inefficient. For example, you may have a test class that is used to house several tests for a single test scenario, i.e. they are all meant to be run against a single, common state. Those tests should not be split apart, because it's more efficient to keep them together and only have to run those fixtures once total, instead of once for each thread they're split across. You can do this quite easily with the `pytest-xdist` plugin by either providing a standard scope to the `--dist` argument, or by defining your own `FixtureScheduling` class for the plugin to use. An example of a custom `FixtureScheduling` class can be found [here](https://github.com/pytest-dev/pytest-xdist/issues/18#issuecomment-392558907).

There are plans to improve this capability and make it more refined, but that's [still being discussed here](https://github.com/pytest-dev/pytest-xdist/issues/19).

Even so, the tests should be designed to assume that they may be running at the same time as any other test.

#### Why?

* Parallelization significantly decreases test runtime.
* It helps make sure tests aren't dependent on other tests.
* It helps make sure tests don't conflict with other tests.
* It helps make sure that test data is dynamically produced and isolated, rather than relying on static assets.

### A test should be completely independent 

A test should be able to build it's entire environment from scratch, including the test data. This does not include things that need to be installed, however. Once everything is installed, you should be able to just run the `pytest` command, and have everything run.

#### Why?

* Running the tests should be trivial, and quick to kickoff.

### A test should *never* be flakey

A test should never pass or fail on a whim. If the test can be run and have different results without any changes to the code, then that test is useless and can't be trusted.

### Mock less

Try to avoid mocking things whenever possible.

#### Why?

* The more you mock, the less you actually test.
* Mocking complicates the test.
* If you're mocking, it can often mean your tests are dependent on implementation.
* Mocking makes tests harder to maintain as you now have to maintain the mocks as well.
* The more you mock, the less trustworthy a test is.

<https://testing.googleblog.com/2013/05/testing-on-toilet-dont-overuse-mocks.html>

### Test coverage is not a metric for what was tested; it's a metric for what code your tests used

Just because there is good code coverage in the tests doesn't mean those are good tests, are that they're even trying to test something. At best, code coverage can be used to determine what areas of your code should be pruned. At worst, it gives developers false confidence in the quality of their code.

<https://testing.googleblog.com/2008/03/tott-understanding-your-coverage-data.html>

### The code should be easy to test

Testing code should not be difficult. If the code is hard to test, then the code is hard to use. This is also indicative of tightly coupled code.

Testing should be a primary concern when writing code. Whenever you change behavior, you should be thinking about how you can test that change in a way that would have failed before the change. if you aren't changing behavior and are only changing the implementation, you should not have to worry about changing your tests.

If you find that you can't easily test your code, you should change the code so it *can* be easily tested. 

#### Why?

* Makes testing code easier.
* Makes *using* code easier.
* Encourages avoiding tightly coupled code

<https://docs.microsoft.com/en-us/dotnet/core/testing/unit-testing-best-practices#less-coupled-code>
