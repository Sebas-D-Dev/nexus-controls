# Nexus Controls - Software Architecture Outline

This document outlines the proposed software architecture for **Nexus Controls**, a customizable, on-screen control surface for desktop computers.

---

## Core Technology Stack

- **Framework:** Electron.js
- **Backend Logic:** Node.js (within Electron's Main process)
- **Frontend UI:** React.js
- **Database:** SQLite (for structured, relational data) & File System (for assets)
- **Global Input:** iohook or a similar native Node.js module for global hotkey listening.

---

## 1. Frontend Architecture (Electron Renderer Process)

This component is responsible for everything the user sees and interacts with. It's the "face" of the application, running in a sandboxed Chromium window.

| Sub-Component Name      | Purpose & Logic                                                                                                                                                                                                 | Interacts With                                 |
|------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------|
| UI Window Manager      | Manages the main application window's state (docked, floating, hidden). Uses Electron's BrowserWindow API to control position, size, transparency, and always-on-top properties. Listens for events from Main.   | IPC Handler, Global Hotkey Manager (via Main)  |
| React App Root         | Main container for all React components. Manages overall app state, routes between views (deck, settings, marketplace).                                                  | State Manager, IPC Handler, Grid View          |
| State Manager          | Central store (Redux/Zustand) for frontend state: layout, button configs, UI settings. Dispatches actions, updates state, triggers re-renders.                           | React App Root, All UI Components              |
| IPC Handler (Frontend) | Module for sending/receiving messages to/from Backend (Main Process). E.g., button press sends `{ type: 'EXECUTE_ACTION', payload: { actionId: '...' } }`.              | Backend IPC Handler, Button, Settings Panel    |
| Grid View              | Renders main 8x8 grid of buttons. Maps layout data from State Manager, creates Button components, handles drag-and-drop for rearranging.                                  | State Manager, Button Component                |
| Button Component       | Represents a single key. Displays icon/label, handles input (click, long-press, touch). Calls IPC Handler to execute action. Visuals from state.                         | IPC Handler, Grid View                         |
| Settings Panel         | View for configuring app: profiles, hotkeys, grid layouts, marketplace login. Changes sent to State Manager and persisted via IPC Handler.                               | IPC Handler, State Manager, Profile Manager    |
| Marketplace Interface  | Fetches/displays plugins and icon packs from remote server. Handles browsing, searching, installing assets. Installs via IPC Handler.                                    | IPC Handler, Marketplace API Client (via Main) |

---

## 2. Backend Architecture (Electron Main Process)

The core of the application, running as a privileged Node.js process. Manages all heavy lifting and OS access.

| Sub-Component Name      | Purpose & Logic                                                                                                                                         | Interacts With                                 |
|------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------|
| Main Process Entry      | `main.js` starts the app, creates BrowserWindow, initializes backend components.                                                                        | UI Window Manager (Frontend), All Backend      |
| IPC Handler (Backend)   | Listens for messages from Frontend, routes tasks to backend services. E.g., 'EXECUTE_ACTION' calls Action Executor, 'SAVE_SETTINGS' calls Profile Manager. | Frontend IPC Handler, Action Executor, Profile |
| Action Executor         | Engine that performs actions assigned to buttons. Maps action types to functions, retrieves details from DB, executes logic.                            | IPC Handler, Plugin Manager, Database Manager  |
| Profile Manager         | Manages user profiles/layouts. Loads active profile, switches/saves profiles, updates frontend Grid View.                                              | IPC Handler, Database Manager                  |
| Plugin Manager          | Manages plugin lifecycle. Scans folder, validates manifests, loads plugins in sandbox. Exposes Nexus Controls API, routes plugin actions.               | Action Executor, Marketplace API, File System  |
| Marketplace API Client  | Communicates with external marketplace server. Fetches plugins/icons, downloads packages, manages auth. Saves assets to user data dir.                  | Plugin Manager, IPC Handler, Marketplace API   |
| Database Manager        | Abstraction for DB operations. Uses knex.js for CRUD on profiles, layouts, buttons. Keeps SQL out of business logic.                                   | SQLite DB, Profile Manager, Action Executor    |

---

## 3. Global Event Listener Architecture

Runs independently of app focus, ensuring global hotkeys work anytime.

| Sub-Component Name      | Purpose & Logic                                                                                                   | Interacts With                |
|------------------------|------------------------------------------------------------------------------------------------------------------|-------------------------------|
| Global Hotkey Manager   | Uses low-level system library (iohook) to listen for keyboard events OS-wide. Maintains registered hotkeys.        | Backend Main Process, OS      |

---

## 4. Data Storage Architecture

Defines how and where user data is stored on disk.

| Sub-Component Name      | Purpose & Logic                                                                                                   | Interacts With                |
|------------------------|------------------------------------------------------------------------------------------------------------------|-------------------------------|
| SQLite Database         | Single file (`nexus_controls_data.sqlite`) for all structured, relational data: profiles, layouts, buttons, actions. | Database Manager              |
| User Data Directory     | Folder in user's app data location (e.g., %APPDATA%). Stores DB file, plugins, icon packs. Keeps user data organized. | Plugin Manager, Marketplace   |

---

## Key Decisions to Make

### Plugin Architecture & Sandboxing

- **Decision:** How will plugins be executed? Separate, sandboxed process (security) vs. main Node.js process (simplicity/performance)?
- **Trade-offs:**
  - Separate process: robust security, isolates plugin code, prevents crashes/access issues, but complex and slower (async IPC).
  - Single process: simple, fast, but security risk (faulty plugin can compromise app).

### Marketplace API Contract

- **Decision:** RESTful API vs. static JSON manifest on CDN?
- **Trade-offs:**
  - RESTful API: scalable, supports accounts, paid assets, dashboards, versioning, reviews. Requires backend dev/maintenance.
  - Static manifest: cheap, easy, but limited (manual updates, no auth/submissions).

### Global Hotkey Listener Implementation

- **Decision:** Use iohook (powerful, low-level) or Electron's globalShortcut (simple)?
- **Trade-offs:**
  - iohook: listens to all events, supports advanced features, but complex (native compilation, OS permissions).
  - globalShortcut: reliable, easy, but limited to registered combos only.

### State Management Strategy

- **Decision:** Redux/Zustand vs. React Context API?
- **Trade-offs:**
  - Redux/Zustand: single source of truth, predictable transitions, debugging tools, but more setup/boilerplate.
  - Context API: simple for small apps, but can cause performance issues and tangled logic in large apps.
