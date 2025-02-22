---
layout: post
title:  "The only way to go well is to go fast"
author: kpapp
image: assets/images/fast.jpg
featured: true
hidden: false
categories:
    - Devops
    - Agile
tags:
    - automation
    - efficienty
    - best practices
---

Most readers might be familiar with Uncle Bob’s quote:

> “The only way to go fast, is to go well.”

This emphasizes that maintaining quality is crucial—if we let it slip, it will eventually slow us down. This article, however, explores human nature and how speed can impact development practices.

### The need for speed

Developers like to move fast. Agile frameworks even reward speed, reinforcing the drive to ship quickly. Missing a sprint goal due to delays can be frustrating. When a team member is stuck, we rush to unblock them because the longer they struggle, the more frustrated they become—and that’s when quality starts to suffer.

So how can we help teams deliver faster without compromising quality?

- Break down tasks into smaller, manageable chunks that fit within a sprint while still delivering value.
- Ensure acceptance criteria and specifications are crystal clear to eliminate ambiguity.
- Implement automation to reduce cognitive load across the software development lifecycle (SDLC).
- Establish clear code review guidelines and coding conventions to minimize friction.
- Use static analysis and automated testing to ease the burden on reviewers.
- Set up automated deployments and end-to-end (E2E) tests so that a pull request’s status is immediately clear.

But is that enough?

### A cautionary tale

Let me share a story about how a lack of speed triggered a destructive cycle.

Once upon a time, a moderately small web project—about 40 people strong—was approaching its release. The team worked across multiple repositories and services, with pipelines in place but limited E2E tests. Two weeks before launch, panic set in.

On Monday morning, the main branch was broken. The database or permission service seemed to be the culprit. After hours of investigation, the root cause emerged: a pull request merged on Friday had rendered the system unusable for two full days. Developers scrambled to fix the issue, wasting dozens of man-days. The problem was a concurrency issue introduced by a security-related change that was required for the release, making it impossible to revert the fix.

The PR contained over 80,000 lines of code and referenced two large, system-wide changes. Over time, the team had developed a habit of batching changes into massive PRs to avoid dealing with slow, unreliable pipelines. Instead of breaking work into smaller, manageable pieces, they combined everything into single, monstrous PRs. This practice was meant to reduce pipeline juggling, but it also made reviews nearly impossible. As a result, critical bugs slipped through. The very attempt to move faster by avoiding slow pipelines had backfired, leading to major slowdowns and last-minute firefighting.

What can be done to avoid these?

The core issue wasn’t the developers—it was the slow, flaky build and deployment process. Instead of pouring resources into new features, the team should have focused on optimizing their delivery pipeline.

Imagine a factory where workers assemble products quickly, but the conveyor belt keeps jamming, preventing items from leaving the factory. Wouldn’t fixing the conveyor belt be the top priority?

### Lessons learned

To truly go fast, teams must invest in:

- Reliable, fast CI/CD pipelines to remove deployment friction.
- Incremental development to prevent massive, unreviewable PRs.
- Comprehensive automation to catch issues early and reduce manual effort.
- A culture of quality where speed and correctness go hand in hand.

Going fast isn't just about writing code quickly—it's about optimizing the entire system so that speed and quality reinforce each other.

The only way to go well is to go fast—but only if you’ve built the right foundation.
