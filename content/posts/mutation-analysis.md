+++
title = 'How to evaluate if the test suite is good?'
date = 2024-10-20T19:02:29+02:00
draft = true
tags = ["solidity", "golang"]
author = "Kamil Chmielewski"
description = ""
+++

Every once in a while a client comes in asking "What else can we do to make our
codebase more secure?". Depending on the types of bugs that we have found during
the audit engagement one of the responses might be: 
- "Well, you should re-write this project from scrach" (never happens)
- "Well, your codebase was solid, you can implement some runtime monitor
  solutions that would react to security incidents automatically"
- "Well, you could improve your test suite."

<!--more-->

To decide where you want to go with your test suite, you have to evaluate where
you currently are. 

### How do you test the test suite?

### What is mutation testing?

Mutation is a technique that allows you to evaluate the tests. Since the tests
evaluate if the application conforms to the specification, the question arises
how do you evaluate the tests themselves? Mutation testing is a valid answer to
this question. It is based on a very simple principle - if you change the logic
in the code in any way - the tests should catch this since the code no longer
reflects the business logic of your application. 


### How to perform mutation testing?

Mutation analysis consists of two stages:
1. Generating the modified versions of your code.
2. Running the test suite over the modified version to see if the tests will
   catch the change.

### How to generate the mutants?

### Evaluate the test suite by slaying the mutant

### Automating mutation analysis with a simple script

### Benefits of mutation testing

Mutation testing is not a silver bullet when it comes to security of the
codebase. However, it is a low-effort way to create value for your clients. It
provides insights on what branches of the code would benefit from higher
coverage before the deployment. Stronger test suite makes it easier to change
the code in the future iterations as well.

Regression testing (?); Performing mutation analysis frequently when new
features are introduced can show if the new code is shipped undertested and
introduces a regression as compared to the old code.

### What are the limitations of mutation testing?

Mutation testing's limitations stem from the limitations of unit tests
themselves. Since unit tests are often stateless, they won't catch the bugs that
require many interactions to occur before the actual bug can happen.

### References

- [Introduction into mutation testing by Joran Honig](https://medium.com/swlh/introduction-into-mutation-testing-d6512dc702b0)

