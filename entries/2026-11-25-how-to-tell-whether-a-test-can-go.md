# How to Tell Whether a Test Can Go

*2026-11-25 · coding agents · test design · repository maintenance*

Suppose you maintain a tool that checks links in a reading list. A checked link can be reported missing. A link whose request timed out needs another look. Both outcomes can leave a blank title in the source response. The distinction matters to the reader deciding which entries to remove.

The first test for that report replaces the writer, checks that it was called once, and expects a 201 response. It was a reasonable first test. An engineer adding the endpoint wanted a quick way to see that a valid request reached the writer. By the time an Agent proposes retiring that test, the tool has a broader report check. That gives us a question we can answer by running code. **With the old check gone, what would tell us that a timed-out link was being reported missing?**

I use a synthetic link-reporting tool below. It goes through a test cleanup and then a decision about removing a documentation directory. In both cases the decision depends on what still uses the file and what we can observe after it goes away.

## Start with a failure someone could see

“Writing the report calls this helper once” is a description of the implementation. “A timeout is printed as a confirmed missing link” describes a failure. A refactor may change the first sentence while the report still works. The second gives a reviewer a particular input and a line to look for in the output.

For the proposed deletion, I would list the distinct failures that the old test can catch. A client that depends on the endpoint receiving a 201 still needs that contract checked. Whether a particular writer function received one call matters if a second call could create a second report. In that case it is possible to check how many reports were published. The output gives the reviewer a way to tell a duplicate side effect from a harmless change in function calls.

A test that only covers retired behavior can leave with its runner entry and fixtures. When two tests protect the same outcome, keep the check that reaches it with the fewest moving parts. When a test is the only check for a supported failure, the proposed replacement has to show it can catch that failure. A successful run of the new test is the starting point.

## Give the replacement a wrong result

Here is a 30-line version of the report check. Save it as `link-report.test.ts` and run `node --test link-report.test.ts` with Node 24. In this example, `POST /link-reports` accepts the outcome of a link check. The handler maps it to a status and writes one CSV row. Both tests call that handler in process.

```ts
import assert from "node:assert/strict";
import fs from "node:fs";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { test } from "node:test";
async function postLinkReport(request: Request, output: string) {
	const status = (await request.text()) === "checked-missing" ? "missing" : "unknown";
	fs.writeFileSync(output, `status\n${status}\n`);
	return new Response(null, { status: 201 });
}
const request = () => new Request("http://example.test/link-reports", {
	method: "POST", body: "timeout",
});
test("Before checks status and writer call", async (t) => {
	const write = t.mock.method(fs, "writeFileSync", () => {});
	const response = await postLinkReport(request(), "unused.csv");
	assert.equal(response.status, 201);
	assert.equal(write.mock.callCount(), 1);
});
test("After keeps unknown in the report", async (t) => {
	const dir = fs.mkdtempSync(join(tmpdir(), "link-report-"));
	const output = join(dir, "link-report.csv");
	t.after(() => {
		fs.rmSync(output, { force: true });
		fs.rmdirSync(dir);
	});
	const response = await postLinkReport(request(), output);
	assert.equal(fs.readFileSync(output, "utf8"), "status\nunknown\n");
	assert.equal(response.status, 201);
});
```

`Before` replaces the writer and checks for a 201 and one call. `After` opens the real `link-report.csv` before checking the response. The input to both is `timeout`. The expected report value is `unknown`. The plain expected string in the test is the business rule for this input.

Run `node --test link-report.test.ts` first. Both tests pass. Change the `unknown` branch to `missing` and run the same command. `Before` still sees a 201 and one call to the writer. `After` fails on the bytes read from `link-report.csv`. Restore the branch before trying the deletion.

This exercise is worth doing on the smallest example that can expose the problem. A test may run a real browser, receive a terminal event, and download a file, yet accept the wrong deliverable if its assertion is “some file was downloaded.” If the user asked for links that were checked and found missing, open the delivered file and distinguish those entries from links that timed out.

## Read the assertion that failed

Here is a reproducible review of an Agent's proposed deletion. Remove `Before` from `link-report.test.ts`. The Agent changes the handler's response from `201` to `500`, runs the default command, and reports that the remaining check detects a bad report. The run is red and the failing test is named **After keeps unknown in the report**.

Open the assertion. The file check has already passed, and the failure is `500 !== 201`. The report still contains `unknown`. The change exercised the HTTP contract. To test the claim about the report, restore the response status and change the timeout classification while leaving the request and assertions in place.

I ran the retained test in these three states using `node --test link-report.test.ts`.

| Change in the handler | What the retained test reports |
| --- | --- |
| Return `500` instead of `201` | The file assertion passes. The response assertion fails with `500 !== 201`. |
| Write `missing` for `timeout` | The file assertion fails. The report contains `missing` where the test expects `unknown`. |
| Restore the handler | The test passes after reading `unknown` from the generated CSV and checking the `201` response. |

Both deliberate changes make the same test red. Only the file assertion establishes the claim about classification. When an Agent supplies a proof run, ask for the application line it changed and the assertion that failed.

The runner is another place a check can get lost. Suppose `npm test` runs a list of test files in the example project. You move the CSV assertion into another file and delete the old entry. Verify that `npm test` reaches the new assertion with the old entry out. Running the retained file by hand proves what is in that file. The normal command proves whether that check is part of the team's verification.

## Give the Agent a rule it can execute

We can write the rule in a root `AGENTS.md` or a verification guide. This version assumes the repository's default verification command is `npm test`. In a repository with a different entry point, use its actual command in both runs.

```markdown
## Deleting or rewriting tests

For each supported behavior affected by a test deletion or rewrite, the
Agent must complete these steps with the proposed test changes in place.

1. State the concrete user failure, the input that reaches it, and the
   expected result.
2. Introduce that failure temporarily in the application code. Run
   `npm test`. The retained check must fail on the stated behavior.
3. Restore the deliberate change. Run `npm test` again and confirm
   that it passes with the proposed test changes still in place.
4. Inspect the actual result from the passing run. Open the delivered
   file, read back the saved record, or inspect the public return value.
   Record which assertion failed and what result you verified.
```

A single demonstration can cover several assertions being removed for the same failure. If a type or a guard takes over a rule, show that the old bad input gets rejected at the entry point and that valid input still works. In the link tool, a guard can reject an unrecognized outcome before the writer sees it. The report check has a separate job. It can show that the handler writes the right row for an outcome the guard accepts.

The inputs come from the supported user story. If you change the account, entry point, or expected outcome to make the check easier to pass, the new run can answer a different question. In `link-report.test.ts`, switching the input from `timeout` to `checked-missing` would give a valid test of the other branch and leave the original failure untested.

## Find out who reads a document before deleting it

Consider the same example project after a few rounds of work. It has a `docs/` directory containing old design notes, an active task list, and commands copied from the README. It also has `docs/help.txt`. A command-line tool reads that file to produce its help output.

These files have different readers. A person investigating an old decision can recover the note from Git. Open work can go into the issue tracker. Current setup instructions can live in the README, while `package.json` specifies what `npm test` actually runs. With those destinations in place, a new contributor has a short path to current work. Someone investigating the old design needs to open the relevant revision.

There is one more check before the entire directory can go. Moving `docs/help.txt` to `assets/help.txt` requires updating the command that reads it. In this example the check is `node cli.mjs --help`. Read the output and look for the documented `--report-dir` flag. A command can return successfully while the help text is incomplete. Inspecting that output tests the dependency that the deleted directory used to satisfy.

This is how I would review the second deletion. Follow each current reader to the place where it gets its information after the cut. For an executable consumer, run it and look at the result. For a historical note, check that the old version can be retrieved with `git log -- docs/` and `git show <revision>:docs/help.txt` when that is the file you need. The two readers require different evidence.

## Use the smallest check that can see the failure

The report check writes a file because the failure is in what a user reads from a file. Cheaper checks have useful work elsewhere. In this tool I would give each one a failure it can observe directly.

| Failure | Where to check it |
| --- | --- |
| A request supplies an invalid outcome, or a count can violate a stable invariant. | Reject invalid input with a guard at the request boundary. For an integer rule such as checked entries plus pending entries equaling the input size, a formal proof in a tool such as Lean can establish the relation over the modeled inputs. |
| A valid input maps to the wrong status. | Test the pure rule directly. A fast unit test can exercise `timeout`, `checked-missing`, and any other valid outcomes without creating a file for each case. |
| The rule returns `unknown`, but the delivered row says `missing` or appears in the wrong place. | Run the handler and writer together, then reopen the delivered file. The `After` check shows how to do this for a single row. |

Keep the fast unit tests for distinct branches such as `timeout` and `checked-missing`. Repeating every branch through a browser, upload, and download would increase the amount of work per run. Use the slower path for failures it can expose, such as a report containing the right status on the wrong row. The unit test for the classifier can be green while that failure reaches a reader.

## Leave proof the next Agent can read

I would ask for a review note with a particular input and a result someone else can try to break.

> With the old writer check removed, `npm test` still fails when a timeout is written as a missing link. The assertion opens `link-report.csv` and reads the status row.

The note gives a new reviewer a command, a failure to introduce, and a file to inspect. If the run stays green after changing the output to `missing`, there is a reason to pause the deletion. If the assertion fails on that row and passes after restoration, the reviewer can see where the behavior is checked in the resulting repository.

I asked a related question in an earlier entry about what an Agent can pick up from a previous run. A deletion review can leave that kind of evidence. The next Agent can reproduce the failure from the current command and inspect the result the user would have received. [An Agent Run Ends. The Work Does Not.](https://github.com/pengpengkuku/agent-harness-diary/blob/main/entries/2026-11-03-an-agent-run-ends-the-work-does-not.md)
