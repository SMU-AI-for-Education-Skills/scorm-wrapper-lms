---
name: scorm-wrapper-lms
description: "Wraps any self-contained HTML learning resource or activity (an uploaded HTML file or one the assistant has built: explainer, quiz, simulation, case study, interactive diagram, game, tool, etc.) in a SCORM 1.2 package so it can be uploaded to and displayed inside the LMS (SMU eLearn / D2L Brightspace). Packages are display-only by default; tracking capabilities (completion, score, saved progress, review/retake) are offered only where relevant to the resource and only added when the user explicitly opts in after a warning. Use this skill whenever the user mentions SCORM, LMS packaging, or uploading or embedding a resource or activity in eLearn, Brightspace, D2L or another LMS."
---

# SCORM Wrapper

Packages a self-contained HTML resource as a SCORM 1.2 `.zip` that displays inside the LMS. Works for any kind of resource or activity; decisions depend on what the resource *does*, not what it is called.

## Principle: display only unless the user opts in

Which SCORM tracking features are enabled on SMU's eLearn is not documented. A package can look as if it works (the content displays and saves appear to succeed in the browser) while the LMS does not keep the data. So by default this skill promises only one thing: **displaying the resource inside the LMS**.

A display-only package opens and closes a session with the LMS, so the player doesn't report a communication error, and writes no tracking data. Completion, score, saved progress and review/retake are added only when the user explicitly chooses them after the warning in Step 3.

## Before you start: what this environment can do

This skill is platform-neutral. Check what the current environment supports and adapt:

- **Can create files and run code:** build, check and zip the package yourself (Steps 4–6), then give the user the `.zip` as a downloadable file.
- **Can create files but can't zip them:** give the user `index.html`, `scorm.js` and `imsmanifest.xml` (plus any bundled files) as separate downloads, with the zipping instructions below.
- **Text only:** output each file in full in its own code block, labelled with its exact file name, with the zipping instructions below. Never abbreviate or elide code.

**Zipping instructions for the user.** Save all the files in one folder. Select **the files themselves, not the folder**, and compress them into a single `.zip`:
- **Windows:** right-click the selected files, then **Send to → Compressed (zipped) folder**.
- **macOS:** right-click the selected files, then **Compress**.

`imsmanifest.xml` must sit at the top level of the zip, not inside a subfolder.

## Step 1: Get the resource

Work on a copy of the HTML; never modify the user's original. Assume the LMS is **SMU eLearn (D2L Brightspace)** unless the user says otherwise.

## Step 2: Inspect the resource

Read the HTML before asking anything. Establish three things.

**1. Self-containment.** List anything loaded from the network: `<script src="https://…">`, stylesheets, web fonts, remote images, `fetch()` calls.
- **Scripts:** plan to bundle them into the package, by obtaining the exact version from the library's official distribution (for example its npm package) and pointing the page at the local file. If you can't download it, tell the user which file to download and where to put it. An LMS may block or slow third-party hosts.
- **Data from external APIs:** flag it; it may not work from inside the LMS.

**2. Frame-sensitive features.** File downloads (Blob links with `download`), printing, pop-ups or new windows, `localStorage`/`sessionStorage`, camera or microphone, and full-height `100vh` layouts can all behave differently inside an LMS frame. Note them for the handoff; don't silently remove them.

**3. Which capabilities are relevant.** Judge from behaviour, using this table. Offer only the rows that apply.

| If the resource… | Relevant option |
|---|---|
| Has a clear end point: a final screen, a results or summary view, the end of a path or sequence, or a natural place for a "Finish" button | **Completion reporting** |
| Produces a score: graded answers, a points total, a performance measure that converts to a percentage | **Score reporting** (implies completion) |
| Holds state a learner would lose by leaving: progress through steps, answers or text entered, choices made, items created, edited, moved or annotated, settings adjusted | **Saved progress (resume)** |
| Gives feedback on answers and reaches an end or results screen | **Review & retake buttons** |
| Does none of the above: read-only, view-only or purely exploratory with nothing to keep | *None.* Package as display-only without asking. |

## Step 3: Present the options and get an explicit decision

If no option is relevant, tell the user it will be display-only and go to Step 5. Otherwise, before building:

1. Show this warning as written:

   > **Before you choose:** which SCORM tracking features are switched on in SMU's eLearn isn't documented. A package can look as if it works (the content displays and saves appear to succeed) while eLearn doesn't actually keep the data. If you intend to rely on any of these features (for example for grades, completion requirements or students' saved work), please check with the LMS support team at **lms_support@smu.edu.sg** before using the resource with students.

2. Present **only the relevant options**, with **"None: display only (recommended default)"** first. If the platform offers a structured way to ask (clickable options, a form, a multi-select), use it; otherwise use a short numbered list and ask the user to reply with their choices. Describe each option in one line in terms of what the instructor gets, filled in for this resource:
   - *Completion reporting:* eLearn marks it complete when the learner [specific end point].
   - *Score reporting:* the score (0–100) is sent to eLearn and can feed a grade item; pass mark 80 unless you say otherwise.
   - *Saved progress:* learners can leave and come back to [what is kept].
   - *Review & retake buttons:* the end screen offers a read-only review and a full restart.

3. If the user picks completion (or score) **and** saved progress, add: once the resource is reported complete, eLearn may start a fresh attempt next time, so saved progress is only reliable until completion.

4. **Wait for the answer.** Don't build, and don't infer a choice from the general request ("put this in eLearn" is not a choice). Only an explicit choice made in this conversation for this resource counts.

## Step 4: Add the runtime and integrate the chosen capabilities

Save this file as `scorm.js` next to the resource's `index.html`. Use it as written rather than writing a new wrapper: it encodes fixes for real failures (see Pitfalls).

```javascript
/* SCORM 1.2 runtime. DISPLAY-ONLY unless window.SCORM_CONFIG (defined BEFORE this file loads)
   switches capabilities on: {completion, score, resume, masteryScore}. Calls for capabilities
   that are off, or with no LMS present, are safe no-ops. Ctrl+Shift+Alt+D: diagnostics. */
var SCORM = (function () {
  var cfg = window.SCORM_CONFIG || {};
  var C = { completion: !!(cfg.completion || cfg.score), score: !!cfg.score, resume: !!cfg.resume,
            masteryScore: typeof cfg.masteryScore === "number" ? cfg.masteryScore : 80 };
  var API = null, ready = false, finished = false, d = { log: [] }, panel = null;
  function scan(w) { var n = 0; try { while (w && !w.API && w.parent && w.parent !== w && n < 10) { w = w.parent; n++; } return (w && w.API) || null; } catch (e) { return null; } }
  function findAPI() { var a = scan(window); if (!a && window.opener) a = scan(window.opener); try { if (!a && window.top.opener) a = scan(window.top.opener); } catch (e) {} return a; }
  function err() { try { var c = API.LMSGetLastError(); return c + (c && c !== "0" ? " " + API.LMSGetErrorString(c) : ""); } catch (e) { return "n/a"; } }
  function note(m) { d.log.push(new Date().toLocaleTimeString() + "  " + m); if (d.log.length > 40) d.log.shift(); draw(); }
  function set(k, v) { var ok = String(API.LMSSetValue(k, v)) === "true"; if (!ok) note("Set " + k + " REJECTED (error " + err() + ")"); return ok; }
  function init() {
    if (ready) return true;
    API = findAPI(); d.api = !!API;
    if (!API) { note("No SCORM API found: not launched from an LMS."); return false; }
    try {
      note("LMSInitialize -> " + API.LMSInitialize("") + " (error " + err() + ")");
      d.entry = API.LMSGetValue("cmi.core.entry"); d.mode = API.LMSGetValue("cmi.core.lesson_mode");
      d.status = API.LMSGetValue("cmi.core.lesson_status"); d.saved = (API.LMSGetValue("cmi.suspend_data") || "").length;
      note("entry=" + d.entry + ", mode=" + d.mode + ", status=" + d.status + ", saved data=" + d.saved + " chars");
      if (C.completion && (d.status === "not attempted" || d.status === "")) set("cmi.core.lesson_status", "incomplete");
      if (C.resume) set("cmi.core.exit", "suspend");
      if (C.completion || C.resume) API.LMSCommit("");
      ready = true;
    } catch (e) { note("init failed: " + e); }
    return ready;
  }
  function terminate() { // runs after the content's own pagehide handler (registered on load)
    if (!ready || finished) return; finished = true;
    try { if (C.resume) API.LMSSetValue("cmi.core.exit", "suspend"); if (C.completion || C.resume) API.LMSCommit(""); API.LMSFinish(""); } catch (e) {}
  }
  window.addEventListener("load", function () { init(); window.addEventListener("pagehide", terminate); window.addEventListener("unload", terminate); });
  function complete() { if (!C.completion || !ready || finished) return; set("cmi.core.lesson_status", "completed"); API.LMSCommit(""); note("reported completed"); }
  function score(raw) {
    if (!C.score || !ready || finished) return;
    var s = Math.max(0, Math.min(100, Math.round(Number(raw) || 0)));
    set("cmi.core.score.min", "0"); set("cmi.core.score.max", "100"); set("cmi.core.score.raw", String(s));
    set("cmi.core.lesson_status", s >= C.masteryScore ? "passed" : "failed"); API.LMSCommit(""); note("reported score " + s);
  }
  function retake() { if (!ready || finished) return; if (C.completion) set("cmi.core.lesson_status", "incomplete"); if (C.resume) set("cmi.suspend_data", ""); if (C.completion || C.resume) API.LMSCommit(""); note("retake: progress cleared"); }
  function trySet(str) {
    try {
      if (String(API.LMSSetValue("cmi.suspend_data", str)) !== "true") { note("save " + str.length + " chars REJECTED (error " + err() + ")"); return false; }
      var back = API.LMSGetValue("cmi.suspend_data");
      if (back !== str) { note("save " + str.length + " chars TRUNCATED to " + (back || "").length); return false; }
      note("saved " + str.length + " chars, commit -> " + API.LMSCommit("")); return true;
    } catch (e) { note("save threw: " + e); return false; }
  }
  function saveState(s) { // object, or array of JSON strings fullest-first; returns index stored or -1
    if (!C.resume || !ready || finished) return -1;
    var c = Array.isArray(s) ? s : [JSON.stringify(s)];
    for (var i = 0; i < c.length; i++) if (trySet(c[i])) return i;
    return -1;
  }
  function loadState() {
    if (!C.resume || !ready) return null;
    try { var raw = API.LMSGetValue("cmi.suspend_data"); if (!raw) { note("no saved state"); return null; } note("restored " + raw.length + " chars"); return JSON.parse(raw); }
    catch (e) { note("restore failed: " + e); return null; }
  }
  function clearState() { if (!C.resume || !ready || finished) return; try { API.LMSSetValue("cmi.suspend_data", ""); API.LMSCommit(""); note("cleared saved state"); } catch (e) {} }
  function draw() {
    if (!panel || panel.hidden) return;
    var on = ["completion", "score", "resume"].filter(function (k) { return C[k]; });
    panel.lastChild.textContent = "Capabilities: " + (on.length ? on.join(", ") : "none (display only)") +
      "\nAPI found: " + d.api + "\nentry at launch: " + d.entry + "\nlesson mode: " + d.mode +
      "\nstatus at launch: " + d.status + "\nsaved data at launch: " + d.saved + " chars\n\n" + d.log.join("\n");
  }
  document.addEventListener("keydown", function (e) {
    if (!(e.ctrlKey && e.shiftKey && e.altKey && e.code === "KeyD")) return;
    e.preventDefault();
    if (!panel) {
      panel = document.createElement("div");
      panel.setAttribute("style", "position:fixed;right:10px;bottom:10px;z-index:99999;width:min(440px,90vw);max-height:60vh;overflow:auto;background:#1f2328;color:#e6edf3;font:12px/1.4 ui-monospace,Menlo,Consolas,monospace;border-radius:8px;padding:10px");
      panel.innerHTML = '<b>SCORM diagnostics</b> <button style="float:right;font:inherit">Close</button><pre style="white-space:pre-wrap;margin:6px 0 0"></pre>';
      panel.querySelector("button").onclick = function () { panel.hidden = true; };
      document.body.appendChild(panel);
    } else panel.hidden = !panel.hidden;
    draw();
  });
  return { config: C, init: init, isReady: function () { return ready; }, terminate: terminate, complete: complete,
           score: score, retake: retake, saveState: saveState, loadState: loadState, clearState: clearState };
})();
```

### Load it

Insert these two tags at the top of `<head>`, right after `<meta charset>`. List only the chosen keys; display-only uses `{}`.

```html
<script>window.SCORM_CONFIG = {};</script>
<script src="scorm.js"></script>
```

Config examples: `{"resume": true}`, `{"completion": true}`, `{"score": true, "masteryScore": 70}`. Score implies completion.

For **display-only, stop here**: the resource needs no other changes.

### Integrate the chosen capabilities

Each runtime call does nothing when its capability is off, but add only the calls for what was chosen.

**Completion.** Call `SCORM.complete()` once, at the end point named in Step 3. If the resource has no natural end, add a "Finish" button and call it there. Never call it on launch: a completed attempt can relaunch as a fresh attempt, which discards saved progress.

**Score.** At the results point, call `SCORM.score(percent)` with an integer 0–100. It sets passed/failed against `masteryScore`.

**Saved progress (resume).**
- **Restore at start.** Call `SCORM.init()` then `SCORM.loadState()` before the first render. If state comes back, rebuild the UI from it. Skip any first-load step that would overwrite it, such as randomisation, shuffling, auto-layout or intro animations.
- **Save on change.** Call `SCORM.saveState(...)` after each meaningful change (answer, navigation, edit, drag end), debounced by about 300–700 ms. Also call it on `pagehide`, `beforeunload`, and `visibilitychange` when the page becomes hidden. Register these handlers in the resource's own script: the runtime closes the session on `pagehide` *after* the resource's handler, so the final save lands first.
- **Keep state under 4,096 characters** (all SCORM 1.2 guarantees). Store only what changed from the original, use short keys, and round numbers. If state could grow, pass an array of JSON strings ordered fullest-first, dropping the least important parts (such as layout or scroll position) in later entries, so the runtime falls back when the LMS rejects the larger one.
- **Show the result.** Add a small status line: "Progress saved in eLearn", "Not opened from eLearn, so progress isn't saved", or a warning when nothing fitted. Never claim it is saved permanently: the runtime can only confirm that eLearn accepted the data this session.
- **Offer a reset.** Add a "Start over" / "Reset" control that calls `SCORM.clearState()` and restores the original state.

**Review & retake.** Add both to the end or results screen.
- **Review** shows the learner's answers and feedback read-only, with an "Exit review" button. It makes no SCORM calls.
- **Retake** calls `SCORM.retake()`, resets all state, and starts over; completion or score is reported again at the end.

**If the resource can save or download a copy of itself,** remove the two injected `<script>` tags from the copy. Guard the copy with `if (typeof SCORM === "undefined") window.SCORM = {init(){return false}, isReady(){return false}, complete(){}, score(){}, saveState(){return -1}, loadState(){return null}, clearState(){}, retake(){}};`. Inside inline JavaScript, write a closing script tag in a string as `<\/script>`.

## Step 5: Write the manifest and zip

Create `imsmanifest.xml` beside `index.html`.
- Generate a fresh UUID.
- Escape `&`, `<` and `>` in the title.
- List every file in the package (`index.html`, `scorm.js`, plus any bundled libraries or media).
- Include the `<adlcp:masteryscore>` line **only** when score reporting is on.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<manifest identifier="pkg-UUID" version="1.0"
  xmlns="http://www.imsproject.org/xsd/imscp_rootv1p1p2"
  xmlns:adlcp="http://www.adlnet.org/xsd/adlcp_rootv1p2"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://www.imsproject.org/xsd/imscp_rootv1p1p2 imscp_rootv1p1p2.xsd http://www.adlnet.org/xsd/adlcp_rootv1p2 adlcp_rootv1p2.xsd">
  <metadata><schema>ADL SCORM</schema><schemaversion>1.2</schemaversion></metadata>
  <organizations default="org1">
    <organization identifier="org1">
      <title>TITLE</title>
      <item identifier="item1" identifierref="res1">
        <title>TITLE</title>
        <adlcp:masteryscore>80</adlcp:masteryscore>
      </item>
    </organization>
  </organizations>
  <resources>
    <resource identifier="res1" type="webcontent" adlcp:scormtype="sco" href="index.html">
      <file href="index.html"/>
      <file href="scorm.js"/>
    </resource>
  </resources>
</manifest>
```

Zip the files (not their folder) so `imsmanifest.xml` sits at the root, for example `cd build && zip -X -r ../<name>-scorm.zip imsmanifest.xml index.html scorm.js [other files]`. If you can't zip, follow "Before you start".

## Step 6: Check before delivering

Do whichever of these checks the environment allows, and tell the user which ones you couldn't run.

- **Syntax.** Check each inline `<script>` with a JavaScript syntax checker (for example `node --check`) if one is available; otherwise read it carefully. A stray `</script>` inside a string is the usual culprit.
- **Display-only:** confirm `SCORM_CONFIG` is `{}` and the resource has no `SCORM.` calls.
- **Capabilities on, if a headless browser is available:** serve the folder over HTTP with a parent page that defines a mock `window.API` (the eight `LMS*` functions over an in-memory object) and loads `index.html` in an iframe.
  1. Act as a learner, then reload the iframe to relaunch.
  2. Confirm the expected `LMSSetValue` calls and that `LMSFinish` ran after the final save.
  3. For resume, confirm the state came back.
  4. Make the mock start a fresh attempt when the previous status was completed, passed or failed, to catch completion breaking resume.

## Step 7: Deliver

Give the user the `.zip` (or the files and zipping instructions) and tell them to upload the `.zip` itself, not its unzipped contents.

**Upload to eLearn (D2L Brightspace):**
1. Go to **Content** and open the module.
2. Click **Add Existing**, then **SCORM / xAPI Object**, and upload the `.zip`.
3. Set the title.
4. Create a grade item only if score reporting was chosen.
5. If asked about **Course Package Review/Retake Options**, choose **Do not add review/retake**.
6. Save.

**Other LMS:** use its SCORM import and the same choices.

**Removing it from eLearn:**
1. Delete the topic from the module.
2. To remove it fully, also delete it from **Add Existing → SCORM/xAPI Object → ⋮ → Delete**.
3. Deleting discards learners' saved progress.

**Handoff note.** End every delivery with a short note covering:
- **Contents:** what was wrapped.
- **Capabilities:** "Display only" or the chosen list, with each trigger.
- **Warning (if any capability is on):** repeat that it may not work as intended in eLearn and to check with **lms_support@smu.edu.sg** before relying on it. Suggest a precise question, for example: *"Is SCORM 1.2 run-time data (`cmi.suspend_data` / `cmi.core.lesson_status` / `cmi.core.score.raw`) stored for student launches, and does relaunching a SCORM object resume the existing attempt?"*
- **Frame-sensitive features:** any found in Step 2 and what was done about them.
- **Testing in eLearn:** use a **student account** (instructor previews may not be tracked) and, for resume, change something, leave, and return. **Ctrl+Shift+Alt+D** opens the diagnostics panel; on relaunch, `entry at launch: resume` and non-zero `saved data at launch` mean saved progress is working.

## Pitfalls

These were all found in real use:

- **Reporting "completed" on launch breaks resume:** the next launch can be a fresh attempt or a review.
- **Missing `LMSFinish`:** without it, some players never write data to the server. The runtime calls it on `pagehide`; don't call it earlier, or the LMS may replace the content with its own completion screen.
- **A literal `</script>` inside an inline script string** ends the script block and silently breaks the page. Write `<\/script>`.
- **Debounced saves are lost on quick exits** unless flushed on `pagehide`.
- **CDN scripts may be blocked or slow inside the LMS.** Bundle them.
- **The browser can't confirm the LMS kept the data.** A successful set only means the LMS accepted it this session; only a relaunch proves it was kept.
