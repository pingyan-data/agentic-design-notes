# Agentic Design Notes

Personal notes on designing multi-agent systems for regulated 
financial workflows. Each document is a standalone design 
exploration, not production code.

## Notes

| Topic | Domain | Status |
|---|---|---|
| [Sanctions Screening Co-pilot](sanctions-screening.md) | Compliance | Draft |
| [Fraud Investigation Co-pilot](fraud-investigation.md) | Financial Crime | Draft |

## Companion code

A working multi-agent prototype implementing similar patterns 
for consumer credit decisions is at 
[agentic-credit-decisions](https://github.com/pingyan-data/agentic-credit-decisions).

## Why these notes exist

GenAI in regulated environments has a different design 
discipline than consumer-facing AI. These notes are an attempt 
to think through that discipline concretely, one domain at a 
time — what to automate, what to leave to humans, where the 
audit trail lives, how to evaluate without ground truth.