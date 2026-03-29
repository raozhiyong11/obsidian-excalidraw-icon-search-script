# Obsidian Excalidraw Icon Search

[中文](README.md) | English

A fast icon search and insertion script for Obsidian Excalidraw.

The author previously switched from Yuque to Obsidian. While using Yuque, the iconfont SVG icon search plugin in the drawing board was frequently used - it was fast and convenient. After switching to Excalidraw, the inability to search for icons when inserting them into drawings became a limitation, with scarce materials available. This project was created to solve that problem.

It primarily addresses the issues of slow icon/SVG image search, tedious insertion steps, and frequent interruptions to the drawing workflow in Excalidraw. Users can now search and insert icons directly within the canvas workflow, significantly improving the efficiency of using image assets.

---

## Important Notice

1. This project is for learning purposes only. Please obtain authorization from the icon author before using any icons. This project is not responsible for any authorization-related disputes.
2. How to obtain authorization:
   You can search for keywords on https://www.iconfont.cn/ to find the corresponding icons. For assets marked as "paid use" on iconfont, please contact the author for authorization before use. You can find the icon author in the icon details section and reach out via direct messaging to obtain authorization.

3. Copyright infringement reporting:
   If you are an author and find someone using your icon without permission, you can provide the document link involved in the infringement along with screenshots proving original ownership. After verification, Yuque will block the infringing document.

---

## Overview

When creating flowcharts, architecture diagrams, product sketches, and visual knowledge diagrams in Obsidian Excalidraw, icons are often very high-frequency elements.

But the default workflow usually means:

- Leaving the canvas
- Opening a browser to search for assets
- Copying or downloading icons
- Returning to Obsidian
- Importing into Excalidraw

This process is both repetitive and breaks creative continuity.

The goal of this project is to compress "searching for icons" and "inserting icons" as much as possible within Excalidraw, making icon asset usage faster, smoother, and better suited for real drawing scenarios.

## Features

- Fast icon search within Obsidian Excalidraw
- Floating panel for browsing search results
- One-click insertion of icons into the current Excalidraw canvas
- Suitable for flowcharts, architecture diagrams, product sketches, knowledge visualization, and more
- Significantly reduces the repetitive operational costs of finding, importing, and inserting images

## Why This Project Is Valuable

This project doesn't just solve "whether you can find icons," but "how to use icons without interrupting your train of thought."

It addresses these high-frequency problems:

- Low efficiency in icon lookup
- Overly long insertion workflows
- Frequent switching between external websites and the canvas
- Noticeable efficiency drop when using many icons in Excalidraw

Its value lies in making icon search and insertion part of the Excalidraw workflow as much as possible.

## Screenshots

Please place screenshots in the following paths:

### Search Panel

Recommended path:
`docs/images/search-panel.png`

![Search Panel](docs/images/search-panel.png)

### Search Results

Recommended path:
`docs/images/search-results.png`

![Search Results](docs/images/search-results.png)

### Inserted on Canvas

Recommended path:
`docs/images/inserted-on-canvas.png`

![Inserted on Canvas](docs/images/inserted-on-canvas.png)

### Real Usage Example

Recommended path:
`docs/images/real-usage-example.png`

![Real Usage Example](docs/images/real-usage-example.png)

## Installation

This project currently works as an Excalidraw Script and does not need to be installed as a standalone Obsidian plugin.

### Requirements

Please ensure you have installed:

- [Obsidian](https://obsidian.md/)
- [Obsidian Excalidraw Plugin](https://github.com/zsviczian/obsidian-excalidraw-plugin)
- A working Excalidraw canvas

### Manual Installation

1. Get the script file:

   `iconfont-obsidian-search.md`

2. Place it in your Excalidraw Scripts directory. A common path is:

   ```text
   <Your Vault>/Excalidraw/Scripts/Downloaded/
   ```

3. Restart Obsidian, or refresh the Excalidraw script list.

4. Open an Excalidraw canvas and run the script from the script menu.

## Usage

### 1. Open the Script

In an Excalidraw canvas, open the script menu and run:

```text
iconfont-obsidian-search
```

After running, the icon search panel will appear.

### 2. Configure Cookie

The current search capability relies on the Yuque icon resource API, so a valid Yuque login cookie is required for first-time use.

Recommended workflow:

1. Log in to Yuque in your browser
2. Open developer tools
3. Find a request sent to Yuque
4. Copy the Cookie, or copy the full request headers
5. Paste into the configuration panel as prompted by the script and save

Once saved, you can search normally.

### 3. Search Icons

Enter keywords in the search box, for example:

- user
- arrow
- cloud
- database
- message

The script will return corresponding icon results.

### 4. Insert into Canvas

Click the icon you want to use, and the script will insert it into the current Excalidraw canvas.

This replaces the traditional multi-step workflow of "open website → download asset → return to canvas to import".

## Use Cases

- Flowcharts
- Architecture diagrams
- Product sketches
- Knowledge visualization notes
- Whiteboard-style design proposals
- Presentations and explanatory diagrams
- Any Excalidraw scenarios requiring frequent icon usage

## Current Status

- Current form: Excalidraw Script
- Main goal: Improve image/icon search and insertion efficiency in Excalidraw
- Currently depends on Yuque API workflow
- First-time use requires cookie configuration

## Technical Notes

- Obsidian Excalidraw Script Engine
- JavaScript
- CSS floating panel
- Obsidian request API
- SVG icon insertion workflow

## Security Notes

- Cookies are only used in the local user environment
- The project primarily serves to enhance local Excalidraw workflow
- Do not share personal account cookies indiscriminately

---

## Acknowledgements

- [Yuque](https://www.yuque.com/)
- [Obsidian](https://obsidian.md/)
- [Excalidraw](https://excalidraw.com/)
- [Obsidian Excalidraw Plugin](https://github.com/zsviczian/obsidian-excalidraw-plugin)

---

## License

MIT
