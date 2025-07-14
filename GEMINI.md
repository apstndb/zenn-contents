# DEV.to Publishing Guidelines

This document outlines the best practices and platform-specific requirements for publishing articles on DEV.to, based on learnings from translating and preparing the "Spanner vs Firestore" cost analysis article.

## 1. AI-Assisted Content

- **Disclosure:** Any content generated or significantly assisted by AI must be clearly disclosed within the article's body. An author's note at the beginning is a good practice.
- **Example Disclosure:** *"This article was translated with the assistance of Google's Gemini 2.5 Pro. I have reviewed and edited the translation for accuracy and clarity."*

## 2. Tagging

- **Limit:** DEV.to enforces a strict maximum of **4 tags** per article.
- **Strategy:** Prioritize the most relevant technical tags (e.g., `gcp`, `spanner`, `firestore`, `cost`). The `#ABotWroteThis` tag is a good practice for transparency but can be omitted if the tag limit is reached, as the in-article disclosure is sufficient.

## 3. Image Handling

- **No Local Paths:** Images must be pre-uploaded. Local paths like `(/images/my-image.png)` will not work.
- **Procedure:** Upload images to DEV.to's uploader or another hosting service, and use the resulting absolute URL in the `![]()` Markdown tag.
- **Example:** `![Alt text](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/md33be8qyxnrq8k3or4r.png)`

## 4. KaTeX Syntax

DEV.to uses a specific Liquid Tag syntax for rendering mathematical formulas with KaTeX. The standard `$$...$$` and `$...$` syntax does not work.

- **Block-level Formulas:** Use the `{% katex %}` liquid tag.
  ```
  {% katex %}
  x = \frac{-b \pm \sqrt{b^2-4ac}}{2a}
  {% endkatex %}
  ```

- **Inline Formulas:** Use the `{% katex inline %}` liquid tag.
  ```
  The formula is {% katex inline %}E=mc^2{% endkatex %}.
  ```

## 5. Admonitions / Callouts

- **No Special Syntax:** DEV.to does not have a built-in syntax for admonitions or message blocks like Zenn's `:::message`.
- **Recommended Alternative:** Use an HTML `<div>` block to semantically group the content. This is preferable to misusing blockquotes (`>`).
- **Example:**
  ```html
  <div>
    <p><strong>Note:</strong> This is an important piece of information that stands apart from the main text.</p>
  </div>
  ```

## 6. Cross-Linking Translations

- For articles translated from another language, include a clear link to the original version in the author's note at the beginning of the article.
