# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Context

A personal To-Do List web app for organizing tasks across custom sections. Designed to be lightweight with real-time sync across devices via Firebase Firestore.

**Non-obvious design decisions:**
- A special "Completed" section (`id: '__completed__'`) is always last, always collapsed by default, and has no rename/delete buttons. Clicking a task moves it here; clicking it again restores it to its original section via `task.originalSectionId`.
- New tasks are inserted at the top of their section, not the bottom.
- Bulk import always targets the General section regardless of the dropdown selection.
- `isSaving` flag prevents `onSnapshot` from triggering a re-render immediately after a local write (race condition mitigation).

## Tech Stack & Third-Party Services

**No build step.** Single HTML file served directly.

**Firebase (project: `todo-list-17c1d`)**
- **Firestore** — entire app state stored as one document at `todo/state`
- SDK loaded via CDN: `https://www.gstatic.com/firebasejs/10.12.0/`
- Config is embedded in `index.html` (intentionally public; security handled by Firestore rules)
- **Free tier limits:** 20,000 writes/month — every user action triggers a full document write, so heavy usage can exhaust quota quickly
- **Current Firestore rules:** test mode (allows unauthenticated read/write — expires after 30 days unless updated in Firebase Console)

**GitHub Pages** — hosts the app at `https://rosedliu.github.io/todo-list/`

## Architecture

### Data model
```js
state = {
  sections: [{ id, name, collapsed }],
  tasks: [{ id, sectionId, text, done, originalSectionId? }]
}
```
Entire state is one Firestore document. Every mutation rewrites the whole document via `setDoc`.

### Data flow
1. `load()` → `onSnapshot(stateRef)` listens for Firestore changes
2. Any change → handler mutates `state` directly → `save()` → `setDoc` to Firestore
3. Firestore update triggers `onSnapshot` → `render()` rebuilds entire DOM

### Rendering
`render()` clears and rebuilds the full DOM on every change — no diffing. Fine for small lists; will lag with 100+ tasks.

Event listeners are attached per-element on each render (no event delegation).

## Common Development Tasks

### Deploy
```bash
cd ~/Test
git add .
git commit -m "message"
git push origin main  # GitHub Pages auto-deploys
```

### Debug sync issues
```js
// In browser console:
console.log(JSON.stringify(state, null, 2))
```
Check the `todo/state` document directly in the Firebase Console to confirm writes are reaching Firestore.

### Firestore rules expiry
Test mode rules expire after 30 days. If the app stops syncing, go to Firebase Console → Firestore → Rules and extend or update them.

## Trust & Safety

- **No authentication.** All visitors share the same `todo/state` document — anyone with the URL can read and overwrite all data.
- **No rate limiting.** Rapid actions could exhaust Firestore free tier quota with no user feedback.
- **No error handling.** Failed Firestore writes are silent — users won't know if a save failed.
- Input is set via `.textContent` (safe from XSS), but no length validation exists.

## Observability

No logging or error tracking is implemented.

**When things break, check:**
1. Firebase Console → Firestore → `todo/state` document (is data there and correct?)
2. Firebase Console → Usage (quota exhausted?)
3. Firebase Console → Firestore → Rules (rules expired?)
4. Browser DevTools → Console (Firebase SDK errors)

## Known Issues

- **Single document bottleneck:** Full state rewrite on every action; will hit 1 MB Firestore document limit with large lists
- **No undo/recovery:** Deleting tasks or sections is permanent
- **Full re-render:** Entire DOM rebuilt on every change — slow with large task lists
- **Collapse state syncs across tabs:** Can feel jarring in multi-tab use
- **No offline support:** Every change requires a live Firestore connection
