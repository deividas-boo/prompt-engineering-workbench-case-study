# Prompt Engineering Workbench

**University of Twente Design Project · Client: Rabobank · Sept–Nov 2026 · Team of 5**

The code belongs to the team and the client, so it isn't public. This page explains
the problem, the solution, my contributions and the results.

## The problem

Bank employees use AI assistants like Microsoft Copilot every day. When writing a
prompt, they have no easy way to check two things: whether it's written well enough
to get a useful answer, and whether it leaks information that shouldn't leave the bank.

The second question is the harder one. The risky content often isn't personal data
but internal knowledge, such as the name of an internal system, an API endpoint or an
architecture choice. That kind of information only looks sensitive in context, so
simple pattern matching can't find it. On top of that, the bank's internal AI usage
guidelines couldn't be shared with the team, so the design had to be built around the
bank's four-level data classification scheme (Low, Moderate, Elevated and High) instead.

## The solution

A web app that checks a prompt before it's sent to an AI assistant. It works in five
steps, with a simple idea behind them: first make the prompt safe, then make it better.

1. **Write.** The employee enters a prompt and picks a role, such as engineer or
   business analyst. The role decides which quality advice they get later.
2. **Mask personal data.** Names, IBANs, citizen service numbers and similar data are
   found automatically and replaced with placeholders such as `<PERSON>`. This step
   only appears when something is found.
3. **Security review.** A language model looks for internal information that patterns
   can't catch, places each finding on one of the four levels, explains why it's a
   problem and suggests a safer rewrite.
4. **Quality score.** 28 rule-based checks score the prompt from 0 to 100 on task,
   clarity, context, constraints and output, and suggest concrete additions.
5. **Improve.** The model writes improved versions side by side. Picking one sends it
   through the security steps again, so new text is always checked before it's saved.

### Architecture

```
Browser ──▶ React frontend ──▶ FastAPI backend ──▶ SQLite (prompt history)
                                  │
                                  ├──▶ Microsoft Presidio (personal data)
                                  └──▶ Language model via gateway (context, quality)
```

Everything runs in Docker, so the project starts the same way on every machine. The
language model is reached through a gateway that only sends text to approved servers,
and the model provider can be swapped with a single setting. The university's model
wasn't accurate enough for the security check, so the team used the Claude API for
testing and the demo, and advised the bank to run its own internal model in production.

**Tech:** React, TypeScript, Python, FastAPI, Microsoft Presidio, SQLite, Docker, pytest.

## My contributions

**Sensitive-content detection.** Owned the Presidio pipeline and its evaluation.
Presidio doesn't support Dutch data out of the box, so it was extended with custom
recognisers:

- citizen service numbers (BSN), validated with the 11-proef so random nine-digit
  numbers aren't flagged
- Dutch addresses with postcodes, dates of birth, employee and passport numbers
- passwords, API keys and access tokens in code or configuration

Some of Presidio's built-in types, like dates and locations, were switched off on
purpose because they flagged ordinary prompts. An allow list keeps known harmless
terms from being masked, and employees can untick any finding they disagree with.

Beyond Presidio, a large part of the work went into the contextual check, which is
the layer that catches internal systems, endpoints, architecture details and config
settings. This also included building the labelled test set, with prompts at every
level and "traps" that only look sensitive, which the team used to measure both layers.

**UX design.** All UX decisions were made based on the client's feedback, and the
client was happy with the result. The guiding idea is that the employee stays in
control and can always go back. In practice that meant:

- every security finding can be applied, declined or undone, and is marked in the prompt
- quality advice can be edited, removed or extended, with fillable blanks like `[length]`
- a back button on every step, plus confirmations before losing work
- a "How it works" guide and a page that explains the four levels in plain language,
  showing level names instead of internal codes like C0–C3

**Client communication.** Acted as the team's single point of contact with Rabobank
and the university supervisor. That covered running the weekly client meetings and
sending a written summary within 24 hours of each one. It also meant agreeing early
what could be published for the public Bachelor Showcase.

**Other work.** Set up Docker for the frontend and backend, connected the frontend to
every backend endpoint, and built the built-in list of test prompts into the app.

## Example

A fictional prompt from a generic employee:

> Write an email to Jan de Vries (IBAN NL91ABNA0417164300) explaining why the
> PaymentHub API at /internal/v2/settle failed last night.

The workbench rated it **High (prohibited)**. It flagged the name and IBAN as personal
data, and the system name and endpoint path as internal technical details. Each finding
came with an explanation and a rewrite. After both rewrites were applied, the prompt became:

> Write an email to \<PERSON\> (IBAN \<IBAN_CODE\>) explaining why the payment
> settlement service failed last night.

The quality check then scored it 66/100. Clarity and task were full marks, but the
prompt had no constraints and little detail about the expected output. The workbench
suggested adding a scope, a tone, a length, an example and permission to say "I don't know".

## Results

The workbench was tested on 31 prompts labelled by the client. In the second test round:

- no sensitive prompt was missed, and no harmless prompt was flagged
- 28 of the 31 prompts got exactly the expected level
- the improved versions raised the quality score by 11.5 points on average

## Key design decision: rules first, model second

Presidio runs before the language model, so personal data is removed before any text
reaches a model, by a tool that behaves the same way on every run. The model only
handles what rules can't: information that is sensitive because of its context.

Neither layer has to be perfect on its own. Presidio is tuned to avoid false alarms,
and anything it misses is still caught by the contextual check, as in the example
above, where the second layer caught the name and IBAN. Letting the model detect
everything would have been simpler, but its output varies between runs, and raw
personal data would reach the model.

## Lessons learned

Working with a real client meant designing around constraints, such as guidelines that
couldn't be shared, a model that wasn't good enough, and a public showcase that had to
respect confidentiality. Combining deterministic tools with a language model also turned
out to work better than either one alone. Each covers the other's weak spots.
