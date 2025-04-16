---
tags:
  - clipboard
  - javascript
  - typescript
  - iOS
  - safari
date: 2025-04-16T16:01:61+02:00
draft: false
---

Stupid Safari and iOS forcing me to do this to copy text to clipboard. 😩

```
export async function copyToClipboard(text: string) {
  try {
    await navigator.clipboard.writeText(text);
  } catch (err) {
    // Fallback for devices like iOS that don't support Clipboard API
    const textarea = document.createElement("textarea");
    textarea.value = text;
    textarea.setAttribute("readonly", "");
    textarea.style.position = "absolute";
    textarea.style.left = "-9999px";
    document.body.appendChild(textarea);

    textarea.select();
    document.execCommand("copy");
    document.body.removeChild(textarea);
  }
}
```