# Self-hosted webfonts

Latin-subset `.woff2` files pulled from the Google Fonts CDN and committed here so
the site serves them same-origin. This removes `fonts.googleapis.com` and
`fonts.gstatic.com` from the critical rendering path.

Only the faces the stylesheet actually uses are included:

| file | family | weight | style | used for |
|---|---|---|---|---|
| `dm-sans-400.woff2` | DM Sans | 400 | normal | body copy |
| `dm-sans-700.woff2` | DM Sans | 700 | normal | `<strong>`, `<th>` |
| `lora-400.woff2` | Lora | 400 | normal | headings, post titles |
| `lora-400-italic.woff2` | Lora | 400 | italic | blockquotes |
| `jetbrains-mono-400.woff2` | JetBrains Mono | 400 | normal | code, nav, meta, TOC |

`@font-face` declarations and the `preload` hints live in
`layouts/partials/extend_head.html`, which runs the files through Hugo Pipes so
they get fingerprinted URLs and immutable caching.

## Licensing

All three families are licensed under the SIL Open Font License 1.1; the full
text is in `OFL.txt`.

- **DM Sans** — Copyright 2014 The DM Sans Project Authors (https://github.com/googlefonts/dm-fonts)
- **Lora** — Copyright 2011 The Lora Project Authors (https://github.com/cyrealtype/Lora-Cyrillic), with Reserved Font Name "Lora".
- **JetBrains Mono** — Copyright 2020 The JetBrains Mono Project Authors (https://github.com/JetBrains/JetBrainsMono)
