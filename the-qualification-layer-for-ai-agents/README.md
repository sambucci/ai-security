# The Qualification Layer for AI Agents

*Luca Sambucci. Originally published on [Are We Safe Yet](https://www.arewesafeyet.com/the-qualification-layer-for-ai-agents/), 4 August 2026.*
*[CC BY-ND 4.0](https://creativecommons.org/licenses/by-nd/4.0/)*

> *"They're coming out of the goddamn walls."* — Pvt. Hudson, Aliens (1986)

---

When a company announces it is putting tens of thousands of AI agents into production, somebody in that building will eventually be asked whether the agents are safe to run.

I've been trying to work out what that person could honestly say. The answer is nothing, and it isn't because they've been careless. **The artifact they would need in order to answer does not exist yet**.

We have spent two decades learning how to assure software. The method works because applications hold still. You test the thing, and the thing you tested is the thing that runs, until someone deploys again. Every assurance practice we have (pentest windows, change advisory boards, sign-off, the annual audit, you name it) rests on that one property.

Agents do not have it.

## The thing that changed wasn't the agent

Picture a claims-triage agent, tested at procurement, approved, running for months.

This morning someone attaches a spreadsheet of privileged customer information to a request, because that is a reasonable thing for an employee to do and nobody told them otherwise.

No code was deployed. No permission was granted. No ticket was raised. **The agent's reachable harm changed because a human dropped a file into its context, and every safety statement anyone had ever made about that agent went stale in that moment**. The [research](https://arxiv.org/abs/2607.05120) on this failure mode is not encouraging: boundary-corruption attacks of exactly this kind succeed around half the time and **defeat every current injection defense** except full data-flow tracking, which costs so much task utility that nobody runs it.

Now multiply that by the number of times per day, across - I don't know - thirty thousand agents, that somebody adds a document, connects a tool, wires in one more system, or lets one agent hand work to another.

Some of the drift has no attacker in it at all. Agent safety degrades progressively as ordinary, benign memory accumulates, [measured across eight memory architectures](https://arxiv.org/abs/2605.17830), worst where retention is longest. And in a [study](https://arxiv.org/abs/2605.13044) of four hundred deployed agent skills, **nearly thirty percent violated their own declared guardrails** under entirely benign inputs. Before anyone attacks anything, the stated behavior is not the actual behavior.

## Your vendor's number describes a system that isn't yours

The natural response is to lean on the model provider. Their safety evaluation is real work, competently done, and it does not bound your exposure.

And that response would be wrong.

The clearest evidence is a variance decomposition. In a controlled [study](https://arxiv.org/abs/2605.28122) of agents exceeding their authorized scope, the agent framework accounted for 56.1% of the variation and the base model for 20.8%. **Most of the risk lives in your deployment, and your vendor has never seen your deployment.**

Architecture behaves as an attack parameter in its own right. [Researchers ran identical web attacks](https://arxiv.org/abs/2608.00202) against a single agent and against the same work split across a crew of cooperating agents: credential theft succeeded 11% of the time against the single agent and 69% once the task was decomposed. **Same model, same attack, same prompt text**. Prompt hardening that helped the single agent sometimes made the crew worse, and detection was close to zero. The failures were silent.

Worse for anyone relying on procurement-time testing. The small things invalidate the verdict too. Changing the *format* of a tool specification, holding the request word-for-word identical, [moved refusal from 58% to 3%](https://arxiv.org/abs/2607.29254) (when dealing with a potential attack, **refusal is good**). In other words, a platform engineer tidying a schema is going to become a security event.

And when two safety fine-tunes are merged, [refusal behavior survives while harm recognition collapses](https://arxiv.org/abs/2607.27240), and weight statistics cannot detect that it happened. **Your supplier can invalidate your assurance on their release schedule**, and you will not get a notification.

## Dumb red-teaming will not close this

The instinct is to buy more testing. More pen-testing, more red-teaming. Alas, it doesn't scale, for reasons that have nothing to do with budget.

Firing more probes is not the same as learning more. Attack success measured against an undefended model predicts defender-side value at a correlation of [about 0.2](https://arxiv.org/abs/2607.17152), close to noise. Two of the standard automated attackers turn out to produce [between 73% and 84% exact duplicate attacks](https://arxiv.org/abs/2606.26793). The large battery is substantially a copy of itself. Meanwhile, structured knowledge of the target beats attack refinement decisively, since separating reconnaissance from exploitation lifted success from [37.7% to 86%](https://arxiv.org/abs/2607.19837).

And the scoring is wrong at the end anyway. Judging only whether harm occurred is structurally insufficient, [73.5% of defense pairs that both produced zero harm diverged](https://arxiv.org/abs/2607.23999) on what the agent actually did along the way.

So the market has one layer that works and one layer that is missing. The layer that works fires a catalog of generic probes at an endpoint and returns a coverage report. "These attack classes were attempted, this many got through". Don't get me wrong, real work happens there, the open-source tooling is good, **and it is commoditizing quickly**, as it should. After all, its output is just a list.

## We need a Qualification layer

Aerospace has a name for what's missing. A part flies only after someone has stressed it against the specific envelope it will operate in, recorded what broke, when, how, under which conditions, and signed a decision. A supplier certificate and a generic bench test don't put anything in the air. **That is qualification**. No engineering discipline that builds safety-critical systems skips it. And yet, still today, AI agents ship without it.

> **A qualification layer for AI agents builds a custom threat model for a specific agent, generates on-the-fly attacks derived from that threat model and specifically tuned to that particular agent, runs them against the agent in its deployed architecture, and reads the result back against thresholds the organization declared in advance, ending in a decision. Qualified or disqualified.**

Dated. Logged. Against a named threat model, at thresholds published alongside the verdict, so that a permissive threshold is visible.

The threat model is what makes this affordable. A chatbot with no route to sensitive data does not need to be tested for intellectual-property extraction, but it needs a very low tolerance for abusive output. An underwriting agent with database write access and an email tool might invert both. The same generic suite fired at both systems wastes most of its effort on one and misses the point of the other.

Because the invalidating events are continuous, qualification has to be event-driven and staged. A fast admission check when something material changes, and - where an organization can degrade an agent into a reduced-capability envelope - a deeper check that runs there without a deadline. Honest numbers require search breadth that cannot fit inside a business-hours window. Quarantine is how you buy the time without stopping the work.

## The four-second test

You can apply this to your own last AI security engagement without reading anything. **Did it end in a decision, or in a list?**

If it ended in a list, somebody downstream converted it into a decision anyway, quietly, against thresholds nobody declared. If you are a CISO, that somebody is usually you. Like it or not, the signature goes on the deployment either way. The only thing you get to choose is what stands behind it.

So, go back to the person in that building, the one who will eventually be asked whether twenty-thirty-forty thousand agents are safe to run on any given day. The artifact they need does not exist yet. But now we know its shape: **a decision, dated, against a named threat model, at clear thresholds**.
