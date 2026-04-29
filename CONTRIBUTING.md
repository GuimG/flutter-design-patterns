# Contributing to Flutter Design Patterns

Thanks for considering a contribution. This repo thrives on real-world experience — if you've solved an architectural problem in Flutter that others keep hitting, this is the place to share it.

## What Makes a Good Contribution

### New patterns

A pattern belongs here if it meets **all three** criteria:

1. **It solves a recurring architectural problem** in Flutter apps (not a widget trick or a one-off hack)
2. **You've used it in production** and can speak to the trade-offs honestly
3. **It's not already well-documented elsewhere** (don't duplicate the Riverpod docs or the Bloc tutorial)

### Improvements to existing patterns

- Corrections to technical inaccuracies
- Additional trade-offs or limitations you've discovered in practice
- Code improvements with clear rationale
- Real-world case studies ("we used this pattern in our app and here's what happened")

### What we don't accept

- Theoretical patterns without production experience
- Framework-specific tutorials (this is about architecture, not "how to use Hive")
- Marketing or promotional content

## Pattern Document Structure

Every pattern document follows this structure. Please maintain it for consistency:

```
# Pattern Name

## 1. The Problem
What goes wrong without this pattern? Be specific and practical.

## 2. The Pattern
Formal name, origin, how it works in the abstract.
Include the mapping table from the theoretical concept to the Flutter equivalent.

## 3. Flutter Approach
How it integrates with Clean Architecture layers.
Which layer owns what. What changes, what stays the same.

## 4. Code
Full, production-ready Dart code with file paths and imports.
Not snippets — complete files that someone can read top to bottom.

## 5. Key Design Decisions
The "why" behind non-obvious choices.
Be honest about trade-offs — every pattern has them.
```

## How to Submit

### For small changes (typos, clarifications, minor code fixes)

1. Fork the repo
2. Make your changes
3. Open a pull request with a clear description of what you changed and why

### For new patterns

1. **Open an issue first** describing the pattern you want to add
   - What problem does it solve?
   - Where have you used it?
   - Why isn't it already covered by an existing pattern?
2. Wait for feedback before writing the full document
3. Fork, write the pattern following the structure above, and open a PR

### For significant changes to existing patterns

1. **Open an issue first** explaining what you want to change and why
2. Include evidence (production experience, bug reports, benchmarks)
3. Fork, make changes, open a PR

## Code Style

- Dart code should follow the [Effective Dart](https://dart.dev/effective-dart) guidelines
- Use meaningful variable names, not `x`, `temp`, `data`
- Include file paths as comments at the top of each code block
- Include imports — readers should be able to understand dependencies at a glance
- Prefer explicit types over `var` in documentation code (clarity over brevity)

## Writing Style

- Write for an experienced Flutter developer who hasn't seen this pattern before
- Explain the "why" before the "how"
- Use concrete examples from real apps, not abstract FooBar examples
- Be honest about limitations — every pattern has trade-offs
- Use proper terminology but explain it when you first introduce it
- No promotional language — let the pattern speak for itself

## Review Process

1. A maintainer will review your PR within a week
2. We may ask for revisions — this is collaborative, not adversarial
3. Once approved, we'll merge and credit you as a contributor

## Code of Conduct

Be respectful, be constructive, be honest. We're all here to learn.

## Questions?

Open an issue with the `question` label. We're happy to help you figure out if your pattern is a good fit before you invest time writing it up.
