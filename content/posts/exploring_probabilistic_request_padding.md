---
title: "Exploring probabilistic classifier bypasses via prompt padding."
date: 2026-09-26T00:00:00+00:00
tags: ["hacking", "english","cybersec","literature","llm"]
author: "XoanOuteiro"
showToc: true
TocOpen: false
draft: false
description: "A theory on how we may see the rise of prompt padding in LLM attacks."
---

I will admit, as this is my personal blog, that I'm not the biggest LLM fan. I do not believe the hype that this is going to further revolutionize cybersecurity in the sense that I think it has furthered automation and that's it, to me, that's not really a praxis change.

However as a hacker I also believe it is important to adapt to tools and help in understanding how they can be attacked, and its adoption is undeniable. Please do not take this as the naive view of "AI is not THAT important", but as a (I hope) informed bias disclosure for whomever it may concern.

Anyhow this post is meant to describe and approach a concept I first became familiar with 3 years ago in web hacking called HTTP Request Padding or sometimes called Junk Data Injection. Not from an academic standpoint - as I'm not an academic - but from a curiosity point of view. If it's of any use to anybody else then this would have been a great use of my time. If you have anything you'd like to discuss or want to build from here please do let me know ideally via LinkedIn.

## So what's the point I'm trying to make

I do believe there's a real analogy between WAF bypasses via HTTP Request Padding and Classifier bypasses. In this post I'll try to prove why the current techniques indicate that this is an attack class worth adding to/ considering when creating taxonomies.

## Basic introduction to HTTP Request Padding

This technique which from now on we will call HTRP is a WAF bypass mechanism that's based on the assumption that cloud-based WAFs have an inherent cost-based CPU usage, and in sending increasingly bloated HTTP requests we can get the WAF to simply choose not to analyze that request in full either be it completely or just reading the first N bytes and then stopping.

If this drop goes according to our interest it will go through the WAF into the final endpoint instead of being dropped. In many cases I've observed this to be the case.

The manner in which we inject data into our HTRP adapted requests is via simply writing literal junk data into their body in such a manner that the request does not fall into malformation. This is oftentimes trivially done in both new and old web stacks.

### Determinism of the attack

Like any WAF attacks determinism is based on the assumption that we are referring to a web based attack. However because we are actively bypassing a security measure most analysts would choose to employ CVSS4.0 modifiers like AC:H to any attack employing this technique.

However request padding works in such a manner that we can oftentimes accurately calculate how many bytes of junk data we will need to insert into a request to get it to bypass the WAF.

### Calculating Padding Sizes

The original NoWafPls plugin simply provided a general list of bytes to insert into the request in order to bypass Cloud-based WAFs according to vendor and tier type.

I was dissatisfied with this as it was - forgive me for the pedantics - unelegant. And so developed a now forgotten WAF bypass tool called Caliper.py that could do binary searches based on request matching and response codes. It indeed pulled a very close number to the byte read limit I set on my WAF and that was that.

I also had the chance to use the tool in some pentests I performed during my career, but in most cases where HTRP worked other payload-based WAF bypass vectors emerged that were trivial to exploit in comparison.

## Why I pivoted padding techniques to prompts

When doing CTFs aided to learn new techniques I sometimes work in conjunction with LLMs like Claude Opus 4.8 or 5 to spot what I may be missing in my writeups and getting tailored clues that don't spoil the exercise.

As I'm on Anthropic's Cyber Verification Programme it is rare for me to get cyber warnings asking me to edit prompt, but when I do they oftentimes brick the whole instance causing any subsequent messages to also be flagged as dangerous.

This is a nuisance.

I once tried to then paste a whole (extremely lengthy) Active Directory writeup into the edit box and send the message, to my surprise the chat begun replying again without those lengthy classifier pauses it used to have before.

Now I can certainly not prove this was a classifier bypass, so I'm not gonna claim it was one.

This led me into a bit of a rabbit hole to try and research if those attacks I had grown fond of in Web were also applicable on LLM hacking.

Here I'll share some of what I found for whoever cares.

As disclosure, along with Google I did use LLMs to help find the applicable sources and also aid structure the techniques into specific slotted mechanisms for the sake of coming back to this easier when interested.

## Padding Techniques on LLM Classifiers

Let's begin with some decent previous reads:

- https://arxiv.org/pdf/2311.09198 (Never Lost in the Middle: Mastering Long-Context Question Answering with Position-Agnostic Decompositional Training. Liu et al.)
- https://arxiv.org/pdf/2406.16008 (Found in the Middle: Calibrating Positional Attention Bias Improves Long Context Utilization)
- https://www-cdn.anthropic.com/af5633c94ed2beb282f6a53c595eb437e8e7b630/Many_Shot_Jailbreaking__2024_04_02_0936.pdf (Many-Shot Jailbreaking. Anil et al.)
- https://github.com/assetnote/nowafpls/blob/main/README.md (The original NoWafPls tool by Assetnote)

And logically as usual I take inspiration in most LLM attacks reasoning from Jason Haddix's Arcanum Sec.


### A Mechanisms: Coverage Evasion

These are attacks on classifier coverage: the classifier never evaluates the malicious span at all. They are deterministic just like WAFs that are byte-limited. This category the closest structural equivalent to HTRP.

#### A1. Input truncation

Deployed guardrails are frequently small models with short, fixed input windows that discard anything past the limit. 

Small context windows with truncation enabled are observable in the wild. Whatever falls beyond the limit is evaluated as though it were absent. 

Truncation is itself a different issue: I found a documented case of head-only truncation, where a payload with a benign opening and a malicious tail was scored entirely on its opening. Padding the front to displace the malicious span past the limit, or exploiting the reverse condition where truncation is tail-side, reproduces the HTRP primitive of overrunning a WAF body-inspection limit.

Sources:
- https://github.com/BP602/omp-auto-mode/issues/10 (head-only truncation hiding the tail of a long value)
- https://huggingface.co/satyamsaf3ai/guardrail-roberta (RoBERTa guardrail, max_length 128, truncation on)
- https://huggingface.co/vijil/prompt-injection-v5-20260827 (windowing and truncation behaviour)

#### A2. Sliding-window scoring and its threshold economics

The standard remediation for A1 is to slide the classifier across the entire input in overlapping windows and flag if any window scores high, which is max-pooling. 

This closes the naive truncation hole but reintroduces it economically. (!- I consider this another case for this type of vector being paid more attention to.)

Because the decision is a maximum taken over many windows, each additional window is another opportunity for a false positive, and false positives compound with window count as the input grows. (Somewhat realated to many-shot jailbreaks.)

I do not believe, however, that furthering the payload with generic data will in practice realistically create false positives in such a manner that matters when performing probabilistic testing.

Sources:
- https://github.com/crunchtools/mcp-trentina/issues/225 (Prompt Guard 2, 512-token window, stride 446, max malicious score across windows)
- https://huggingface.co/vijil/prompt-injection-v5-20260827 (any-positive max-pool, false positives compounding with window count, higher threshold advised for large inputs)

#### A3. Cost-gated inspection

At scale, not every request is subjected to the expensive classifier. Pipelines may use a cheap pre-filter, sample a fraction of traffic, or scan only inputs that appear suspicious to deep inspection.

A large, unremarkable wrapper can suppress whatever low-cost signal decides escalation, such that the expensive check is less often started. (We may compare this to a simple WAF governing wether or not a request warrants deep packet inspection.)

This is the most faithful to HTRP in spirit and the least represented in the formal literature as far as I know.

Sources:
- https://github.com/assetnote/nowafpls/blob/main/README.md (the cost-driven inspection-budget premise, transposed from WAFs)
- Haven't found any real source for this technique, part of the reason why I'm writing this.

### B Mechanisms: Attention Dilution

These are attacks on classifier attention rather than coverage, they have no meaningful web analog.

The classifier reads the malicious section but distributes its attention across the whole content, so a small adversarial region inside a large benign one is under-weighted. 

This category is probabilistic: padding raises the probability of evasion, it does not guarantee it. (!! This is the first real mention of probabilistic bypasses via padding.)

#### B1. Positional dilution

Attention is spread more thinly as the sequence lengthens, and content positioned in the middle competes with the greatest number of neighbours for weight, so it is attended least.

Measured accuracy drops are reported when the target span is moved from the edges to the middle of the context. Anecdotically I've experienced this during LLM Hacking challenges.

Padding exploits this by relocating the payload into that low-attention interior.

Sources:
- https://arxiv.org/pdf/2311.09198 (Lost in the Middle / Never Lost in the Middle, U-shaped positional degradation)
- https://arxiv.org/pdf/2406.16008 (Found in the Middle, positional attention bias)

#### B2. Length-induced safety degradation

The strongish result in recent work is that the padding need not be adversarial in content. (In fact I'd argue it's logically better if it's not.)

Two papers that mention this are Ninja, from Jailbreaking in the Haystack, reasons context length as the variable and demonstrates that refusal rates decline rapidly as benign context grows (again very much the point of this article), outpacing the very notable decline in general capability, and that innocuous padding is harder to detect than the injection of harmful examples. 

Sources:
- https://arxiv.org/pdf/2511.04707 (Jailbreaking in the Haystack / Ninja, context length as primary driver of safety degradation)
- https://arxiv.org/html/2510.26418v1 (Chain-of-Thought Hijacking, benign reasoning diluting scrutiny and verdict signals)
- https://www-cdn.anthropic.com/af5633c94ed2beb282f6a53c595eb437e8e7b630/Many_Shot_Jailbreaking__2024_04_02_0936.pdf (Many-Shot Jailbreaking, long-context exploitation of in-context learning)

#### B3. Cognitive overload

A demanding, unrelated task is placed ahead of the malicious request.

The decoy consumes the model's effective attention so that moderation is not meaningfully applied to the payload that follows.

This belongs to the same family as B1 and B2, being an attention misallocation, but it's complexity based instead of length based. We may see further mention of these techniques as Agentic LLM usage increases.

Sources:
- https://www.pillar.security/blog/deep-dive-into-the-latest-jailbreak-techniques-weve-seen-in-the-wild (Distract and Attack, context-prioritisation limits)
- https://arxiv.org/pdf/2510.13893 (Guarding the Guardrails, Cognitive Overload and Attention Misalignment taxonomy)

### C Mechanisms: Padding as Force-Multiplier

In this category padding is not the whole attack.

#### C1. Payload splitting

A blocklisted term or instruction is fragmented into individually benign pieces, optionally with padding interposed between them, and the model is relied upon to reassemble the whole.

Classifiers that score fragments independently, or that inspect only the last N turns, will not read the reassembled intent.

The corresponding defence is to decode and concatenate prior to classification. 

In RAG this generalises to splitting an instruction across separately-scanned documents that recombine at retrieval time.

Sources:
- https://chat-test.learnprompting.org/docs/prompt_hacking/offensive_measures/payload_splitting (payload splitting overview, Kang et al.)
- https://inferensys.com/glossary/preemptive-algorithmic-cybersecurity/ai-red-teaming-automation/payload-splitting (tokenisation-level splitting, truncated safety scanning, RAG indirect injection)
- https://arxiv.org/pdf/2510.13893 (Guarding the Guardrails, payload splitting under decomposition attacks)

#### C2. Token-boundary and character injection

The surface text is modified so that the classifier's tokeniser observes something harmless while the model still infers the intent. Junk characters joined to trigger words, as in TokenBreak, or homoglyphs, zero-width and invisible characters, and emoji-based smuggling.

Recently as of 28/09/2026 there has been a trend of employing norse runes for DAN prompts. I am yet to see this technique be as effective as other researchers claim.

In isolation these are said to reach very high evasion rates, in some cases approaching 100% against several commercial guardrails, while still beign legible to the model.

Volume padding should then make the malformed region less suspicious.

Sources:
- https://arxiv.org/abs/2504.11168 (Bypassing LLM Guardrails, character injection and AML evasion up to 100% ASR)
- https://mindgard.ai/blog/outsmarting-ai-guardrails-with-invisible-characters-and-adversarial-prompts (invisible characters, homoglyphs, emoji smuggling)
- https://www.pillar.security/blog/deep-dive-into-the-latest-jailbreak-techniques-weve-seen-in-the-wild (TokenBreak, token-boundary manipulation)


## Other applications

I do believe that we will soon see application of these techniques regarding LLM writing detectors, not to bypass classifiers but to hide LLM written sections.
