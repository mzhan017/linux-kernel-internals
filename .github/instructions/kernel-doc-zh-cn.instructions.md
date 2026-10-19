---
applyTo: "docs/**/zh-CN/**/*.md"
---

# Rules for Simplified Chinese Kernel Documentation

- Write explanatory documentation in natural, technically precise Simplified
  Chinese.
- Keep kernel symbols, function and variable names, types, macros, enum values,
  configuration symbols, sysctl names, commands, filenames, file paths, URLs,
  and link destinations unchanged.
- Preserve inline code and fenced source-code, configuration, and command
  blocks exactly, including comments. In plain-text diagrams, translate only
  human-readable labels; keep identifiers and diagram structure intact.
- Translate descriptive headings, prose, table descriptions, admonitions,
  and plain-language diagram labels. Preserve Markdown formatting, ordering,
  and examples; do not omit source content or introduce new technical claims.
- Descriptive natural-language link labels can be translated, but retain
  identifiers and exact names that identify a source or project.
- Before changing a heading, preserve any existing explicit ID and check
  references to its generated fragment. Keep internal links working; use an
  explicit ID if necessary.
- Use terminology consistently, favoring “内存回收” (reclaim), “分配器”
  (allocator), “水位线” (watermark), “回写” (writeback), and “空闲页面”
  (free pages). Retain technical terms such as NUMA, LRU, cgroup, slab, zone,
  and kswapd where appropriate.
- Review the complete edited page for untranslated explanatory English,
  inadvertent changes to protected content, broken Markdown, and whitespace
  issues. For documentation-only edits, use focused checks rather than
  unrelated tests.
