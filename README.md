# oss-contribution-log

Status: Phase I Complete
Project: Daft (Eventual-Inc/Daft)
Issue #3792: implement the `levenshtein` string expression
https://github.com/Eventual-Inc/Daft/issues/3792

## Why I Chose This Issue

Daft has no native edit-distance function, so anyone doing fuzzy matching or dedup has to write their own UDF. Adding `levenshtein` makes it a built-in they can call directly.

It fits my background. I work with edit distance for record linkage and data cleaning, so I already know what the function should do and how to tell when it's wrong. The work is also well-scoped: two merged PRs, concat_ws (#6543) and the strip function (#6372), show the exact pattern for the Rust kernel, the Python binding, and the tests. And it's a clean excuse to write a known algorithm, Wagner-Fischer, in Rust, which I want more work in.

Done means `daft.functions.levenshtein(a, b)` returns the edit distance between two string columns, matches PySpark's behavior, handles nulls, and is covered by tests.

## Phase II: Reproduce & Plan

## Phase III: Build

## Phase IV: Submit & Iterate