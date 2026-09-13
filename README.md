<p align="center">
  <img src="https://raw.githubusercontent.com/llm-lens-agent/llm-lens-agent/main/assets/banner.svg" alt="LLM Lens. AI visibility, audited" width="100%">
</p>

---

## 👋 Hi, you probably got here from a pull request

This account is the automated assistant of [**LLM Lens**](https://llm-lens.thebluetonguegiraffe.online/). It doesn't write code and it isn't a person. Its only job is to deliver the files that LLM Lens generates, as a normal pull request, to the repository their owner asked us to send them to.

If a PR from this account showed up in your repo, it's because **someone with access to your project ran an LLM Lens audit on your site and chose "open a pull request" as the way to receive the results.** Nothing was sent anywhere without a human asking for it.

## What LLM Lens is

More and more people find websites through AI assistants (ChatGPT, Claude, Perplexity, Gemini) instead of a traditional search engine. Those assistants don't read a page the way a person or a classic search crawler does, so a site that ranks perfectly well on Google can still be confusing, invisible or off-limits to them.

**LLM Lens audits a website from that angle and tells you what an AI model actually sees when it looks at your site.** Then it goes one step further: instead of just handing you a list of problems, it writes the files that fix them.

## How it works

1. **You give it a URL.** You paste your address into [llm-lens.thebluetonguegiraffe.online](https://llm-lens.thebluetonguegiraffe.online/) and LLM Lens visits the site the way an AI agent would, loading the real page rather than just the source code.
2. **It runs an audit.** It checks whether the site tells AI assistants who you are and what you offer, whether it allows or blocks them, how fast and well structured the page is, and whether anything on it is meant to be used by an agent.
3. **It explains the findings in plain language.** A readable report, not a dump of scores.
4. **It generates the fixes.** You pick which file you want and LLM Lens writes it, tailored to your specific site rather than copied from a generic template.
5. **You review and take it.** You can ask questions about anything it produced, ask for a different version if you don't like the result, and finally download the files or have them delivered as a pull request. **You always see a file before it goes anywhere.**

## What ends up in a pull request

LLM Lens can generate five files today. Which ones arrive depends on what the audit found and on what the person running it chose, one at a time, approving each result before export.

| File | What it's for |
| :--- | :--- |
| **`llms.txt`** | A short, structured summary of your site written for AI assistants: what the project is, what it does, and which pages matter. It acts as a front door, so a model doesn't have to guess by scraping its way around. |
| **`robots.txt`** | The file that tells crawlers and AI bots where they may go. LLM Lens keeps every rule you already had and only adds explicit permissions for AI assistants, so you decide who gets in instead of leaving it to chance. |
| **`meta-tags.html`** | Your title, description and social preview tags, rewritten so an assistant summarising your page uses the description you intended rather than whatever text it happens to find first. Delivered as a snippet for your page's `<head>`. |
| **`structured-data.jsonld`** | A machine readable description of your content in the Schema.org format that search engines and assistants already understand: what kind of thing the page is, who publishes it, and how the pieces fit together. |
| **`mcp.json`** | A manifest listing the actions an agent can actually carry out on your site, such as searching, subscribing or booking a demo. It's only offered when your pages already declare those actions, because the manifest is assembled from them. |

One further output, the HTML annotations that mark up those agent actions inside your pages, is on the roadmap and not shipped yet.

### What a pull request from this account will **never** do

- It won't touch, rewrite or delete your existing code. It only adds new files.
- It won't merge itself. It's an ordinary PR: review it, merge it, or close it.
- It won't come back on its own. One audit, one pull request, requested by a human.

Not interested? Closing the PR is a perfectly fine answer, and nothing else happens.

## Who it's for

Developers, technical founders and small teams who want their project to be found, understood and correctly represented by AI assistants, without having to become an expert in a set of conventions that barely existed a year ago.

## Where to find us

Everything about LLM Lens lives at **[llm-lens.thebluetonguegiraffe.online](https://llm-lens.thebluetonguegiraffe.online/)**: that's where you run an audit, read what the tool found, and see the files before anyone opens a pull request with them.

**Questions, doubts about a pull request you received, or something that went wrong?** Use the **"Report an issue"** button on that site. It's the channel we actually read, so please don't open a GitHub issue here. The same page has a **"Join the waitlist"** button if you'd rather hear about what's coming next.

## Project status

LLM Lens is in active development. This account only handles delivery, while the source code lives in a private repository as the project matures.

---

<p align="center">
  <img src="https://raw.githubusercontent.com/llm-lens-agent/llm-lens-agent/main/assets/mark.svg" alt="" width="26">
</p>
<p align="center"> <sub> <strong><a href="https://llm-lens.thebluetonguegiraffe.online/">LLM Lens</a></strong> · an agentic auditor for the age of AI search · © 2026 The Blue Tongue Giraffe </sub> </p>
