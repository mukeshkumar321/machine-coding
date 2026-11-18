# Frontend Machine Coding Interview Series 🚀

A structured repository for developers preparing for **frontend machine coding rounds** (React + hooks, 4 years of experience level). Every question has the problem statement, requirements, a design approach, a complete reference solution, edge cases, accessibility, performance, follow-ups, and a practice checklist.

This repository is designed for:

- Frontend developers preparing for machine coding / UI design rounds
- Engineers moving from 2–3 YOE to senior roles
- Anyone who wants a repeatable way to build UI components in 45–90 minutes

## 🎯 What You Will Learn

- Breaking down a vague UI problem into requirements and components
- State design: lifting state, `useReducer`, derived state, immutable updates
- Async UI: debounce, race conditions, optimistic updates, polling, infinite scroll
- Reusable hooks and component APIs (controlled vs uncontrolled, composition)
- Accessibility: ARIA roles, keyboard support, focus management
- Performance: memoization, observers, debounce, avoiding re-renders
- Common utilities: debounce, custom hooks, optimistic updates

## ✅ File Template

Every question file follows the same structure:

1. Problem Statement
2. Requirements (Functional / Non-functional)
3. Clarifying Questions to Ask
4. Approach & Component Design
5. Reference Solution (collapsible)
6. How It Works
7. Edge Cases
8. Accessibility
9. Performance Considerations
10. Follow-up Questions
11. Common Mistakes

## 📚 Questions

Questions are ordered from easiest to hardest (difficulty first, estimated time as the tie-break).

### Easy (1–6)

| No. | Question | Difficulty | Link |
| --- | --- | --- | --- |
| 01 | Accordion | Easy | [Open](./01-accordion.md) |
| 02 | Todo List | Easy | [Open](./02-todo-list.md) |
| 03 | Star Rating | Easy–Medium | [Open](./03-star-rating.md) |
| 04 | Counter, Stopwatch and Timer | Easy–Medium | [Open](./04-counter-stopwatch-timer.md) |
| 05 | Progress Bars | Easy–Medium | [Open](./05-progress-bars.md) |
| 06 | Tabs | Easy–Medium | [Open](./06-tabs.md) |

### Medium (7–17)

| No. | Question | Difficulty | Link |
| --- | --- | --- | --- |
| 07 | Like Button (Optimistic Update) | Medium | [Open](./07-like-button-optimistic-update.md) |
| 08 | OTP Input | Medium | [Open](./08-otp-input.md) |
| 09 | Modal / Dialog | Medium | [Open](./09-modal-dialog.md) |
| 10 | Infinite Scroll | Medium | [Open](./10-infinite-scroll.md) |
| 11 | Pagination | Medium | [Open](./11-pagination.md) |
| 12 | Tic-Tac-Toe and Connect-4 | Medium | [Open](./12-tic-tac-toe-connect-4.md) |
| 13 | Image Carousel | Medium | [Open](./13-image-carousel.md) |
| 14 | Toast / Notification System | Medium | [Open](./14-toast-notification.md) |
| 15 | Stepper / Multi-Step Form | Medium | [Open](./15-stepper-multi-step-form.md) |
| 16 | Form with Validation | Medium | [Open](./16-form-validation.md) |
| 17 | Shopping Cart | Medium | [Open](./17-shopping-cart.md) |

### Medium–Hard and Hard (18–24)

| No. | Question | Difficulty | Link |
| --- | --- | --- | --- |
| 18 | Autocomplete / Typeahead | Medium–Hard | [Open](./18-autocomplete-typeahead.md) |
| 19 | Search with Filters | Medium–Hard | [Open](./19-search-with-filters.md) |
| 20 | Nested Comments | Hard | [Open](./20-nested-comments.md) |
| 21 | File Explorer / Folder Tree | Hard | [Open](./21-file-explorer.md) |
| 22 | Multi-Select Dropdown | Hard | [Open](./22-multi-select-dropdown.md) |
| 23 | Kanban Board | Hard | [Open](./23-kanban-board.md) |
| 24 | Data Table | Hard | [Open](./24-data-table.md) |

> Interview frequency does not follow difficulty. Revise 02, 09, 10, 11, 14, 16, 18, 20, 21 and 24 first.

### Extra practice: gaps worth covering (25–32)

| No. | Question | Difficulty | Link |
| --- | --- | --- | --- |
| 25 | Utility Hooks (`useDebounce`, `useFetch`, `useOnClickOutside`, ...) | Easy–Medium | [Open](./25-utility-hooks.md) |
| 26 | JS Polyfills and Utilities (`debounce`, `Promise.all`, `bind`, ...) | Medium | [Open](./26-js-polyfills-utilities.md) |
| 27 | Nested Checkboxes | Medium | [Open](./27-nested-checkboxes.md) |
| 28 | Tooltip / Popover | Medium–Hard | [Open](./28-tooltip-popover.md) |
| 29 | Undo / Redo (`useHistory`) | Medium–Hard | [Open](./29-undo-redo.md) |
| 30 | Drag-and-Drop Sortable List | Hard | [Open](./30-drag-drop-sortable-list.md) |
| 31 | Chat UI | Hard | [Open](./31-chat-ui.md) |
| 32 | Virtualized List | Hard | [Open](./32-virtualized-list.md) |

## 🗓️ Suggested 4-Week Plan

| Week | Focus |
| --- | --- |
| 1 | 01–06 |
| 2 | 07–13 |
| 3 | 14–19 |
| 4 | 20–24, then redo the weakest ones as timed mock interviews |
| 5 (optional) | 25–32: utilities and senior-level topics |

## 🧠 How To Use This Repository

1. Read the problem statement and requirements only. Do not open the solution.
2. Set a timer (45–60 minutes) and build it from scratch in a Vite + React sandbox.
3. Compare with the reference solution, then read edge cases, accessibility and follow-ups.
4. Tick the practice checklist. Redo the question after a week without looking.
5. Practice saying the clarifying questions and trade-offs out loud.

## 🔎 What Interviewers Evaluate

| Area | Examples |
| --- | --- |
| Component design | Reusability, props API, composition |
| State | Lifting up, `useReducer`, derived state |
| Edge cases | Empty, loading and error states, double clicks |
| Performance | `memo`, `useCallback`, debounce, observers |
| Accessibility | ARIA roles, keyboard support, focus management |
| Code quality | Folder structure, naming, small functions |
| Communication | Clarify first, plan, explain trade-offs |

## 🛠️ Quick Setup For Practice

```bash
npm create vite@latest practice -- --template react
cd practice
npm install
npm run dev
```

## 🔥 Most Frequently Asked (Revise First)

Todo list, autocomplete, modal, infinite scroll, nested comments, file explorer, data table, toast system, star rating, and form validation.

---

**Last Updated**: 2026 | **Status**: Questions 01–24 ordered Easy to Hard, plus 8 extra topics (25–32).
