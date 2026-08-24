---
name: greeter
description: Greet the user by collecting their details and delegating to the greeter recipe
---

# Greeter

When asked to greet the user, follow these steps:

## Step 1: Read the Recipe Parameters

Read the recipe file at `~/.config/goose/recipes/greeter.yaml` and parse the
`parameters` array. For each entry note the `key`, `input_type`, and `options`
(if present).

## Step 2: Ask for Each Required Parameter One at a Time

For each parameter where `requirement: required`, ask one question and wait for
the user's answer before asking the next:
- `input_type: string` → ask for a free-text value
- `input_type: select` → present the `options` as a numbered list, wait for choice

## Step 3: Delegate to the Greeter Recipe

Once all values are collected, delegate to the **greeter** recipe, passing the
collected values. The recipe will return a greeting string as its response.

## Step 4: Display the Greeting

Show the greeting text returned by the recipe to the user.
