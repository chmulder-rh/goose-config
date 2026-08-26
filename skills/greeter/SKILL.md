---
name: greeter
description: Greet the user by piping a name-parsing recipe into the greeter recipe via shell, proving the recipe-chaining-via-shell pattern
---

# Greeter (Shell-Piped Proof of Concept)

This skill demonstrates chaining two recipes together deterministically using
shell commands, rather than relying on the LLM to reason about a previous
recipe's output. This is the pattern described in
`docs/guides/recipes/session-recipes.md` ("Structured Output for Automation").

**Note:** This pattern requires goose to be in `auto` mode
(`GOOSE_MODE: auto` / `/mode auto`) — nested `goose run` calls launched from a
`shell` tool call fail in `approve`/`smart_approve` mode.

Use the Todo extension to track progress through the steps below: create a
checklist at the start covering all 5 steps, and check items off as they
complete. This gives a clear, inspectable record of which stage the skill is
on at any point.

When asked to greet the user, follow these steps:

## Step 1: Ask for the User's Full Name

Ask the user for their full name as a single free-text answer (e.g. "Charles Mulder").
Wait for their answer before continuing.

## Step 2: Ask for Time of Day

Ask the user to choose one:
1. Good morning
2. Good afternoon
3. Good evening

Wait for their answer before continuing.

## Step 3: Run the Name Parser Recipe via Shell

Using the shell tool, run the `greeter-name-parser` recipe non-interactively and
extract its structured JSON output. Use `-q` for quiet output and `tail -n 1` to
grab the final structured-output line, per the documented automation pattern:

```bash
goose run --recipe /home/chmulder/.config/goose/recipes/greeter-name-parser.yaml \
  --params raw_name="<full name from Step 1>" \
  -q | tail -n 1
```

Parse the resulting JSON line (it will have `first_name` and `last_name` keys).
Do NOT reason about or reconstruct the name split yourself — use only the values
from this JSON output. This is the deterministic hand-off step.

## Step 4: Run the Greeter Recipe via Shell, Using the Parsed Values

Take the `first_name` and `last_name` values extracted in Step 3, plus the
`time_of_day` answer from Step 2, and pass them as `--params` to the greeter
recipe:

```bash
goose run --recipe /home/chmulder/.config/goose/recipes/greeter.yaml \
  --params first_name="<first_name from Step 3>" \
  --params last_name="<last_name from Step 3>" \
  --params time_of_day="<time_of_day from Step 2>" \
  -q
```

## Step 5: Display the Greeting

Show the final greeting text (the output of Step 4) to the user as-is.
