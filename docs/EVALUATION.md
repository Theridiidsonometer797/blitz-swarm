# Evaluation

## Scoring System

The quality judge agent scores each round's collective output on 4 dimensions (0-10):

| Metric | Definition | What It Does NOT Capture |
|--------|-----------|--------------------------|
| Coverage | Breadth of topic dimensions addressed | Whether important dimensions were missed entirely |
| Accuracy | Correctness of technical claims | Subtle errors the judge LLM may not catch |
| Clarity | Readability and coherence of output | Whether the structure aids comprehension for specific audiences |
| Depth | Technical depth of analysis | Whether depth is proportional to importance |

## Consensus Metrics

| Metric | Definition |
|--------|-----------|
| Rounds to convergence | Number of blast rounds before unanimous "ready" |
| Ready votes per round | Count of agents voting "ready" |
| Avg confidence per round | Mean of agent confidence scores (0.0-1.0) |
| Override applied | Whether holdout was overridden after 3+ rounds |

## Automatic vs. Human Metrics

All metrics are automatic. No human evaluation has been conducted. No inter-rater reliability exists.

## Current Data

With 2 runs, no aggregate statistics are meaningful. See results/RESULTS.md for raw data.
