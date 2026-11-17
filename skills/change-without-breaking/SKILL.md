---
name: change-without-breaking
description: >-
  Load BEFORE editing anything that already works: a bug fix, a review-and-fix, a refactor, a prompt
  or rule change, a feature in an existing project, a live app, bot, scheduled job, migration or
  customer data. Keeps everything that worked still working, with proof: checkpoint and do-not-change
  list, find every user of what changes, pin current behavior, smallest change, look-alike inputs that
  must NOT trigger it, never weaken a test, before/after diff, independent review; for live systems,
  loop and quiet-hour checks, read-only migration preview, health check, real test, rollback. Triggers:
  "fix", "don't break anything", "refactor", "review and fix", "deploy", "migration", "production",
  and in Spanish "arregla", "no rompas nada", "sin romper", "revisa y arregla", "sube la versión".
license: MIT
---

# Change without breaking

An agent asked to fix one thing often breaks two others. The fix itself is usually right. What goes wrong is everything around it: a caller nobody searched for, a rule that now fires on the wrong input, a test quietly loosened until it passed, a job that resends forever, a migration that touched more rows than anyone checked.

Real cases this skill was built from, all in one week of fixing live apps:
- **A "ready to buy" rule** treated questions like "how long does one box last?" as a purchase decision.
- **A health-risk detector** blocked a sale because the customer wrote "muscle mass".
- **A new reminder** was about to resend every 15 minutes, forever.

Each one passed its own happy-path test.

The goal of this skill: **everything that worked before the change still works after it, and you can prove it.**

Companions: `blast-radius` digs deeper into what one diff can break, `working-with-legacy-code` covers untested code, and `shipping-and-launch` covers staged rollouts.

## 0. Size it
- **Tiny and local** (a typo, a label, one line nobody else calls): do steps 1, 4 and 7, and run the tests that exist.
- **Everything else:** all steps. The more people or data it touches, the more of step 9 applies.

## 1. Do-not-change list and checkpoint
Before the first edit:
- Write 3-6 lines:
  - what you are changing (one thing);
  - what must stay exactly the same;
  - how you go back.
- Make a checkpoint you can return to in one command: note the current commit (`git rev-parse HEAD`) or save a stash ref (`git stash create`), or keep a copy of the one config you are editing. Don't commit on the user's behalf unless they allow it.
- If the working tree already has someone else's uncommitted changes, don't mix them with yours. Stop and say so.

## 2. Find everything that uses what you touch
- Search every reader and caller of the function, field, file, setting, prompt section or data format you will change: other modules, scripts, scheduled jobs, other languages and services, dashboards, saved data.
- A symbol search is not enough. Also look for:
  - string keys;
  - JSON fields a client reads;
  - column names;
  - environment variables;
  - text that another rule matches on.
- List what you found. If something reads it and you won't test it, say so in the report.

## 3. Pin today's behavior
- Run the existing tests BEFORE changing anything. Write down what was already failing, so you don't blame your change for it, or hide behind it.
- No test covers the part you are touching? Write a **characterization test** first: it records what the code does today, right or wrong.
- **For bots and prompts:** replay real samples, meaning real conversations taken from the app's logs or backup. Anonymize them first: names become "Customer 1", phone numbers become 5550000001, emails become `customer1@example.test`, and drop sensitive data you don't need. Never commit them or share them outside the project. Save the current outputs as the "before" snapshot.

## 4. Make the smallest change
- One fix at a time. If something breaks, you know which change did it.
- No drive-by refactors, renames, reformatting or dependency bumps in the same change.
- Keep public interfaces, defaults, ordering, error behavior and data formats the same unless they are the thing being fixed.

## 5. Look-alike tests: what must NOT change
For every rule, condition, detector, regex, keyword or prompt instruction you add or touch, write:
- **5 inputs that SHOULD trigger it;**
- **5 look-alikes that must NOT trigger it:**
  - **questions:** "how long does one box last?" is not "I'll take it";
  - **negations:** "not yet";
  - **the word inside another meaning:** "muscle mass" is not a health problem;
  - **other tenses:** "I bought it before";
  - **edge values:** an empty string, zero, null, the boundary number;
  - **typos and voice-dictated text.**

If a must-not input fires, the change is wrong. Fix the change, not the test.

## 6. Tests are never weakened
- Don't delete a test, skip it, drop an assert, widen a tolerance or rewrite the expected value just to get green.
- If an old test really demands the wrong behavior, change it and write the reason next to it: date, what was wrong, and who asked for the new behavior.
- A test that fails after your change is information. Read it before touching it.

## 7. Diff before vs after
- Run the full test suite, plus the characterization tests or replayed samples from step 3, on the new version.
- Count how many outputs changed, and explain each one:
  - **expected:** it is the fix;
  - **unexpected:** stop and look.
- An unexpected change in something you did not mean to touch is a break, even if the new output "looks fine". Fix it, or go back to the checkpoint.

## 8. Independent review
For anything beyond tiny, a separate reviewer runs the real code before it ships: another agent (ideally a stronger model) or a person. The reviewer:
- gets the do-not-change list, the diff and the test command, and nothing else: not your conclusion;
- runs the tests and the look-alike inputs.

Ship only on a "pass" that says what was run.

## 9. Live systems: extra checks
Apply the parts that fit.

**Automated messages and jobs** (reminders, follow-ups, notifications, cron), tested with a simulated clock:
- **Loop:** run the job 10 times in a row for the same contact. It must act **once**. That needs a stored "done" mark, checked before acting.
- **Two workers at once:** reserve the item (write the mark, or `SELECT ... FOR UPDATE SKIP LOCKED` inside a transaction, on Postgres or MySQL 8) before acting, so two instances or a retry don't double-send.
- **Quiet hours:** use the **recipient's** time zone, not the server's. Test both edges, for example 8:59 and 20:01.
- **Messaging rules:** for example, WhatsApp's 24-hour window means free text only within 24 h of the customer's last message, and only approved templates after that; test 23 h and 25 h. Never message anyone who opted out.
- **Rollout:** a per-contact cap, a flag that turns it off without a deploy, and shadow mode first. In shadow mode it only logs who it WOULD message, someone checks that list, then it goes on for a test contact, then for everyone.

**Data changes** (migrations, backfills, cleanups, scripts on production data):
- First run the **read-only query** with the same filter (`SELECT` / `count`) and see how many rows, and which ones, it will touch. More than expected means stop.
- Pick backfill values so a new job does not fire on the whole history the moment it ships.
- Prefer a versioned, additive migration over code that runs on every startup.
- Take a snapshot of the key counts before and after: contacts, orders, bookings, and the records that must not change. Touch no more data until they match.

**Deploy and prove:**
1. Deploy at the quietest hour, with someone able to watch.
2. Check health: the app answers, and the logs show no new errors in the first minutes.
3. Run one real end-to-end test from a test account, never a customer's real number or account.
4. Something off? Roll back first, debug second.

## Never
- Say "done" without the step 7 diff (and step 9's real test for live systems).
- Ship two fixes together.
- Weaken a test, remove a validation or silence an error to make something pass.
- Run a data change that skipped its read-only preview.
- Test on real customers.

## Report (one screen)
- **Changed:** one line, plus the commit.
- **Checkpoint:** how to go back.
- **Who uses it:** what you found in step 2, and anything you did not test.
- **Before vs after:** tests and samples run, outputs changed, expected, unexpected.
- **Look-alike tests:** how many, and that none of the must-not inputs fired.
- **Reviewer:** what was run, and the verdict.
- **Live systems:** loop, hours, data counts before and after, health, real test, each with its result or N/A.
