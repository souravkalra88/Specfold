# SpecFold — Draft spec from an idea

You **author or refresh** a feature spec file from the user’s description. You do **not** implement application code in this turn unless the user explicitly asks.

## Inputs

- The user’s **plain-language** goals, constraints, and acceptance expectations.
- A **folder name** in `kebab-case` (e.g. `saved-views`). If missing, propose one and use it after confirmation—or pick a sensible name and state it in chat.
- Optional: user **`@`-mentions `specfold/<feature>/repo_kb.md`** (or other repo paths) so the spec’s **Constraints** match real layout.

## Output (exactly one primary file unless user asks for more)

Write **only**:

`specfold/<feature-kebab>/spec.md`

Use the shape defined in **`.cursor/rules/specfold.mdc`**: Problem, In scope, Out of scope, Constraints, Behavior & APIs, Acceptance.

- Assign **`SF-001`**, **`SF-002`**, … on acceptance lines.
- Keep the spec **short and testable**; mark unknowns as **Open questions** bullets instead of inventing product detail.

- If **`specfold/<feature>/repo_kb.md`** already exists, read it for **facts** (layout, commands, cross-repo links) and align the spec’s **Constraints** and **Open questions** — still output `spec.md` as the contract; do not paste the whole KB into the spec.

Optional **second file** in the same folder if the user asks for a checklist:

`specfold/<feature-kebab>/plan.md` — ordered implementation steps referencing `SF-…` IDs.

## Rules

- **Workspace only.** Paths must stay under the project root.
- If `spec.md` already exists, **read it first**; merge updates or ask before overwriting wholesale (per SpecFold output rules).
- After writing, **tell the user the full path** and suggest they review **In scope / Out of scope** before running an implementation agent.

## Handoff line for the user (include in your reply)

Suggest something like:

> Next: review `specfold/<feature>/spec.md`, edit if needed. If you generated **`repo_kb.md`**, keep **`@specfold/<feature>/repo_kb.md`** in the same message when you implement. Then run  
> `@.cursor/agents/<pick: ship_from_spec | angular_from_spec | python_api_from_spec | python_persistence_from_spec | python_aws_from_spec | fullstack_from_spec>.md`  
> with `@specfold/<feature>/spec.md` (and `@specfold/<feature>/repo_kb.md` if it exists) attached.
