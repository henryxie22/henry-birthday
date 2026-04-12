# Event Invite Generator

A skill for generating beautiful, themed, single-page HTML event invitations with RSVP functionality connected to Google Sheets.

---

## How to Use This Skill

When a user asks to create an event invite, ask the following questions before generating anything. Collect all answers first, then produce the full `index.html` in one shot.

---

## Intake Questions

Ask these one group at a time for a conversational feel:

### 1. Event Basics
- What is the **event name**? (e.g. "Henry's 4th Birthday", "Sarah's Baby Shower", "Team Offsite 2026")
- What **type of event** is it? (birthday, baby shower, wedding, graduation, corporate, holiday party, etc.)
- Who is the **honoree or host name**?

### 2. Date & Time
- What is the **date**? (day of week, month, day, year)
- What are the **start and end times**?

### 3. Location
- What is the **venue name**?
- What is the **full address**?
- Is there a **Google Maps link**?
- Any **parking notes**?

### 4. What's Included
- Will food/drinks be provided? If yes, what?
- Any other inclusions guests should know about?

### 5. Important Reminders
- Any **rules or requirements** guests must follow? (e.g. dress code, waivers, socks required)
- Any **logistics** to highlight? (parking, baby facilities, accessibility, etc.)

### 6. RSVP
- What is the **RSVP deadline**?
- What is the **host's email address** for notifications?
- Do they have a **Google Apps Script URL** already? If not, note the setup steps below.
- Should the form collect: adults/kids/infants count? Dietary restrictions? A message to the honoree?

### 7. Theme & Style
- Do they have a **reference image or style** in mind? (ask them to attach one)
- What is the **color palette**? Or should it be auto-derived from the theme?
- Any **character or motif**? (e.g. Peppa Pig, dinosaurs, florals, minimalist, etc.)
- Any **photos** to include? (hero banner image, honoree photo)

### 8. Contact Info
- Who should guests contact with questions? (name + email)

---

## Technical Architecture

### Stack
- Pure HTML/CSS/JS — single `index.html` file, no frameworks, no build tools
- Google Fonts for typography
- Google Apps Script as the RSVP backend
- GitHub + GitHub Pages for free hosting

### File Structure
```
project-name/
├── index.html        # Everything lives here
└── pic/
    ├── banner.jpg    # Hero background image
    └── honoree.jpg   # Circular profile photo (optional)
```

### Key HTML Sections (in order)
1. `<head>` — fonts, CSS variables, all styles
2. `#hero` — banner image + overlay + honoree photo + title + date
3. `.emoji-strip` (optional) — decorative icon row
4. `#details` — date/time/venue/space cards
5. `#food` — what's included pills
6. `#reminders` — good-to-know items
7. `#rsvp` — form with Google Sheets integration
8. `<footer>` — summary + contact info

---

## CSS Design System

### Color Palette Strategy
Define all colors as CSS variables in `:root`. Derive from the theme:

| Theme Type | Background | Accent | Text |
|---|---|---|---|
| Kids birthday | Warm cream `#f7f3ec` | Bold red/coral | Near-black `#1a1a1a` |
| Baby shower | Soft blush `#fdf0f3` | Dusty rose | Warm gray |
| Elegant/wedding | Off-white `#fafaf8` | Gold `#c9a84c` | Charcoal |
| Corporate | White | Brand color | Dark gray |
| Tropical/summer | Light aqua `#f0faf8` | Teal + coral | Dark teal |

### Typography Pairing
- **Serif display** (Playfair Display, Cormorant, Lora) for headings — gives warmth and elegance
- **Clean sans-serif** (DM Sans, Inter, Nunito Sans) for body — readable at small sizes
- Load from Google Fonts

### Hero Section Pattern
```css
.hero-img-wrap img {
  width: 100%;
  max-height: 400px;
  object-fit: cover;
  object-position: center 70%; /* adjust to frame the image well */
}
.hero-img-wrap::after {
  /* fade bottom of image into page background */
  background: linear-gradient(transparent, var(--bg));
}
```
- Honoree photo sits in a circular frame overlapping the hero image bottom
- Title uses large serif font, subtitle uses spaced uppercase sans-serif

### Card Pattern
```css
.card {
  background: white;
  border-radius: 16px;
  border: 1.5px solid var(--border);
  box-shadow: 0 2px 16px rgba(0,0,0,.05);
}
```

### Scroll Reveal
```js
const obs = new IntersectionObserver(els => {
  els.forEach(e => { if(e.isIntersecting){ e.target.classList.add('visible'); obs.unobserve(e.target); }});
}, { threshold: 0.08 });
document.querySelectorAll('.reveal').forEach(el => obs.observe(el));
```
Add `.reveal` class to each section, `.reveal.visible { opacity:1; transform:none }`.

---

## RSVP + Google Sheets Integration

### Frontend (in index.html)
```js
const SHEET_URL = 'YOUR_APPS_SCRIPT_URL';

async function submitRSVP() {
  // validate fields...
  const payload = { timestamp, name, email, childName, attending, adults, kids, infants, dietary, note };
  await fetch(SHEET_URL, { method: 'POST', mode: 'no-cors', body: JSON.stringify(payload) });
  // mode: 'no-cors' is required — Apps Script returns opaque response
  showSuccess(attending, name);
}
```

> **Important:** Always use `mode: 'no-cors'` — without it the browser throws a CORS error even though the data was saved successfully.

### Backend (Google Apps Script)
Paste into Extensions → Apps Script in the Google Sheet:

```js
const NOTIFY_EMAIL = 'host@example.com';

function doPost(e) {
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
  const d = JSON.parse(e.postData.contents);

  sheet.appendRow([
    d.timestamp, d.name, d.email, d.childName,
    d.attending, d.adults, d.kids, d.infants,
    d.dietary, d.note
  ]);

  const subject = d.attending === 'yes'
    ? `🎉 New RSVP: ${d.name} is coming!`
    : `😢 RSVP: ${d.name} can't make it`;

  const body = `Name: ${d.name}\nEmail: ${d.email}\nAttending: ${d.attending}\nAdults: ${d.adults} | Kids: ${d.kids} | Infants: ${d.infants}\nDietary: ${d.dietary || 'none'}\nMessage: ${d.note || 'none'}`;

  MailApp.sendEmail(NOTIFY_EMAIL, subject, body);

  return ContentService
    .createTextOutput(JSON.stringify({ status: 'ok' }))
    .setMimeType(ContentService.MimeType.JSON);
}
```

### Deployment Steps
1. Create a new Google Sheet
2. Add column headers in row 1: `Timestamp | Name | Email | Child Name(s) | Attending | Adults | Kids | Infants | Dietary | Message`
3. Extensions → Apps Script → paste script → Save
4. Deploy → New Deployment → Web App
   - Execute as: **Me**
   - Who has access: **Anyone**
5. Copy the Web App URL → paste into `SHEET_URL` in `index.html`

### Linking the Web App URL to index.html

After deploying, the Web App URL looks like:
```
https://script.google.com/macros/s/AKfycb.../exec
```

Find this line in `index.html` and replace the placeholder:
```js
// Before
const SHEET_URL = 'YOUR_APPS_SCRIPT_URL';

// After
const SHEET_URL = 'https://script.google.com/macros/s/AKfycb.../exec';
```

> **Note:** Every time you redeploy the Apps Script (e.g. after editing it), Google generates a **new URL**. Make sure to update `SHEET_URL` in `index.html` and push to GitHub again whenever this happens.

### Testing the Connection
1. Open the live page and submit a test RSVP
2. Check the Google Sheet — a new row should appear within seconds
3. Check the notification email inbox for the alert
4. If the form shows "Something went wrong" but the sheet still received data, the `mode: 'no-cors'` fix is needed (see Common Pitfalls below)

---

## Hosting on GitHub Pages

1. Create a new GitHub repo (public)
2. Push `index.html` and `pic/` folder
3. Settings → Pages → Branch: `main`, folder: `/ (root)` → Save
4. Live URL: `https://{username}.github.io/{repo-name}/`

### Image Size Warning
GitHub HTTP push limit is ~50MB but large images cause HTTP 400 errors in practice. Keep each image under 500KB. If push fails:
- Compress images further
- Or use `git filter-branch` to purge large files from history before re-pushing

---

## Theming Checklist

When adapting for a new event, change these:

- [ ] CSS variables (colors, fonts)
- [ ] Google Font imports
- [ ] Hero banner image (`pic/banner.jpg`)
- [ ] Honoree/host photo (`pic/honoree.jpg`)
- [ ] All text content (name, date, time, venue, address, food, reminders)
- [ ] Emoji choices throughout (match the theme/character)
- [ ] RSVP deadline date
- [ ] `SHEET_URL` (new Apps Script deployment per event)
- [ ] Notification email in Apps Script
- [ ] Page `<title>` and footer contact info
- [ ] `object-position` on hero image (adjust % to frame the photo well)

---

## Common Pitfalls

| Problem | Cause | Fix |
|---|---|---|
| "Something went wrong" on RSVP submit | Missing `mode: 'no-cors'` | Add `mode: 'no-cors'` to fetch call |
| Styles not applying | CSS variables using `–` (en-dash) instead of `--` (double hyphen) | Find/replace all `–` with `--` in CSS |
| Git push HTTP 400 | Images too large in history | Compress images; use `git filter-branch` to clean history |
| GitHub Pages not updating | Browser cache | Hard refresh with `Cmd+Shift+R` (Mac) or `Ctrl+Shift+R` (Windows) |
| Hero image cropped wrong | Default `object-position` | Adjust `object-position: center X%` until framing looks right |
| Setup note still visible | Forgot to remove host instructions div | Delete the yellow setup note `<div>` from HTML |

---

## Character Image Processing (Background Removal)

One of the most impactful decorations is floating character images with transparent backgrounds. Here's the full reusable workflow.

### Why This Matters
User-uploaded JPGs always have a solid background color. Removing it lets characters "float" on the page over any background — no white/blue/purple box around them.

### Requirements
```bash
conda run python -c "from PIL import Image; from scipy import ndimage; print('ok')"
# If missing: pip install Pillow scipy
```

### The Multi-Pass Background Removal Script

This is the most reliable approach — it handles both outer background AND enclosed interior pockets (e.g. gaps between legs, inside a scooter wheel, etc.):

```python
from PIL import Image
import numpy as np
from scipy import ndimage
import os

def remove_background(src_path, out_path, threshold=12, max_pocket_size=10000, crop_watermark=True, max_size=400):
    """
    Remove solid-color background from a JPG character image.
    
    - threshold: how close a pixel must be to bg color to be removed (lower = tighter)
      Use 8-12 for characters with colors similar to background
      Use 20-40 for characters with very distinct colors from background
    - max_pocket_size: interior enclosed bg regions smaller than this are also removed
      Increase if interior pockets remain; decrease if character parts disappear
    - crop_watermark: crops bottom 12% to remove stock photo watermarks
    - max_size: thumbnail max dimension for output
    """
    img = Image.open(src_path).convert('RGBA')
    w, h = img.size
    
    if crop_watermark:
        img = img.crop((0, 0, w, int(h * 0.88)))
    
    data = np.array(img)
    h2, w2 = data.shape[:2]
    
    # Auto-detect background color from corners
    corners = [data[0,0], data[0,w2-1], data[h2-1,0], data[h2-1,w2-1]]
    bg_color = np.mean([c[:3] for c in corners], axis=0)
    print(f"Detected BG: R={bg_color[0]:.0f} G={bg_color[1]:.0f} B={bg_color[2]:.0f}")
    
    # Find all pixels close to bg color
    diff = np.abs(data[:,:,:3].astype(float) - bg_color).max(axis=2)
    candidate_bg = diff < threshold
    
    # Label connected regions
    labeled, num = ndimage.label(candidate_bg)
    print(f"Found {num} connected regions")
    
    # Border-connected regions = outer background
    border_mask = np.zeros((h2, w2), dtype=bool)
    border_mask[0,:] = border_mask[h2-1,:] = True
    border_mask[:,0] = border_mask[:,w2-1] = True
    border_labels = set(labeled[border_mask & candidate_bg].tolist()) - {0}
    
    # Remove border bg + small interior pockets
    is_bg = np.zeros((h2, w2), dtype=bool)
    for lbl in range(1, num+1):
        region = labeled == lbl
        if lbl in border_labels or np.sum(region) < max_pocket_size:
            is_bg |= region
    
    data[:,:,3] = np.where(is_bg, 0, 255)
    result = Image.fromarray(data)
    
    # Crop tight to visible bounding box
    alpha = data[:,:,3]
    rows = np.any(alpha > 0, axis=1)
    cols = np.any(alpha > 0, axis=0)
    if rows.any() and cols.any():
        rmin, rmax = np.where(rows)[0][[0,-1]]
        cmin, cmax = np.where(cols)[0][[0,-1]]
        result = result.crop((cmin, rmin, cmax+1, rmax+1))
    
    result.thumbnail((max_size, max_size), Image.LANCZOS)
    result.save(out_path, 'PNG', optimize=True)
    print(f"Saved: {out_path} — {os.path.getsize(out_path)/1024:.1f} KB, size: {result.size}")
    return result

# Example usage:
remove_background('pic/character.jpg', 'pic/character.png', threshold=12, max_pocket_size=10000)
```

### Threshold Guide

| Background Type | Recommended Threshold |
|---|---|
| Solid white | 8 |
| Solid blue/green/purple (uniform) | 12–18 |
| Gradient or noisy background | 20–30 |
| Character colors very similar to bg | 5–8 (tight) |

### Diagnosing Issues

**Character parts disappearing** → threshold too high, lower it (e.g. 40 → 12)

**Background patches remaining** → interior pockets not caught, increase `max_pocket_size` (e.g. 2000 → 10000). To find the right value:
```python
# Run this to see pocket sizes
sizes = {lbl: int(np.sum(labeled==lbl)) for lbl in range(1, num+1) if lbl not in border_labels}
for lbl, sz in sorted(sizes.items(), key=lambda x: -x[1])[:10]:
    ys, xs = np.where(labeled==lbl)
    print(f"label {lbl}: size={sz}, center=({int(xs.mean())},{int(ys.mean())})")
```
Then set `max_pocket_size` just above the largest remaining pocket.

**Image appears invisible on page** → character was tiny in a large canvas. The tight bounding box crop fixes this automatically.

**JPEG compression artifacts** → slight color variation around edges. Lower threshold slightly or accept minor fringing.

---

## Floating Character CSS Pattern

Once you have transparent PNGs, use this CSS to float them on the page without affecting layout:

```css
/* Characters float absolutely within their section */
.section {
  position: relative;
  overflow: visible;
}

.char-wrap {
  position: absolute;
  right: 0;        /* or left: 0 for left side */
  top: 50%;
  transform: translateY(-50%);
  height: 500px;   /* adjust per character */
  width: auto;
  pointer-events: none;
  z-index: 2;
  animation: char-float 2.8s ease-in-out infinite;
}

.char-wrap img {
  height: 100%;
  width: auto;
  display: block;
}

@keyframes char-float {
  0%, 100% { transform: translateY(-50%) rotate(-3deg); }
  50%       { transform: translateY(calc(-50% - 8px)) rotate(3deg); }
}

/* Fine-tune per section */
.char-wrap.section-1 { top: 30%; }
.char-wrap.section-2 { top: 20%; left: 0; right: auto; }

/* Always update mobile too! */
@media (max-width: 600px) {
  .char-wrap { height: 300px; }
  .char-wrap.section-2 { height: 200px; top: 15%; }
}
```

> **Mobile tip:** Always update both desktop AND mobile CSS together. The mobile `@media` query overrides desktop values — if you only change desktop, mobile won't update.

### Two-Column Layout (Character + Content Side by Side)

For sections where you want a character on the left and a content card on the right:

```html
<section class="section">
  <img src="pic/character.png" class="char-abs" alt="Character">
  <div class="section-inner">
    <h2>Section Title</h2>
    <div class="content-card">...</div>
  </div>
</section>
```

```css
.char-abs {
  position: absolute;
  left: 20px;
  top: 50%;
  transform: translateY(-50%);
  height: 220px;
  width: auto;
  pointer-events: none;
  z-index: 2;
  animation: char-float 2.8s ease-in-out infinite;
}
/* Push content card to the right */
.section-inner {
  display: flex;
  justify-content: flex-end;
}
```

---

## Sprinkles / Confetti Background

Match the party theme with scattered decorative dots using CSS `background-image` tiling:

```css
body::before {
  content: '';
  position: fixed;
  inset: 0;
  background-image:
    radial-gradient(circle, #f5e642 3px, transparent 3px),   /* yellow */
    radial-gradient(circle, #f4b8cc 4px, transparent 4px),   /* pink */
    radial-gradient(circle, #a8c8f0 3px, transparent 3px),   /* blue */
    radial-gradient(circle, #a8dbb8 3px, transparent 3px),   /* green */
    radial-gradient(circle, #c9b0e8 4px, transparent 4px),   /* purple */
    radial-gradient(circle, #f5c89a 3px, transparent 3px);   /* orange */
  background-size:
    130px 170px, 160px 200px, 180px 150px,
    150px 190px, 210px 160px, 170px 220px;
  background-position:
    15px 40px, 55px 20px, 120px 150px,
    80px 60px, 190px 30px, 60px 130px;
  opacity: 0.35;
  pointer-events: none;
  z-index: 0;
}
```

Add floating star emojis for extra flair:
```html
<div class="sprinkle-layer" aria-hidden="true">
  <span style="left:8%;top:12%">⭐</span>
  <span style="left:88%;top:8%">✨</span>
  <span style="left:5%;top:35%">🌟</span>
  <!-- add more as needed -->
</div>
```
```css
.sprinkle-layer {
  position: fixed; inset: 0;
  pointer-events: none; z-index: 0; overflow: hidden;
}
.sprinkle-layer span {
  position: absolute; font-size: 14px; opacity: 0.25;
  animation: sprinkle-spin 6s ease-in-out infinite;
}
@keyframes sprinkle-spin {
  0%, 100% { transform: rotate(0deg) scale(1); }
  50% { transform: rotate(20deg) scale(1.15); }
}
```

---

## Theme Color Palette (Peppa Pig / Kids Party)

Derived from the Peppa Pig invitation reference:

| Element | Color | Hex |
|---|---|---|
| Page background | Muted lavender | `#c9b8e8` |
| Hero / Footer bg | Deep purple | `#6B3FA0` |
| Content sections | Warm white | `#F8F4FF` |
| Card / box bg | Soft lavender | `#EDE7F6` |
| Main headline | Soft gold | `#F5C842` |
| Subtext / labels | Slate blue | `#7BA7C7` |
| Body text | Deep purple | `#3D2070` |
| Muted accents | Light purple | `#9B7CC0` |
| Input fields | Very light blue | `#eef6ff` |

### Typography
- **Headlines**: `Baloo 2` weight 800 — playful, rounded, kid-friendly
- **Body / labels**: `Nunito` weight 600 — clean and readable
- **Age / number callout**: `Architects Daughter` — handwritten feel, rotate 3deg for personality

```html
<link href="https://fonts.googleapis.com/css2?family=Baloo+2:wght@400;600;800&family=Nunito:wght@400;600;700&family=Architects+Daughter&display=swap" rel="stylesheet">
```
