# GTM Form Engagement Waterfall

A Google Tag Manager container recipe that tracks a contact form's full lifecycle — **visibility → field engagement → submit → success/error** — as four stages of a single GA4 event, so the whole thing drops straight into a GA4 Funnel Exploration as a form-abandonment waterfall.

Built and documented against Blogger's built-in contact form widget, but the pattern generalizes to any form.

## Contents

- [`gtm-waterfall-form-field-engagement.json`](./gtm-waterfall-form-field-engagement.json) — the exportable GTM container (Export Format Version 2)

## Table of Contents

- [Overview](#overview)
- [The Four Stages](#the-four-stages)
- [Requirements](#requirements)
- [Installation](#installation)
- [Configuration](#configuration)
- [Event Reference](#event-reference)
- [Container Contents](#container-contents)
- [Adapting to Your Own Form](#adapting-to-your-own-form)
- [Building the GA4 Funnel](#building-the-ga4-funnel)
- [Credits & Related Work](#credits--related-work)
- [License](#license)

## Overview

Most form-tracking setups fire a single event on submit and call it done. That tells you a form was submitted — it tells you nothing about the funnel that led there, where visitors dropped off, or whether the submission actually succeeded once it reached the server.

This container sends **four GA4 events, in page order, all sharing one event name** — `contact_form_step` — differentiated only by a `contact_step_label` parameter. Because every stage shares the same event name, GA4's Funnel Exploration can treat `contact_step_label` as the step axis and chart the full waterfall: how many visitors saw the form, engaged with a field, submitted it, and got a successful response — without any custom event-name mapping.

## The Four Stages

| # | Stage | Trigger Type | `contact_step_label` |
|---|-------|--------------|------------------------|
| 1 | **Visible** — the form scrolls into view | Element Visibility | `visible` |
| 2 | **Field** — a visitor changes a field's value | Custom Event (`gtm.change`) | the field's `name` attribute |
| 3 | **Submit** — the visitor clicks Send | Click | `submit` |
| 4 | **Response** — a success or error message appears | Element Visibility | `success` / `error` |

Stage 4 also fires a second, separately-named conversion event (`contact-form-success` / `contact-form-error`) so a clean conversion metric exists independent of the funnel's dynamic label.

## Requirements

- A Google Tag Manager (Web) container with **Workspaces**, **Templates**, and **Environments** support enabled (standard on all current GTM accounts)
- A GA4 property and Measurement ID
- A form with elements you can target by CSS selector (the container ships pre-configured for Blogger's contact form widget)

## Installation

1. Download [`gtm-waterfall-form-field-engagement.json`](./gtm-waterfall-form-field-engagement.json) from this repo.
2. In GTM, go to **Admin → Import Container**.
3. Choose the downloaded file, select your target workspace, and pick **Merge** (recommended, to preserve your existing tags/triggers/variables) or **Overwrite**.
4. Confirm the import.

## Configuration

Before publishing, update the following to match your own property and markup:

| What to change | Where |
|---|---|
| GA4 Measurement ID (dev) | Variable: `Measurement Stream ID Development` |
| GA4 Measurement ID (prod) | Variable: `Measurement Stream ID Production` |
| Form container selector | Trigger: `contact-form-widget` → `elementSelector` |
| Submit button identifiers | Trigger: `contact-form-button-submit` → Click ID / Click Classes filters |
| Success message selector | Trigger: `contact-form-success-message-with-border` → `elementSelector` |
| Error message selector | Trigger: `contact-form-error-message-with-border` → `elementSelector` |
| Success/error → label mapping | Variable: `contact form step conversion status` |
| Success/error → conversion event name mapping | Variable: `contact form conversion metric name` |

In GA4, register these as **event-scoped custom dimensions** (Admin → Custom definitions):

- `contact_step_class`
- `contact_step_name`
- `contact_step_id`
- `contact_step_label`

Then mark `contact-form-success` as a **Conversion** (Admin → Events).

## Event Reference

| Stage | Event Name | Parameters |
|---|---|---|
| 1. Visible | `contact_form_step` | `contact_step_class`, `contact_step_name`, `contact_step_id`, `contact_step_label = "visible"` |
| 2. Field | `contact_form_step` | same four params, `contact_step_label` = changed field's `name` |
| 3. Submit | `contact_form_step` | same four params, `contact_step_label = "submit"` |
| 4. Response | `contact_form_step` | same four params, `contact_step_label = "success"` \| `"error"` |
| 4b. Conversion | `contact-form-success` \| `contact-form-error` | `conversion_label`, `conversion_value`, `conversion_name` |

## Container Contents

**Tags**
- `Google Analytics Configuration` — base GA4 config tag
- `onChange Listener` — Custom HTML; attaches a native `change` listener to `document` on `gtm.js` load
- `contact form step visible` — Stage 1
- `contact form step field` — Stage 2
- `contact form step submit` — Stage 3
- `contact form step response` — Stage 4 (waterfall)
- `contact form conversion metric` — Stage 4 (discrete conversion event)
- `onChange Event Push` — legacy tag from the original 2013-pattern recipe; **paused**, superseded by `contact form step field`

**Triggers**
- `gtm.js load` — Custom Event, initializes the onChange listener
- `contact-form-widget` — Element Visibility, Stage 1
- `gtm.js change` — Custom Event, Stage 2
- `contact-form-button-submit` — Click, Stage 3
- `contact-form-success-message-with-border` — Element Visibility, Stage 4 (success)
- `contact-form-error-message-with-border` — Element Visibility, Stage 4 (error)

**Key Variables**
- `generic event handler` — Custom JavaScript; the dataLayer-push callback shared by any native browser event
- `coalesce className` / `coalesce id` — walk up the DOM to return the first non-empty class/id
- `Coalesce Click Elements` — returns the first usable label from Click Text → Element Title → Element Name → Element Value
- `contact form step conversion status` — Switch/Lookup Table mapping Form ID → `"success"` / `"error"`
- `contact form conversion metric name` — Switch/Lookup Table mapping Form ID → conversion event name

**Custom Templates**
- `If Else If – Advanced Lookup Table` (Sublimetrix) — powers every coalesce variable
- `Timestamp` (Luratic)
- `Get Root Domain` (mbaersch)

## Adapting to Your Own Form

This recipe is markup-agnostic by design — nothing about the tags or the waterfall structure is Blogger-specific. To point it at a different form:

1. Update the four CSS selectors listed under [Configuration](#configuration).
2. Update the Click ID / Click Classes filter values on the submit trigger to match your submit button.
3. If your framework doesn't expose distinct DOM elements for success/error states, replace the two Element Visibility triggers in Stage 4 with whatever signal your form does expose (a class toggle, a `gtm.formSubmit` variant, or a custom `dataLayer.push()` from your own code).
4. Leave Stages 1–3 as-is — they depend only on generic browser events (`change`, `click`) and the coalesce variables, not on any Blogger-specific markup.

## Building the GA4 Funnel

1. **Explore → Funnel exploration** in GA4.
2. Add one step per stage, each filtered on `Event name = contact_form_step`.
3. Add a second condition per step on `contact_step_label`: `visible`, then your field name(s), then `submit`, then `success`.
4. Optionally break the field step down further by `contact_step_id` or `contact_step_name` to compare individual fields.
5. Toggle **Open funnel** on or off depending on whether you want to include visitors who entered mid-sequence.

## Credits & Related Work

This container builds on two prior techniques, both credited in full in the accompanying write-up:

- **Field engagement (Stage 2)** uses the generic `document.addEventListener` dataLayer-push pattern originally published by [Simo Ahava](https://www.simoahava.com) in 2013, adapted for GA4 in [*Listen to Any Browser Event in GTM*](https://drewspen.blogspot.com/2026/06/listen-to-any-browser-event-in-gtm.html).
- **Element identification (Stages 1 & 4)** uses the DOM-coalescing technique from [*GTM Universal Click – No JavaScript*](https://drewspen.blogspot.com/2026/06/gtm-universal-click-no-javascript.html).
- The full narrative write-up for this recipe — including why the auto-event variable families (Click ID, Form ID, etc.) all read from the same underlying dataLayer keys, and how that's exploited across every stage — is on [drewspen.blogspot.com](https://drewspen.blogspot.com).
- Community custom templates: `If Else If – Advanced Lookup Table` by [Sublimetrix](https://github.com/sublimetrix/gtm-template-ifelseif), `Timestamp` by [Luratic](https://www.luratic.com), `Get Root Domain` by [mbaersch](https://github.com/mbaersch).

## License

No license has been specified for this repository yet. Add a `LICENSE` file (MIT is a common, permissive choice for GTM recipe repos) before treating this as open source.
