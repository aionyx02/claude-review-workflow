# REVIEW.md

Instructions for the automated reviewer. Keep it short. When the reviewer (or you) keeps making the same comment, move it here.

## What failure looks like here

<!-- One or two lines, specific to this repo. e.g. "a render silently produces the wrong cut", "a notification goes to the wrong user". -->

## Always review deeply

<!-- Paths where a missed bug is expensive. -->
- `.github/workflows/**`

## Skip

<!-- Generated, vendored, fixture or lock files. -->

## Conventions to enforce

<!-- Only what a linter can't check. Point to docs instead of copying them. -->

## Reporting

- Cite file:line as evidence, never a guess from a name.
- At most 3 Nits per round.
- The summary starts with a count and says plainly when there is nothing to fix.
- Language: English
