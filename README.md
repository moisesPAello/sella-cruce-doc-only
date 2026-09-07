# Sella Cruce

Bank statement reconciliation for a Mexican accounting firm. It reads a bank statement PDF, matches
every movement against an Excel of expected movements, stamps the annotated PDF, and hands back the
exceptions a human still has to decide.

I built it as the only developer at the firm, from requirements to production, between July 2025 and
July 2026.

*English · [Español](README.es.md)*

**The source is closed** — it belongs to the firm and it processes their clients' financial data.
This repository documents the engineering: the problem, the decisions, and what they cost.

---

## The problem

Before this existed, reconciliation was a person with two documents open.

On one side, a bank statement PDF: a month of movements, each with a date, an amount, a reference,
and a description written however that particular bank writes descriptions. On the other, an Excel
of the movements the firm's records said *should* be there.

The job was to go line by line and answer one question for each row: is this movement in both
places? When it was, you marked the PDF. When it wasn't, you flagged it. A statement with a few
hundred movements took an afternoon, and the work was pure attention — no judgment until something
didn't match, and by then you had spent your attention on the three hundred rows that did.

![The manual process: a person comparing a bank statement PDF against an Excel of expected movements, row by row, with the mismatches that make exact matching useless.](assets/diagrams/before-manual-process.svg)

Two things made it worse than it sounds.

**Amounts don't line up cleanly.** The same transaction can differ by a few cents between the bank
and the ledger, arrive on a different day, or appear in the Excel as one row and in the statement
as two. A dumb exact-match tool would flag half the statement as an exception and save nobody
anything.

**Every bank lays out its PDF differently.** Not slightly — structurally. Header in a different
place, columns in a different order, running balance included or not, dates in a different format,
totals in a different position. There is no standard. And it is not even one layout per bank: the
same institution ships a completely different layout for a credit card statement than for a
checking account.

That last point is the whole shape of the project. It is why roughly 40% of the codebase is parsing
and not reconciliation.

![Nine statement layouts side by side across eight institutions, each with its header, columns and totals in a different place. Five of the nine do not end with a totals row at all.](assets/diagrams/statement-formats.svg)

---

## What it does

Nine parsers covering eight institutions. Each one is a folder with its own module and its own
declarative config; adding a format means adding a folder, not touching the core.

![The pipeline: one extraction path from PDF to normalized movements, then two stamping modes — stamp every movement, or stamp by match against the expected-movements Excel with human resolution for ambiguous groups.](assets/diagrams/pipeline.svg)

There are three entry points sharing those stages: full reconciliation against an Excel, a
statement-wide pass with no Excel, and a plain export of parsed movements to CSV or XLSX.

---

## Decisions, and what they cost

### The tool has two modes because the design changed halfway

The first version stamped indiscriminately: give it a statement, tell it the bank and the currency,
and it found every movement of the requested type and marked them all. No ledger required.

That was useful and insufficient. Marking every movement tells you what the bank says; it doesn't
tell you whether your books agree. So the second version took the expected-movements report as an
input and stamped by match instead — same extraction, different question.

What I didn't expect is that the first mode never became obsolete. Stamping everything is the right
tool when you want a clean annotated statement and don't have a report to reconcile against, and
the team kept using it. Both modes shipped and both stayed.

**Cost:** two reconciliation paths to maintain, sharing extraction but diverging after it. The
specific-matching path is where the complexity concentrated, and it's the module that eventually
grew too large.

### Stamps are cross-references, not checkmarks

A matched movement doesn't get a tick. It gets an identifier that ties that line of the PDF to its
counterpart in the report, so the annotated statement is navigable on its own: an auditor holding
the stamped PDF can trace any marked line back without the original spreadsheet in front of them.

That requirement is why stamping is a real engine and not a drawing call — it has to find the
amount on the page, find free space near it that doesn't collide with existing content, and place
the annotation there.

### Ambiguity is a pipeline stage, not an error

The matcher scores candidates and produces three outcomes: a confident match, no match, or an
ambiguous group. Early on, ambiguous meant failure — the run stopped and a human started over by
hand.

That was the wrong model. Ambiguity is the normal state of this problem, not an exception to it, so
I made it a stage: ambiguous groups travel back to the UI, the user picks, and the result is
reassembled server-side and continues through stamping.

**Cost:** the pipeline is no longer a single pass. State has to survive a round trip through a
person, which is where most of the session handling below comes from.

### File sessions live on the server

The first version passed files to the browser as base64 and kept them in `sessionStorage`. It
worked until a real statement blew past the storage quota and the tab silently lost the document.

Files now live in a server-side session directory keyed by UUID, with a 30-minute TTL and expired
sessions cleaned up when the API starts. The browser holds an ID, not a document.

**Cost:** the API now owns temp files and their lifecycle, which is a class of bug that only shows
up after something has been running for weeks.

### One parser and one config file per format, discovered at runtime

The core doesn't know which banks exist. It scans the parser directory and loads what it finds, so
onboarding a new statement layout is adding a folder.

There is also a path for statements whose format has no parser yet: rather than failing, the run
falls through to manual bank selection so a user can still push a document through the parts of the
pipeline that don't depend on layout.

**Cost, and this one bit me:** a separate alias table drives bank detection from the PDF text, and
it drifted out of sync with the parser folders — it lists an institution with no parser and omits
two that have one. Two sources of truth for the same fact, which is exactly the failure mode this
design was supposed to avoid. The registry should have been derived from the folders, not
maintained beside them.

### OCR decides per page, not per document

Scanned pages get OCR'd; digital pages don't. The check runs page by page and the OCR layer is
merged with the digital one, deduplicated by bounding-box overlap.

The reason is that real statements are mixed — a digital PDF with one scanned insert. Deciding at
the document level means either OCR'ing clean pages, which is slow and loses accuracy, or skipping
a page that needed it.

### Amounts are `Decimal`, never `float`

Non-negotiable in accounting software, and cheap if you decide it on day one instead of hunting
rounding drift later.

### The application ships as a compiled binary

The API image is built multi-stage: compile to bytecode, package with PyInstaller, delete the
source, and copy only the binary into a slim runtime.

The reason is that it runs on the firm's own server, and the source is the asset.

**Cost:** builds got slow, and debugging production got meaningfully harder — a stack trace from a
frozen binary tells you less than one from source. If I were choosing again for a system this size,
I'd want a clearer answer on whether that trade was worth it.

### Errors are codes, not strings

Around a dozen structured error codes shared between API and frontend, so a failure says *bank not
detected* or *no room to stamp near this amount* instead of *something went wrong*. Users of an
internal tool can act on the first kind; they file a ticket for the second.

---

## Stack and deployment

**Backend:** Python, FastAPI, pandas, Tesseract for OCR. 17 functional HTTP endpoints across
processing, export, and session modules.

**Frontend:** Next.js with the App Router, TypeScript, React.

**Deployment:** two containers on an internal server. Non-root users, healthchecks, CPU and memory
limits, environment-driven configuration, CORS allowlist with no wildcard in production.

**CI/CD:** four GitHub Actions workflows — tests on a matrix, tagged releases publishing built
images, auto-deploy through a self-hosted runner on the server, and one that fails the build when
code changes and documentation doesn't.

![Deployment: a git tag triggers a build that publishes both images, a self-hosted runner deploys them to the firm's internal server, and the accounting team reaches the Next.js container over the LAN.](assets/diagrams/deployment.svg)

The frontend resolves the API address from the browser's own hostname rather than a baked-in build
variable, so one image runs on any host without a rebuild.

---

## Where it got to

- **9 statement formats** across 8 institutions.
- **~30,000 lines of Python**, about 40% of it parsing and 22% of it tests.
- **3–5 daily users:** a core team of three, sometimes four, plus other teams that picked it up on
  their own. Heavy weeks during VAT refund periods.
- **5 tagged releases.** First packaged build November 2025; formal production stack March 2026.
- One developer, start to finish.

---

## What I'd do differently

**Derive the bank registry from the filesystem.** The drift described above was predictable the
moment I wrote the second source of truth.

**Split the reconciliation module earlier.** The specific-matching module grew past 1,700 lines
because every new case had an obvious home in it. There was a point where it should have become
three modules and I went past it without noticing.

**Decide the distribution model before building around it.** Compiling to a binary shaped the
Dockerfile, the CI, and the debugging story. It was a reasonable call, but I made it partway
through rather than up front, and the whole build pipeline had to be reworked to accommodate it.

---

## On the closed source

The code is the property of the firm that commissioned it, and it processes financial documents
belonging to their clients. This repository describes the engineering without reproducing the
system: no parsing rules, no matching thresholds, no configuration, no client data. Screenshots use
invented data.

Happy to talk through any of it in more detail.
