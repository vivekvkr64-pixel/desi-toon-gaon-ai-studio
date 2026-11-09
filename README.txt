DESI TOON GAON AI YOUTUBE STUDIO v1
===================================

PACKAGE CONTENTS
----------------
1. index.html       Mobile-responsive dashboard prototype
2. BUILD_SPEC.md    Technical architecture and implementation plan
3. README.txt       This file

QUICK START
-----------
1. Download and extract Desi_Toon_Gaon_AI_YouTube_Studio_v1.zip.
2. Open index.html in Chrome on Android or any modern desktop browser.
3. Use the left navigation (on mobile, tap the menu icon) to open all dashboard sections.
4. Try Research brief creation, Content Plan slot creation, Project draft creation,
   and SEO checklist drafting. These prototype interactions run in the browser only.

IMPORTANT STATUS
----------------
This is a real, runnable frontend prototype plus a backend build specification.
It is NOT a deployed backend and is NOT connected to YouTube or Google OAuth.
No live analytics, trend results, competitor results, uploads, publishing, or
monitoring are claimed. Missing data is shown as "—", "Not connected", or
"Prototype" rather than invented.

SECURITY
--------
- Never paste Google Client Secret, access token, or refresh token into index.html.
- Do not store OAuth tokens in localStorage/sessionStorage.
- Follow the server-side OAuth, token vault, and approval architecture in BUILD_SPEC.md.
- Upload, publish, delete, visibility changes, metadata changes, and public comment
  replies each require a separate explicit approval in the production system.

SIX-HOUR MONITORING
-------------------
The package documents a 6-hour scheduler architecture. No monitoring job is active
until a backend, scheduler, OAuth connection, and alert/report delivery are deployed.

CUSTOMIZATION
-------------
The prototype's channel, categories, labels, and settings are preconfigured for
Desi Toon Gaon. Browser-local draft entries are temporary and are not a durable
database. For production persistence and AI generation, implement the APIs and
data model in BUILD_SPEC.md.

TROUBLESHOOTING
---------------
- If opened from a file manager and a browser blocks a feature, open index.html
  directly in Chrome.
- Clipboard support may require HTTPS or localhost; copy manually if unavailable.
- The prototype has no external JavaScript dependencies and does not need a build step.

VERSION
-------
Desi Toon Gaon AI YouTube Studio v1
Frontend prototype package; backend integration pending.
