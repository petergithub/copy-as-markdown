# Requirments

## copy url without encoding option

prompt:
target: copy url without encoding for Unicode characters. add a option let user can optionally encode the url
e.g. `https://github.com/yorkxin/copy-as-markdown?%E4%B8%AD%E6%96%87` -> `https://github.com/yorkxin/copy-as-markdown?中文`
[yorkxin/copy-as-markdown: A browser extension to copy tabs and links as Markdown](https://github.com/yorkxin/copy-as-markdown?%E4%B8%AD%E6%96%87)
[yorkxin/copy-as-markdown: A browser extension to copy tabs and links as Markdown](https://github.com/yorkxin/copy-as-markdown?中文)
[gitlab.jasolar.com](https://gitlab.jasolar.com/Institute-of-IM/ai/agent-manager/amdoc/-/blob/main/API设计与管理规范.md?ref_type=heads)

## copy url without encoding menu

prompt:
target: copy url without encoding for Unicode characters.
1. add a button 'Current tab link without encoding' in popup
2. location: below the button 'Current tab link' in popup @src/static/popup.html

**Special Characters in URLs**
- Unicode characters (like `%E4%B8%AD%E6%96%87`) will be decoded to readable text (like `中文`)
- Spaces (`%20`) and parentheses (`%28`, `%29`) remain encoded for markdown compatibility
- Other reserved characters may be decoded or remain encoded based on markdown URL requirements

e.g. `https://github.com/yorkxin/copy-as-markdown?%E4%B8%AD%E6%96%87` -> `https://github.com/yorkxin/copy-as-markdown?中文`

Which characters besides spaces (%20) and parentheses (%28, %29) should remain encoded in the URL? The requirement mentions 'other reserved characters may be decoded
or remain encoded based on markdown URL requirements.' Should we keep square brackets (%5B, %5D) encoded to avoid breaking markdown link syntax? What about backticks,
asterisks, underscores?

  1. Decode all except spaces/parentheses
     Keep only %20, %28, %29 encoded; decode everything else (including square brackets). Simpler but may break markdown if brackets appear in URL.
  2. Keep markdown-special chars encoded
     Keep spaces, parentheses, square brackets (%5B, %5D), backticks (%60), asterisks (%2A), underscores (%5F) encoded. Safer for markdown compatibility.
❯ 3. Use a curated deny list
     Keep spaces, parentheses, square brackets encoded; decode others. I'll research markdown URL spec to determine which chars are problematic.
  4. Type something.


 ● Should the new button have a visibility setting in the options page (like the existing 'Current tab link' button)? Following existing pattern, we would add
 'singleLinkWithoutEncoding' to built-in style settings.
   → Yes, add setting
 ● Should we add 'link-without-encoding' format to multiple tab exports (All tabs / Selected tabs) now, or only for single tab?
   → Only single tab
