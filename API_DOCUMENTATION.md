# script.trakt - Complete API and Implementation Documentation

## Table of Contents
- [Overview](#overview)
- [Architecture](#architecture)
- [Installation & Setup](#installation--setup)
- [Core Components](#core-components)
- [API Reference](#api-reference)
- [Data Structures](#data-structures)
- [Integration Guide](#integration-guide)
- [Configuration](#configuration)
- [Advanced Usage](#advanced-usage)
- [Security Considerations](#security-considerations)
- [Troubleshooting](#troubleshooting)

---

## Overview

### Project Description
**script.trakt** is a comprehensive Kodi addon that provides seamless integration with Trakt.tv, enabling automatic scrobbling, library synchronization, and media tracking for TV shows and movies.

### Key Features
- **Automatic Scrobbling**: Real-time tracking of watched TV episodes and movies
- **Library Synchronization**: Bi-directional sync between Kodi and Trakt.tv
- **Collection Management**: Automatic collection updates and cleanup
- **Playback State Sync**: Synchronize watched statuses and playback progress
- **Rating System**: Rate movies and episodes directly from Kodi
- **Multi-Episode Support**: Handle multi-part episodes intelligently
- **PVR Support**: Support for live TV and PVR recordings

### Version Information
- **Current Version**: 3.8.0
- **Minimum Kodi Version**: Python 3.0.0+
- **License**: GPL-2.0-only

### Dependencies
```xml
<requires>
    <import addon="xbmc.python" version="3.0.0"/>
    <import addon="script.module.trakt" version="4.4.0"/>
    <import addon="script.module.dateutil" version="2.8.1"/>
</requires>
```

---

## Architecture

### System Design

```
┌─────────────────────────────────────────────────────────────┐
│                         Kodi Core                            │
│                    (JSON-RPC Interface)                      │
└──────────────────┬──────────────────────────────────────────┘
                   │
                   ├─────────────────────────────────────┐
                   │                                     │
┌──────────────────▼──────────┐      ┌─────────────────▼──────┐
│      Service Module          │      │   Script Module         │
│   (Background Service)       │      │  (User Actions)         │
│  - Scrobbler                 │      │  - Manual Sync          │
│  - Auto Sync                 │      │  - Rating               │
│  - Event Monitoring          │      │  - Context Menu         │
└──────────────┬───────────────┘      └──────────┬──────────────┘
               │                                 │
               └────────┬────────────────────────┘
                        │
            ┌───────────▼────────────┐
            │   Trakt API Module     │
            │  - Authentication      │
            │  - OAuth Management    │
            │  - API Calls           │
            └───────────┬────────────┘
                        │
            ┌───────────▼────────────┐
            │      Trakt.tv API      │
            │   (REST Endpoints)     │
            └────────────────────────┘
```

### Module Structure

```
script.trakt/
├── default.py              # Service entry point
├── defaultscript.py        # Script entry point
├── addon.xml              # Addon metadata and configuration
├── resources/
│   ├── lib/
│   │   ├── service.py            # Main service orchestrator
│   │   ├── scrobbler.py          # Scrobbling logic
│   │   ├── sync.py               # Synchronization coordinator
│   │   ├── syncEpisodes.py       # Episode sync implementation
│   │   ├── syncMovies.py         # Movie sync implementation
│   │   ├── traktapi.py           # Trakt API wrapper
│   │   ├── rating.py             # Rating functionality
│   │   ├── utilities.py          # Helper utilities
│   │   ├── kodiUtilities.py      # Kodi-specific utilities
│   │   ├── script.py             # Script command handler
│   │   ├── traktContextMenu.py   # Context menu implementation
│   │   ├── sqlitequeue.py        # Queue management
│   │   ├── deviceAuthDialog.py   # OAuth dialog
│   │   ├── kodilogging.py        # Logging configuration
│   │   └── globals.py            # Global state
│   ├── settings.xml         # Addon settings schema
│   ├── language/           # Localization files
│   └── skins/              # UI skins
└── tests/                  # Unit tests
```

---

## Installation & Setup

### Installation Methods

#### Method 1: Official Kodi Repository (Recommended)
1. Open Kodi
2. Navigate to **Add-ons** → **Install from repository** → **Kodi Add-on repository**
3. Go to **Services** → **Trakt**
4. Click **Install**

#### Method 2: Manual Installation (Development)
```bash
# Download and install from zip
1. Download the latest release
2. In Kodi: Settings → Add-ons → Install from zip file
3. Select the downloaded zip file

# Or clone to addons directory
cd ~/.kodi/addons/
git clone https://github.com/trakt/script.trakt.git
```

### Initial Configuration

1. **Enable the Addon**:
   - Navigate to **Settings** → **Add-ons** → **Enabled add-ons** → **Services** → **Trakt**

2. **Authenticate**:
   ```python
   # Get PIN from https://trakt.tv/pin/999
   # Enter in addon settings
   ```

3. **Configure Settings**:
   - General settings (scrobbling, sync)
   - Exclusions (Live TV, HTTP, plugins)
   - Notification preferences
   - Sync timing

---

## Core Components

### 1. Service Module (`service.py`)

The main orchestrator running as a background service.

#### Key Class: `traktService`

```python
class traktService:
    """
    Main service that handles all background operations.
    
    Attributes:
        scrobbler: Scrobbler instance for tracking playback
        syncThread: Thread for synchronization operations
        dispatchQueue: Queue for action dispatching
    """
    
    def __init__(self):
        """Initialize the service with necessary components."""
        threading.Thread.name = "trakt"
    
    def run(self):
        """
        Main service loop.
        
        Features:
        - Startup delay configuration
        - Queue processing
        - Monitor integration
        - Event handling
        """
```

#### Event Handling

```python
def _dispatch(self, data):
    """
    Dispatch actions based on event type.
    
    Supported Actions:
    - started: Playback started
    - ended/stopped: Playback ended
    - paused: Playback paused
    - resumed: Playback resumed
    - seek/seekchapter: Seeking
    - scanFinished: Library scan completed
    - databaseCleaned: Database cleanup completed
    - markWatched: Manual watched status update
    - manualRating: Manual rating
    - addtowatchlist: Add to watchlist
    - manualSync: Manual synchronization
    - settings: Open settings dialog
    - auth_info: Authentication dialog
    
    Args:
        data (dict): Action data with 'action' key and parameters
    """
```

### 2. Scrobbler Module (`scrobbler.py`)

Handles real-time tracking of media playback.

#### Key Class: `Scrobbler`

```python
class Scrobbler:
    """
    Manages scrobbling of currently playing media.
    
    Attributes:
        isPlaying (bool): Current playback state
        isPaused (bool): Pause state
        isPVR (bool): Whether playing PVR content
        isMultiPartEpisode (bool): Multi-part episode flag
        videoDuration (int): Total duration in seconds
        watchedTime (int): Current watch time in seconds
        curVideoInfo (dict): Current media information
        videosToRate (list): Queue of videos to rate
    """
    
    def __init__(self, api):
        """
        Initialize scrobbler with API instance.
        
        Args:
            api: traktAPI instance
        """
        self.traktapi = api
    
    def playbackStarted(self, data):
        """
        Called when playback starts.
        
        Args:
            data (dict): Playback data including media info
        """
    
    def playbackEnded(self):
        """Handle playback end, trigger scrobble completion."""
    
    def playbackPaused(self):
        """Handle playback pause."""
    
    def playbackResumed(self):
        """Handle playback resume."""
    
    def playbackSeek(self):
        """Handle seek events."""
```

#### Scrobbling States

```python
# Scrobble actions sent to Trakt
SCROBBLE_STATES = {
    'start': 'Begin watching',      # Sent at playback start
    'pause': 'Paused',               # Sent when paused
    'stop': 'Completed/stopped'      # Sent at end or stop
}

# Progress calculation
def __calculateWatchedPercent(self):
    """
    Calculate watched percentage.
    
    Returns:
        float: Percentage watched (0-100)
    
    Formula:
        (watchedTime / videoDuration) * 100
    """
```

#### Multi-Part Episode Handling

```python
def _currentEpisode(self, watchedPercent, episodeCount):
    """
    Determine current episode in multi-part.
    
    Args:
        watchedPercent (float): Overall watch progress
        episodeCount (int): Total episode count
    
    Returns:
        int: Current episode index (0-based)
    """
    split = 100 / episodeCount
    for i in range(episodeCount - 1, 0, -1):
        if watchedPercent >= (i * split):
            return i
    return 0
```

### 3. Synchronization Module (`sync.py`)

Coordinates library synchronization between Kodi and Trakt.

#### Key Class: `Sync`

```python
class Sync:
    """
    Handles synchronization operations.
    
    Args:
        show_progress (bool): Display progress dialog
        run_silent (bool): Silent mode (no dialogs)
        library (str): Library to sync ('all', 'movies', 'episodes')
        api: traktAPI instance
    """
    
    def __init__(self, show_progress=False, run_silent=False, 
                 library="all", api=None):
        self.traktapi = api
        self.show_progress = show_progress
        self.run_silent = run_silent
        self.library = library
    
    def sync(self):
        """
        Execute synchronization.
        
        Process:
        1. Check sync settings
        2. Sync movies (if enabled)
        3. Sync episodes (if enabled)
        4. Update progress display
        """
```

#### Sync Configuration Checks

```python
def __syncCheck(self, media_type):
    """
    Determine if sync is needed for media type.
    
    Checks:
    - Collection sync (add/clean)
    - Watched status sync
    - Playback progress sync
    - Ratings sync
    
    Args:
        media_type (str): 'movies' or 'episodes'
    
    Returns:
        bool: True if any sync is enabled
    """
```

### 4. Trakt API Module (`traktapi.py`)

Wrapper for Trakt.tv API interactions.

#### Key Class: `traktAPI`

```python
class traktAPI:
    """
    Trakt API wrapper with OAuth authentication.
    
    Attributes:
        __client_id: Application client ID
        __client_secret: Application client secret
        authorization: OAuth tokens
    """
    
    __client_id = "d4161a7a106424551add171e5470112e4afdaf2438e6ef2fe0548edc75924868"
    __client_secret = "b5fcd7cb5d9bb963784d11bbf8535bc0d25d46225016191eb48e50792d2155c0"
    
    def __init__(self, force=False):
        """
        Initialize API with OAuth.
        
        Args:
            force (bool): Force new authentication
        
        Features:
        - Proxy configuration
        - Token refresh
        - Persistent authentication
        """
    
    def login(self):
        """
        Authenticate with Trakt.tv using OAuth device flow.
        
        Process:
        1. Request device code
        2. Display code to user
        3. Poll for authentication
        4. Store tokens on success
        """
```

#### OAuth Device Flow

```python
# Device authentication callbacks
def on_aborted(self):
    """Called when authentication is cancelled."""

def on_authenticated(self, authorization):
    """
    Called on successful authentication.
    
    Args:
        authorization (dict): OAuth tokens
    """

def on_expired(self):
    """Called when device code expires."""

def on_poll(self, callback):
    """Called during polling."""

def on_token_refreshed(self, authorization):
    """
    Called when access token is refreshed.
    
    Args:
        authorization (dict): New OAuth tokens
    """
```

### 5. Kodi Utilities Module (`kodiUtilities.py`)

Kodi-specific helper functions.

#### Settings Management

```python
def getSetting(setting):
    """
    Get addon setting value.
    
    Args:
        setting (str): Setting key
    
    Returns:
        str: Setting value (stripped)
    """

def setSetting(setting, value):
    """
    Set addon setting value.
    
    Args:
        setting (str): Setting key
        value: Setting value (converted to string)
    """

def getSettingAsBool(setting):
    """
    Get setting as boolean.
    
    Returns:
        bool: True if setting is 'true' (case-insensitive)
    """

def getSettingAsInt(setting):
    """Get setting as integer with error handling."""

def getSettingAsFloat(setting):
    """Get setting as float with error handling."""
```

#### Kodi JSON-RPC Interface

```python
def kodiJsonRequest(params):
    """
    Execute Kodi JSON-RPC request.
    
    Args:
        params (dict): RPC parameters with 'method' key
    
    Returns:
        dict: Response result or None on error
    
    Example:
        params = {
            'jsonrpc': '2.0',
            'method': 'VideoLibrary.GetMovies',
            'params': {'properties': ['title', 'year']},
            'id': 1
        }
        result = kodiJsonRequest(params)
    """
```

#### Media Information Retrieval

```python
def getMovieDetailsFromKodi(movieid, fields):
    """
    Get movie details from Kodi library.
    
    Args:
        movieid (int): Kodi movie database ID
        fields (list): Fields to retrieve
    
    Returns:
        dict: Movie details
    
    Available Fields:
        title, year, imdbnumber, rating, playcount, 
        lastplayed, file, uniqueid, runtime, etc.
    """

def getEpisodeDetailsFromKodi(episodeid, fields):
    """
    Get episode details from Kodi library.
    
    Args:
        episodeid (int): Kodi episode database ID
        fields (list): Fields to retrieve
    
    Returns:
        dict: Episode details
    
    Available Fields:
        showtitle, season, episode, title, tvshowid,
        playcount, lastplayed, file, uniqueid, runtime, etc.
    """

def getTVShowDetailsFromKodi(tvshowid, fields):
    """Get TV show details from Kodi library."""
```

#### Media Type Detection

```python
def getMediaType():
    """
    Detect current media type in Kodi.
    
    Returns:
        str: 'movie', 'episode', 'show', 'season', or None
    
    Uses Kodi InfoLabels:
        - ListItem.DBTYPE
        - Container.Content
    """
```

#### Exclusion Checks

```python
def checkExclusion(fullpath):
    """
    Check if path should be excluded from tracking.
    
    Args:
        fullpath (str): Full file path
    
    Returns:
        bool: True if excluded
    
    Exclusion Types:
        - Live TV (pvr://)
        - HTTP streams (http://, https://)
        - Plugins (plugin://)
        - Custom paths (user-defined regex)
    """

def checkScriptExclusion():
    """
    Check if current item has script.trakt.exclude property.
    
    Returns:
        bool: True if excluded via listitem property
    """
```

### 6. Utilities Module (`utilities.py`)

General utility functions for data processing.

#### Media Type Helpers

```python
def isMovie(type):
    """Check if type is 'movie'."""

def isEpisode(type):
    """Check if type is 'episode'."""

def isShow(type):
    """Check if type is 'show'."""

def isSeason(type):
    """Check if type is 'season'."""

def isValidMediaType(type):
    """Validate media type."""
```

#### Data Processing

```python
def chunks(list, n):
    """
    Split list into chunks of size n.
    
    Args:
        list: List to split
        n (int): Chunk size
    
    Returns:
        list: List of chunks
    
    Example:
        chunks([1,2,3,4,5], 2) → [[1,2], [3,4], [5]]
    """

def getFormattedItemName(type, info):
    """
    Format media item name for display.
    
    Args:
        type (str): Media type
        info (dict): Media information
    
    Returns:
        str: Formatted name
    
    Formats:
        - Movie: "Title (Year)"
        - Episode: "S02E05 - Episode Title"
        - Season: "Show Title - Season 2"
        - Show: "Show Title"
    """
```

#### Media Matching

```python
def findMediaObject(mediaObjectToMatch, listToSearch, matchByTitleAndYear):
    """
    Find matching media object in list.
    
    Matching Strategy:
    1. Match by IMDb ID
    2. Match by TMDb ID
    3. Match by TVDb ID (episodes)
    4. Match by title and year (fallback)
    
    Args:
        mediaObjectToMatch (dict): Object to find
        listToSearch (list): List to search
        matchByTitleAndYear (bool): Enable title/year matching
    
    Returns:
        dict: Matched object or None
    """

def findMovieMatchInList(movie, trakt_movies, matchByTitleAndYear=True):
    """Find movie in Trakt movie list."""

def findShowMatchInList(show, trakt_shows):
    """Find show in Trakt show list."""

def findSeasonMatchInList(season, trakt_seasons):
    """Find season in Trakt season list."""

def findEpisodeMatchInList(episode, trakt_episodes):
    """Find episode in Trakt episode list."""
```

#### Time Utilities

```python
def _to_sec(time_str):
    """
    Convert time string to seconds.
    
    Args:
        time_str (str): Time in "HH:MM:SS" format
    
    Returns:
        int: Total seconds
    
    Example:
        _to_sec("01:30:45") → 5445
    """

def isoDateToTimestamp(iso_date):
    """
    Convert ISO date string to Unix timestamp.
    
    Args:
        iso_date (str): ISO 8601 date string
    
    Returns:
        int: Unix timestamp
    """

def timestampToIsoDate(timestamp):
    """Convert Unix timestamp to ISO date string."""
```

---

## API Reference

### Trakt API Endpoints Used

#### Authentication
```
POST /oauth/device/code
POST /oauth/device/token
POST /oauth/token (refresh)
```

#### Scrobbling
```
POST /scrobble/start
POST /scrobble/pause
POST /scrobble/stop
```

#### Sync
```
GET  /sync/collection/movies
GET  /sync/collection/shows
POST /sync/collection
POST /sync/collection/remove

GET  /sync/watched/movies
GET  /sync/watched/shows
POST /sync/history
POST /sync/history/remove

GET  /sync/playback
POST /sync/playback/{id} (delete)

GET  /sync/ratings/movies
GET  /sync/ratings/shows
GET  /sync/ratings/episodes
POST /sync/ratings
POST /sync/ratings/remove
```

#### Watchlist
```
GET  /sync/watchlist/movies
GET  /sync/watchlist/shows
POST /sync/watchlist
POST /sync/watchlist/remove
```

### JSON-RPC API (Kodi)

#### Video Library Methods

```javascript
// Get all movies
{
    "jsonrpc": "2.0",
    "method": "VideoLibrary.GetMovies",
    "params": {
        "properties": [
            "title", "year", "imdbnumber", "playcount",
            "lastplayed", "file", "uniqueid", "runtime"
        ]
    },
    "id": 1
}

// Get all TV shows
{
    "jsonrpc": "2.0",
    "method": "VideoLibrary.GetTVShows",
    "params": {
        "properties": ["title", "year", "imdbnumber", "uniqueid"]
    },
    "id": 1
}

// Get episodes for a show
{
    "jsonrpc": "2.0",
    "method": "VideoLibrary.GetEpisodes",
    "params": {
        "tvshowid": 123,
        "properties": [
            "showtitle", "season", "episode", "title",
            "playcount", "lastplayed", "file", "uniqueid", "runtime"
        ]
    },
    "id": 1
}

// Mark as watched
{
    "jsonrpc": "2.0",
    "method": "VideoLibrary.SetMovieDetails",
    "params": {
        "movieid": 123,
        "playcount": 1
    },
    "id": 1
}

// Set playback position
{
    "jsonrpc": "2.0",
    "method": "VideoLibrary.SetMovieDetails",
    "params": {
        "movieid": 123,
        "resume": {
            "position": 1234.5,
            "total": 5400
        }
    },
    "id": 1
}
```

### Addon Script API

Execute addon actions via RunScript:

```python
# Manual sync
RunScript(script.trakt, action=sync)
RunScript(script.trakt, action=sync, silent=True)
RunScript(script.trakt, action=sync, library=movies)
RunScript(script.trakt, action=sync, library=episodes)

# Authentication
RunScript(script.trakt, action=auth_info)

# Context menu
RunScript(script.trakt, action=contextmenu)

# Rating
RunScript(script.trakt, action=rate, media_type=movie, dbid=123)
RunScript(script.trakt, action=unrate, media_type=episode, dbid=456)

# Toggle watched
RunScript(script.trakt, action=togglewatched, media_type=movie, dbid=123)

# Add to watchlist
RunScript(script.trakt, action=addtowatchlist, media_type=show, dbid=789)
```

---

## Data Structures

### Media Object Structure

#### Movie Object
```python
{
    'title': 'Movie Title',
    'year': 2023,
    'ids': {
        'trakt': 123456,
        'imdb': 'tt1234567',
        'tmdb': 98765
    },
    'playcount': 1,
    'plays': 1,
    'lastplayed': '2023-01-15 20:30:00',
    'collected': True,
    'collected_at': '2023-01-10T12:00:00.000Z',
    'runtime': 120,  # minutes
    'rating': 8,
    'rated_at': '2023-01-16T10:00:00.000Z'
}
```

#### Episode Object
```python
{
    'showtitle': 'TV Show Title',
    'season': 1,
    'episode': 5,
    'title': 'Episode Title',
    'tvshowid': 42,
    'ids': {
        'trakt': 789012,
        'tvdb': 654321,
        'imdb': 'tt7654321',
        'tmdb': 11111
    },
    'playcount': 1,
    'plays': 1,
    'lastplayed': '2023-01-15 21:00:00',
    'collected': True,
    'collected_at': '2023-01-10T12:00:00.000Z',
    'runtime': 45,  # minutes
    'rating': 9,
    'rated_at': '2023-01-16T10:00:00.000Z'
}
```

#### Show Object
```python
{
    'title': 'TV Show Title',
    'year': 2020,
    'ids': {
        'trakt': 345678,
        'tvdb': 987654,
        'imdb': 'tt9876543',
        'tmdb': 22222
    }
}
```

#### Season Object
```python
{
    'title': 'TV Show Title',
    'season': 1,
    'ids': {
        'trakt': 456789,
        'tvdb': 112233,
        'tmdb': 33333
    },
    'episodes': [...]  # List of episode objects
}
```

### Scrobble Data Structure

```python
{
    'action': 'start',  # or 'pause', 'stop'
    'movie': {
        'title': 'Movie Title',
        'year': 2023,
        'ids': {'imdb': 'tt1234567', 'tmdb': 98765}
    },
    # OR for episodes:
    'episode': {
        'season': 1,
        'number': 5,
        'ids': {'tvdb': 654321}
    },
    'show': {
        'title': 'Show Title',
        'ids': {'tvdb': 112233}
    },
    'progress': 45.5,  # Percentage watched (0-100)
    'app_version': '3.8.0',
    'app_date': '2023-01-15'
}
```

### Sync Data Structure

#### Collection Sync
```python
{
    'movies': [
        {
            'title': 'Movie Title',
            'year': 2023,
            'ids': {'imdb': 'tt1234567'},
            'collected_at': '2023-01-10T12:00:00.000Z'
        }
    ],
    'episodes': [
        {
            'ids': {'tvdb': 654321},
            'collected_at': '2023-01-10T12:00:00.000Z'
        }
    ]
}
```

#### Watched History Sync
```python
{
    'movies': [
        {
            'title': 'Movie Title',
            'year': 2023,
            'ids': {'imdb': 'tt1234567'},
            'watched_at': '2023-01-15T20:30:00.000Z'
        }
    ],
    'episodes': [
        {
            'ids': {'tvdb': 654321},
            'watched_at': '2023-01-15T21:00:00.000Z'
        }
    ]
}
```

---

## Integration Guide

### Using script.trakt in Your Addon

#### 1. Check if script.trakt is Installed

```python
import xbmc
import xbmcaddon

def is_trakt_installed():
    """Check if script.trakt is installed and enabled."""
    try:
        addon = xbmcaddon.Addon('script.trakt')
        return True
    except:
        return False

def is_trakt_authorized():
    """Check if user has authorized Trakt."""
    if not is_trakt_installed():
        return False
    
    try:
        addon = xbmcaddon.Addon('script.trakt')
        auth = addon.getSetting('authorization')
        return bool(auth)
    except:
        return False
```

#### 2. Trigger Manual Sync

```python
def trigger_trakt_sync(library='all', silent=False):
    """
    Trigger manual Trakt sync.
    
    Args:
        library (str): 'all', 'movies', or 'episodes'
        silent (bool): Run without dialogs
    """
    xbmc.executebuiltin(
        f'RunScript(script.trakt, action=sync, library={library}, silent={silent})'
    )

# Usage examples
trigger_trakt_sync()  # Sync everything with dialogs
trigger_trakt_sync(library='movies', silent=True)  # Sync movies silently
trigger_trakt_sync(library='episodes')  # Sync episodes with progress
```

#### 3. Exclude Items from Tracking

```python
def set_trakt_exclusion(listitem):
    """
    Mark a listitem to be excluded from Trakt tracking.
    
    Args:
        listitem: xbmcgui.ListItem object
    """
    listitem.setProperty('script.trakt.exclude', 'true')

# Example usage in addon
import xbmcgui

listitem = xbmcgui.ListItem('Movie Title')
set_trakt_exclusion(listitem)  # This won't be tracked by Trakt
```

#### 4. Manual Rating

```python
def rate_movie(movie_dbid):
    """
    Open rating dialog for a movie.
    
    Args:
        movie_dbid (int): Kodi database ID
    """
    xbmc.executebuiltin(
        f'RunScript(script.trakt, action=rate, media_type=movie, dbid={movie_dbid})'
    )

def rate_episode(episode_dbid):
    """
    Open rating dialog for an episode.
    
    Args:
        episode_dbid (int): Kodi database ID
    """
    xbmc.executebuiltin(
        f'RunScript(script.trakt, action=rate, media_type=episode, dbid={episode_dbid})'
    )
```

#### 5. Add to Watchlist

```python
def add_to_watchlist(media_type, dbid):
    """
    Add item to Trakt watchlist.
    
    Args:
        media_type (str): 'movie', 'show', 'season', 'episode'
        dbid (int): Kodi database ID
    """
    xbmc.executebuiltin(
        f'RunScript(script.trakt, action=addtowatchlist, media_type={media_type}, dbid={dbid})'
    )
```

#### 6. Monitor Trakt Events

```python
import xbmc
import json

class TraktMonitor(xbmc.Monitor):
    """Monitor for Trakt-related events."""
    
    def onNotification(self, sender, method, data):
        """
        Handle Kodi notifications.
        
        Relevant methods:
        - VideoLibrary.OnScanFinished
        - VideoLibrary.OnCleanFinished
        - VideoLibrary.OnUpdate
        """
        if method == 'VideoLibrary.OnScanFinished':
            # Library scan completed
            # script.trakt will auto-sync if configured
            pass
        
        elif method == 'VideoLibrary.OnUpdate':
            data_dict = json.loads(data)
            if data_dict.get('playcount'):
                # Item watched status changed
                pass

# Usage
monitor = TraktMonitor()
while not monitor.abortRequested():
    if monitor.waitForAbort(1):
        break
```

### Integration with Custom Players

```python
import xbmc
import xbmcgui

class CustomPlayer(xbmc.Player):
    """
    Custom player that works with script.trakt.
    
    Important: Ensure proper metadata is set for tracking.
    """
    
    def play_movie(self, title, year, imdb_id, tmdb_id):
        """Play a movie with Trakt-compatible metadata."""
        
        listitem = xbmcgui.ListItem(title)
        
        # Set video info
        listitem.setInfo('video', {
            'title': title,
            'year': year,
            'mediatype': 'movie'
        })
        
        # Set unique IDs for Trakt matching
        listitem.setUniqueIDs({
            'imdb': imdb_id,
            'tmdb': str(tmdb_id)
        })
        
        # Play the item
        self.play('http://example.com/movie.mp4', listitem)
    
    def play_episode(self, show_title, season, episode, 
                     episode_title, tvdb_id, imdb_id=None):
        """Play an episode with Trakt-compatible metadata."""
        
        listitem = xbmcgui.ListItem(episode_title)
        
        # Set video info
        listitem.setInfo('video', {
            'title': episode_title,
            'tvshowtitle': show_title,
            'season': season,
            'episode': episode,
            'mediatype': 'episode'
        })
        
        # Set unique IDs
        unique_ids = {'tvdb': str(tvdb_id)}
        if imdb_id:
            unique_ids['imdb'] = imdb_id
        listitem.setUniqueIDs(unique_ids)
        
        # Play the item
        self.play('http://example.com/episode.mp4', listitem)
```

---

## Configuration

### Settings Structure

#### General Settings
```xml
<category id="general" label="General">
    <setting id="user" type="string" label="Username" />
    <setting id="Auth_Info" type="action" label="Authorize" />
    <setting id="authorization" type="string" visible="false" />
    <setting id="startup_delay" type="integer" min="0" max="30" default="0" />
</category>
```

#### Scrobbling Settings
```xml
<category id="scrobbler" label="Scrobbler">
    <setting id="scrobble_movie" type="bool" default="true" />
    <setting id="scrobble_episode" type="bool" default="true" />
    <setting id="scrobble_minimum" type="integer" default="90" />
    <setting id="scrobble_fallback" type="bool" default="true" />
</category>
```

#### Sync Settings
```xml
<category id="sync" label="Synchronization">
    <setting id="sync_on_update" type="bool" default="false" />
    <setting id="add_movies_to_trakt" type="bool" default="true" />
    <setting id="add_episodes_to_trakt" type="bool" default="true" />
    <setting id="clean_trakt_movies" type="bool" default="false" />
    <setting id="clean_trakt_episodes" type="bool" default="false" />
    <setting id="kodi_movie_playcount" type="bool" default="true" />
    <setting id="kodi_episode_playcount" type="bool" default="true" />
    <setting id="trakt_movie_playcount" type="bool" default="true" />
    <setting id="trakt_episode_playcount" type="bool" default="true" />
    <setting id="trakt_movie_playback" type="bool" default="true" />
    <setting id="trakt_episode_playback" type="bool" default="true" />
</category>
```

#### Exclusion Settings
```xml
<category id="exclusions" label="Exclusions">
    <setting id="ExcludeLiveTV" type="bool" default="true" />
    <setting id="ExcludeHTTP" type="bool" default="false" />
    <setting id="ExcludePlugin" type="bool" default="false" />
    <setting id="ExcludePath" type="string" default="" />
</category>
```

#### Notification Settings
```xml
<category id="notifications" label="Notifications">
    <setting id="show_sync_notifications" type="bool" default="true" />
    <setting id="hide_notifications_playback" type="bool" default="false" />
    <setting id="show_rating_dialog" type="bool" default="true" />
</category>
```

### Programmatic Configuration

```python
import xbmcaddon

addon = xbmcaddon.Addon('script.trakt')

# Get settings
scrobble_enabled = addon.getSetting('scrobble_movie') == 'true'
sync_on_update = addon.getSetting('sync_on_update') == 'true'
startup_delay = int(addon.getSetting('startup_delay'))

# Set settings (requires elevated permissions in most cases)
addon.setSetting('scrobble_movie', 'true')
addon.setSetting('startup_delay', '10')
```

---

## Advanced Usage

### Custom Queue Implementation

The addon uses `sqlitequeue.py` for persistent queuing:

```python
from resources.lib.sqlitequeue import SqliteQueue

# Create queue
queue = SqliteQueue()

# Add items
queue.append({'action': 'sync', 'data': {...}})

# Process queue
while len(queue) > 0:
    item = queue.pop()
    # Process item
```

### Direct API Usage

```python
from resources.lib.traktapi import traktAPI

# Initialize API
api = traktAPI()

# Access underlying trakt.py library
from trakt import Trakt

# Get user's watched movies
with Trakt.configuration.oauth.from_response(api.authorization):
    watched_movies = Trakt['sync/watched'].movies()
    
    for movie in watched_movies:
        print(f"{movie.title} ({movie.year})")

# Add movie to collection
with Trakt.configuration.oauth.from_response(api.authorization):
    Trakt['sync/collection'].add({
        'movies': [
            {
                'title': 'Movie Title',
                'year': 2023,
                'ids': {'imdb': 'tt1234567'}
            }
        ]
    })
```

### Multi-Part Episode Detection

```python
from resources.lib import kodiUtilities

def check_multi_part(episode_data):
    """
    Check if episode is multi-part.
    
    Args:
        episode_data (dict): Episode data from Kodi
    
    Returns:
        dict: Enhanced data with multi-part info
    """
    if 'multi_episode_count' in episode_data:
        # Multi-part episode detected
        episode_count = episode_data['multi_episode_count']
        current_episode = episode_data.get('multi_episode_current', 0)
        
        return {
            'is_multi_part': True,
            'episode_count': episode_count,
            'current_episode': current_episode,
            'episodes': episode_data.get('multi_episode_data', [])
        }
    
    return {'is_multi_part': False}
```

### Custom Sync Strategy

```python
from resources.lib.sync import Sync
from resources.lib.traktapi import traktAPI

def custom_sync(library='all', progress_callback=None):
    """
    Implement custom sync logic.
    
    Args:
        library (str): Library to sync
        progress_callback (callable): Progress update callback
    """
    # Initialize API
    api = traktAPI()
    
    # Create sync instance
    sync = Sync(
        show_progress=True,
        run_silent=False,
        library=library,
        api=api
    )
    
    # Override progress update if needed
    if progress_callback:
        original_update = sync.UpdateProgress
        
        def wrapped_update(*args, **kwargs):
            original_update(*args, **kwargs)
            progress_callback(*args, **kwargs)
        
        sync.UpdateProgress = wrapped_update
    
    # Run sync
    sync.sync()
```

---

## Security Considerations

### OAuth Credentials

```python
# Credentials are stored in addon settings
# Location: ~/.kodi/userdata/addon_data/script.trakt/settings.xml

# Authorization format:
{
    'access_token': 'xxxxxxxxxx',
    'refresh_token': 'yyyyyyyyyy',
    'expires_in': 7776000,
    'created_at': 1234567890,
    'token_type': 'Bearer',
    'scope': 'public'
}
```

**Security Best Practices:**

1. **Never hardcode tokens** in your code
2. **Use the OAuth device flow** for authentication
3. **Respect token expiration** and refresh automatically
4. **Don't log sensitive data** (tokens, credentials)
5. **Use HTTPS** for all API communications

### Data Privacy

- **Local Data**: All sync data is stored locally in Kodi's database
- **Network Traffic**: All API calls use HTTPS
- **User Consent**: Authentication requires explicit user action
- **Minimal Data**: Only necessary metadata is transmitted

### Proxy Configuration

```python
from resources.lib.kodiUtilities import checkAndConfigureProxy

# Proxy is automatically configured from Kodi or addon settings
proxy_url = checkAndConfigureProxy()

# Override proxy for Trakt requests
from trakt import Trakt
if proxy_url:
    Trakt.http.proxies = {
        'http': proxy_url,
        'https': proxy_url
    }
```

---

## Troubleshooting

### Common Issues

#### 1. Authentication Failures

**Problem**: Cannot authenticate with Trakt.tv

**Solutions**:
```python
# Force re-authentication
import xbmc
xbmc.executebuiltin('RunScript(script.trakt, action=auth_info)')

# Check network connectivity
import urllib.request
try:
    urllib.request.urlopen('https://api.trakt.tv', timeout=5)
    print("Network OK")
except:
    print("Network Error")

# Check proxy settings
from resources.lib.kodiUtilities import checkAndConfigureProxy
proxy = checkAndConfigureProxy()
print(f"Proxy: {proxy}")
```

#### 2. Scrobbling Not Working

**Problem**: Playback not being tracked

**Checklist**:
```python
# 1. Check if scrobbling is enabled
addon = xbmcaddon.Addon('script.trakt')
scrobble_movie = addon.getSetting('scrobble_movie') == 'true'
scrobble_episode = addon.getSetting('scrobble_episode') == 'true'

# 2. Check if item is excluded
from resources.lib.kodiUtilities import checkExclusion, checkScriptExclusion
file_path = xbmc.Player().getPlayingFile()
is_excluded = checkExclusion(file_path)

# 3. Check metadata
# Ensure proper uniqueid is set (IMDb, TMDb, TVDb)
```

#### 3. Sync Issues

**Problem**: Library not syncing with Trakt

**Debug Steps**:
```python
# Enable debug logging
import xbmc
xbmc.executebuiltin('SetProperty(script.trakt.debug,true,Home)')

# Check sync settings
addon = xbmcaddon.Addon('script.trakt')
sync_on_update = addon.getSetting('sync_on_update')
add_movies = addon.getSetting('add_movies_to_trakt')
add_episodes = addon.getSetting('add_episodes_to_trakt')

# Manual sync with progress
xbmc.executebuiltin('RunScript(script.trakt, action=sync)')
```

#### 4. Log File Analysis

```python
# Enable debug mode
# Kodi Settings → System → Logging → Enable debug logging

# Enable addon debug
import xbmcaddon
addon = xbmcaddon.Addon('script.trakt')
# Add to settings.xml:
# <setting id="debug" type="bool" default="true" />

# Log location:
# Linux: ~/.kodi/temp/kodi.log
# Windows: %APPDATA%\Kodi\kodi.log
# macOS: ~/Library/Logs/kodi.log

# Search for Trakt entries:
# grep -i trakt ~/.kodi/temp/kodi.log
```

### Performance Optimization

#### Reduce API Calls

```python
# Chunk requests to reduce API calls
from resources.lib.utilities import chunks

movies = [...]  # Large list of movies
chunked_movies = chunks(movies, 100)  # Process 100 at a time

for chunk in chunked_movies:
    # Process chunk
    pass
```

#### Queue Management

```python
# Clear stuck queue items
from resources.lib.sqlitequeue import SqliteQueue

queue = SqliteQueue()
queue.clear()  # Clear all items
```

### Error Handling

```python
from resources.lib.utilities import createError
import logging
import xbmcgui

logger = logging.getLogger(__name__)

try:
    # Your code here
    pass
except Exception as ex:
    error_message = createError(ex)
    logger.error(error_message)
    # Show user-friendly error
    xbmcgui.Dialog().notification(
        'Trakt Error',
        'An error occurred. Check logs.',
        xbmcgui.NOTIFICATION_ERROR
    )
```

---

## Appendix

### A. Complete Settings Reference

| Setting ID | Type | Default | Description |
|------------|------|---------|-------------|
| `user` | string | "" | Trakt username (display only) |
| `authorization` | string | "" | OAuth tokens (JSON) |
| `startup_delay` | integer | 0 | Service startup delay (seconds) |
| `scrobble_movie` | bool | true | Enable movie scrobbling |
| `scrobble_episode` | bool | true | Enable episode scrobbling |
| `scrobble_minimum` | integer | 90 | Minimum watch percentage for scrobble |
| `sync_on_update` | bool | false | Auto-sync after library update |
| `add_movies_to_trakt` | bool | true | Add movies to Trakt collection |
| `add_episodes_to_trakt` | bool | true | Add episodes to Trakt collection |
| `clean_trakt_movies` | bool | false | Remove movies from Trakt not in Kodi |
| `clean_trakt_episodes` | bool | false | Remove episodes from Trakt not in Kodi |
| `kodi_movie_playcount` | bool | true | Update Kodi from Trakt watched status |
| `kodi_episode_playcount` | bool | true | Update Kodi from Trakt watched status |
| `trakt_movie_playback` | bool | true | Sync movie playback progress to Trakt |
| `trakt_episode_playback` | bool | true | Sync episode playback progress to Trakt |
| `ExcludeLiveTV` | bool | true | Exclude Live TV from tracking |
| `ExcludeHTTP` | bool | false | Exclude HTTP streams |
| `ExcludePlugin` | bool | false | Exclude plugin content |
| `show_sync_notifications` | bool | true | Show sync notifications |
| `show_rating_dialog` | bool | true | Show rating dialog after playback |

### B. Supported Media Identifiers

| ID Type | Media Types | Format | Example |
|---------|-------------|--------|---------|
| IMDb | Movies, Shows, Episodes | tt + 7 digits | tt1234567 |
| TMDb | Movies, Shows, Episodes | Integer | 12345 |
| TVDb | Shows, Episodes | Integer | 67890 |
| Trakt | All | Integer | 98765 |

### C. API Rate Limits

Trakt API rate limits:
- **Authentication**: 10 requests per minute
- **Scrobbling**: 1 request per 10 seconds per user
- **Sync**: 1000 requests per hour
- **General**: 1000 requests per 5 minutes

### D. Version History

| Version | Release Date | Key Changes |
|---------|--------------|-------------|
| 3.8.0 | 2023 | Add support for script.trakt.exclude property |
| 3.7.1 | 2023 | Fix title/year fallback |
| 3.7.0 | 2023 | Translation updates |
| 3.6.0 | 2023 | New settings format, proxy override |
| 3.5.0 | 2023 | Skin improvements, proxy config |

### E. Resources

- **Official Repository**: https://github.com/trakt/script.trakt
- **Trakt.tv API**: https://trakt.docs.apiary.io/
- **Kodi JSON-RPC**: https://kodi.wiki/view/JSON-RPC_API
- **Forum**: https://forum.kodi.tv/showthread.php?tid=220547
- **Translations**: https://kodi.weblate.cloud/

### F. Contributing

To contribute to this project:

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Run tests: `python -m pytest tests/`
5. Submit a pull request

**Note**: Translation pull requests should go through WebLate, not GitHub.

### G. License

```
script.trakt - Trakt.tv integration for Kodi
Copyright (C) 2023 Trakt.tv, Razzeee and contributors

This program is free software; you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation; either version 2 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.
```

---

## Quick Reference Card

### Essential Commands
```bash
# Manual sync
RunScript(script.trakt, action=sync)

# Silent sync
RunScript(script.trakt, action=sync, silent=True)

# Sync only movies
RunScript(script.trakt, action=sync, library=movies)

# Authenticate
RunScript(script.trakt, action=auth_info)

# Rate current item
RunScript(script.trakt, action=rate)
```

### Essential Functions
```python
# Check if item should be tracked
from resources.lib.kodiUtilities import checkExclusion
is_excluded = checkExclusion(filepath)

# Get media type
from resources.lib.kodiUtilities import getMediaType
media_type = getMediaType()  # 'movie', 'episode', 'show', 'season'

# Format display name
from resources.lib.utilities import getFormattedItemName
name = getFormattedItemName('movie', {'title': 'Title', 'year': 2023})
```

### Essential Imports
```python
from resources.lib.traktapi import traktAPI
from resources.lib.kodiUtilities import *
from resources.lib.utilities import *
from resources.lib.sync import Sync
from resources.lib.scrobbler import Scrobbler
```

---

**Document Version**: 1.0  
**Last Updated**: 2023  
**Maintainer**: Trakt.tv, Razzeee  
**Contact**: https://github.com/trakt/script.trakt

