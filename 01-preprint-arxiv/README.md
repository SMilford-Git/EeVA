---
editor_options: 
  markdown: 
    wrap: 72
---

# EeVA

## Ethical eValuation Agent

EeVA is a research prototype for **LLM-assisted ethical evaluation** of
technical or operational use cases. A user uploads a use case, the
system evaluates it through multiple ethical frameworks, generates a
synthesis, stores the result in an n8n data table, and makes the result
retrievable through a simple web interface.

EeVA is designed to support **ethical reflection**, not to deliver a
single final moral answer. The core idea behind the project is that
ethical reasoning is plural, contested, and often resistant to reduction
into one optimisation rule. Rather than pretending to solve ethics, EeVA
aims to help users situate a proposal within a broader ethical landscape
and reflect more rigorously on its strengths, weaknesses, tensions, and
trade-offs.

------------------------------------------------------------------------

## Why this project exists

Technical teams often need practical guidance. They need to make
decisions, justify trade-offs, document reasoning, and move from ideas
into implementation. At the same time, ethical analysis rarely produces
one neat, universally accepted answer. Different ethical frameworks can
illuminate different concerns, and the same intervention can appear
persuasive under one lens and deeply problematic under another.

EeVA was created as a proof-of-concept response to that gap.

Instead of asking users to become specialists in moral philosophy, or
requiring constant access to multiple ethicists, EeVA uses large
language models to provide structured comparative evaluations across
several ethical frameworks. The system is intended to improve the
**quality of reflection around a use case**, not to replace human
judgement.

------------------------------------------------------------------------

## What EeVA does

At a high level, EeVA currently follows this flow:

1.  A user uploads a use case file through a webpage.
2.  n8n creates a `run_id` and stores an initial record.
3.  A worker workflow evaluates the use case across multiple ethical
    frameworks.
4.  The system produces a synthesis / summary.
5.  The result is stored in the `ethical_evaluation_runs` data table.
6.  A separate results workflow retrieves the current state by `run_id`.
7.  The frontend displays the summary and framework evaluations to the
    user.

The current stable path is based on **TXT** and **MD** uploads.

------------------------------------------------------------------------

## Project architecture

EeVA currently uses **three n8n workflows**.

### 1. Starter workflow

Purpose: - accepts the upload - validates file type / size - creates the
`run_id` - inserts the initial row in the data table - returns an
immediate response to the frontend - triggers the worker workflow in the
background

This separation was important. Earlier versions tried to handle upload,
response, and long-running evaluation in the same flow, which caused the
webpage to hang or time out.

**Repository file:** `Ethical Evaluation Starter Workflow.json`

![Starter Workflow](Starter%20Workflow.png)

### 2. Worker workflow

Purpose: - receives the file and metadata from the starter workflow -
extracts text - runs the framework evaluations - builds the synthesis -
prepares final output - updates the existing row from `processing` to
`complete`

This workflow now functions as a **sub-workflow** rather than a public
webhook workflow.

**Repository file:** `Ethical Evaluation Worker.json`

![Worker Workflow](Worker%20Workflow.png)

### 3. Get-results / emitter workflow

Purpose: - accepts a `run_id` - looks up the existing row in the data
table - returns `processing`, `complete`, `error`, or `not_found` -
allows the frontend to retrieve results separately from the upload step

**Repository file:** `Ethical Evaluation Agent - Emitter Workflow.json`

![Emitter Workflow](Emitter%20Workflow.png)

------------------------------------------------------------------------

## Data model

The main n8n data table used in the project is:

`ethical_evaluation_runs`

Important fields include: - `run_id` - `status` - `use_case_name` -
`source_filename` - `upload_timestamp` - `version_tag` - `summary` -
`evaluations` - `report_html` - `error_message`

### Important note

Older versions of the workflows sometimes used `evaluations_json`. That
field name is legacy and caused several downstream problems. The
intended current field is:

`evaluations`

------------------------------------------------------------------------

## Frontend

EeVA includes a lightweight HTML interface that allows a user to: -
enter a use case name - upload a file - start an evaluation - receive a
`run_id` - manually check the evaluation status - display the summary
and framework evaluations

The frontend renders headings and evaluation cards in a readable format
and falls back to `source_filename` if no usable display name is
available.

### Current behaviour

The current frontend uses a **manual “Check status” button** rather than
aggressive auto-polling. This was introduced to reduce unnecessary n8n
executions.

### Current stable input

Although some earlier project conversations explored PDF support, the
stable input path described in this repository is: - `.txt` - `.md`

The binary upload field name is:

`file`

------------------------------------------------------------------------

## Repository contents

This repository currently contains a mix of workflows, diagrams,
prompts, framework material, and test artefacts.

### Core workflow files

-   `Ethical Evaluation Starter Workflow.json`
-   `Ethical Evaluation Worker.json`
-   `Ethical Evaluation Agent - Emitter Workflow.json`

### Workflow diagrams

-   `Starter Workflow.png`
-   `Worker Workflow.png`
-   `Emitter Workflow.png`

### Framework and prompt material

-   `Framework Overviews - No Bibliographies.docx`
-   `Prompts with Framework-Relative Metrics.docx`

### Example use cases / test material

-   `Use-Case 1 - ...`
-   `Use-case 2 - ...`
-   `Use-case 3 - ...`
-   `20260406 WP V2.0 Full Test 1.0 - UC 1.mhtml`
-   `20260406 WP V2.0 Full Test 1.0 - UC 2.mhtml`
-   `20260406 WP V2.0 Full Test 1.0 - UC 3.mhtml`

These files document the current state of the prototype and preserve
examples of testing and workflow evolution.

------------------------------------------------------------------------

## How to use this repository

### 1. Import the workflows into n8n

Import the three JSON workflow files into your n8n instance.

### 2. Set up the data table

Create or connect the table used by the workflows:

`ethical_evaluation_runs`

Make sure the expected fields exist and that the completion step writes
to `evaluations`, not `evaluations_json`.

### 3. Configure credentials and endpoints

Update the workflows to match your own: - n8n credentials - data table
connection - webhook URLs - LLM model / provider settings

### 4. Connect the frontend

Point the HTML interface to your own: - starter webhook URL -
get-results webhook URL

### 5. Run a test using TXT or MD input

Upload a simple use case file, capture the returned `run_id`, and
retrieve the result through the emitter workflow.

------------------------------------------------------------------------

## Known issues and lessons learned

A number of practical issues shaped the current design.

### Starter response timing

The frontend must receive a response quickly. Long-running processing
should happen after the starter responds.

### Worker configuration

The worker should remain a callable sub-workflow rather than a
public-facing webhook.

### Metadata preservation

Fields such as `run_id`, `use_case_name`, `source_filename`,
`upload_timestamp`, and `version_tag` must be preserved through later
formatting / build nodes. If these are lost, completion updates may fail
and rows can remain stuck at `processing`.

### Schema consistency

If the backend uses `evaluations_json` while the frontend expects
`evaluations`, results will not render properly.

### Frontend debugging principle

When something looks wrong in the HTML, check the **live get-results
response first**. The frontend cannot display fields that are not
actually being returned.

### Execution cost

Frequent auto-polling creates unnecessary n8n executions. Manual status
checking is currently the preferred behaviour.

------------------------------------------------------------------------

## Research framing

EeVA should not be understood as an “ethics answer machine.” Its purpose
is not to adjudicate moral questions with false certainty. The broader
research argument behind the project is that ethical problems are often
plural, contested, and framework-dependent. In that context, the role of
a tool like EeVA is to support structured reflection, surface tensions,
and improve the atmosphere for ethical decision-making.

In that sense, EeVA is better described as a **reflective ethical
evaluation prototype** than as a final decision-maker.

------------------------------------------------------------------------

## Current status

What is already in place: - the three-workflow architecture - a working
HTML frontend - run-based retrieval using `run_id` - storage of summary
and evaluations in the n8n data table - framework-based evaluation and
synthesis generation

What still needs improvement: - stronger synthesis integration across
frameworks - continued hardening of insert-row / update-row
reliability - cleanup of minor frontend wording and configuration
issues - future support for additional file types - refinement of the
project as a more fully agentic system

------------------------------------------------------------------------

## Intended audience

This repository is most relevant for: - researchers in AI ethics,
bioethics, and philosophy of technology - developers building reflective
evaluation tools with n8n - teams interested in multi-framework ethical
analysis of technical proposals - anyone exploring how LLMs might assist
ethical deliberation without collapsing it into a single score or rule

------------------------------------------------------------------------

## Project status note

EeVA is a **research prototype**. It is not production software, not a
regulatory compliance tool, and not a substitute for formal ethical
review, governance, or expert judgement.
