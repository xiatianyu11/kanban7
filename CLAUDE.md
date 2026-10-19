# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file IT PMO Kanban board (`index.html`) for a **fictitious** bank, used as an internal demo and training tool. Keep all naming neutral: the "IT PMO" wordmark, `ITPM-####` task IDs and `[IT PMO]` email subjects. Never use a real bank's name, logo or trademarks, or imitate an official system.

## Running

There is no build, no package manager and no test suite. Open `index.html` directly in a browser by double-clicking it, or on Windows with `Start-Process index.html`. It must keep working from `file://` without a server.

## Hard constraints (from the original spec)

- **Everything stays in one file.** Use vanilla HTML/CSS/JS only: one `<style>` block and one `<script>` block. No frameworks, CDNs, web fonts, image files or other external resources. Icons are inline SVG or Unicode.
- **No persistence.** Don't use localStorage, sessionStorage, IndexedDB or cookies. A refresh resets the board to the seed data on purpose, and the header shows a note saying so.
- **No native dialogs.** Don't use `alert()` or `confirm()`. Validation errors appear inline under each field, and deletes are confirmed inside the card ("Delete? Yes / No").
- **No `!important`.** Colours and spacing come from CSS custom properties on `:root`.
- **The only network call is to FormSubmit.** `notifyNewTask()` posts to `FORMSUBMIT_ENDPOINT`, a config constant at the top of the script and the one place to change the email address. Each new address needs a one-time activation: the first submission sends a confirmation email instead of delivering. A failed FormSubmit call must never break the board. The task is added optimistically, and a failure shows only the warning toast "Card added locally — email notification failed".

## Architecture

- **One source of truth.** The `state` object holds `tasks`, `filters`, `nextId`, plus UI flags: `pendingDelete`, `openMoveMenu`, `focusAfterRender` and `sending`.
- **Re-render from state.** Every change goes through an action function (`addTask`, `moveTask`, `deleteTask`, filter handlers), which updates `state` and then calls `renderBoard()`. `renderBoard()` rebuilds all four columns from `applyFilters(state.tasks)`, then runs `renderSummary()` and `restoreFocus()`.
- **Card contents come only from `renderBoard()`/`renderCard()`.** Don't patch card DOM directly. The open Move menu and the delete confirm are state flags that the render shows, not DOM toggles.
- **Escape user strings.** `renderCard()` returns an HTML string, so every user-supplied value must go through `escapeHtml()`. Toasts set `textContent` directly.
- **Events are delegated.** Click and drag listeners are attached to `#board` and `document` once in `init()`, so they survive re-renders.
  - Drag and drop uses the native HTML5 API, with the task id carried in `dataTransfer`.
  - The `.drag-over` class marks the column being hovered.
- **Keyboard focus survives re-renders.** Set `state.focusAfterRender = { id, action }` before rendering. `restoreFocus()` then focuses `[data-action=…]` inside that card.
- **Dates are local-time `YYYY-MM-DD` strings.** They're compared as strings, so use `todayISO()` and don't use `toISOString()`, which would shift the day under UTC. Seed due dates are offsets from today, so the Overdue badges always show.
- **Status, project, category and priority lists** are the constants `STATUSES`, `PROJECTS`, `CATEGORIES` and `PRIORITIES`. They feed the board columns, the selects and validation.
- **Accessibility requirements:**
  - visible `:focus-visible` rings
  - an `aria-label` on every icon-only button
  - `aria-live="polite"` on the toast region
  - priority shown as text as well as colour
  - a `<label for>` on every input
  - columns stack below 768px
