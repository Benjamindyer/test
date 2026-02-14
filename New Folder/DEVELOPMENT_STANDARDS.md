---
id: a8dd0d38-fa0d-4814-8a41-6219442fb533
created: '2026-02-14T20:47:21.349Z'
modified: '2026-02-14T20:47:45.950Z'
---
**MagicMarkdownLast Updated:** February 2026

Coding standards, development practices, and quality guidelines for all contributors (human and AI).

---

## Table of Contents

1.  [Code Style](#code-style)
    
2.  [TypeScript Guidelines](#typescript-guidelines)
    
3.  [React & Next.js Patterns](#react--nextjs-patterns)
    
4.  [State Management (Zustand)](#state-management-zustand)
    
5.  [Storage & Data Layer](#storage--data-layer)
    
6.  [Editor (Tiptap v3)](#editor-tiptap-v3)
    
7.  [Styling (Tailwind CSS 4)](#styling-tailwind-css-4)
    
8.  [Performance](#performance)
    
9.  [Security](#security)
    
10.  [Git Workflow](#git-workflow)
     
11.  [AI Agent Guidelines](#ai-agent-guidelines)
     

---

## Code Style

### Key Rules

| Rule | Setting | Reason |
| --- | --- | --- |
| Quotes | Single (`'`) | Consistency |
| JSX Quotes | Single (`'`) | Consistency |
| Indentation | 2 spaces | React community standard |
| Semicolons | None | Next.js convention |
| Trailing Commas | ES5 | Cleaner diffs |
| Max Line Length | 120 chars | Readable on modern screens |

### Import Order

```typescript
// 1. External packages
import { useState } from 'react'
import { create } from 'zustand'

// 2. Internal aliases (@/)
import { useNotesStore } from '@/stores/notes-store'
import { cn } from '@/lib/utils/cn'

// 3. Relative imports
import { NoteListItem } from './NoteListItem'

// 4. Type imports (last)
import type { Note, Folder } from '@/types/index'
```

### Naming Conventions

| Type | Convention | Example |
| --- | --- | --- |
| Components | PascalCase | `NoteListItem.tsx` |
| Hooks | camelCase, `use` prefix | `useAutoSave.ts` |
| Stores | kebab-case, `-store` suffix | `notes-store.ts` |
| Utilities | camelCase | `formatDate.ts` |
| Constants | SCREAMING\_SNAKE | `MAX_FILE_SIZE` |
| Types/Interfaces | PascalCase | `interface Note {}` |
| CSS custom properties | kebab-case, `--color-` prefix | `--color-text-primary` |

### File Organization

```typescript
'use client'                                    // 1. Directive (if needed)

import { useState } from 'react'               // 2. External imports
import { useNotesStore } from '@/stores'        // 3. Internal imports
import { SomeChild } from './SomeChild'         // 4. Relative imports
import type { Note } from '@/types/index'       // 5. Type imports

interface Props {                               // 6. Types/interfaces
  note: Note
}

const MAX_TITLE_LENGTH = 100                    // 7. Constants

function formatPreview(text: string) { ... }    // 8. Helpers

export function MyComponent({ note }: Props) {  // 9. Component
  // Hooks first
  const [state, setState] = useState()

  // Derived values
  const preview = formatPreview(note.content)

  // Event handlers
  const handleClick = () => { ... }

  // Effects last (before return)
  useEffect(() => { ... }, [])

  // Render
  return ( ... )
}
```

---

## TypeScript Guidelines

### Strict Mode

TypeScript is configured with strict mode. All code must pass `npx tsc --noEmit`.

-   **Target:** `ES2022` (required for regex dotAll flag and modern features)
    
-   Next.js auto-manages `jsx`, `module`, and path aliases in `tsconfig.json`
    

### Type Definitions

**Prefer interfaces for objects:**

```typescript
// Good
interface Note {
  id: string
  title: string
  folderId: string | null
}

// Use `type` for unions and utilities
type EditorMode = 'wysiwyg' | 'markdown'
type NoteWithFolder = Note & { folder: Folder }
```

### Avoid `any`

When `any` is unavoidable (third-party library gaps), add a comment:

```typescript
// eslint-disable-next-line @typescript-eslint/no-explicit-any
const data = response as any // TODO: Define proper type
```

### Core Types

All shared types live in `/types/`:

| File | Contents |
| --- | --- |
| `index.ts` | `Note`, `Folder`, `Workspace`, `UserSettings` |
| `storage.ts` | `StorageProvider` interface |
| `ai.ts` | `AIProvider`, `AIAction`, message types |

---

## React & Next.js Patterns

### Next.js 16 Conventions

-   **App Router** with route groups: `(auth)` for login/signup, `(app)` for settings
    
-   Route groups don't add path segments: `(app)/settings/page.tsx` maps to `/settings`
    
-   Use `'use client'` directive for all components that use hooks, state, or browser APIs
    
-   `middleware.ts` is deprecated in Next.js 16 — use proxy pattern instead
    

### Component Types

| Type | Location | Purpose |
| --- | --- | --- |
| Pages | `app/` | Route-level, data fetching |
| Features | `components/editor/`, `components/sidebar/` | Business logic, stateful |
| UI primitives | `components/ui/` | Presentational, reusable, props-only |

### Hooks

Custom hooks live in `/hooks/`. Keep hooks focused and under 200 LOC.

```typescript
// Good: Single responsibility
export function useAutoSave(saveFn: () => Promise<void>, deps: unknown[]) {
  // Debounced auto-save logic
}

// Good: Compose multiple store reads
export function useEditor() {
  const editor = useEditorInstance()
  const fontSize = useSettingsStore((s) => s.fontSize)
  return { editor, fontSize }
}
```

### Error Handling

```typescript
// Good: Type-safe error handling
try {
  await saveNote(data)
  addToast('Note saved', 'success')
} catch (error: unknown) {
  const message = error instanceof Error ? error.message : 'An error occurred'
  addToast(message, 'error')
}

// Bad: Never use `any` for errors
catch (error: any) { ... }
```

---

## State Management (Zustand)

### Store Structure

All stores live in `/stores/` and follow this pattern:

```typescript
import { create } from 'zustand'
import { persist } from 'zustand/middleware' // only if state should persist

interface MyState {
  // State
  items: Item[]
  isLoading: boolean

  // Actions
  loadItems: () => Promise<void>
  addItem: (item: Item) => Promise<void>
}

export const useMyStore = create<MyState>((set, get) => ({
  items: [],
  isLoading: false,

  loadItems: async () => {
    set({ isLoading: true })
    const items = await storage.getItems()
    set({ items, isLoading: false })
  },

  addItem: async (item) => {
    await storage.createItem(item)
    set((state) => ({ items: [...state.items, item] }))
  },
}))
```

### Store Conventions

-   **One store per domain:** `notes-store`, `workspace-store`, `ui-store`, `ai-store`, `settings-store`
    
-   **Use selectors** to avoid unnecessary re-renders:
    
    ```typescript
    // Good: Only re-renders when activeNoteId changes
    const activeNoteId = useNotesStore((s) => s.activeNoteId)
    
    // Bad: Re-renders on any store change
    const store = useNotesStore()
    ```
    
-   **Use** `persist` **middleware** only for user preferences (settings, AI config) — not for content data (that goes through the storage layer)
    
-   **Keep actions in the store**, not in components. Components call store actions; stores call storage providers.
    

### Current Stores

| Store | Persisted | Purpose |
| --- | --- | --- |
| `notes-store` | No  | Notes CRUD, active note |
| `workspace-store` | No  | Workspaces and folders |
| `ui-store` | No  | Sidebar state, editor mode |
| `settings-store` | Yes | Theme, font size |
| `ai-store` | Yes (partial) | Provider, API key, model |
| `auth-store` | No  | User session |
| `toast-store` | No  | Toast notifications |

---

## Storage & Data Layer

### Storage Abstraction

All data access goes through the `StorageProvider` interface defined in `/types/storage.ts`. This allows swapping between IndexedDB (default) and Google Drive.

```typescript
interface StorageProvider {
  // Notes
  getNotes(workspaceId: string): Promise<Note[]>
  createNote(note: Partial<Note>): Promise<Note>
  updateNote(id: string, updates: Partial<Note>): Promise<Note>
  deleteNote(id: string): Promise<void>

  // Folders, Workspaces, etc.
  // ...
}
```

**Rules:**

-   Components never call IndexedDB or Google Drive APIs directly
    
-   Components call store actions → stores call `getStorageProvider()` → provider handles persistence
    
-   The active provider is determined at runtime via `getStorageProvider()` in `/lib/storage/`
    

### IndexedDB (Default Provider)

-   Uses the `idb` library for promise-based IndexedDB access
    
-   Database name and version managed in `/lib/storage/indexeddb.ts`
    
-   All operations are async — always `await` storage calls
    

### Supabase (Sync Layer)

Supabase is used for authentication and optional cloud sync, not as the primary data store.

-   Migrations live in `/supabase/migrations/`
    
-   Migrations must be idempotent (`CREATE TABLE IF NOT EXISTS`, `DROP POLICY IF EXISTS`)
    
-   Client setup in `/lib/supabase/`
    

---

## Editor (Tiptap v3)

### Key Differences from Tiptap v2

These are hard-won lessons — do not repeat these mistakes:

| Issue | Detail |
| --- | --- |
| Table extensions | Use **named exports**: `{ Table, TableRow, TableCell, TableHeader }` — NOT default exports |
| BubbleMenu | Not exported from `@tiptap/react` in v3 — we implement our own in `components/editor/BubbleMenu.tsx` |
| CodeBlock lowlight | CJS compatibility issues with Node 25 — use StarterKit's built-in `codeBlock` instead |

### Editor Architecture

```
components/editor/
├── Editor.tsx              # Main editor component, manages content + title
├── EditorToolbar.tsx       # Top toolbar (headings, formatting, lists, etc.)
├── BubbleMenu.tsx          # Floating menu on text selection (formatting + AI)
├── EditorStyles.css        # All editor content styles (headings, lists, code, etc.)
├── extensions/
│   └── index.ts            # Tiptap extension configuration
└── LinkModal.tsx           # Modal for inserting links
```

### Editor Styles

All editor content styling is in `EditorStyles.css` using the `.editor-content` class. Use `em` units for sizes so they scale with the user's font size setting (applied via `--editor-font-size` CSS variable).

**Important:** Tailwind's preflight resets list styles, heading margins, etc. The editor CSS must explicitly restore these for content inside `.editor-content`.

---

## Styling (Tailwind CSS 4)

### Key Differences from Tailwind v3

| Feature | Tailwind 4 | Tailwind 3 |
| --- | --- | --- |
| Config | `@theme` block in CSS | `tailwind.config.ts` |
| Import | `@import "tailwindcss"` | `@tailwind base/components/utilities` |
| PostCSS | `@tailwindcss/postcss` plugin | `tailwindcss` plugin |

### Design Tokens

All colors, fonts, and radii are defined as CSS custom properties in the `@theme` block in `globals.css`. Reference them via Tailwind classes:

```tsx
// Good: Use semantic token classes
<div className="bg-bg-primary text-text-secondary border-border-default" />

// Bad: Hardcoded colors
<div className="bg-gray-100 text-gray-600 border-gray-200" />
```

### Utility Patterns

-   Use `cn()` helper (from `/lib/utils/cn.ts`) for conditional classes
    
-   Prefer Tailwind utilities over custom CSS — except for editor content styles
    
-   Use `group` and `group-hover:` for parent-child hover interactions
    

```typescript
import { cn } from '@/lib/utils/cn'

<div className={cn(
  'px-4 py-2 rounded-lg transition-colors',
  isActive && 'bg-bg-active',
  isDisabled && 'opacity-50 pointer-events-none',
)} />
```

---

## Performance

### Must-Have Optimizations

1.  **Debounce auto-save** — don't save on every keystroke (use `useAutoSave` hook)
    
2.  **Zustand selectors** — subscribe to specific fields, not entire stores
    
3.  **Lazy loading** — use `React.lazy` for route-level components
    
4.  **Truncate in sidebar** — use CSS `truncate`, never render full note content in lists
    
5.  **Batch operations** — use `Promise.all` for independent async operations
    

### Avoid

```typescript
// Bad: N+1 pattern — sequential updates
for (const note of notes) {
  await storage.updateNote(note.id, { folderId })
}

// Good: Parallel updates
await Promise.all(notes.map(n => storage.updateNote(n.id, { folderId })))
```

-   Don't load all note content when only titles are needed
    
-   Don't create inline functions in render without `useCallback` when passed as props
    
-   Don't subscribe to full Zustand store objects — use selectors
    

---

## Security

### Input Validation

-   Validate file uploads (type, size) before processing
    
-   Sanitize HTML content if rendering user-provided markup outside the editor
    
-   Validate AI provider API keys format before storing
    

### API Keys

-   AI API keys are stored in the browser via Zustand persist (localStorage)
    
-   Keys are sent directly from the browser to the AI provider — they never touch our servers
    
-   Never log or expose API keys in error messages or toasts
    

### AI Prompt Safety

```typescript
// Good: System prompt with guardrails, user data separate
const messages = [
  { role: 'system', content: getSystemPrompt() },
  { role: 'user', content: `${getActionPrompt(action)}\n\n${selectedText}` },
]

// Bad: Concatenating user input directly into system prompts
const prompt = `You are an editor. Fix this: ${userInput}`
```

---

## Git Workflow

### Commit Messages

Format: concise description of what changed and why.

```bash
# Development Standards
git commit -m "Wire up AI bubble menu to editor text selection"
git commit -m "Fix task list checkbox/text alignment in editor CSS"
git commit -m "Add move-to-folder submenu in note context menu"

# Bad
git commit -m "fix stuff"
git commit -m "updates"
git commit -m "wip"
```

### Before Pushing

1.  Run `npx tsc --noEmit` — must pass with zero errors
    
2.  Test the feature manually in the browser
    
3.  Check browser console for errors
    
4.  Verify on a narrow viewport (sidebar + editor should not break)
    

---

## AI Agent Guidelines

### Before Starting Work

1.  Read `MEMORY.md` for project context and known gotchas
    
2.  Read the files you plan to modify before editing
    
3.  Understand existing patterns before introducing new ones
    

### During Development

1.  Follow all standards in this document
    
2.  Run `npx tsc --noEmit` after changes
    
3.  Fix errors you introduce — don't leave broken types
    
4.  Keep changes focused on the task — don't refactor surrounding code
    

### Code Quality Checklist

-   Error handling uses `catch (error: unknown)` with `instanceof` checks
    
-   Zustand stores use selectors (not full store subscriptions)
    
-   Hooks are under 200 LOC
    
-   Batch operations use `Promise.all` (no sequential loops)
    
-   AI prompts separate system instructions from user data
    
-   New CSS in editor uses `.editor-content` scope with `em` units
    
-   Tailwind classes use semantic design tokens, not hardcoded colors
    
-   Components use `'use client'` directive when needed
    

### Communication

-   Be direct and concise
    
-   Explain what you're doing briefly
    
-   Don't over-engineer — minimum complexity for the current task
    
-   When unsure, ask rather than guess
    

---

## Quick Reference

### Commands

```bash
npx next dev --turbopack   # Start dev server
npx next build             # Production build
npx tsc --noEmit            # TypeScript check
```

### File Locations

| What | Where |
| --- | --- |
| Core types | `/types/index.ts`, `/types/storage.ts`, `/types/ai.ts` |
| Stores | `/stores/*-store.ts` |
| Hooks | `/hooks/use*.ts` |
| Editor | `/components/editor/` |
| Editor styles | `/components/editor/EditorStyles.css` |
| AI components | `/components/ai/` |
| UI primitives | `/components/ui/` |
| Storage providers | `/lib/storage/` |
| Design tokens | `/app/globals.css` (`@theme` block) |
| Supabase migrations | `/supabase/migrations/` |

### Architecture Diagram

```
Browser
├── Next.js App Router
│   ├── (auth) routes → Login / Signup
│   ├── (app) routes → Settings
│   └── Root route → AppShell
│       ├── Sidebar (folders, notes, workspace selector)
│       └── Editor (Tiptap v3 + toolbar + bubble menu)
├── Zustand Stores (client state)
│   └── StorageProvider interface
│       ├── IndexedDB (default, offline-first)
│       └── Google Drive (optional sync)
└── AI Providers (direct browser → API)
    ├── OpenAI / Anthropic / Google / etc.
    └── Configured via Settings → AI
```
