---
name: More Advanced Topics
id: more_advanced_topics
dependsOn: [
  technology_and_tooling.testing.diagnosing_issues
]
tags: [pytest]
attribution: 
    - citation: >
        "Aleksandra Nenadic, Steve Crouch, James Graham, et al. (2022). carpentries-incubator/python-intermediate-development: beta (beta). Zenodo. https://doi.org/10.5281/zenodo.6532057"
      url: https://doi.org/10.5281/zenodo.6532057
      image: https://carpentries-incubator.github.io/python-intermediate-development/assets/img/incubator-logo-blue.svg
      license: CC-BY-4.0
---

## Introduction

Having completed the previous sections of the course, we now know:
- how to write tests
- how to debug problems with our code 
- how to code in a defensive manner in order to prevent invalid inputs from causing problems.

Up to this point, our examples have focused on unit testing of simple functions that formed part of larger program to analyse inflammation data from patients in a drug trial. As we develop this program further, the complexity of the codebase will increase which, in turn, means a larger number of tests will need to written and organised.  We may also want to integrate the program with other entities that are external to the application itself such as a database, a web-based service or even analysis components that are written in another language such as C++.

In this section of the course, we will cover these topics:

- Designing testable code
- Fixtures
- Mocking
- Testing long-running code
- Testing Parallel code
- Testing GPU code
- Testing C++ in Python code

## Designing testable code

It is worth taking time to think about how your code will be structured, not only for readability that will allow others (and perhaps yourself, when viewing it several months later!) to understand what the code does and how it does it, but also to make testing the code more straightforward. Increasing the ease of writing tests can result in increased test coverage, and thereby reduce the chance that future changes made to the codebase will introduce regressions. In fact, writing testable code often also results in a cleaner and more modular structure that adheres to best practices.

In one software development methodology, Test-Driven Development (TDD), tests are actually written before the code which ensures that the design for testability is in mind from the onset. TDD typically involves a process of adding one test at time. This newest test will initially fail since the functionality has not yet been implemented. The code is then written that allows this test to pass and the process is repeated, ensuring that requirements are thought about before writing the code.   

Another technique which can lead to more testable code is to use pure functions that have no side effects, this is because the outputs depend on the inputs alone. It can be ensured that the results are deterministic in this case since side-effects have been ruled-out. For more information, see the [functional programming paradigm](https://train.oxrse.uk/material/HPCu/software_architecture_and_design/functional), pages in our training material.

Another way to reduce the degree of coupling between a function being tested by a unit test and any dependencies, is to use *dependency injection*. This involves passing an object or function to our code rather than creating such objects internally. For example, 

TODO: Insert example here showing how to make an example more testable - example with side effects -> pure functions? dependency injection?

For example if we were to modify a data structure, such as a dictionary in our function, we need to ensure that the structure is correct both before and after testing if using a function with side-effects. A pure function, that creates a new dictionary, only needs the output tested.

~~~python
def create_rolling_average(data):

~~~


## Fixtures

When writing your tests, you will often find that different tests benefit from the same or similar setup of objects, variables or even connections to allow creation of certain scenarios. After testing, there may also be *teardown* functions or procedures that need to be run in order to clean up files that have be generated or to close database connections that have been opened. This is where fixtures come to the rescue. 

Fixtures are created by using the `@pytest.fixture` decorator on a function and the function name of the fixture can then can be passed to your test functions as an argument. If there is a cleanup part to the code, then the fixture function should be written using the `yield` statement rather than a `return` statement. Anything up to the `yield` statement is setup code, and anything after the statement will be run post-testing in order to clean up.

~~~python
import pytest

@pytest.fixture
def setup_database():
    # Setup database connection
    db = connect_to_database()
    yield db  # Provide the fixture value
    # Teardown database connection
    db.disconnect()

def test_database_operation(setup_database):
    # Test logic using the fixture
    db = setup_database
    assert db.query("SELECT * FROM table") == expected_result
~~~

TODO: Give examples of using fixture scope and parametrising fixtures

Fixtures can be scoped to different part of the code
There are also predefined fixtures e.g. Paths and directories

Give example of using a temp dir

## Mocking

Sometimes we may not want to use "real" objects or functions in our tests, such as those that write to a production database, those that read data from external services or simply those parts that take a long time to run. The technique of mocking allows these objects and functions to be replaced with ones that simulate the same behaviour for the purpose of testing. Doing this allows us to create different scenarios whilst isolating the test code and ensuring that the tests are run in an environment that is independent of external factors. 

TODO: Give examples of using mocking
- Inbuit mocking library vs pytest-mock
- mixing mocking and fixtures
- monkeypatching vs mocking?


## Testing long-running code

As mentioned in the section on mocking above, long-running processes and functions can be replaced with "faked" objects that return a predefined result instantly, however this bypasses actually testing these long running functions. If we need to check that the long running code itself is working correctly, what are our options?

- Simulate passage of time by using the `time` library and check that that dependencies surrounding the long-running code still function correctly
- Run smaller tests in parallel to reduce overall execution time in order to compensate for the long running tests
- Accepting dependencies as parameters will allow easier mocking of long-running processes
- Use timeouts to ensure long-running tests cannot run indefinitely
- Use custom Pytest markers to exclude the long-running tests in the normal development cycle but ensure that the tests are still run on a regular basis or as part of continuous integration pipelines (such as an overnight schedule) this will ensure that these portions of the code do not get overlooked. The hardware that automated testing schedules are run on may also be more performant, thereby reducing the time taken to complete computationally heavy tasks

## Testing Parallel code

- Threads and race conditions with example of counter?
- Difficult to replicate race conditions - good to be able to reproduce in order to ensure it's fixed

Things to try/consider:
- Thread safety analysis - look up details
- Stress tests
- randomised testing may help uncover rare concurrency bugs/ edge cases that will not be found by deterministic tests
- write test to check that access to shared resources is properly synchronised and no deadlock (processes waiting for other member to release resource) or livelock (states change in respect ot one another, no process progresses)
- mock shared resources to simulate concurrent access



## Testing GPU code

- Again depending on hardware, mark tests to be run as part of CI pipeline
- If using libraries such as PyTorch, can easily check values of tensors whilst still on GPU but expensive to keep moving code to and from device
- Memory on GPU devices varies in size, so beware of resource size changing between test runs depending on how GPUs are allocated
- Rather than writing low level functions to repeat the process of compiling GPU code, allocating memory, calling the kernel etc, use a library that abstracts these common processes and concentrate on teh input data and the result


## Testing C++ in Python code
