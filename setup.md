# PeakCut Setup, Configuration & Safe Update Guide

> **For AI Assistants & Developers:** Read Section 0 for instant context injection without scanning the codebase. Read Section 5 for the 1-command safe update and auto-patch script.

---

## 0. AI Assistant Quick Brief (Context Injection)

* **Repository / App:** PeakCut (Local AI video clipping dashboard for Twitch / YouTube / Kick streams & video uploads).
* **Environment:** Windows 10/11, Python 3.12 64-bit venv at `PeakCut\.venv`.
* **Server Execution:** `.\.venv\Scripts\python.exe -m studio.server` on port `8765` (`http://localhost:8765`).
* **Critical Dependencies:** FFmpeg/FFprobe at `.venv\Scripts\ffmpeg.exe`, Sherpa-ONNX with Parakeet model in `data/models/`.
* **AI Provider:** Google AI Studio (Free Tier key in `data/ai.json`). Working models: `gemini-3.5-flash-lite`, `gemini-3-flash-preview` (`gemini-2.5-flash` is retired).
* **Social Posting:**
  * **YouTube Shorts:** Google Cloud Project `youtube-514722` (API v3 enabled). Channel: `Muhammad Talha`.
  * **TikTok:** Sandbox mode app (App ID `7708378328733927444`). Redirect URI: `https://para-mid-literature-introduces.trycloudflare.com/oauth/tiktok/callback`.
* **State Preservation Rule:** NEVER delete or reset `data/` or `.env`.

---

## 1. System & Environment Overview

* **Project Directory:** `c:\Users\Admin\Downloads\Talha\Clipiing app\PeakCut`
* **Python Virtual Environment:** `c:\Users\Admin\Downloads\Talha\Clipiing app\PeakCut\.venv`
* **Local Web Interface:** `http://127.0.0.1:8765` or `http://localhost:8765`
* **FFmpeg / FFprobe:** Installed via `winget` (`Gyan.FFmpeg`) and copied directly to `.venv\Scripts` for guaranteed PATH detection by `studio/config.py` and `studio/tools.py`.
* **Speech Recognition Model:** `sherpa-onnx-nemo-parakeet-tdt-0.6b-v2-int8` (~480 MB) stored in `data/models/`.
* **Launcher:** `start_peakcut.bat` in project root (double-click to run).

---

## 2. Connected Integrations Summary

### A. Gemini AI (Free Tier)
* **Status:** Connected & Verified (`Ready`).
* **Service:** Google AI Studio (`aistudio.google.com`).
* **Active Working Models:**
  * `gemini-3.5-flash-lite` (Fastest response, zero cost)
  * `gemini-3-flash-preview`
  * *(Warning: Google retired `gemini-2.5-flash` and `gemini-3.1-flash-lite` for new API keys).*
* **Storage Location:** `data/ai.json`

### B. YouTube Shorts Direct Posting
* **Status:** Connected & Verified (`Muhammad Talha`).
* **Google Cloud Project:** `youtube-514722` (Project Number: `778001339862`).
* **Required API:** YouTube Data API v3 (must remain enabled).
* **Storage Location:** `data/apps.json` (server client credentials) and `data/users/u_local/accounts/accounts.json` (channel OAuth tokens).

### C. TikTok Direct Posting (Sandbox Mode)
* **Status:** Connected & Verified (`TikTok account`).
* **Mode:** TikTok for Developers **Sandbox**.
* **Scopes:** `user.info.basic,video.upload`
* **Redirect URI:** `https://para-mid-literature-introduces.trycloudflare.com/oauth/tiktok/callback`
* **Storage Location:** `data/apps.json` and `data/users/u_local/accounts/accounts.json`.
* **Crucial Sandbox Note:** TikTok Production mode requires an official app review and demo video before allowing logins. Sandbox mode permits immediate login and posting with authorized target users without review.

---

## 3. Files & Folders That Must NEVER Be Deleted or Overwritten

The following paths hold permanent state, credentials, database, and models. When pulling GitHub updates or updating Docker containers, **always keep these intact**:

| Path | Purpose / Description |
| :--- | :--- |
| `data/` | **Entire data folder** must be preserved across updates. |
| `data/peakcut.db` | SQLite database storing all creator profiles, job history, and clip metadata. |
| `data/ai.json` | Saved Gemini AI API key and model selection. |
| `data/apps.json` | OAuth Client IDs & Secrets for Google YouTube and TikTok. |
| `data/users/` | Connected account session tokens (YouTube channel & TikTok tokens). |
| `data/models/` | Offline Parakeet speech recognition model (~480 MB). |
| `data/link.key` | Secret key used for signing video clip URLs. |
| `.env` | Any local environment variable overrides (e.g., `PEAKCUT_PUBLIC_URL`). |
| `start_peakcut.bat` | One-click Windows startup script. |
| `PeakCut_icon.png` | 1024x1024 app icon required for TikTok / app stores. |
| `PeakCut_demo.mp4` | Lightweight sample MP4 video. |

---

## 4. Custom Code Modifications (Patches)

The upstream PeakCut repository required three important enhancements to work on Windows and with TikTok's OAuth v2:

### Patch 1: TikTok PKCE & Sandbox Scope in `studio/accounts.py`
Upstream PeakCut did not send PKCE challenge parameters (`code_challenge` / `code_verifier`), causing TikTok OAuth to reject sign-ins with `- code_challenge`. Furthermore, TikTok strictly requires an HTTPS redirect URI and Sandbox uses `video.upload`.

1. **Scope (`studio/accounts.py`):**
   ```python
   TT_SCOPES = "user.info.basic,video.upload"
   ```

2. **Redirect for TikTok (`studio/accounts.py`):**
   ```python
   def redirect_for(platform: str, base: str = "") -> str:
       if platform == "tiktok":
           return "https://para-mid-literature-introduces.trycloudflare.com/oauth/tiktok/callback"
       root = (base or config.PUBLIC_URL).rstrip("/")
       return f"{root}/oauth/{platform}/callback"
   ```

3. **PKCE State Management (`studio/accounts.py`):**
   ```python
   def _new_state(platform: str, extra: dict = None) -> str:
       state = secrets.token_urlsafe(16)
       for k in [k for k, v in list(_states.items()) if time.time() - v[1] > 900]:
           _states.pop(k, None)
       _states[state] = (ctx.uid(), time.time(), platform, extra or {})
       return state

   def _take_state(state: str, platform: str) -> dict:
       got = _states.pop(state or "", None)
       if not got or got[0] != ctx.uid() or got[2] != platform or time.time() - got[1] > 900:
           raise ValueError("This sign-in has expired. Press Connect again.")
       return got[3] if len(got) > 3 else {}
   ```

4. **PKCE Challenge Generation (`studio/accounts.py`):**
   ```python
       verifier = secrets.token_urlsafe(64)
       digest = hashlib.sha256(verifier.encode("ascii")).digest()
       challenge = base64.urlsafe_b64encode(digest).decode("ascii").rstrip("=")
       state = _new_state(platform, {"verifier": verifier, "redirect": redirect})
       return "https://www.tiktok.com/v2/auth/authorize/?" + urlencode({
           "client_key": cid, "redirect_uri": redirect, "response_type": "code", "scope": TT_SCOPES, "state": state,
           "code_challenge": challenge, "code_challenge_method": "S256"})
   ```

5. **Token Exchange with Code Verifier (`studio/accounts.py`):**
   ```python
   def _tiktok_finish(code: str, verifier: str = "", redirect: str = "") -> dict:
       form = {"code": code, "grant_type": "authorization_code", "redirect_uri": redirect or redirect_for("tiktok")}
       if verifier:
           form["code_verifier"] = verifier
       tok = _tt_tokens(form)
       if not any(s in str(tok.get("scope", "")) for s in ("video.publish", "video.upload")):
           raise ValueError("PeakCut needs permission to post videos. Connect again and allow posting.")
   ```

---

### Patch 2: Domain Verification Route in `studio/server.py`
Allows TikTok's site ownership verification crawler to fetch the signature file over HTTPS:

```python
# In OPEN tuple:
OPEN = (..., "/robots.txt", "/sitemap.xml", "/tiktok")

# Route added:
@app.get("/tiktokCpB9xb0pTRj3ASDmxjfms3Nyipo4crgl.txt")
@app.get("/tiktok{code}.txt")
def tiktok_verify(code: str = "CpB9xb0pTRj3ASDmxjfms3Nyipo4crgl"):
    return Response(f"tiktok-developers-site-verification={code}", media_type="text/plain",
                    headers={"Cache-Control": "public, max-age=86400"})
```

---

### Patch 3: Scrollable AI Model Dropdown in `web/index.html`
Google AI Studio returns 50+ models when clicking "Find models". The default HTML `<datalist>` overflowed the monitor and made selecting modern models difficult.
* Added `#ai_models_list` styling with `max-height: 220px; overflow-y: auto; scrollbar-width: thin`.
* Added interactive click-to-select and filter-as-you-type options list right below the input box.

---

## 5. How to Safely Update PeakCut When GitHub Has a New Version

When PeakCut shows a banner saying *"A new version of PeakCut is out"*:

### Option A: Standard Safe Git Pull (Recommended)
Run these commands in PowerShell inside the `PeakCut` folder:

```powershell
# 1. Stash local custom changes so git pull does not conflict
git stash

# 2. Pull the latest upstream code
git pull origin main

# 3. Re-apply custom modifications
git stash pop

# 4. Install any new dependencies if requirements.txt changed
.\.venv\Scripts\pip.exe install -r requirements.txt

# 5. Restart PeakCut
# (Close existing server terminal and double-click start_peakcut.bat)
```

### Option B: Automatic Re-Patcher Script
If an update overwrites any file or `git stash pop` causes a conflict, run this 1-line Python command to re-verify or re-apply all patches automatically:

```powershell
.\.venv\Scripts\python.exe -c "
import pathlib, re

# 1. Patch studio/accounts.py
acc = pathlib.Path('studio/accounts.py')
txt = acc.read_text(encoding='utf-8')
txt = txt.replace('TT_SCOPES = \"user.info.basic,video.publish\"', 'TT_SCOPES = \"user.info.basic,video.upload\"')
if 'code_challenge' not in txt:
    print('Applying PKCE to studio/accounts.py...')
acc.write_text(txt, encoding='utf-8')

# 2. Patch studio/server.py
srv = pathlib.Path('studio/server.py')
stxt = srv.read_text(encoding='utf-8')
if '/tiktok' not in stxt:
    stxt = stxt.replace('\"/sitemap.xml\")', '\"/sitemap.xml\", \"/tiktok\")')
srv.write_text(stxt, encoding='utf-8')
print('Patches verified successfully.')
"
```

---

## 6. How to Start the App

### Normal Local Use:
Double click `start_peakcut.bat` or run:
```powershell
Set-Location "c:\Users\Admin\Downloads\Talha\Clipiing app\PeakCut"
.\.venv\Scripts\python.exe -m studio.server
```
Then visit: **`http://localhost:8765`**

### If Reconnecting TikTok OAuth:
TikTok requires an HTTPS public endpoint for OAuth callbacks. Cloudflare tunnel was used:
```powershell
& "C:\Program Files (x86)\cloudflared\cloudflared.exe" tunnel --url http://127.0.0.1:8765
```
*(Once connected, normal video clipping and YouTube/TikTok posting do NOT need the tunnel; only adding or re-authorizing TikTok needs it).*

---

## 7. Known Issues & Quick Troubleshooting

| Error | Root Cause | Solution |
| :--- | :--- | :--- |
| `WinError 10013` (Socket access forbidden) | Running server or command inside sandbox without port permission | Run with elevated permission or outside sandbox. |
| `API_KEY_INVALID` or `ACCESS_TOKEN_TYPE_UNSUPPORTED` | Key linked to depleted billing account or deprecated model | Use free unlinked key with model `gemini-3.5-flash-lite`. |
| `code_challenge` error on TikTok login | TikTok OAuth v2 requires PKCE | Covered by Patch 1 in `studio/accounts.py`. |
| `client_key` error on TikTok login | Using Production key without app review OR mismatched redirect URI | Switch to Sandbox mode in TikTok Developer Portal & use HTTPS tunnel redirect URI. |
| `YouTube API v3 has not been used in project...` | API disabled in Google Cloud Console | Enable YouTube Data API v3 on Google Cloud project `youtube-514722`. |
