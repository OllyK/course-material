---
name: More Advanced Topics
id: more_advanced_topics
dependsOn: [
  technology_and_tooling.testing.diagnosing_issues
]
tags: [pytest, fixtures, mocking]
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

Another technique which can lead to more testable code is to use pure functions that have no side effects, this is because the outputs depend on the inputs alone. In this case, it can be ensured that the results are deterministic. For more information, see the [functional programming paradigm](https://train.oxrse.uk/material/HPCu/software_architecture_and_design/functional), pages in our training material.

A way to reduce the degree of coupling between a function being tested by a unit test and any dependencies, is to use *dependency injection*. This involves passing an object or function to our code rather than creating such objects internally. In the following example, we have a function `query_database` that utilises a connection to a [SQLite](https://www.sqlite.org/) database. It is going to be difficult to test this function without connecting to the `example.db` database. The contents of our file, named `sqlite_example.py` are shown here:

~~~python
# Original code: Function that performs a database query
import sqlite3

def query_database(sql):
    conn = sqlite3.connect('example.db')
    cursor = conn.cursor()
    cursor.execute(sql)
    result = cursor.fetchall()
    conn.close()
    return result

~~~

If we refactor the function to inject the database connection dependency, we can easily replace that connection during testing with one that is connected to a test database. Alternatively we could replace it with a fake (*mocked*) object that represents the connection, meaning that we do not have to connect to an actual database at all in order to test the function. Information on mocking will be given later in this lesson.

~~~python
# Rewritten code: Performs a database query with dependency injection
import sqlite3

def connect_to_database(filename):
  return sqlite3.connect(filename)

def query_database(sql, connection=None):
    if connection is None:
        raise TypeError("No database connection given.")
    cursor = connection.cursor()
    cursor.execute(sql)
    result = cursor.fetchall()
    connection.close()
    return result
~~~

Here is an example of some tests for these functions, these can be created in a new file named `test_sqlite.py` within a `/tests` directory. If you would like to learn more about the Structured Query Language (SQL) expressions in this example that are used to interact with the database see the [SQL Zoo](https://sqlzoo.net/wiki/SQL_Tutorial) site:

~~~python
import pytest
import sqlite3
from pathlib import Path
from sqlite_example import connect_to_database, query_database

def test_connect_to_db_type():
    """
    Test that connect_to_db function returns sqlite3.Connection
    """
    conn = connect_to_database('test.db')
    assert isinstance(conn, sqlite3.Connection)
    conn.close()

def test_connect_to_db_name():
    """
    Test that connect_to_db function connects to correct DB file
    """
    conn = connect_to_database('test.db')
    cur = conn.cursor()
    # List current databases https://www.sqlite.org/pragma.html#pragma_database_list
    cur.execute('PRAGMA database_list;')
    # Unpack the three parameters returned 
    db_index, db_type, db_filepath = cur.fetchone()
    # Extract just the filename from the full filepath
    db_filename = Path(db_filepath).name 
    assert db_filename == 'test.db'
    conn.close()

def test_query_database():
    """
    Test that query_database retrieves the correct data
    """
    # if the database already exists, delete it
    if Path("test.db").exists():
        Path.unlink("test.db")
    # Create a new test database and enter some data
    conn = sqlite3.connect("test.db")
    cur = conn.cursor()
    cur.execute("CREATE TABLE Animals(Name, Species, Age)")
    cur.execute("INSERT INTO Animals VALUES ('Bugs', 'Rabbit', 6)")
    # Use query_database to retrieve data
    sql = "SELECT * FROM Animals"
    result = query_database(sql, connection=conn)
    # Result returned is a list (cursor.fetchall)
    assert isinstance(result, list)
    # There should just be one record
    assert len(result) == 1
    # That record should be the data we added
    assert result[0] == ("Bugs", "Rabbit", 6)

~~~

As you can see, we can test the `connect_to_database` and `query_database` functions separately. The tests are becoming complex, however, especially the one for `query_database`. Next we can look at how fixtures can help us to reduce this complexity, especially when we want to reuse resources such as a test database.

## Fixtures

When writing your tests, you will often find that different tests benefit from the same or similar setup of objects, variables or even connections to allow creation of certain scenarios. After testing, there may also be *teardown* functions or procedures that need to be run in order to clean up files that have be generated or to close database connections that have been opened. This is where fixtures come to the rescue. 

Fixtures are created by using the `@pytest.fixture` decorator on a function which allows this function to be passed an argument to your tests and used within them. If there is a cleanup part to the code, then the fixture function should be written using the `yield` statement rather than a `return` statement. Anything up to the `yield` statement is setup code, and anything after the statement will be run post-testing in order to clean up.

In the example below, we can use a fixture named `setup_database` to create our test database, add data and also remove the database file once the tests have finished running. As a result, our `test_query_database` function can be simplified and if we want to use the test database in an other tests, we simply need to add `setup_database` as an argument to those tests.

~~~python
import pytest
import sqlite3
from pathlib import Path
from sqlite_example import connect_to_database, query_database

@pytest.fixture
def setup_database():
    # Setup database connection
    conn = sqlite3.connect("test.db")
    cur = conn.cursor()
    cur.execute("CREATE TABLE Animals(Name, Species, Age)")
    cur.execute("INSERT INTO Animals VALUES ('Bugs', 'Rabbit', 6)")
    yield conn  # Provide the fixture value
    # Teardown database connection
    conn.close()
    Path.unlink("test.db")

def test_query_database(setup_database):
    conn = setup_database
    sql = "SELECT * FROM Animals"
    result = query_database(sql, connection=conn)
    # Result returned is a list (cursor.fetchall)
    assert isinstance(result, list)
    # There should just be one record
    assert len(result) == 1
    # That record should be the data we added
    assert result[0] == ("Bugs", "Rabbit", 6)

~~~

:::callout
## Discussion Point: Should We Use Multiple `asserts` in One Test Function?

According to the book, The Art of Unit Testing, a unit test by definition should test a *unit of work*, what this means exactly is itself a point for discussion, but generally it means actions that take place between an entry point (e.g. a declaration fo a function) and an exit point (e.g. the output of a function). It is also often said that each test should fail for only one reason alone. 

Does using multiple `assert` statements in one test contravene these guidelines?

Given that, unlike some other testing frameworks, `pytest` will output an error showing which of the `assert` statements in the test failed and why, does this change the situation`?
:::

By default, any fixtures created will be created when first requested by a test and will be destroyed at the end of the test. We can change this behaviour by defining the *scope* of the fixture. If we we to use the decorator `@pytest.fixture(scope="session")` for example, the fixture will only be destroyed at the end of the entire test session. Modifying this behaviour is especially useful if the fixture is expensive to create (such as a large file) and we do not need to recreate it for each test. 

As well as writing our own fixtures, we can use those that are [predefined/(built-in)](https://docs.pytest.org/en/latest/reference/fixtures.html). For example we may want to use a temporary directory for our files during testing rather than creating files in the directory that we are working from (this is what currently happens when we run our database tests). The built-in fixture `temp_path_factory` allows us to to do this. We can refactor our code to add an extra fixture that uses feature and then it can be used by all the tests that we have written as well as by the `setup_database` fixture. The contents of our `test_sqlite.py` is now:

~~~python
import pytest
import sqlite3
from pathlib import Path
from sqlite_example import connect_to_database, query_database


@pytest.fixture(scope="session")
def database_fn_fixture(tmp_path_factory):
    """
    Uses tmp_path_factory to create a filename in a temp directory
    """
    yield tmp_path_factory.mktemp("data") / "test.db"


@pytest.fixture(scope="session")
def setup_database(database_fn_fixture):
    # Setup database connection
    conn = sqlite3.connect(database_fn_fixture)
    cur = conn.cursor()
    cur.execute("CREATE TABLE Animals(Name, Species, Age)")
    cur.execute("INSERT INTO Animals VALUES ('Bugs', 'Rabbit', 6)")
    yield conn  # Provide the fixture value
    # Close database connection and delete file
    conn.close()
    Path.unlink(database_fn_fixture)


def test_connect_to_db_type(database_fn_fixture):
    """
    Test that connect_to_db function returns sqlite3.Connection
    """
    conn = connect_to_database(database_fn_fixture)
    assert isinstance(conn, sqlite3.Connection)
    conn.close()


def test_connect_to_db_name(database_fn_fixture):
    """
    Test that connect_to_db function connects to correct DB file
    """
    conn = connect_to_database(database_fn_fixture)
    cur = conn.cursor()
    # List current databases https://www.sqlite.org/pragma.html#pragma_database_list
    cur.execute("PRAGMA database_list;")
    # Unpack the three parameters returned
    db_index, db_type, db_filepath = cur.fetchone()
    # Test that the database filename is the same as the one from the fixture
    assert Path(db_filepath).name == Path(database_fn_fixture).name
    conn.close()


def test_query_database(setup_database):
    """
    Test that query_database retrieves the correct data
    """
    conn = setup_database
    sql = "SELECT * FROM Animals"
    result = query_database(sql, connection=conn)
    # Result returned is a list (cursor.fetchall)
    assert isinstance(result, list)
    # There should just be one record
    assert len(result) == 1
    # That record should be the data we added
    assert result[0] == ("Bugs", "Rabbit", 6)

~~~


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
