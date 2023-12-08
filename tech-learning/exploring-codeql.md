---
layout: default
title: "Exploring CodeQL: A Practical Guide"
permalink: /tech-learning/exploring-codeql/
description: "Exploring CodeQL: A Practical Guide"
---
<h1>{{ page.title }}</h1>
<p class="subtitle">December 2023</p>
<div style="text-align: center;">
    <img src="/images/codeql-article.jpeg" alt="CodeQL" title="CodeQL: A Practical Guide">
</div>

## Intro
CodeQL is a tool that was developed by GitHub to identify vulnerabilities and thereby enhance code security. It plays a vital role in shifting security left in the SDLC especially if integrated into a CI/CD pipeline for automatic code checking. A number of companies like Wise, Palantir and Tide seem to be using it to scan their codebases. As such, I thought it would be a great addition to my application security skill set. This article aims to practically get you up and running with learning CodeQL. I will use Python as my language of choice (as I feel decently comfortable with it in describing code vulnerabilities) but you can choose whatever language you feel comfortable with as well as what CodeQL currently supports.

## Getting Started
Firstly, create a new repository on GitHub with a name of your choice, preferably descriptive in nature. I picked CodeQL-Python-Learning as can be seen [here](https://github.com/tmendonca28/CodeQL-Python-Learning). This will house all my experiments when it comes to using CodeQL.
Git clone the repo locally and you are ready to get started.

## Installing CodeQL for Python
We need to download and install the CodeQL CLI locally. This can be done by visiting the CodeQL GitHub and following the instructions in the [docs](https://docs.github.com/en/code-security/codeql-cli/getting-started-with-the-codeql-cli/setting-up-the-codeql-cli). I was working in Windows so I downloaded the bundle, unzipped it and added it to my system path so that I could access it using _codeql_. Once installed you can run _codeql resolve languages_ to see what languages it currently supports as seen below.
<div style="text-align: center;">
    <img src="/images/codeql-languages.png" alt="CodeQL languages" style="max-width: 80%; height: auto;" title="CodeQL languages">
</div>

## Writing and running your first codeql query for python
GitHub's [security lab repository](https://github.com/github/securitylab) has a number of CodeQL queries that one can analyze with respect to Python.\
A best practice one can follow is to create a directory within your local repo to house queries, and aptly name it _queries_. You can then create a .ql file, and potentially name it in line with a specific vulnerability you are scanning for. For example, if I am trying to scan against sensitive data exposure, I would call my file "sensitive_data_exposure.ql".\
I created a simple python file, that contained a redundant 'if' statement. Starting off with this to understand the workflow of CodeQL as well as the query language used. The 'pass' makes the if statement redundant as flow would simply pass over the if and do nothing.
The python file looked as follows:
```python
print("Hello there")
if error: pass
```
The next step is to create a database of the code, this allows you to check for potential vulnerabilities in your code. Use the following command to do so:
```
codeql database create codeqldb --language=python
```
Once this completes, we have a dir created called codeqldb, which we can then query for vulnerabilities or errors.

We also need to create a _qlpack.yml_ file that declares the necessary dependencies, we can create this in the _queries_ directory.\
Mine looked as follows:
```
name: codeql/python-queries
version: 0.0.0
libraryPathDependencies: codeql-python
```
I then created a simple query to find the 'redudant if statement' and called it quick-query.ql as seen below:
```
/**
 * @name Find Redundant ifs
 * @description Detects redundant ifs in Python code.
 * @kind problem
 * @problem.severity error
 * @precision very-high
 * @id python/redundant-ifs
 * @tags security
 */

import python

from If ifstmt, Stmt pass
where pass = ifstmt.getStmt(0) and
  pass instanceof Pass
select ifstmt, "This 'if' statement is redundant."
```
You can find out more details about the metadata included to a query [here](https://codeql.github.com/docs/writing-codeql-queries/metadata-for-codeql-queries/)

## Querying the database
Firstly, we can create a folder called _codeql-results_ where we can store results of our queries.\
I then ran the following command:
```
codeql database analyze codeqldb .\queries\quick-query.ql --format=sarif-latest --output=.\codeql-results\python-queries.sarif
```
This outputs the results in a [SARIF](https://docs.oasis-open.org/sarif/sarif/v2.1.0/sarif-v2.1.0.html) format.\
You can then install a SARIF viewer extension, to view the results.
<div style="text-align: center;">
    <img src="/images/redundant-if.png" alt="redundant-if" style="max-width: 80%; height: auto;" title="redundant-if">
</div>
This is a really simple example to get to grips with setting up and running CodeQL locally, the next step would, of course, be running it in a CI/CD pipeline.
I will be exploring how to add more queries for specific python-related vulnerabilities to understand the query language better.

## Conclusion
Why do all this? This simple example allows you to grasp the fundamentals of CodeQL, the ultimate goal is to fortify your applications against potential security threats thereby instilling a more secure SDLC.