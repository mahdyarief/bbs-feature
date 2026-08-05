---
status: draft
owner: product
reviewed-by: engineering, security
date: 2026-08-05
---

# Edge Cases & Decisions: Pickup Person Display in Gate Check-Out Flow

> This document captures known edge cases, their impact, and the agreed-upon decision. All decisions are binding for implementation.

| # | Edge Case | Impact if Unhandled | Decision | Owner / Notes |
|---|-----------|----------------------|----------|---------------|
| EC-1 | `GET /studentPickUpPersons?studentId=:id` returns HTTP 500 or timeout | Modal shows blank/empty section → gate staff loses trust in feature | ✅ Show static error message: "Unable to load authorized persons. Please try again." + retry button. Do NOT crash modal or block submit. | Frontend — use `useFromApi` error state + button triggers refetch |
| EC-2 | Student has > 10 pickup persons | Long list overflows modal → unreadable, hard to scan | ✅ Render max 5 items by default + "Show all (X)" link. Clicking expands full list in scrollable container (max-height: 200px). | Frontend — use `useState` + `slice()` + toggle |
| EC-3 | `relationship` field is empty/null in DB (e.g., `"relationship": null`) | Display shows `• Budi Santoso ()` — unprofessional, confusing | ✅ Fallback to "Other" if `relationship` is empty/null/whitespace. Render as `• Budi Santoso (Other)`. | Backend — no change; Frontend handles in mapper/render |
| EC-4 | `full_name` exceeds 100 chars (e.g., long formal name) | Text overflow breaks layout or causes horizontal scroll | ✅ Truncate to 30 chars + ellipsis: `• Muhammad Rizky Alamsyah Putra… (Father)`. Tooltip on hover shows full name. | Frontend — use `title` attr + CSS `text-overflow` |
| EC-5 | Modal opens with `studentId = null` or falsy (e.g., during initial load) | API called with invalid ID → 400 error spam in console | ✅ Skip fetch if `studentId` is falsy (`!studentId`). Show "Loading…" only after valid ID arrives. | Frontend — guard `useEffect` with `if (!studentId) return;` |
| EC-6 | User switches student quickly (e.g., rapid click between students) | Multiple concurrent requests → race condition → stale data shown | ✅ Cancel previous request using `AbortController` (already supported by `fromApi`/`useFromApi`). | Frontend — `useFromApi` supports abort — enable it |
| EC-7 | `studentPickUpPersons` array is empty (`[]`) | No visual feedback — appears as missing section | ✅ Always render section header "Authorized Pickup Persons" + message "No authorized pickup persons registered…" — never hide section. | Frontend — conditional render, not data-driven show/hide |
| EC-8 | Network is offline | Fetch fails silently → no feedback | ✅ Show offline-aware message: "Offline. Authorized persons unavailable." — no retry button. | Frontend — detect `navigator.onLine` + useEffect |
| EC-9 | `full_name` contains special characters (emoji, RTL, quotes) | Rendering broken or XSS risk | ✅ Render as plain text (no `dangerouslySetInnerHTML`). React escapes by default — no sanitization needed. | Frontend — safe; Backend stores as-is |
| EC-10 | Modal stays open while student’s pickup persons change (e.g., parent updates via portal) | List becomes stale — gate staff sees outdated info | ✅ No auto-refresh. List is snapshot-on-open. To see update, user must close & reopen modal. | Frontend — no polling; explicit refresh only via retry button (EC-1) |

## Open Questions (To Be Resolved Before Implementation)

| # | Question | Owner | Status |
|---|----------|-------|--------|
| OQ-1 | Should we log display events (e.g., "PickUpModal opened for student X with Y pickup persons") for analytics? | Product | ⏳ Pending confirmation |
| OQ-2 | Is there a business requirement to highlight `is_primary: true` persons (e.g., bold, icon)? | Product | ⏳ Pending confirmation |

> 🔔 **Note**: All decisions above are final unless updated via formal change request (`notes.md`). Engineering must implement *exactly as decided* — no reinterpretation.
