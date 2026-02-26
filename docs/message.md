Hey! While building my app on Desktop, I ran into a few macOS-specific behaviors that don't quite follow platform conventions. I've patched them locally in my fork and wanted to check if you'd be interested in a PR, or if you'd prefer I just keep these in my app.

1. App Menu items (About / Preferences)

macOS expects menu items tagged with specific roles to auto-move into the application menu (the bold-titled menu left of "File"). Currently, Desktop strips everything except Quit from the Apple menu. I added a role attribute on menu items (e.g., role="about", role="preferences") that maps to the standard wxID_ABOUT / wxID_PREFERENCES IDs, so wxWidgets handles the placement automatically. Without this, clicking "About" or "Preferences" in the app menu does nothing since the Window process doesn't forward those events to the menu handler.

2. max_size window option

min_size already exists but there's no max_size equivalent. I needed fixed-size windows (e.g., an About dialog) and added max_size to mirror the existing min_size pattern.
Both are fairly small, self-contained changes. Happy to clean them up into a proper PR with tests if you think they'd be useful upstream. Otherwise I'll just keep them in my local fork - no worries either way.

