# wlx_enterprise-architect

Total Commander Lister-Plugin (WLX) for viewing UML class diagrams stored in
Sparx Enterprise Architect's SQLite-based project files (`.qea` / `.qeax`),
in both the F3 quick view and the Ctrl+Q panel.

Renders class/object diagrams (boxes with attribute/operation compartments —
operation signatures include parameters, attributes show multiplicity
(`[0..*]`) and default values where set. A default that's actually an
embedded resource blob — e.g. a stereotype's custom "EAShapeScript" (a ZIP
archive containing EA's own tiny vector-shape scripting language: UTF-16
source text with commands like `moveto`/`lineto`/`ellipse`/
`fillandstrokepath`), stored as a giant base64 string in an ordinary
`_image` attribute — shows a short placeholder in the text line itself,
plus an actual small preview icon rendered next to it by unzipping,
decoding, parsing, and interpreting the script's drawing commands (a
bounded core subset — see Known limitations); generalization/realization/
association/aggregation/composition, notes, packages, interfaces), package
diagrams, use case diagrams (actors, use cases, system boundaries), sequence
diagrams (lifelines, SeqNo-ordered messages), activity/statechart diagrams
(actions, states with nesting, decision/merge diamonds with their guard-
condition labels, fork/join bars, initial/final/history pseudostates,
control/object/state flows), and component/deployment/composite-structure
diagrams (component boxes with provided/required interfaces, 3D node/device
boxes, artifacts, ports, nested composite-structure parts). Read-only — no
editing.

Elements keep their own EA colors (Backcolor/Fontcolor/Bordercolor, set via
EA's own "Format" dialogs) instead of always rendering black-on-white. An
element with a custom image assigned in EA (`t_diagramobjects.ObjectStyle`
"ImageID=..." pointing into `t_image`) renders that actual embedded picture
— a screenshot, icon, or diagram mockup — via GDI+ instead of a generic
shape; these images travel with the `.qea`/`.qeax` file itself (no external
Sparx image library needed to view them). EA's own "NameUnderImage" setting
decides whether the element's name is also shown as a caption below the
image.
Connectors show their own name at the midpoint when set — this is also how
guard conditions on Activity decision/merge edges are stored in EA (as the
connector's name), so they now appear on the diagram automatically — plus
association role names at each end, and each connector's own EA color/line
weight instead of a uniform black 1px line. Notes
strip raw HTML markup (`<b>`, `<i>`, `<br>`, ...) that EA sometimes stores
verbatim in the Note field, showing the plain text instead of the tag
syntax. Dashboard-style "NavigationCell" tiles (links to another diagram,
e.g. on a "Getting Started" or custom overview page) show EA's own `Alias`
text when set — the field EA itself uses for a tile's human-readable
caption, which is often the ONLY place the visible title is stored (Name is
routinely empty on these).

Click a diagram element to select it (shown with a blue outline). With an
element selected, Ctrl+C / the context menu's "copy text" copies only that
element's name/stereotype/attributes/operations/note instead of the whole
diagram's text. Click empty space to deselect.

Right-click the diagram for a context menu: save the whole diagram as a PNG
file, copy it as an image to the clipboard, or copy text (the selected
element's text if one is selected, otherwise the whole diagram's text
summary). The exported image always covers the full diagram content, not
just the currently visible/panned viewport.

## Dependency: sqlite3.dll

This plugin does not bundle SQLite. You must place the official `sqlite3.dll`
next to `eaviewer.wlx` (32-bit) and/or `eaviewer.wlx64` (64-bit) — matching
Total Commander's own bitness — before the plugin can open any file.

Download the precompiled Windows binaries from the SQLite project itself:

- 64-bit: [sqlite-dll-win-x64-3530400.zip](https://www.sqlite.org/2026/sqlite-dll-win-x64-3530400.zip)
- 32-bit: [sqlite-dll-win-x86-3530400.zip](https://www.sqlite.org/2026/sqlite-dll-win-x86-3530400.zip)

(Both linked from the official [SQLite download page](https://www.sqlite.org/download.html) —
check there for a newer version if these links go stale.)

Unzip each archive and copy the contained `sqlite3.dll` next to the matching
plugin file.

PNG export (right-click context menu) and rendering an element's own custom
image (see above) both use `gdiplus.dll`, which ships with Windows itself
since XP SP2 — nothing to download for that.

Decoding EAShapeScript preview icons (see above) needs DEFLATE
decompression for the ZIP-wrapped script text — uses FPC's own `paszlib`
package (`ZStream`/`TDecompressionStream`, raw-deflate mode), which ships
with the Lazarus/FPC installation itself (precompiled `.ppu` units for both
`i386-win32` and `x86_64-win64` were already present in this project's FPC
3.2.2 install) — nothing extra to download or configure for that either.

## Build

See the repository-wide `CLAUDE.md` for the Lazarus/FPC build setup. Two
build modes in `eaviewer.lpi`:

```
lazbuild.exe --build-mode="Release 32" eaviewer.lpi
lazbuild.exe --build-mode="Release 64" eaviewer.lpi
```

Produces `eaviewer.wlx` (32-bit) and `eaviewer.wlx64` (64-bit).

**Both build modes use `-O2`, not `-O3`** — deliberately, since `-O3`
demonstrably miscompiled this codebase once already (see below) and a
GDI-drawing Lister-plugin has no hot loop that needs the extra aggressive
optimization; `-O2` is the safer default here, not just a workaround for one
known instance.

**FPC 3.2.2 `-O3` pitfall, confirmed in this codebase (this is *why* the
project builds at `-O2`):** a nested `Min(Round(X * Zoom), Min(A, B) div N)`
expression assigned directly to a result variable can silently miscompile at
`-O3` — the sub-expressions evaluate correctly when printed individually,
but the direct assignment produces a wrong (much smaller) value. Observed in
`DrawRoundedActionBox`/`DrawStateBox` (`uRenderActivity.pas`) after adding a
few more local variables nearby (color-handling `OldBrush`/`Pen`/
`OldTextColor`) tipped some register-allocation threshold: every Action/
Activity/State box rendered as a full ellipse instead of a rounded
rectangle, on both 32- and 64-bit, reproduced with a clean rebuild (ruling
out stale `.ppu`/`.o` caching) and confirmed architecture-independent via a
standalone GDI harness calling the real `DrawActivityObject` directly. The
affected expressions were *also* rewritten as separate named intermediate
variables (`ZoomCap`/`SizeCap` in the functions above, and `Draw3DBox` in
`uRenderStructural.pas` defensively) — kept even after the switch to `-O2`,
since that rewrite is harmless and remains a second line of defense if a
future change ever moves the project back to `-O3`. If a shape or size
looks subtly wrong after editing a function that computes a radius, cap, or
similar bound this way, suspect this before suspecting the surrounding
logic — regardless of which optimization level is configured at the time.

## Supported diagram types

`Logical`/`Class`/`Object`/`Package`/`Collaboration`/`Timing` (one shared
renderer — real EA package diagrams routinely mix in classes, actors, use
cases, boundaries and text labels alongside packages, and Collaboration/
Timing diagrams use mostly the same element vocabulary), `Use Case`/
`UseCase`, `Sequence`, `Activity`/`Statechart`/`StateMachine`/
`InteractionOverview`/`Analysis`/`UMLDiagram` (one shared renderer — all
reuse the same StateNode/Decision/ControlFlow vocabulary; `Analysis` is
EA's informal business-process variant of an Activity diagram),
`Component`/`Deployment`/`CompositeStructure` (one shared renderer —
Component/Port/ProvidedInterface/RequiredInterface occur in all three),
and `Custom`.

`Custom` is EA's generic free-form diagram type — a blank canvas not tied to
any single UML notation, used for things like "Getting Started" pages,
dashboards, or arbitrary documentation layouts, and in practice mixing in
element types from every other diagram type at once. It is rendered by
routing each element to the most specific renderer already available for
its `Object_Type` (structural shapes first, then activity/state shapes),
falling back to a plain named box for anything with no dedicated shape
(`Requirement`, `GUIElement`, `Screen`, `Task`, `Defect`, `Risk`, ...) —
broad coverage rather than a pixel-accurate notation, since `Custom` has no
fixed notation to be accurate to in the first place. A `GUIElement` (or
`Class`) box that spatially encloses several other elements is detected as a
frame (win32 dialog mockups, composite-structure classifiers) and drawn
hollow instead of filled — otherwise its own fill, drawn after its contents
in EA's own object order, would paint over everything inside it.

Any other `Diagram_Type` not listed above shows a "not supported yet"
message instead of a blank window (none exist in the reference EAExample.qea
file this plugin was verified against).

## Known limitations

- Stereotype icons (the small `<<interface>>`/`<<enumeration>>`/
  `<<exception>>` markers drawn next to a stereotype label) are small
  generic vector glyphs, not EA's own bitmap icon set — no icon assets are
  bundled for those built-in stereotypes. This is separate from an
  element's own custom image (`ImageID` in `ObjectStyle`), which — when
  present — is read from the file and rendered as the actual picture (see
  above); EA's built-in stereotype icon set itself is not bundled or
  reproduced, only what a diagram already carries as embedded image data.
- EAShapeScript preview icons (a stereotype's `_image` attribute, see
  above) cover only a bounded core command set: `moveto`/`lineto`/
  `startpath`/`endpath`/`fillandstrokepath`/`strokepath`/`fillpath`/
  `rectangle`/`ellipse`/`setlinestyle`. No expressions/variables beyond
  simple numeric or string literals, no conditionals, no color assignments
  (`setfillcolor`/`setpencolor`/`GetUserFillColor()` etc. are recognized
  but ignored — everything draws in black-on-white), no text, no
  `drawnativeshape()` (the element's own default shape underneath a
  script's overlay), no sub-shapes. Unrecognized statements are skipped
  rather than aborting the whole preview, so a partially-understood script
  still shows whatever of it this subset can draw. The icon is a
  standalone preview (auto-scaled to fit, using only the script's own
  `main` block, falling back to the first block found) — it is not wired
  into actual connector endpoints or element outlines, since that would
  need real diagram context (endpoint position/direction, element size)
  this attribute-line preview doesn't have.
- Sequence diagrams: EA does not reliably store per-message Y coordinates,
  so messages are stacked top-to-bottom in `SeqNo` order with a fixed row
  height rather than at their original diagram positions. No activation
  bars, combined fragments (alt/loop/opt), or nesting depth.
- Activity/statechart diagrams: pseudostate kind (initial/final/history) is
  guessed from the element name (EA's schema has no reliable subtype column
  for `StateNode`) — an unnamed pseudostate falls back to a plain filled
  circle. Swimlanes (`ActivityPartition`) are drawn as plain hollow boxes,
  not full lanes spanning the diagram.
- **External element labels (pin/port boxes, pseudostates, decisions) use
  EA's own `LBL=CX=..:CY=..:OX=..:OY=..` position hint** from
  `t_diagramobjects.ObjectStyle` when present, instead of guessing a
  placement. This field gives the label's intended size and its offset from
  the object's own top-left corner — used by `DrawSmallPinBox`
  (ActionPin/ObjectNode/Port/EntryPoint/ExitPoint/Event/CentralBufferNode/
  ActivityParameter in Activity diagrams), `DrawPortBox` (Port in Component/
  Deployment/CompositeStructure diagrams), `DrawPseudoState` (StateNode —
  previously never shown at all for named pseudostates like "Choice" or a
  custom-named Initial/Final), and `DrawDecisionDiamond` (Decision/
  MergeNode — previously never shown at all; verified against EAExample.qea
  that most Decision nodes with a name were silently invisible before this).
  The actual text is still measured and wrapped with our own font metrics
  (`DT_CALCRECT`), not EA's raw `CX`/`CY`, since those can differ slightly
  from our rendering; if that measured size exceeds the hinted size, the
  label grows away from the object rather than into it. Without a `LBL=`
  field, each of these falls back to its previous placement (centered
  overflow for pin boxes, below the shape for decisions/pseudostates).
  `OX=0:OY=0` together is treated as "no offset set" (falls back too) rather
  than "label at the object's own top-left corner" — that exact pair covers
  34% of all `LBL=` occurrences in EAExample.qea, far too common to be a
  deliberate per-object position; taking it literally put a `StateNode`
  named "Final" directly on top of its own (tiny) circle in
  "ServerStateMachine", reported by the user as "the position looks wrong",
  while objects with a real nonzero offset in the same diagram (e.g.
  "Initial") rendered correctly.
- **Pin/port labels are now drawn in a separate, final pass, after every
  object AND every connector.** `DrawSmallPinBox` (Activity/Statechart) was
  split into `DrawSmallPinBoxBody` (the small rectangle itself, drawn inline
  in the normal object loop as before) and `DrawSmallPinBoxLabel` (the name
  text, using the `LBL=OX=.../OY=..` offset described above). The label call
  moved out of the per-object loop into a third loop that runs after all
  objects and all connectors in both `TActivityDiagramRenderer.Render` and
  `TCustomDiagramRenderer.Render` (both route `ActionPin`/`ObjectNode`/`Port`/
  `EntryPoint`/`ExitPoint`/`Event`/`CentralBufferNode` through the same
  `DrawActivityObject`/`IsSmallPinBoxType`). Reason: EA's own `OX`/`OY` offset
  routinely places a pin's label outside the pin's own (often 16x16 pixel)
  box and into a *neighboring, non-containing* sibling object's area —
  `IsCompositeFrame` does not apply here since there is no containment
  relationship, just spatial adjacency. `t_diagramobjects` sequence order is
  otherwise arbitrary with respect to such neighbors, so when the neighbor
  happened to be sequenced after the pin, its opaque fill silently painted
  over the pin's label. Reported by the user against "Invoice Payment" in
  EAExample.qea ("stimmt hier der obere bereich mit dem verdeckten text?"):
  both `ActionPin` objects are named "Invoice" (`Object_ID` 513/514, between
  "Send Invoice" and "Customer Payment"), and each one's `LBL=`-positioned
  label fell almost entirely inside the *other* Action box, which is
  sequenced right after it. Drawing all pin labels last, regardless of
  sequence order, fixes this independent of which neighbor happens to be
  listed first — verified against "Invoice Payment" (fixed) and regression-
  tested against "Fibonacci With Link Event" and "SubMachine" (both from an
  earlier session's `DrawSmallPinBox` fix, still correct, unaffected by the
  reordering since their labels don't collide with a neighbor to begin with).
- **Composite/frame elements, across every diagram type.** A `Class` or
  `GUIElement` that spatially encloses at least 2 other diagram elements —
  a composite-structure "owning classifier", a win32Dialog/win32GroupBox
  mockup frame, a Class used as a statechart swimlane, a nested class inside
  an ordinary class diagram — is detected heuristically (`IsCompositeFrame`
  in `uRenderer.pas`, shared by every diagram-type renderer) and drawn
  hollow (name at top, no fill) instead of through the normal opaque
  class-box rendering. Without this, since such a frame is typically
  sequenced *after* the elements it contains, its fill would silently paint
  over everything inside it — this was found and fixed incrementally across
  the session for one diagram type at a time (Component/Deployment/
  CompositeStructure, then Custom, then Activity/Statechart, then plain
  Class/Logical/Object/Package/Collaboration/Timing and Use Case diagrams)
  after a user question ("why does a fix only apply to one diagram type?")
  prompted an audit that found the same gap in the two remaining renderers.
  `Activity`/`StructuredActivityNode` elements get the same treatment for a
  composite Activity nesting its own sub-actions, and so does `UseCase` —
  found afterwards against "Manage Titles State" (a Statechart diagram whose
  outermost element is, unusually, a `UseCase` ellipse enclosing 8 nested
  State/StateNode/Text elements): the huge white-filled ellipse, sequenced
  last, erased the entire embedded state machine underneath it, leaving only
  the *connectors* visible (drawn in a separate, later pass) — floating
  transition labels and arrows with no boxes, reported as "die Texte sind
  falsch platziert" / "der Text unten wird vom Oval überlappt". Drawing it
  hollow instead loses the ellipse notation (falls back to the same
  rectangular `DrawBoundary` used for the other composite-frame types), the
  same accepted shape-vs-visibility tradeoff as everywhere else this
  heuristic applies. `Package` gets the same treatment too — arguably EA's
  *most* natural use of this whole pattern, since a Package drawn as a
  visual frame around its full contents (a namespace grouping enclosing
  every class/actor/note in the diagram) is an entirely ordinary,
  intentional layout, not an edge case — found against "Domain Model" in
  "Listening Domain" (EAExample.qea): `DrawPackage`'s default white fill,
  sequenced last, erased nearly every class/actor/note box in the diagram,
  leaving only a handful of stray connector-line fragments and two
  aggregation diamonds visible ("was fehlt hier? text? bild?"). After that
  seventh individual find, audited every remaining `CreateObjectFillBrush`
  user (opaque default fill) across the structural/activity/class renderers
  for plausible container semantics instead of waiting for an eighth
  one-off report — added `Component`/`Node`/`Device`/`ExecutionEnvironment`
  (deployment/component diagrams routinely nest artifacts and parts inside
  these — standard UML deployment notation, verified against real,
  previously-broken diagrams in EAExample.qea: "Application Taxonomy",
  "Deployment Overview") and `Artifact`/`Action` (rarer as containers, but
  the same bug if it occurs, and the existing contained-element threshold
  already guards against false positives on their much more common
  non-container use). Left out as implausible: Port/ProvidedInterface/
  RequiredInterface (boundary markers), Decision/ObjectNode/ActionPin/Event
  (pin/diamond glyphs), InteractionOccurrence, Actor/Note/Lollipop — none of
  these are ever used as a spatial container in practice. Node/Device/
  ExecutionEnvironment all use the same
  3D-box look (no distinct icon set). Assembly/delegate connectors show no
  ball-and-socket notation beyond the Provided/RequiredInterface shapes
  themselves.
- **The contained-element threshold for the composite-frame heuristic above
  is 2, not the 3 it started at.** Raised concern initially (2 felt like a
  weaker signal, more likely to catch an ordinary box that just happens to
  spatially overlap one or two nearby elements by coincidence) and left
  alone rather than lowered speculatively — until "Custom Drawing Styles"
  itself (EAExample.qea) turned up a real instance: its own "Rotation" group
  box has exactly 2 children and stayed opaque, silently hiding them, with
  no way to fix it without touching the shared threshold ("müsste hier bei
  Rotation was sein?"). Rather than special-case that one box, every
  diagram in EAExample.qea was scanned for any object with exactly 2
  spatially-contained children before committing to the change: 39 hits,
  every single one a clearly-named container holding two clearly-named
  children (`Strategy` → `Goal`/`Objective`, `Headquarters` → `IT
  Resources`/`Human Resources`, `Environment` → `Noise`/`Weather`, and so
  on) — zero coincidental overlaps in the entire file. On that evidence, 2
  is justified; 1 was not tested and is deliberately not assumed to be safe
  by extension.
- **`LoopNode`/`ConditionalNode`/`SequenceNode` added to the same
  contained-element threshold, and `ActivityPartition`/`StateMachine`/
  `ExpansionRegion` given their own *unconditional* hollow rendering,
  reachable from every renderer, not only the Activity-family one.** Found
  against the "UML" showcase diagram (`Diagram_Type='Use Case'`,
  EAExample.qea) — "ExpansionNode2" (an `ActionPin`-like pin element on the
  border of an `ExpansionRegion`) had its EA-positioned label cut off on
  both sides ("hier ist der text rechts und links der box verdeckt"): the
  region box itself (`ExpansionRegion1`, sequenced right after both of its
  pins) painted over their overflowing labels. `ExpansionRegion` couldn't
  just join the threshold-based list above — an `ExpansionNode` sits
  *straddling* the region's border by UML notation, half in/half out, so it
  never satisfies the `RectContainsRect` containment check the threshold
  relies on, no matter how the threshold is tuned. Investigating the same
  diagram surfaced two more, structurally identical problems: `LoopNode1`
  spatially contains 3 real children (`Action1`, two `ActionPin`s) that
  qualify for the existing threshold once `LoopNode` is added to its type
  list — same fix as the `Action`/`Activity`/`StructuredActivityNode`
  entries above, just a missing type. And `StateMachine1`/`ExpansionRegion1`
  themselves were being drawn *opaque*, even though `ActivityPartition` and
  `StateMachine` were already hardcoded hollow — but only inside
  `DrawActivityObject` (`uRenderActivity.pas`), the Activity/Statechart
  renderer. This "UML" diagram is `Diagram_Type='Use Case'`, which routes
  through `DrawDiagramObject` (`uRenderClass.pas`) instead, a completely
  different fallback that never knew about either type. Scanning the whole
  file for `ActivityPartition`/`StateMachine` objects embedded in
  non-Activity diagram types turned up real, currently-broken instances well
  beyond this one showcase diagram: several "Logical"-type Balanced-Scorecard
  templates ("Strategic Plan", "Style 2", and four more with the same
  `Financial Perspective`/`Customer Perspective`/`Internal Process`/`Learning
  and Growth` swimlane pattern) use `ActivityPartition` as their swimlane
  frame with 1-12 spatially contained child boxes each, all previously
  rendered as opaque white rectangles hiding their own content. Fix: added a
  hollow-drawing branch straight in `DrawDiagramObject` for
  `ActivityPartition`/`StateMachine`/`ExpansionRegion` — unlike the
  threshold-gated list, every instance of these three is hollow
  unconditionally (0 or 1 children included), since UML notation makes them
  a frame regardless of how many children happen to be visible on a given
  diagram, not just when 2 or more happen to spatially fit inside. Verified
  against "UML" (ExpansionNode1/2 labels fully visible, LoopNode1/
  ConditionalNode1/StateMachine1 all hollow with visible contents) and
  regression-tested against "Strategic Plan"/"Style 2" (Balanced-Scorecard
  swimlanes now show their Strategy/Process boxes), "Invoice Payment", and
  "Custom Drawing Styles" (both unaffected, still correct).
- Win32 dialog mockups (`GUIElement` with stereotypes like `win32Button`,
  `win32Edit`, `win32CheckBox`, ...) render as plain labeled boxes, not
  actual button/edit/checkbox shapes — EA's dialog canvas can pack these
  very densely (10-20 world units tall). `DrawClassBox` gives `GUIElement`
  names a dedicated single-line layout (`DT_SINGLELINE`, no `DT_WORDBREAK`):
  a wrapped multi-word caption ("&Limit recording frame depth:") in one of
  these thin boxes used to break onto 2-3 lines that blew straight through
  the tiny box height into the *next row* of controls — a cascading,
  practically unreadable "text soup" across several stacked controls at
  once (reported against "IDD_DEBUGMARKUP"/"IDD_APP_EDIT_DLG" in
  EAExample.qea: "die sehen sehr falsch aus"). Kept single-line, the same
  caption instead only overflows sideways into its immediate horizontal
  neighbor (e.g. a caption running into the edit/spin control right next to
  it) — still crowded, but confined to one row instead of bleeding across
  three. That remaining sideways overlap between tightly packed same-row
  controls is inherent to a generic box-with-label fallback with no layout
  awareness of neighboring controls, and is not planned to get a dedicated
  win32-control renderer. Only `GUIElement` uses this single-line rule —
  and only when the name is actually short enough to plausibly be a control
  caption: some `GUIElement`/`win32StaticText` objects hold a whole
  free-floating documentation paragraph instead (100+ characters, same
  stereotype as an ordinary short label, no way to tell them apart except
  by length), and forcing those onto one line sent them shooting far off
  past the diagram's edge instead of wrapping (reported against "Customer
  Login Dialog" in EAExample.qea: "hier liegt Text hinter der Box").
  `DrawClassBox` measures the name's natural single-line width before
  deciding: only under roughly 3× the box's own width does it stay
  single-line (a real short caption overflows a tight box by maybe 2-3×
  even at full size — a 100+ character paragraph by 6-7× or more, a wide
  enough gap to tell the two apart reliably); above that threshold it falls
  back to the normal word-wrap-then-shrink-then-overflow path used
  everywhere else. Every other object type keeps the normal word-wrapping
  name layout unconditionally.
- Connector name labels (association role names, activity/state guard
  conditions) are placed at the line's midpoint. When two connectors join
  the same pair of elements in opposite directions and both carry a name
  (e.g. a statechart toggle transition like "Pause"/"Resume"), the two
  labels would otherwise land exactly on top of each other; this is
  detected (same endpoint pair, reversed direction, both named) and the
  labels are offset symmetrically to either side of the line instead. EA's
  own `t_diagramlinks.Geometry` field does carry a proprietary per-connector
  label-offset hint (`LMT=CX=..:CY=..:OX=..:OY=..;`), but its exact
  coordinate semantics are unconfirmed, so it is intentionally not parsed —
  the offset used here is a self-contained perpendicular displacement
  instead. Only this specific reciprocal-pair collision is handled; three or
  more connectors sharing an endpoint pair, or unrelated crossing
  connectors, can still overlap. Association role names (`SourceRole`/
  `DestRole` in `t_connector` — e.g. "billing"/"shipping"/"DMZ", distinct
  from the connector's own midpoint `Name`) are shown near each end,
  slightly further out than the multiplicity label at that same end so the
  two don't overlap when both are set — these were loaded from the database
  since early in the project but never actually drawn until this was
  noticed and fixed.
- Connectors keep their own EA color and line weight (`LineColor`/`IsBold`
  in `t_connector`) instead of always rendering as a plain thin black line;
  this applies to the line itself and to arrowheads/aggregation diamonds
  drawn at its ends, so a recolored connector doesn't end up with a
  mismatched black tip.
- **EA's "Custom Draw Style"** (per-element right-click "Enable Custom Draw
  Style", or diagram-level "Custom Style" — see EAExample.qea's own
  "Custom Drawing Styles" demo diagram) is honored for `Class`/`GUIElement`
  objects: shape override (rectangle/rounded/ellipse/diamond/triangle),
  fill opacity (0–100 %, via `AlphaBlend`, applies to the fill only — the
  border stays fully opaque so a 0 %-opacity box doesn't vanish entirely),
  border line style (solid/dash/dot/dash-dot/dash-dot-dot/none), a stacked-
  copies effect (offset toward NE/SE/SW/NW), and name text justification
  (any combination of left/center/right × top/middle/bottom). This data
  is *not* stored on the object itself — it lives in
  `t_diagram.StyleEx` as `OPTIONS_<DUID>=Key=Val:...;` entries, cross-
  referenced via the object's own `DUID=` in `t_diagramobjects.ObjectStyle`
  (`TEaCustomDrawStyle`/`ParseCustomDrawStyle` in `uEaModel.pas`; applied in
  `DrawClassBox`, `uRenderClass.pas`). Before this, none of these five
  properties were read at all — a diagram built specifically to showcase
  them rendered as identical plain black-on-white rectangles, reported as
  "die sehen sehr falsch aus" against EAExample.qea's own demo diagram.
  **Not implemented: text rotation** (`SIRot`) — meaningfully more complex
  (LOGFONT escapement + repositioning math) than the other five properties
  and not what was reported as wrong, deliberately deferred. The demo
  diagram's own "Rotation" group box only shows its two children's plain
  (unrotated) names as a result — labeled, but not actually rotated.
- **Connector routing through `t_diagramlinks.Path` waypoints is drawn as a
  smooth curve, not straight line segments** (`BuildSmoothCurve`,
  `DrawGenericConnector` in `uRenderer.pas`) whenever a connector has at
  least one real intermediate waypoint (self-transitions; manually routed
  transitions between two different states) — a uniform Catmull-Rom spline
  through the clipped start point, every stored waypoint, and the clipped
  end point, converted to cubic Bezier segments and drawn with `PolyBezier`.
  The two-point case (no stored waypoints at all — the vast majority of
  ordinary associations/dependencies) is untouched and still a plain
  straight line via `MoveToEx`/`LineTo`, since a "curve" through only two
  points is a straight line anyway. Before this, waypoints were connected
  with hard straight segments, which turned a manually-routed transition
  with a waypoint far from a direct line into a sharp, angular "tent" or
  triangle instead of the rounded loop/arc EA itself displays — reported
  against "CD Paused" → "CD Stopped" in "StateMachine" (EAExample.qea):
  "die Linie geht mit Sicherheit nicht so rechtwinkelig so hoch raus". The
  arrowhead/diamond at each end is angled along the curve's own tangent at
  that point (the nearest Bezier control point), not the old raw-waypoint
  direction, so it still lines up visually with the curve it's attached to.
  That tangent point is deliberately exempted from the older "degenerate
  anchor" fallback (see below) that substitutes a completely unrelated point
  — a real waypoint sitting exactly on a box edge is a genuine degenerate
  case worth a fallback, but a Catmull-Rom control point sitting close to
  its own endpoint (typically ~1/6 of the way back toward the previous
  point, by construction) is normal and not degenerate; applying that
  fallback to it substituted the connector's far *start* point as the
  arrow's reference, producing a visibly wrong angle unrelated to the
  actual curve (reported against "Send Invoice" in
  "InterruptibleActivityRegion", EAExample.qea: "der winkel des pfeils
  stimmt nicht mehr").
  **Deliberately routed right-angle paths are exempted from smoothing**
  (`HasOrthogonalWaypoint`): when two consecutive points in the full path
  (clipped anchor → waypoints → clipped anchor) share the same X or Y,
  that's treated as EA's own signal for an intentional elbow — e.g. several
  org-chart connectors dropping straight down from each manager box onto a
  shared horizontal "trunk" before turning down into the CEO box. A
  Catmull-Rom spline through a sharp ~90° turn where the adjacent segments
  differ wildly in length (a short box-to-trunk drop next to a long shared
  trunk run) overshoots badly, and several such connectors converging on
  nearly the same point compounded into large self-overlapping loops that
  swallowed neighboring boxes — reported against "The Org Chart"
  (EAExample.qea): "die geschwungenen linien sind generell zu geschwungen
  und überlappen teilweise boxen oder sich selbst". Self-transitions are
  exempt from this exemption (`Link.Conn.StartObjectID = EndObjectID`
  always still curves): their waypoint loop is *also* right-angled by
  construction, but its segments stay short and comparable in length (a
  compact loop back to the same box), which is exactly what keeps
  Catmull-Rom from overshooting there in the first place — segment-length
  mismatch, not right-angledness per se, is what actually causes the
  overshoot, and same-object heuristically stands in for "segments are
  comparably short" without measuring each one.
- Connector line/arrowhead/diamond endpoints are computed by ray-casting from
  the connected box's own center through the nearest stored waypoint,
  clipped to that box's boundary — this can choose the "wrong" edge (a side
  edge instead of top/bottom, say) when the stored path is sparse and the
  box is much wider than it is tall, which happens in EA's "Grid Style" for
  Requirements-traceability diagrams (zero-gap boxes laid out in a row, fed
  by a shared vertical routing trunk — see "Requirements Grid" in
  EAExample.qea). Reconstructing the geometrically "correct" edge would need
  decoding EA's proprietary per-endpoint `EDGE=`/`SX=`/`SY=` fields in
  `t_diagramlinks.Geometry`, whose exact semantics are unconfirmed — the
  same reason `LMT=` label offsets are deliberately left unparsed (see
  `ReciprocalLabelSign` above), and decoding them wrong risks moving
  connectors in *other*, already-correctly-rendered diagrams. Instead,
  `DrawGenericConnector` defensively excludes every *other* diagram object's
  rect from the clip region before drawing a connector's line, arrowhead, or
  labels (`ExcludeClipRect`, one call per unrelated object) — regardless of
  which edge the endpoint calculation lands on, a connector can no longer
  visually paint over an unrelated box's text. A composite-frame container
  (see `IsCompositeFrame` below) that spatially contains one of the two
  connected endpoints is exempted from this exclusion, since a connector
  between two children of the same hollow container is expected to cross
  visually through that container's rect. Before this, a diamond landing
  exactly on a shared, zero-gap border with a neighboring box merged
  visually with that neighbor's leading letter (reported as "REQ018" reading
  as "PEQ018" — "ich sehe Verbindungslinien, aber die sind aufgrund einer
  falschen Positionierung nicht richtig zu sehen"). The underlying
  edge-selection is still not "correct" in the EA sense, only no longer
  destructive to unrelated content.
- **Multiplicity ("1", "0..*") and role-name labels at the same connector
  end no longer overlap each other.** Both are positioned by nudging outward
  from the connector's endpoint along the same ray, at two different
  distances — but that distance used to be a fixed, un-scaled pixel count
  (14 and 28) while the label font itself scales with zoom
  (`CreateScaledFont`). At higher zoom the text grew past the gap between
  the two anchor points, so multiplicity and role text landed on top of
  each other whenever a connector carried both (reported against "Domain
  Model" in EAExample.qea: "1"/"item", "0..*"/"order"). Both distances now
  scale with `View.Zoom`, and the gap between them is derived from the
  actual font's measured line height (`GetTextMetricsW`) instead of a
  guessed second constant — guarantees clearance regardless of zoom or
  font metrics, rather than "usually enough at the zoom levels tested."
- **"Fit" (zoom-to-fit) accounts for labels that are drawn outside their
  owning object's own rect**, not just the raw object rectangles themselves
  (`ComputeContentBounds`, `uRenderer.pas`): small "pin"-style elements
  (Event/ActionPin/ObjectNode/Port/PseudoState, drawn via `DrawSmallPinBox`/
  `DrawPortBox`/`DrawPseudoState`) can carry an EA-supplied `LBL=` position
  hint that places their name well outside their own tiny (often
  20-30-world-unit) box; connector names (guard conditions like "Initial
  Estimate Not Accepted", role labels) are drawn at the line's midpoint,
  entirely outside any object's rect. Neither was considered when computing
  the bounding box "Fit" zooms/pans to, so a label near the diagram's edge
  got cut off by the viewport — reported against "Car Repair" ("Customer
  Arrives"/"Customer Leaves Shop" clipped on the left/right) and "Repair
  Car" ("die texte sind falsch plaziert" / "wird oben und an den Seiten
  massiv abgeschnitten", both in EAExample.qea). Both cases now grow the
  content bounds with a fixed safety margin around the label's expected
  position — the real `LBL=` offset for pin-style labels, a rough midpoint-
  of-both-endpoints estimate for connector names, since neither the actual
  measured text width nor the exact line-midpoint geometry (which needs a
  DC/font and the full path-waypoint logic from `DrawGenericConnector`,
  respectively) is available at bounds-computation time. Errs toward
  zooming out slightly more than strictly necessary rather than risking a
  clipped label again.
- Actor/Lollipop/Socket name labels (drawn below/beside the small figure or
  icon, not inside a box) size their label area from a fixed "2.2 lines"
  estimate rather than measuring the actual text — this undercounts when a
  name has an *embedded* line break (EA stores some Actor names as
  `"Actor 2\r\n- global appearance"`, which `DrawTextW` treats as a forced
  break the same as `DT_WORDBREAK` does for a long line) and the second
  line is itself long enough to wrap again, needing 3 lines where only
  2.2's worth of height was reserved — the bottom line then gets silently
  clipped, since none of these three draw calls pass `DT_NOCLIP` (reported
  against "Actor 2 - global appearance" in "Element Appearance",
  EAExample.qea). `DrawActor`/`DrawLollipop` (`uRenderClass.pas`) and
  `DrawSocket` (`uRenderStructural.pas`) now measure the label's real
  height with `DT_CALCRECT` first instead of guessing.
- **A diagram that loaded successfully but has zero elements now shows an
  explicit "Dieses Diagramm enthält keine Elemente." message instead of a
  blank white pane.** `PaintEmptyMessage` (`uViewer.pas`) already covered
  three other "nothing to show" cases (no diagrams in the file at all, load
  failed, diagram type not supported) but not this fourth one — a real
  diagram, of a supported type, that simply has no objects in
  `t_diagramobjects`. This happens in practice: EAExample.qea's own "Phase
  D" through "Phase G" (a TOGAF ADM template) were never filled in and have
  0 objects each, and the previous plain-white rendering looked
  indistinguishable from something being broken ("was fehlt hier?") rather
  than the source diagram genuinely being empty.
- **Timing diagrams now render their state segments, not just an empty
  lifeline row.** The claim that state-value data isn't accessible (an
  earlier version of this bullet) was wrong — it's not in `PDATA`/child
  objects/connectors as assumed, but in `t_object.RunState`
  (`"@VAR;Variable=<state>;Value=<position>;Op==;[Event=<name>;]
  [DConst=..;][TConst=..;]@ENDVAR;"` per segment, descending by `Value`),
  never loaded or parsed until now. Found against "Token 4"/"Pen & Paper
  Analysis 7 Customers" (`Diagram_Type='Timing'`): "soll die box leer sein?"
  — no, `RunState` has three segments ("Customer calls in" →
  "Service Customer" → "Customer hangs up"). `TEaObject.RunState` is now
  loaded (`QueryObject`, `uEaModel.pas`, with a fallback query for schema
  variants missing the column, same defensive pattern as the `Classifier`
  fallback already there) and rendered by a new `DrawTimeLine`
  (`uRenderClass.pas`, replacing the old `DrawBoundary`-only call): each
  distinct state name gets its own horizontal row (assigned in order of
  first chronological appearance — this alone reproduces the classic
  "square wave" look for the common 2-state active/inactive case, e.g.
  "Life Line Of The Tasks"), with vertical step lines at transitions and the
  triggering `Event=` name labeled above the step, matching "Timing
  Diagram"'s `User`/`ACSystem`/`UserAccepted` lifelines (`Idle`→`WaitCard`→
  `WaitAccess`→`Idle`, with `Code`/`OK` event labels) exactly. `DConst=`/
  `TConst=` (duration/time constraints) are parsed but not drawn — a
  deliberate v1 scope cut, consistent with the EAShapeScript subset
  elsewhere in this list. Two data-driven fixes needed along the way: (1)
  the chronologically LAST segment has no "next" entry to bound its own
  width, and its own `Value` is by construction the data's maximum — using
  that maximum directly as the axis's right edge collapsed the last segment
  to zero width and made it vanish (found exactly in "Pen & Paper": "Customer
  hangs up" was invisible, only the middle segment showed); fixed by
  extending the axis past the true max by one average inter-segment step, a
  reasonable heuristic since EA's `RunState` doesn't otherwise encode "where
  the timeline ends". (2) A segment whose own duration is a near-zero sliver
  next to its neighbor (e.g. "Customer calls in" → "Service Customer" 0.0017
  time units apart) draws too narrow a line to hold a centered label without
  hiding it entirely — the *line* stays exactly as narrow as the real data
  (no invented minimum duration), but its *label* gets a minimum on-screen
  width and switches to left-aligned so the state name stays legible even
  when its segment is visually a hairline. Verified against "Token 4"
  (all three segments now visible and correctly positioned) and regression-
  checked "Timing Diagram" (`User`/`ACSystem`/`UserAccepted`, 3-4 states
  with named events, unaffected) and "Life Line Of The Tasks" (seven
  2-state active/inactive waveforms, unaffected).
- `.eap`/`.eapx` (MS Jet/ACE) and `.feap` (Firebird) are out of scope — both
  need a completely different access technology (ODBC/DAO/COM automation,
  and a Firebird client library, respectively) than the SQLite path this
  plugin uses for `.qea`/`.qeax`.
- Not implemented: clicking a `$diagram:` navigation link (either the
  `$diagram://GUID` or the plain `$diagram:Name` form) jumps nowhere —
  NavigationCell dashboard tiles show their `Alias` text (EA's own
  human-readable link caption) when set, otherwise a neutral placeholder;
  tagged values / custom element properties are not shown; a diagram's own
  UML frame (the pentagon title box some diagrams have) is not drawn, only
  its content.
- Text that doesn't fit the box it's placed in (long class names, attribute/
  operation lines, note/free-text bodies) is no longer silently clipped. A
  class/requirement/feature-style name that's too long for its box (common
  for Requirement/Feature/Defect elements, which often use a full sentence
  as their Name) first wraps, then shrinks its font down to 60% before
  falling back to overflowing past the box's own border — shrinking first
  avoids the shrunk-or-not text visually colliding with whatever diagram
  element sits directly below or beside it. The attribute/operation
  compartment is shrunk the same way as a block (all lines together, down
  to 60%) when they don't collectively fit the box height; the stereotype
  line (`<<...>>`) and a single unbreakable long word (no spaces to wrap
  on, e.g. a camelCase instance name in a narrow Collaboration-diagram box)
  get the same shrink-first treatment down to 40%. The stereotype line is
  the one exception to "overflow past the border as a last resort" —
  a very narrow box (50 world units, e.g. the `<<stakeholder>>` boxes in
  "Stereotyping", EAExample.qea) can still be too narrow for `<<stakeholder>>`
  even at the shrink floor, and letting a short decorative marker spill its
  `<`/`>` characters past the box edge reads as visibly broken rather than
  as "more information than fits" (reported as "die größer/kleinerzeichen
  gehen über den rand hinaus"). Once shrinking bottoms out and it still
  doesn't fit, the stereotype line switches to `DT_END_ELLIPSIS`
  (`<<stakehol…`) confined to the box width instead of overflowing —
  purely a fallback for the case that was already overflowing before, wider
  boxes are unaffected. Notes and free-text
  ("Text") elements shrink the same way (down to 60%) before overflowing —
  avoids a note or text block growing into a tightly neighboring element
  below it. `DrawRoundedActionBox` (Action/Activity/State boxes — the single
  most common element type in Activity/Statechart diagrams) had NEITHER
  shrinking NOR an overflow fallback at all until this was found: an
  unbreakable name with no spaces (an assignment-expression-style label like
  "next=first+second", common in algorithm-demo Activity diagrams — `=` and
  `+` aren't DT_WORDBREAK break points) stayed one line wider than the box,
  and the previous hard clip to the box's own rect cut off *both* ends
  symmetrically around the centered text (reported against "Fibonacci With
  Link Event" in EAExample.qea: "next=first+second" showed as
  "ext=first+secon" — both the leading "n" and trailing "d" were gone, not
  just one end). Now shrinks down to 60% first, same as the other element
  types, before falling back to the same overflow-past-the-border escape
  valve. Only once shrinking still isn't enough do attribute/operation
  lines, notes, and free text fall back to overflowing past the box's own
  border (measuring their
  needed height first and growing outward). This can still make text
  visually spill into the space below or beside it when the source diagram
  sized the box too small for its own content even at minimum font size —
  a faithful reproduction of an oversized note/label, not a bug. If
  overflowing text appears to run off the bottom of the visible panel,
  that's normal scrolling behaviour (pan down) rather than truncation —
  verified by panning to confirm the full text is present.

- **ProvidedInterface/RequiredInterface (lollipop/socket) labels — three
  compounding fixes, found via "Regional Connections" (Port labels
  right-aligned into a neighbor) and "Physical Components" (`Ports`/lollipop-
  socket labels asking "was sind diese kleinen Rechtecke und Kreis am Rand
  der großen Boxen?").**
  1. **Missing dispatch outside their "home" renderer.** `Port`/
     `ProvidedInterface`/`RequiredInterface`/`Component`/`Node`/`Device`/
     `ExecutionEnvironment`/`Artifact` had dedicated drawing routines
     (`DrawPortBox`/`DrawLollipop`/`DrawSocket`/`DrawComponentBox`/...) only
     reachable via the Component/Deployment/CompositeStructure renderer (or
     Custom/Activity's own frame-check paths) — the *generic*
     `DrawDiagramObject` fallback used by Class/Object/Package/Collaboration/
     Timing/Use-Case diagrams didn't know about any of them and fell through
     to the plain `DrawClassBox` box instead. A `Port` drawn this way ignores
     its own `LBL=OX/OY` offset entirely and centers the (often 3-line
     wrapped) name directly on the tiny port position — which, if that
     position sits at the edge of a neighboring box, reads as wrong
     alignment ("Head Office Lime Street London" appearing to run right-to-
     left, reported as "der Text scheint rechtsbündig statt linksbündig", in
     "Regional Connections", EAExample.qea). Fixed by routing
     `IsStructuralElementType` object types through `DrawStructuralObject`
     from `DrawDiagramObject` too (implementation-only back-reference from
     `uRenderClass.pas` to `uRenderStructural.pas` — legal in FPC/Delphi
     since only one direction is in each unit's `interface` section).
  2. **Lollipop/socket labels ignored `LBL=` entirely.** Unlike
     `DrawPortBox`/`DrawSmallPinBoxLabel`/`DrawPseudoState`, `DrawLollipop`/
     `DrawSocket` always centered the name a fixed distance below the icon,
     never consulting EA's own `LBL=OX=..:OY=..` position hint — even though
     ProvidedInterface/RequiredInterface objects carry that field just as
     often as Port/ActionPin (`"Account Details"` in "Components",
     EAExample.qea: `LBL=OX=37:OY=23`, previously landing mid-way through the
     owning "Account" component's own name instead of beside it). Fixed by
     giving both the same `ComputeLabelRect`/`GrowLabelRectAwayFrom` handling
     already established for pins/ports.
  3. **The "no `LBL=`" fallback's fixed 2px gap.** Even with (2) fixed, a
     `ProvidedInterface`/`RequiredInterface` with `OX=OY=0` (EA's "no offset
     set" case) still fell back to a *fixed 2-pixel* gap below the icon —
     unrelated to zoom or to the size of whatever the icon sits on the edge
     of. Since these pins routinely sit on the border of a much taller
     Component box, 2px isn't remotely enough to clear the component's own
     name compartment: "File Pickup"/"File Drop" in "Physical Components"
     rendered fused into "Stock Optima"/"Pickman Organizer" ("Stock
     OptimaFile Pickup"). Widened to a zoom-scaled 15 world units — enough to
     clear typical component-box heights in EAExample.qea without
     reintroducing a *different* collision with unrelated content below (a
     50-unit version was tried first and tested clean on "Physical
     Components", but pushed the label into the row of `<<interface>>` boxes
     underneath in "Services (Interfaces)"; 15 is deliberately a compromise,
     not a value that guarantees clearing every component-box height in
     every file — the underlying limitation is that this fallback still has
     no way to know the geometry of whatever box the pin happens to sit on).
  Split each of `DrawLollipop`/`DrawSocket` into an Icon-only proc (drawn
  inline, first pass) and a Label-only proc, deferred to the same
  "after every object AND every connector" final pass already established
  for `DrawSmallPinBoxLabel` (`IsDeferredLabelType`/`DrawDeferredLabel` in
  `uRenderStructural.pas`) — added to all four renderers that can reach a
  Port/ProvidedInterface/RequiredInterface (`TStructuralDiagramRenderer`,
  `TCustomDiagramRenderer`, `TClassDiagramRenderer`, `TUseCaseDiagramRenderer`,
  `TActivityDiagramRenderer`'s fallback path) so a later-sequenced neighbor
  can never paint over the label regardless of which renderer reaches it.
  **Debugging note for next time:** most of the investigation time on this
  went into a false lead — a `build/` output directory
  (`build/<arch>/release/*.ppu`) that `lazbuild` silently kept reusing across
  several edit-rebuild-test cycles despite the source changing, making
  screen-position debug markers appear to "not render" when the real cause
  was a stale binary. `rm -rf build` before rebuilding resolved it; see the
  root `CLAUDE.md`, which already documents an analogous lazbuild caching
  gap for units outside the project folder — this is the same class of
  problem for the in-project `build/` cache too. Separately, a large chunk of
  *that* investigation also chased a red herring caused by not passing the
  probe's `nozoom` argument — its default `LC_SETPERCENT` override zooms
  ~150% past the diagram's own "Fit" level, which silently scrolls low
  content out of the captured screenshot and looks identical to genuinely
  missing/invisible content. Debugging label-position bugs near a diagram's
  bottom edge should always start with a `nozoom` capture.

- **`DrawPackage` had no shrink/overflow fallback at all** — the one drawing
  routine in `uRenderClass.pas` that still hard-clipped its name to the box
  with a plain `IntersectClipRect`, no `DT_CALCRECT` pre-measurement, no
  font-shrink loop, no `DT_NOCLIP` last resort (every other box-with-a-name
  routine — `DrawClassBox`, `DrawNote`, `DrawRoundedActionBox` — already got
  this earlier in the session). Invisible for an ordinary Package box with
  headroom to spare, but EA's "Roadmap" notation (`t_object.Object_Type =
  'Package'`, no distinguishing stereotype — the diagram itself carries a
  `$help://roadmap_diagram.htm` marker) draws each phase as a flat, wide bar
  only ~14-28 world units tall with a full bold name centered in it. Found
  against "Online Bookstore Architecture" (EAExample.qea): "Logistics
  Rationalization"/"Warehouse Optimization"/etc. all appeared with their text
  sliced through top and bottom (only the lower half of ascenders / upper
  half of descenders visible), reported as "prüfe hier die schriftgröße/
  boxgröße". Fixed with the same measure-then-shrink-then-overflow sequence
  as everywhere else (60% floor, then grow past the border symmetrically —
  horizontally too, since a long name can also exceed the bar's width once
  shrinking bottoms out). Verified against "Online Bookstore Architecture"
  (all five roadmap bars now fully legible) and regression-tested against
  "Package Dependencies" (ordinary Package boxes with normal headroom,
  unaffected).

- **`ObjectNode`/`ActivityParameter`/`ActionPin`/... had the same missing-
  dispatch gap as `Port`/`ProvidedInterface`/`RequiredInterface`, just one
  renderer over.** Their dedicated small-pin-box drawing (`DrawSmallPinBox*`,
  `uRenderActivity.pas`) is only reachable via the Activity/Statechart
  renderer or Custom's own routing — the Component/Deployment/
  CompositeStructure renderer (`TStructuralDiagramRenderer`) falls through to
  the same generic `DrawDiagramObject`/`DrawClassBox` fallback as before for
  anything it doesn't recognize, and `ActivityParameter` isn't one of the
  types it recognizes. Found against "IO" (EAExample.qea,
  `Diagram_Type='CompositeStructure'`): "iNoOfBytes"/"sFileName"
  (`ActivityParameter`, positioned just above the "readPort"/"createPort"
  Activity boxes) were centered directly on their own tiny pin position
  instead of using their `LBL=OX/OY` offset, landing invisibly behind the
  Activity box's own name — "der text über readport und createport ist stark
  verdeckt". Fixed the same way as the Port/ProvidedInterface case: added an
  `IsSmallPinBoxType` branch to `DrawDiagramObject` (drawing only the pin's
  body immediately, `DrawSmallPinBoxBody` — newly exported from
  `uRenderActivity.pas`'s interface, it was previously implementation-only
  and inaccessible from outside), and added the matching deferred label pass
  (`DrawSmallPinBoxLabel`, already established) to
  `TClassDiagramRenderer`/`TUseCaseDiagramRenderer`/
  `TStructuralDiagramRenderer` alongside the `IsDeferredLabelType` pass they
  already had. `uRenderClass.pas`'s implementation now also imports
  `uRenderActivity` (same implementation-only back-reference pattern as its
  existing `uRenderStructural` import — legal because the circularity only
  runs one direction in each unit's own `interface` section).
  Initially still overlapped the Activity box's own name after this fix
  (EA's `LBL=OX=-23:OY=16` offset places the label just inside the box, not
  clear of it) — resolved as a side effect of the next fix below, once
  `readPort`/`createPort` themselves switched from the generic `DrawClassBox`
  to the real `DrawRoundedActionBox` (different internal name layout gave the
  pin label and the box name enough room to land on separate lines instead of
  literally on top of each other).
- **The same missing-dispatch gap a third time, and more broadly than either
  case above: `Action`/`Activity`/`StructuredActivityNode`/`State`/
  `StateNode`/`Decision`/`MergeNode`/`Synchronization`/
  `InteractionOccurrence` were ALSO only drawn correctly inside
  `DrawActivityObject`.** Outside an Activity/Statechart or Custom diagram,
  every one of these fell back to a plain rectangular `DrawClassBox` instead
  of its real UML shape (rounded action box, pseudostate circle, decision
  diamond, sync bar, interaction ref-box) — a *shape* bug, not just a text-
  position one, and the broadest of the three "eigene Zeichenroutine nur im
  Activity-Renderer" gaps found this session. Visible across the entire "UML"
  showcase diagram (`Diagram_Type='Use Case'`, already the source of several
  earlier finds in this file): `*ActivityInitial`/`*Final`/`*History`
  rendered as plain boxes instead of the initial/final/history pseudostate
  glyphs, `State1`/`State2` as square-cornered rectangles instead of rounded
  state boxes. Fixed with the same pattern as the other two gaps, but simpler
  — these five shapes are drawn immediately, no deferred label pass needed —
  so `DrawDiagramObject` just hands matching types straight to
  `DrawActivityObject` itself via a new exported predicate
  (`IsActivityOnlyType`, `uRenderActivity.pas`), deliberately excluding every
  type already handled by an earlier branch in the same function (the pin
  types and `ActivityPartition`/`StateMachine`/`ExpansionRegion`) so the two
  dispatch layers never double up. Verified against "IO" (Action boxes now
  correctly rounded, pin-label collision from the previous fix gone as a side
  effect) and "UML" (pseudostates/states/decision diamonds all showing their
  real UML notation now); regression-tested against "Fibonacci With Link
  Event" and "Services (Interfaces)" (both unaffected, still correct).
  **Pattern across all three finds this session:** a drawing routine
  exported only for its "home" diagram-type renderer (plus whichever other
  renderer happened to need it before) is not automatically safe elsewhere —
  the type can appear in *any* diagram type EA allows it in, and each prior
  fix in this family was found by a fresh bug report landing on exactly the
  renderer combination not yet covered, not by systematic review. Worth
  treating as a standing question when the *next* one-off type gets a
  dedicated drawing routine: is it reachable from every renderer that could
  plausibly contain it, or only from the one it was designed for?

- **Sequence diagrams (`uRenderSequence.pas`) drew EVERY diagram object as a
  lifeline — header box plus a dashed line running the full height of the
  diagram — regardless of its actual `Object_Type`.** No type check existed
  at all: a real lifeline participant (`Object`/`Class`/`Part`/`Actor`/
  `State`/`Interaction`, and — confirmed against EAExample.qea — the older
  `Sequence` object type, used by named participants like "UserInterface"/
  "DataSource"/"DataControl" that span the same near-full diagram height as
  an ordinary `Object`) got exactly the same treatment as a free-floating
  `Text` note, a combined-fragment `InteractionFragment` frame, an
  `InteractionOccurrence` reference box, or a `MessageEndpoint` gate marker —
  all of which got a spurious header box and a full-length dashed lifeline
  they have no business having. Affects **every** Sequence diagram containing
  any of these — found against "doReadUSB" (EAExample.qea): a `Text` object
  literally named "Text" (a floating annotation, unrelated position) rendered
  as if it were its own lifeline participant column — "bist du sicher, das
  'Text' samt box dort richtig platziert ist? das betrifft alle diagramme
  dieser art". Fixed with a new `IsLifelineType` predicate gating the
  existing header+lifeline code, and a second pass for everything else that
  routes each non-lifeline type to its own proper drawing routine at its
  *own* rect instead of the synthesized fixed-height header rect: `Text` →
  `DrawTextLabel`, `Note` → `DrawNote`, `InteractionOccurrence` →
  `DrawActivityObject` (renders the correct `ref` interaction-use box, not a
  generic rectangle — noticeably better than before even beyond just fixing
  the placement), `InteractionFragment` → `DrawBoundary` (hollow combined-
  fragment frame). `MessageEndpoint` deliberately gets no box at all — it's a
  small anonymous gate marker on a message line, and inventing a visible
  shape for it would be less faithful than showing nothing. The message-
  baseline calculation (`MsgBaselineWorld`, decides where the first message
  arrow row starts) is now also computed only over real lifeline objects, for
  the same reason a `Text` object's arbitrary rect shouldn't distort it.
  `uRenderSequence.pas` now imports `uRenderClass` and `uRenderActivity`
  (plain one-directional additions — neither of those units references
  `uRenderSequence`, so no circularity concerns here unlike the
  `uRenderClass`⇄`uRenderStructural`/`uRenderActivity` back-references
  documented above). Verified against "doReadUSB" (the frame border and
  `ref` box for "setupUSB" now render correctly, "Text" no longer shows a
  phantom lifeline) and regression-tested against "Login" and "Add To
  Shopping Cart" (both ordinary Sequence diagrams with only real lifelines
  and floating text notes, unaffected).

- **A `$help://...htm` link and its own explanatory caption are two
  independent `Text` objects that can genuinely overlap in the source
  data — no rendering-order fix can help, since both draw transparent text
  with no fill.** EA generates a recurring documentation block in several
  diagram types: a short `$help://code_generation_-_activity_dia.htm`-style
  link object, paired with a separate, longer "For more information on Code
  Generation from ... refer to:" caption object placed just below it. Found
  against "IO" (EAExample.qea, `Diagram_Type='CompositeStructure'`): of the
  three such pairs in that one diagram, two rendered fine (link cleanly above
  its caption, non-overlapping rects), but the third
  (`$help://code_state_machine.htm`) had its text run straight through the
  caption below it — "diese url ist immer falsch positioniert. optimiere
  das." Root cause, confirmed in the data itself: that caption's `Note` field
  carries three extra trailing blank lines the other two don't, giving it a
  taller stored rect whose top edge lands almost exactly where the link
  object already sits — a genuine near-collision between two *independently*
  positioned objects, not a case of our renderer misplacing either one
  (each draws exactly at its own EA-supplied rect, top-anchored, matching
  the two pairs that render correctly with the same code). Since neither
  object has a background fill, deferring *which one draws last* changes
  nothing — transparent text drawn over transparent text still interleaves
  both, regardless of order. Fixed by giving link objects specifically
  (`$help://...` in `Name`, no `Note` — reliably identifies this narrow
  category, distinct from ordinary free text) an opaque white backdrop sized
  to their own measured text (`DrawHelpLinkBackdrop`, `uRenderClass.pas`),
  drawn in a final pass after every other object
  (`IsHelpLinkText`/`TStructuralDiagramRenderer.Render`, currently only
  wired for the Component/Deployment/CompositeStructure renderer where this
  was found — the same recurring EA template block also appears embedded in
  Activity/Statechart/Sequence-type diagrams directly, per the file naming
  of the other two link variants in this same set, so the identical fix may
  eventually be needed in `TActivityDiagramRenderer`/`TSequenceDiagramRenderer`
  too if a future report lands there — deliberately not preemptively wired
  into every renderer without a concrete case to verify against). This keeps
  the link fully legible no matter what happens to be sequenced after it,
  while the caption text is simply hidden behind the opaque patch wherever
  it would have collided — an acceptable trade given the caption is a
  duplicated, non-essential explanation and the link is the one piece of
  potentially-actionable text. Verified against "IO" (the state-machine link
  is now cleanly readable; the two already-correct pairs remain unaffected).

- **Free-floating ProvidedInterface/RequiredInterface icons (no `LBL=`
  offset) drift away from their adjacent Port instead of touching it.**
  Found against "Prophet Account Suite Interfaces" (`Diagram_Type=
  'CompositeStructure'`): "hier sind kreise und halbkreise frei in der luft.
  fehlt hier was?" — four lollipop/socket icons sat visibly detached from
  the small Port squares they belong to. Root cause: when EA has no `LBL=`
  hint for one of these objects, its stored `t_diagramobjects` rect is often
  noticeably wider/taller than the actual icon (EA reserves room for a label
  that, in the free-floating case, EA itself draws elsewhere) — and
  `DrawLollipopIcon`/`DrawSocketIcon` simply center the icon within whatever
  rect they're given, so the icon ends up centered in the middle of that
  oversized rect instead of hugging the edge nearest the real Port. Fixed
  with a new `AdjustFreeInterfaceRect` (`uRenderer.pas`), called only for the
  no-`LBL=` branch in `TStructuralDiagramRenderer.Render`: it searches
  `Data.Objects` for a `Port` whose rect overlaps this one by at least 60% of
  the smaller object's height and sits within a small horizontal gap, then
  shrinks/repositions the icon's rect to the edge touching that Port, using
  `Min()` of the two objects' heights as the icon size. The 60%-overlap
  threshold and the `Min()`-based sizing were both needed after an initial,
  looser version (any positive Y-overlap) wrongly matched a vertically-
  stacked, connector-wired Port/Interface pair and used the tall Interface's
  own full height as the icon size — producing an oversized lollipop where a
  small one belonged (caught via own regression screenshot, not user-
  reported, before shipping). Verified against "Prophet Account Suite
  Interfaces" (all four free-floating icons now sit flush against their
  ports; the one connector-wired lollipop correctly stayed at its normal
  small size) and regression-tested against "Components" (Account
  Details/Payment lollipop+socket pair, `LBL=`-positioned, unaffected since
  they take the other branch) and "Physical Components" (File Pickup/File
  Drop lollipop+socket pair, no regression).

- **"Style 3" (Balanced-Scorecard-style Logical diagram, `SM_Family`/
  `SM_Process`/`SM_OrganizationCapital`/... stereotypes) — z-order checked
  and confirmed correct, not a bug.** "überprüfe die z-order" — the four
  `<<SM_Family>>` category boxes visibly cut into the "Organization
  Capital"/"Information Capital"/"Human Capital" bars behind them, leaving
  only fragments like "nization C"/"ormation"/"HumanCa" readable in the gaps
  between boxes. Checked against `t_diagramobjects.Sequence` (the field this
  renderer already sorts `Data.Objects` by via `ORDER BY Sequence ASC` in
  `uEaModel.pas`, and draws strictly in that ascending order with no
  resorting or reversal anywhere in `TClassDiagramRenderer.Render` —
  confirmed by reading the loop): the three capital-bar `Class` objects have
  `Sequence` 14/18/34, the four `SM_Family` objects have `Sequence` 35–38 —
  i.e. EA's own stored draw order puts the family boxes strictly *after*,
  and therefore on top of, the bars they visually cut into. Our renderer is
  faithfully reproducing exactly that stored order; the covered/fragmented
  bar text is a property of the source diagram data, not of our z-order
  handling. Separately, this is likely also a case of the "EAShapeScript
  preview icons"/"Stereotype icons" limitation documented above:
  `SM_Family` is a stereotype from an external Balanced-Scorecard MDG
  Technology with no matching row in this file's own `t_stereotypes` table,
  so its true custom shape (probably a short header-only glyph that leaves
  the bar below visible, rather than a full-height opaque box) isn't
  available to reproduce — we fall back to a generic opaque `Class` box for
  it, which is wider/taller than the real shape and therefore hides more
  than genuine EA (with that Technology installed) would. No code change;
  documented here since it's a case worth being able to point back to rather
  than re-investigating from scratch.

- **Connector endpoint clipping ran the connected line straight through the
  child object's own box when one end spatially contains the other**, found
  against "External" (`Diagram_Type='Logical'`): "sind die linienstart-/
  endpunkte richtig?" — two composition connectors from "Weather"/"Noise" to
  their enclosing "Environment" frame each rendered as a diamond sitting at
  Environment's border, connected by a line that ran straight through the
  *entire* Weather/Noise box before reaching it, instead of a short stub.
  Root cause in `ClipToRectBoundary` (`uRenderer.pas`): it always casts a ray
  from a rect's own center toward a target point and returns where that ray
  crosses the rect's own boundary — correct for two non-overlapping objects
  (both endpoints land *between* the two centers), but when one rect (here,
  "Environment") fully contains the other ("Weather"), the two centers'
  connecting line has the child's center sitting *inside* the parent, so the
  "child center → parent center" ray and the "parent center → child center"
  ray point in exactly opposite directions — the child's own anchor lands on
  the side of the child *away* from where the parent's boundary anchor ends
  up, and the drawn segment between them has to cross the entire child box
  to connect the two. Fixed by detecting containment (`RectContainsRect`,
  already used for composite-frame detection) and, in that case only,
  re-clipping the *inner* object's anchor toward the *already-computed*
  outer anchor point instead of the outer object's raw center — both anchors
  then land on the same side of the child, producing a short stub from the
  child's near edge to the parent's boundary. Deliberately scoped to
  connectors with no stored `Path` (`t_diagramlinks.Path` empty) — an
  explicit EA-stored route already expresses real routing intent and isn't
  touched. Verified against "External" (both Weather→Environment and
  Noise→Environment now draw as short stubs, diamonds unchanged in
  position). Checked the whole file for other same-diagram containing/
  contained connector pairs (`t_connector` join against `t_diagramobjects`
  with a rect-containment test, excluding self-referencing sequence-message
  rows) — "External" is the only diagram in EAExample.qea with this pattern,
  so no separate regression case existed to test against; re-verified
  "Prophet Account Suite Interfaces" (non-contained connectors, unaffected)
  as a sanity check on the general connector path instead.

- **`Part` was missing from the composite-frame whitelist (`IsCompositeFrame`,
  `uRenderer.pas`)** — the same "opaque container sequenced last erases its
  own contents" bug already fixed for `Class`/`GUIElement`/`Package`/
  `Component`/`Node`/`Device`/`ExecutionEnvironment`/`Artifact`/`Action`/
  `Activity`/`StructuredActivityNode`/`LoopNode`/`ConditionalNode`/
  `SequenceNode`/`UseCase`, now found for an eighth-ish container type.
  Found against "Seat Control IBV" (`Diagram_Type='Activity'`, a SysML/
  AUTOSAR-style Internal Block Diagram): "fehlt hier was in der box? text/
  symbole?" — the outer "SeatControl" `Part` box (stereotype "AUTOSAR
  Component") showed only its own header, a note, a few small Port/pin
  squares, and a tangle of connector lines running to nowhere; every Action/
  Object box normally nested inside it (`AUTOSARRunable1`, `SeatRunnable`,
  `init runnable`, two `AUTOSAR Inter Runnable Variable` boxes, etc.) was
  invisible. Root cause: `Part` (EA's object type for a component/block
  instance, common in Composite-Structure/SysML-flavored diagrams — used
  across ~50 diagrams in EAExample.qea, mostly as plain non-container
  leaves) wasn't in `IsCompositeFrame`'s type whitelist, so this particular
  `Part` — sequenced last (17 of 17) and spatially containing 13 of the
  other 16 objects — fell through to the generic, opaque-filled
  `DrawClassBox` fallback and erased everything drawn before it; only the
  later, separate connector-line pass and the small Port/pin bodies (which
  double as connector endpoints) remained visible on top. Fixed by adding
  `Part` to the same whitelist used by the other container types — same
  `ContainedCount >= 2` threshold already in place, so ordinary non-
  container `Part` instances (the common case) are completely unaffected.
  Verified against "Seat Control IBV" (all previously-missing boxes now
  render); regression-checked "Composite Structure-Properties" (`BookStock`/
  `records` `Part` boxes, non-container, unaffected).

- **`StyleFlag` silently failed to find a Key=Value pair whenever the key had
  leading whitespace (`"...; LBL=..."` instead of `"...;LBL=..."`) — and a
  new `HDN=1` ("this label is hidden") sub-flag inside `LBL=` was never
  respected at all**, found against "Regions and Locations Simple"
  (`Diagram_Type='Logical'`): "hier sehen viele text fehlplaziert aus" — text
  fragments like "ondon"/"rk"/"eet" (word-wrapped and clipped mid-word)
  cascaded out of nearly every small `Port` box into neighboring class boxes
  and connector labels. Two compounding root causes, both in
  `t_diagramobjects.ObjectStyle`: (1) every one of this diagram's 23 `Port`
  objects has `LBL=...HDN=1...` — EA's own "user hid this element's name
  label" flag, previously never parsed anywhere in the codebase, so the
  (very long, e.g. "Chicago  William Place Data Center") port names were
  drawn centered over their own 15×15-world-unit boxes even though EA itself
  keeps them invisible (their role is already covered by the diagram's
  actual connector-name labels, e.g. "Dark Fiber 10 GB"/"private cloud" —
  which display correctly and are unrelated to this). (2) `StyleFlag`
  (`uRenderer.pas`) split `ObjectStyle` on `;` and compared each token's key
  via exact match (`IsType`, case-insensitive only) — but EA writes some of
  these ";"-separated tokens with a leading space (`"OY=0; LBL=CX=..."`)
  and some without (`"...;LBL=CX=..."`), inconsistently, even within the
  *same* diagram's Port list (15 of 23 ports here had the leading space, 8
  didn't). `" LBL" <> "LBL"` under an exact match meant `StyleFlag`
  couldn't find the `LBL=` block at all for those 15 — silently affecting
  every existing `LBL=`-based feature (position AND now hidden-check), not
  just this new one, for any object whose style happened to carry that
  space. Fixed `StyleFlag` to trim leading/trailing whitespace off each
  parsed key before comparing (root-cause fix, benefits every existing
  `LBL=` consumer retroactively) and added `IsLabelHidden` (checks for
  `HDN=1` inside the `LBL=` block), wired into the three places that draw an
  object's own name via `ComputeLabelRect`/`LBL=` and had confirmed `HDN=1`
  data in EAExample.qea: `DrawSmallPinBoxLabel` (uRenderActivity.pas, Port/
  ActionPin outside Component/Deployment/CompositeStructure diagrams),
  `DrawPortBox` (uRenderStructural.pas, Port — the actual code path this
  diagram's ports use, since `Port` is checked by `IsStructuralElementType`
  before `IsSmallPinBoxType` in the shared dispatch), `DrawPseudoState`
  (StateNode) and `DrawDecisionDiamond` (Decision). ProvidedInterface/
  RequiredInterface labels (`DrawLollipopLabel`/`DrawSocketLabel`) and
  `DrawPackage` were checked via a file-wide SQL scan for `LBL=...HDN=1...`
  and don't occur with this flag in EAExample.qea, so left untouched.
  Verified against "Regions and Locations Simple" (all 23 port labels now
  correctly hidden, only the legitimate connector-name labels remain);
  regression-checked "IO" (`iNoOfBytes`/`readPort` etc. — `LBL=`-positioned
  pin labels without `HDN=1`, still drawn and correctly placed).

- **Self-loop connectors could balloon into a wildly overshooting curve
  instead of a small clean loop**, found against "Link 16" in "System
  Exchange Matrix" (`Diagram_Type='Logical'`): "prüfe die position des
  pfeils und fixe übergreifend" — a compact rectangular self-association
  loop rendered as a large, distorted arc bulging far past the box before
  looping back. Root cause in `BuildSmoothCurve`'s point chain
  (`DrawGenericConnector`, `uRenderer.pas`): the clipped start/end anchors
  (`P1`/`P2`) for a manually-routed path very often land within a pixel or two
  of their neighboring raw waypoint (the stored path's first/last point is
  usually already right on the box's own edge) — feeding the Catmull-Rom
  "missing neighbor → use outermost existing point" boundary rule a
  near-duplicate point instead of a genuinely distinct one skews the tangent
  estimate at exactly the segment that matters most for a loop's overall
  shape. Self-loops (`Start_Object_ID = End_Object_ID`) are hit hardest
  because *both* ends of the path have this near-duplicate characteristic at
  once (unlike an ordinary routed connector between two different objects,
  where usually only one end — if any — is this close). Fixed with a new
  `DedupPoints` (2px tolerance, same as `HasOrthogonalWaypoint`'s), applied
  to the point chain right before the curve-vs-straight decision and before
  `BuildSmoothCurve` itself — collapsing the near-duplicate boundary point so
  the tangent rule falls back to the actual next distinct point instead of a
  copy of itself. Only affects the curve math; the un-smoothed straight-
  segment path and the arrowhead-direction fallbacks still use the original,
  non-deduplicated point chain (unrelated to this bug). Verified against
  "System Exchange Matrix" (the loop is now a small, correctly-proportioned
  curve close to the box, matching the second self-loop already on the same
  box). Regression-checked "StateMachine" (`CD Playing`/`CD Paused` — their
  own "barrel" look turned out to already be two separate, correctly-shaped
  loops on opposite box edges stacked together, not a curve-overshoot bug,
  so unaffected by this fix) and "The Org Chart" (the orthogonal collector-
  branch exception this same code area already handles, still un-curved).

- **`Artifact` boxes had no shrink-then-overflow mechanism at all** — found
  against "icons" (`Diagram_Type='Custom'`): "die schrift scheint zu groß
  hier" — tiny (~25×16 world-unit) icon-tile artifacts named "browse.png"/
  "menu-icon.png" showed only a truncated fragment ("brov"/"men") of the
  name, hard-clipped at a fixed font size. `DrawArtifactBox`
  (`uRenderStructural.pas`) had simply never received the shrink-to-60%-
  then-overflow-past-the-box treatment already standard for `DrawClassBox`/
  `DrawPackage`/`DrawNote`/`DrawRoundedActionBox` — it drew the name straight
  into the box at a fixed size with a hard `IntersectClipRect`. Added the
  same pattern, with one adjustment specific to this element: a filename has
  no spaces, so `DT_WORDBREAK` can never wrap it onto a second line no matter
  how narrow the box is — the first shrink attempt only checked the
  (always-single-line) measured *height* against available height, which
  the string already satisfied without shrinking at all, leaving the real
  problem (excess *width*) completely unaddressed. Fixed by shrinking (and,
  if still too wide even at the 60% floor, overflowing past the box) based
  on *either* dimension exceeding its available space, not just height.
  Verified against "icons" (`browse.png`/`menu-icon.png` now fully legible,
  overflowing cleanly below/past the tiny icon tile rather than being
  clipped mid-word); regression-checked "Document Artifact" (`Business Case
  - Extended Selling`, a normal-sized Artifact box with a space-containing
  name that already wrapped and fit correctly, unaffected).

- **A connector role-name label pushed outward to the left or above a box
  could still grow back INTO that box's own content**, found against
  "FaceImageType" in "PersonalizedMessage Request" (`Diagram_Type='Logical'`):
  "der text am rand der box überlappt" — the `DestRole` label "FacialImage"
  sat right at FaceImageType's left edge, overlapping the wrapped second
  line ("[0..88]") of its own long attribute. Root cause in
  `DrawGenericConnector` (`uRenderer.pas`): `OutwardPoint` correctly pushes
  the label's *anchor* away from the box along the box-center-to-connection-
  point direction, but the actual `TextOutW` call always draws growing
  right/down from that anchor regardless of which way "away" pointed — for
  a box positioned to the *right* of its neighbor (so "away" points left),
  the anchor sits just left of the box edge, but the text itself, growing
  rightward from there, walks straight back across the edge into the box's
  own compartment text. Same issue for a box below its neighbor ("away"
  points up). Fixed with a new `DirectedLabelOrigin`: along whichever axis
  dominates the outward direction (matching `OutwardPoint`'s own axis
  choice), if that direction points left/up, shift the text origin back by
  its own measured width/height so the rendered block ends at the anchor
  instead of starting there — the perpendicular axis is deliberately left
  untouched (an earlier version also centered that axis, which pulled a
  vertically-approaching connector's label onto its own line, e.g. "item" in
  "Domain Model" started rendering with the connector line through the
  middle of the word instead of beside it, a new problem that didn't exist
  before; not touching the perpendicular axis avoids this and matches
  exactly what `TextOutW` already did in every case that wasn't broken).
  Applied to all four `SourceCard`/`DestCard`/`SourceRole`/`DestRole` call
  sites, since they all share the identical anchor-then-draw pattern.
  Verified against "PersonalizedMessage Request" (`FacialImage` now sits
  cleanly in the gap between the two boxes); regression-checked "Domain
  Model" (the original `CardDist`/`RoleDist` scaling fix's own example —
  `item`/`status`/`trans` etc. still positioned the same as before, no new
  overlap introduced). A separate, pre-existing issue also visible in
  "Domain Model" — several *different* connectors' role/multiplicity labels
  converging near the same box corner and overlapping *each other* (not a
  box's own content) — is a distinct root cause (unrelated connectors
  landing at similar angles) and out of scope here; not touched.

- **EA's own outer sequence-diagram frame (the "sd `<Name>`" box around the
  whole diagram) was mistaken for a real lifeline participant**, found
  against "Start Vehicle Black Box" (`Diagram_Type='Sequence'`): "drive oder
  start vehicle scheint mit der verbindungslinie falsch plaziert" — the
  message "StartVehicle" overlapped the lifeline header name "driver", and a
  phantom dashed line ran down the middle of the whole diagram. Root cause:
  EA stores this frame as an ordinary `t_diagramobjects` row of
  `Object_Type='Interaction'`, with a rect spanning (and spatially
  containing) the entire diagram — `IsLifelineType` already treats
  `Interaction` as a normal lifeline type (needed for a genuine `Interaction`
  participant elsewhere), so this frame got its own diagram-wide header box
  *and* a centered dashed lifeline down the full height, and — worse — its
  header's artificially high bottom edge (computed from its own huge rect)
  dragged `MsgBaselineWorld` (where the first message row starts) up above
  the real lifelines' own header boxes, so messages rendered *inside* them.
  Fixed with a new `IsInteractionFrame` (`uRenderSequence.pas`): the same
  `RectContainsRect`-based containment check already used for
  `IsCompositeFrame` elsewhere (`>= 2` contained objects), applied to
  `Interaction`-typed objects specifically — verified this pattern holds
  for *every* `Interaction`-typed object across all six Sequence diagrams
  with one in EAExample.qea (each one spatially contains its diagram's
  entire remaining content, without exception). Excluded from
  `MsgBaselineWorld` and from the header+lifeline drawing loop; still drawn
  via the existing hollow `DrawBoundary` (name + frame, no fake lifeline)
  so it doesn't just vanish. Verified against "Start Vehicle Black Box"
  (`StartVehicle` message now sits cleanly below both header boxes, no
  phantom centered dashed line); regression-checked "Maintain Audio Player"
  (`listener`/`deviceInContext` lifelines with several `ref` boxes,
  unaffected).

- **"TankPI" (SysML Internal Block Diagram, `Diagram_Type='CompositeStructure'`,
  `t_diagram.StyleEx` carries `MDGDgm=SysML1.4::InternalBlock`) — two of six
  port labels genuinely overlap their `Part` box's own name, investigated
  and left as a known limitation rather than patched speculatively.**
  "sind qout, cout, qin, cin richtig plaziert?" — `cIn` overlaps
  `piContinuous`, `tSensor` overlaps `tank`; `cOut`/`tActuator`/`qOut`/`qIn`
  on the same two boxes render cleanly. Measured directly against the data:
  the two colliding ports (`cIn`, `tSensor`) sit only 8–9 world units below
  their own box's top edge (10–16% of the box height) — closer to the top
  than any other port on either box — while `DrawClassBox`'s title row
  (single bold line, `BASE_FONT_HEIGHT` plus padding) needs roughly that
  same vertical space regardless of which port happens to be nearby. The
  right-side ports on the same boxes (`tActuator`, `cOut`) sit at a near-
  identical vertical position but don't collide, purely because their
  `LBL=` offset lands further from the box's left edge than the (short,
  centered) name text reaches — confirming this is specifically a left-edge-
  near-top case, not a general "name too tall" one. Since `Part` here
  represents a SysML block instance and the diagram is explicitly tagged as
  a SysML Internal Block Diagram, this is very likely the same class of gap
  as `SM_Family` in "Style 3" above: EA's real rendering for this MDG
  Technology almost certainly uses a SysML-specific, more compact Part-box
  layout (not the generic UML `Class` compartment box `DrawClassBox` falls
  back to for the ~379 attribute-less `Part` instances in this file) that
  reserves less vertical space at the top specifically so port labels this
  close to the corner don't collide — information not available to us
  without that MDG Technology's own shape definitions. No code change;
  documented here so a future report against the same pattern doesn't need
  re-deriving this from scratch.

Connector routing (via `t_diagramlinks.Path`) and interface lollipop
notation (via `t_diagramobjects.ObjectStyle` "Lollipop=1") are implemented
and verified against real EA-generated files.
