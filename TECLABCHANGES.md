# Teclab Changes to Tktable

## Memory leak fixes

### `generic/tkTableCmds.c` — `Table_CurselectionCmd`

**Problem:** `TableCellSortObj` takes a list object and, when the list is non-empty,
returns a *new* sorted copy — leaving the original `objPtr` (refcount 0) with no owner
and no free path. The leak fired on every `curselection` call that returned at least one
selected cell.

**Fix:** Bracket the `TableCellSortObj` call with `Tcl_IncrRefCount`/`Tcl_DecrRefCount`
so that the original object is released in all cases:

- `TableCellSortObj` returns a new sorted object → `DecrRefCount` frees the original.
- `TableCellSortObj` returns the same object (empty list) → `Tcl_SetObjResult` raises
  the refcount to 2 before `DecrRefCount` drops it back to 1; the object survives.
- `TableCellSortObj` returns `NULL` (parse error) → `DecrRefCount` frees the original;
  the interp error set by `Tcl_ListObjGetElements` is preserved.

---

### `generic/tkTableWin.c` — `Table_WindowCmd`, `WIN_NAMES` case

Two memory leaks were fixed in the same switch case.

**Problem 1 — early return before allocation was correct:**
`Tcl_NewObj()` was called at the very top of the `WIN_NAMES` case, before the argument
count check. A call with the wrong number of arguments caused an immediate
`return TCL_ERROR`, leaking the freshly allocated object.

**Fix:** Moved `Tcl_NewObj()` (and `Tcl_IncrRefCount`) to after the argument check so
that the object is only created when we are certain the command will proceed.

**Problem 2 — same `TableCellSortObj` ownership pattern as above:**
Identical to the `Table_CurselectionCmd` issue; fired on every `window names` call that
matched at least one embedded window.

**Fix:** Same `Tcl_IncrRefCount`/`Tcl_DecrRefCount` bracket around `TableCellSortObj`.
