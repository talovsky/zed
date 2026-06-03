# Zed

Domain vocabulary for editor behavior where source-code editing semantics interact with bidirectional text.

## Language

**Editor Base Direction**:
Source lines are laid out with left-to-right paragraph direction unless Zed introduces an explicit right-to-left editor mode.
_Avoid_: Natural paragraph direction, automatic line direction

**Editor Indentation**:
Leading whitespace used for source indentation belongs to the left-to-right editor structure, even when followed by right-to-left text.
_Avoid_: Paragraph indentation

**LTR-Based Bidi Run Support**:
Support for shaping, displaying, cursoring through, and selecting right-to-left text runs inside Zed's left-to-right code editor model.
_Avoid_: Full bidi support, RTL editor mode

**Logical Cursor Movement**:
Cursor movement where the buffer index changes according to source text order rather than visual x-order.
_Avoid_: Visual cursor movement

**Logical Selection**:
A contiguous range in buffer order, regardless of how many visual pieces are needed to paint it.
_Avoid_: Visual selection

**Visual Selection Segment**:
A painted highlight span representing part of a logical selection after shaping and bidirectional reordering.
_Avoid_: Selection range

**Visual Text Segment**:
A visually contiguous shaped span with a logical buffer range, x range, and direction or affinity metadata.
_Avoid_: Glyph run, selection segment

**Default Bidi Caret Affinity**:
When one logical insertion index maps to multiple plausible visual caret positions, Zed chooses the left-to-right-side caret position unless a future interaction model tracks approach direction.
_Avoid_: Automatic caret affinity, visual caret movement

**Code Island**:
Source-code syntax embedded in surrounding right-to-left natural language text that should remain visually left-to-right and code-readable.
_Avoid_: Neutral text
