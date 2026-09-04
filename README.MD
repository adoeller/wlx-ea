# wlx_enterprise-architect

Total Commander Lister-Plugin (WLX) for viewing UML class diagrams stored in
Sparx Enterprise Architect's SQLite-based project files (`.qea` / `.qeax`),
in both the F3 quick view and the Ctrl+Q panel.

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

and verified against real EA-generated files.
