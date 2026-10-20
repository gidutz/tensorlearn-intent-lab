# Intent

## Goal
A support coordinator needs to preview the tickets contained in a CSV file before any
later import is approved. The outcome is a pure preview function that takes CSV text
and returns either the full list of parsed tickets, in input order, or a single error
code explaining why the file cannot be imported. Nothing is saved or changed; the
coordinator only sees what an import would produce.

## Scope
The future implementation may add exactly two files: `src/import-tickets.mjs`, which
exports the preview function, and `tests/import-tickets.test.mjs`, which covers every
acceptance case below. It reuses the existing `ticket()` helper from `src/tickets.mjs`
as the stable ticket representation rather than redefining it.

Deliberately excluded: any UI, database, network calls, file writes, persistence of
the preview, and new package dependencies. Do not modify `src/tickets.mjs`, anything
under `.github/` (including the grading workflow), `CLAUDE.md`, `package.json` or the
existing tests. If a change outside the two allowed files seems necessary, stop and
ask the user instead of expanding scope.

**This round is documentation only.** Do not implement the feature now; explain the
plan and ask the user for approval before writing any code.

## Constraints
- Preview only: the function has no side effects and returns a plain result object.
- Columns: `title` is required and `priority` is optional. Any other column name, a
  missing `title`, a duplicated column, a malformed row (wrong number of fields or
  broken quoting) or an empty CSV with no header returns `{"error":"unknown-column"}`.
- Values are trimmed of surrounding whitespace. An empty title after trimming returns
  `{"error":"empty-title"}`.
- Priorities are `low`, `normal` and `urgent`. A missing column or blank value becomes
  `normal`; any other value returns `{"error":"invalid-priority"}`.
- At most 100 data rows are accepted. The row count is checked before any row values
  are validated, so 101 or more rows return `{"error":"row-limit"}` even if a row is
  also invalid.
- Standard CSV quoting must be respected, so a quoted title may contain commas.
- Success is `{"tickets":[{"title":"...","priority":"normal"}]}` in input order.
  Failure is `{"error":"<code>"}`. There is no partial success: one bad row fails the
  whole preview.
- Use only Node.js built-ins (Node 22+, ES modules); add no dependencies.

## Plan
1. Ask the user for approval before starting implementation in a future round.
2. Write `tests/import-tickets.test.mjs` from the acceptance cases below, and confirm
   the new tests fail before the module exists.
3. In `src/import-tickets.mjs`, write a small CSV parser that handles quoted fields,
   escaped double quotes, and `\n` or `\r\n` line endings, ignoring a final newline.
4. Validate the header (empty input, unknown, missing or duplicate columns), then
   check the data-row count against the 100-row limit.
5. Validate each row in order: field count, trimmed title, priority default and
   allowed values. Build tickets with the existing `ticket()` helper.
6. Return the full tickets list, or the first error code found, without side effects.
7. Run the verification commands, then show the user the diff before any commit.

```json
{
  "schema": 1,
  "change": "ticket-csv-preview",
  "allowed_changes": ["src/import-tickets.mjs", "tests/import-tickets.test.mjs"],
  "protected_paths": ["src/tickets.mjs", ".github/", "CLAUDE.md"],
  "non_goals": ["ui", "database", "network"],
  "requirements": {
    "columns": ["title", "priority"],
    "default_priority": "normal",
    "valid_priorities": ["low", "normal", "urgent"],
    "reject_unknown_columns": true,
    "reject_empty_title": true,
    "max_rows": 100,
    "side_effects": "preview-only"
  },
  "verification": ["npm test", "git diff --check"],
  "approval": "ask-before-implementation",
  "acceptance_cases": [
    {
      "csv": "title,priority\nPrinter issue,urgent\n",
      "expected": {"tickets":[{"title":"Printer issue","priority":"urgent"}]}
    },
    {
      "csv": "title\nReset password\n",
      "expected": {"tickets":[{"title":"Reset password","priority":"normal"}]}
    },
    {
      "csv": "title,priority\nVPN is down,\n",
      "expected": {"tickets":[{"title":"VPN is down","priority":"normal"}]}
    },
    {
      "csv": "title,priority\n\"Question about billing, October\",low\n",
      "expected": {"tickets":[{"title":"Question about billing, October","priority":"low"}]}
    },
    {
      "csv": "title,priority\n   ,urgent\n",
      "expected": {"error":"empty-title"}
    },
    {
      "csv": "title,owner\nPrinter issue,Sam\n",
      "expected": {"error":"unknown-column"}
    },
    {
      "csv": "",
      "expected": {"error":"unknown-column"}
    },
    {
      "csv": "title,priority\nPrinter issue,critical\n",
      "expected": {"error":"invalid-priority"}
    },
    {
      "csv": "title\nPrinter issue\n",
      "repeat": 101,
      "expected": {"error":"row-limit"}
    }
  ]
}
```

## Acceptance
The work is complete when every case in the JSON contract above holds:
- Valid explicit priority: `Printer issue,urgent` previews as one urgent ticket.
- Default priority: a CSV with only a `title` column, or a blank priority value,
  previews with priority `normal`.
- Quoted title: `"Question about billing, October"` stays one title with its comma.
- Empty title: a whitespace-only title returns `empty-title`.
- Unknown column: an `owner` column, or an empty CSV, returns `unknown-column`.
- Invalid priority: `critical` returns `invalid-priority`.
- Too many rows: 101 data rows return `row-limit` before any values are checked.

No input produces partial results, and the existing `src/tickets.mjs` tests still pass
unchanged.

## Verification
Run `npm test` and confirm every existing and new test passes, then run
`git diff --check` to confirm there are no whitespace errors. Check that `git status`
shows changes only to `src/import-tickets.mjs` and `tests/import-tickets.test.mjs`.
Ask the user for approval before implementing, before touching any other file or
expanding scope, and before committing, pushing or opening a pull request.
