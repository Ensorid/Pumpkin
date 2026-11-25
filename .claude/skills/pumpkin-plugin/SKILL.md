---
name: pumpkin-plugin
description: Write Pumpkin plugins (Minecraft server in Rust) compiled to WebAssembly with the `pumpkin-plugin-api` crate. Use it to initialize a plugin or to add a command to an existing plugin.
---

# Pumpkin plugins (Rust + WASM)

Entry point of the skill. Read this file first, then open only the file that matches the task.

## Overview

- A plugin is a **WebAssembly component** (`.wasm`) loaded by the server from the `./plugins` folder.
- It is written in Rust with the `pumpkin-plugin-api` crate (MIT OR Apache-2.0, separate from the GPL-3.0 server code).
- A plugin is a type that implements the `Plugin` trait, registered with the `register_plugin!` macro.
- The API definitions live in `crates/pumpkin-plugin-wit` (`.wit` files, versions `v0.1` and `v0.2`). The `pumpkin-plugin-api` crate is the Rust SDK on top of them.

## Files in this skill

| File | When to read it |
|---|---|
| [initialize-plugin.md](initialize-plugin.md) | Create a new plugin: Cargo project, metadata, `on_load`, build, installation. |
| [create-command.md](create-command.md) | Add a command (with subcommands and arguments) to a plugin. |

## Rules

- Never copy server code (GPL-3.0) into a plugin: go through `pumpkin-plugin-api` only.
- If the repository code and these files disagree, the repository code wins: fix the skill file.
- Each file lists unverified points in a "To verify" section.
