# User Profile

*Adapted from OpenViking's `bot/workspace/USER.md` (vikingbot template) — nearly unchanged, it was already a good fit. Fill in the blanks; delete the checkbox options you don't want.*

Information about the user to help personalize interactions. This file lives in the `_global` memory scope once `ai-memory` is running (§12 Phase J2 in `FUTURE_PLANS.md`) — it should follow you into every project, not just one repo.

## Basic Information

- **Name**: (your name)
- **Timezone**: (your timezone)
- **Language**: (preferred language)

## Preferences

### Communication Style

- [ ] Casual
- [ ] Professional
- [ ] Technical

### Response Length

- [ ] Brief and concise
- [ ] Detailed explanations
- [ ] Adaptive based on question

### Technical Level

- [ ] Beginner
- [ ] Intermediate
- [ ] Expert

## Work Context

- **Primary Role**: (developer / whatever fits)
- **Main Projects**: (the repos Kip will actually work across — list them; this is what makes cross-repo memory in Phase J3 meaningful)
- **Tools You Use**: (languages, frameworks, editors, the homelab stack)
- **Homelab**: (what's actually running — services, hosts, anything Kip will be asked to manage per Phase J6)

## Coding Preferences

*(New section vs. the vikingbot original — worth having since Kip writes code, which vikingbot doesn't do.)*

- **Style conventions**: (tabs/spaces, naming, testing habits — whatever you actually care about)
- **What to always ask before doing**: (e.g. force-pushes, deleting branches, editing CI config, homelab restarts during work hours)
- **What never needs asking**: (e.g. running the test suite, reading files, routine git status checks)

## Topics of Interest

-
-
-

## Special Instructions

(Any specific instructions for how Kip should behave that don't fit above)

---

*Edit this file to customize Kip's behavior. Once `ai-memory` is running, this becomes a real memory page — edits should go through `memory_write_page` (scope: "global") rather than hand-editing the file, so they're versioned instead of silently overwritten by the self-improvement loop.*
