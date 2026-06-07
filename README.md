# Contribution #1: ImportFeeds — catch symlink failures on Windows (beets #840)

**Contribution Number:** 1  
**Student:** Temitope S. Olugbemi  
**Issue:** https://github.com/beetbox/beets/issues/840  
**Status:** Phase I — In Progress

---

## Why I Chose This Issue

I chose issue [#840 "ImportFeeds: fix/disable/catch symlink usage on windows"](https://github.com/beetbox/beets/issues/840) in [beetbox/beets](https://github.com/beetbox/beets) because it is a real, well-scoped bug in Python — one of the languages I work in most — and it lets me practice exactly the skills I want to build in this program: reading an unfamiliar, mature codebase, fixing a defect cleanly, and writing a regression test that proves the fix. It is labeled `bug` and `good first issue`, and the maintainer already described the intended fix in the thread ("we should catch and skip this"), so the acceptance criteria are clear.

I'm interested in this because:

1. **It's squarely in my skill set.** The fix is a focused change to a single Python plugin file (`beetsplug/importfeeds.py`) plus a test — no framework or domain knowledge I'd have to ramp up from scratch.
2. **The scope is contained.** It's a small, bounded defect (a missing `try/except` around one filesystem call), not an open-ended feature — a realistic first contribution.
3. **The project is active and the area is well-supported.** beets is pushed to almost daily, has a thorough `CONTRIBUTING.rst`, and there is a recently merged, nearly identical fix (the fetchart `OSError` handling in PR #6662) that I can use as a template for the fix, the test, and the changelog entry.
4. **It matches my learning goals.** I want to get comfortable with Python error handling and filesystem edge cases, with mock-based regression testing, and with the full open-source PR/review workflow — and this issue exercises all three.

From reading the issue thread and the code, I understand the current problem: when the importfeeds plugin is configured with `formats: link`, a failure to create a symlink (for example on Windows, where creating one requires elevated privilege) raises out of `beets.util.link()` and **aborts the entire `beet import` run** — even though the music files were already imported. My contribution will catch that failure, log a clear per-item warning, and let the import continue, making the plugin resilient on Windows instead of crashing.

I have already left a comment on the issue introducing myself as a CodePath AI301 student, summarizing the fix I plan to make, and asking the maintainer a scoping question (whether to catch `beets.util.FilesystemError` — what the helper actually raises — versus a broader `OSError`).

---

## Understanding the Issue

### Problem Description

[In your own words, what's broken or missing?]

### Expected Behavior

[What should happen?]

### Current Behavior

[What actually happens?]

### Affected Components

[Which parts of the codebase are involved?]

---

## Reproduction Process

### Environment Setup

[Notes on setting up your local development environment - challenges you faced, how you solved them]

### Steps to Reproduce

1. [Step 1]
2. [Step 2]
3. [Observed result]

### Reproduction Evidence

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

---

## Solution Approach

### Analysis

[Your analysis of the root cause - what's causing the issue?]

### Proposed Solution

[High-level description of your fix approach]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** [Restate the problem]

**Match:** [What similar patterns/solutions exist in the codebase?]

**Plan:** [Step-by-step implementation plan]
1. [Modify file X to do Y]
2. [Add function Z]
3. [Update tests]

**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]

**Evaluate:** [How will you verify it works?]

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
