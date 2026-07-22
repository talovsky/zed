# Linux LTR-Based Bidi Run Support Plan

## Goal

Bring Linux text shaping to parity with the existing macOS implementation of
LTR-based bidi run support. Source lines must keep a left-to-right editor base
direction while correctly shaping, displaying, cursoring through, hit-testing,
and selecting right-to-left text runs.

Linux uses `gpui_wgpu::CosmicTextSystem` for both Wayland and X11, so the work
belongs primarily in `crates/gpui_wgpu/src/cosmic_text_system.rs`. The shared
GPUI and editor mapping code should remain platform-independent.

## Scope

This work must provide:

- A forced left-to-right paragraph base direction for source lines.
- Correct visual ordering and shaping for Hebrew, Arabic, and other RTL runs.
- Accurate logical byte ranges, visual x ranges, and resolved directions in
  `LineLayout::visual_text_segments`.
- Correct caret placement, mouse hit-testing, and selection painting for pure
  RTL and mixed-direction text.
- Consistent behavior under both Wayland and X11.

This work does not include:

- A full right-to-left editor mode.
- Visual-order cursor navigation.
- Direction-aware caret affinity or dual carets at ambiguous bidi boundaries.
- New behavior for rectangular or block selections.
- Windows parity, which uses a separate DirectWrite text system.

## Current State

The shared editor implementation already uses `VisualTextSegment` as the
logical-to-visual mapping primitive. macOS fills those segments from CoreText's
resolved runs and explicitly gives each source line an LTR paragraph style.

Cosmic Text already shapes bidi text and exposes the information needed for the
mapping on each `LayoutGlyph`:

- `start` and `end` identify the logical UTF-8 cluster range.
- `x` and `w` identify the cluster's visual bounds.
- `level` identifies the resolved bidi embedding level and direction.

The Linux path currently discards that information. It builds
`visual_text_segments` with `LineLayout::default_visual_text_segments`, which
assumes that all glyphs progress left to right. RTL text can therefore be drawn
in visual order while caret placement, hit-testing, and partial selections use
incorrect mappings.

Cosmic Text also currently creates its bidi analysis with an automatically
chosen paragraph level. A line whose first strong character is RTL can become
an RTL paragraph, which conflicts with Zed's LTR code-editor model and can move
leading indentation or code syntax to the wrong visual side.

## Required Changes

### 1. Allow Cosmic Text shaping to use an explicit LTR base level

Add or adopt a Cosmic Text API that accepts an optional paragraph base level
when constructing a `ShapeLine`. Zed's Linux text system must pass
`unicode_bidi::Level::ltr()`.

The preferred implementation is an upstreamable Cosmic Text API rather than
injecting LTR formatting characters into the shaped string. Injected
characters would shift byte indices, interact with font fallback, and require
filtering synthetic glyphs from the resulting layout.

If Cosmic Text cannot be updated immediately, any temporary local patch must:

- Preserve byte indices relative to the original source string.
- Exclude synthetic controls from shaped glyphs and visual text segments.
- Preserve line width and glyph positions.
- Include regression tests that make the workaround's assumptions explicit.

### 2. Build visual text segments from Cosmic Text layout glyphs

Replace the call to `LineLayout::default_visual_text_segments` in the Cosmic
Text layout path with a helper that consumes the original `LayoutGlyph`s.

For each visually laid-out cluster, the helper must produce a
`VisualTextSegment` containing:

- `logical_range`: the cluster's `start..end` byte range.
- `x_range`: the occupied visual range derived from `x` and `w`.
- `direction`: `RightToLeft` for odd embedding levels and `LeftToRight` for
  even embedding levels.

The helper must account for:

- Multiple glyphs belonging to the same logical cluster.
- Ligatures that cover multiple source characters.
- Combining marks and zero-advance glyphs.
- Missing or deliberately skipped glyphs, including the existing emoji
  variation-selector handling.
- Font fallback boundaries that split `ShapedRun`s without splitting the
  logical cluster mapping incorrectly.
- Floating-point positions whose order differs by negligible rounding error.

Prefer constructing the segments while iterating Cosmic Text's layout glyphs,
before glyphs are regrouped into GPUI `ShapedRun`s by font. Font grouping is a
painting concern and must not determine bidi mapping order.

### 3. Preserve segment metadata through existing layout transformations

Verify that Cosmic Text segments remain correct when the layout is:

- Split around bidi paragraph separators.
- Offset and joined by `layout_line_with_separators`.
- Split into wrapped rows by GPUI.
- Clipped into shaped-line fragments.
- Combined with inline elements or invisibles in the editor.

Existing shared code already carries visual segments through most of these
operations. Fix shared code only when a Linux test demonstrates a
platform-independent mapping bug.

### 4. Add focused Cosmic Text tests

Add tests alongside `cosmic_text_system.rs` for at least these cases:

- A Hebrew word has decreasing logical indices in increasing visual x order.
- An Arabic word shapes correctly and maps every cluster to a valid UTF-8
  boundary.
- Leading spaces and tabs remain at the LTR start of a Hebrew-first line.
- Opening and closing code syntax remain outside an embedded RTL run, such as
  `<span>Hebrew text</span>`.
- LTR text followed by RTL text and then LTR text produces segments in the
  expected visual order and direction.
- Numbers and punctuation inside or adjacent to an RTL run retain their
  resolved bidi levels.
- Ligatures, combining marks, and emoji do not produce overlapping or invalid
  logical ranges.
- Font fallback within RTL text does not change logical-to-visual mapping.
- Bidi paragraph separators preserve absolute byte indices and x offsets.

Tests should assert segment ranges and mapping behavior, not exact glyph IDs or
font-specific pixel widths unless the test supplies a deterministic font.

### 5. Add shared mapping and editor integration coverage

Exercise the resulting Linux-shaped layouts through the shared APIs:

- `LineLayout::x_for_index`
- `LineLayout::closest_index_for_x`
- `LineLayout::index_for_x`
- `LineLayout::x_ranges_for_range`

Integration coverage must verify:

- Clicking each visual side of an RTL cluster returns the expected logical
  boundary.
- Logical cursor movement produces the expected caret x positions.
- A selection contained within one RTL run paints the correct visual range.
- A logical selection crossing mixed-direction runs may paint multiple visual
  ranges without covering excluded code islands.
- A complete-line selection still covers the full line.
- Wrapped mixed-direction lines preserve mappings on every display row.

Use GPUI's test text system with Cosmic Text so these tests can run without a
live Wayland or X11 session. Add a Linux smoke test only if platform wiring or
font discovery behaves differently from the headless text system.

## Validation

Run the focused GPUI and Cosmic Text tests first, followed by the editor tests
that cover cursor placement, hit-testing, selection painting, and wrapping.
Use `./script/clippy` for lint validation.

Manual Linux validation should cover both Wayland and X11 with the same sample
document containing:

- Indented Hebrew and Arabic lines.
- RTL comments next to LTR source code.
- HTML-like tags surrounding RTL text.
- Numbers, punctuation, combining marks, and emoji.
- Single-run, cross-run, multiline, and wrapped selections.

Confirm rendering, horizontal scrolling, mouse selection, keyboard movement,
copying, and editing at both boundaries of every RTL run.

## Completion Criteria

Linux parity is complete when:

- Linux shapes every editor line with an LTR paragraph base direction.
- Cosmic Text produces directional visual segments from its resolved glyph
  clusters rather than the LTR-only fallback helper.
- Caret placement, hit-testing, and selection tests pass for pure and mixed RTL
  content.
- Leading indentation and surrounding source syntax stay in the LTR editor
  structure.
- Segment ranges are valid UTF-8 ranges with no gaps caused by font fallback or
  skipped glyphs.
- Wayland and X11 show equivalent behavior in manual smoke testing.
- Existing LTR shaping, wrapping, font fallback, and missing-glyph tests remain
  unchanged or gain only intentional expectation updates.

## Suggested Implementation Order

1. Add tests that expose the current Linux segment-direction and base-direction
   failures.
2. Add explicit LTR paragraph-level support to Cosmic Text and use it from
   `CosmicTextSystem`.
3. Build visual text segments directly from `LayoutGlyph` cluster data.
4. Fix cluster merging and zero-width edge cases revealed by focused tests.
5. Run shared GPUI/editor mapping tests and address any transformation bugs.
6. Validate manually on Wayland and X11.

The expected implementation is localized, but paragraph-level control in
Cosmic Text is the main dependency risk. Once that API is available, the GPUI
mapping work should be straightforward.
