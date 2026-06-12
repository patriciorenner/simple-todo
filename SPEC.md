# Kanban — Specification

> A lightweight, Mac-native kanban board for personal short-term planning.
> Single user. Local-only. Not intended for distribution.

Status: **Draft v1** · Last updated: 2026-06-12

---

## 1. Purpose & Scope

A personal kanban app to manage "what I'm currently working on" — short-term
planning and organization. One user, one machine, no collaboration, no sync,
no distribution. Optimized for the author's specific hardware.

**Out of scope (v1):** team/collaboration features, accounts, sharing,
iCloud/CloudKit sync, JSON export/import, multi-device, web/iOS targets,
priority field, WIP limits, due-soon (non-overdue) cues, arrow-key card
navigation.

---

## 2. Target Environment & Constraints

| Aspect | Target |
|---|---|
| Machine | MacBook Pro 14" (2021), Apple M1 Pro, 16 GB |
| OS | macOS Tahoe 26.5.1 (deployment target: macOS 26) |
| Display | Built-in Liquid Retina XDR, native ~3024×1964, default scaled **1512×982 pt**, P3 wide gamut, ProMotion 120 Hz |
| Architecture | Apple Silicon (arm64) only — no Intel/universal build needed |
| Distribution | None. Local debug/run build only; no notarization, no signing for release. |

**Hardware-driven requirements:**
- **HW-1** Board must display **3 columns side-by-side at half-screen width** (~756 pt) without horizontal scrolling. Target column width ≈ 230 pt.
- **HW-2** Animations (drag, reorder, sheet transitions) must target **120 fps** (ProMotion); no dropped-frame jank during card drag.
- **HW-3** Card and tag colors authored in **Display P3** to use the wide gamut.

---

## 3. Technology

- **UI:** SwiftUI (macOS 26 APIs), `NavigationSplitView` for the sidebar layout.
- **Persistence:** SwiftData (SQLite-backed), local app container only.
- **Undo:** native `UndoManager` integration (SwiftData supports undo/redo).
- **No third-party dependencies.**

---

## 4. Data Model

### 4.1 Entities

**Board**
- `id`
- `name`
- `sortOrder` (position in sidebar)
- `columns` → ordered list of Column (cascade delete)
- `tags` are app-global, not owned by a board (see Tag)
- timestamps: `createdAt`, `modifiedAt`

**Column**
- `id`
- `name`
- `sortOrder` (position within board)
- `board` (parent)
- `cards` → ordered list of Card. **Delete rule: nullify, not cascade** — deleting a column must not destroy cards that reference it (archived cards survive; see Card).
- No WIP limit.

**Card**
- `id`
- `title` (required, always shown)
- `notes` / `description` (optional, long text)
- `dueDate` (optional)
- `color` (Post-it color; default **canary yellow**)
- `tags` → many-to-many with Tag (global library)
- `checklistItems` → ordered list of ChecklistItem (cascade delete)
- `isArchived` (Bool) + `archivedAt` (optional)
- `board` (owning board — direct reference so per-board archive and restore survive column deletion; cascade delete from Board)
- `originColumn` (the column this card lives in / was archived from; **nullable** — set to nil if that column is deleted, which makes restore fall back to the board's first column)
- `sortOrder` (position within its column)
- timestamps: `createdAt`, `modifiedAt`

**ChecklistItem**
- `id`
- `text`
- `isDone` (Bool)
- `sortOrder`
- `card` (parent)

**Tag** (global, app-wide library)
- `id`
- `name` (unique, used as pill label)
- `color` (from the tag palette)
- relationship: many Cards across any board

### 4.2 Color palettes

- **Post-it palette (card background):** canary yellow (default), pink, orange, green, blue, purple. Authored in Display P3, with dark-mode-aware variants for legible text contrast.
- **Tag palette:** a defined set of distinct colors chosen when a tag is created. Independent of the Post-it set.

---

## 5. Features

### 5.1 Boards
- Multiple boards, each with its own columns and cards.
- Managed from a **collapsible left sidebar** (`NavigationSplitView`): list of boards; create, rename, delete, reorder (drag).
- Switch boards by selecting in the sidebar.
- **State restoration:** on launch, reopen the last-selected board, window size, and position.
- **First launch:** auto-create a starter board named "My Board" with default columns **To Do · Doing · Done**.

### 5.2 Columns
- Customizable: add, rename, reorder (drag), delete.
- **Deletion is blocked only by active (non-archived) cards.** A column with no active cards may be deleted — including a column whose cards have all been **archived**. To delete a column with active cards, first move or archive them.
- No WIP limits.
- Columns scroll horizontally only when they exceed the available width; 3 fit at half-screen (HW-1).

### 5.3 Cards

**Collapsed (on-board) card shows:**
- **Title** (always), on the Post-it color background.
  - Title rendered **red when the card is overdue** (dueDate in the past). This is the only overdue cue; no due-soon state.
- **Tags** as **colored pills with text**, truncating/wrapping to a single row.
- Nothing else (no due-date badge, no checklist count, no description preview).

**Detail (modal sheet) shows/edits:**
- Title
- Description (long text)
- Due date (optional; set/clear)
- Tags (add/remove from global library; create new tag inline)
- Checklist: add / check-off / reorder / delete items; progress (e.g. "2/5") shown here
- Color picker (Post-it palette)
- Created / modified timestamps (read-only)
- "Move to board" action (see 5.6)
- Archive / Delete actions

**Creation:**
- **Quick-add** field at the bottom of each column: type title, Enter to commit.
- New cards default to **canary yellow**; recolor later in the detail sheet.

### 5.4 Interactions
- **Drag & drop:** move cards between columns and reorder within a column (120 fps target, HW-2).
- **Inline title edit** on the collapsed card.
- **Detail editing** via modal sheet (slides over the window; does not consume board width).
- Open detail with **Enter** (selected card) / click; close with **Esc**.

### 5.5 Search & Filter
- **Search:** field filters visible cards by title/description text.
- **Tag filter:** narrow the board to selected tag(s) from the global library (checklist-style picker).
- Search and tag filter apply to the currently selected board's active cards.

### 5.6 Card lifecycle
- **Archive (primary):** move a card out of the active board into the **per-board archive** (searchable, restorable). Archived cards retain their origin column for restore.
- **Delete (true removal):** permanent removal, with **undo (⌘Z)**.
- **Restore** from the per-board archive returns the card to its origin column. If that column has since been deleted, the card restores to the board's **first column** instead.
- **Move to board:** card context menu / detail action sends the card to another board, landing in that board's first column.

### 5.7 Appearance
- **Middle-ground visual style:** native macOS layout and typography (SF), with the **full Post-it color as the card background** (flat, no skeuomorphic paper texture).
- **Follows system light/dark mode**, with dark-mode-aware color variants ensuring readable text contrast on every Post-it color.
- Clean, uncluttered main board.

### 5.8 Bulk column operations
- A column exposes a **bulk-actions menu** (column header overflow `⋯`) that operates on **all active cards in that column** at once:
  - **Archive all** — move every card in the column to the per-board archive.
  - **Delete all** — permanently remove every card in the column (single **undo (⌘Z)** restores the whole batch).
  - **Move all** — send every card to a chosen target: another column on the current board, or another board (lands in that board's first column).
  - **Tag all** — apply (or remove) a selected tag to every card in the column.
- Destructive bulk actions (delete all, archive all) require a **confirmation** stating the affected count.
- Bulk operations act on the column's **active** cards only (already-archived cards are unaffected) and respect the **current search/tag filter** — they target exactly the cards currently visible in the column. The confirmation count reflects the filtered subset, and when a filter is active the bulk menu labels this (e.g. "Archive 3 filtered cards") to make the scope explicit.
- Bulk archive / delete / move are also the intended way to **empty a column before deleting it** (column deletion remains blocked while non-empty — see 5.2).

### 5.9 Window & app behavior
- Regular windowed app (Dock + Spotlight launch).
- State restoration (last board, size, position).
- **Minimum window size** chosen so 3 columns render at ~230 pt each within ~756 pt (half-screen), plus sidebar/chrome. (Sidebar collapsible to reclaim width.)
- No menu-bar mode. No launch-at-login by default (user can enable via System Settings).

### 5.10 Storage & backup
- SwiftData, **local-only**, fully offline.
- **No JSON export/import.** Backup relies on Time Machine / the local store.

---

## 6. Keyboard Shortcuts (v1)

| Action | Shortcut |
|---|---|
| New card (focus quick-add in selected/first column) | ⌘N |
| New board | ⌘⇧N |
| Search (focus search field) | ⌘F |
| Delete card | ⌘⌫ |
| Undo (incl. delete) | ⌘Z |
| Open detail sheet (selected card) | Enter |
| Close detail sheet | Esc |

Deferred: arrow-key navigation between cards/columns; new-column shortcut.

---

## 7. Acceptance Criteria

**Boards**
- AC-B1 First launch with empty store auto-creates "My Board" with columns To Do · Doing · Done.
- AC-B2 User can create, rename, delete, and reorder boards from the sidebar.
- AC-B3 Sidebar is collapsible; collapsing widens the board area.
- AC-B4 On relaunch, the last-selected board, window size, and position are restored.

**Columns**
- AC-C1 User can add, rename, reorder (drag), and delete columns.
- AC-C2 Deleting a column with ≥1 **active** card is blocked; a column with no active cards (including one whose cards are all archived) can be deleted. Deleting a column does not destroy its archived cards.
- AC-C2a An archived card whose origin column was deleted restores to the board's first column.
- AC-C3 Three columns render fully side-by-side at half-screen width (~756 pt) with no horizontal scroll.

**Cards — board face**
- AC-K1 Collapsed card shows title + tag pills only; no other metadata.
- AC-K2 Title renders red when dueDate is in the past; normal otherwise.
- AC-K3 Tags render as colored pills with text, clipped to one row.
- AC-K4 New card via quick-add (type + Enter) is created canary yellow in the target column.

**Cards — detail**
- AC-D1 Detail opens as a modal sheet (Enter/click), closes with Esc.
- AC-D2 Sheet edits: title, description, due date (set/clear), tags, checklist, color.
- AC-D3 Checklist supports add/check/reorder/delete; progress "n/m" shown in the sheet.
- AC-D4 Created/modified timestamps are visible and read-only.

**Interactions**
- AC-I1 Cards drag between columns and reorder within a column; order persists.
- AC-I2 Drag animation sustains ProMotion frame rates without visible jank.
- AC-I3 Title is editable inline on the collapsed card.

**Search & filter**
- AC-S1 Search filters cards by title/description substring on the active board.
- AC-S2 Tag filter narrows the active board to selected tag(s).

**Lifecycle**
- AC-L1 Archiving moves a card to the per-board archive; it disappears from the board.
- AC-L2 Archived cards are searchable and restorable to their origin column.
- AC-L3 Delete removes a card permanently; ⌘Z restores it.
- AC-L4 "Move to board" sends a card to another board's first column.

**Bulk column operations**
- AC-X1 A column's bulk menu offers archive-all, delete-all, move-all, tag-all.
- AC-X2 Bulk archive/delete require confirmation showing the affected card count.
- AC-X3 Delete-all is reversed as a single batch by ⌘Z.
- AC-X4 Move-all sends every active card to the chosen target column/board.
- AC-X5 Tag-all adds (or removes) the selected tag on every active card in the column.
- AC-X6 Bulk operations act only on the cards currently visible in the column (active cards matching any active search/tag filter); the confirmation count and menu label reflect this filtered scope.

**Tags**
- AC-T1 Tags are a global, app-wide library (name + color), reusable across all boards.
- AC-T2 Creating/renaming/recoloring a tag updates it everywhere it's used.

**Appearance & platform**
- AC-A1 Card background is the full Post-it color; layout/typography are native.
- AC-A2 App follows system light/dark; text remains legible on every Post-it color in both modes.
- AC-A3 Data persists across launches via SwiftData, fully offline.

---

## 8. Definition of Done

A feature is **done only when its acceptance criteria are validated against a
live, running build of the app** — not against unit logic, mocks, or a passing
compile alone.

**Done requires all of the following:**

1. **Live validation of every AC.** Each acceptance criterion in §7 is exercised
   on an actual running build (`.app` launched on macOS 26) and observed to hold.
   No AC is considered met by reasoning or by lower-level tests alone.
2. **Automated UI validation.** Behavioral ACs are covered by **XCUITest** cases
   that launch the real app and assert on real UI, run headless via
   `xcodebuild test`. Logic-level ACs (data model, bulk ops, archive/restore,
   overdue, delete-block rules) are covered by **XCTest**. Every AC maps to at
   least one automated test.
3. **Visual validation.** Appearance ACs (AC-A1, AC-A2, AC-C3, AC-K2, AC-K3) are
   confirmed by screenshot of the running app — including light **and** dark mode,
   and the 3-columns-at-half-screen layout — captured with realistic seeded data.
4. **Gates green.** Format, lint, build, and the full test suite pass on every
   change (automated gates always on).
5. **Accessibility identifiers present.** Views expose the identifiers their
   XCUITests rely on; these are added as features are built, not retrofitted.
6. **Traceability.** A checklist maps each AC → the test(s)/screenshot that
   validate it, so 100% AC coverage is demonstrable, not asserted.

Done = **100% of §7 acceptance criteria validated live**, with the evidence
(tests + screenshots) recorded.

---

## 9. Open Items / Assumptions to Confirm

- **App name** — working title "Kanban"; final name TBD.
- **Tag palette** — exact color set to be finalized during design.
- **Post-it dark-mode variants** — exact contrast-adjusted values TBD in design.
- **Selection model** — how a card becomes "selected" for Enter/⌘⌫ (click-to-select vs. focus ring) to be defined in UI design.
