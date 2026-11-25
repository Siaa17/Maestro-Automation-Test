# Maestro test - TC-16: Close (X) returns without applying changes

Automated scenario from Part 1: **TC-16 - Verify that the Close (X) icon returns to the Search screen without applying changes.**

Test file: `tc16_close_without_applying.yaml`. **It fails on purpose because of BUG-05.**

## How to run

Requirements:
- Java 17 or newer, with `JAVA_HOME` set (Android Studio includes one in `jbr`)
- Android SDK with `adb` in `PATH`
- An Android emulator or a real device with USB debugging
- Maestro CLI - https://maestro.mobile.dev

Steps:
1. Start the emulator (I used a Pixel 9 Pro AVD, Android 17).
2. Install the Jetsnack app (`com.example.jetsnack`). From the Jetsnack project folder:
   ```
   ./gradlew installDebug
   ```
3. From this folder, run:
   ```
   maestro test tc16_close_without_applying.yaml
   ```

Maestro prints each step as COMPLETED or FAILED. Screenshots, logs and, on failure, the UI tree are saved in `~/.maestro/tests/<date>/`.

### Differences from the written test case

The app does not match the requirements, so the test is adapted:
- The Filters modal is opened from the **Home** screen, because the Search screen has no Filters icon (BUG-01).
- The "5 items" check on the Search screen is left out, because filters cannot be opened from Search.
- Everything else follows TC-16: select "Organic", `$$` and "Alphabetical", tap Close (X), open Filters again, and check that nothing is selected and the default sort has the check mark.

### Expected result of the run

All steps before the final check pass. The sort check also **passes**: "Android's favorite (default)" has the check mark again, and "Alphabetical" does not.

The test **fails** at the check that `$$` is not selected:

```
Assert that Contains child: "Android's favorite \(default\)" is visible... COMPLETED
Assert that Contains child: "Alphabetical" is not visible... COMPLETED
Assert that Contains child: "\$\$" is not visible... FAILED
```

`$$` is still selected after the modal is reopened. This is **BUG-05** (Close (X) and tap outside keep the unapplied filter selections). Maestro stops at the first failed step, so the "Organic" check after it does not run. Screenshot `tc16_2_reopened` in `~/.maestro/tests/<date>/` shows the reopened modal with `$$` still highlighted.

The test is meant to stay red until BUG-05 is fixed. Then it becomes a regression test.

## Why I chose this scenario

- **High priority.** Requirement AC7 says the user must be able to leave the modal without applying changes. If this does not work, the user cannot cancel what they selected.
- **It finds a real bug.** The test fails because of BUG-05. That is the main purpose of automation: catching regressions. Once BUG-05 is fixed, the test will protect the fix.
- **It can run in the current app.** Most other High priority scenarios (TC-06 - TC-15, TC-19) need the Apply button, which does not exist (BUG-03). TC-16 needs only the Close (X) icon.
- **It covers more than one part of the modal.** It uses the Sort, Price and LifeStyle sections, scrolls down and back up, closes and reopens the modal. This gave me more to learn than a single tap test.

## What I learned

- **Maestro basics.** A test is a list of YAML steps such as `tapOn`, `assertVisible` and `scrollUntilVisible`. No code or compilation is needed. Maestro waits for the screen by itself, so no `sleep` steps are needed.
- **How elements are found.** Maestro finds elements by visible text or by the accessibility label (content description). The Filters icon has no text, but its label is "Filters", so `tapOn: "Filters"` works.
- **The same text can appear twice.** "Organic" is on the Home screen and in the modal. The selector `below: "LifeStyle"` makes sure the test taps the chip in the modal.
- **Check the state, not the text.** Maestro saves a screenshot and the UI tree when a step fails. From the UI tree I saw that the selected (`checked`) state is on the chip container, not on its text. My first check looked at the text, so it always passed, which was wrong. Now the test uses `containsChild: "Organic"` with `checked: true`.
- **The sort options work the same way.** I first checked the sort row with `selected: true`, which failed. The UI tree showed that Android reports the sort row's state as `checked`, so the test uses `checked: true` there as well.
- **Text in selectors is a regular expression.** `$$` and "Android's favorite (default)" contain special characters, so they are written as `"\\$\\$"` and `"Android's favorite \\(default\\)"`.
- **A tool limitation.** In Maestro 2.12 `checked: false` did not find the chips, while `checked: true` worked. I check "not selected" as "a selected chip is not visible" (`assertNotVisible` + `checked: true`). I proved with a separate small test that this really tells selected and unselected chips apart.
- **Scrolling inside the modal.** The LifeStyle section is below the visible part, so the test scrolls down first. After that, the Close (X) icon is scrolled out of view, so the test must scroll back up before tapping it.
- **Windows.** The test `name` must not contain `:`, because Maestro uses it as a folder name.
- **A failing test can be a good result.** The test fails because the app is wrong, not because the test is wrong. Writing down which bug causes the failure makes the result clear.

## What I would automate next

1. **TC-18 - tap outside the modal.** Same check as TC-16 with a different way of closing. It also fails because of BUG-05.
2. **TC-17 - system Back.** Maestro has a `back` step. Today it would fail because of BUG-04 (Back closes the whole app).
3. **TC-02 - default state of the modal.** All sections are visible, nothing is selected, and the default sort is checked. It is a fast smoke test.
4. **TC-03 / TC-04 / TC-05 - selecting lifestyle, price and category filters.** These use the same selectors as TC-16 (`containsChild` + `checked: true`).
5. **TC-08 - TC-15 - filtered results after Apply.** These can be automated once BUG-02 and BUG-03 are fixed. The test would check the "N items" header and the product names.
6. **CI.** Run all tests on every pull request, for example with GitHub Actions and an Android emulator, and save the JUnit report (`maestro test --format junit`).
