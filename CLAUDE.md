# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is the Claude Code CLI — a TypeScript terminal application using React/Ink for UI, Bun for building, and the Anthropic SDK for AI interaction. See [CODEBASE.md](./CODEBASE.md) for full architecture details.

## Key Conventions

- **Linter**: Biome. Respect `biome-ignore` directives. Custom ESLint rules enforce: no top-level side effects, no `process.env` at top level, no `process.exit`.
- **Feature gates**: Use `feature('FLAG')` from `bun:bundle` for build-time conditional code. This enables dead code elimination.
- **Lazy requires**: Use `() => require('./module.js')` pattern to break circular dependencies.
- **State**: Use `useAppState(selector)` hook pattern for accessing app state. Do not mutate state directly.
- **Large files**: `main.tsx` (804KB), `query.ts` (69KB), `print.ts` (213KB) — read targeted line ranges, not the whole file.
