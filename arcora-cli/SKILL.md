---
name: arcora-cli
description: Arcora provides collaboration capabilities for AutoCAD drawings. Use arcora-cli to inspect, view, and edit drawings.
---


# Arcora Overview

Arcora homepage: https://www.arcora-ai.com/
Arcora enables collaborative viewing and editing for AutoCAD drawings. It consists of three main parts:

  - `arcora-plugin`: A plugin that runs inside AutoCAD. It connects the currently opened drawing to the Arcora collaboration environment and syncs collaboration results back to AutoCAD.

  - `arcora-server`: The Arcora collaboration service. It manages collaboration sessions, stores the collaborative drawing state, and syncs changes from different participants to others.

  - `arcora-cli`: A command-line tool for agents. Agents use it to read the current drawing collaboration context, inspect drawing content, and submit edits.

# How To Use arcora-cli

## Prerequisites

Before using `arcora-cli`, complete the following setup:

1. Install `arcora-cli`

   `arcora-cli` is published as the npm package `@arcora_ai/arcora-cli`. It requires Node.js `>=20` and access to the npm registry.

   ```bash
   npm install -g @arcora_ai/arcora-cli
   ```

2. Verify the installation

   ```bash
   arcora-cli help
   ```

   If the help command prints structured JSON successfully, the CLI is installed correctly. Treat this help output as the source of truth for supported commands, usage strings, required flags, and examples.

3. Obtain the shared link

   Obtain the shared link from the AutoCAD drawing you want to work with. The shared link allows `arcora-cli` to connect to the corresponding collaborative drawing.

   Note: shared links have an expiration time. If the connection fails or the link has expired, obtain a new shared link from the drawing.

4. Obtain the user token

   The user token is the unique identifier for the user's Arcora identity. Request a user token by emailing Arcora.

   The user token can be stored in a system environment variable. By default, `arcora-cli` reads the token from `ARCORA_USER_TOKEN`.

## Basic Workflow

This workflow defines the basic working order. Adjust it flexibly based on the user's request. Decide the concrete commands according to the capabilities shown by `arcora-cli help`.

1. Discover CLI capabilities

   Run the help command first and parse its structured JSON. Use the discovered command names, usage strings, required flags, and examples to decide the exact invocation for the current installed CLI version.

2. Check the session state

   If there is no running session, start one. If a session already exists, reuse the current session instead of starting another one.

3. Start the session when needed

   Use the shared link from the user and deliver the token through the safest supported method.

4. Confirm that the drawing is available

   Check status again after starting or reusing a session. Confirm that the session is running, startup is ready, and the drawing mirror is ready. If status is unavailable, startup failed, or the mirror is not ready, stop further operations and explain the issue.

5. Read the collaboration context

   This context provides basic information about the current drawing, layers, spaces, available owners, and related context.

6. Read the drawing content

   The agent must make decisions based on the latest drawing content, not on stale context or assumptions.

7. Discover schemas before editing

   When the task requires edits or detailed validation, use the schema commands to inspect the current document response schema and DSL plan schema before generating an edit plan.

8. Decide whether to edit based on the user's request

   If the user only asks to view, analyze, or ask questions about the drawing content, respond based on the retrieved result. If the user requests drawing edits, first generate a clear edit plan that conforms to the discovered DSL schema.

9. Submit the edit plan

   Write edit operations into a DSL plan file, then submit it through the apply-DSL command. A successful submission only means the edit has been submitted. It does not mean the edit has fully synced.

10. Wait for the edit result to converge

   Prefer waiting for the last submitted operation when appropriate, or wait by operation id when the operation id is known. Only after convergence completes can the agent treat the edit as having a definitive result.

11. Read the drawing again and confirm the result

   Read the drawing again after convergence. Report to the user based on the final retrieved drawing result, not merely on the submitted plan.

12. Stop the session when necessary

   If the user explicitly ends the current collaboration operation, or if you need to switch to another drawing, stop the current session by using the stop-session command. Maintain only one session and one drawing at a time.

## Operational Rules

Agents must follow these rules during execution:

- Treat the help output of the installed CLI as the command source of truth. Do not assume this skill contains the exact command syntax for every version.
- Always parse CLI output as JSON. Successful commands emit a single JSON object. Errors emit a JSON object with `error.code`, `error.message`, and `error.retryable`.
- Always read the latest drawing content before each edit.
- After submitting an edit, always wait for convergence before confirming the final result.
- If the session is not started, startup is not ready, the drawing mirror is not ready, the shared link is invalid, the target object does not exist, the DSL plan is invalid, or the CLI returns an error, stop immediately and explain the issue to the user.
- Agents must not construct low-level protocol data directly and must not bypass `arcora-cli` when operating on drawings.

## DSL Planning Rules

Use the DSL schema command discovered from help as the source of truth. The current CLI supports controlled drawing edits such as creating, updating, and deleting common drawing entities, including lines, circles, arcs, polylines, DBText, MText, and block references.

When planning edits:

- Prefer stable `globalId` references from the latest document output.
- For create actions, let the CLI generate `globalId` unless the task requires a specific one.
- For update and delete actions, only target entities that exist in the latest mirror and match the intended entity type.
- Prefer explicit placement by global id for layers and owner block table records. If using names, they must resolve uniquely.
- Be careful with block references: creating a block reference requires a resolvable referenced block table record and may require an existing template instance for attributes and dynamic properties.
- Keep large edits within the CLI safety limits. Split large plans into smaller batches and wait for convergence after each batch.

The local document mirror is not the authoritative server state. Treat the final document read after convergence as the confirmation point.
