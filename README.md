<h1 align="center">Safe and Secure</h1>
<h3 align="center">Using CodeQL to validate safety-related applications</h3>
<h5 align="center">@lcartey</h3>

<p align="center">
  <a href="#overview">Overview</a> •
  <a href="#learning-objectives">Learning Objectives</a> •
  <a href="#exercises">Exercises</a>
</p>

## Overview

> <h3>🗒️ This page is available from https://gh.io/safe-and-secure-sandbox!</h3>

Welcome to the "Safe and Secure" sandbox session at GitHub Universe 2024!

What is a sandbox session, you might ask? Well, it is part demo, part workshop, where we aim to provide participants a hands-on, interactive exploration in only 40 minutes!

In this session we cover the use of CodeQL Coding Standards, our functional safety related query ruleset for CodeQL. Our sandbox will be [openpilot](https://comma.ai/openpilot), an open source application that allows customers to retrofit autonomous driving features to their existing vehicles. We will show how CodeQL Coding Standards can be enabled and configured for `commaai/panda`, the central component that controls communication between `openpilot` and the vehicle.

## Learning Objectives

- Able to explain why CodeQL and Code Scanning are a good fit for functional safety use cases.
- Can enable CodeQL Coding Standards on a real-world safety relevant application.
- Understand how to configure CodeQL Coding Standards to remove irrelevant results and adhere to functional safety requirements.

## Exercises

### Exercise 1: Fork the `panda` repository

We will be using a pinned copy of `commaai/panda` repository for this session. You will need to first fork the repository:

 1. Visit https://github.com/githubuniverseworkshops/safe-and-secure-commaai-panda.
 1. Click "Fork".
 1. Choose a suitable account to fork it to. It should either be a public account, or an account with GitHub Advanced Security enabled.

### Exercise 2: Enable CodeQL Coding Standards

We will enable CodeQL Coding Standards on this repo by adding a GitHub Actions workflow file that runs CodeQL and the Coding Standards queries. We take inspiration from the existing MISRA analysis script for a different tool at `tests/misra/test_misra.sh`.

Copy the following configuration to a new file in your repository called `.github/workflows/codeql.yml` and commit the result to `main`:

```yaml
name: CodeQL Coding Standards - MISRA C 2012 analysis

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

permissions:
  actions: read
  contents: read
  packages: read
  pull-requests: read
  security-events: write

jobs:
  misra:
    name: MISRA C:2012
    runs-on: ubuntu-latest
    container:
      image: ghcr.io/commaai/panda:latest
    steps:
      - uses: actions/checkout@v4
      - uses: github/codeql-action/init@v3
        with:
          languages: cpp
          packs: advanced-security/misra-c-coding-standards@2.37.1
          tools: https://github.com/github/codeql-action/releases/download/codeql-bundle-v2.16.6/codeql-bundle-linux64.tar.gz
      - name: Build FW
        run: scons
      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3
        with:
          output: sarif-results
          upload: failure-only
      - name: filter-sarif
        uses: advanced-security/filter-sarif@v1
        with:
          patterns: |
            -**/inc/*.h
            -**/include/*.h
          input: sarif-results/cpp.sarif
          output: sarif-results/cpp.sarif
      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: sarif-results/cpp.sarif
```

This workflow file:
 * Runs the workflow in a docker container, using an image built from the repo that contains all the dependencies.
 * Uses "Advanced Setup" for CodeQL.
 * Initializes CodeQL to analyze C++, use a specified compatible version of CodeQL (2.16.6) and to include the CodeQL Coding Standards MISRA C query pack. This initializes instrumentation of the build.
 * Runs a regular build using `scons`
 * Runs the CodeQL analysis step, producing SARIF results.
 * Saves the SARIF file, and filters out results in `include` and `inc`, as per the existing tool, and uploads them to GitHub.

Analysis will take around 5 minutes.

### Exercise 3: Review the results

Your GitHub Action workflow run should now have completed, and you should see results in the "Security" > "Code Scanning" tab within your repository view to browse and understand the results. Use this view to answer the following questions:

 1. How many results do we have for the Mandatory category? _Hint:_ search for the tag `misra/external/obligation/mandatory`.
 2. How many results do we have for advisory rules in main.c? _Hint:_ use the `path:` filter.
 3. Review a few of the results for Rule 11.4 (`RULE-11-4`). Are they violations? Are they necessary?

### Exercise 4: Apply a Coding Standards configuration file

A Coding Standards configuration file allows you to disapply or deviate against Coding Standards rules which do not need to be adhered to within a given project.

This project does not adhere to the Advisory rules Rule 11.4 and Dir 4.6. In addition, they deviate against certain checked results for Rule 11.3. Let's include that in our configuration:

 1. Create a `coding-standards.yml` file in the root of the repository. Add the following:
    ```yaml
    guideline-recategorizations:
    - rule-id: "RULE-11-4"
      justification: "Advisory: casting from void pointer to type pointer is ok. Done by STM libraries as well"
      category: "disapplied"
    - rule-id: "DIR-4-6"
      justification: "Fixed-width integers not used in this project"
      category: "disapplied"
    deviations:
    - rule-id: "RULE-11-3"
      code-identifier: "cppcheck-suppress misra-c2012-11.3"
      justification: "Alignment checked before casting"
    ```
 1. Update your `.github/workflows/codeql.yml` file to index the configuration files in the repository using the `github/codeql-coding-standards/apply-configuration` action:
    <pre><code language="yaml">
          ...
          - uses: github/codeql-action/init@v3
          ...
          <b>- uses: github/codeql-coding-standards/apply-configuration@v2.37.1</b>
          - name: Build FW
            run: scons
          ...
    </pre></code>
 1. [Optional] Review the `tests/misra/suppressions.txt` file to determine other rules that should be disapplied, and add them to the `guideline-recategorizations`.
 1. [Optional] Search the repository for other suppression markers, and add new `deviation` records to the deviations file.

<details><summary>Full reference `codeql.yml` file</summary>

```yaml
name: CodeQL Coding Standards - MISRA C 2012 analysis

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

permissions:
  actions: read
  contents: read
  packages: read
  pull-requests: read
  security-events: write

jobs:
  misra:
    name: MISRA C:2012
    runs-on: ubuntu-latest
    container:
      image: ghcr.io/commaai/panda:latest
    steps:
      - uses: actions/checkout@v4
      - uses: github/codeql-action/init@v3
        with:
          languages: cpp
          packs: advanced-security/misra-c-coding-standards@2.37.1
          tools: https://github.com/github/codeql-action/releases/download/codeql-bundle-v2.16.6/codeql-bundle-linux64.tar.gz
      - uses: github/codeql-coding-standards/apply-configuration@v2.37.1
      - name: Build FW
        run: scons
      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3
        with:
          output: sarif-results
          upload: failure-only
      - name: filter-sarif
        uses: advanced-security/filter-sarif@v1
        with:
          patterns: |
            -**/inc/*.h
            -**/include/*.h
          input: sarif-results/cpp.sarif
          output: sarif-results/cpp.sarif
      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: sarif-results/cpp.sarif
```


