---
name: kernel-doc-zh-translation
description: Translate Linux kernel documentation pages from English into Simplified Chinese while preserving code, kernel identifiers, filenames, links, and Markdown structure. Use when asked to translate a documentation page or selected English content in this repository.
---

# Linux Kernel Documentation English-to-Chinese Translation

Translate the requested documentation page or selected passage into clear,
technically accurate Simplified Chinese. When a file is tagged or selected,
use that file as the translation target unless the user specifies otherwise.

## Workflow

1. Read the complete target page and check the surrounding Chinese
   documentation for established terminology and style.
2. Identify prose, headings, admonitions, table descriptions, diagram labels,
   and other human-readable English separately from code and identifiers.
3. Translate all in-scope explanatory English. Do not stop after the visible
   excerpt if the request concerns the whole page.
4. Preserve the document's organization and verify that links, anchors, and
   Markdown structure remain valid.
5. Review the finished page for untranslated explanatory English, accidental
   changes to protected content, and whitespace errors.

## Translation rules

- Use Simplified Chinese. Prefer concise, natural technical writing over
  word-for-word translation.
- Preserve kernel terminology consistently. Use established translations
  such as “内存回收” for reclaim, “分配器” for allocator, “水位线” for
  watermark, “回写” for writeback, and “空闲页面” for free pages. Keep terms
  such as NUMA, LRU, cgroup, slab, zone, and kswapd when translating them
  would make the technical meaning less precise.
- Keep C identifiers, function and variable names, macros, enum names, types,
  constants, kernel configuration symbols, sysctl names, command names, file
  paths, source filenames, and literal code tokens unchanged.
- Keep URLs and link destinations unchanged. Descriptive natural-language
  link labels may be translated; filenames, symbol names, project names, and
  titles whose exact wording is needed to identify a source should remain
  unchanged.
- Preserve inline code exactly. Do not translate quoted identifiers or
  command fragments.
- Leave fenced source-code, configuration, and command blocks unchanged,
  including source comments and shell comments. For plain-text diagrams,
  translate human-readable labels only; preserve identifiers, arrows,
  alignment, and diagram structure.
- Translate explanatory text in headings, blockquotes, admonitions, tables,
  and prose. Retain Markdown formatting, list structure, table columns, and
  code spans.
- Keep existing link targets and explicit heading IDs intact. Before
  translating a heading, check whether another page links to its generated
  fragment. If the translated heading would break an existing fragment, add
  or retain an explicit ID using the repository's supported Markdown
  extensions.
- Do not silently omit a sentence, example, caveat, or section. Do not add
  technical claims that are not present in the source.

## Completion checks

- The requested page or selection has been translated in full.
- Natural-language English is not left behind except where intentionally
  retained as a technical term, proper name, identifier, filename, quotation,
  link label, or protected code/comment.
- All identifiers, paths, URLs, code blocks, and link destinations remain
  unchanged.
- Markdown fences, tables, list indentation, relative links, and heading
  anchors remain valid.
- Run a focused documentation or whitespace check if one is available; do not
  run unrelated builds or tests for a translation-only change.
