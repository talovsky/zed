# LTR-Based Bidi Run Support

Zed is a code editor first, so initial bidirectional text support will preserve left-to-right editor structure while supporting right-to-left text runs for shaping, cursor display, hit-testing, and selection. This favors source-code editing semantics over full natural-language RTL editor behavior: cursor movement and selections remain logical buffer operations, ambiguous bidi caret positions default to the LTR-side position, and leading indentation remains part of the LTR editor structure.

Future work includes rectangular and block selections in mixed bidi text, visual-order cursor navigation, caret affinity based on approach direction, dual carets at bidi boundaries, wrapped-line continuation behavior for mixed bidi segments, Windows/Linux parity, and a full RTL editor mode.
