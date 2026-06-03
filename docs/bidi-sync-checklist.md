# Bidi Sync Checklist

Use this checklist when rebasing or merging upstream Zed changes into the bidi fork.

- Check `LineLayout` mapping APIs: `x_for_index`, `closest_index_for_x`, `index_for_x`, and `x_ranges_for_range`.
- Check platform shaping output still creates visual text segments for bidi mapping.
- Check selection painting consumes multiple visual segments per line.
- Check caret display uses default LTR affinity at ambiguous bidi boundaries.
- Check leading indentation before RTL text remains part of the LTR editor structure.
- Prefer upstream glyph drawing, font fallback, shaping internals, and rendering performance changes unless they remove the visual segment mapping data.
- Preserve fork semantics for cursor x lookup, hit-testing, range highlighting, and line layout mappings.
- Do not resolve conflicts by restoring glyph-neighbor direction inference.
- Run the bidi-focused tests after every conflict resolution.
