# KUtrace for Raspberry Pi 5

This fork ports [KUtrace](https://github.com/dicksites/KUtrace) to Raspberry Pi 5.

KUtrace is a low-overhead kernel/user tracer. It records every user↔kernel transition on every core and renders an interactive HTML timeline.

## Agent-assisted troubleshooting

With OpenClaw, an agent can monitor and troubleshoot performance issues in real time, and analyze the trace JSON without manual inspection.

- [`.claude/skills/kutrace-tracing/`](./.claude/skills/kutrace-tracing/SKILL.md) — skill that drives trace capture and post-processing.

Ask the agent to trace a workload; it runs the toolchain and produces the tracing.

## Docs

- [`RPI5_BUILD.md`](./RPI5_BUILD.md) — build and install.
- [`RPI5_TRACING.md`](./RPI5_TRACING.md) — capture a trace.
- [`linux/RPI5_KERNEL_PORTING.md`](./linux/RPI5_KERNEL_PORTING.md) — port hooks to another kernel.
- [`README.kutrace`](./README.kutrace) — original README.

## License

BSD-3 for most code; GPL-2.0 for the kernel module. See `LICENSE`.
