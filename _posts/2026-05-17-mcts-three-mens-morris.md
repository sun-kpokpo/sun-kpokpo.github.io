---
layout: post
title: "The Minimax vs MCTS Paradox: Why Approximation Seems to 'Beat' Perfection"
date: 2026-05-17 12:00:00
description: "An analysis of a counter-intuitive behavior observed in an AI agent tournament on Three Men's Morris."
tags: [game-theory, algorithms, AI, python, reinforcement-learning]
categories: research
math: true
---

## Introduction

Three Men's Morris is a minimalist board game — 3 pieces per player, 9 positions,
win by aligning 3 in a row. Despite its apparent simplicity, its state space makes it an ideal benchmark:
small enough for exact methods like Minimax to solve completely, yet large enough
to expose the weaknesses of approximate methods like Q-learning.

We pitted three agents against each other over 200 games each: **MCTS** (Monte Carlo
Tree Search, 1000 simulations per move), a **Q-table** agent trained with UCB
exploration, and **Minimax** with a full transposition table. The results were
largely expected — except for one number that demands explanation: MCTS wins only
86.5% of its games against Minimax. How can a theoretically optimal agent lose
13% of the time to a simulation-based approximation?

## 1. Minimax and MCTS vs Q-table: The Implacability of Determinism

Minimax achieves a **100% win rate** against the Q-table agent. This is theoretically
expected: Minimax explores the complete game tree exhaustively and always selects
the provably optimal move, leaving no exploitable gap for a Q-learner that hasn't
fully converged on such a sparse state space.

MCTS, by contrast, scores **94%** against the same Q-table — meaning it loses 6%
of the time. Those losses are not a sign of weakness; they reveal something
fundamental about MCTS's probabilistic nature. With only 1000 simulations per move,
MCTS doesn't guarantee optimal play — it approximates it. In rare game states where
the Q-table agent stumbles into a sequence that MCTS undersampled, the approximation
fails. This is the price of trading exactness for scalability.

## 2. The 86.5% "Paradox": An Asymmetric Optical Illusion

Three Men's Morris is **not a fair game**. Like most alignment games on small boards,
the first player holds a significant structural advantage — more opening options,
more initiative, more forcing lines. This asymmetry is critical to interpreting
the tournament results.

In our setup, roles were not alternated symmetrically across all 200 games. This
means the win rate between MCTS and Minimax is partially a reflection of **who played
first**, not just who played better. When MCTS holds the first-player advantage,
it wins comfortably. When Minimax moves first against a near-optimal opponent,
the margin shrinks dramatically.

There is a second, subtler effect. When Minimax finds itself in a losing position
against what it evaluates as a near-perfect opponent, it **does not fight back
aggressively**. Its evaluation function recognizes the loss as inevitable and
plays passively — it has no incentive to complicate the position or set traps,
because its tree search already assumes the opponent will respond optimally.
MCTS, however, is not perfect. It occasionally misses the refutation of a
passive trap. The result: Minimax's "resignation to loss" accidentally generates
positions that MCTS mishandles.

The 86.5% figure is therefore not a paradox — it is an asymmetric optical illusion
produced by role assignment bias and MCTS's sampling imperfection under passive play.

## 3. Results

{% include figure.liquid path="assets/img/posts/mcts_results.png" caption="Algorithm Comparison — Three Men's Morris (1000 sims MCTS)" %}

The chart confirms the hierarchy clearly: Minimax and MCTS both dominate the Q-table,
with Minimax achieving perfection and MCTS falling just short. The MCTS vs Minimax
matchup is the only one that produces a non-trivial loss rate — and as argued above,
that loss rate is as much a product of tournament design as of algorithmic performance.
The Q-table's consistent underperformance across all matchups suggests that tabular
Q-learning, without sufficient exploration and convergence time, is simply outclassed
by both search-based methods on this problem.

## Conclusion

The main lesson here is one that applies broadly to AI benchmarking: **a win rate
is only as meaningful as the experimental design that produced it**. Raw numbers
on a graph can mislead if the role assignment, the number of simulations, and the
convergence state of learned agents are not accounted for. The 86.5% figure looked
like a paradox until we unpacked the asymmetry hiding inside it.

More generally: Minimax remains the method of choice when the state space is small
and exact computation is feasible. The moment the game tree explodes — Go, real-time
strategy, or physical robot manipulation — exact search becomes intractable and we
are forced to approximate. MCTS scales gracefully into that regime. And when even
MCTS struggles with the curse of dimensionality, we reach for Deep Reinforcement
Learning — which trades the hand-crafted search tree for a learned value function
that generalizes across states. Each method has its domain. The art is knowing
which tool fits the problem.
