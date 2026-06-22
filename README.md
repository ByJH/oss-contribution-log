# oss-contribution-log

Status: Phase II In Progress
Project: Daft (Eventual-Inc/Daft)
Issue #3792: implement the `split_part` string expression
https://github.com/Eventual-Inc/Daft/issues/3792

## Why I Chose This Issue

Issue #3792 tracks the PySpark string functions Daft is still missing. split_part is one of
them. It splits a string on a delimiter and returns a single field by position. Daft already
has split, but it returns the whole list of fields. There's no function that splits and pulls
out one field directly, which is the gap I want to fill.

It fits my background. I work with delimited string data often enough to know where pulling a
single field by position saves a two-step split and index. That comes up with CSV-style
fields, file paths, and IDs packed into one column. The work is also well-scoped. substring_index
is already in the tree and takes the same shape of arguments, a string, a delimiter, and an
index. Its Python and Rust sides are both visible, so I have a close template to copy instead
of guessing at the structure.

Done means daft.functions.split_part(s, delimiter, part) returns the part-th field after
splitting on the delimiter, matches PySpark's split_part, returns null on null input, and is
covered by tests.

## Phase II: Reproduce & Plan

### Understanding the issue

split_part should take a string column, a delimiter, and a 1-based field index, then return
that field after splitting the string on the delimiter. PySpark exposes this as split_part.
Daft has split, which returns the full list of fields, but no function that returns a single
field by position. The gap is that the function is absent, not that it returns a wrong answer.

### Environment setup

Built Daft from source in WSL under a venv. daft.__version__ reports 0.3.0-dev0, so the
editable build is reading the working tree. Installed the pre-commit hooks with
pre-commit install --install-hooks so commits format to the project's mypy, ruff, and codespell
config without me running anything by hand. The source build and the hook install both went
through on the first try, with no errors worth recording.

### Steps to reproduce

The behavior here is an absence, so reproducing it means showing the function can't be reached:

1. Activate the venv and confirm the build: python -c "import daft; print(daft.__version__)" -> 0.3.0-dev0
2. Try to import it: python -c "from daft.functions import split_part"
3. Observed: ImportError: cannot import name 'split_part' from 'daft.functions' (daft/functions/__init__.py)

Once implemented, the import should succeed and daft.functions.split_part(s, delimiter, part)
should return the selected field for each row.

### Reproduction evidence

Branch: https://github.com/ByJH/Daft/tree/issue-3792-split-part
Finding: git grep -i split_part returns no matches in the tree, so there's no existing or
partial implementation. split exists for the full list of fields, but nothing returns a single
field by position.

### Solution plan (UMPIRE)

Understand: Daft has split, which returns every field as a list, but there's no way to get one
field by position. The task is to add split_part(s, delimiter, part) so it's callable like the
other string expressions and matches PySpark's, including null handling.

Match: substring_index is the closest existing function and the one to copy. It is a
three-argument string function that takes a string, a delimiter, and an integer, with its
kernel in Rust and a thin Python wrapper:
- Rust kernel: src/daft-functions-utf8/src/, its own module registered in lib.rs
- Python wrapper: daft/functions/str.py, the def substring_index function
- export: daft/functions/__init__.py
Run git grep -in substring_index | cat to get the exact paths. The four-file layout is the same
one I traced through soundex.

Plan:
1. Add the Rust kernel in src/daft-functions-utf8/src/split_part.rs. It splits each value on
   the delimiter and returns the part-th field. The index is 1-based, a negative index counts
   from the end, and an out-of-range index returns an empty string to match Spark. A null in
   any argument gives a null out.
2. Register it in lib.rs the same way the others are, with a mod line, a pub use, and an add_fn
   call.
3. Add the Python wrapper in daft/functions/str.py as def split_part(expr, delimiter, part),
   which calls _call_builtin_scalar_fn("split_part", expr, delimiter, part). Include a docstring
   and a doctest example.
4. Export it in daft/functions/__init__.py.
5. Add tests next to the existing string-function tests.

Implement: (Phase III) branch above; commits to follow.

Review: read CONTRIBUTING before the PR. Commit messages use the feat: prefix, so the commit
will read feat: add split_part string function. Confirm I've been assigned split_part, or that
no one else has a PR open for it, before pushing one.

Evaluate: unit tests covering the cases that matter. A basic split, where
split_part("a,b,c", ",", 2) gives "b". The first and last fields, where index 1 gives "a" and
index -1 gives "c". An out-of-range index, where 5 gives an empty string. And a null input,
which should return null. Confirm Spark's behavior for the edge cases of part 0, out-of-range,
and negative index against the PySpark docs before implementing. Spot-check a couple of values
against PySpark, then run the string-function test suite to confirm nothing else broke.


## Phase III: Build

## Phase IV: Submit & Iterate