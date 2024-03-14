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

Up to this point, our examples have focused on unit testing of simple functions that form part of larger program to analyse inflammation data from patients in a drug trial. As we develop this program further, the complexity of the codebase will increase which, in turn, means a larger number of tests will need to written and organised.  We may also want to integrate the program with other entities that are external to the application itself such as database, a web-based service or even analysis components that are written in another language such as C++.

In this section of the course, we will cover:

- Designing testable code
- Mocking
- Testing long-running code
- Testing Parallel code
- Testing GPU code
- Testing C++ in Python code

## Designing testable code

It is worth taking time to think about how your code will be structured, not only for readability that will allow others (and perhaps yourself, when viewing it several months later!) to understand what the code does and how it does it, but also to make testing the code more straightforward. Increasing the ease of writing tests can lead to increased coverage, and thereby reduce the chance that future changes made to the codebase will introduce regressions. In fact, writing testable code often also results in a cleaner and more modular structure that adheres to best practices.

In one software development methodology, Test-Driven Development (TDD), tests are written before the code which ensures that the design for testability is in mind from the onset. In addition, following the functional programming paradigm (described in section ?), utilising pure functions that have no side effects can also lead to easier test creation since consistent results are always expected for the same inputs. 

## Mocking

## Testing long-running code

## Testing Parallel code

## Testing GPU code

## Testing C++ in Python code
