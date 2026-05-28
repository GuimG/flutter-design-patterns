# Flutter Design Patterns

**🧩 Building resilient and modular applications at scale.**

Production-tested architectural patterns for Flutter apps that need to work offline, scale across dozens of feature modules, and keep their codebase navigable as the team grows.

This is not a package you install. It's a reference — **read it, understand the trade-offs, adapt the code to your architecture**.

## Table of contents

|     | Pattern                                                          | Problem it solves                                               |
| --- | ---------------------------------------------------------------- | --------------------------------------------------------------- |
| 🔌  | [Dependency Inversion](docs/patterns/dependency-inversion.md)    | Concrete SDKs are scattered across the codebase.                |
| 📡  | [In-Process Domain Event Bus](docs/patterns/domain-event-bus.md) | Features depend on each other, creating a circular import web.  |
| ⚡  | [Actions](docs/patterns/actions.md)                              | Imperative one-shot operations don't fit `AsyncValue`.          |
| 🔁  | [Retry with Backoff](docs/patterns/retry.md)                     | Transient network blips surface to the user as hard failures.   |
| 📦  | [Outbox](docs/patterns/outbox.md)                                | Writes that absolutely must reach the server keep getting lost. |

## Contributing

Have a pattern that saved your team 40 hours of refactoring? We want it!

- Check out [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.
- Open an issue to discuss a new pattern before submitting a PR.

## License

[MIT](LICENSE) — use it however you want.
