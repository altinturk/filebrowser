# Changes

| ID | Description | Timestamp | Status | Action taken |
|---|---|---|---|---|
| BUG1 | top bar hamburger menu cannot be minimized, it's always on | 2026-10-21 | Fixed | The hamburger now hides the docked sidebar and shows it again on the next click. |
| BUG2 | on settings page when you extend the user management. there's a circular shape cutting corners not only of "user management" button but all the sub buttons as well - one big circle applied to all 4 items. it should have been one by one. | 2026-10-21 | Fixed | The rounded shape was moved from the group container to each row individually. |
| BUG3 | when i change level on tree - the left hand hamburger menu wiggles. therefore, tree as well. | 2026-10-21 | Fixed | The sidebar and content card keep their layout during the brief loading state between folders. |
| BUG4 | left hand menu active "disk" or "Share" icon is excessively big and round and not aligned | 2026-10-21 | Fixed | Only the name row is highlighted now, with the usage bar below it and the icons aligned. |
| BUG5 | New button is equally big -- lots of padding. | 2026-10-21 | Fixed | The New button was reduced from 56px to 44px with less padding. |
| FEATURE1 | add under system-admin section, an option to change the main title and icon: now it's showing FileBrowser Quantum with fb default icon. (if the setting and logo change depend on backend, add the options anyway but make them inactive/grayed out. | 2026-10-21 | Done (read-only) | A greyed-out Branding block shows the current name and icon, because they can only be changed in config.yaml. |
| FEATURE2 | the left hand menu i like is this: (mockup image) | 2026-10-21 | Done | The sidebar now shows the folder tree on top with a source root row and the links pinned at the bottom. |
| BUG6 | the storage bar and percentage text is still overlapping | 2026-10-21 | Fixed | The percentage text is now placed under the storage bar instead of centred on it. |
