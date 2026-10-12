# 100+ Days of DevOps + SRE: Daily Build System

This is the **documentation repo** for a hands-on DevOps and SRE project: Kubernetes (EKS), Terraform, GitHub Actions, Helm, Argo CD, Prometheus, Grafana, Alertmanager, centralized logging, tracing (OpenTelemetry) and DevSecOps tooling (Gitleaks, Trivy, SonarQube).

The project itself lives here: https://github.com/Vivixell/eks-sre-observability

This repo holds the plan, the daily progress and the notes behind it.

## The idea

You don't become consistent by relying on motivation. You become consistent by building a **system**:
small daily tasks that build on each other, until a large project exists.

## The rules

- Do **one day's task per day**. Each takes about 30 minutes.
- You can do more, but extra work never replaces tomorrow. Tomorrow you show up again and continue from where you stopped.
- A "Day" is a small work package, not a calendar date. Days are numbered continuously.
- Work Monday to Friday. Use Sunday to review and plan the next week.

## How to build the same system for any topic

1. **Pick a daily time slot** and stick to it. Thirty minutes is enough to count as showing up.
2. **Pick what to master.** It can be one tool or a combination. List the topics in the order they should flow.
3. **Plan the first week.**
   - State exactly what each day must achieve, however small.
   - Each day builds on the previous day, and each week on the previous week.
   - Keep days very small and achievable. If you have more time, do the next day's task.
   - Every Sunday, plan the next week.
4. **Create a documentation repo** and write a short log for every day's work.
5. **Execute.**

## Tips for engineers (DevOps / SRE / software)

- **Pass 1: build with AI help.** Use AI as an assistant while you build the complete project. Focus on building, and use the daily build to track progress. Use the rest of your day to study courses that deepen your understanding.
- **Pass 2: rebuild without AI.** When the full project works, rebuild it following the same plan, slowly, using only official documentation and your notes. This is what prepares you for technical interviews.

## What is in this repo

| File / folder | Purpose |
|---|---|
| `master-plan.md` | Goals, rules, standards and the full roadmap |
| `weekly-plans/` | The detailed plan for each week, one file per week |
| `progress.md` | Current day, pass and status |
| `daily-logs/` | One log per completed day |
| `notes/` | Troubleshooting, concepts and reference notes |
| `new-ai-chat-handoff.md` | A short message for starting a new AI chat |

## How to use this with an AI

**Starting a new chat or account:** share three files.
1. `master-plan.md`
2. The current `weekly-plans/week-NN-plan.md`
3. `progress.md`

Then add a short message:

```
Continue my DevOps/SRE daily build challenge. Use master-plan.md as the source of truth,
the weekly plan for today's task, and progress.md for my position.
I'm on Day [N], Pass [1 or 2]. Help only with this day's work package.
```

**In an ongoing chat:** share only the day, the pass, what you tried and the exact error. Re-share the three files only if the chat loses context.

**Pass 1:** the AI explains and helps you build.
**Pass 2:** the AI gives hints and documentation pointers only. You attempt it first.

## Adapting it to your own topic

Copy the structure and replace the content:
1. Write your own `master-plan.md` (goal, rules, roadmap split into 30-minute days).
2. Write a detailed plan for your first week.
3. Create `progress.md` and track the days.
4. Start logging in `daily-logs/`.

## Status

See `progress.md` for the current day.