# change-without-breaking

**Make your agent fix the one thing you asked, and leave everything that already worked still working. With proof.**

`change-without-breaking` is an agent skill for Claude Code and any agent that reads `SKILL.md` skills. It targets a costly agent mistake: a correct fix that quietly breaks something next to it.

Before, during and after any change to something that already works, the agent has to:
1. **Write a do-not-change list and make a checkpoint** it can return to in one command.
2. **Find everything that uses what it touches**, not just the symbol: string keys, JSON fields, columns, env vars, text other rules match on.
3. **Pin today's behavior.** It runs the existing tests first, writes a characterization test if none exist, and for bots and prompts replays anonymized real conversations.
4. **Make the smallest change**, one fix at a time, with no drive-by refactors.
5. **Test look-alikes that must NOT trigger the change.** A rule for "ready to buy" must not fire on "how long does one box last?".
6. **Never weaken a test to get green.** If an old test is wrong, it changes with a written reason.
7. **Diff before vs after.** Every unexpected difference counts as a break, even when it "looks fine".
8. **Get an independent review** that runs the real code.
9. **For live systems:** automated messages are tested for loops, double workers, quiet hours and messaging-window rules, with shadow mode first. Migrations get a read-only preview of the rows they touch and a before/after count. Deploys get a health check, a real end-to-end test and a rollback path.

## Why it exists

It was built from three real breaks in live WhatsApp sales assistants, all in one week:
- **A "ready to buy" rule** fired on questions like "how long does one box last?".
- **A health-risk detector** blocked a sale because the customer wrote "muscle mass".
- **A new follow-up reminder** was about to resend every 15 minutes, forever.

Every one of them passed its own happy-path test.

## Does it work?

An earlier draft (the live-systems part, before the general steps were added) was tested on the same model (Haiku), with and without the skill, on two real-shaped tasks: a "customer decided to buy" rule, and a 2-hour follow-up plus a startup backfill.
- **Without the skill**, the plan was decent but had no before/after diff on real conversations, no independent review, no read-only preview of the rows the backfill touches, and no before/after data snapshot.
- **With the skill**, the plan covered all of them. It also split the work into separate deploys, and it caught that the backfill value decides whether the new job messages the whole history on day one.

The without-skill run had three good ideas the skill lacked: shadow mode, protection against two workers at once, and opt-out exclusions. They are in the skill now.

This is a small sample (one run each) and the skill can't promise zero breaks. What it does is make the agent prove that nothing else changed before it says "done".

**Skip it for** brand-new code that nothing depends on yet.

**Status:** new (November 2026), installed in the author's own multi-agent setup. Plain instructions, no scripts, no network calls. MIT. Issues welcome.

## Install

```bash
npx skills add RabbitDesigns22/change-without-breaking
```

Or copy `skills/change-without-breaking/` into `~/.claude/skills/`. To make every session use it, add one line to your `CLAUDE.md`: "Before editing anything that already works, load the `change-without-breaking` skill."

## Related

- [`blast-radius`](https://skills.sh/backnotprop/pstack/blast-radius) goes deeper on what a single diff can break.
- [`working-with-legacy-code`](https://skills.sh/wondelai/skills/working-with-legacy-code) covers code without tests.
- [`shipping-and-launch`](https://skills.sh/addyosmani/agent-skills/shipping-and-launch) covers staged rollouts.
