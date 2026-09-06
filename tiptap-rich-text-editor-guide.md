# Tiptap Rich-Text Editor Guide

A complete, copy-paste-able guide for building the same feature-rich WYSIWYG rich-text editor (as used for blog posts) in any Next.js (App Router) project. Tiptap v3 + React + Tailwind CSS v4.

## Table of Contents

1. [Overview & Features](#1-overview--features)
2. [Install Dependencies](#2-install-dependencies)
3. [File Reference Map](#3-file-reference-map)
4. [The Editor Hook](#4-the-editor-hook)
5. [The Toolbar](#5-the-toolbar)
6. [Link Prompt](#6-link-prompt)
7. [Host Form Wiring](#7-host-form-wiring)
8. [Image Upload Integration](#8-image-upload-integration)
9. [CSS & Styling](#9-css--styling)
10. [Public Rendering](#10-public-rendering)
11. [Save / Load Flow](#11-save--load-flow)
12. [Key Gotchas](#12-key-gotchas)
13. [Flow Diagram](#13-flow-diagram)

---

## 1. Overview & Features

A Tiptap-based rich-text editor with a full toolbar. Content is produced as **HTML** (`editor.getHTML()`) and stored as a string — which makes saving, reloading into the editor, and rendering on public pages trivial.

**Editing features:**

- Headings **H1–H6** + Paragraph (dropdown)
- Font size (Normal, 12–72 px)
- Bold, Italic, Underline, Strikethrough
- Highlight (multicolor)
- Text color (arbitrary via color picker)
- Bullet list, Ordered list, Blockquote, Code block
- Text alignment: left / center / right / justify
- Inline images (upload → URL), resizable
- Links (via a small URL prompt), auto-link + link-on-paste
- Tables — insert, add/delete row/column, merge/split cells, cell background color, delete table
- Placeholder text when empty
- **Paste cleanup** for Word/Google Docs markup (`mso-*`, `Mso*` classes, bogus `<span>` styling)
- **Paste & drag-drop image upload** (image files intercepted before text insertion)
- Live word / character count
- Read-only (`view`) mode for previewing saved content
- Live preview pane using the same CSS as the public pages

**Behavioral defaults:**

- `StarterKit` with headings enabled up to level 6
- `immediatelyRender: false` (required for Next.js SSR — avoids hydration mismatch)
- Images never stored as base64 (`allowBase64: false`) — always uploaded to the server and referenced by URL
- Images resizable in-editor (`resize: { enabled: true, minWidth: 100, minHeight: 100 }`)
- Links default to `https://` when no protocol is given

---

## 2. Install Dependencies

```bash
npm install @tiptap/react @tiptap/starter-kit \
  @tiptap/extension-text-style \
  @tiptap/extension-font-size \
  @tiptap/extension-color \
  @tiptap/extension-highlight \
  @tiptap/extension-underline \
  @tiptap/extension-link \
  @tiptap/extension-image \
  @tiptap/extension-placeholder \
  @tiptap/extension-text-align \
  @tiptap/extension-table \
  @tiptap/extension-table-row \
  @tiptap/extension-table-cell \
  @tiptap/extension-table-header \
  react-icons
```

> Target Tiptap **v3** (`^3.x`). `react-icons` is used for toolbar icons (`react-icons/hi`, `react-icons/hi2`); the rest of the icons are inline `<svg>` elements.
>
> **Version pin warning:** `@tiptap/extension-font-size` is currently at `3.0.0-next.3` (a pre-release). If your lockfile resolves a stable release instead, verify the `setFontSize` / `unsetFontSize` commands still exist before relying on the font-size dropdown.

---

## 3. File Reference Map

| File | Responsibility |
|------|----------------|
| `src/hooks/useBlogEditor.tsx` | Tiptap editor state, extension config, paste/drop handlers, Word HTML cleaner |
| `src/components/blogs/BlogEditorToolbar.tsx` | Full toolbar UI (all formatting + table sub-toolbar) |
| `src/components/blogs/BlogLinkPrompt.tsx` | Small URL entry bar for inserting links |
| `src/components/blogs/BlogComposerForm.tsx` | Host form: `<EditorContent>`, image upload, link handling, validation, save |
| `src/components/blogs/BlogLivePreview.tsx` | Live preview pane (renders `editor.getHTML()` with the shared `.blog-prose` CSS) |
| `src/lib/upload.ts` | `uploadImage()` helper — uploads a `File`, returns a URL |
| `src/app/api/upload/route.ts` | Upload endpoint (`multipart/form-data`) |
| `src/app/api/blog/images/[filename]/route.ts` | Serves uploaded images (see the image-upload guide) |
| `src/utils/blog-image.ts` | `normalizeBlogContentHtml()` — normalizes inline image URLs |
| `src/app/globals.css` | `.blog-prose` + `.tiptap` editor/preview/rendered-content CSS |

---

## 4. The Editor Hook

`src/hooks/useBlogEditor.tsx`

This single hook owns the whole editor instance. It configures every extension, wires paste/drop behavior, and reports content changes up to the host form via the `onUpdate` callback.

```tsx
import { useEditor } from "@tiptap/react";
import StarterKit from "@tiptap/starter-kit";
import { TextStyle } from "@tiptap/extension-text-style";
import { FontSize } from "@tiptap/extension-font-size";
import { Color } from "@tiptap/extension-color";
import Underline from "@tiptap/extension-underline";
import Link from "@tiptap/extension-link";
import ImageExtension from "@tiptap/extension-image";
import Placeholder from "@tiptap/extension-placeholder";
import TextAlign from "@tiptap/extension-text-align";
import { Highlight } from "@tiptap/extension-highlight";
import { Table } from "@tiptap/extension-table";
import TableRow from "@tiptap/extension-table-row";
import TableCell from "@tiptap/extension-table-cell";
import TableHeader from "@tiptap/extension-table-header";

function cleanPastedHTML(html: string): string {
  const div = document.createElement("div");
  div.innerHTML = html;

  div.querySelectorAll("o\\:p, <!--[if gte mso 9]-->").forEach((el) => el.remove());
  div.querySelectorAll('[class^="Mso"]').forEach((el) => {
    el.removeAttribute("class");
  });

  div.querySelectorAll("*").forEach((el) => {
    const style = el.getAttribute("style") || "";
    const cleaned = style
      .split(";")
      .filter((s) => !s.trim().toLowerCase().startsWith("mso-"))
      .join(";");
    if (cleaned.trim()) {
      el.setAttribute("style", cleaned.trim());
    } else {
      el.removeAttribute("style");
    }
  });

  div.querySelectorAll("b[style]").forEach((el) => {
    const style = el.getAttribute("style") || "";
    if (/font-weight\s*:\s*normal/i.test(style)) {
      const parent = el.parentNode;
      while (el.firstChild) {
        parent?.insertBefore(el.firstChild, el);
      }
      parent?.removeChild(el);
    }
  });

  // Normalize <span style="font-weight:bold/700/bolder"> → <strong>
  div.querySelectorAll("span[style]").forEach((el) => {
    const style = el.getAttribute("style") || "";
    if (/font-weight\s*:\s*(bold|700|bolder)/i.test(style)) {
      const strong = document.createElement("strong");
      strong.innerHTML = el.innerHTML;
      const cleanedStyle = style
        .split(";")
        .filter((s) => !/font-weight/i.test(s))
        .join(";");
      if (cleanedStyle.trim()) {
        strong.setAttribute("style", cleanedStyle.trim());
      }
      el.parentNode?.replaceChild(strong, el);
    }
  });

  // Normalize <span style="font-style:italic"> → <em>
  div.querySelectorAll("span[style]").forEach((el) => {
    const style = el.getAttribute("style") || "";
    if (/font-style\s*:\s*italic/i.test(style)) {
      const em = document.createElement("em");
      em.innerHTML = el.innerHTML;
      const cleanedStyle = style
        .split(";")
        .filter((s) => !/font-style/i.test(s))
        .join(";");
      if (cleanedStyle.trim()) {
        em.setAttribute("style", cleanedStyle.trim());
      }
      el.parentNode?.replaceChild(em, el);
    }
  });

  // Strip font-family from spans (let CSS control font)
  div.querySelectorAll("span[style]").forEach((el) => {
    const style = el.getAttribute("style") || "";
    const cleaned = style
      .split(";")
      .filter((s) => !/font-family/i.test(s))
      .join(";");
    if (cleaned.trim()) {
      el.setAttribute("style", cleaned.trim());
    } else {
      el.removeAttribute("style");
    }
  });

  return div.innerHTML;
}

export const useBlogEditor = (
  initialContent: string,
  isReadOnly: boolean,
  onUpdate: (html: string) => void,
  insertImages: (files: File[]) => void
) => {
  return useEditor({
    immediatelyRender: false,
    editable: !isReadOnly,
    extensions: [
      StarterKit.configure({
        heading: { levels: [1, 2, 3, 4, 5, 6] },
      }),
      TextStyle,
      FontSize,
      Color.configure({
        types: ["textStyle"],
      }),
      Highlight.configure({
        multicolor: true,
      }),
      Underline,
      Link.configure({
        autolink: true,
        defaultProtocol: "https",
        linkOnPaste: true,
        openOnClick: false,
      }),
      ImageExtension.configure({
        allowBase64: false,
        resize: {
          enabled: true,
          minWidth: 100,
          minHeight: 100,
          alwaysPreserveAspectRatio: true,
        },
      }),
      Placeholder.configure({
        placeholder: isReadOnly
          ? "Content preview"
          : "Start drafting your blog content...",
      }),
      TextAlign.configure({
        types: ["heading", "paragraph"],
      }),
      Table.configure({
        resizable: true,
        HTMLAttributes: {
          class: "blog-table",
        },
      }),
      TableRow,
      TableHeader,
      TableCell,
    ],
    content: initialContent,
    editorProps: {
      attributes: {
        class:
          "tiptap blog-prose min-h-[22rem] rounded-[24px] px-5 py-4 text-sm leading-7 text-gray-800 outline-none focus:outline-none sm:min-h-[28rem] sm:px-6 sm:py-5",
      },
      handlePaste: (view, event) => {
        if (isReadOnly) return false;
        const files = Array.from(event.clipboardData?.files ?? []).filter((file) =>
          file.type.startsWith("image/")
        );
        if (files.length > 0) {
          insertImages(files);
          return true;
        }

        // Clean pasted HTML from Word/Google Docs
        const clipboardData = event.clipboardData;
        const html = clipboardData?.getData("text/html");
        if (html) {
          const cleaned = cleanPastedHTML(html);
          // eslint-disable-next-line @typescript-eslint/no-explicit-any
          const editor = (view as any).editor;
          if (editor) {
            editor.chain().focus().insertContent(cleaned).run();
            return true;
          }
        }
        return false;
      },
      handleDrop: (_, event) => {
        if (isReadOnly) return false;
        const files = Array.from(event.dataTransfer?.files ?? []).filter((file) =>
          file.type.startsWith("image/")
        );
        if (files.length === 0) return false;
        insertImages(files);
        return true;
      },
    },
    onCreate: ({ editor }) => onUpdate(editor.getHTML()),
    onUpdate: ({ editor }) => onUpdate(editor.getHTML()),
  });
};
```

**How to read it:**

- **`immediatelyRender: false`** — prevents Tiptap from rendering during server-side render; without it, Next.js throws hydration mismatches. Keep this.
- **`cleanPastedHTML()`** — runs before text paste is inserted. It strips Microsoft Office garbage (`mso-*` styles, `Mso*` classes, conditional comments), unwraps pointless `<b>` tags, and normalizes inline-styled bold/italic into real `<strong>`/`<em>`.
- **`handlePaste` / `handleDrop`** — intercept image files first (upload via `insertImages`), *then* clean-and-insert text HTML. Returning `true` tells Tiptap "handled, stop default behavior".
- **`onCreate`/`onUpdate`** — push `editor.getHTML()` back to the host so the host always holds the current content string.
- **The `class` in `editorProps.attributes`** is what gives the editable area its surface styling (`tiptap blog-prose ...`). The `tiptap` + `blog-prose` classes are styled in `globals.css` (section 9).

---

## 5. The Toolbar

`src/components/blogs/BlogEditorToolbar.tsx`

Full toolbar (verbatim). Renders nothing in read-only mode. Props:

```ts
interface BlogEditorToolbarProps {
  editor: Editor | null
  isReadOnly: boolean
  inlineImageInputRef: RefObject<HTMLInputElement | null>
  onLinkPromptClick: () => void            // open the link prompt
  onInlineImageUpload: (event: ChangeEvent<HTMLInputElement>) => void
  inlineUploading: boolean
}
```

```tsx
"use client"

import { useState, useCallback } from "react"
import { BlogEditorToolbarProps } from "@/types/blog"
import { HiOutlineCode, HiOutlinePhotograph } from "react-icons/hi"
import {
  HiOutlineBold,
  HiOutlineItalic,
  HiOutlineLink,
  HiOutlineListBullet,
  HiOutlineMinusSmall,
} from "react-icons/hi2"

const FONT_SIZES = [
  { label: "Normal", value: "" },
  { label: "12", value: "12px" },
  { label: "14", value: "14px" },
  { label: "16", value: "16px" },
  { label: "18", value: "18px" },
  { label: "20", value: "20px" },
  { label: "24", value: "24px" },
  { label: "28", value: "28px" },
  { label: "32", value: "32px" },
  { label: "36", value: "36px" },
  { label: "48", value: "48px" },
  { label: "64", value: "64px" },
  { label: "72", value: "72px" },
]

function getCurrentHeadingType(editor: BlogEditorToolbarProps["editor"]) {
  if (!editor) return "paragraph"
  for (let level = 1; level <= 6; level++) {
    if (editor.isActive("heading", { level })) return `h${level}`
  }
  return "paragraph"
}

function getCurrentFontSize(editor: BlogEditorToolbarProps["editor"]): string {
  if (!editor) return ""
  try {
    const marks = editor.state.selection.$head.marks()
    const textStyle = marks.find((m) => m.type.name === "textStyle")
    return textStyle?.attrs.fontSize || ""
  } catch {
    return ""
  }
}

export default function BlogEditorToolbar({
  editor,
  isReadOnly,
  inlineImageInputRef,
  onLinkPromptClick,
  onInlineImageUpload,
  inlineUploading,
}: BlogEditorToolbarProps) {
  const [cellBgColor, setCellBgColor] = useState("#ffffff")

  const handleSetCellBg = useCallback(
    (color: string) => {
      if (!editor) return
      setCellBgColor(color)
      editor.chain().focus().setCellAttribute("backgroundColor", color).run()
    },
    [editor]
  )

  const handleClearCellBg = useCallback(() => {
    if (!editor) return
    setCellBgColor("#ffffff")
    editor.chain().focus().setCellAttribute("backgroundColor", null).run()
  }, [editor])

  if (isReadOnly) return null

  const currentHeading = getCurrentHeadingType(editor)
  const currentFontSize = getCurrentFontSize(editor)

  const currentColor = (() => {
    if (!editor) return "#000000"
    try {
      const marks = editor.state.selection.$head.marks()
      const ts = marks.find((m) => m.type.name === "textStyle")
      return (ts?.attrs.color as string) || "#000000"
    } catch {
      return "#000000"
    }
  })()

  const marks = editor ? editor.state.selection.$head.marks() : []
  const hasBold = marks.some((m) => m.type.name === "bold")
  const hasItalic = marks.some((m) => m.type.name === "italic")
  const hasUnderline = marks.some((m) => m.type.name === "underline")
  const hasStrike = marks.some((m) => m.type.name === "strike")
  const hasHighlight = editor?.isActive("highlight") || false

  function toggleMark(name: string) {
    if (!editor) return
    const active = editor.state.selection.$head.marks().some((m) => m.type.name === name)
    if (active) {
      editor.chain().focus().unsetMark(name).run()
    } else {
      editor.chain().focus().setMark(name).run()
    }
  }

  function setFontSize(size: string) {
    if (!editor) return
    if (!size) {
      editor.chain().focus().unsetFontSize().run()
    } else {
      editor.chain().focus().setFontSize(size).run()
    }
  }

  return (
    <>
    <div className="flex flex-wrap items-center gap-2 border-b border-gray-100 px-4 py-3">
      {/* Heading select */}
      <select
        value={currentHeading}
        onChange={(e) => {
          const val = e.target.value
          if (val === "paragraph") {
            editor?.chain().focus().setParagraph().run()
          } else {
            const level = Number(val.replace("h", "")) as 1 | 2 | 3 | 4 | 5 | 6
            editor?.chain().focus().toggleHeading({ level }).run()
          }
        }}
        className="inline-flex cursor-pointer items-center gap-2 rounded-xl border border-gray-200 px-3 py-2 text-xs font-semibold text-gray-600 outline-none transition-colors hover:border-primary hover:text-primary"
      >
        <option value="paragraph">Paragraph</option>
        <option value="h1">H1</option>
        <option value="h2">H2</option>
        <option value="h3">H3</option>
        <option value="h4">H4</option>
        <option value="h5">H5</option>
        <option value="h6">H6</option>
      </select>

      {/* Font size dropdown */}
      <select
        value={currentFontSize}
        onChange={(e) => setFontSize(e.target.value)}
        className="inline-flex cursor-pointer items-center gap-2 rounded-xl border border-gray-200 px-3 py-2 text-xs font-semibold text-gray-600 outline-none transition-colors hover:border-primary hover:text-primary"
        title="Font size"
      >
        {FONT_SIZES.map((size) => (
          <option key={size.value} value={size.value}>
            {size.label}
          </option>
        ))}
      </select>

      <div className="h-5 w-px bg-gray-200" />

      {/* Bold */}
      <button
        type="button"
        onClick={() => toggleMark("bold")}
        className={`inline-flex items-center gap-2 rounded-xl border px-3 py-2 text-xs font-semibold transition-colors ${
          hasBold
            ? "border-primary bg-primary text-white"
            : "border-gray-200 text-gray-600 hover:border-primary hover:text-primary"
        }`}
      >
        <HiOutlineBold className="h-4 w-4" />
        Bold
      </button>
      {/* Italic */}
      <button
        type="button"
        onClick={() => toggleMark("italic")}
        className={`inline-flex items-center gap-2 rounded-xl border px-3 py-2 text-xs font-semibold transition-colors ${
          hasItalic
            ? "border-primary bg-primary text-white"
            : "border-gray-200 text-gray-600 hover:border-primary hover:text-primary"
        }`}
      >
        <HiOutlineItalic className="h-4 w-4" />
        Italic
      </button>
      {/* Underline */}
      <button
        type="button"
        onClick={() => toggleMark("underline")}
        className={`inline-flex items-center gap-2 rounded-xl border px-3 py-2 text-xs font-semibold transition-colors ${
          hasUnderline
            ? "border-primary bg-primary text-white"
            : "border-gray-200 text-gray-600 hover:border-primary hover:text-primary"
        }`}
      >
        <svg className="h-4 w-4" viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
          <path d="M3 14h10M3 3v5a5 5 0 0 0 10 0V3" />
        </svg>
        Underline
      </button>
      {/* Strikethrough */}
      <button
        type="button"
        onClick={() => toggleMark("strike")}
        className={`inline-flex items-center gap-2 rounded-xl border px-3 py-2 text-xs font-semibold transition-colors ${
          hasStrike
            ? "border-primary bg-primary text-white"
            : "border-gray-200 text-gray-600 hover:border-primary hover:text-primary"
        }`}
      >
        <svg className="h-4 w-4" viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
          <path d="M3 8h10M5.5 4h5c1.1 0 2 .9 2 2s-.9 2-2 2h-5c-1.1 0-2 .9-2 2s.9 2 2 2h5" />
        </svg>
        Strike
      </button>
      {/* Highlight */}
      <button
        type="button"
        onClick={() => editor?.chain().focus().toggleHighlight().run()}
        className={`inline-flex items-center gap-2 rounded-xl border px-3 py-2 text-xs font-semibold transition-colors ${
          hasHighlight
            ? "border-primary bg-primary text-white"
            : "border-gray-200 text-gray-600 hover:border-primary hover:text-primary"
        }`}
      >
        <svg className="h-4 w-4" viewBox="0 0 16 16" fill="currentColor">
          <rect x="1" y="10" width="14" height="4" rx="1" opacity="0.3" />
          <path d="M6 2l4 0 1 6H5z" />
        </svg>
        Highlight
      </button>

      <div className="h-5 w-px bg-gray-200" />

      {/* Text color */}
      <div className="flex items-center gap-1">
        <input
          type="color"
          value={currentColor}
          onChange={(e) => editor?.chain().focus().setColor(e.target.value).run()}
          className="h-9 w-9 cursor-pointer rounded-xl border border-gray-200 p-1"
          title="Text color"
        />
        {currentColor !== "#000000" && (
          <button
            type="button"
            onClick={() => editor?.chain().focus().unsetColor().run()}
            className="inline-flex items-center gap-1 rounded-xl border border-gray-200 px-2 py-2 text-xs font-semibold text-gray-600 transition-colors hover:border-primary hover:text-primary"
            title="Clear color"
          >
            ✕
          </button>
        )}
      </div>

      <div className="h-5 w-px bg-gray-200" />

      {/* Bullet list */}
      <button
        type="button"
        onClick={() => editor?.chain().focus().toggleBulletList().run()}
        className={`inline-flex items-center gap-2 rounded-xl border px-3 py-2 text-xs font-semibold transition-colors ${
          editor?.isActive("bulletList")
            ? "border-primary bg-primary text-white"
            : "border-gray-200 text-gray-600 hover:border-primary hover:text-primary"
        }`}
      >
        <HiOutlineListBullet className="h-4 w-4" />
        List
      </button>
      {/* Ordered list */}
      <button
        type="button"
        onClick={() => editor?.chain().focus().toggleOrderedList().run()}
        className={`inline-flex items-center gap-2 rounded-xl border px-3 py-2 text-xs font-semibold transition-colors ${
          editor?.isActive("orderedList")
            ? "border-primary bg-primary text-white"
            : "border-gray-200 text-gray-600 hover:border-primary hover:text-primary"
        }`}
      >
        <svg className="h-4 w-4" viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round">
          <path d="M4 4h8M4 8h8M4 12h8" />
          <text x="1" y="5.5" fontSize="4" fill="currentColor" stroke="none" fontFamily="sans-serif">1</text>
          <text x="1" y="9.5" fontSize="4" fill="currentColor" stroke="none" fontFamily="sans-serif">2</text>
          <text x="1" y="13.5" fontSize="4" fill="currentColor" stroke="none" fontFamily="sans-serif">3</text>
        </svg>
        Ordered
      </button>
      {/* Blockquote */}
      <button
        type="button"
        onClick={() => editor?.chain().focus().toggleBlockquote().run()}
        className={`inline-flex items-center gap-2 rounded-xl border px-3 py-2 text-xs font-semibold transition-colors ${
          editor?.isActive("blockquote")
            ? "border-primary bg-primary text-white"
            : "border-gray-200 text-gray-600 hover:border-primary hover:text-primary"
        }`}
      >
        <HiOutlineMinusSmall className="h-4 w-4" />
        Quote
      </button>
      {/* Code block */}
      <button
        type="button"
        onClick={() => editor?.chain().focus().toggleCodeBlock().run()}
        className={`inline-flex items-center gap-2 rounded-xl border px-3 py-2 text-xs font-semibold transition-colors ${
          editor?.isActive("codeBlock")
            ? "border-primary bg-primary text-white"
            : "border-gray-200 text-gray-600 hover:border-primary hover:text-primary"
        }`}
      >
        <HiOutlineCode className="h-4 w-4" />
        Code
      </button>

      <div className="h-5 w-px bg-gray-200" />

      {/* Text alignment */}
      <button
        type="button"
        onClick={() => editor?.chain().focus().setTextAlign("left").run()}
        className={`inline-flex items-center rounded-xl border px-2.5 py-2 text-xs font-semibold transition-colors ${
          editor?.isActive({ textAlign: "left" })
            ? "border-primary bg-primary text-white"
            : "border-gray-200 text-gray-600 hover:border-primary hover:text-primary"
        }`}
        title="Align left"
      >
        <svg className="h-4 w-4" viewBox="0 0 16 16" fill="currentColor">
          <rect x="1" y="2" width="14" height="1.5" rx="0.5" />
          <rect x="1" y="5.5" width="10" height="1.5" rx="0.5" />
          <rect x="1" y="9" width="14" height="1.5" rx="0.5" />
          <rect x="1" y="12.5" width="8" height="1.5" rx="0.5" />
        </svg>
      </button>
      <button
        type="button"
        onClick={() => editor?.chain().focus().setTextAlign("center").run()}
        className={`inline-flex items-center rounded-xl border px-2.5 py-2 text-xs font-semibold transition-colors ${
          editor?.isActive({ textAlign: "center" })
            ? "border-primary bg-primary text-white"
            : "border-gray-200 text-gray-600 hover:border-primary hover:text-primary"
        }`}
        title="Align center"
      >
        <svg className="h-4 w-4" viewBox="0 0 16 16" fill="currentColor">
          <rect x="1" y="2" width="14" height="1.5" rx="0.5" />
          <rect x="3" y="5.5" width="10" height="1.5" rx="0.5" />
          <rect x="1" y="9" width="14" height="1.5" rx="0.5" />
          <rect x="4" y="12.5" width="8" height="1.5" rx="0.5" />
        </svg>
      </button>
      <button
        type="button"
        onClick={() => editor?.chain().focus().setTextAlign("right").run()}
        className={`inline-flex items-center rounded-xl border px-2.5 py-2 text-xs font-semibold transition-colors ${
          editor?.isActive({ textAlign: "right" })
            ? "border-primary bg-primary text-white"
            : "border-gray-200 text-gray-600 hover:border-primary hover:text-primary"
        }`}
        title="Align right"
      >
        <svg className="h-4 w-4" viewBox="0 0 16 16" fill="currentColor">
          <rect x="1" y="2" width="14" height="1.5" rx="0.5" />
          <rect x="5" y="5.5" width="10" height="1.5" rx="0.5" />
          <rect x="1" y="9" width="14" height="1.5" rx="0.5" />
          <rect x="6" y="12.5" width="8" height="1.5" rx="0.5" />
        </svg>
      </button>
      <button
        type="button"
        onClick={() => editor?.chain().focus().setTextAlign("justify").run()}
        className={`inline-flex items-center rounded-xl border px-2.5 py-2 text-xs font-semibold transition-colors ${
          editor?.isActive({ textAlign: "justify" })
            ? "border-primary bg-primary text-white"
            : "border-gray-200 text-gray-600 hover:border-primary hover:text-primary"
        }`}
        title="Justify"
      >
        <svg className="h-4 w-4" viewBox="0 0 16 16" fill="currentColor">
          <rect x="1" y="2" width="14" height="1.5" rx="0.5" />
          <rect x="1" y="5.5" width="14" height="1.5" rx="0.5" />
          <rect x="1" y="9" width="14" height="1.5" rx="0.5" />
          <rect x="1" y="12.5" width="14" height="1.5" rx="0.5" />
        </svg>
      </button>

      <div className="h-5 w-px bg-gray-200" />

      {/* Insert image */}
      <button
        type="button"
        onClick={() => inlineImageInputRef.current?.click()}
        className="inline-flex items-center gap-2 rounded-xl border border-gray-200 px-3 py-2 text-xs font-semibold text-gray-600 transition-colors hover:border-primary hover:text-primary"
      >
        <HiOutlinePhotograph className="h-4 w-4" />
        Image
      </button>
      {/* Link */}
      <button
        type="button"
        onClick={onLinkPromptClick}
        className={`inline-flex items-center gap-2 rounded-xl border px-3 py-2 text-xs font-semibold transition-colors ${
          editor?.isActive("link")
            ? "border-primary bg-primary text-white"
            : "border-gray-200 text-gray-600 hover:border-primary hover:text-primary"
        }`}
      >
        <HiOutlineLink className="h-4 w-4" />
        Link
      </button>

      <div className="h-5 w-px bg-gray-200" />

      {/* Insert table */}
      <button
        type="button"
        onClick={() => editor?.chain().focus().insertTable({ rows: 3, cols: 3, withHeaderRow: true }).run()}
        className={`inline-flex items-center gap-2 rounded-xl border px-3 py-2 text-xs font-semibold transition-colors ${
          editor?.isActive("table")
            ? "border-primary bg-primary text-white"
            : "border-gray-200 text-gray-600 hover:border-primary hover:text-primary"
        }`}
      >
        <svg className="h-4 w-4" viewBox="0 0 16 16" fill="currentColor">
          <rect x="1" y="1" width="14" height="14" rx="2" fill="none" stroke="currentColor" strokeWidth="1.2" />
          <line x1="5.5" y1="1" x2="5.5" y2="15" stroke="currentColor" strokeWidth="1" />
          <line x1="10.5" y1="1" x2="10.5" y2="15" stroke="currentColor" strokeWidth="1" />
          <line x1="1" y1="5.5" x2="15" y2="5.5" stroke="currentColor" strokeWidth="1" />
          <line x1="1" y1="10.5" x2="15" y2="10.5" stroke="currentColor" strokeWidth="1" />
        </svg>
        Table
      </button>

      <input
        ref={inlineImageInputRef}
        type="file"
        accept="image/*"
        multiple
        className="hidden"
        onChange={onInlineImageUpload}
        disabled={inlineUploading}
      />
    </div>

    {/* Table sub-toolbar — shown when cursor is inside a table */}
    {editor?.isActive("table") && (
      <div className="flex flex-wrap items-center gap-2 border-b border-gray-100 bg-gray-50/80 px-4 py-2">
        <span className="text-[10px] font-bold uppercase tracking-wider text-gray-400">Table</span>

        <div className="h-4 w-px bg-gray-200" />

        {/* Row operations */}
        <button type="button" onClick={() => editor.chain().focus().addRowBefore().run()} className="table-action-btn" title="Add row above">
          <svg className="h-3.5 w-3.5" viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5"><rect x="1" y="4" width="14" height="8" rx="1" /><path d="M8 2v4M5 4l3-2 3 2" /></svg>
          Row↑
        </button>
        <button type="button" onClick={() => editor.chain().focus().addRowAfter().run()} className="table-action-btn" title="Add row below">
          <svg className="h-3.5 w-3.5" viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5"><rect x="1" y="4" width="14" height="8" rx="1" /><path d="M8 14v-4M5 12l3 2 3-2" /></svg>
          Row↓
        </button>
        <button type="button" onClick={() => editor.chain().focus().deleteRow().run()} className="table-action-btn text-red-500 hover:!border-red-300 hover:!text-red-600" title="Delete row">
          <svg className="h-3.5 w-3.5" viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5"><rect x="1" y="4" width="14" height="8" rx="1" /><path d="M5 7l6 0" /></svg>
          Del Row
        </button>

        <div className="h-4 w-px bg-gray-200" />

        {/* Column operations */}
        <button type="button" onClick={() => editor.chain().focus().addColumnBefore().run()} className="table-action-btn" title="Add column left">
          <svg className="h-3.5 w-3.5" viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5"><rect x="4" y="1" width="8" height="14" rx="1" /><path d="M2 8h4M4 6l-2 2 2 2" /></svg>
          Col←
        </button>
        <button type="button" onClick={() => editor.chain().focus().addColumnAfter().run()} className="table-action-btn" title="Add column right">
          <svg className="h-3.5 w-3.5" viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5"><rect x="4" y="1" width="8" height="14" rx="1" /><path d="M14 8h-4M12 6l2 2-2 2" /></svg>
          Col→
        </button>
        <button type="button" onClick={() => editor.chain().focus().deleteColumn().run()} className="table-action-btn text-red-500 hover:!border-red-300 hover:!text-red-600" title="Delete column">
          <svg className="h-3.5 w-3.5" viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5"><rect x="4" y="1" width="8" height="14" rx="1" /><path d="M7 5v6" /></svg>
          Del Col
        </button>

        <div className="h-4 w-px bg-gray-200" />

        {/* Merge / Split */}
        <button type="button" onClick={() => editor.chain().focus().mergeCells().run()} className="table-action-btn" title="Merge selected cells">
          <svg className="h-3.5 w-3.5" viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5"><rect x="1" y="1" width="14" height="14" rx="1" /><path d="M1 8h14M8 1v14" /></svg>
          Merge
        </button>
        <button type="button" onClick={() => editor.chain().focus().splitCell().run()} className="table-action-btn" title="Split cell">
          <svg className="h-3.5 w-3.5" viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5"><rect x="1" y="1" width="14" height="14" rx="1" /><path d="M8 1v14" strokeDasharray="2 2" /></svg>
          Split
        </button>

        <div className="h-4 w-px bg-gray-200" />

        {/* Cell background color */}
        <div className="flex items-center gap-1" title="Cell background color">
          <input
            type="color"
            value={cellBgColor}
            onChange={(e) => handleSetCellBg(e.target.value)}
            className="h-7 w-7 cursor-pointer rounded-lg border border-gray-200 p-0.5"
          />
          {cellBgColor !== "#ffffff" && (
            <button type="button" onClick={handleClearCellBg} className="text-[10px] font-semibold text-gray-500 hover:text-primary" title="Clear cell color">
              Clear
            </button>
          )}
        </div>

        <div className="h-4 w-px bg-gray-200" />

        {/* Delete table */}
        <button type="button" onClick={() => editor.chain().focus().deleteTable().run()} className="table-action-btn text-red-500 hover:!border-red-300 hover:!text-red-600" title="Delete table">
          <svg className="h-3.5 w-3.5" viewBox="0 0 16 16" fill="none" stroke="currentColor" strokeWidth="1.5"><rect x="1" y="1" width="14" height="14" rx="1" /><path d="M4 4l8 8M12 4l-8 8" /></svg>
          Delete Table
        </button>
      </div>
    )}
    </>
  )
}
```

**Toolbar key points:**

- **Active-state via marks**: bold/italic/underline/strike read `editor.state.selection.$head.marks()`; heading, lists, link, table use `editor.isActive(...)`.
- **Tailwind `primary`**: the active button style (`border-primary bg-primary text-white`) relies on a `--color-primary` Tailwind theme token — define it in your `globals.css` (`@theme`) or replace `primary` with your own color.
- **Hidden file input lives in the toolbar** and its `click()` is triggered by the Image button; selected files flow up via `onInlineImageUpload`.
- **`table-action-btn`**, **image resize handles**, and the **placeholder** all have required CSS in section 9.
- The `useCallback` on cell-bg handlers (not strictly required) avoids stale closures on the editor reference.

---

## 6. Link Prompt

`src/components/blogs/BlogLinkPrompt.tsx`

A small modal/inline bar used to collect the URL. Supports Enter to confirm, Escape to cancel.

```tsx
import { BlogLinkPromptProps } from "@/types/blog"

export default function BlogLinkPrompt({
  linkUrl,
  onUrlChange,
  onConfirm,
  onCancel,
}: BlogLinkPromptProps) {
  return (
    <div className="flex items-center gap-3 rounded-2xl border border-gray-200 bg-gray-50 px-4 py-3">
      <input
        type="url"
        value={linkUrl}
        onChange={(e) => onUrlChange(e.target.value)}
        placeholder="https://example.com"
        className="min-w-0 flex-1 rounded-xl border border-gray-200 bg-white px-3 py-2 text-sm outline-none focus:border-primary"
        autoFocus
        onKeyDown={(e) => {
          if (e.key === "Enter") onConfirm()
          if (e.key === "Escape") onCancel()
        }}
      />
      <button
        type="button"
        onClick={onConfirm}
        className="rounded-full bg-primary px-4 py-2 text-xs font-bold text-white transition-colors hover:bg-primary/90"
      >
        Add
      </button>
      <button
        type="button"
        onClick={onCancel}
        className="rounded-full border border-gray-200 px-4 py-2 text-xs font-bold text-gray-600 transition-colors hover:bg-gray-50"
      >
        Cancel
      </button>
    </div>
  )
}
```

Props type:

```ts
export interface BlogLinkPromptProps {
  linkUrl: string
  onUrlChange: (url: string) => void
  onConfirm: () => void
  onCancel: () => void
}
```

---

## 7. Host Form Wiring

The host is a `"use client"` form that creates the editor and lays out `<EditorContent>` under the toolbar. The complete form also collects title/author/category/excerpt/thumbnail; only the **editor-specific wiring** is shown here (minus the plain field markup). All the important integration points:

### 7.1 Create the editor + track raw HTML

```tsx
const initialContent = initialPost?.content || "<p></p>"

const [editorHtml, setEditorHtml] = useState(() => initialContent)

function handleEditorChange(html: string) {
  setEditorHtml(html)
  clearFieldError("content")
}

const editor = useBlogEditor(
  initialContent,
  isReadOnly,
  handleEditorChange,
  insertImages
);
```

### 7.2 Selection-tick hack (keep toolbar active states fresh)

Tiptap does not re-render the host on every cursor movement, so the toolbar's active states (bold/heading/etc.) would go stale. Force a re-render on `selectionUpdate`:

```tsx
const [, setSelectionTick] = useState(0)
useEffect(() => {
  if (!editor) return
  const handler = () => setSelectionTick((n) => n + 1)
  editor.on("selectionUpdate", handler)
  return () => { editor.off("selectionUpdate", handler) }
}, [editor])
```

### 7.3 Inline image upload → insert into the document

```tsx
// Upload array of image files and insert into editor
async function insertImages(files: File[]) {
  if (!editor || isReadOnly || files.length === 0) return

  setInlineUploading(true)
  setErrorMessage("")

  try {
    for (const file of files) {
      const url = await uploadImage(file)                       // returns /api/blog/images/<name>
      editor.chain().focus().setImage({ src: url, alt: file.name }).run()
    }
  } catch (uploadError) {
    const message =
      uploadError instanceof Error
        ? uploadError.message
        : "Unable to upload inline image"
    setErrorMessage(message)
  } finally {
    setInlineUploading(false)
  }
}

// Upload inline images from file input
async function handleInlineImageUpload(event: React.ChangeEvent<HTMLInputElement>) {
  const files = Array.from(event.target.files ?? []).filter((file) =>
    file.type.startsWith("image/")
  )
  await insertImages(files)
  event.target.value = ""
}
```

### 7.4 Link confirm

```tsx
function handleLinkConfirm() {
  const trimmed = linkUrl.trim()
  if (trimmed) {
    editor?.chain().focus().extendMarkRange("link").setLink({ href: trimmed }).run()
  }
  setShowLinkPrompt(false)
  setLinkUrl("")
}
```

`extendMarkRange("link")` extends the selection across the nearest link so the URL overwrites it rather than stacking.

### 7.5 Word / character count

```tsx
const editorText = editor?.getText().trim() ?? ""
const charCount = editorText.length
const wordCount = editorText ? editorText.split(/\s+/).filter(Boolean).length : 0
```

### 7.6 Layout — toolbar + `<EditorContent>`

```tsx
<div className={`rounded-[24px] border bg-white shadow-sm lg:max-h-[65vh] lg:overflow-y-auto ${fieldErrors.content ? "border-red-400" : "border-gray-200"}`}>
  <div className="lg:sticky top-0 z-10 rounded-t-[24px] bg-white/80 backdrop-blur-md">
    <BlogEditorToolbar
      editor={editor}
      isReadOnly={isReadOnly}
      inlineImageInputRef={inlineImageInputRef}
      onLinkPromptClick={handleLinkPromptClick}
      onInlineImageUpload={handleInlineImageUpload}
      inlineUploading={inlineUploading}
    />
  </div>
  <EditorContent editor={editor} />
</div>
```

The toolbar is sticky while the content area scrolls (`lg:max-h-[65vh] lg:overflow-y-auto` + `lg:sticky top-0`).

### 7.7 Validation — content must have visible text

```tsx
function validateForm(): boolean {
  const errors: Record<string, string> = {}
  // ... other fields ...

  const contentText = editor?.getText().trim() ?? ""
  if (!contentText) errors.content = "Content is required"

  // ...
  case "content":
    editor?.commands.focus()
    break
  // ...
}
```

`editor.getText()` returns only visible text — an empty editor (even with placeholder) fails validation.

### 7.8 Save — submit the HTML string

```tsx
body: JSON.stringify({
  title: form.title.trim(),
  excerpt: form.excerpt.trim(),
  author: form.author.trim(),
  category: form.category.trim(),
  content: editor?.getHTML() || form.content,
  thumbnailImage: form.thumbnailImage.trim() || null,
}),
```

The full HTML (with inline image `<img>` tags) is what gets persisted.

---

## 8. Image Upload Integration

Inline and thumbnail images are **never base64** — each file is uploaded to a server route and the returned URL is used in the document / stored on the record.

### Client helper — `uploadImage()`

`src/lib/upload.ts`

```ts
export async function uploadImage(file: File, folder?: string) {
  const formData = new FormData()
  formData.append("file", file)
  if (folder) formData.append("folder", folder)

  const response = await fetch("/api/upload", {
    method: "POST",
    body: formData,
  })

  const data = await response.json()

  if (!response.ok || !data?.success) {
    throw new Error(data?.message || "Image upload failed")
  }

  return String(data.data.url)
}
```

### Endpoint summary — `POST /api/upload`

- Accepts `multipart/form-data`: field `file` (image only), optional `folder`.
- Rejects non-image MIME types with `400`.
- Writes to `public/images/<folder>/` with a collision-safe name (`${Date.now()}-${crypto.randomUUID()}${ext}`).
- Returns `201` with `data.url` = `/api/<folder>/images/<fileName>`.

The full upload + serving pattern (path-traversal guards, immutable caching, orphan cleanup) is documented separately in **`runtime-image-upload-in-public-folder-with-best-practice.md`** — read that for the complete server-side story.

For this editor specifically: call `uploadImage(file)` (no folder → defaults to `blog`) when inserting inline images, and the documented `BlogLivePreview` + public pages render those URLs via `./api/blog/images/<fileName>`.

---

## 9. CSS & Styling

Two concerns:

1. **`.blog-prose`** — shared typography used by the editor, the live preview, *and* the public pages, so content looks identical everywhere.
2. **`.tiptap`** — editor-only chrome: outline reset, placeholder, table mechanics, resize handles, selected-cell highlight.

Add all of this to your global CSS (`src/app/globals.css` with Tailwind v4 — `@import "tailwindcss"` at the top; for Tailwind v3 add inside `@layer components`).

```css
/* === Unified Blog Prose (Editor + Preview + Frontend) === */
.blog-prose {
  overflow-x: auto;
  -webkit-overflow-scrolling: touch;
}

.blog-prose p {
  margin: 0 0 0.9rem;
  min-height: 1em;
}

.blog-prose p:last-child {
  margin-bottom: 0;
}

.blog-prose h1,
.blog-prose h2,
.blog-prose h3,
.blog-prose h4,
.blog-prose h5,
.blog-prose h6 {
  font-weight: 800;
  line-height: 1.15;
  color: #0A0D14;
  margin: 1.3rem 0 0.6rem;
}

.blog-prose h1 { font-size: 2.1rem; margin: 1.4rem 0 0.7rem; }
.blog-prose h2 { font-size: 1.7rem; }
.blog-prose h3 { font-size: 1.45rem; }
.blog-prose h4 { font-size: 1.25rem; margin: 1.2rem 0 0.55rem; }
.blog-prose h5 { font-size: 1.1rem; margin: 1rem 0 0.45rem; }
.blog-prose h6 { font-size: 1rem; margin: 0.9rem 0 0.4rem; text-transform: uppercase; letter-spacing: 0.12em; }

.blog-prose strong,
.blog-prose b {
  font-weight: 700 !important;
}

.blog-prose em,
.blog-prose i {
  font-style: italic;
}

.blog-prose u {
  text-decoration: underline;
}

.blog-prose s,
.blog-prose strike {
  text-decoration: line-through;
}

.blog-prose mark {
  background-color: #fef08a;
  padding: 0.1em 0.2em;
  border-radius: 0.2em;
}

.blog-prose blockquote {
  margin: 1rem 0;
  border-left: 4px solid #DCAE1D;
  border-radius: 0 0.9rem 0.9rem 0;
  background: rgba(220, 174, 29, 0.08);
  padding: 0.95rem 1rem;
  color: #334155;
  font-style: italic;
}

.blog-prose ul,
.blog-prose ol {
  margin: 0.85rem 0;
  padding-left: 1.35rem;
  list-style: revert;
}

.blog-prose li {
  margin: 0.35rem 0;
}

.blog-prose ul[data-type="taskList"] {
  list-style: none;
  padding-left: 0;
}

.blog-prose a {
  color: #00303F;
  text-decoration: underline;
  text-underline-offset: 0.18em;
}

.blog-prose img {
  display: block;
  max-width: 100%;
  height: auto;
  border-radius: 1rem;
  margin: 1rem 0;
}

.blog-prose pre {
  margin: 1rem 0;
  overflow-x: auto;
  border-radius: 1rem;
  background: #0f172a;
  color: #e2e8f0;
  padding: 1rem 1.1rem;
}

.blog-prose code {
  border-radius: 0.5rem;
  background: rgba(15, 23, 42, 0.06);
  padding: 0.16rem 0.4rem;
  font-size: 0.92em;
}

.blog-prose pre code {
  background: transparent;
  color: inherit;
  padding: 0;
}

.blog-prose hr {
  margin: 1.2rem 0;
  border: none;
  border-top: 1px solid #e5e7eb;
}

/* Table styles — editor + public rendering */
.tiptap .tableWrapper {
  width: 100%;
  position: relative;
  overflow-x: auto;
}

/* Table column resize handle */
.column-resize-handle {
  position: absolute;
  right: -2px;
  top: 0;
  bottom: 0;
  width: 4px;
  background: #0567E1;
  cursor: col-resize;
  z-index: 10;
  pointer-events: auto;
}

.resize-cursor {
  cursor: col-resize !important;
}

.blog-prose table,
.tiptap table {
  border-collapse: separate;
  border-spacing: 0;
  width: 100%;
  margin: 1.2rem 0;
  border: 1px solid #e5e7eb;
  border-radius: 0.5rem;
  overflow: hidden;
}

.blog-prose th,
.tiptap th {
  background-color: #f8fafc;
  font-weight: 700;
  text-align: left;
  padding: 0.6rem 0.8rem;
  border-right: 1px solid #e5e7eb;
  border-bottom: 1px solid #e5e7eb;
}

.blog-prose th:first-child,
.tiptap th:first-child {
  border-left: none;
}

.blog-prose tr:first-child th,
.tiptap tr:first-child th {
  border-top: none;
}

.blog-prose td,
.tiptap td {
  padding: 0.5rem 0.8rem;
  border-right: 1px solid #e5e7eb;
  border-bottom: 1px solid #e5e7eb;
  vertical-align: top;
}

.blog-prose td:first-child,
.tiptap td:first-child {
  border-left: none;
}

.blog-prose tr:last-child td,
.tiptap tr:last-child td {
  border-bottom: none;
}

.blog-prose tr:hover td,
.tiptap tr:hover td {
  background-color: #f8fafc;
}

/* Editor-specific: selected cell highlight */
.tiptap .selectedCell {
  background-color: rgba(5, 103, 225, 0.08);
}

.blog-prose p[style*="text-align: justify"],
.blog-prose h1[style*="text-align: justify"],
.blog-prose h2[style*="text-align: justify"],
.blog-prose h3[style*="text-align: justify"],
.blog-prose h4[style*="text-align: justify"],
.blog-prose h5[style*="text-align: justify"],
.blog-prose h6[style*="text-align: justify"] {
  text-align: justify;
}

/* TipTap editor surface — editor-only overrides */
.tiptap {
  outline: none;
}

.tiptap p.is-editor-empty:first-child::before {
  color: #94a3b8;
  content: attr(data-placeholder);
  float: left;
  height: 0;
  pointer-events: none;
}

/* Table sub-toolbar action buttons */
.table-action-btn {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  padding: 4px 8px;
  font-size: 11px;
  font-weight: 600;
  color: #4b5563;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  background: white;
  cursor: pointer;
  transition: all 0.15s ease;
  white-space: nowrap;
}

.table-action-btn:hover {
  border-color: #0567E1;
  color: #0567E1;
}

/* Image resize handles */
[data-resize-handle] {
  width: 12px;
  height: 12px;
  background: #0567E1;
  border: 2px solid white;
  border-radius: 50%;
  z-index: 10;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.2);
}

[data-resize-handle]:hover {
  background: #0456b8;
  transform: scale(1.15);
}

.tiptap img.ProseMirror-selectednode {
  outline: 2px solid #0567E1;
  outline-offset: 2px;
  border-radius: 0.5rem;
}
```

**Why both selectors?** Table styles target both `.blog-prose table` (public/preview) and `.tiptap table` (inside the editor). The placeholder needs `.tiptap p.is-editor-empty:first-child::before` to render `data-placeholder` text Tiptap injects. Image resize handles are the `[data-resize-handle]` 12px blue dots; the blue selection ring is `.tiptap img.ProseMirror-selectednode`.

> If your template uses Tailwind v4's `@theme` to define `--color-primary: #0567E1;`, the toolbar's `text-primary`/`bg-primary`/`border-primary` classes already match the resize-handle blue in this CSS.

---

## 10. Public Rendering

### 10.1 Render saved HTML with the shared prose styles

```tsx
<div
  className="blog-prose mt-10 text-[15px] leading-8"
  dangerouslySetInnerHTML={{ __html: post.content }}
/>
```

The same `.blog-prose` CSS from section 9 styles the rendered article, so public output matches the editor and preview exactly.

### 10.2 Normalize inline image URLs (legacy → API form)

If older records stored direct public paths (`/images/blog/x.jpg`) but new uploads use `/api/blog/images/x.jpg`, normalize before render:

```ts
export const BLOG_PUBLIC_IMAGE_PREFIX = "/images/blog/"
export const BLOG_API_IMAGE_PREFIX = "/api/blog/images/"

export function normalizeBlogContentHtml(html: string): string {
  return html.replace(
    /src=(["'])(\/(?:api\/blog\/images|images\/blog)\/[^"']+)\1/g,
    (_match, quote: string, src: string) => {
      const filename = path.basename(src)
      return `src=${quote}${BLOG_API_IMAGE_PREFIX}${filename}${quote}`
    }
  )
}
```

Call it server-side when saving and when serializing responses.

### 10.3 Images use plain `<img>`

Uploaded images are served through API routes (not the Next.js image optimizer), so rendered content — including the inline images inside the rich-text HTML — uses standard `<img>` tags. When you author the preview component, note that `dangerouslySetInnerHTML` content is **trusted admin-authored HTML** (no sanitization library is used; only the paste-cleaner in section 4 runs on input).

---

## 11. Save / Load Flow

```
BLOG POST LIFE CYCLE
====================

CREATE
  BlogComposerForm (mode="create", initialPost=null)
    └─ initialContent = "<p></p>"
    └─ useBlogEditor("<p></p>", ...)
    └─ user types / uploads images → onCreate/onUpdate → editorHtml state
    └─ handleSubmit → POST /api/blog
         body.content = editor.getHTML()   ← <p>…</p><img src="/api/blog/images/…">
    └─ server: normalizeBlogContentHtml(content) → store

EDIT
  DashboardBlogCard → openEdit(post) → BlogComposerModal key={`edit-${post.id}`}
    └─ BlogComposerForm initialPost = post
    └─ initialContent = post.content            ← saved HTML seeded back in
    └─ useBlogEditor(post.content, ...)         ← editor shows existing content
    └─ handleSubmit → PUT /api/blog/:id
         body.content = editor.getHTML()        ← edited HTML
    └─ server: diff old vs new HTML → orphan cleanup for removed images

VIEW  (read-only)
  BlogComposerForm mode="view"
    └─ isReadOnly → editable:false, toolbar hidden, editor non-interactive

PUBLIC
  Blog detail page → GET /api/blog?slug=…
    └─ serializePost → normalizeBlogContentHtml(content)
    └─ <div className="blog-prose" dangerouslySetInnerHTML={{ __html: content }} />
```

Two details make reloading reliable:

1. **`content: initialContent`** — Tiptap accepts raw saved HTML and reconstructs the document. Pair it with `immediatelyRender: false`.
2. **`key={`${dialogMode}-${selectedPost?.id}`}` on the modal** — guarantees a **fresh editor instance** per open. Without the key, React reuses the same editor between posts and stale content leaks in.

---

## 12. Key Gotchas

1. **`immediatelyRender: false` is mandatory** in Next.js. Without it you get hydration mismatches because Tiptap renders on the server.
2. **The selection-tick hack** (section 7.2) is required or the toolbar's active-button states freeze as you move the cursor.
3. **`@tiptap/extension-font-size` is a pre-release** (`3.0.0-next.3`). It depends on `TextStyle`. If a stable release resolves, verify `setFontSize`/`unsetFontSize` still exist.
4. **No undo/redo button** — StarterKit enables the commands, but the toolbar doesn't expose them. Add an undo/redo pair if your users expect them.
5. **Content is trusted HTML.** There is no `DOMPurify`/`sanitize-html` on output — only the Word/Google-Docs cleanup on paste. Only allow admin/trusted users to author content.
6. **Images bypass `next/image`.** Because uploaded images are served from your own `/api/.../images` routes, inline rich-text images are plain `<img>`. Don't try to feed them through `<Image>`.
7. **Find the image URLs in host, not hook.** `insertImages` is passed *into* the hook from the host — the hook never touches `fetch`. This keeps the editor pure and reusable.
8. **`openOnClick: false`** on the Link extension means links don't navigate in the editor — intended. Use the URL prompt / `onLinkPromptClick` to set links.

---

## 13. Flow Diagram

```
Toolbar "Image" btn ──► inlineImageInputRef.click() ──► hidden <input type=file> ──┐
                                          │                                     │ files
                                          ▼                                     ▼
Paste image  ─────────► handlePaste ─────► insertImages(files) ──► uploadImage(file)   │
                                          │                          │ fetch POST /api/upload
Drag image ───────────► handleDrop ──────►│                          ▼
                                          │                     {url: "/api/blog/images/…"}
                                          ▼
                          editor.chain().focus().setImage({ src: url })

                                    ✦ ✦ ✦

Editor typing / formatting
        │  onCreate / onUpdate
        ▼
onUpdate(editor.getHTML()) ──► editorHtml state ──► wordCount/charCount + live preview

Toolbar actions ──► editor.chain().focus().toggleBold()/setColor()/…run()

                                    ✦ ✦ ✦

handleSubmit ──► content: editor.getHTML() ──► POST/PUT /api/blog
        │
        ▼
store HTML ──► normalizeBlogContentHtml(image srcs)
        │
        ▼
Public page ──► <div className="blog-prose" dangerouslySetInnerHTML={{ __html: content }} />
        │
        ▼
Images load via GET /api/blog/images/[filename] (immutable cache)
```