# Design Decisions

## Classifier and Review as separate services vs. one app
**Options considered:** one combined app with classification and human
review in the same codebase and database, vs. two independently
deployable services that only talk over HTTP with no shared database
**Chosen:** two separate services
**Why:** establishes a repeatable pattern rather than a one-off split.
The boundary (auth on each side, no shared DB, HTTP-only contract) is
now a known shape — the next service we add follows the same template
instead of requiring the boundary to be re-derived from scratch.
