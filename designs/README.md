# Design proposals

Designs for changes that are **not built yet**. The case studies in the parent
folder describe systems as they run, with real measurements; these documents
describe what is proposed, why, and what would have to be true to build it.
When a proposal ships, its outcome, including a negative result, goes into the
matching case study.

Both proposals below answer the same question for a different system: can a
small fine-tuned model of our own take over work from a hosted model, at no
cost, and be measured against the checks the system already trusts?

| Proposal | System | What changes | Where the model runs |
| --- | --- | --- | --- |
| [A local fine-tuned news classifier](vivatrades-news-classifier.md) | VivaTrades | Headline sentiment and importance move from a hosted sub-agent to a small local model, for all 500 stocks | The machine that already runs the nightly pipeline |
| [A fine-tuned model of our own, with Groq as fallback](shubh-bhagya-own-model.md) | Shubh Bhagya | Live readings get a model trained only on outputs that pass the site's rule checks | Cloudflare Workers AI free allocation, first in the provider chain |

Shared rules in both:

- **No user data and no hosted-model outputs as training data.** Training sets
  come from synthetic inputs or public text, labelled by open-weight models and
  corrected by hand.
- **The existing checks are the bar.** A model ships for a task only if it beats
  what runs today on the same measurements.
- **The current path stays as the fallback.** Nothing is removed until the new
  path has run in production with its numbers visible.

Status: written 2026-11-03, awaiting owner review.
