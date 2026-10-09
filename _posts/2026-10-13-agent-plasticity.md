---
layout: post
title: "Agent Plasticity: Measuring How Well AI Agents Get Better Through Experience"
date: 2026-10-13 09:00:00
author: <a href="https://harmandotpy.github.io/">Harman Singh</a>, <a href="https://github.com/akhti">Anton Bakhtin</a>, <a href="https://rulinshao.github.io/">Rulin Shao</a>, <a href="https://syhw.github.io/">Gabriel Synnaeve</a>, <a href="https://uralik.github.io/">Ilia Kulikov</a>, <a href="https://cs.nyu.edu/~fergus/">Rob Fergus</a>,<br><a href="https://www.cs.princeton.edu/~arora/">Sanjeev Arora</a>, <a href="https://people.eecs.berkeley.edu/~keutzer/">Kurt Keutzer</a>, <a href="https://ai.meta.com/people/jason-weston/">Jason Weston</a>, <a href="https://anuj-mahajan.github.io/">Anuj Mahajan$^\dagger$</a> and <a href="https://anirudh9119.github.io/">Anirudh Goyal$^\dagger$</a>
# PRODUCTION (uncomment after uploading to /static/blog/agent-plasticity/):
# img: https://bair.berkeley.edu/static/blog/agent-plasticity/cover.png
# PREVIEW (github.io) - use until server static upload:
img: assets/agent-plasticity/cover.png
excerpt_separator: <!--more-->
visible: True
show_comments: False
---

<!-- twitter -->
<meta name="twitter:title" content="Agent Plasticity: Measuring How Well AI Agents Get Better Through Experience">
<meta name="twitter:card" content="summary_large_image">
<!-- PRODUCTION: <meta name="twitter:image" content="https://bair.berkeley.edu/static/blog/agent-plasticity/cover.png"> -->
<meta name="twitter:image" content="https://bairblog.github.io/assets/agent-plasticity/cover.png">

<meta name="keywords" content="agent plasticity, self-improvement, LLM agents, continual learning, persistent artifacts, evaluation, chess, Go, Hex, NetHack">
<meta name="description" content="Most evaluations ask what an AI agent can do today. We measure how efficiently frontier agents turn experience into persistent, held-out improvement: some improve a lot, others barely move, the strongest final performer is not the most efficient learner, and improvement often breaks down at artifact reuse or artifact quality.">
<meta name="author" content="Harman Singh, Anton Bakhtin, Rulin Shao, Gabriel Synnaeve, Ilia Kulikov, Rob Fergus, Sanjeev Arora, Kurt Keutzer, Jason Weston, Anuj Mahajan, Anirudh Goyal">

<style>
.ap-fig { display: block; text-align: center; margin: 2em 0; line-height: 1.4; }
.ap-fig img { display: block; margin: 0.6em auto 0; height: auto; max-width: 100%; }
.ap-fig i { display: block; margin: 0.7em auto 0; max-width: 92%; font-size: 0.92em; }
.ap-finding { background: #f4f2fb; border-radius: 10px; padding: 0.9em 1.2em; margin: 1.6em 0; }
</style>

<p class="ap-fig">
<!-- PRODUCTION: <img src="https://bair.berkeley.edu/static/blog/agent-plasticity/fig1_learning_curves.png" alt="Learning curves of held-out score against learning cost, and plasticity-to-saturation bars" width="100%"> -->
<img src="/assets/agent-plasticity/fig1_learning_curves.png" alt="Learning curves of held-out score against learning cost, and plasticity-to-saturation bars" width="100%">
<i>
Frontier agents differ widely in how efficiently experience helps them improve. Left: held-out score against
cumulative learning cost, averaged over chess, Go and Hex; ★ marks where each learning curve levels off. Right:
plasticity to saturation ($P_{\text{sat}}$), the held-out points gained up to that point per \$1,000 of learning cost.
</i>
</p>

Most evaluations of AI agents ask what an agent can do at a fixed point in time. But agents increasingly outlive a
single task: they diagnose their failures, write tools, keep notes and build up skills that later runs reuse. Two
agents that start out equally strong can then diverge, one steadily improving with experience and the other staying
where it began. In our new paper, we ask a different question: **how well do agents get better?**

<!--more-->

We study three questions. Does future performance improve, and does it generalize beyond the interactions the agent
learned from? How efficiently are new capabilities acquired? And when self-improvement stalls, where does the process
break down? To answer them we introduce **agent plasticity**: the efficiency with which an agent converts experience
into gains in future, held-out performance.

## Learning from experience, with the weights frozen

In our setup the model's weights never change. What changes is an **inventory** of persistent **artifacts** that the
agent writes and revises for itself: executable Python tools, written strategies and skills, and memory notes. Every
acting episode starts in a fresh context, so the inventory is the only thing that carries knowledge forward. The model
plus its inventory after $t$ rounds of learning is **checkpoint $t$**.

<p class="ap-fig">
<!-- PRODUCTION: <img src="https://bair.berkeley.edu/static/blog/agent-plasticity/learning_from_experience.gif" alt="Animation: agents play games, compile their experience into reusable infrastructure, and use it in the next round" width="100%"> -->
<img src="/assets/agent-plasticity/learning_from_experience.gif" alt="Animation: agents play games, compile their experience into reusable infrastructure, and use it in the next round" width="100%">
<i>
Learning from experience: agents play games, then compile what happened (trajectories, environment feedback, tool
calls and outputs, failures, metadata) into reusable infrastructure that the next round starts from.
</i>
</p>

Each round follows the same loop. The agent plays training games; it receives outcomes, traces and move-quality
feedback; a fresh instance of the same model reflects on that evidence and proposes edits to the inventory; and the
edits must pass structural and executable checks before they persist. Rejected attempts still count towards the
learning cost.

<p class="ap-fig">
<!-- PRODUCTION: <img src="https://bair.berkeley.edu/static/blog/agent-plasticity/learning_loop.png" alt="The learning loop: inventory, act, feedback, reflect, validate" width="100%"> -->
<img src="/assets/agent-plasticity/learning_loop.png" alt="The learning loop: inventory, act, feedback, reflect, validate" width="100%">
<i>
The learning loop. Every checkpoint is scored on held-out games whose trajectories and scores are never shown to
reflection.
</i>
</p>

Crucially, every checkpoint is scored on **held-out** games: held-out **in-distribution (ID)** opponents match the
training difficulty, and held-out **out-of-distribution (OOD)** opponents are stronger. We run 20 checkpoints of chess
(two difficulty profiles, Hard and Easy), 5×5 Go and 6×6 Hex against Stockfish, GNU Go and a native Hex engine, and 40
checkpoints of NetHack, a long-horizon, partially observed dungeon game. We evaluate frontier models including Claude
Fable 5, Claude Opus 5, GPT-5.6 Sol, Claude Opus 4.8, Gemini 3.1 Pro, GPT-5.5, Claude Opus 4.6 and GPT-5.6 Luna; NetHack
uses the models available through Claude Code and Codex.

## Measuring plasticity

**Learning cost** is the estimated dollar cost of the model calls spent on training games and reflection (including
rejected edits), which we use as a proxy for compute; held-out evaluation is not counted. The simplest measure of
plasticity is the held-out gain divided by the learning cost:

$$P = \frac{S_t - S_0}{C_t}$$

where $S_t$ is the held-out score at checkpoint $t$ and $C_t$ the learning cost spent up to that checkpoint. This ratio
keeps shrinking once a learner plateaus, because spending continues after the gains stop. So our headline measure,
**plasticity to saturation** ($P_{\text{sat}}$), fits a smooth learning curve to each agent and stops the clock at
saturation: the first checkpoint that reaches 90% of the fitted rise. $P_{\text{sat}}$ is the gain up to that point
divided by the cost of getting there.

<p class="ap-fig">
<!-- PRODUCTION: <img src="https://bair.berkeley.edu/static/blog/agent-plasticity/plasticity_concept.gif" alt="Conceptual animation of three agents' learning curves and their saturation points" width="85%"> -->
<img src="/assets/agent-plasticity/plasticity_concept.gif" alt="Conceptual animation of three agents' learning curves and their saturation points" width="85%">
<i>
A conceptual illustration. Agent A starts and ends highest but saturates only after a large learning cost; agent B
starts lower but rises quickly, so it has the highest plasticity; agent C starts above B and ends below it.
</i>
</p>

## Finding 1: Persistent improvement varies across models and tasks

Every agent gets the same interface for feedback, reflection and artifact construction, yet the learning curves come
apart almost immediately. In Hard chess, Claude Fable 5's held-out ID score rises from 37.5% at checkpoint 0 to 73.3%
averaged over checkpoints 16–20, and Claude Opus 5's from 25.0% to 66.4%. GPT-5.6 Sol starts near 0% but gains about as
much (to 36.9%). Several other models stay close to where they started, and a few decline.

<p class="ap-fig">
<!-- PRODUCTION: <img src="https://bair.berkeley.edu/static/blog/agent-plasticity/chess_train_id_ood.gif" alt="Hard chess scores by checkpoint on training, held-out ID and held-out OOD games" width="100%"> -->
<img src="/assets/agent-plasticity/chess_train_id_ood.gif" alt="Hard chess scores by checkpoint on training, held-out ID and held-out OOD games" width="100%">
<i>
Hard chess: score by checkpoint on training games, held-out ID games and held-out OOD games (a stronger Stockfish).
Gains against the stronger opponent are smaller and less consistent.
</i>
</p>

Improving within the training regime and transferring beyond it are different outcomes: gains against stronger held-out
OOD opponents are generally smaller or less consistent. What does improvement look like on the board? Below, the same
model plays the same Stockfish level at checkpoint 0 (empty inventory) and at checkpoint 15, after it has built and
patched its own chess engine.

<p class="ap-fig">
<!-- PRODUCTION: <img src="https://bair.berkeley.edu/static/blog/agent-plasticity/chess_replay_fable_vs_skill8.gif" alt="Chess replay: Claude Fable 5 against Stockfish skill 8 at checkpoints 0 and 15" width="100%"> -->
<img src="/assets/agent-plasticity/chess_replay_fable_vs_skill8.gif" alt="Chess replay: Claude Fable 5 against Stockfish skill 8 at checkpoints 0 and 15" width="100%">
<i>
Claude Fable 5 as White against Stockfish skill 8 (held-out ID). Checkmated on move 45 at checkpoint 0; at checkpoint
15 it holds a level game for 55 moves, then converts when Stockfish drifts. On average it scores 0% against this
opponent at checkpoint 0 and 40% over checkpoints 16–20.
</i>
</p>

The phenomenon extends beyond chess, but it is task-dependent. In Go, Claude Fable 5's held-out ID score rises from 20%
to 80%; in Hex, GPT-5.6 Sol improves from 40% to 77.5% on held-out ID games. But Claude Opus 4.8 loses ground in Go
while improving substantially in Hex, and Claude Opus 5 improves substantially in Go before a tool-making error at the
final checkpoint makes its score drop sharply.

**NetHack** is the longest-horizon test. At every checkpoint, each agent plays the same ten fresh, unseen games, which
are scored before reflection uses them. Over 40 checkpoints, only Claude Opus 5.5 shows a clear, persistent gain: its
mean score grows from about 2,000 over checkpoints 0–2 to about 6,000 over checkpoints 36–40, saturating at checkpoint
11. Claude Opus 5 and Claude Sonnet 5 rise more modestly, Claude Opus 4.8 stays roughly flat, and GPT-5.6 Luna declines.
So does GPT-5.6 Sol (about 570 to 420), the most plastic model on the board games.

<p class="ap-fig">
<!-- PRODUCTION: <img src="https://bair.berkeley.edu/static/blog/agent-plasticity/nethack_median.png" alt="NetHack median score by checkpoint for six models" width="100%"> -->
<img src="/assets/agent-plasticity/nethack_median.png" alt="NetHack median score by checkpoint for six models" width="100%">
<i>
NetHack median score (log scale) over checkpoints 0–40, ten games per checkpoint. Lines are smoothed; faint markers are
single checkpoints.
</i>
</p>

One persistent fix makes this concrete. In NetHack, praying heals you only if you are in serious trouble and have not
prayed recently. At checkpoint 1, Claude Opus 5.5 prays too soon through raw keystrokes and is killed. Reflecting on that
game, it adds a rule to its CLAUDE.md: pray only through its own controller, which checks whether prayer is safe. At
checkpoint 10, in another low-health emergency, it waits until the controller allows a prayer (8 of 94 hit points) and is
fully healed.

<p class="ap-fig">
<!-- PRODUCTION: <img src="https://bair.berkeley.edu/static/blog/agent-plasticity/nethack_replay_checkpoint1.gif" alt="NetHack replay at checkpoint 1: the fatal prayer" width="100%"> -->
<img src="/assets/agent-plasticity/nethack_replay_checkpoint1.gif" alt="NetHack replay at checkpoint 1: the fatal prayer" width="100%">
<i>
Checkpoint 1: Claude Opus 5.5 prays successfully on turn 203; on turn 325, low on health again, it prays through raw
keystrokes too soon after the last prayer and is killed (final score 576).
</i>
</p>

<p class="ap-fig">
<!-- PRODUCTION: <img src="https://bair.berkeley.edu/static/blog/agent-plasticity/nethack_replay_checkpoint10.gif" alt="NetHack replay at checkpoint 10: prayers through the controller" width="100%"> -->
<img src="/assets/agent-plasticity/nethack_replay_checkpoint10.gif" alt="NetHack replay at checkpoint 10: prayers through the controller" width="100%">
<i>
Checkpoint 10: with the new rule in place, prayers go through the controller; both heal it, and the game runs until the
time limit with 25,822 points.
</i>
</p>

<div class="ap-finding">
<b>Finding 1. Persistent self-improvement is model- and task-dependent.</b> A common interface for feedback, reflection
and artifact construction produces sharply different learning curves in all four games, and transfer to stronger
opponents is not uniform. This motivates evaluating plasticity across a distribution of agentic tasks rather than
treating it as a task-independent model property.
</div>

## Finding 2: Acquisition efficiency differs from endpoint capability

Comparing agents after the same number of update rounds hides the fact that those rounds cost very different amounts.
Plotting held-out performance against learning cost (the figure at the top of this post) separates two questions: how
good does the agent end up, and how quickly does experience pay off?

On the combined chess, Go and Hex curve, GPT-5.6 Sol has the highest estimated $P_{\text{sat}}$ at 298 points per \\$1,000
of learning cost, followed by Claude Opus 5 at 233, Claude Fable 5 at 57 and Claude Opus 4.8 at 14; GPT-5.6 Luna shows
no reliable rise, and GPT-5.5 is still improving when its runs end. Claude Fable 5 reaches the highest fitted
performance, but it saturates only after about \\$610 of learning cost, roughly five times the \\$120 at which GPT-5.6 Sol
levels off. GPT-5.6 Sol also starts below Claude Opus 4.8, yet ends above it in both held-out performance and estimated
plasticity.

These estimates depend on provider prices and on the saturation fit, so we read plasticity as a measurement tied to this
protocol, not a universal ranking of models.

<div class="ap-finding">
<b>Finding 2. Endpoint capability and acquisition efficiency answer different questions.</b> Claude Fable 5 reaches the
highest performance on the combined-game curve, while GPT-5.6 Sol has the largest estimated plasticity.
</div>

## Finding 3: Where does the self-improvement process break down?

Learning curves tell us whether an agent improves, not why it does not. So we trace every identified decision failure
through the artifact lifecycle: did a covering artifact exist, was it actually used (read or executed), and did the
failure happen anyway?

<p class="ap-fig">
<!-- PRODUCTION: <img src="https://bair.berkeley.edu/static/blog/agent-plasticity/reuse_relevant_decisions.png" alt="Share of relevant chess decisions reused successfully, reused with failure, or missed, per model" width="100%"> -->
<img src="/assets/agent-plasticity/reuse_relevant_decisions.png" alt="Share of relevant chess decisions reused successfully, reused with failure, or missed, per model" width="100%">
<i>
Relevant decisions in Hard chess (held-out ID games, checkpoints 1–20 pooled): moves where at least one saved artifact
was relevant, split into reused successfully, reused but the failure remained, and missed (no artifact used).
</i>
</p>

**Low artifact reuse accompanies weak improvement.** Pooled over Hard and Easy chess, GPT-5.6 Luna makes about 98% of
its relevant decisions without using any artifact, Gemini 3.1 Pro about 87%, Claude Opus 4.6 about 51% and GPT-5.5
about 28%, compared with 2–6% for Claude Fable 5, Claude Opus 5, Claude Opus 4.8 and GPT-5.6 Sol. Across 19 model–game
cells in chess, Go and Hex, held-out reuse correlates with held-out gains. Writing down an instruction is also not the
same as following it: in one GPT-5.5 run, reflection keeps telling the agent to load its saved move-picking program right
away, yet the agent still spends its first steps listing and reading files.

**High reuse does not explain the remaining gap.** Claude Fable 5, Claude Opus 5, Claude Opus 4.8 and GPT-5.6 Sol all
reuse artifacts at 94–98% of relevant chess decisions, yet reach substantially different game scores. **And strong
improvers still fail when using artifacts:** they make fewer mistakes overall (about 0.16 identified failures per
relevant decision, against 0.38 for the other models), but 83–99% of their remaining failures happen while a relevant
artifact is in use.

<p class="ap-fig">
<!-- PRODUCTION: <img src="https://bair.berkeley.edu/static/blog/agent-plasticity/reuse_identified_failures.png" alt="Share of identified failures by whether a covering artifact was absent, unused, or used" width="100%"> -->
<img src="/assets/agent-plasticity/reuse_identified_failures.png" alt="Share of identified failures by whether a covering artifact was absent, unused, or used" width="100%">
<i>
Identified failures in the same games, by where the artifact lifecycle broke down: no covering artifact, artifact
available but not used, or artifact used and the failure remained.
</i>
</p>

<div class="ap-finding">
<b>Finding 3. Artifact reuse is associated with improvement, but is not a proxy for capability.</b> High reuse may still
leave significant failures, hinting at problems with artifact quality, generality, or use. These diagnostics are
observational: they suggest candidate bottlenecks rather than establishing causes.
</div>

## What gets carried forward

What does an agent actually carry forward? One Claude Fable 5 lineage writes its own chess engine, without outside
libraries, and patches it after specific mistakes: adding a search for moves that escape check after its king is left
under attack, changing how it handles repeated positions after throwing away winning games, and giving the engine more
search time in endgames. After a game is scored 0 at the 300-ply move limit, it makes its engine value draws more as the
limit nears; at checkpoint 18 the engine holds a draw until Stockfish errs, then mates.

<p class="ap-fig">
<!-- PRODUCTION: <img src="https://bair.berkeley.edu/static/blog/agent-plasticity/chess_replay_300ply_lesson.gif" alt="Chess replay: a checkpoint-6 game lost at the move limit, and a checkpoint-18 game won after the fix" width="100%"> -->
<img src="/assets/agent-plasticity/chess_replay_300ply_lesson.gif" alt="Chess replay: a checkpoint-6 game lost at the move limit, and a checkpoint-18 game won after the fix" width="100%">
<i>
A fix that persists after a failure. Left: a checkpoint-6 training game reaches the 300-ply limit and is scored 0.
Right: at checkpoint 18 the engine holds its draw values until Stockfish errs with 124...Rb8, then mates.
</i>
</p>

<p class="ap-fig">
<!-- PRODUCTION: <img src="https://bair.berkeley.edu/static/blog/agent-plasticity/fixes_that_persist.png" alt="Paper Figure 8: fixes that persist after a failure, in chess and NetHack" width="100%"> -->
<img src="/assets/agent-plasticity/fixes_that_persist.png" alt="Paper Figure 8: fixes that persist after a failure, in chess and NetHack" width="100%">
<i>
Fixes that persist after a failure (Figure 8 of the paper), in chess (top) and NetHack (bottom).
</i>
</p>

## Looking ahead

Static evaluations measure where an agent stands. For agents meant to accumulate experience, how efficiently they
improve matters as much as where they start. Our results suggest three practical lessons: measure the slope, not just
the endpoint, since the best final score and the best learning efficiency can belong to different models; check that
artifacts are actually used, for example by loading or invoking them automatically; and once reuse is frequent, focus on
artifact quality and application, validating whether an artifact helps rather than only whether it runs.

Plasticity may itself be a target for post-training. A natural next step is to **evaluate and hill-climb plasticity on a
distribution of useful agentic tasks**: building agents that are not only more capable, but better at becoming more
capable.

*Limitations.* Frozen weights and fresh contexts isolate persistent adaptation, but do not separate the value of
environment feedback from the extra computation spent developing artifacts. Learning cost uses provider prices as a proxy
for compute, the reuse diagnostics are observational, and plasticity estimates depend on pricing and on the saturation
fit. Small evaluation sets add checkpoint noise, and separately trained runs do not establish cross-task transfer.

---

This post is based on the paper [Agent Plasticity: Measuring Self-Improvement Through
Experience](https://arxiv.org/abs/2610.08902). An interactive version of this post, with game replays you can step
through, is available [here](https://harmandotpy.github.io/agent-plasticity/). $^\dagger$Joint supervision.

```
@article{singh2026agentplasticity,
  title   = {Agent Plasticity: Measuring Self-Improvement Through Experience},
  author  = {Singh, Harman and Bakhtin, Anton and Shao, Rulin and
             Synnaeve, Gabriel and Kulikov, Ilia and Fergus, Rob and
             Arora, Sanjeev and Keutzer, Kurt and Weston, Jason and
             Mahajan, Anuj and Goyal, Anirudh},
  journal = {arXiv preprint arXiv:2610.08902},
  year    = {2026}
}
```
