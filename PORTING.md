# Porting notes: CollaborativeFC on FreeCAD 1.1

This repository is a **compatibility port** of
[OpenCollaborationPlatform/CollaborativeFC](https://github.com/OpenCollaborationPlatform/CollaborativeFC)
(upstream commit `2c46b48`, 2022-07-02) to the current stable FreeCAD.

Nothing was redesigned: the user interface, the document observer layer, the
state machines, the WAMP/OCP protocol, the peer model and the permission
handling are the originals. The changes below only make that code run on the
Qt/Python versions shipped with FreeCAD 1.1.

## Target environment

| Item | Value |
| --- | --- |
| FreeCAD | 1.1.4 (2026-09-28), Windows x86_64 |
| Qt | 6.8.3 |
| Python bindings | PySide6 6.8.3, reached through FreeCAD's `PySide` module |
| Python | 3.11.14 (FreeCAD's bundled interpreter) |
| OCP node | `ocp` 0.1.9 (`ocp-windows-amd64` 0.1.9) |

## Compatibility changes

### Qt 5 / PySide2 → Qt 6 / PySide6

| Change | Why |
| --- | --- |
| `from PySide2 import ...` → `from PySide import ...` in `InitGui.py`, `Collaboration.py` users, `Interface/*.py`, `Manager/*.py`, `OCP/*.py`, `Utils/*.py`, `Resources/resources.py` | FreeCAD 1.1 has no PySide2. `PySide` is FreeCAD's own compatibility module and resolves to PySide6. Using it keeps one import style for all modules. |
| Vendored `Qasync` updated 0.22.0 → 0.28.0 (same package name and API, BSD license and headers kept) | qasync 0.22 is not compatible with Python 3.11's asyncio: the async event loop raised `RuntimeError: Cannot enter into task ... while another task ... is being executed`, which aborted node startup and WAMP callbacks. 0.28 supports current Python and PySide6. The add-on's used API (`QEventLoop(..., already_running=True)`, `asyncSlot`) is unchanged. |
| `Qasync` Qt binding preference ordered PySide6-first | The add-on imports `PySide`; if a PyQt5 happened to be installed, qasync would previously have picked it and mixed two Qt bindings in one process. |
| `QSocketNotifier.Read/Write` → `QSocketNotifier.Type.Read/Write` (qasync unix backend) | Qt 6 requires the scoped enum. |
| Mixin constructors `AsyncSlotWidget.__init__` and `StateMachineProcessWidget.__init__` take `parent=None` | PySide6's `QObject.__init__`/`QWidget.__init__` call the next class in the MRO without arguments. `AsyncSlotPromoter`, `DocEdit` and `DocWidget` failed with `TypeError: ... missing 1 required positional argument: 'parent'`. The explicit call still passes the real parent. |

`Qt5`-era enum shorthand (`Qt.KeepAspectRatio`, `Qt.DisplayRole`, …) still
works in PySide6's compatibility mode and was left untouched. The Qt5-format
compiled resource file (`Resources/resources.py`) also still registers under
Qt 6; it only needed its import changed.

### Python 3.11 / Windows

| Change | Why |
| --- | --- |
| `signal.SIGUSR1` handler registered only if available | `signal.SIGUSR1` does not exist on Windows, so importing `Collaboration.py` raised `AttributeError` and the add-on never loaded. On POSIX the debug handler behaves exactly as before. |

### FreeCAD 1.1 specifics

| Change | Why |
| --- | --- |
| `OCP/Node.py` no longer overwrites `ocp.node_path` with a hardcoded developer path (`/home/stefan/Projects/OCP/OCP`) | Upstream had a leftover development line that made node startup impossible on every machine except the author's. The `ocp` package path is used again, as intended. |
| `Interface/Installer.py` runs FreeCAD's own `python.exe` and installs into `AdditionalPythonPackages/py311` | FreeCAD 1.1 adds that directory to `sys.path` and the user site-packages is *not* importable. The original `python` call could install into an unrelated interpreter, leaving FreeCAD unable to import the packages. |
| `Manager/Manager.py`: `_onOCPDocumentInvited` used an undefined name `id` | Latent `NameError` that broke the invitation event path (shared document appearing at the other node). Uses the `doc` argument it was given, as the rest of the method already does. |

No other FreeCAD API used by the add-on was removed in 1.1: `FreeCADGui.PySideUic`,
`FreeCADGui.addCommand`, the global `Custom_N` toolbar parameter layout, the
document/GUI document observers, `dumpPropertyContent`/`restorePropertyContent`
and the property status API all still work.

## Verification performed

Environment: FreeCAD 1.1.4 (Windows, Qt 6.8.3, Python 3.11.14), add-on installed
in `%APPDATA%\FreeCAD\v1-1\Mod`, `ocp` 0.1.9 installed in
`AdditionalPythonPackages/py311`.

Observed and confirmed:

* Add-on initializes at startup with no errors in the FreeCAD report/console.
* `Collaborate` command is registered (`FreeCADGui.listCommands()`).
* Global toolbar `Custom_1` ("Collaboration Network", active, command
  `Collaborate`) is created and persists.
* Panels render correctly under Qt 6: node/document tabs, all indicators,
  fonts, installer page and document list widgets (verified by capturing the
  shown widget).
* Node startup from the panel works: `ocp start` spawns the node, the add-on
  reports "OCP node running" with P2P `0.0.0.0:7000` and API `localhost:8000`.
* The WAMP API connects: "WAMP API ready", `api.connected == True`.
* Opening a FreeCAD document creates a local collaboration entity
  (`status == "Local"`), and document changes (adding a `Part::Box`,
  recompute) still flow through the original observer pipeline.
* `Utils/StateMachineTest.py` was executed against PySide6 and fails at import
  with `module 'StateMachine' has no attribute 'Transitions'`. This is
  pre-existing: `Utils/StateMachine.py` has never defined `Transitions`, in the
  untouched baseline as well, so the bundled test is stale. Apart from the Qt
  import line it is unchanged here. The add-on ships no other tests, so there is
  no automated test suite to run.

## Remaining incompatibilities / not verified

1. **Document sharing and synchronisation cannot be completed with the
   available OCP backend.** Pressing *Share* reaches the node
   (`ocp.documents.create`) but the node answers
   `failed to find any peer in table` / `Unable to setup new swarm`
   (`OCP/p2p/Host.go`). The OCP node only replicates a document inside a swarm
   of peers, and the original OCP bootstrap servers are no longer online, so a
   node can never find a peer. This is a property of the abandoned OCP backend,
   not of the FreeCAD port; fixing it would mean replacing the networking
   architecture, which is explicitly out of scope.
   Consequently **share → invite → open → sync has not been verified**, with one
   instance or with two.
2. **The OCP node's WAMP router violates the current WAMP protocol.** It answers
   the add-on's parallel registrations (49 sessions, `roundrobin`) with a
   duplicate registration ID, which autobahn ≥ 23 rejects with
   `ProtocolError: REGISTERED received for already existing registration ID`.
   Older autobahn (22.7.1, the version of the add-on's era) tolerates it, but
   that release does not connect at all on Python 3.11 (its `cryptography`
   dependency and asyncio handling predate the current stack). The add-on is
   currently used with a current autobahn, where the node and API connect and
   plain calls work, but the registration storm causes session churn
   (`asyncio.QueueFull` in `OCP/API.py`). This is a node-side protocol defect.
3. **The add-on's built-in pip installer fails for `ocp` on FreeCAD 1.1.4.**
   FreeCAD bundles pip 25.3 with packaging 25.0, which raises
   `InvalidVersion: 'AMD64'` while evaluating `ocp`'s
   `platform_machine == "AMD64"` marker. `autobahn` and `aiofiles` install
   normally. Install `ocp` with a current standalone pip as shown in the README;
   this is why the README does not rely on the in-panel installer.
4. **Only Windows was tested.** The Linux/macOS code paths (`Qasync/_unix.py`)
   were updated for Qt 6 but not executed.
5. **The network indicators stay "Unknown"** because the node has no peers
   (see item 1). The `ocp.p2p.*` calls themselves are unchanged.
6. The original alpha limitations (concurrent edits, links between documents,
   mixing FreeCAD versions) are unchanged and still apply.

## Installing the ported add-on

```
# 1. requirements, into FreeCAD's importable package directory
py -3 -m pip install --target "%APPDATA%\FreeCAD\v1-1\AdditionalPythonPackages\py311" ocp "autobahn[serialization]" aiofiles

# 2. the add-on itself
cd "%APPDATA%\FreeCAD\v1-1\Mod"
git clone https://github.com/DVVagram/CollaborativeFC-Modernized.git

# 3. restart FreeCAD
```

Verified dependency versions: `ocp 0.1.9`, `autobahn 26.11.1`, `txaio 26.6.1`,
`aiofiles 25.1.0`, `cryptography 50.0.2`, `msgpack 1.2.4` (msgpack is also
shipped with FreeCAD).

`requirements.txt` is unchanged from upstream.

## Attribution

Original add-on © Stefan Troeger, LGPL-2.1 (see `LICENSE`). This port keeps the
original license, headers and authorship notices. `Qasync` remains the BSD
licensed work of its authors (see the file headers).
