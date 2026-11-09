# FCCollaboration
FreeCAD add-on for concurrent edits, alpha state

Allows to share and edit documents with others concurrently. Based on the Open Collaboration Platform.

[![Alpha version preview](https://img.youtube.com/vi/fKSW4EgJups/0.jpg)](http://www.youtube.com/watch?v=fKSW4EgJups)

> **This repository is the original add-on ported to current FreeCAD.**
> It is a compatibility port only: the user interface, the collaboration
> workflows, the networking stack and the OCP protocol are unchanged.
> See [PORTING.md](PORTING.md) for what changed, what was tested and what
> still does not work.

## Compatibility

| Component | Version |
| --- | --- |
| FreeCAD | 1.1.x (tested with 1.1.4) |
| Qt / Python bindings | Qt 6 with PySide 6 (via FreeCAD's `PySide` module) |
| Python | 3.11 (the interpreter shipped with FreeCAD) |

FreeCAD 1.1 no longer ships PySide2/Qt5, so the original add-on refused to
load. This port uses FreeCAD's own `PySide` compatibility module, which
resolves to PySide6. Older FreeCAD versions (0.19, 0.20, 1.0) are **not**
supported by this repository.

## Installation

1. Install the requirements into FreeCAD's own package directory. The bundled
   `pip` of FreeCAD 1.1.4 cannot resolve the `ocp` package (it trips over a
   platform marker), so use any recent standalone Python for this one step:

   ```
   py -3 -m pip install --target "%APPDATA%\FreeCAD\v1-1\AdditionalPythonPackages\py311" ocp "autobahn[serialization]" aiofiles
   ```

   That directory is on FreeCAD's `sys.path`. On Linux/macOS use the matching
   `AdditionalPythonPackages/pyXY` directory inside FreeCAD's user data folder.

2. Clone this repository into FreeCAD's user `Mod` directory:

   ```
   cd "%APPDATA%\FreeCAD\v1-1\Mod"
   git clone https://github.com/DVVagram/CollaborativeFC-Modernized.git
   ```

   The add-on can also be added to the Add-on Manager as a custom repository
   and installed from there.

3. Restart FreeCAD. The Collaboration icon appears in the toolbar.

## Usage

After installation the CollaborativeFC icon appears in the toolbar, all interaction happens with this command:

![icon](https://user-images.githubusercontent.com/348477/132891156-e96d2ddf-79c7-4ec2-b812-d1e95e8ef80f.png)


Activating the command opens the collaboration panel:

![node](https://user-images.githubusercontent.com/348477/132888811-97c4b3b9-f5d0-42c4-9187-3a7e64547a6a.png)

### Connect to the network
In the node tab your connection status to the collaboration network is visible, the document tab is used to share individual documents with others.
Initially you are not connected to the network, which is indicated by the grey circles. So first you need to startup the node by pressing the button next to the first indicator. Make sure all 4 indicators in the node panel turn green, only then you are fully setup. The reachability indicator can take a few seconds. If it turns red you need to configure your router with port forwarding.

Once connected, your Node ID will be shown at the top (next to the logo). This id will always stay the same for you.

**Note:** on first startup the node creates its storage directory at
`%USERPROFILE%\.ocp` (or `~/.ocp`). The two rightmost indicators stay grey /
"Unknown" until the node has at least one peer, because the original OCP
bootstrap servers are no longer online. See PORTING.md.

### Share documents
When fully connected switch to the documents tab. There all relevant documents are handled, which are:
1. **`Local documents`**: The ones open in your FreeCAD, but are not shared
2. **`Shared documents`** Open in FreeCAD, and actively shared
3. **`Node documents`**: Shared, and hence available on the node, but not open in FreeCAD
4. **`Invited documents`**: Someone else added shared a document with you, but you did not yet open it, so its not available on the node

**Note:** Which document has which status can be seen by the `Type` description of the document (as seen in the figure below)

![documents](https://user-images.githubusercontent.com/348477/132892236-0e339c5b-eb1e-4219-919b-bf1d7c3710b7.png)

The 3 buttons for each document control all relevant actions needed to change the status of the document:
1. The first switches between `Share` and `Stop`. Pressing `Share` adds the document to the node and hence makes it available in the network. Pressing `Stop` removes it from the node and hence ends sharing.
2. The second button switches between `Open` and `Close`, and handles the document within FreeCAD. It allows to open a document that is only available on the node.
3. The `Edit` button allows to define the sharing details for the document, like the other people to share it with.

The simplest way of sharing a document is simply open one in FreeCAD, and than press the `Share` button for it.

### Invite others to collaborate
Sharing the document makes it available in the network, but does not yet allow anyone to open and work on it except you. This can be seen when using the `Edit` button, your node is the only added one (compare the listed Node ID with your own):

![edit](https://user-images.githubusercontent.com/348477/132892928-ce65a265-77a7-4a33-8edb-cc621f012a96.png)

To add others you need to know unique Node ID. Currently there is no way to find people in the network, so figure out a way to exchange with them in some other medium (chat, social media, email etc...). Once their ID is known, add it to the Node ID text field at the bottom and hit `Add`. **Note:** by default nodes are added with 'Read' rights only, so click the `Edit rights` checkbox before using `Add` to allow the other node to make changes.

Once added your document will show up at the other node as type `invited`, and the person can choose to join the document.

## Alpha state limitations:
1. Links into other documents do not work
2. Does not work reliable with different FreeCAD versions. All 0.18 or all 0.19 should work, but not a mix
3. Concurrent editing will fail at the moment. You can work in both FreeCAD instances and it should be mirrored, but if you do it at the same time synchronization is not guaranteed yet. E.g. if one instance has a longrunning update on a model, and the other instance edits stuff during the update, it currently breaks.
4. If you are brave enough to test it over different PCs most likely it will fail quite fast

## License

Original add-on and this port: GNU Lesser General Public License v2.1, see
[LICENSE](LICENSE). Original author: Stefan Troeger. The bundled `Qasync`
package is BSD licensed (see its file headers).
