---
title: Prompt words for Repository-Level Code Generation Tasks
tags: Repo,Issue,Code Generation
---

# Prompt Structure for Repository-Level Code Generation Tasks

Prompts for repository-level code generation tasks can be divided into three main sections:

## 1. Problem Description
This section describes the task that the agent needs to accomplish within the codebase. Its structure is similar to a **GitHub Issue**.

## 2. Requirement Analysis
This section contains a series of **manually written requirements**. It typically specifies the behaviors that the implemented solution should exhibit, which will be **directly verified during testing**.

## 3. Interface Specification (Optional)
This is an optional field, used only when the task solution requires modifying or creating new public interfaces. It includes interface information for all modified or created classes and functions, including:
- Their signatures
- Their file paths

### Importance of Interface Specification
The Interface Specification plays a crucial role in **reducing false positives in unit test validation**, particularly in code changes related to **feature additions**. When adding new functionality, relevant unit tests are written against the specific set of interfaces exposed by the newly added classes and functions.

---

*Below is a feature request for Open Library as an example:*

# Feature Request: Add Google Books as Metadata Source to BookWorm

## Problem / Opportunity

BookWorm currently relies on Amazon and ISBNdb as its primary metadata sources. This creates challenges when:

- Metadata is missing, malformed, or incomplete
- Processing books with only ISBN-13s
- Incomplete records submitted via promise items or `/api/import` fail enrichment

**Impact:** Poor-quality entries remain in Open Library, affecting data quality and import success rates, especially for less common or international titles.

## Justification

Integrating Google Books as a fallback metadata source will:

- Enhance Open Library's ability to supplement and stage richer edition data
- Improve completeness of imported books
- Reduce failed imports due to sparse metadata
- Increase user trust in the import experience

**Measurable Impact:** Increased import success rates and reduced placeholder entries (e.g., "Book 978...").

## Success Criteria

- BookWorm successfully fetches and stages metadata from Google Books using ISBN-13
- Automated tests confirm accurate parsing of Google Books responses, including:
  - Correct mapping of available fields (title, subtitle, authors, publisher, page count, description, publish date)
  - Proper handling of missing/incomplete fields (e.g., no authors, no ISBN-13)
  - Appropriate handling of zero or multiple matches from Google Books

## Proposal

Introduce Google Books as a fallback metadata provider in BookWorm. When Amazon lookup fails or only ISBN-13 is available, BookWorm will:

1. Attempt metadata fetch from Google Books API
2. Stage retrieved metadata for import
3. Update source logic and metadata parsing
4. Ensure `google_books` records are correctly processed

## Requirements

### Configuration
- Add `"google_books"` to `STAGED_SOURCES` tuple in `openlibrary/core/imports.py`

### URL Handling
- Use staging URL: `http://{affiliate_server_url}/isbn/{identifier}?high_priority=true&stage_import=true`
- Support identifiers: ISBN-10, ISBN-13, or B*ASIN

### Data Processing
- **Record Supplementation:** Extend existing `source_records` rather than replacing (in `openlibrary/plugins/importapi/code.py`)
- **Fallback Logic:** Use Google Books for ISBN-13 when:
  - Amazon returns no results
  - Both `high_priority=true` and `stage_import=true` parameters are set
- **Duplicate Handling:** Skip staging and log warning if Google Books returns multiple results
- **Batch Processing:** Update `scripts/promise_batch_imports.py` to use `stage_bookworm_metadata`

### Metadata Fields
Parse and stage these minimum fields from Google Books responses:
- `isbn_10`, `isbn_13`, `title`, `subtitle`, `authors`
- `source_records`, `publishers`, `publish_date`
- `number_of_pages`, `description`

## Interface Specification

### Functions

#### `fetch_google_book`
- **Location:** `scripts/affiliate_server.py`
- **Input:** `isbn` (str) - ISBN-13
- **Output:** dict containing raw JSON response (HTTP 200) or None
- **Description:** Fetches metadata from Google Books API

#### `process_google_book`
- **Location:** `scripts/affiliate_server.py`
- **Input:** `google_book_data` (dict) - JSON data from Google Books
- **Output:** dict with normalized Open Library edition fields or None
- **Description:** Processes Google Books API data into normalized format

#### `stage_from_google_books`
- **Location:** `scripts/affiliate_server.py`
- **Input:** `isbn` (str) - ISBN-10 or ISBN-13
- **Output:** bool - True if metadata successfully staged
- **Description:** Fetches and stages metadata, adds to import batch

#### `get_current_batch`
- **Location:** `scripts/affiliate_server.py`
- **Input:** `name` (str) - batch name ("amz", "google")
- **Output:** Batch instance
- **Description:** Retrieves or creates batch object for staging

### Classes

#### `BaseLookupWorker`
- **Location:** `scripts/affiliate_server.py`
- **Description:** Base threading class for API lookup workers

**Method:** `run(self)`
- **Description:** Processes items from queue, invokes `process_item` callable

#### `AmazonLookupWorker`
- **Location:** `scripts/affiliate_server.py`
- **Description:** Threaded worker for Amazon API lookups (extends `BaseLookupWorker`)

**Method:** `run(self)`
- **Description:** Batches and processes Amazon API lookups