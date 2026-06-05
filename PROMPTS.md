General prompts for day-to-day use.

---

## Documents & Meetings

```
List every action item and decision mentioned in this document, with the owner where one is named.
```

```
Summarise this in 3 bullets and flag any open questions.
```

```
Summarise this Slack thread for my weekly sync: what was announced, what changed, what people pushed back on, what (if anything) needs a follow-up from me. [paste the permalink]
```

---

## Code Review

Report everything — don't filter. Models follow filtering instructions too literally and silently drop real bugs.

```
Review the following code for best practices, performance, security, and maintainability:

[PASTE CODE]

Language/Framework: [e.g. Python/Django, TypeScript/React]

Report every issue you find, including low-severity and uncertain ones. Include a confidence level and estimated severity for each. Do not filter — I will triage downstream.
```

---

## Debugging

Specify error, expected vs actual, and ask for root cause + fix together.

```
Analyze and fix the following code. It produces this error: "[ERROR MESSAGE]"

Expected behavior: [what should happen]
Actual behavior: [what happens instead]

[PASTE CODE]

What is the root cause, and how do I fix it?
```

---

## Documentation

Constrain format + required fields to prevent omissions.

```
Generate API documentation for:

[PASTE API CODE OR ENDPOINTS]

Format: [OpenAPI / Swagger / Markdown]

For each endpoint include: description, HTTP method & path, request parameters, response shape, error codes.
```
