## Purpose
The scorm-wrapper-lms skill allows HTML-based learning activities to be hosted on eLearn or other learning management systems, by packaging them into SCORM 1.2-compliant file.

## Using the skill 
You may download the latest version of this skill as a ZIP file under `Releases` on the right of this Github Repository. You may refer to our guide to learn how to [use this skill within your AI tools](https://github.com/SMU-AI-for-Education-Skills#accessing-and-using-skills). 

## How this works
The skill inspects an HTML learning resource, adds the necessary SCORM files, and bundles everything into a ZIP ready for LMS upload. By default, it only displays the resource; optional capabilities associated with SCORM such as tracking features can be added with the user’s approval.

## Inputs Required
* An HTML learning resource to package—either an uploaded self-contained HTML file or one created from your specifications.
* The target LMS, only if it is not SMU eLearn.
* An explicit tracking choice, or other feature, if relevant: display only, completion, score, saved progress, and/or review and retake. Display-only needs no additional choice.

## Notes and Limitations
* Display-only by default: Tracking is added only with explicit user approval.
* Tracking is not guaranteed: SMU eLearn’s support for SCORM completion, scores, and saved progress is undocumented; confirm with lms_support@smu.edu.sg before relying on it.
* Saved progress is limited: SCORM 1.2 guarantees only 4,096 characters of saved-state data.
* External dependencies may fail: CDN scripts, external APIs, fonts, and media should ideally be bundled locally.
* LMS framing can affect features: Downloads, printing, pop-ups, browser storage, camera/microphone access, and full-screen layouts may behave differently.
* Testing is essential: Test using a student account, especially for tracking and resume functionality.
* The final ZIP must contain imsmanifest.xml at its root and should be uploaded without unzipping.

## Attribution
* Author:
Shamie Jegan, AI for Learning Innovation Specialist, Centre for Teaching Excellence
Isaac Koh, Pedagogical Innovations Lead, Centre for Teaching Excellence

## Contributing to the repository  
You may contribute to our repository of skills by suggesting modifications to an existing skill or by proposing a new skill. Please refer to the [guidelines on contribution](https://github.com/SMU-AI-for-Education-Skills#contributing-to-the-skills-repository). 
