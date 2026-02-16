---
id: 10eade59-e536-48e7-8364-59ccdda48f06
created: '2026-02-11T16:40:10.047Z'
modified: '2026-02-16T12:08:16.275Z'
---
# Spec: Git-Powered Features

**Version History, Diffs, and Branches as Drafts 22222**

---

## Overview

Three features that lean into the "Git-powered notes" identity. All three only apply when `storageType === 'github'` — they're hidden/disabled for filesystem storage.

---

## Feature 1: Version History

### What it does

<u>Browse the commit history of any note and restore previous versions.</u>

### UI

**Trigger:** Clock icon button in `EditorToolbar.tsx`, right side, next to the export dropdown. Only visible when `storageType === 'github'`.

**Panel:** Right-side slide-over panel (`~340px` wide), same pattern as `NoteListPanel`:

-   Desktop: Collapses inline, pushes editor narrower
    
-   Mobile: Fixed overlay with backdrop
    

**Panel layout:**

```
┌─────────────────────────────┐
│ ← Version History      ✕    │  h-12 header
├─────────────────────────────┤
│ ● Current version           │
│   2 minutes ago             │
│                             │
│ ○ Update via MagicMarkdown  │
│   Yesterday at 3:42 PM      │
│                             │
│ ○ Update via MagicMarkdown  │
│   Feb 8 at 11:15 AM        │
│                             │
│ ○ Created via MagicMarkdown │
│   Feb 7 at 9:00 AM         │
│                             │
│         Load more           │
└─────────────────────────────┘
```

**Clicking a version:**

-   The version entry expands inline to show:
    
    -   "Restore" button (primary)
        
    -   "Compare" button (secondary) — opens diff view (Feature 2)
        
-   The editor switches to a **read-only preview** of that version's content
    
-   A banner appears above the editor: `"Viewing version from Feb 8 at 11:15 AM"` with a "Back to current" link
    

**Restoring a version:**

-   Confirmation modal: "Restore this version? Your current content will be replaced."
    
-   On confirm: calls `updateNote()` with the old content → triggers normal save → new commit
    
-   Toast: "Version restored"
    

### API Calls (via Octokit, in `GitHubFsAdapter`)

```typescript
// List commits for a note file
octokit.repos.listCommits({
  owner, repo,
  path: 'notes/my-note.md',  // note's file path in the repo
  sha: branch,                // current branch
  per_page: 20,
  page: 1,
})

// Get file content at a specific commit
octokit.repos.getContent({
  owner, repo,
  path: 'notes/my-note.md',
  ref: commitSha,             // the commit SHA to read from
})
```

### New Methods on GitHubFsAdapter

```typescript
// Add to GitHubFsAdapter class
async getFileHistory(path: string, page?: number): Promise<FileCommit[]>
async getFileAtCommit(path: string, commitSha: string): Promise<string>
```

```typescript
// New types
interface FileCommit {
  sha: string           // commit SHA
  message: string       // commit message
  date: string          // ISO timestamp
  author: string        // committer name/login
}
```

### Exposing Git Methods to the App

The current `StorageProvider` and `FileSystemAdapter` interfaces don't have Git-specific methods. Rather than polluting those generic interfaces, we expose Git functionality through a separate accessor:

```typescript
// lib/storage/index.ts — new export
export function getGitHubAdapter(): GitHubFsAdapter | null
```

This returns the underlying `GitHubFsAdapter` when GitHub storage is active, or `null` for filesystem storage. Components check for `null` and hide Git features accordingly.

### New Files

| File | Purpose |
| --- | --- |
| `stores/history-store.ts` | Version history state (commits list, selected version, loading) |
| `components/editor/VersionHistoryPanel.tsx` | The slide-over panel UI |
| `components/editor/VersionBanner.tsx` | "Viewing version from..." banner above editor |

### State (history-store.ts)

```typescript
interface HistoryState {
  isOpen: boolean
  commits: FileCommit[]
  selectedCommit: FileCommit | null
  previewContent: string | null  // HTML content of selected version
  isLoading: boolean
  hasMore: boolean
  page: number

  open: () => void
  close: () => void
  loadHistory: (notePath: string) => Promise<void>
  loadMore: (notePath: string) => Promise<void>
  selectVersion: (commit: FileCommit, notePath: string) => Promise<void>
  clearSelection: () => void
}
```

Not persisted — ephemeral UI state.

---

## Feature 2: Diff View

### What it does

Show a side-by-side or inline diff comparing two versions of a note, with additions highlighted green and deletions highlighted red.

### UI

**Entry points:**

1.  "Compare" button on a version in the history panel (compares that version vs. current)
    
2.  Future: compare any two versions (dropdown selectors) — skip for v1
    

**Display:** Replaces the editor area with a diff view (same space, not a new panel):

```
┌──────────────────────────────────────────────┐
│ Comparing: Feb 8 at 11:15 AM → Current   ✕  │
├──────────────────────────────────────────────┤
│                                              │
│  ## Introduction                             │
│                                              │
│ - This was the old intro paragraph that      │  red bg
│ - explained the concept briefly.             │  red bg
│ + This is the new intro paragraph with       │  green bg
│ + more detail about the concept.             │  green bg
│                                              │
│  The rest of the content stays the same...   │
│                                              │
└──────────────────────────────────────────────┘
```

**Diff format:** Unified inline diff (not side-by-side). Simpler, works better on narrow screens, familiar to developers.

### Implementation

**Diff library:** `diff` (npm package, ~7KB gzipped). Lightweight, well-maintained, works client-side.

```bash
npm install diff
npm install -D @types/diff
```

**Diffing strategy:**

-   Fetch both versions as **markdown** (not HTML) — diffs of markdown are human-readable; diffs of HTML are not
    
-   Use `Diff.diffLines()` for line-by-line comparison
    
-   Render result with colored backgrounds using Tailwind classes
    

The `getFileAtCommit()` method already returns raw markdown from GitHub. For the current version, convert current HTML content back to markdown using the existing `htmlToMarkdown()` utility.

### New Files

| File | Purpose |
| --- | --- |
| `components/editor/DiffView.tsx` | The diff display component |

### State

Add to `history-store.ts`:

```typescript
// Additional state for diff
diffMode: boolean
diffContent: DiffLine[] | null

showDiff: (oldContent: string, newContent: string) => void
closeDiff: () => void
```

```typescript
interface DiffLine {
  type: 'added' | 'removed' | 'unchanged'
  content: string
}
```

---

## Feature 3: Branches as Drafts

### What it does

Create named drafts that live on Git branches. Write freely on a draft without affecting your main notes. Merge back when ready, or discard.

### Concept Mapping

| Git concept | User-facing term |
| --- | --- |
| Branch | Draft |
| `main` branch | Published / Main |
| Create branch | Start a draft |
| Switch branch | Switch to draft |
| Merge to main | Publish draft |
| Delete branch | Discard draft |

Users never see the word "branch" — it's always "draft."

### UI

**Draft indicator in toolbar:** When on a draft, a pill badge appears in the toolbar next to the version history button:

```
[📝 Draft: blog-rewrite ▾]
```

Clicking it opens a dropdown:

-   List of drafts (branches other than main, prefixed with `draft/`)
    
-   "Main" entry at top (always present)
    
-   "New draft..." option at bottom
    
-   "Publish draft" option (only when on a draft, merges to main)
    
-   "Discard draft" option (only when on a draft, deletes branch)
    

**New draft flow:**

1.  User clicks "New draft..."
    
2.  Small modal: text input for draft name + "Create" button
    
3.  Creates branch `draft/{name}` from current `main` HEAD
    
4.  Switches storage to that branch (reinitializes `GitHubFsAdapter`)
    
5.  Toast: "Draft started"
    
6.  All edits now commit to the draft branch
    

**Publish flow:**

1.  User clicks "Publish draft"
    
2.  Confirmation modal: "Publish draft '{name}'? This will merge your changes into main."
    
3.  Merges `draft/{name}` → `main` via GitHub API
    
4.  Switches back to `main` branch
    
5.  Deletes the draft branch
    
6.  Toast: "Draft published"
    
7.  Reloads notes from main
    

**Discard flow:**

1.  User clicks "Discard draft"
    
2.  Confirmation modal: "Discard draft '{name}'? All changes on this draft will be lost."
    
3.  Switches back to `main` branch
    
4.  Deletes the draft branch
    
5.  Toast: "Draft discarded"
    

### Branch Naming Convention

All draft branches use the prefix `draft/`:

-   `draft/blog-rewrite`
    
-   `draft/meeting-notes-cleanup`
    
-   `draft/new-section`
    

This keeps them separate from any branches the user might have for other purposes. When listing drafts, we filter branches by this prefix.

### API Calls (via Octokit, in `GitHubFsAdapter`)

```typescript
// List branches (filter for draft/* prefix)
octokit.repos.listBranches({ owner, repo, per_page: 100 })

// Create branch
octokit.git.createRef({
  owner, repo,
  ref: 'refs/heads/draft/my-draft',
  sha: mainHeadSha,
})

// Merge branch to main
octokit.repos.merge({
  owner, repo,
  base: 'main',
  head: 'draft/my-draft',
  commit_message: 'Publish draft: my-draft',
})

// Delete branch
octokit.git.deleteRef({
  owner, repo,
  ref: 'heads/draft/my-draft',
})
```

### New Methods on GitHubFsAdapter

```typescript
async listDraftBranches(): Promise<DraftBranch[]>
async createDraftBranch(name: string): Promise<DraftBranch>
async mergeDraftToMain(branchName: string): Promise<void>
async deleteBranch(branchName: string): Promise<void>
async getMainHeadSha(): Promise<string>
```

```typescript
interface DraftBranch {
  name: string          // e.g. "draft/blog-rewrite"
  displayName: string   // e.g. "blog-rewrite"
  sha: string           // branch HEAD commit SHA
}
```

### Switching Branches

When the user switches drafts, we:

1.  Flush any pending changes on the current branch
    
2.  Call `resetStorage()` to tear down the current provider
    
3.  Update `github-store.selectedBranch` to the new branch
    
4.  Call `initializeGitHubStorage()` with the new branch
    
5.  Reload notes and folders from the new branch
    

This uses the existing initialization flow — no new storage plumbing needed.

### New Files

| File | Purpose |
| --- | --- |
| `stores/drafts-store.ts` | Draft branches state and actions |
| `components/editor/DraftIndicator.tsx` | Toolbar pill + dropdown |
| `components/editor/NewDraftModal.tsx` | Name input modal for creating a draft |

### State (drafts-store.ts)

```typescript
interface DraftsState {
  drafts: DraftBranch[]
  activeDraft: DraftBranch | null  // null = on main
  isLoading: boolean
  isSwitching: boolean

  loadDrafts: () => Promise<void>
  createDraft: (name: string) => Promise<void>
  switchToDraft: (branchName: string) => Promise<void>
  switchToMain: () => Promise<void>
  publishDraft: (branchName: string) => Promise<void>
  discardDraft: (branchName: string) => Promise<void>
}
```

Not persisted — the `github-store.selectedBranch` already persists the active branch.

### Merge Conflicts

If a merge fails (409 Conflict from GitHub API), we:

1.  Show an error toast: "Merge conflict — your draft has changes that conflict with main."
    
2.  Don't delete the branch
    
3.  For v1, the user resolves conflicts manually in GitHub's UI
    
4.  Future: show conflicting files and let user choose "keep draft" vs "keep main" per file
    

---

## Implementation Order

### Phase 1: Version History (foundation)

1.  Add `getFileHistory()` and `getFileAtCommit()` to `GitHubFsAdapter`
    
2.  Add `getGitHubAdapter()` export to `lib/storage/index.ts`
    
3.  Create `history-store.ts`
    
4.  Build `VersionHistoryPanel.tsx` and `VersionBanner.tsx`
    
5.  Add clock icon button to `EditorToolbar.tsx`
    
6.  Add `versionHistoryOpen` state to `ui-store.ts`
    
7.  Wire panel into `AppShell.tsx` layout
    

### Phase 2: Diff View (builds on Phase 1)

1.  Install `diff` package
    
2.  Build `DiffView.tsx` component
    
3.  Add diff state to `history-store.ts`
    
4.  Wire "Compare" button in version history panel
    
5.  Add markdown conversion for current content in diff flow
    

### Phase 3: Branches as Drafts (independent of Phase 1-2)

1.  Add branch management methods to `GitHubFsAdapter`
    
2.  Create `drafts-store.ts`
    
3.  Build `DraftIndicator.tsx` dropdown
    
4.  Build `NewDraftModal.tsx`
    
5.  Implement branch switching (flush → reset → reinitialize)
    
6.  Wire into `EditorToolbar.tsx`
    

---

## Files Changed (Existing)

| File | Changes |
| --- | --- |
| `lib/storage/github-fs-adapter.ts` | Add history + branch methods |
| `lib/storage/index.ts` | Add `getGitHubAdapter()` export |
| `stores/ui-store.ts` | Add `versionHistoryOpen` state |
| `components/editor/EditorToolbar.tsx` | Add history button + draft indicator |
| `components/layout/AppShell.tsx` | Add `VersionHistoryPanel` to layout |

## Files Created (New)

| File | Purpose |
| --- | --- |
| `stores/history-store.ts` | Version history + diff state |
| `stores/drafts-store.ts` | Draft branch management state |
| `components/editor/VersionHistoryPanel.tsx` | History slide-over panel |
| `components/editor/VersionBanner.tsx` | "Viewing old version" banner |
| `components/editor/DiffView.tsx` | Inline diff display |
| `components/editor/DraftIndicator.tsx` | Draft pill + dropdown |
| `components/editor/NewDraftModal.tsx` | Create draft modal |

---

## Design Decisions

1.  **Git methods on adapter, not on StorageProvider** — keeps the generic interface clean; Git features are GitHub-only
    
2.  **Separate accessor (**`getGitHubAdapter()`**)** — components check for `null` to know if Git features are available
    
3.  **Diff on markdown, not HTML** — human-readable diffs, avoids noisy tag changes
    
4.  **Unified diff, not side-by-side** — simpler, works on mobile, familiar format
    
5.  **Branch switching via full reinitialization** — reuses existing boot flow, no new storage plumbing
    
6.  `draft/` **prefix convention** — isolates MagicMarkdown branches from user's other branches
    
7.  **User-facing term is "draft", not "branch"** — accessible to non-developers
    
8.  **No version history for filesystem storage** — no Git data available; feature is hidden entirely
