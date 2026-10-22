# Security review: hyprbars `set_title`

This review covers the fork **Libertas-le-Spes/hyprland-plugins** as it stood on **2026-10-22**. It looks at Kabelo's hyprbars change that lets a script put a title of its own choosing in a window's bar. The code was read at these commits:

| Branch | Commit | What it is |
|---|---|---|
| `hyprbars-set-title` | `061cdee` | The change as offered upstream. It is identical to upstream's merged commit `d2427d4` (hyprwm/hyprland-plugins#716, merged 2026-10-20). |
| `set-title` | `0df37b2` | The same change, built on an older upstream (`7644cec`) for Hyprland 0.56.2, the version on the desktop. |
| `main` | `428bfcd` | The fork's main. It does not contain the change yet. |

The proposals below are planned, not done. This document will change as they land. Whoever fixes a finding updates this document in the same pull request: mark the finding fixed, link the pull request, and correct anything that is no longer true.

## Verdict

The change is small and mostly sound. It holds windows through weak references. It looks bars up safely, draws titles as plain text rather than Pango markup, and clips them to the bar. Nothing in it lets a web page, a remote party or another user control or crash the compositor. Only processes running as Kabelo can call `set_title`: the Hyprland control socket lives in a directory only his user can open (`drwx------`, checked on the live machine).

Two things should still be fixed before more code is built on this, both in `setLuaTitle`:

1. **Nothing limits what the title can be.** A title of about a megabyte, or one with thousands of line breaks, freezes the whole desktop for seconds. It also leaves a texture of hundreds of megabytes pinned for as long as that title stands. Every other title source already has a limit; `set_title` is the one place without one.
2. **A title of the wrong type skips C++ cleanup.** `luaL_checkstring` leaves the function by `longjmp`, jumping over the destructor of the window pointer it holds. That is undefined behaviour in C++. In practice the closed window's object is never freed. A short program against the same Lua library shows it happening (F2).

Neither one is an attack from outside. Both are the kind of mistake a script makes by accident, and both land on the compositor that the whole desktop runs in.

## Findings

| ID | Severity | Title | Location |
|---|---|---|---|
| F1 | Medium | `set_title` accepts a title of any length or shape; a large one freezes the desktop and pins hundreds of MB | `hyprbars/main.cpp:152` (`set-title`: `:203`), drawn by `hyprbars/barDeco.cpp:338` |
| F2 | Medium | A title that is not a string leaves by `longjmp` past a live window pointer: undefined behaviour, and the window is never freed | `hyprbars/main.cpp:152` (`set-title`: `:203`) |
| F3 | Low | The README shows building Lua source code around the title, which invites scripts to splice untrusted text into code that can run commands | `hyprbars/README.md:93-97` |
| F4 | Low | The README's window selector, `address:`, can point at a different, newer window once the first one closes | `hyprbars/README.md:96`, used by the window-names tool |
| F5 | Info | Titles are shown exactly as given: direction-override characters, look-alike letters and invalid UTF-8 all reach the bar | `hyprbars/barDeco.cpp:572` |
| F6 | Info | The fork's CI runs a workflow from another organisation at a moving ref, with the fork's secrets | `.github/workflows/nix-build.yml` |

Line numbers are for `hyprbars-set-title` unless a `set-title` number is given.

---

### F1 — `set_title` accepts a title of any length or shape (Medium)

**What it is.** `set_title` is the Lua function a script calls, usually through `hyprctl eval`, to replace the text in one window's bar. The plugin stores whatever string it is given (`main.cpp:152` → `barDeco.cpp:678`). On the next frame it hands that string to Hyprland's `renderText`, which lays it out with Pango, draws it into a Cairo image and uploads it as a GPU texture (`barDeco.cpp:338`). All of this happens on the compositor's only thread. While it runs, nothing on screen moves and no input is handled. Hyprland's wiki notes the control socket is handled "completely synchronously" ([Hyprland IPC](https://wiki.hypr.land/IPC/)).

**Why it matters.** Pango shortens a long line with "…", but only per line, and only up to a size. Every line break starts a new line that gets its own row in the texture. I copied `renderText`'s Pango and Cairo steps into a standalone program (same Pango 1.58.2 and Cairo 1.18.6 as the desktop). I ran it with a 2000-pixel-wide bar; it never touched the running compositor. Results:

| Title given to the bar | Time on the compositor thread | Image Hyprland builds | Outcome |
|---|---|---|---|
| `Notes` | under 10 ms | 27 × 13 px | normal |
| 4 KB of text, about the most a Wayland app can set | 9 ms | 2000 × 143 px, about 1 MB | normal |
| one line, 300 KB | 37 ms | 600 × 13 px | Pango shortens it |
| one line, 500 KB | 56 ms | 2,097,150 × 13 px | too wide for Cairo; the title vanishes |
| 2,400 long lines, about 970 KB | **5.4 s** | 2000 × 31,213 px, **about 250 MB** in memory, then the same again on the GPU | desktop frozen 5 s; memory held until the title changes |
| 6,000 long lines, about 2.4 MB | **33 s** | too tall for Cairo | desktop frozen 33 s; title vanishes |

The 250 MB is held per bar: ten windows given such a title hold 2.5 GB. The bar clips what it draws (`barDeco.cpp:532`, `scissor(ctx, titleBarBox)`), so the text never spills over other windows. The cost is in building the image, not showing it.

**Why only `set_title` is exposed.** Hyprland already bounds every other title. Wayland messages cap an app's title at about 4 KB. Hyprland reads an X11 title with a 2048-word limit, at most 8 KB (`src/xwayland/XWM.cpp`, `xcb_get_property(..., 0, 2048)`). `set_title` has no such bound. The IPC request is capped at 1 MiB on Hyprland main, but that does not bound the title: Lua can build a long string from a short request.

**Who it lands on and when.** Kabelo, at his desk. A script he trusts passes a title it did not check, and the desktop locks up for seconds or minutes with no sign of why. One example is a tool that labels bars from file names, clipboard text or command output. Someone who already runs code as him gains nothing new: `hyprctl eval` has no time limit (no `lua_sethook` in Hyprland's Lua setup), so such code could freeze the desktop anyway. That is why this is Medium and not High.

**Fix options**, safest first:

| Option | What it does | What it fails to stop |
|---|---|---|
| A. A `BarTitle` type that can only be made by a checking function, used in `setTitle`'s signature | `set_title` refuses, with an error and no change to the bar, any title over 1024 bytes, any invalid UTF-8, and any control character (line breaks included). `setTitle` takes `std::optional<BarTitle>`, so code that skips the check does not compile. | Client titles, which never pass through `set_title`. They are already capped at 8 KB, but can still hold line breaks. |
| B. Make every title safe at the point of drawing | Before `renderText`, any title from any source is made one line (controls become spaces), cut to 1024 bytes on a character boundary, and given placeholders for invalid bytes (`g_utf8_make_valid`). | Nothing extra on its own, but it fails open for `set_title`: a bad title is shown cut down instead of refused, so the calling script never learns it sent something wrong. |
| C. Limit how often a title can change | Throttles calls. | Not needed: the texture is rebuilt at most once per frame per bar, and one large call is enough to freeze. Does not address size. |

**Recommendation: A and B together.** A fails closed where the new input enters: a bad title is refused and the caller gets an error. B is defence in depth for every title, including client titles and anything added later. Use one limit constant for both. 1024 bytes is far more than a bar can show and far below anything that costs time.

**How to seal it.**
- The type is the seal. `BarTitle` has a private constructor and a single `static std::expected<BarTitle, std::string> from(std::string_view)`. `CHyprBar::setTitle` accepts only `std::optional<BarTitle>`. It lives in its own file pair, `hyprbars/BarTitle.hpp/.cpp`, because the rules for what a title may contain change for different reasons than the bar's drawing.
- Unit tests for `BarTitle::from` and the draw-time cleaner. These are plain C++ with no Hyprland dependency, so they can run in CI. Use made-up inputs: `Notes`, `Reading list`, a 1025-byte string, `"line one\nline two"`, `"Not\xff"`, `"Notes ‮"`.
- A libFuzzer target on both functions, built with `-fsanitize=address,undefined`, run for a short time in CI.

---

### F2 — A wrong-type title skips C++ cleanup (Medium)

**What it is.** `setLuaTitle` first looks up the window, which gives it `WINDOW`, a shared pointer that keeps the window object alive while the function runs (`main.cpp:144`). Then it reads the title with `luaL_checkstring(L, 2)` (`main.cpp:152`). If the title is not a string or a number, for example a table or `true`, `luaL_checkstring` raises a Lua error. Hyprland's Lua is the system `liblua.so.5.5`, built as C. Lua's manual says that "internally, Lua uses the C `longjmp` facility to handle errors" ([Lua 5.4 manual §4.4](https://www.lua.org/manual/5.4/manual.html#4.4)). `ldd /usr/bin/Hyprland` confirms the library, and the library imports `__longjmp_chk`. The jump goes straight back to the `lua_pcall` inside Hyprland's eval, past `setLuaTitle`'s frame, so `WINDOW`'s destructor never runs. C++ calls this undefined behaviour: "if replacing `std::longjmp` with `throw` … would invoke a non-trivial destructor for any automatic object, the behavior … is undefined" ([cppreference: longjmp](https://en.cppreference.com/w/cpp/utility/program/longjmp)).

**Shown, not reasoned.** A short C++ program linked against the same Lua 5.5.1 library mirrors `setLuaTitle`. It holds a shared pointer, calls `luaL_checkstring` on argument 2 inside `lua_pcall`, then prints how many owners the window object has:

| Title passed | Call result | Owners of the window afterwards (should be 1) |
|---|---|---|
| `"Notes"` | ok | 1 |
| `42` | ok (numbers become strings) | 1 |
| `{}` | error | **2** |
| `true` | error | **3** |

When the program then drops its own pointer, "window object freed" never prints. The window outlives its owner for good.

**Who it lands on.** Kabelo. Each mistaken call, such as a script that passes a value that is not a string, leaks one window object for the rest of the session. That includes its decorations: the bar, and the bar's texture. Hyprland's weak references to that window also keep resolving, so code that asks "is this window still alive?" gets the wrong answer. I did not demonstrate any effect inside Hyprland beyond the leak, because loading test code into the live compositor was out of bounds. Because the behaviour is undefined, a different compiler or Lua build could do worse.

**Fix options**, safest first:

| Option | What it does | What it fails to stop |
|---|---|---|
| A. Read every Lua argument before making any C++ object, and never call a raising `luaL_check*` / `luaL_arg*` / `lua_error` in plugin code | Check arg 2 with `lua_type` (`LUA_TNONE`, `LUA_TNIL` or `LUA_TSTRING`). Read it with `lua_tolstring`, which also gives the length, so text after an embedded NUL byte is not silently dropped. Report problems with Hyprland's `configError`, which only logs and returns (`LuaBindingsInternal.cpp`, `configError` → `reportError`, no `lua_error`). Only then look up the window. | A later edit that brings back a raising call. |
| B. A CI check that fails the build if `luaL_check`, `luaL_opt`, `luaL_arg`, `luaL_error`, `lua_error` or `luaL_typeerror` appears under `hyprbars/` | Stops A from regressing. | Calls hidden inside Hyprland helpers. `windowFromLuaSelectorOrObject` and `configError` were read and do not raise. |
| C. Build Lua as C++ so errors become exceptions | Destructors would run. | Not the plugin's choice: Hyprland links the distribution's C build. |

**Recommendation: A, sealed by B.** A removes the cause. B is a one-line `grep` in CI that makes the wrong thing fail the build. `newLuaButton` already follows A's pattern (`lua_isstring` then `configError`). `set_title` should match it.

---

### F3 — The README teaches building Lua code around the title (Low)

**What it is.** The README tells script writers to call `set_title` through `hyprctl eval 'hl.plugin.hyprbars.set_title("address:…", "Notes")'` (`README.md:93-97`). `eval` runs its argument as Lua code inside the compositor, and that Lua can start programs (`hl.exec_cmd`, plus the full Lua standard library from `luaL_openlibs`). The natural next step for a reader is to put a variable where `"Notes"` is.

**What happens if left alone.** A script that splices a title it did not write into the code can be steered by whoever wrote that title. A browser tab is one such source: web pages choose their titles. Text that ends the string can append Lua of its own, and that Lua runs as Kabelo. This is the same class of bug as SQL injection ([CWE-94](https://cwe.mitre.org/data/definitions/94.html)). It is not a bug in the plugin today, and the README example only uses a constant. **The window-names tool, the one real caller, is safe.** It escapes `"`, `\` and every character below 32 as Lua `\ddd` codes, and it has a test for exactly this (`window_names/desktops/hyprland.py`, `test_backslashes_and_line_breaks_cannot_end_the_lua_string`). I checked that escaping against Lua's string rules and found no way out of the string.

**Fix options**, safest first:

| Option | What it does | What it fails to stop |
|---|---|---|
| A. README shows a safe way to pass text, with a warning | Show escaping every byte as a Lua `\ddd` code, the way window-names does. Say plainly never to splice raw text into `eval`. | Readers who skip the warning. |
| B. Add a way to pass the title as data rather than code, such as a plugin dispatcher | Nothing to escape. | Depends on Hyprland's Lua-era API still offering one. Needs research before it is promised. |

**Recommendation: A now. B only if Hyprland offers a supported data path.**

**How to seal it.** Keep the window-names test above. Any other script that calls `set_title` gets the same test: a title of `"` followed by text must arrive in the bar unchanged.

---

### F4 — `address:` can point at a different window later (Low)

**What it is.** A window selector is the string that tells `set_title` which window to change. The README uses `address:0x…`. Hyprland builds that from the window object's memory address (`ViewQuery.cpp`, `MODE_ADDRESS`: `std::format("0x{:x}", rc<uintptr_t>(w.get()))`). When a window closes, its memory is freed and can be reused, so the next window can get the same address.

**What happens if left alone.** A script reads the window list, the window closes, a new window opens at the same address, and the script's `set_title` lands on the new window. The new window's bar then shows the old label. A browser could show the label of a closed terminal, for example. It is a short race and it corrects itself on the next naming pass, but for that time the bar says something false about the window. Selectors `title:` and `class:` are worse for this use: any app can choose its own title, so it can make itself the match.

**Fix.** Hyprland also has `stableid:`, a counter that never repeats in a session (`WindowMetadata.cpp`: `m_stableID(windowIDCounter++)`, shown as `stableId` in `hyprctl clients -j`). Recommend `stableid:` in the README example, and switch window-names to it. Seal it with a window-names test that asserts the request starts with `stableid:`.

---

### F5 — Titles are shown exactly as given (Info)

Pango draws invalid UTF-8 with a placeholder glyph ([Pango docs](https://docs.gtk.org/Pango/method.Layout.set_text.html)) and logs a warning each time it lays the title out. Unicode direction controls such as U+202E make text read in a different order than it was written; this is the "Trojan Source" trick, [CVE-2021-42574](https://confluence.atlassian.com/security/multiple-products-security-advisory-unrendered-unicode-bidirectional-override-characters-cve-2021-42574-1086419475.html). Look-alike letters from other alphabets pass through too. This was true of hyprbars before this change, for every app's own title. `set_title` adds a second way in. F1's `BarTitle` rejects direction controls for `set_title`. The draw-time cleaner can drop them for every title. No more is needed.

### F6 — CI runs another organisation's workflow at a moving ref (Info)

`nix-build.yml` uses `hyprwm/actions/.github/workflows/nix.yml@main` with `secrets: inherit`. This comes from upstream. In the fork it means whatever hyprwm pushes to that branch runs with the fork's secrets. The fork has no secrets that matter today. If it ever gets any, pin the workflow to a commit SHA.

---

## What was checked and found sound

| Concern | What I found | Where |
|---|---|---|
| Dangling window pointer after close | The bar holds its window weakly and locks it each time (`getOwner` → `m_pWindow.lock()`). `setLuaTitle` holds a strong pointer only for the length of the call. | `barDeco.cpp:675`, `main.cpp:144` |
| Dangling bar pointer | The bar list holds weak pointers. `set_title` checks each one (`bar && …`) before use. A bar removes itself from the list in its destructor. | `main.cpp:148`, `barDeco.cpp:59-61` |
| List changed while being walked | None of the loops over `bars` can destroy a bar. `onConfigReloaded` and `updateRules` only mark state and request redraws. | `main.cpp:50`, `:136` |
| Pango markup injection | Titles are drawn with `pango_layout_set_text`, never `set_markup`, on both Hyprland main and v0.56.2. `<b>` shows as the letters `<b>`. This is the plain-text approach sway moved to in 2018 ("Use pango_layout_set_text instead of pango_layout_set_markup"), and the one Waybar reached by escaping titles. | Hyprland `src/render/Renderer.cpp`, `renderText` |
| Title drawn outside the bar | Clipped to the bar's box. | `barDeco.cpp:532` |
| A texture per title change | Built at most once per frame per bar, only when the title, size, colour or scale changed. The old texture is freed when replaced. Many calls in a row do not pile up textures. | `barDeco.cpp:572-577` |
| Oversized image | Cairo refuses images over 32,767 px on a side and returns a "nil" surface. Hyprland then makes an empty 0 × 0 texture, so the title simply disappears; it does not crash. Measured for Cairo; not run inside Hyprland. | Hyprland `src/render/GLRenderer.cpp`, `createTexture(cairo_surface_t*)` |
| Errors from Hyprland helpers | `windowFromLuaSelectorOrObject` and `configError` log and return; they never raise a Lua error. | Hyprland `LuaBindingsInternal.cpp` |
| Who can call `set_title` | Only processes that can open `$XDG_RUNTIME_DIR/hypr/<instance>/.socket.sock`. Its directory is mode 0700, so that means Kabelo's own processes. | live system |
| Threads | Lua, IPC and drawing all run on the compositor's main thread. `set_title` touches no shared state from another thread. | |
| Embedded NUL bytes | `std::string` built from `const char*` stops at the first NUL, so the bar shows less than was sent. That is harmless, and F2's fix (`lua_tolstring` with length) lets `BarTitle` reject it instead. | `main.cpp:152` |

## Step plan

Each step is small, ships on its own, and leaves the plugin working. They are ordered by safety gained.

1. **Read arguments before acquiring anything (F2-A).** In `setLuaTitle`, check argument 2's type with `lua_type`, read it with `lua_tolstring`, report with `configError`, and only then look up the window. Ship it with the CI grep that bans raising Lua calls under `hyprbars/` (F2-B).
2. **Add `BarTitle` (F1-A).** Add the new files `hyprbars/BarTitle.hpp/.cpp` with `from()`, rejecting titles over 1024 bytes, invalid UTF-8, control characters and direction controls. Change `setTitle` to take `std::optional<BarTitle>`. Add its unit tests and a CI job that builds and runs them under ASan/UBSan.
3. **Make every title safe to draw (F1-B, F5).** Add a draw-time cleaner, beside `BarTitle` because it applies the same rules: one line, cut on a character boundary, invalid bytes made valid. Call it in `renderPass` before `renderBarTitle`. Add a test that a 4 KB title with line breaks comes out one line and at most 1024 bytes.
4. **Fuzz both (seal for 2 and 3).** Add a libFuzzer target over `BarTitle::from` and the cleaner, run for a short time in CI.
5. **README (F3-A, F4).** Change the example to `stableid:`, and add the escaping rule and warning. In window-names, switch to `stableid:`, cut titles to the same 1024-byte limit before sending, and add the test that the request starts with `stableid:`.
6. **Offer steps 1–3 upstream** as a follow-up to #716, since upstream now ships the same code.
7. **Optional (F6).** Pin the CI workflow to a commit SHA if the fork ever holds secrets.

## Sources

- Hyprland wiki, IPC sockets and their synchronous handling: <https://wiki.hypr.land/IPC/>
- Hyprland wiki, `hyprctl eval`: <https://wiki.hypr.land/Configuring/Advanced-and-Cool/Using-hyprctl/>
- hyprwm/hyprland-plugins#716, the upstream change: <https://github.com/hyprwm/hyprland-plugins/pull/716>
- Hyprland source read for this review (main and v0.56.2): `src/render/Renderer.cpp` (`renderText`), `src/render/GLRenderer.cpp`, `src/config/lua/bindings/LuaBindingsInternal.cpp`, `src/config/lua/ConfigManager.cpp`, `src/ipc/s1/Unix.cpp`, `src/xwayland/XWM.cpp`, `src/desktop/state/ViewQuery.cpp`, `src/desktop/view/window/WindowMetadata.cpp` — <https://github.com/hyprwm/Hyprland>
- Lua 5.4 reference manual §4.4, errors use `longjmp` in C builds: <https://www.lua.org/manual/5.4/manual.html#4.4>
- cppreference, `std::longjmp` and skipped destructors: <https://en.cppreference.com/w/cpp/utility/program/longjmp>
- C++ Core Guidelines, resource management and ownership (R.1, R.20, F.7): <https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#S-resource>
- Pango, `pango_layout_set_text` and invalid UTF-8: <https://docs.gtk.org/Pango/method.Layout.set_text.html>
- Cairo image surfaces and nil surfaces: <https://www.cairographics.org/manual/cairo-Image-Surfaces.html>
- Waybar, escaping markup in window titles: <https://git.nixnet.services/blankie/Waybar/commit/157ea445102b0bc2db02733095a4350c7b2089d1>, and making markup opt-in: <https://git.nixnet.services/blankie/Waybar/commit/ea722615c46d07f5859d2c6a78caebf1f069bc2e>
- sway, moving titles from `pango_layout_set_markup` to `set_text` (2018): <https://git.marussy.com/sway/log/common/pango.c?h=v1.6>
- Terminal title injection, the class of bug where a displayed title is turned against the user: <https://www.openwall.com/lists/oss-security/2015/08/11/14>, <https://hdm.io/writing/termulation.txt>
- Trojan Source, CVE-2021-42574: <https://confluence.atlassian.com/security/multiple-products-security-advisory-unrendered-unicode-bidirectional-override-characters-cve-2021-42574-1086419475.html>
- CWE-400 Uncontrolled resource consumption: <https://cwe.mitre.org/data/definitions/400.html>
- CWE-770 Allocation without limits: <https://cwe.mitre.org/data/definitions/770.html>
- CWE-94 Code injection: <https://cwe.mitre.org/data/definitions/94.html>
- CWE-74 Injection: <https://cwe.mitre.org/data/definitions/74.html>
- CWE-416 Use after free: <https://cwe.mitre.org/data/definitions/416.html>
- CWE-401 Missing release of memory: <https://cwe.mitre.org/data/definitions/401.html>
- CWE-451 User interface misrepresentation: <https://cwe.mitre.org/data/definitions/451.html>
