# CrewAI Build Report

**Name:** Shreeyash Baranwal<br>
**Topic I used:** retrieval-augmented generation

## 1. What I changed

I added a fourth agent, a Fact Checker, to `crewai-kit/main.py`. I added a new `Agent` block (`fact_checker`) right after the editor, and a new `Task` block (`fact_check_task`) right after `editing_task`. Then I added both to the end of the `agents` and `tasks` lists in the `Crew(...)` call.

The task asks the Fact Checker to go through the edited explainer, pull out every factual claim, and label each one VERIFIED, UNCERTAIN, or LIKELY WRONG with a one-line reason. I set `context=[editing_task]` so it works on the Editor's final version, and I wrote the backstory to say it has no internet access and shouldn't pretend to have checked sources.

I picked this because the kit has no web tool, so everything the Research Analyst says comes from the model's memory. The Editor only marks things as [source needed], which doesn't tell you how worried to be about each claim. A separate confidence label per claim seemed like a useful next step, and it's easy to spot in the output.

## 2. How CrewAI works

CrewAI is built around three pieces. An **agent** is a role: it has a `role`, a `goal`, and a `backstory`, and CrewAI turns those into the system prompt for that agent's model calls. That's why the backstory changes how an agent behaves even though it's just plain text. A **task** is a unit of work with a `description`, an `expected_output` (what "done" looks like), and the agent who owns it. The **crew** ties agents and tasks together and decides how they run. This kit uses `Process.sequential`, so tasks run in list order and each one gets the earlier output as context.

So the whole pipeline is Analyst → Writer → Editor, and now → Fact Checker. My change worked because I added both an agent and a matching task, and put the task last in the list. Since it's last, its output is what `crew.kickoff()` returns, and `main.py` saves `result.raw` to `output/explainer.md`. That's why the Fact-Check table shows up in `explainer_run2.md` without me changing anything else in the file.

One thing I noticed: in both runs the Editor's final answer included the whole research brief as well as the explainer. So the Editor seems to get more than just the Writer's draft from the earlier tasks. I set `context=[editing_task]` on my task explicitly so it was clear which output the Fact Checker should review.

## 3. Evidence

`run1_original.txt` shows the original three agents. `explainer_run1.md` has the research brief plus the explainer, with nine [source needed] flags from the Editor.

`run2_changed.txt` shows a fourth "Fact Checker" agent box after the Editor. `explainer_run2.md` ends with a new "Fact-Check Table" listing 16 claims: 13 VERIFIED, 3 UNCERTAIN, 0 LIKELY WRONG. The labels don't always agree with the Editor. The Editor flagged "facts can be added to the index without re-training" as [source needed], but the Fact Checker marked the same claim VERIFIED.

Since it's the same topic and the same model, the brief and explainer text also differ between runs. That's expected because the output is non-deterministic.

What went wrong:
- In run 2, the Technical Writer's answer cut off after about two sentences ("Imagine a writer who first searches a library for relevant..."). The Editor just finished the sentence and moved on, so the "250-word explainer" in run 2 is really two sentences. My change didn't cause this, because the Writer runs before the Fact Checker. The same cut-off happened in an earlier attempt I threw away. Nothing in the crew checks that `expected_output` was actually met, so a broken step passes through silently. I'm reporting it instead of rerunning until I got a clean one.
- My first attempt at run 1 was also missing the command line at the top of the terminal text. I'd pasted `cls` and the run command together, so `cls` cleared the echoed command. I reran with `cls` on its own first.

## 4. Choosing a framework

CrewAI fits work that splits naturally into roles that hand off to each other. That's what this was: gather, write, edit, check. It also matches how my M&A due-diligence work at PwC was structured, where one person pulls the data, someone drafts the findings, and a reviewer checks them before anything goes out. Describing that as agents and tasks in plain English was quick. Adding a whole new role took two blocks of code and two list edits.

Compared with **LangGraph**, I think the same pipeline would take more code, because I'd have to define the state, the nodes, and every edge myself. But LangGraph would have made the problem I hit easier to handle. I could add a conditional edge after the Writer that checks the draft length and loops back if it's cut off, instead of letting a two-sentence "explainer" go through to the Editor. LangGraph also has `interrupt_before`, which would let me pause and approve the draft before it's checked. CrewAI may have ways to do this too, but the kit didn't show them.

So I'd prototype in CrewAI, and move to LangGraph if this needed to be reliable enough that someone else depends on the output.

## 5. Verification and AI use

The kit has no web access, so I treated everything in the outputs as unverified. I read both explainers myself, and the general description of RAG (retriever plus generator, updating the index without retraining) matches what I know from working on a RAG project. I spot-checked two citations the agents used. "Karpukhin et al., 2020" is the Dense Passage Retrieval (DPR) paper, but the Fact Checker cited it as evidence for reranking. "Izacard & Grave" is the Fusion-in-Decoder paper, published in 2021, while the agents cite it as 2022. The Fact Checker marked both claims VERIFIED anyway. That's the main limit of my change: the Fact Checker uses the same model with the same memory, so it can't catch the model's own mistakes. A real fix would be giving it a search tool.
