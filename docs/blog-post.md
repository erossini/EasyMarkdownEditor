---
title: "EasyMarkdownEditor — a drop-in Markdown editor for JavaScript that doesn't fight your CSS"
date: 2026-04-15
tags: [javascript, markdown, codemirror, editor, open-source]
---

# EasyMarkdownEditor — a drop-in Markdown editor for JavaScript that doesn't fight your CSS

Markdown has quietly become the default for every "give the user a rich-ish text field" problem: product descriptions, blog posts, issue trackers, wiki pages, in-app notes. What hasn't become default is the editor itself. If you've ever tried to wire a "nice" Markdown editor into a project, you already know the drill: pick one, fight its CSS, rewrite half the toolbar, give up on the spellchecker, ship.

**[EasyMarkdownEditor](https://github.com/erossini/EasyMarkdownEditor)** (published on npm as [`psc-markdowneditor`](https://www.npmjs.com/package/psc-markdowneditor)) is my attempt to close that gap. It is a hard fork of the well-known `easy-markdown-editor` with the bugs I kept hitting fixed, the parts I kept re-implementing turned into options, and the CSS made to stop colliding with Bootstrap. If you just want a `<textarea>` that becomes a Markdown editor with a toolbar, live preview, autosave, and optional image upload — this is for you.

This post walks through what the component does, how to drop it into a page in thirty seconds, and then through each of the features that I've added or re-worked recently: the toolbar class-prefixing to avoid Bootstrap collisions, the new runtime `resize` handle, image upload, autosave, keyboard shortcuts, and how to build from source if you want to hack on it.

## Table of contents

- [What it is, in one paragraph](#what-it-is-in-one-paragraph)
- [Thirty-second install](#thirty-second-install)
- [The default toolbar — and why the class names have an `mde-` prefix now](#the-default-toolbar--and-why-the-class-names-have-an-mde--prefix-now)
- [Resizable editor](#resizable-editor)
- [Callouts and video embeds](#callouts-and-video-embeds)
- [Sizing, theming, and preview](#sizing-theming-and-preview)
- [Fullscreen mode and z-index conflicts](#fullscreen-mode-and-z-index-conflicts)
- [Image upload](#image-upload)
- [Autosave](#autosave)
- [Events and the editor instance](#events-and-the-editor-instance)
- [Keyboard shortcuts](#keyboard-shortcuts)
- [Building from source](#building-from-source)
- [Where to next](#where-to-next)

## What it is, in one paragraph

EasyMarkdownEditor is a thin wrapper around [CodeMirror 5](https://codemirror.net/5/) that turns a `<textarea>` into a Markdown editor. It ships as a single UMD bundle (`dist/easymde.min.js` + `dist/easymde.min.css`), has zero framework dependencies at runtime, weighs in around ~250 KB minified (CodeMirror included), and exposes a well-typed JavaScript API plus TypeScript definitions. It gives you: syntax highlighting while typing, a toolbar with the usual Bold/Italic/Link/Image/etc. buttons, a live HTML preview (normal and side-by-side), a status bar with cursor/line/word counts, keyboard shortcuts, a native spell checker, autosave to `localStorage`, drag-and-drop image upload, and — new — a runtime resize handle.

There's a Blazor sibling of this component too, [BlazorMarkdownEditor](https://github.com/erossini/BlazorMarkdownEditor), which wraps the same JS library for Blazor Server and Blazor WebAssembly apps. The two repos are maintained in parallel.

## Thirty-second install

The fastest possible integration: a plain HTML page, a CDN link, a `<textarea>`, one line of JavaScript.

```html
<!DOCTYPE html>
<html>
<head>
    <link rel="stylesheet"
          href="https://unpkg.com/psc-markdowneditor/dist/easymde.min.css">
    <script src="https://unpkg.com/psc-markdowneditor/dist/easymde.min.js"></script>
</head>
<body>
    <textarea id="editor"></textarea>
    <script>
        const easyMDE = new EasyMDE({ element: document.getElementById('editor') });
    </script>
</body>
</html>
```

That's it. Reload the page and the `<textarea>` is gone; in its place is a full Markdown editor. Call `easyMDE.value()` to read the Markdown, `easyMDE.value("# new")` to set it, and `easyMDE.toTextArea()` to tear it down and get the original `<textarea>` back.

If you prefer a bundler, `npm install psc-markdowneditor` and `import EasyMDE from 'psc-markdowneditor'` works too.

## The default toolbar — and why the class names have an `mde-` prefix now

If you've used the original library inside a Bootstrap page, you've probably met one of its sharper edges: the toolbar button for **Insert Table** has the class `table`, which Bootstrap also claims for its `.table` utility — so the button suddenly renders as a full-width striped table row. The same collision hits `.image`, `.link`, and a few others.

EasyMarkdownEditor now ships with a sensible default prefix. Every toolbar button's class is prefixed with `mde-`, so you get `mde-bold`, `mde-italic`, `mde-table`, `mde-link`, `mde-image`, `mde-preview`, and so on. Bootstrap's `.table` and friends no longer interfere. If you already style toolbar buttons in your own CSS, update your selectors from `.editor-toolbar button.table` to `.editor-toolbar button.mde-table`.

If you need the old behavior (for example because you have existing CSS you can't touch right now), pass an empty string:

```js
new EasyMDE({
    toolbarButtonClassPrefix: '',   // back to bare classes: .bold, .table, etc.
});
```

Or pick a different prefix:

```js
new EasyMDE({
    toolbarButtonClassPrefix: 'myapp',   // → .myapp-bold, .myapp-table, …
});
```

This was a breaking change for anyone who had custom CSS targeting `.editor-toolbar button.bold` etc. — but in practice fixing it is a one-line search-and-replace, and the default stops silently breaking Bootstrap users.

## Resizable editor

One of the more common feature requests on Markdown editors is: "let the user make it taller if they're writing a long post." EasyMarkdownEditor now supports this out of the box with the `resize` option.

```js
new EasyMDE({
    resize: 'vertical',   // or true, 'horizontal', 'both'
    maxHeight: '300px',   // initial height
});
```

What it does:

- Adds a native CSS drag handle at the corner of the `.EasyMDEContainer` (bottom-right for `vertical`/`both`, right edge for `horizontal`).
- The handle resizes the **entire** container — toolbar at the top, editor in the middle, status bar at the bottom — so they all grow and shrink together. No more "the editor shrank but the toolbar didn't" layout breakage.
- Internally, a `ResizeObserver` watches the container and calls `cm.setSize()` with the newly-computed dimensions so CodeMirror's cursor positions, gutters, and line measurements stay in sync with the dragged size.
- `maxHeight`, when you set it, becomes the **initial** container height instead of a fixed height on the editor itself. That way the user can drag past it.

The option accepts `true` (treated as `'vertical'`, the common case), `'vertical'`, `'horizontal'`, `'both'`, or `false`/omitted to disable. There's a live demo at `example/index_resize.html` in the repo showing all three directions side-by-side.

One caveat: `ResizeObserver` is universally supported in modern browsers but is absent in very old ones. In those, the handle will still drag the container (that's plain CSS), but CodeMirror won't re-measure on the fly. For most apps this doesn't matter; if you need to support ancient browsers, turn `resize` off.

## Callouts and video embeds

Technical writing lives and dies by the little "heads up" boxes — the green tip, the amber warning, the red "don't do this in production." Most Markdown editors punt on them: you get inline HTML, or a custom plugin, or nothing. EasyMarkdownEditor now ships with four callout styles and a video embed block baked in, all expressed as ordinary fenced code blocks so your Markdown stays portable.

Five new fenced-code languages render as rich blocks in the preview:

- ` ```att ` → red **Attention** callout
- ` ```note ` → blue **Note** callout
- ` ```tip ` → green **Tip** callout
- ` ```warn ` → amber **Warning** callout
- ` ```video ` → responsive 16:9 embed; recognises YouTube and Vimeo URLs, otherwise drops the URL into an HTML5 `<video>` tag

Each one has a matching toolbar button (`attention`, `note`, `tip`, `warning`, `video`) that wraps the current selection in the right fence — so the writer can either type the block or click the button.

```js
new EasyMDE({
    element: document.getElementById('editor'),
    toolbar: [
        'bold', 'italic', 'heading', '|',
        'attention', 'note', 'tip', 'warning', 'video', '|',
        'preview', 'side-by-side', 'fullscreen',
    ],
});
```

The authored Markdown looks like this:

````markdown
```note
This ships in the next release.
```

```warn
Don't forget to rotate the API key after deploy.
```

```video
https://www.youtube.com/embed/dQw4w9WgXcQ
```
````

Under the hood this is implemented as a `marked` renderer override rather than a `highlight` hook — important because `highlight` forces marked to wrap the output in `<pre><code>`, which means the editor's grey code-block background would frame every callout. Overriding `renderer.code` lets the callout `<div>` replace the wrapper entirely, so it sits flush in the preview. If you also use hljs for real code highlighting, the custom languages short-circuit before hljs sees them, so nothing gets mangled.

Styling lives in `src/css/alert.css` and `src/css/video.css` and is bundled into `dist/easymde.min.css` automatically. There's a working demo at `example/index_alerts_video.html`.

## Sizing, theming, and preview

Beyond the resize handle, the editor has the usual sizing knobs:

```js
new EasyMDE({
    minHeight: '300px',       // grows from here until maxHeight
    maxHeight: '600px',       // hard cap; editor scrolls past this
    spellChecker: true,       // uses the bundled spell-checker
    nativeSpellcheck: true,   // plus the browser's native spellcheck
    lineNumbers: true,
    tabSize: 4,
    indentWithTabs: false,
    direction: 'ltr',         // 'rtl' for right-to-left languages
});
```

A subtle point about `minHeight` + `maxHeight` that used to bite people: in the original library, setting `maxHeight` silently forced `minHeight` to equal `maxHeight`, which gave you a fixed-height editor no matter what `minHeight` you passed. That's the opposite of the usual "start short, grow up to a cap, then scroll" pattern most rich-text UIs want. That's fixed now — if you set both values explicitly, they're both honored, and the editor uses the `max-height` CSS property so content grows the editor up to the cap before scrolling. If you only set `maxHeight` (no `minHeight`), the historical fixed-height behavior is preserved so existing consumers aren't surprised.

```js
// Editor starts at 150px, grows with the content up to 400px, then scrolls.
new EasyMDE({
    minHeight: '150px',
    maxHeight: '400px',
});
```

The preview is driven by [marked](https://github.com/markedjs/marked) by default, but you can swap in any renderer:

```js
new EasyMDE({
    previewRender: (plainText) => customMarkdownToHTML(plainText),
});
```

Side-by-side preview (`F9` by default) gives you live HTML next to your source as you type — the preview re-renders on every change, debounced so it doesn't fight the keypress.

## Fullscreen mode and z-index conflicts

`F11` toggles fullscreen — the editor fills the viewport with a dedicated toolbar, no distractions, just writing. The CSS gives it a `z-index` of `8` for the editor and `9` for the toolbar, which is fine for most pages but breaks the moment your layout has a sticky header, a toast container, or a modal with its own `z-index` above that. Bootstrap's modals default to `z-index: 1040`, which means a fullscreen editor opened underneath one of them is invisible.

The `fullScreenZIndex` option lets you lift the fullscreen stacking context to whatever your page needs:

```js
new EasyMDE({
    fullScreenZIndex: 1050,   // above Bootstrap modals
});
```

When set, the editor wrapper gets the given value and the toolbar + side-by-side preview get `value + 1` (preserving the 1-level gap the default CSS uses). On exit from fullscreen, the inline z-indexes are cleared so the stylesheet defaults come back — no stale `style` attributes lingering on the DOM. The option accepts a number or a CSS string.

## Image upload

Writing a blog post without being able to paste an image from the clipboard is 2015 energy. The editor ships with drag-and-drop + clipboard paste support:

```js
new EasyMDE({
    uploadImage: true,
    imageUploadEndpoint: '/api/markdown/upload',
    imageAccept: 'image/png, image/jpeg',
    imageMaxSize: 2 * 1024 * 1024,   // 2 MB
});
```

That's the built-in uploader. It POSTs the image as multipart `form-data` to `imageUploadEndpoint` and expects a JSON response of the shape `{ data: { filePath: "..." } }`. If you already have an upload pipeline and just want control, plug in a custom handler:

```js
new EasyMDE({
    uploadImage: true,
    imageUploadFunction: (file, onSuccess, onError) => {
        myUploader.send(file)
            .then((url) => onSuccess(url))
            .catch((err) => onError(err.message));
    },
});
```

The status bar shows progress while an image is uploading, and the editor inserts `![](url)` at the cursor on success.

## Autosave

For long-form writing, the editor can autosave to `localStorage` so a tab crash doesn't eat the user's work:

```js
new EasyMDE({
    autosave: {
        enabled: true,
        uniqueId: 'blog-post-draft',   // required — the localStorage key
        delay: 1000,                   // ms between saves
        submit_delay: 5000,            // ms of idle before writing
        timeFormat: {
            locale: 'en-US',
            format: { year: 'numeric', month: 'long', day: '2-digit', hour: '2-digit', minute: '2-digit' },
        },
        text: 'Autosaved: ',
    },
});
```

The status bar shows the last-saved timestamp. Call `easyMDE.clearAutosavedValue()` when the user hits "Publish" so stale drafts don't linger.

## Events and the editor instance

EasyMarkdownEditor exposes CodeMirror's full event surface through `.codemirror`:

```js
const easyMDE = new EasyMDE({ element: textarea });

easyMDE.codemirror.on('change', () => {
    console.log(easyMDE.value());
});

easyMDE.codemirror.on('cursorActivity', (cm) => {
    const pos = cm.getCursor();
    console.log(`cursor at line ${pos.line + 1}, column ${pos.ch + 1}`);
});
```

Useful instance methods:

- `easyMDE.value()` / `easyMDE.value('# Hello')` — get or set the Markdown content.
- `easyMDE.isPreviewActive()` / `easyMDE.isSideBySideActive()` / `easyMDE.isFullscreenActive()` — layout queries.
- `easyMDE.togglePreview()` / `easyMDE.toggleSideBySide()` / `easyMDE.toggleFullScreen()` — programmatic toggles.
- `easyMDE.toTextArea()` — destroy the editor and restore the original `<textarea>`.

## Keyboard shortcuts

All the defaults you'd expect. A subset:

| Shortcut             | Action                 |
| -------------------- | ---------------------- |
| `Cmd/Ctrl + B`       | Toggle bold            |
| `Cmd/Ctrl + I`       | Toggle italic          |
| `Cmd/Ctrl + K`       | Create link            |
| `Cmd/Ctrl + H`       | Decrease heading       |
| `Shift + Cmd/Ctrl + H` | Increase heading     |
| `Cmd/Ctrl + '`       | Blockquote             |
| `Cmd/Ctrl + L`       | Unordered list         |
| `Cmd/Ctrl + Alt + L` | Ordered list           |
| `Cmd/Ctrl + Alt + C` | Code block             |
| `Cmd/Ctrl + P`       | Toggle preview         |
| `F9`                 | Toggle side-by-side    |
| `F11`                | Toggle fullscreen      |

They're remappable per action via the `shortcuts` option.

## Building from source

If you want to hack on the component — add a toolbar button, tweak the CSS, fix a bug — the build is deliberately simple: Gulp + Browserify, no TypeScript step on the source, ES5 output so it runs without transpilation.

```bash
npm install           # once
npm run prepare       # full build → dist/easymde.min.{js,css}
npx gulp watch        # rebuild on every src change
npm run lint          # ESLint over src/js
npm test              # lint + type-check + Cypress end-to-end tests
```

There's only one JavaScript file in `src/js/easymde.js` — everything (action implementations, toolbar registry, preview logic, autosave, image upload) lives there. The `bindings` map at the top of the file is the canonical list of actions that can be bound to toolbar buttons or keyboard shortcuts; if you're adding a feature, that's where you start. Tests are Cypress end-to-end specs under `cypress/e2e/`.

The TypeScript type definitions live at `types/easymde.d.ts` and are hand-maintained; `npm run test:types` type-checks them so they don't drift.

## Where to next

- **[GitHub repo](https://github.com/erossini/EasyMarkdownEditor)** — source, issues, pull requests.
- **[npm package](https://www.npmjs.com/package/psc-markdowneditor)** — `npm install psc-markdowneditor`.
- **[Blazor wrapper](https://github.com/erossini/BlazorMarkdownEditor)** — if you're in .NET land.
- **[Live demo](https://markdown.puresourcecode.com/)** — drop-in playground.

If you hit a bug or a collision I haven't thought of, open an issue — PRs welcome. Defaults are deliberately "works on Bootstrap pages out of the box" and "doesn't surprise the user" — if you find a case where either of those breaks, I want to hear about it.
