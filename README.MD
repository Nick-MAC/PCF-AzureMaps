# PCF-AzureMaps

> **Disclosure:** This project is co-authored with [Claude](https://www.anthropic.com/claude), an AI assistant made by Anthropic. Claude helped write the code, documentation and reviews, working under the direction of the maintainer, who owns the decisions and the releases. Commits that Claude helped write carry a `Co-Authored-By: Claude` line.

A Power Apps Component Framework (PCF) control that embeds an interactive [Azure Maps](https://learn.microsoft.com/en-us/azure/azure-maps/) map in a **canvas app**. It renders pins from a bound table, lets users search, select, add, delete and open locations, and reports the city / county / state / ZIP of any point, all through a canvas-first data contract (a table in, output properties out).

It works in Azure Commercial and in Azure Government (GCC, GCC High, DoD) tenants.

- [Why This Control Exists](#why-this-control-exists)
- [Feature Overview](#feature-overview)
- [Azure Maps Authentication Options](#azure-maps-authentication-options)
  - [SAS URL Azure Architecture (High Level)](#sas-url-azure-architecture-high-level)
  - [Securing the token broker](#securing-the-token-broker)
- [How To Use](#how-to-use)
  - [Prereqs](#prereqs)
  - [Part 1: Install the solution zip](#part-1-install-the-solution-zip)
  - [Part 2: Choose and set up authentication](#part-2-choose-and-set-up-authentication)
  - [Part 3: Configure the control in your Canvas app](#part-3-configure-the-control-in-your-canvas-app)
- [Property Reference](#property-reference)
- [Feature Details](#feature-details)
  - [Pins, hover and selection](#pins-hover-and-selection)
  - [Syncing a gallery with the map](#syncing-a-gallery-with-the-map)
  - [Filtering from a cluster](#filtering-from-a-cluster)
  - [Per-pin colors](#per-pin-colors)
  - [Clustering](#clustering)
  - [Search](#search)
  - [Default view for empty datasets](#default-view-for-empty-datasets)
  - [Map controls](#map-controls)
  - [Error handling](#error-handling)
- [Adding, Deleting and Opening Points](#adding-deleting-and-opening-points)
  - [Wiring it up with OnChange](#wiring-it-up-with-onchange)
- [Address (City / County / State / ZIP)](#address-city--county--state--zip)
  - [New points](#new-points)
  - [Existing records](#existing-records)
- [Canvas App Dataset Terminology](#canvas-app-dataset-terminology)
- [Building From Source](#building-from-source)
- [Troubleshooting](#troubleshooting)
- [Acknowledgments](#acknowledgments)

## Why This Control Exists
Geospatial controls are unavailable in Power Apps US Government (including GCC), per [Microsoft Learn](https://learn.microsoft.com/en-us/power-apps/maker/canvas-apps/geospatial-overview#prerequisites-for-full-support). So a PCF is required to get a map UX in a Power App in GCC/GCC-H/DOD tenants.

There are already some Azure Maps controls in the [PCF ecosystem](https://pcf.gallery/search?q=azure+maps), but many are designed primarily around model-driven app patterns or don't provide a clean, canvas-first data contract for table-style `Items` data. This control is intentionally built for canvas app usage where makers typically pass a table/collection and map columns explicitly.

## Feature Overview

**Map and data**
- Pins rendered from the rows of a bound dataset; rows with missing or non-numeric coordinates are skipped
- Hover popups on pins showing the record title; click a pin to select it and click empty map to deselect, with the `selectedRecordId` output (blank when nothing is selected)
- Two-way selection sync: the app can drive the highlighted pin with `activeRecordId`
- Optional per-pin colors from a dataset column
- Clustering (on by default); clicking a cluster zooms in, and projects sharing one exact point open a "Select a project" list instead
- Configurable default camera for datasets with no points

**Interaction**
- In-map location search (address, city, state, ZIP, country); when Search Country Set includes US, a state name or two-letter abbreviation returns the state itself
- Style picker, clustering on/off toggle, zoom in/out, and a refocus-to-dataset button
- Add / Update point, Delete point, an optional Open record callout (`showOpenRecord`), and an optional cluster filter (`showClusterFilter`), all reported to the app through output properties (`pendingAction` is `add`, `delete`, `open` or `filter`); handle them in `OnChange`, because the control never raises `OnSelect`
- The search box, style picker, clustering toggle, add and delete controls can each be hidden; zoom in/out and refocus are hidden together

**Address data**
- City, county, state and ZIP are reverse geocoded automatically for every newly added point
- An on-demand lookup (`lookupLatitude` / `lookupLongitude`) returns the same address fields for existing records and for backfills, through the `lookupCity` / `lookupCounty` / `lookupState` / `lookupPostalCode` outputs, with `lookupStatus` reporting progress. The control only reports them, so your app saves them. Blank inputs clear the outputs

**Platform**
- Subscription key or SAS-token authentication (no key on the client with SAS)
- Azure Commercial and Azure Government domains
- Production webpack bundle (no `eval`; dev builds embed it and the Power Apps solution checker flags it)
- Map errors are shown in the control; in SAS mode, a transient failure of the first token request retries on its own (the token request retries, then the map rebuilds itself up to 3 times) before the error appears, and a shown error clears when the subscription key, auth function URL or map domain changes

## Azure Maps Authentication Options
For an introduction see https://learn.microsoft.com/en-us/azure/azure-maps/azure-maps-authentication. This control supports two [authentication approaches](https://learn.microsoft.com/en-us/javascript/api/azure-maps-control/atlas.authenticationoptions):

1. **Subscription key** (simpler, less secure for production)
2. **[SAS token URL](https://learn.microsoft.com/en-us/azure/azure-maps/azure-maps-authentication#configuring-application-with-sas-token)** (recommended for production)

| Auth Mode | Advantages | Disadvantages |
| --- | --- | --- |
| Subscription key | Fastest setup for dev/test; no extra Azure resources. | Key is exposed to client; broader access scope; harder production rotation/governance. |
| SAS URL (token broker) | No key in client; short-lived tokens; centralized policy and control in backend. | More setup and moving parts (Function, identity, CORS, ops); higher maintenance overhead. |

The same authentication is used for the map tiles, the location search and the address (reverse geocode) lookups. No additional Azure resources are needed for search or address data.

> [!NOTE]
> This control cannot use a Power Platform custom connector or any other endpoint that requires the caller to sign in. It requests the token from `azureMapsAuthFunctionUrl` with a plain `GET` and sends no credentials or auth header, and it has no property that accepts a token from the app. The token broker therefore has to be reachable without authentication (a function key embedded in the URL also works). Putting a custom connector or Entra ID sign-in in front of the broker would need a change to the control.

### SAS URL Azure Architecture (High Level)
When using `azureMapsAuthFunctionUrl`, the architecture is:

1. **Canvas App + PCF control** requests a SAS token from your auth endpoint.
2. **Azure Function token broker** authenticates to Azure Resource Manager using managed identity.
3. Function calls [Azure Maps management API `listSas`](https://learn.microsoft.com/en-us/rest/api/maps-management/accounts/list-sas) to mint a short-lived token.
4. Function returns token payload to the PCF control.
5. PCF initializes Azure Maps Web SDK using SAS auth and the returned token. The control also sends the account client ID (the broker's `clientId` field if present, otherwise the token's `iss` claim) as the `x-ms-client-id` header on Azure Maps requests.

Typical Azure resources involved:
- Azure Maps account (the map/data resource).
- Azure Function App (the token broker endpoint). Its CORS settings must allow the origin(s) your canvas app runs from (in GCC High these are `island.powerapps.us` hosts. Copy each exact origin from the browser's `Origin` request header, with no trailing slash or path, for example `https://authoring.<region>.gateway.gov.island.powerapps.us` for Studio. The published app runs from a different origin, so add that one too).
- [Managed identity with RBAC on Azure Maps](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview) (used to request SAS tokens).

### Securing the token broker
Read this before sharing a canvas app that uses SAS mode.

**What the control does.** It requests the token with a plain `GET` and no credentials or auth header, so the broker endpoint has to be reachable without authentication (or have a function key embedded in the URL, which is no more secret). That means **anyone who learns the URL can mint tokens for your Azure Maps account**, and the usage is billed to you. CORS does not prevent this: it only stops other *websites* from reading the response in a browser, not scripts or `curl`. Treat the Function URL like a credential: it lives in a control property of the app, so anyone who can edit the app or inspect its network traffic can see it.

**Limit the damage (in Azure, on the broker side):**
- **Short token lifetime.** Set a short `expiry` in the `listSas` request so a leaked token dies quickly.
- **Rate limit the token.** Set a low `maxRatePerSecond` in the `listSas` request.
- **Least privilege.** Give the managed identity only the roles it needs on the Maps account (for example **Azure Maps Data Reader** for the data access the token carries), and nothing broader.
- **Watch spend.** Configure Azure Maps account usage alerts and a cost budget, so abuse shows up quickly.
- **CORS allowlist.** Allow only the exact Power Apps origins you use, no wildcards and no credentials. This is hygiene, not protection against scripted abuse.
- **Restrict network access** to the Function App where your environment allows it (access restrictions, private endpoints).
- **Not available with this control.** The token request carries no credentials, so a Function that requires Entra ID sign-in (or a custom connector in front of it) cannot be used as the broker.

**What the control does for you:**
- Requires an `https://` auth URL.
- Reuses one token until about a minute before its `exp` claim, so search and address lookups do not hit the broker on every request (a token without a readable `exp` is not cached).
- Retries the token request up to twice (after 0.4 s and 1.2 s) on a network error, HTTP 408, 429 or 5xx. If the first token request still fails with one of those transient errors, the map rebuilds itself up to 3 times before showing the error. A problem that persists, such as a CORS block or an unreachable URL, still shows the error once the retries run out, so it can take a while to appear.
- Never shows the broker's error response body in the app or the console (only the HTTP status), so a misconfigured Function cannot leak stack traces to users.

**Subscription key mode** sends the key from the client (and in the query string of search requests). Use it for development only, and rotate the key if it was ever shared.

To report a vulnerability in this control, see [SECURITY.md](SECURITY.md).

## How To Use
### Prereqs
1. Power Apps Premium licensing is required for this control. It uses external services (`atlas.microsoft.com` / `atlas.azure.us`), so treat it as a premium canvas app component. See https://learn.microsoft.com/en-us/power-apps/developer/component-framework/overview#licensing.
2. PCF/code components must be allowed in the target environment before the control can be used in Canvas apps. In the Power Platform admin center (PPAC), verify the environment setting that enables code components for canvas apps. See https://learn.microsoft.com/en-us/power-apps/developer/component-framework/component-framework-for-canvas-apps#prerequisites

### Part 1: Install the solution zip
1. Download `PCF_AzureMaps_#-#-#-#.zip` from the latest [GitHub Release](../../releases/latest) (it is also committed under `PCF_AzureMaps/bin/Release/`).
2. Open the Power Apps maker portal for your cloud (for example [make.powerapps.com](https://make.powerapps.com/) for commercial).
3. Go to `Solutions`.
4. Select `Import solution`.
5. Upload the zip and complete the import.

> [!NOTE]
> The solution is managed. Importing a zip with the **same version number** as the installed one is skipped, so every new build ships with a new version number. When a newer version is imported over an existing one, open your app in the designer and accept the prompt to update the code component.

### Part 2: Choose and set up authentication
1. [Create an Azure Maps resource](https://learn.microsoft.com/en-us/azure/azure-maps/quick-demo-map-app)
2. Choose one auth mode:

  **Option A: Subscription key**
  - Retrieve a subscription key (Primary or Secondary).
  - Pass that key to `subscriptionKey`.
  - Use for dev/test or low-risk scenarios.

  **Option B: SAS URL (recommended)**
  - Deploy an auth endpoint (for example Azure Function) that answers an unauthenticated `GET` with the SAS token. Return JSON with `Content-Type: application/json` (`{ "token": "..." }`; `accountSasToken` and `sasToken` are also read), or the bare token as plain text. An optional `clientId` field sets the `x-ms-client-id` value; without it the control uses the token's `iss` claim.
  - Configure `azureMapsAuthFunctionUrl` with that endpoint.
  - Leave `subscriptionKey` empty when using SAS URL mode.

Provide **exactly one** of `subscriptionKey` or `azureMapsAuthFunctionUrl`. If both hold valid values the control shows a configuration error. The auth function URL must be `https://` (plain `http://` is accepted only for a loopback dev Function: `localhost`, `127.0.0.1` or `[::1]`).

### Part 3: Configure the control in your Canvas app
The control expects a bound dataset (PCF `DataSet` input) and a small set of additional property values.

Each row in the dataset represents one location. Rows with missing or non-numeric latitude/longitude are ignored. `recordIdColumn` should be unique per row to avoid ambiguous selection behavior. Map these dataset columns:

| Property | Required | Datatype | Notes |
| --- | --- | --- | --- |
| `recordIdColumn` | Yes | Text/Guid/Number (string-convertible unique id) | Stable identity for each pin; used for selection sync and for every record id the control outputs. |
| `latitudeColumn` | Yes | Decimal number | Valid range is `-90` to `90`. |
| `longitudeColumn` | Yes | Decimal number | Valid range is `-180` to `180`. |
| `titleColumn` | No | Text | Label shown on hover and in the list that opens for projects sharing one exact point. |
| `detailColumn` | No | Text | Second line shown under the title in the list that opens for projects sharing one exact point, for example a project number. When not mapped, the record id is shown. Display only: it does not change `selectedRecordId` or any other output. |
| `colorColumn` | No | Text | Pin color as a hex value (`#C62828`) or CSS color name (`red`). Blank or invalid values use the default pin color. |

Set each mapping property by typing the source column name as a string value (for example, `"Latitude"`, `"Longitude"`, `"AccountId"`). These mapping properties are not shown in the standard properties pane. Use the property picker/search, find the PCF mapping property name, and enter the column name string there.

## Property Reference

### Inputs

| Property | Required | Datatype | Notes |
| --- | --- | --- | --- |
| `sampleDataSet` | Yes | Dataset | Map Points Dataset: one row per location. In Canvas Studio you bind it through the control's `Items` property, then map its columns with the `*Column` properties in [Part 3](#part-3-configure-the-control-in-your-canvas-app). |
| `subscriptionKey` | Conditional | Text | Subscription-key auth mode. Leave empty when using `azureMapsAuthFunctionUrl`. |
| `azureMapsAuthFunctionUrl` | Conditional | Text | URL of your token broker endpoint that returns an Azure Maps SAS token payload. Leave empty when using `subscriptionKey`. |
| `mapDomain` | No | Text | [Azure Maps domain host](https://learn.microsoft.com/en-us/azure/azure-maps/how-to-use-map-control#azure-government-cloud-support). Use `atlas.microsoft.com` (Azure public cloud) or `atlas.azure.us` (Azure Government, for example GCC High). Blank or any other value silently falls back to `atlas.microsoft.com`, so set `atlas.azure.us` explicitly in GCC High. |
| `mapStyle` | No | Text | Default [Azure Maps style](https://learn.microsoft.com/en-us/azure/azure-maps/choose-map-style#add-the-style-picker-control) name. One of `road`, `grayscale_light`, `grayscale_dark`, `night`, `road_shaded_relief`, `satellite`, `satellite_road_labels`, `blank` or `high_contrast_dark`; blank or any other value uses `road`. |
| `defaultLatitude` / `defaultLongitude` | No | Decimal | Camera center used only when the dataset has no points (for example `39.8` / `-98.5` for the continental US). Set both; the center is ignored unless latitude is `-90` to `90` and longitude is `-180` to `180`. |
| `defaultZoom` | No | Whole number | Camera zoom used only when the dataset has no points (for example `4` for the continental US). Valid range is `0` to `24`; other values are ignored. |
| `searchCountrySet` | No | Text | Comma-separated ISO country codes that limit the location search (for example `US`). When it includes `US`, state names and two-letter abbreviations are recognized. Blank searches worldwide. Every entry must be a two-letter code; if any entry is not (for example `USA`), the whole value is ignored and the search is worldwide. |
| `activeRecordId` | No | Text | Record id to highlight and center from the app side. |
| `hideAddPoint` | No | Yes/No | Hides the **Add/Update Point** control and disables click-to-add. Defaults to shown. |
| `hideDeletePoint` | No | Yes/No | Hides the **Delete point** control. Defaults to shown. |
| `hideSearchBar` | No | Yes/No | Hides the location search box. Defaults to shown. |
| `hideStyleControl` | No | Yes/No | Hides the map style menu. Defaults to shown. |
| `hideClusterControl` | No | Yes/No | Hides the clustering on/off toggle. Clustering behavior is unaffected. Defaults to shown. |
| `hideZoomControls` | No | Yes/No | Hides the refocus and zoom in/out controls. Defaults to shown. |
| `showOpenRecord` | No | Yes/No | A selected pin shows a callout with an **open** button that emits `pendingAction = "open"`. Defaults to off. |
| `openButtonLabel` | No | Text | Label for the open button. Defaults to `"Open project"`. |
| `showClusterFilter` | No | Yes/No | Clicking a cluster still zooms into it and also emits `pendingAction = "filter"` with the ids of every project in that cluster. The shared-point list also gets a filter button. Defaults to off. |
| `filterButtonLabel` | No | Text | Label for the shared-point list button. Defaults to `"Filter to these"`. |
| `lookupLatitude` / `lookupLongitude` | No | Decimal | Coordinates to reverse geocode on demand. Set both; the lookup runs whenever they change, so they can be bound straight to form fields, and the result arrives through `OnChange`. Blank them to clear the `lookup*` outputs or to look the same point up again. See [Existing records](#existing-records). |

Every on-map control is individually hideable, so the same control can be dropped on a read-only overview screen (`hideAddPoint` / `hideDeletePoint` = `true`) and on a full-edit screen without code changes. The `hide*` flags default to **shown** (unset = visible), while `showOpenRecord` and `showClusterFilter` are opt-in.

### Outputs

| Property | Datatype | Notes |
| --- | --- | --- |
| `selectedRecordId` | Text | Record id of the selected pin. Blank when nothing is selected (the user clicked empty map, closed the callout, or deleted the selected point). |
| `pendingAction` | Text | `"add"`, `"delete"`, `"open"` or `"filter"`: the action the user requested. Read in `OnChange`. |
| `pendingLatitude` / `pendingLongitude` | Decimal | Coordinates of the point to add (when `pendingAction` is `"add"`). They are `0` after any other action, so read them only for `"add"`. |
| `pendingCity` / `pendingCounty` / `pendingState` / `pendingPostalCode` | Text | Address of the point to add (when `pendingAction` is `"add"`). State is a two-letter abbreviation in the US. Blank if the lookup failed or there is no address at that spot. |
| `pendingRecordId` | Text | Record id of the affected point (when `pendingAction` is `"delete"` or `"open"`). |
| `pendingRecordIds` | Text | Comma-separated record ids of the projects to filter to (when `pendingAction` is `"filter"`, which requires `showClusterFilter = true`): every project in the clicked cluster, or the projects in the shared-point list. |
| `actionToken` | Text | Changes on every add / delete / open / filter request so `OnChange` fires reliably, even for repeated actions. |
| `lookupStatus` | Text | `"pending"` while an on-demand lookup runs, `"done"` when the lookup outputs are valid, `"failed"` if the request errored. Blank if none was requested, and while either `lookupLatitude` or `lookupLongitude` is blank (blank inputs clear all `lookup*` outputs). |
| `lookupCity` / `lookupCounty` / `lookupState` / `lookupPostalCode` | Text | Address for the lookup coordinates. Valid only when `lookupStatus` is `"done"`; blank while pending, after a failure, or when the lookup inputs are blank. |

## Feature Details

### Pins, hover and selection
- Each valid dataset row becomes a pin. Pin labels are not permanently displayed; the title appears on hover.
- Clicking a pin selects it, centers on it (with clustering on, zooming in to at least level 15 so the pin is out of any cluster) and emits its id as `selectedRecordId`. Setting `activeRecordId` from the app selects and centers a pin from the other direction, and `selectedRecordId` follows so the two always agree.
- Clicking empty map, the **×** on the open-record callout, or **Delete point** deselects, and `selectedRecordId` becomes blank. Selecting another pin replaces the selection.
- To keep a gallery in sync with the map, see [Syncing a gallery with the map](#syncing-a-gallery-with-the-map).
- Rows with blank coordinates are skipped (no stray pin at 0,0).
- Adding or updating a point does not pan the camera back to the old location.

### Syncing a gallery with the map
You can keep a gallery next to the map in sync in both directions: selecting a project in the gallery selects its pin, and selecting a pin highlights its project in the gallery and, with the gallery `Items` sort shown below, moves it to the top of the list.

The reliable way is to keep **one variable**, `varSelectedId`, as the shared selection. The examples use a collection named `colProjects` (the gallery's source) and an id column named `recordId`; swap in your own names. Ids are text, so wrap non-text ids with `Text(...)`.

**1. Initialize the variable** in `App.OnStart` (or the screen's `OnVisible`):

```powerapps
Set(varSelectedId, "")
```

**2. Map control properties**

```powerapps
Active Record Id = varSelectedId
```

**3. Gallery, so a click selects the pin.** Gallery `OnSelect`:

```powerapps
Set(varSelectedId, Text(ThisItem.recordId))
```

The map selects that pin, centers on it, and shows the open-record callout if `showOpenRecord` is on.

**4. Gallery, so the selected project is highlighted.** Gallery `TemplateFill`:

```powerapps
If(
    Text(ThisItem.recordId) = varSelectedId,
    RGBA(255, 108, 87, 0.25),   // highlight color
    RGBA(0, 0, 0, 0)
)
```

**5. Map control `OnChange`, so a pin click updates the gallery.** Add this alongside your add / delete / open logic. It has its own condition, so it does not need the `actionToken` guard:

```powerapps
If(
    Self.selectedRecordId <> varSelectedId,
    Set(varSelectedId, Self.selectedRecordId)
)
```

**6. Gallery `Items`, so the selected project is listed first.** Sort the gallery's collection so the selected record comes first, whichever side made the selection:

```powerapps
SortByColumns(
    AddColumns(colProjects, "isSel", Text(recordId) = varSelectedId),
    "isSel",
    SortOrder.Descending
)
```

Here `colProjects` is the collection your map's dataset also reads; swap in your own. This works on a local collection only (it is not delegable). Use it for the gallery's `Items` only and keep the map's dataset `Items` on the unsorted collection: the control compares the dataset position by position, so a reordered list counts as changed data and makes the map redraw its pins and re-center the camera.

How it behaves:
- **Gallery to map:** a gallery click sets `varSelectedId`, the map selects and centers on the pin, and `selectedRecordId` follows it, so both sides always agree.
- **Map to gallery:** a pin click (or a pick from a cluster list) emits `selectedRecordId`, `OnChange` copies it into `varSelectedId`, the row highlights, and if the gallery `Items` sorts the selected record first, it moves to the top of the gallery.
- **Deselecting:** clicking empty map, closing the callout or deleting the selected point makes `selectedRecordId` blank. `OnChange` then clears `varSelectedId`, which removes the highlight.
- **Why a variable, not `Gallery.Selected`:** after the user deselects on the map, the gallery still has its row selected, so clicking that same row again would not change `activeRecordId` and the pin would not reselect. Clearing `varSelectedId` on deselect avoids this.

### Filtering from a cluster
With `showClusterFilter = true`, clicking a cluster bubble zooms into it as usual **and** emits `pendingAction = "filter"` with `pendingRecordIds` holding the ids of every project in that cluster (comma separated, however many). Check for `"filter"` in `OnChange`. For projects sharing one exact point, clicking the bubble opens their list as before, and that list has a single **Filter to these (N)** button that zooms to the point and emits the same signal. The button label is set by `filterButtonLabel`. The control does not filter anything itself; it hands your app the ids and your app filters, so the map **and** the gallery update together when both read the same collection.

**Setup**
1. Keep your full data in one collection or table (here `colAllProjects`), and a second collection that feeds the map and gallery (here `colMapProjects`). Fill it at start:

```powerapps
// App.OnStart or screen OnVisible
ClearCollect(colMapProjects, colAllProjects)
```

2. Set both the map's dataset `Items` and the gallery's `Items` to `colMapProjects`. If you use the selected-first sort from [Syncing a gallery with the map](#syncing-a-gallery-with-the-map), put that sort around `colMapProjects` on the gallery's `Items` only and leave the map on the plain collection. Re-sorting the rows the map reads makes it redraw its pins and move the camera.
3. Handle `"filter"` in the control's `OnChange`, inside the `actionToken` guard (see the full example in [Wiring it up with OnChange](#wiring-it-up-with-onchange)):

```powerapps
"filter",
ClearCollect(
    colMapProjects,
    Filter( colAllProjects, Text(recordId) in Split(Self.pendingRecordIds, ",").Result )
)
```

4. Add your own button (outside the map) to clear the filter:

```powerapps
// "Show all projects" button OnSelect
ClearCollect(colMapProjects, colAllProjects)
```

Notes:
- Ids are comma separated, so this assumes your record ids do not contain commas (GUIDs and project numbers do not).
- Filter a local collection (as above) rather than the source table directly: `Filter(... in Split(...))` may not be delegable.
- The shared-point list closes when you click its filter button. When the collection changes, the pins redraw and the camera refits to the remaining projects (or re-centers on the selected project, if it is still in the collection).
- Filtering a very large cluster means a long `pendingRecordIds`. If your gallery and the map read the same collection, filtering also shrinks the map and the camera refits; give the gallery its own collection if the map should keep every pin.
- The control has no clear signal; add your own button that restores the full collection. With `showClusterFilter` off, clicking a spread-out cluster just zooms in.

### Per-pin colors
Map the optional `colorColumn` to a column holding a hex value or CSS color name, for example:

```powerapps
AddColumns(
    <your existing Items expression>,
    PinColor,
    If(ThisRecord.NeedsReview, "#C62828", "#2F6AB3")
)
```

Pins without a valid color use the default coral pin; clusters use a dark coral (bright coral when the cluster contains the selected pin); the selected pin is bright coral so it stays visible against the rest.

### Clustering
- Clustering is on by default and can be toggled in the map (hide the toggle with `hideClusterControl`).
- **Real clusters** (projects spread over an area): the cursor becomes a pointer, and clicking zooms into the cluster. With `showClusterFilter = true` the click also emits the filter signal (see below).
- **Projects sharing one point:** clicking the bubble opens a **Select a project (N)** list instead of zooming, since zooming would never separate them. The list opens on click and names the projects. The list shows at most the first 100 projects. Choosing one selects it. Each entry shows the title and, underneath, the value of `detailColumn` (for example a project number); if `detailColumn` is not mapped, the record id is shown instead.
- With `showClusterFilter = true`, that list also has a **Filter to these (N)** button (zooms to the point and filters). See [Filtering from a cluster](#filtering-from-a-cluster).

### Search
- The search box accepts an address, city, state, ZIP or country (powered by [Azure Maps Fuzzy Search](https://learn.microsoft.com/en-us/rest/api/maps/search/get-search-fuzzy)) and shows up to 8 matches. Selecting a result places a marker and moves the camera.
- Area results (states, cities) frame the whole area instead of zooming to a single point.
- With `searchCountrySet` including `US`, a query that is a state name or two-letter abbreviation (for example `maryland` or `md`) returns the actual state and frames it, instead of same-named towns.

### Default view for empty datasets
When the dataset has no points the camera falls back to `defaultLatitude`, `defaultLongitude` and `defaultZoom` (for example `39.8`, `-98.5`, `4` for the continental US). Datasets with points still zoom to fit their points. The center needs both `defaultLatitude` (-90 to 90) and `defaultLongitude` (-180 to 180); `defaultZoom` (0 to 24) applies on its own, and out-of-range values are ignored. With nothing set, an empty map starts near Washington, DC at zoom 5.

### Map controls
All controls can be hidden individually with the `hide*` inputs.
- **Style picker:** switch between Azure Maps base styles.
- **Cluster toggle:** turn clustering on or off.
- **Add/Update Point:** enters one-shot add mode; click the map to place the point.
- **Delete point:** shown while a pin is selected; requests deletion of that pin (`pendingAction = "delete"`). Your app removes the row.
- **Refocus / zoom in / zoom out.**

### Error handling
If the map cannot load (bad key, blocked domain, CORS, SAS failure) the control shows a message instead of a blank map. In SAS mode a transient failure of the first token request is retried first: the token request retries, and the map rebuilds itself up to 3 times before the message appears. Once the message is showing, nothing retries on its own. Changing the subscription key, auth function URL or map domain clears the message and rebuilds the map; after a fix made elsewhere (for example adding a CORS origin), reload the app. A bad auth configuration is reported before any request is made.

## Adding, Deleting and Opening Points
The control can request row changes, but it does **not** write to your data source itself. Instead it emits output properties and the maker wires `Patch()` / `Remove()` in the control's `OnChange`.

In-map UX:
- **Add / update a point:** click **Add/Update Point** (top-right) to enter add mode, then click anywhere on the map. The coordinates and address are emitted as `pendingLatitude` / `pendingLongitude` / `pendingCity` / `pendingCounty` / `pendingState` / `pendingPostalCode` with `pendingAction = "add"`. Add mode is one-shot (it turns off after one placement). Your `OnChange` decides whether that creates a new row or updates an existing one.
- **Delete a point:** click a pin to select it, then click **Delete point**. The selected id is emitted as `pendingRecordId` with `pendingAction = "delete"`.
- **Open a record:** with `showOpenRecord = true`, clicking a pin shows a callout with an **open** button (label set by `openButtonLabel`). Clicking it emits the pin's id as `pendingRecordId` with `pendingAction = "open"`. The control never raises `OnSelect` (Studio lists the property, but it does nothing here); every action goes through `OnChange` + `actionToken`: handle `"open"` in `OnChange` to run your own navigation Power Fx (for example `Navigate(...)`).
- **Filter to a cluster:** with `showClusterFilter = true`, clicking a cluster zooms in and emits the ids of its projects as `pendingRecordIds` (comma separated) with `pendingAction = "filter"`. The list that opens for projects sharing one exact point has a **Filter to these (N)** button that does the same for its projects. N is the number of projects, and the text before it can be changed with `filterButtonLabel`.

> [!NOTE]
> "Creating the relevant columns" happens at design time: the table and its columns (latitude, longitude, title, city, state, ZIP, etc.) must already exist in your data source. The control provides **values** for a new row; it does not create table schema at runtime.

### Wiring it up with OnChange

The control raises its single `OnChange` event whenever an add, delete, open or filter is requested. Two things make this reliable:

1. **Always-defined outputs.** The control reports concrete values (empty string / `0`) rather than blanks, so the outputs are always defined; this is meant to keep `OnChange` firing reliably.
2. **The `actionToken` guard.** `OnChange` fires for *every* output change, including selecting a pin (which updates `selectedRecordId`) and lookup results. Because `pendingAction` stays set ("sticky") until the next action, re-running the formula on an unrelated `OnChange` would repeat the last add/delete. `actionToken` increments only on a genuine add/delete/open/filter request, so gate your logic on it. Each new action also resets the `pending*` fields that do not belong to it to empty / `0`.

Initialize a tracker in **App.OnStart**:

```powerapps
Set(varLastToken, "0")
```

Then set the control's **OnChange** (collection example; swap in your collection/column names):

```powerapps
// Only act when actionToken changes (a real add/delete/open click), never on incidental OnChange.
If(
    Self.actionToken <> varLastToken,
    Set(varLastToken, Self.actionToken);
    Switch(
        Self.pendingAction,

        "add",
        Collect(
            colLocations,
            {
                recordId:  Text(GUID()),
                latitude:  Self.pendingLatitude,
                longitude: Self.pendingLongitude,
                city:      Self.pendingCity,
                county:    Self.pendingCounty,
                state:     Self.pendingState,
                zip:       Self.pendingPostalCode,
                title:     "New location"
            }
        ),

        "delete",
        RemoveIf( colLocations, recordId = Self.pendingRecordId ),

        // Requires showOpenRecord = true. Swap in your existing navigation Power Fx.
        "open",
        Navigate( DetailScreen, ScreenTransition.None, { RecordId: Self.pendingRecordId } ),

        // Requires showClusterFilter = true. colAllProjects / colMapProjects are the collections from "Filtering from a cluster", not colLocations above.
        "filter",
        ClearCollect(
            colMapProjects,
            Filter( colAllProjects, Text(recordId) in Split(Self.pendingRecordIds, ",").Result )
        )
    )
)
```

For a Dataverse table instead of an in-memory collection, swap the bodies for `Patch(YourTable, Defaults(YourTable), {...})` and `Remove(YourTable, LookUp(YourTable, <idColumn> = Self.pendingRecordId))`, keeping the same `actionToken` guard.

After the `Collect`/`RemoveIf` (or `Patch`/`Remove`) updates the bound dataset, the new or removed pin is reflected automatically on the next refresh. Make sure the id you store is text so `recordId = Self.pendingRecordId` compares correctly (the control emits ids as strings); wrap a non-text id with `Text(...)`.

## Address (City / County / State / ZIP)
Address data comes from the [Azure Maps reverse geocoding](https://learn.microsoft.com/en-us/rest/api/maps/search/get-search-address-reverse) service, called with the same key or SAS token as the map. No extra Azure Function is needed. Lookups are best effort: if one fails, or the spot has no address (for example open water), the address fields are blank and nothing else is blocked. In the US the state is returned as a two-letter abbreviation. If several ZIPs are returned for an area, the first is used.

### New points
Every newly added point is reverse geocoded automatically. The add is emitted once the lookup returns, so one `OnChange` carries the coordinates and the address together (`pendingCity`, `pendingCounty`, `pendingState`, `pendingPostalCode`). Save them in the same `Patch` / `Collect` as the coordinates (see the example above).

### Existing records
Existing records are not geocoded on their own. To get the address for a record that is already in your table (before saving it, or to backfill many records), trigger the on-demand lookup:

1. Bind the control's `lookupLatitude` / `lookupLongitude` inputs to variables, e.g. `varLookupLat` / `varLookupLon`.
2. To start a lookup, set the variables to the record's coordinates. The control does not repeat a lookup for the coordinates it just looked up (including after a failure), so to look up the same coordinates again the inputs must go blank in between (`Blank()`). Blank inputs also clear `lookupStatus` and the `lookup*` outputs, so a previous record's address is never left behind.
3. In the control's `OnChange`, wait for `Self.lookupStatus = "done"`, then use `Self.lookupCity`, `Self.lookupCounty`, `Self.lookupState` and `Self.lookupPostalCode`.

**Simpler when the coordinates are already on screen:** if a form already shows the latitude and longitude, bind the inputs straight to those fields (for example `Lookup Latitude = <latitude field>.Value`, `Lookup Longitude = <longitude field>.Value`). The lookup then runs whenever the coordinates change, including when the form opens a record, so by the time the user saves, `lookupCity`, `lookupCounty` and `lookupState` are already filled and the save formula can read them directly. Use them only while `lookupStatus = "done"`; they are blank while a lookup runs or after a failure, so on an update guard each field with `If(<ControlName>.lookupStatus = "done", <ControlName>.lookupCity, <existing value>)` to avoid blanking saved data.

Example: look up the address when saving a project (or when the project opens), then finish the save in three steps.

**1. Bind the inputs in the control's properties pane**

```powerapps
Lookup Latitude  = varLookupLat
Lookup Longitude = varLookupLon
```

**2. On the Save button (or when the project opens), start the lookup instead of saving straight away**

Use `Gallery.Selected.latitude` or your form's `Form.Item.latitude` in place of `ThisItem.latitude` if `ThisItem` isn't in scope.

```powerapps
// Save button OnSelect: start the lookup instead of saving immediately
Set(varLookupLat, Blank());
Set(varLookupLon, Blank());
Set(varLookupLat, ThisItem.latitude);
Set(varLookupLon, ThisItem.longitude);
Set(varSavePending, true)
```

**3. In the control's OnChange, finish the save when the lookup is done**

Swap in your table, id and column names. Your normal add, delete and open logic can sit in the same `OnChange`, because it only runs on the `actionToken` change.

```powerapps
// Control OnChange: finish the save when the lookup returns
If(
    varSavePending && Self.lookupStatus = "done",
    Set(varSavePending, false);
    Patch(
        YourTable,
        LookUp(YourTable, recordId = varSelectedId),
        {
            city:   Self.lookupCity,
            county: Self.lookupCounty,
            state:  Self.lookupState,
            zip:    Self.lookupPostalCode
        }
    ),
    varSavePending && Self.lookupStatus = "failed",
    Set(varSavePending, false);
    Notify("Could not look up the address.", NotificationType.Error)
)
```

Things to know:
- **Status:** `lookupStatus` goes `"pending"`, then `"done"` (or `"failed"`). The outputs are only valid on `"done"`, so use `lookupStatus` rather than reading them right after setting the inputs. A new lookup resets the outputs to blank while `"pending"`, and blank inputs reset `lookupStatus` to blank and clear the outputs. Setting the inputs to the same coordinates as the last lookup does nothing, even after `"failed"`, until the inputs have gone blank or changed in between. That is why step 2 sets them to `Blank()` first.
- The lookup takes a moment and runs once per change.
- Changing the inputs to different coordinates starts a new lookup immediately, and a result that has been superseded by a newer request is discarded.
- **Skip when already filled:** If the record already has an address, check that first, for example `If(IsBlank(ThisItem.city), <start lookup>, <normal save>)`. That avoids a lookup on every save.
- **Backfilling many records:** Do one record at a time. Set the inputs for record 1, patch it on `"done"`, then move to the next. A single `ForAll` can't wait for each result.

## Canvas App Dataset Terminology
In canvas app terms, think of the input as a **table** (the same conceptual shape used by a control's `Items` property):
- A dataset is a table.
- Each row is a record.
- Each mapped column points to a field in that record.

Example logical shape:

```text
Table(
  { recordId: "A-1001", latitude: 47.6101, longitude: -122.2015, title: "Seattle HQ" },
  { recordId: "A-1002", latitude: 37.7749, longitude: -122.4194, title: "San Francisco" }
)
```

> [!NOTE]
> If your source data does not have clean coordinates, you can geocode text addresses into coordinates with [Azure Maps Fuzzy Search](https://learn.microsoft.com/en-us/rest/api/maps/search/get-search-fuzzy) in your own flow, then bind the resulting latitude/longitude columns here.

## Building From Source
Requirements: Node.js 20 or later with npm, the .NET SDK, and optionally the [Power Platform CLI (`pac`)](https://learn.microsoft.com/en-us/power-platform/developer/cli/introduction).

```bash
npm install
npm run build        # compile and bundle the control (development build)
npm run lint         # eslint
```

**Local test harness.** `npm run start:watch` serves the PCF test harness on `127.0.0.1:8181` and rebuilds on every edit. Notes:
- The harness fills each property by type, not with `val` everywhere: text properties get the literal text `val`, number properties get `0`, and Yes/No properties get `true`. So every `hide*` option, `showOpenRecord` and `showClusterFilter` all start on, which hides the map controls. Set the `hide*` properties to `false` to see the controls. Replace `subscriptionKey` and `mapDomain` with real values (a `val` key returns 401, and `val` in `mapDomain` is ignored so the map falls back to `atlas.microsoft.com`).
- Test locally with `subscriptionKey` auth. SAS mode depends on your Function App CORS settings, which normally only allow your Power Apps origins, so it cannot be exercised from `localhost`.
- Set `mapDomain` to exactly `atlas.azure.us` for an Azure Government Maps account.
- Dataset columns for the harness: `recordIdColumn`, `latitudeColumn`, `longitudeColumn` (required) and `titleColumn`, `colorColumn`, `detailColumn` (optional).

**Packaging the solution.**

```bash
dotnet build PCF_AzureMaps/PCF_AzureMaps.cdsproj -c Release
```

This produces `PCF_AzureMaps/bin/Release/PCF_AzureMaps.zip` using the production bundle. For each release:
1. Bump `<Version>` in `PCF_AzureMaps/src/Other/Solution.xml` (`2.1.0.X`) and `version` on the `<control>` element in `mapPCF/ControlManifest.Input.xml` (`2.1.Y`).
2. Copy the generated zip to a versioned name, `PCF_AzureMaps_2-1-0-X.zip`, in `PCF_AzureMaps/bin/Release/`, commit it, and attach it to a GitHub Release.
3. Check that the bundle has no `eval(`: unzip and run `grep -c 'eval(' Controls/*/bundle.js`; the result must be `0`.
4. Add `docs/releases/v2-1-0-X_app_setup.txt`, the import and app-update checklist for that version.

> [!NOTE]
> In `ControlManifest.Input.xml`, the `description-key` and `display-name-key` attributes must not contain apostrophes or double quotes (the schema rejects them and the solution import fails). Rephrase instead.

## Troubleshooting

| Symptom | Likely cause and fix |
| --- | --- |
| Map shows `401` / blank in the test harness | Properties still hold the harness placeholder `val`, the key is wrong, or `mapDomain` does not match the Maps account's cloud. |
| Map shows "Auth Function URL must be a valid https:// URL" | The URL is not `https://`. Use the Function's https address. |
| `401 InvalidClientId` in SAS mode | The `x-ms-client-id` header is missing or wrong. The control sends it using `clientId` from the Function's JSON response, or the token's `iss` claim if `clientId` is absent. Use a current version of the control and make sure that value is the Maps account's Client ID. |
| `Failed to fetch` in SAS mode | Almost always CORS on the Function App (or a wrong Function host name), not the token. Add the origin your canvas app runs from. A failed first token request is retried and the map rebuilds itself up to 3 times, so one-off network hiccups no longer show this error. If it stays on screen, the cause is real. |
| Map loads on `atlas.microsoft.com` in a Government tenant | `mapDomain` is blank or misspelled; use exactly `atlas.azure.us`. |
| Re-imported solution but nothing changed | The managed solution ignores a re-import of the same version. Bump the version numbers (see Building From Source). After importing a newer version, open the app in the designer and accept the prompt to update the code component. |
| Solution import fails on a `*-key` attribute | An apostrophe or double quote slipped into manifest text. |
| `OnChange` runs but nothing happens | `OnChange` also fires for pin selection and lookup results, which do not change `actionToken`, so the guard skips them by design. For a real add, delete, open or filter, check that your `Switch` has a branch for that `pendingAction` value and that `varLastToken` is initialized in `App.OnStart` as shown above. |
| Address fields are blank | No address exists at that point (open water, remote area), or the lookup failed. For on-demand lookups, check `lookupStatus`: `"pending"` means the lookup is still running, `"done"` with blank fields means no address was found, `"failed"` means the request errored, and blank means no lookup is active because `lookupLatitude` or `lookupLongitude` is blank (blank inputs clear the `lookup*` outputs and `lookupStatus`). |
| Lookup ignored when set to the same coordinates | Set the lookup inputs to `Blank()` first, then to the coordinates again. |
| Map shows `Map failed (authenticate (SAS)): Auth function failed (<status> ...)` | The token Function answered with that HTTP status. Only the status is shown (the response body is never displayed or logged), so check the Function's own logs. `408`, `429` and `5xx` responses are retried first, so a persistent one appears after a few seconds. |
| Clicking a cluster only zooms in | `showClusterFilter` is off (the default). Set it to `true` to also get `pendingAction = "filter"` with `pendingRecordIds` (every project in the cluster) in `OnChange`. |
| The map control's own `OnSelect` never runs | The control never raises `OnSelect`, even though Studio lists it. Put your logic in `OnChange`: read `selectedRecordId` for pin selection and `pendingAction` for add, delete, open and filter. |

## Acknowledgments
This project is derived from [Grayson-Bishop/AzureMapsPCF](https://github.com/Grayson-Bishop/AzureMapsPCF) by Grayson Bishop, and is released under the license in [`LICENSE`](LICENSE).
