# How to Tell Whether a Test Can Go

*2026-11-25 · coding agents · test design · repository maintenance*

Suppose a document tool lets you press Escape to stop a preview. During an ordinary key press, it should request a stop once. When an input method is still composing text, the preview should keep running. A short, standalone test made that distinction easy to see when the shortcut was added.

Later, a broader interface test could exercise the same keyboard behavior. We could move the cases there and take the standalone script out of the default run. The same question comes up when you remove a whole documentation directory. **Once this file is gone, how can I verify that the work it was doing still gets done?**

The examples here use a composite workbook service. For a test, I start with a failure the user could observe. With the old check out, I make that failure happen once and read the assertion that fails. After restoring the code, I run the usual command and inspect what it produced.

## Start with the failure, then decide what to keep

The Escape case is easy to merge because the behavior is precise. Use the same handler to exercise an ordinary key press, an input method composition, and a repeated key press. Server-side cancellation has its own job. A stop request should wait for its execution to settle while a second job can continue. The keyboard handler can work while either server guarantee breaks.

A different kind of check searches the route's source for function calls in a particular order. Moving those calls into a helper can trip the assertion even if cancellation still works. Before dropping it, look for the check that exercises the stop and handoff. A mandatory example for every capability has a related problem. An example still earns its place when it evaluates a concrete decision or failure. An empty slot in a list tells you little about whether the application works.

If you are considering a deletion, try describing the failure without naming a helper, a line of source, or a call count. “Escape cancels a preview while I am composing text” tells you which input to send and which behavior to observe. “The handler ran twice” needs a further question. What would the caller see if that count changed?

Run the affected checks after the cut and review which failures they can still catch. Keep the full run and any focused rerun distinct. If a full run reports a permission error during temporary-directory cleanup and the affected test later passes on its own, record both observations.

## Give the replacement a wrong result

A workbook test can expose a gap an interface check cannot see. Imagine a tool that looks up a list of identifiers in an external catalogue and fills in the original workbook. An early test supplies a stand-in for the lookup handler. The stand-in first says authentication is required, then reports a completed job. That is a reasonable way to exercise handoff. It leaves the lookup handler, the filled workbook, and the saved result to other checks.

Build the replacement from a workbook you can read independently. In one illustrative fixture, six positions include one blank and one duplicate, so the expected request contains four distinct lookups. Control the browser operations and source responses, and run the service's own session handling, lookup tool, workbook writer, and storage operations.

Open the delivered workbook. A lookup the source checked and found missing should produce a different cell value from a lookup the source failed to verify. One is a result the reader can act on. The other needs checking. Read the cell from the file, check that existing formulas and formatting survived, and read back the saved records after the user explicitly asks to keep the results.

Then change one thing at a time. Skip filling the workbook and see whether the test fails on the target cells. Break resumption and see whether it catches the switch away from the original job. Misclassify a failed lookup and read the wrong cell value. Skip saving and check whether the test reaches the missing record. Restore each change before trying the next. The replacement can be longer than the checks it takes over. Here the added work is opening the file and reading the saved result.

### A 30-line example you can run

Save this as `report.test.ts` and run `node --test report.test.ts` with Node 24. Both tests call the same request handler in process. `Before` replaces the writer and checks a 201 response and one call. `After` opens the actual `report.csv` before checking the response.

```ts
import assert from "node:assert/strict";
import fs from "node:fs";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { test } from "node:test";
async function postReport(request: Request, output: string) {
	const status = (await request.text()) === "verified-empty" ? "not_found" : "unverified";
	fs.writeFileSync(output, `status\n${status}\n`);
	return new Response(null, { status: 201 });
}
const request = () => new Request("http://example.test/reports", {
	method: "POST", body: "source-error",
});
test("Before checks status and writer call", async (t) => {
	const write = t.mock.method(fs, "writeFileSync", () => {});
	const response = await postReport(request(), "unused.csv");
	assert.equal(response.status, 201);
	assert.equal(write.mock.callCount(), 1);
});
test("After keeps unverified in the report", async (t) => {
	const dir = fs.mkdtempSync(join(tmpdir(), "report-example-"));
	const output = join(dir, "report.csv");
	t.after(() => {
		fs.rmSync(output, { force: true });
		fs.rmdirSync(dir);
	});
	const response = await postReport(request(), output);
	assert.equal(fs.readFileSync(output, "utf8"), "status\nunverified\n");
	assert.equal(response.status, 201);
});
```

Both tests pass as written. Change the `unverified` branch to `not_found` and run `node --test report.test.ts` again. `Before` still sees a 201 and one call to the writer. `After` fails because the file says `not_found` where it should say `unverified`. Restore the branch before continuing.

The same question applies to a test that joins two workbooks. Start from authenticated upload, run the registered tools, download the result, and reopen it. A stand-in that returns a completed receipt can miss a failure in publication or a wrong cell after the join. An ownership probe also has to reach the ownership decision. If the request fails before it reaches that point, set the probe aside and keep its output with the review notes.

## Read the assertion that failed

Here is a reproducible review of an Agent's proposed deletion. Remove `Before` from `report.test.ts`. To show that `After` catches a bad report, the Agent changes the response from `201` to `500`. The normal command reports `not ok` beside **After keeps unverified in the report**. The test name and exit code make the evidence look plausible.

Read the assertions in order. The CSV check has passed. The failure says `500 !== 201`, and the report still contains `unverified`. The run tells us about the HTTP contract. Restore the status and change the classification while keeping the request and assertions in place. The failure now shows the incorrect CSV value.

I ran the retained test in these three states with `node --test report.test.ts`.

| Deliberate change | What the retained test reported |
| --- | --- |
| Return `500` instead of `201` | The CSV assertion passed. The response assertion failed with `500 !== 201`. |
| Write `not_found` for `source-error` | The CSV assertion failed. The file contained `not_found` where it expected `unverified`. |
| Restore the handler | The test passed after reading `unverified` from the CSV and checking the `201` response. |

Both changes turn the same test red. The assertion that reads the CSV is the evidence for the classification claim. When an Agent brings back a proof run, ask for the line it changed and the assertion that failed.

The runner is another place a check can get lost. Suppose you move two report assertions into an existing route test. One verifies that the requesting account gets its own file. The other reads the delivered CSV. With the old test entry removed, follow `npm test` to the retained assertions. A focused run of the route test establishes what that file checks. The default command establishes whether the team actually runs it.

## Put the rule where the Agent will find it

The first version of the verification guide would tell an Agent to start at the user's entry point, run the operations that produce the result, and get the expected answer from the business rule or original material. Shared protections belong at the boundary where they apply. Verify that the affected check catches the stated error and that the normal runner reaches it.

A later revision needs to handle a familiar mistake. Say the user needs an existing report address to keep working. A green check at a newly added address answers a question about that address. Put the established entry point, acting account, object, and expected result into the test story before changing the implementation. The example starts with `source-error` for the same reason. Changing the input to `verified-empty` gives you a different test.

The next revision can make room for a rule enforced by the program. If a type or guard takes over an existing responsibility, send the old invalid input through the actual entry point and verify that it is rejected. Then send valid input through the same path. The reviewer can now decide whether the old assertion duplicates that protection.

Here is the rule I would put in a root `AGENTS.md` or verification guide. This version uses `npm test` as the example project's default command.

```markdown
## Deleting or rewriting tests

For each supported behavior affected by a test deletion or rewrite, the
Agent must complete these steps with the proposed test changes in place.

1. State the concrete user failure, the input that reaches it, and the
   expected result.
2. Temporarily introduce that failure in the application code. Run
   `npm test`. The retained check must fail on the stated behavior.
3. Restore the deliberate change. Run `npm test` again and confirm that
   it passes with the proposed test changes still in place.
4. Inspect the actual result from the passing run. Open the delivered
   file, read back the saved record, or inspect the public return value.
   Record the failing assertion and the result you verified.
```

Demonstrate a shared failure once against the final test set when several removed assertions covered it. For a test that covers retired behavior, name the behavior and check that its fixture and runner entry can go with it.

## Find out who reads a document before deleting it

Now consider the second deletion. The example project's `docs/` directory contains a verification guide, architecture notes, task lists, and old release records. The directory can go once the work those files did has a place to happen.

The old notes explained decisions. The task lists carried open work across sessions. The guide made testing rules explicit while we were working them out. A reader can recover an old decision with the corresponding change in Git. Open work can live in the issue tracker. The README can hold current setup steps, `AGENTS.md` can hold working rules, and `package.json` can show what `npm test` invokes. Someone researching an old design now has to open the relevant revision. Someone starting current work can go straight to the entry point.

A program may also read a document. Suppose a tool that builds an example bundle reads `docs/bundle-guide.md`. To remove the directory, move that input to `assets/bundle-guide.md` and change the builder to read it there. Run the builder and open the resulting archive. Check that the public guide has the expected content and that its raw template stays out of the bundle. The builder can exit successfully even when the bundle is missing the guide.

That is the check for this second deletion. For a current program reader, run the program and inspect its output. For a historical reader, check that Git can still retrieve the old file. A file count can describe the size of the cut. It cannot verify that the bundle still works.

## Use the smallest check that can see the failure

Different checks can own different parts of one report. A type or an input guard can reject a malformed outcome at the request boundary. A formal proof can cover a stable arithmetic rule such as checked rows plus pending rows equaling the input size. A report test can establish which rows the application actually counted and what appeared in the delivered file.

| Failure | Where to check it |
| --- | --- |
| The request supplies an unrecognized outcome, or a count violates a fixed relationship. | Validate the input at the request boundary. Use a tool such as Lean to prove the arithmetic rule over the modeled inputs if it warrants a proof. |
| A valid source outcome maps to the wrong status. | Keep a fast unit test of the rule. `source-error` should give `unverified`. `verified-empty` should give `not_found`. |
| The classifier returns `unverified`, but the writer puts `not_found` in the file or writes to the wrong row. | Run the handler and writer together, then open the delivered result. The `After` check does this for one CSV row. |

Keep the unit cases that distinguish decisions. Uploading a workbook and opening a browser for each classifier branch would cost time in every run. Use the full path when the failure depends on something crossing a boundary, such as a valid status reaching the wrong cell. The classifier can be entirely correct while the workbook is still wrong.

## What I would ask for in a deletion review

Start with one sentence the next reviewer can try to disprove.

> With the old writer check removed, `npm test` fails if an unverified lookup is delivered as a confirmed absence. The assertion reads the affected row from the generated report.

The reviewer has a command, a fault to introduce, and a file to inspect. For a document that a program reads, name the consumer and the output instead. In the bundle example, that means running the builder and opening the archive after the old path has gone.

In an earlier entry, I asked what a new Agent would need to continue a piece of work without reconstructing the previous run. A deletion review can leave evidence of the same kind. The next Agent can run the current command, reproduce a specific failure, and see the result the user would have received. [An Agent Run Ends. The Work Does Not.](https://github.com/pengpengkuku/agent-harness-diary/blob/main/entries/2026-11-03-an-agent-run-ends-the-work-does-not.md)
