# Prompt: Build a Single-File EPUB Creator

## Goal

Build a **single static HTML file** that functions as a full-featured EPUB 3.0 creator. The app must run entirely in the browser with **zero backend, zero install**. The user opens the HTML file, creates their ebook, and downloads a valid `.epub` file.

---

## Core Architecture

- **Single `index.html` file** — all HTML, CSS, and JS inline. No build step.
- **External CDN dependencies only** (loaded via `<script>` / `<link>` tags):
  - `JSZip` — for assembling the EPUB zip container
  - `FileSaver.js` — for triggering the download
  - A lightweight rich-text editor (pick ONE): `Quill`, `TinyMCE (free tier)`, or `Trix`
  - *(Optional)* `marked.js` if you add a markdown input mode
- The app must still work offline after the first load (use CDN with `integrity` hashes, or inline the libs if small enough).

---

## EPUB 3.0 Structure Reference

When the user clicks "Export", generate a `.epub` file (which is a ZIP) with this exact structure:

```
mimetype                          ← must be first file, stored uncompressed, content: "application/epub+zip"
META-INF/
  container.xml                   ← points to content.opf
OEBPS/
  content.opf                     ← package manifest + spine
  toc.xhtml                       ← navigation document (EPUB 3 nav)
  toc.ncx                         ← legacy NCX table of contents (EPUB 2 compat)
  styles.css                      ← shared stylesheet
  chapters/
    chapter-001.xhtml             ← one file per chapter
    chapter-002.xhtml
    ...
  images/
    image-001.png                 ← embedded images (base64 decoded to binary)
    image-002.jpg
    ...
```

### Key EPUB rules to follow:
1. `mimetype` file MUST be the first entry in the ZIP, stored with NO compression (compression level 0).
2. All XHTML files must be valid XHTML5 (self-closing tags, proper namespace `xmlns="http://www.w3.org/1999/xhtml"`).
3. `content.opf` must list every file in `<manifest>` and every chapter in `<spine>` in reading order.
4. Every image embedded in chapters must also appear in the manifest with correct `media-type`.
5. Generate a UUID for the book's unique identifier (`dc:identifier`).

---

## UI / UX Requirements

### Layout
Use a **two-panel layout**:
- **Left sidebar** (≈250px): chapter list, book metadata button, export button
- **Main area**: the rich-text editor for the currently selected chapter

### Book Metadata Modal/Panel
Provide fields for:
- **Title** (required)
- **Author** (required)
- **Language** (default: `en`, dropdown or text input)
- **Description** (optional, textarea)
- **Cover image** (optional, file upload — will become the EPUB cover)
- **Publisher**, **Date**, **Rights/License** (optional)

### Chapter Management
- "Add Chapter" button — creates a new chapter with a default title like "Chapter N"
- Each chapter in the sidebar should show:
  - An **editable title** (inline rename on double-click or edit icon)
  - A **delete button** (with confirmation)
  - **Drag-and-drop reordering** (use native HTML5 drag-and-drop or a lightweight lib)
- Clicking a chapter loads its content into the editor
- **Auto-save** content to the in-memory data model on every edit (no explicit save button needed)

### Rich-Text Editor
The editor must support at minimum:
- **Bold**, **italic**, **underline**, **strikethrough**
- **Headings** (H1–H3)
- **Ordered and unordered lists**
- **Block quotes**
- **Hyperlinks** (insert/edit URL)
- **Image insertion** — via file picker or drag-and-drop onto the editor
  - Convert images to base64 for in-editor display
  - On EPUB export, decode to binary and place in `OEBPS/images/`
  - Replace `<img src="data:...">` with `<img src="../images/image-NNN.ext"/>` in chapter XHTML
- **Code blocks** (optional but nice)
- A **clean HTML output** — strip any editor-specific markup or classes before writing to EPUB XHTML

### Export Button
- Labeled "Download EPUB" or "Export .epub"
- On click:
  1. Validate: at least a title, author, and one chapter with content
  2. Show errors inline if validation fails
  3. Assemble the EPUB zip structure (see above)
  4. Trigger browser download of `{book-title}.epub`
- Show a brief toast/notification on success

### Save / Load Project (Local Persistence)
- **Auto-save** the entire project state to `localStorage` on every change (debounced, e.g., every 2 seconds)
- On page load, detect and offer to **restore** the previous session
- Provide a **"New Project"** button that clears state (with confirmation)
- *(Bonus)* "Export Project" as a `.json` file for backup, and "Import Project" to restore from JSON

---

## Styling / Design

- Clean, modern, minimal UI. Think Notion or Typora aesthetic.
- Light color scheme by default. *(Bonus: dark mode toggle)*
- Use system fonts or a single Google Font (e.g., Inter).
- Responsive enough to work on tablets, but desktop is the primary target.
- The editor area should feel spacious — generous padding, max-width ~700px centered, comfortable line-height.

---

## EPUB Content Templates

### `mimetype`
```
application/epub+zip
```
(No trailing newline.)

### `META-INF/container.xml`
```xml
<?xml version="1.0" encoding="UTF-8"?>
<container version="1.0" xmlns="urn:oasis:names:tc:opendocument:xmlns:container">
  <rootfiles>
    <rootfile full-path="OEBPS/content.opf" media-type="application/oebps-package+xml"/>
  </rootfiles>
</container>
```

### `OEBPS/content.opf` (generate dynamically)
```xml
<?xml version="1.0" encoding="UTF-8"?>
<package xmlns="http://www.idpf.org/2007/opf" version="3.0" unique-identifier="bookid">
  <metadata xmlns:dc="http://purl.org/dc/elements/1.1/">
    <dc:identifier id="bookid">urn:uuid:{GENERATED-UUID}</dc:identifier>
    <dc:title>{TITLE}</dc:title>
    <dc:creator>{AUTHOR}</dc:creator>
    <dc:language>{LANG}</dc:language>
    <meta property="dcterms:modified">{ISO-DATE}</meta>
    <!-- optional: dc:description, dc:publisher, dc:rights, dc:date -->
  </metadata>
  <manifest>
    <item id="nav" href="toc.xhtml" media-type="application/xhtml+xml" properties="nav"/>
    <item id="ncx" href="toc.ncx" media-type="application/x-dtbncx+xml"/>
    <item id="css" href="styles.css" media-type="text/css"/>
    <!-- one <item> per chapter -->
    <!-- one <item> per image -->
    <!-- if cover: <item id="cover-image" href="images/cover.jpg" media-type="image/jpeg" properties="cover-image"/> -->
  </manifest>
  <spine toc="ncx">
    <!-- <itemref idref="chapter-001"/> etc. in reading order -->
  </spine>
</package>
```

### `OEBPS/toc.xhtml` (EPUB 3 nav — generate dynamically)
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml" xmlns:epub="http://www.idpf.org/2007/ops">
<head><title>Table of Contents</title></head>
<body>
  <nav epub:type="toc">
    <h1>Table of Contents</h1>
    <ol>
      <!-- <li><a href="chapters/chapter-001.xhtml">{Chapter Title}</a></li> -->
    </ol>
  </nav>
</body>
</html>
```

### `OEBPS/toc.ncx` (EPUB 2 compat — generate dynamically)
```xml
<?xml version="1.0" encoding="UTF-8"?>
<ncx xmlns="http://www.daisy.org/z3986/2005/ncx/" version="2005-1">
  <head>
    <meta name="dtb:uid" content="urn:uuid:{SAME-UUID}"/>
  </head>
  <docTitle><text>{TITLE}</text></docTitle>
  <navMap>
    <!-- <navPoint id="chapter-001" playOrder="1">
      <navLabel><text>{Chapter Title}</text></navLabel>
      <content src="chapters/chapter-001.xhtml"/>
    </navPoint> -->
  </navMap>
</ncx>
```

### `OEBPS/styles.css`
```css
body {
  font-family: Georgia, "Times New Roman", serif;
  margin: 1em;
  line-height: 1.6;
}
h1, h2, h3 { margin-top: 1.5em; }
img { max-width: 100%; height: auto; }
blockquote { margin-left: 1.5em; font-style: italic; }
code, pre { font-family: monospace; font-size: 0.9em; }
```

### Chapter XHTML template
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml">
<head>
  <title>{Chapter Title}</title>
  <link rel="stylesheet" type="text/css" href="../styles.css"/>
</head>
<body>
  <h1>{Chapter Title}</h1>
  {CHAPTER-HTML-CONTENT}
</body>
</html>
```

---

## Image Handling Details

This is the trickiest part — get it right:

1. **On insert**: User picks an image → read via `FileReader` as base64 data URL → insert into editor as `<img src="data:image/png;base64,...">`.
2. **In memory**: Store each image's binary data (or base64) in the project data model with a generated filename like `image-001.png`.
3. **On export**:
   - Scan each chapter's HTML for `<img src="data:...">` tags.
   - For each, decode the base64 to a `Uint8Array`.
   - Write to `OEBPS/images/{filename}` in the ZIP.
   - Replace the `src` attribute with the relative path `../images/{filename}`.
   - Add the image to `content.opf` manifest with the correct MIME type.

---

## Data Model (In-Memory)

Use a single JavaScript object as the source of truth:

```javascript
const project = {
  metadata: {
    title: "",
    author: "",
    language: "en",
    description: "",
    publisher: "",
    date: "",
    rights: "",
    coverImage: null  // { filename, mimeType, base64 }
  },
  chapters: [
    {
      id: "chapter-001",
      title: "Chapter 1",
      content: "<p>...</p>",  // HTML string from editor
      order: 0
    }
  ],
  images: [
    // { id, filename, mimeType, base64 }
  ]
};
```

---

## Edge Cases & Validation

- **Empty chapters**: Allow export but include them as blank pages
- **No cover image**: Skip cover-related entries in OPF — this is valid
- **Special characters in titles**: Escape for XML (`&`, `<`, `>`, `"`, `'`)
- **Duplicate chapter titles**: Allowed — they have unique IDs anyway
- **Very large images**: Warn if total project size exceeds ~50MB (optional)
- **HTML sanitization**: Strip `<script>`, `onclick`, and other dangerous attributes from editor output before writing to EPUB

---

## Bonus Features (If Feasible)

- **Markdown mode toggle** — switch between rich-text and markdown editing per chapter (use `marked.js` to convert on export)
- **EPUB import** — read an existing `.epub` file and load it into the editor for editing (use JSZip to unpack + DOMParser to parse XHTML)
- **Live preview** — show a read-only rendered preview of the current chapter as it would appear in an ereader
- **Keyboard shortcuts** — Ctrl+S to force save, Ctrl+E to export, Ctrl+N for new chapter
- **Word/character count** per chapter and total
- **Find and replace** across all chapters

---

## Quality Checklist

Before considering the output complete, verify:

- [ ] `mimetype` is first in ZIP, stored uncompressed
- [ ] All XHTML is well-formed (namespace, self-closing tags)
- [ ] `content.opf` manifest includes every chapter and image
- [ ] `spine` order matches the chapter sidebar order
- [ ] `toc.xhtml` and `toc.ncx` both list all chapters
- [ ] Images are properly extracted from base64 and placed in `images/`
- [ ] Image `src` paths in chapter XHTML point to correct relative paths
- [ ] The generated `.epub` passes validation with [EPUBCheck](https://www.w3.org/publishing/epubcheck/) (or at minimum opens correctly in Calibre / Apple Books / Google Play Books)
- [ ] `localStorage` persistence works across page reloads
- [ ] The entire app is a single HTML file under 500KB (excluding CDN loads)
