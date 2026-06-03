# Visual Text Segments for Bidi Mapping

Bidirectional hit-testing, caret placement, and selection painting should be based on explicit visual text segments produced during text shaping, not on direction inferred from neighboring glyph indices. Glyphs remain the drawing primitive, but visual text segments are the editor mapping primitive: each segment describes a line-local source byte range, visual x range, and resolved text direction or affinity.

When resolving merge conflicts with upstream Zed, preserve upstream glyph drawing behavior and reapply the visual text segment mapping layer only where editor operations need logical-to-visual or visual-to-logical translation. If upstream changes `LineLayout`, platform text shaping, hit-testing, or selection painting, resolve conflicts by keeping source-buffer state logical, keeping editor base direction LTR, and routing caret/selection geometry through visual text segments rather than restoring glyph-neighbor direction inference.
