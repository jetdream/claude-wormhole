# File Preview — Design Spec

## Summary

Add a file browser panel (right side) and preview modal to the wormhole web UI. Users can browse the project filesystem tree and preview files (markdown, images, HTML, code) without leaving the terminal.

## Layout

```
+---------+----------------------+----------+
| Sidebar |    Terminal (xterm)  |   File   |
|  (w-56) |                     | Browser  |
|         |                     |  (w-64)  |
|         |                     |          |
|         |    [click file]     |  > src/  |
|         |         |           |  > docs/ |
|         |   +----------+     |    pkg.. |
|         |   | Preview  |     |          |
|         |   |  Modal   |     |          |
|         |   +----------+     |          |
+---------+----------------------+----------+
```

### Desktop
- File browser as right panel (`w-64`), collapsible via toggle button.
- Terminal shrinks to accommodate when panel is open.

### Mobile
- File browser as slide-out drawer from right edge (mirrors sidebar drawer from left).
- Or bottom sheet — whichever fits better during implementation.

## File Browser (Right Panel)

### Tree View
- Rooted at session's `workingDir` (resolved from live tmux session).
- Expand/collapse folders on click.
- Lazy-load children on folder expand (one level per API call, not full tree upfront).
- File icons by extension (simple, CSS-only or inline SVG).
- Sorted: directories first, then files, alphabetical within each group.

### Excluded Paths
`.git`, `node_modules`, `.next`, `dist`, `__pycache__`, `.worktrees`

### Quick Filter
- Text input at top of panel to narrow visible files by name.
- Client-side filter on already-loaded tree nodes (no API call on each keystroke).

## Preview Modal

### Trigger
Click any file in the tree browser.

### Modal Layout
- Reuse existing modal pattern: `fixed inset-0 bg-black/60 z-50`.
- File name + relative path in header.
- Close: X button, Escape key, backdrop click.
- Content area scrollable, `max-h-[80vh]`.
- Copy button in header for text/code files.

### Rendering by File Type

| Type | Detection | Rendering |
|---|---|---|
| Markdown | `.md`, `.mdx` | Parsed to HTML via `react-markdown`, styled with prose classes. Raw/rendered toggle. |
| Images | `.png`, `.jpg`, `.jpeg`, `.gif`, `.svg`, `.webp` | `<img>` tag, fit-to-modal sizing, centered. |
| HTML | `.html`, `.htm` | Sandboxed iframe (`sandbox="allow-same-origin"`), no script execution. |
| Code/text | Everything else (`.ts`, `.js`, `.json`, `.py`, `.sh`, `.css`, `.yaml`, etc.) | Syntax-highlighted with line numbers via `shiki`. |

### Size Cap
- Files over 500KB show an error message instead of content: "File too large to preview (X MB)".
- Images exempt from this cap (loaded as data URLs, browser handles sizing).

## API

### `GET /api/files?session=<name>&dir=<relative>`

List directory contents (one level).

**Query params:**
- `session` (required) — tmux session name
- `dir` (optional, default `.`) — relative path within workingDir

**Response:**
```json
{
  "root": "/Users/shyam/www/claude-wormhole",
  "dir": "src/components",
  "files": [
    { "name": "Sidebar.tsx", "type": "file", "size": 8234, "ext": ".tsx" },
    { "name": "lib", "type": "dir" }
  ]
}
```

**Behavior:**
- Sorted: dirs first, then files, alphabetical.
- Excludes configured hidden dirs.
- Validates `dir` stays within workingDir (rejects `..` traversal).

### `GET /api/files/read?session=<name>&path=<relative>`

Read file content for preview.

**Query params:**
- `session` (required) — tmux session name
- `path` (required) — relative file path within workingDir

**Response (text/code):**
```json
{
  "name": "Sidebar.tsx",
  "content": "...",
  "mimeType": "text/typescript",
  "size": 8234
}
```

**Response (images):**
```json
{
  "name": "logo.png",
  "content": "data:image/png;base64,...",
  "mimeType": "image/png",
  "size": 45678
}
```

**Response (too large):**
```json
{
  "error": "too_large",
  "name": "bundle.js",
  "size": 2345678
}
```

**Security:**
- Path validation: resolve to absolute, confirm it starts with workingDir. Reject `..` segments.
- No symlink following outside workingDir.
- HTML rendered in sandboxed iframe (no script execution).

## Dependencies

| Package | Purpose | Notes |
|---|---|---|
| `react-markdown` | Markdown rendering | Lightweight, standard in Next.js ecosystem |
| `shiki` | Syntax highlighting | Same engine as VS Code, theme-aware, all languages |

No PDF viewer. No heavy rich-text editors.

## Resolving workingDir

Both API endpoints resolve the session's working directory by calling:
```
tmux display-message -p -t <session> '#{pane_current_path}'
```
Same approach used by `addSavedSession` in `src/lib/sessions.ts`. Falls back to the `workingDir` stored in `sessions.json` if tmux command fails.

## Components

| Component | Location | Purpose |
|---|---|---|
| `FileBrowser` | `src/components/FileBrowser.tsx` | Right panel tree view + filter input |
| `FilePreviewModal` | `src/components/FilePreviewModal.tsx` | Modal with type-aware rendering |

State for panel open/closed and preview file path lifted to `page.tsx`, passed as props.

## Non-Goals

- File editing (read-only preview only)
- Terminal path click-to-preview (future enhancement)
- PDF rendering
- File upload/download
- Git diff view
