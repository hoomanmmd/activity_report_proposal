# Activity Report Proposal

The requirements are divided into two categories: reports that can be tracked via the API, including:
·	The time spent on each application.
·	The applications they downloaded or deleted during the week.
·	How often applications are opened/closed.
·	The total number of notifications received.
 
And those that require a more complex approach:
·	Whether they beat any of their high scores in their favorite games.
·	The total amount of video content they consumed, across all applications.
·	The product for which they saw the most advertising, across all applications.

# API based reports
This part can be implemented with 100% accuracy, as the required features are exposed through well-defined APIs.

**Android**

-   **`UsageStatsManager`**  gives per-app foreground time, launch counts, and session start/stop events.
-   **`AccessibilityService`** see which screen is showing in another app and read some on-screen text.

# Non API based reports
The idea is to build a pipeline that uses accessibility features to detect and extract content from the screen.

## Restrictions
- **Data privacy:** Since the app may handle highly sensitive data, the preferred approach is to process all data locally on the device.
- **Accuracy:** Accuracy is lower due to the nature of non-API-based reporting. However, it can be improved over time by fine-tuning the processing pipeline for each individual app.
- **Limitations:** Not all apps implement accessibility features, so this may not work across all apps.

## Proposed architecture

```
                    ┌──────────────────────────┐
                    │       Mobile App         │
                    └────────────┬─────────────┘
                                 │
              ┌──────────────────┴──────────────────┐
              │                                     │
       ┌──────▼──────┐                       ┌──────▼──────┐
       │   Android   │                       │    iOS      │
       │ Monitoring  │                       │ Monitoring  │
       │   Pipeline  │                       │  Pipeline   │
       └──────┬──────┘                       └──────┬──────┘
              │                                     │
       Accessibility /                       DeviceActivity /
       UI observation                         Screen Time
              │                                     │
              └──────────────────┬──────────────────┘
                                 │
                         ┌───────▼────────┐
                         │ Local Event DB │
                         │    SQLite      │
                         └───────┬────────┘
                                 │
                         ┌───────▼────────┐
                         │ Sync / Backend │
                         └────────────────┘
```

### 1. Shared application layer
Since a significant part of the app is the UI, which is shared across platforms, **Flutter** is a good fit for sharing both the UI and the application layer.

But the actual monitoring code should be **native Kotlin on Android and Swift on iOS**, exposed to Flutter through platform channels.

----------

# Android architecture

An Android Accessibility Service can run in the background and receive UI events from other applications. With appropriate configuration, it can inspect accessibility nodes and their text/content. Android specifically supports filtering events by package name and retrieving window content.

So the architecture could look like:

```
Instagram
   │
   ▼
Android UI
   │
   ▼
AccessibilityService
   │
   ├── package = Instagram
   ├── UI changes
   ├── text / accessibility nodes
   ├── scrolling events
   └── window changes
           │
           ▼
     Video Detector
           │
           ├── creator
           ├── caption
           ├── identifiable text
           ├── possible URL
           └── video transition
                   │
                   ▼
             Session Tracker
                   │
             ┌─────┴─────┐
             │           │
        video_start   video_end
             │           │
             └─────┬─────┘
                   ▼
             Local Database
```

----------

# Data architecture
```
apps
-------------------------
id
package_name
platform
display_name


video_sessions
-------------------------
id
app_id
video_key
creator
title
url
started_at
ended_at
duration_seconds
confidence
source


monitoring_sessions
-------------------------
id
app_id
started_at
ended_at


scores
-------------------------
id
app_id
value
timestamp

viewed_products
-------------------------
id
app_id
value
timestamp


events
-------------------------
id
timestamp
app
event_type
payload
```


Stack would be:

-   **Flutter** — UI
-   **Kotlin** — Android monitoring
-   **Swift** — iOS monitoring
-   **SQLite** — local data
----------
# A More Complex Approach
This part can also be implemented using screen capture combined with computer vision and OCR. This approach is particularly useful when accessibility information is insufficient or unavailable.

## Restrictions
- **Device performance:** Low- and mid-range devices have limited processing capabilities. To accommodate these devices, the processing frequency can be adjusted, which may reduce overall accuracy.
- **Platform restrictions:** Google Play and the App Store might restrict this type of usage, making this approach more suitable for personal use or side-loaded applications.

# Privacy

This application would potentially have access to extremely sensitive behavioral information.

So I'd design it as local-first and give users:

-   pause monitoring
-   exclude applications
-   delete individual records
-   delete all history
-   export history
-   encryption
-   clear indication of what is being collected


