# Database Schema

**Related**: [[00-index]] | [[05-version-compatibility]] | [[06-organizational-structures]]

## Overview

The Photos.sqlite database is a Core Data-managed SQLite database containing all metadata for photos, albums, faces, and organizational structures. The schema varies significantly by Photos version.

## Database Files

### Main Database: Photos.sqlite

- **Location**: `database/Photos.sqlite`
- **Format**: SQLite 3
- **Journal Mode**: WAL (Write-Ahead Logging)
- **Encoding**: UTF-8
- **Access**: Read-only recommended

### Companion Files

- `Photos.sqlite-wal`: Write-ahead log
- `Photos.sqlite-shm`: Shared memory file
- Both must be present when accessing an active database

### Search Database: psi.sqlite

- **Location**: `database/search/psi.sqlite`
- **Purpose**: Machine learning labels and search indexing
- **Availability**: Photos 5+

## Core Data Conventions

### Naming

- Tables prefixed with `Z`: `ZASSET`, `ZGENERICALBUM`
- Primary keys: `Z_PK` (integer)
- Foreign keys: Column name matches referenced table
- Join tables: `Z_[N]TABLENAME` where N is a number

### Metadata Table

Every Core Data database has a `Z_METADATA` table:

```sql
CREATE TABLE Z_METADATA (
    Z_VERSION INTEGER PRIMARY KEY,
    Z_UUID VARCHAR(255),
    Z_PLIST BLOB
);
```

The `Z_PLIST` column contains a binary property list with:
- `PLModelVersion`: Integer model version for version detection

## Asset Tables

The primary asset/photo table name **varies by version**:

| Photos Version | macOS Version | Asset Table |
|----------------|---------------|-------------|
| Photos 5 | 10.15 | ZGENERICASSET |
| Photos 6+ | 11.0+ | ZASSET |

### ZGENERICASSET (Photos 5)

```sql
CREATE TABLE ZGENERICASSET (
    Z_PK INTEGER PRIMARY KEY,
    Z_ENT INTEGER,
    Z_OPT INTEGER,
    ZUUID VARCHAR,
    ZDIRECTORY VARCHAR,
    ZFILENAME VARCHAR,

    -- Asset Type
    ZKIND INTEGER,                    -- 0=photo, 1=video
    ZKINDSUBTYPE INTEGER,             -- See Kind Subtypes below
    ZUNIFORMTYPEIDENTIFIER VARCHAR,   -- UTI of file

    -- Dates (Core Data timestamp: seconds since 2001-01-01)
    ZDATECREATED TIMESTAMP,
    ZMODIFICATIONDATE TIMESTAMP,
    ZADDEDDATE TIMESTAMP,
    ZTRASHEDDATE TIMESTAMP,

    -- Dimensions
    ZHEIGHT INTEGER,
    ZWIDTH INTEGER,
    ZORIENTATION INTEGER,             -- EXIF orientation

    -- Location
    ZLATITUDE FLOAT,                  -- -180.0 = null
    ZLONGITUDE FLOAT,                 -- -180.0 = null

    -- Flags
    ZHIDDEN INTEGER,                  -- 0=visible, 1=hidden
    ZFAVORITE INTEGER,                -- 0=not favorite, 1=favorite
    ZTRASHEDSTATE INTEGER,            -- 0=not trashed, 1=trashed
    ZVISIBILITYSTATE INTEGER,         -- 0=visible, 2=burst non-selected
    ZHASADJUSTMENTS INTEGER,          -- 0=no edits, 1=edited

    -- Burst Photos
    ZAVALANCHEUUID VARCHAR,           -- Burst set UUID
    ZAVALANCHEPICKTYPE INTEGER,       -- See Burst Pick Types below

    -- Cloud/Sharing
    ZCLOUDASSETGUID VARCHAR,          -- iCloud asset GUID (null if not in cloud)
    ZCLOUDBATCHPUBLISHDATE TIMESTAMP, -- Non-null if shared
    ZCLOUDLOCALSTATE INTEGER,
    ZCLOUDOWNERHASHEDPERSONID VARCHAR,

    -- Import
    ZSAVEDASSETTYPE INTEGER,          -- See Saved Asset Types below

    -- Relationships
    ZMOMENT INTEGER,                  -- FK to ZMOMENT
    ZMOMENTSHARE INTEGER,             -- FK to ZSHARE
    ZMASTERFINGERPRINT VARCHAR,       -- Deprecated in Photos 10+

    -- Adjustments
    ZADJUSTMENTTIMESTAMP TIMESTAMP,

    -- Other
    ZDURATION FLOAT,                  -- Video duration in seconds
    ZVIDEOCPDISPLAYVALUE INTEGER,
    ZVIDEOCPDURATIONVALUE INTEGER
);

CREATE INDEX ZGENERICASSET_UUID_INDEX ON ZGENERICASSET(ZUUID);
CREATE INDEX ZGENERICASSET_KIND_INDEX ON ZGENERICASSET(ZKIND);
CREATE INDEX ZGENERICASSET_DIRECTORY_INDEX ON ZGENERICASSET(ZDIRECTORY);
```

### ZASSET (Photos 6+)

Similar structure to ZGENERICASSET with some differences:

```sql
CREATE TABLE ZASSET (
    Z_PK INTEGER PRIMARY KEY,
    Z_ENT INTEGER,
    Z_OPT INTEGER,
    ZUUID VARCHAR,
    ZDIRECTORY VARCHAR,
    ZFILENAME VARCHAR,

    -- Most columns same as ZGENERICASSET
    ZKIND INTEGER,
    ZKINDSUBTYPE INTEGER,
    ZUNIFORMTYPEIDENTIFIER VARCHAR,

    -- Dates
    ZDATECREATED TIMESTAMP,
    ZMODIFICATIONDATE TIMESTAMP,
    ZADDEDDATE TIMESTAMP,
    ZTRASHEDDATE TIMESTAMP,

    -- Dimensions
    ZHEIGHT INTEGER,
    ZWIDTH INTEGER,
    ZORIENTATION INTEGER,

    -- Location
    ZLATITUDE FLOAT,
    ZLONGITUDE FLOAT,

    -- Flags
    ZHIDDEN INTEGER,
    ZFAVORITE INTEGER,
    ZTRASHEDSTATE INTEGER,
    ZVISIBILITYSTATE INTEGER,

    -- Adjustments (Photos 6-9)
    ZHASADJUSTMENTS INTEGER,          -- Removed in Photos 10
    ZADJUSTMENTTIMESTAMP TIMESTAMP,

    -- Adjustments (Photos 10+)
    ZADJUSTMENTSSTATE INTEGER,        -- Replaces ZHASADJUSTMENTS

    -- Burst
    ZAVALANCHEUUID VARCHAR,
    ZAVALANCHEPICKTYPE INTEGER,

    -- Cloud
    ZCLOUDASSETGUID VARCHAR,
    ZCLOUDBATCHPUBLISHDATE TIMESTAMP,
    ZCLOUDLOCALSTATE INTEGER,
    ZCLOUDOWNERHASHEDPERSONID VARCHAR,

    -- Import
    ZSAVEDASSETTYPE INTEGER,

    -- Relationships
    ZMOMENT INTEGER,
    ZMOMENTSHARE INTEGER,

    -- Other
    ZDURATION FLOAT,
    ZPACKEDACCEPTABLECROPRE CT BLOB,
    ZPACKEDPREFERREDCROPRECT BLOB
);

CREATE INDEX ZASSET_UUID_INDEX ON ZASSET(ZUUID);
CREATE INDEX ZASSET_KIND_INDEX ON ZASSET(ZKIND);
```

### Kind Values

```python
ZKIND = {
    0: "photo",
    1: "video"
}
```

### Kind Subtypes

```python
ZKINDSUBTYPE = {
    0: "normal",
    1: "panorama",
    2: "live_photo",
    10: "screenshot",
    100: "hdr",
    101: "slow_motion",
    102: "time_lapse",
    103: "screen_recording",
    110: "portrait"
}
```

### Burst Pick Types

```python
ZAVALANCHEPICKTYPE = {
    0: "none",           # Not in a burst
    2: "not_selected",   # In burst, not selected
    4: "default",        # Default pick
    8: "user_selected",  # User selected
    16: "key_photo"      # Key/representative photo
}
```

### Saved Asset Types

```python
ZSAVEDASSETTYPE = {
    0: "unknown",
    1: "alternate",
    2: "cloud_shared",
    3: "copied",         # Copied into library
    4: "cloud_shared_album",
    6: "imported_from_cloud",
    8: "guest_asset",
    10: "referenced"     # Referenced file (not copied)
}
```

## Extended Attributes Table

### ZADDITIONALASSETATTRIBUTES

Extended metadata for each asset:

```sql
CREATE TABLE ZADDITIONALASSETATTRIBUTES (
    Z_PK INTEGER PRIMARY KEY,
    Z_ENT INTEGER,
    Z_OPT INTEGER,

    -- Relationship
    ZASSET INTEGER,                    -- FK to ZGENERICASSET/ZASSET

    -- Titles and Descriptions
    ZTITLE VARCHAR,
    ZASSETDESCRIPTION INTEGER,         -- FK to ZASSETDESCRIPTION

    -- Original File Info
    ZORIGINALFILENAME VARCHAR,
    ZORIGINALFILESIZE INTEGER,         -- Bytes
    ZORIGINALHEIGHT INTEGER,
    ZORIGINALWIDTH INTEGER,
    ZORIGINALORIENTATION INTEGER,
    ZUNIFORMTYPEIDENTIFIER VARCHAR,

    -- Timezone
    ZTIMEZONEOFFSET INTEGER,           -- Seconds
    ZINFERREDTIMEZONEOFFSET INTEGER,   -- Seconds
    ZTIMEZONENAME VARCHAR,

    -- Camera/Capture
    ZCAMERACAPTUREDEVICE INTEGER,      -- 1=front camera (selfie)
    ZCAMERAMAKE VARCHAR,
    ZCAMERAMODEL VARCHAR,
    ZLENSMODEL VARCHAR,

    -- Location
    ZREVERSELOCATIONDATA BLOB,         -- Binary plist with place names
    ZREVERSELOCATIONDATAISVALID INTEGER,
    ZSHIFTEDLOCATIONISVALID INTEGER,

    -- Import Info (Photos 7+)
    ZIMPORTEDBYDISPLAYNAME VARCHAR,    -- App that imported
    ZIMPORTEDBYBUNDLEIDENTIFIER VARCHAR,
    ZIMPORTEDBYBADGES INTEGER,

    -- RAW Handling
    ZORIGINALRESOURCECHOICE INTEGER,   -- 0=JPEG primary, 1=RAW primary

    -- Fingerprints (version-dependent)
    ZMASTERFINGERPRINT VARCHAR,        -- Photos 5-9
    ZORIGINALSTABLEHASH INTEGER,       -- Photos 10+

    -- HDR
    ZHDRTYPE INTEGER,                  -- Photos 7+, replaces ZCUSTOMRENDEREDVALUE

    -- Editing
    ZUNMANAGEDADJUSTMENT INTEGER,      -- FK to ZUNMANAGEDADJUSTMENT

    -- Other
    ZEXIFTIMESTAMPSTRING VARCHAR,
    ZAVALANCHEAUTOPICKIDENTIFIER VARCHAR,
    ZORIGINALRESOURCECHOICE INTEGER
);

CREATE INDEX ZADDITIONALASSETATTRIBUTES_ASSET ON ZADDITIONALASSETATTRIBUTES(ZASSET);
```

### ZASSETDESCRIPTION

Long descriptions for assets:

```sql
CREATE TABLE ZASSETDESCRIPTION (
    Z_PK INTEGER PRIMARY KEY,
    Z_ENT INTEGER,
    Z_OPT INTEGER,
    ZLONGDESCRIPTION VARCHAR,
    ZASSETATTRIBUTES INTEGER          -- FK to ZADDITIONALASSETATTRIBUTES
);
```

### Reverse Location Data Format

The `ZREVERSELOCATIONDATA` blob contains a binary plist with structure:

```python
{
    'placeNames': [
        'Country',        # Index 0
        'State/Province', # Index 1
        'City',          # Index 2
        'SubLocality',   # Index 3
        'Street',        # Index 4
        'POI'            # Index 5 (Point of Interest)
    ]
}
```

## Resource Table

### ZINTERNALRESOURCE

File resources associated with assets:

```sql
CREATE TABLE ZINTERNALRESOURCE (
    Z_PK INTEGER PRIMARY KEY,
    Z_ENT INTEGER,
    Z_OPT INTEGER,

    -- Relationship
    ZASSET INTEGER,                    -- FK to ZGENERICASSET/ZASSET
    ZFINGERPRINT VARCHAR,

    -- File Info
    ZUNIFORMTYPEIDENTIFIER VARCHAR,
    ZCOMPACTUTI VARCHAR,
    ZDATALENGTH INTEGER,               -- File size in bytes
    ZDATASTORESUBTYPE INTEGER,         -- See Data Store Subtypes
    ZRESOURCETYPE INTEGER,             -- Resource type code

    -- Availability
    ZLOCALAVAILABILITY INTEGER,        -- 1=available locally
    ZREMOTEAVAILABILITY INTEGER,       -- 1=available in cloud
    ZTRASHEDSTATE INTEGER,

    -- File System
    ZFILESYSTEMBOOKMARK BLOB,          -- Bookmark to file location
    ZFILESYSTEMVOLUME INTEGER,         -- FK to ZFILESYSTEMVOLUME

    -- Cloud
    ZCLOUDLOCALSTATE INTEGER,
    ZCLOUDSOURCETYPE INTEGER,

    -- RAW
    ZRECIPEID INTEGER,

    -- Version
    ZVERSION INTEGER,
    ZUNORIENTEDHEIGHT INTEGER,
    ZUNORIENTEDWIDTH INTEGER,

    -- Other
    ZCODECFOURCHARCODENAME VARCHAR,
    ZCLOUDBATCHID VARCHAR
);

CREATE INDEX ZINTERNALRESOURCE_ASSET ON ZINTERNALRESOURCE(ZASSET);
CREATE INDEX ZINTERNALRESOURCE_FINGERPRINT ON ZINTERNALRESOURCE(ZFINGERPRINT);
```

### Data Store Subtypes

```python
ZDATASTORESUBTYPE = {
    0: "unknown",
    1: "original",               # Original imported file
    2: "alternative",
    3: "full_size_render",       # Full-size edited version
    4: "fast_thumbnail",
    5: "medium_thumbnail",
    6: "full_size_preview",
    17: "raw",                   # RAW file in RAW+JPEG pair
    18: "original_adjustment_data",
    19: "adjustment_secondary_data",
    20: "penultimate_full_size_render"
}
```

## People and Faces

### ZPERSON

Person/face groupings:

```sql
CREATE TABLE ZPERSON (
    Z_PK INTEGER PRIMARY KEY,
    Z_ENT INTEGER,
    Z_OPT INTEGER,

    -- Identity
    ZPERSONUUID VARCHAR,
    ZPERSONURI VARCHAR,
    ZCONTACTMATCHINGDICTIONARY BLOB,

    -- Names
    ZFULLNAME VARCHAR,
    ZDISPLAYNAME VARCHAR,

    -- Counts
    ZFACECOUNT INTEGER,

    -- Key Face
    ZKEYFACE INTEGER,                  -- FK to ZDETECTEDFACE

    -- Type and Sorting (Photos 5+)
    ZTYPE INTEGER,                     -- -1=not favorite, 1=favorite
    ZMANUALORDER INTEGER,              -- Sort order for favorites

    -- Merge
    ZMERGETARGETPERSON INTEGER,        -- FK to ZPERSON (merge target)

    -- Verified
    ZVERIFIEDTYPE INTEGER
);

CREATE INDEX ZPERSON_UUID ON ZPERSON(ZPERSONUUID);
CREATE INDEX ZPERSON_KEYFACE ON ZPERSON(ZKEYFACE);
```

### ZDETECTEDFACE

Individual face detections:

```sql
CREATE TABLE ZDETECTEDFACE (
    Z_PK INTEGER PRIMARY KEY,
    Z_ENT INTEGER,
    Z_OPT INTEGER,

    -- Identity
    ZUUID VARCHAR,

    -- Relationships (column names vary by version)
    -- Photos 5-7:
    ZPERSON INTEGER,                   -- FK to ZPERSON
    ZASSET INTEGER,                    -- FK to ZGENERICASSET/ZASSET

    -- Photos 8+:
    ZPERSONFORFACE INTEGER,            -- FK to ZPERSON
    ZASSETFORFACE INTEGER,             -- FK to ZASSET

    -- Position (normalized 0.0 to 1.0)
    ZCENTERX FLOAT,
    ZCENTERY FLOAT,
    ZSIZE FLOAT,

    -- Quality
    ZQUALITYMEASURE FLOAT,
    ZQUALITYMEASU INTEGER,

    -- Detection Info
    ZDETECTIONTYPE INTEGER,
    ZFACEPRINT BLOB,                   -- Face recognition data
    ZFACEPRINTVERSION INTEGER,

    -- Naming
    ZNAMEDSOURCE INTEGER,              -- How person was named
    ZCONFIRMEDFACECROPGENERATIONSTATE INTEGER,

    -- Other
    ZASSETHIDDEN INTEGER,
    ZASSETVISIBLE INTEGER,
    ZCLOUDLOCALSTATE INTEGER,
    ZTRAININTYPE INTEGER
);

CREATE INDEX ZDETECTEDFACE_UUID ON ZDETECTEDFACE(ZUUID);
CREATE INDEX ZDETECTEDFACE_PERSON ON ZDETECTEDFACE(ZPERSON);
CREATE INDEX ZDETECTEDFACE_ASSET ON ZDETECTEDFACE(ZASSET);
```

**Important**: Face foreign key column names changed in Photos 8:
- Photos 5-7: `ZPERSON`, `ZASSET`
- Photos 8+: `ZPERSONFORFACE`, `ZASSETFORFACE`

## Albums and Folders

### ZGENERICALBUM

All albums, folders, and organizational collections:

```sql
CREATE TABLE ZGENERICALBUM (
    Z_PK INTEGER PRIMARY KEY,
    Z_ENT INTEGER,
    Z_OPT INTEGER,

    -- Identity
    ZUUID VARCHAR,

    -- Basic Info
    ZTITLE VARCHAR,
    ZKIND INTEGER,                     -- See Album Kinds below
    ZKINDVERSION INTEGER,

    -- Hierarchy
    ZPARENTFOLDER INTEGER,             -- FK to ZGENERICALBUM (for folders)

    -- Dates
    ZCREATIONDATE TIMESTAMP,
    ZSTARTDATE TIMESTAMP,              -- For import sessions, smart albums
    ZENDDATE TIMESTAMP,

    -- State
    ZTRASHEDSTATE INTEGER,
    ZCLOUDLOCALSTATE INTEGER,

    -- Sharing (for shared albums)
    ZCLOUDOWNERFIRSTNAME VARCHAR,
    ZCLOUDOWNERLASTNAME VARCHAR,
    ZCLOUDOWNERHASHEDPERSONID VARCHAR,
    ZCLOUDGUID VARCHAR,

    -- Sorting
    ZCUSTOMSORTKEY INTEGER,            -- Custom sort field
    ZCUSTOMSORTASCENDING INTEGER,      -- Sort direction

    -- Counts
    ZCACHEDCOUNT INTEGER,
    ZCACHEDPHOTOSCOUNT INTEGER,
    ZCACHEDVIDEOSCOUNT INTEGER,

    -- Projects
    ZPROJECTDOCUMENTTYPE VARCHAR,      -- For project albums
    ZPROJECTEXTENSIONIDENTIFIER VARCHAR,

    -- Import
    ZIMPORTEDBYBUNDLEIDENTIFIER VARCHAR,
    ZPENDITEMSCOUNT INTEGER,
    ZPENDITEMSTYPE INTEGER
);

CREATE INDEX ZGENERICALBUM_UUID ON ZGENERICALBUM(ZUUID);
CREATE INDEX ZGENERICALBUM_KIND ON ZGENERICALBUM(ZKIND);
CREATE INDEX ZGENERICALBUM_PARENT ON ZGENERICALBUM(ZPARENTFOLDER);
```

### Album Kinds

```python
ZKIND = {
    2: "album",                  # User album
    1500: "smart_album",
    1501: "special_location",
    1502: "special_timeline",
    1505: "shared_album",        # iCloud shared album
    1506: "import_session",
    1507: "legacy_album",
    1508: "project",             # Book, calendar, slideshow, etc.
    1509: "special_purpose",
    1510: "trip",
    1552: "memories",
    3571: "user_library",
    3999: "root_folder",         # Top-level folder container
    4000: "folder"               # User folder
}
```

## Asset-Album Relationships

### Z_[X]ASSETS Join Table

Links assets to albums. **Table name varies by Photos version**:

| Photos Version | Table Name |
|----------------|------------|
| Photos 5 | Z_26ASSETS |
| Photos 6 | Z_26ASSETS |
| Photos 7 | Z_27ASSETS |
| Photos 8-9.5 | Z_28ASSETS |
| Photos 9.6 | Z_29ASSETS |
| Photos 10 Beta | Z_31ASSETS |
| Photos 10 | Z_30ASSETS |
| Photos 11 | Z_32ASSETS |
| Photos 11.1 | Z_33ASSETS |

```sql
CREATE TABLE Z_26ASSETS (              -- Example: Photos 5/6
    Z_26ALBUMS INTEGER,                -- FK to ZGENERICALBUM.Z_PK
    Z_3ASSETS INTEGER,                 -- FK to ZGENERICASSET.Z_PK
    Z_FOK_3ASSETS INTEGER              -- Sort order within album
);

CREATE INDEX Z_26ASSETS_ALBUMS ON Z_26ASSETS(Z_26ALBUMS);
CREATE INDEX Z_26ASSETS_ASSETS ON Z_26ASSETS(Z_3ASSETS);
```

**Note**: The column numbers (e.g., `Z_26ALBUMS`, `Z_3ASSETS`) also vary by version. Check actual schema.

## Keywords

### ZKEYWORD

Keywords/tags:

```sql
CREATE TABLE ZKEYWORD (
    Z_PK INTEGER PRIMARY KEY,
    Z_ENT INTEGER,
    Z_OPT INTEGER,
    ZUUID VARCHAR,
    ZTITLE VARCHAR,
    ZSHORTCUT VARCHAR
);

CREATE INDEX ZKEYWORD_TITLE ON ZKEYWORD(ZTITLE);
```

### Z_1KEYWORDS Join Table

Links keywords to additional asset attributes. **Column name varies by version**:

| Photos Version | FK Column |
|----------------|-----------|
| Photos 5 | Z_37KEYWORDS |
| Photos 6 | Z_36KEYWORDS |
| Photos 7 | Z_38KEYWORDS |
| Photos 8-9.5 | Z_40KEYWORDS |
| Photos 9.6 | Z_41KEYWORDS |
| Photos 10 Beta | Z_48KEYWORDS |
| Photos 10 | Z_47KEYWORDS |
| Photos 11 | Z_51KEYWORDS |
| Photos 11.1 | Z_52KEYWORDS |

```sql
CREATE TABLE Z_1KEYWORDS (
    Z_1ASSETATTRIBUTES INTEGER,        -- FK to ZADDITIONALASSETATTRIBUTES
    Z_37KEYWORDS INTEGER               -- FK to ZKEYWORD (version-specific)
);

CREATE INDEX Z_1KEYWORDS_ATTRS ON Z_1KEYWORDS(Z_1ASSETATTRIBUTES);
CREATE INDEX Z_1KEYWORDS_KEYWORDS ON Z_1KEYWORDS(Z_37KEYWORDS);
```

## Moments

### ZMOMENT

Automatic photo groupings by time/location:

```sql
CREATE TABLE ZMOMENT (
    Z_PK INTEGER PRIMARY KEY,
    Z_ENT INTEGER,
    Z_OPT INTEGER,

    -- Identity
    ZUUID VARCHAR,

    -- Info
    ZTITLE VARCHAR,
    ZSUBTITLE VARCHAR,

    -- Dates
    ZSTARTDATE TIMESTAMP,
    ZENDDATE TIMESTAMP,
    ZREPRESENTATIVEDATE TIMESTAMP,

    -- Location
    ZAPPROXIMATELATITUDE FLOAT,
    ZAPPROXIMATELONGITUDE FLOAT,

    -- Counts
    ZCACHEDCOUNT INTEGER,
    ZCACHEDPHOTOSCOUNT INTEGER,
    ZCACHEDVIDEOSCOUNT INTEGER
);
```

## Cloud and Sync

### ZCLOUDMASTER

iCloud sync information:

```sql
CREATE TABLE ZCLOUDMASTER (
    Z_PK INTEGER PRIMARY KEY,
    Z_ENT INTEGER,
    Z_OPT INTEGER,

    -- Identity
    ZCLOUDMASTERGUID VARCHAR,
    ZUUID VARCHAR,

    -- State
    ZCLOUDLOCALSTATE INTEGER,          -- 3=fully synced
    ZCLOUDMASTERDATEATEATED TIMESTAMP,

    -- Resources
    ZIMPORTSESSIONID VARCHAR,
    ZORIGINALFILENAME VARCHAR,
    ZUNIFORMTYPEIDENTIFIER VARCHAR,

    -- Media
    ZMEDIAMETADATATYPE INTEGER,
    ZFULLSIZEJPEGSOURCE INTEGER,

    -- Relationships
    ZMOMENT INTEGER                    -- FK to ZMOMENT
);
```

### ZSHARE

Shared albums and moments:

```sql
CREATE TABLE ZSHARE (
    Z_PK INTEGER PRIMARY KEY,
    Z_ENT INTEGER,
    Z_OPT INTEGER,

    -- Identity
    ZUUID VARCHAR,
    ZSHAREURL VARCHAR,

    -- Scope
    ZSCOPETYPE INTEGER,
    ZSCOPEIDENTIFIER VARCHAR,

    -- State
    ZSTATUS INTEGER,
    ZTRASHEDSTATE INTEGER,

    -- Sharing
    ZCLOUDPHOTOCOUNT INTEGER,
    ZCLOUDVIDEOCOUNT INTEGER,

    -- Owner
    ZCLOUDOWNERHASHEDPERSONID VARCHAR,
    ZCLOUDOWNERDISPLAYNAME VARCHAR,

    -- Dates
    ZCREATIONDATE TIMESTAMP,
    ZEXPIRYDATE TIMESTAMP
);
```

## File System

### ZFILESYSTEMVOLUME

External volumes/drives:

```sql
CREATE TABLE ZFILESYSTEMVOLUME (
    Z_PK INTEGER PRIMARY KEY,
    Z_ENT INTEGER,
    Z_OPT INTEGER,

    -- Identity
    ZUUID VARCHAR,

    -- Info
    ZNAME VARCHAR,                     -- Display name
    ZVOLUMEUUIDSTRING VARCHAR          -- Volume UUID
);
```

## Adjustments

### ZUNMANAGEDADJUSTMENT

Edit/adjustment references:

```sql
CREATE TABLE ZUNMANAGEDADJUSTMENT (
    Z_PK INTEGER PRIMARY KEY,
    Z_ENT INTEGER,
    Z_OPT INTEGER,

    -- Identity
    ZUUID VARCHAR,
    ZADJUSTMENTFORMATIDENTIFIER VARCHAR,
    ZADJUSTMENTFORMATVERSION VARCHAR,

    -- Data
    ZADJUSTMENTBASEIMAGEFORMAT INTEGER,
    ZADJUSTMENTRENDERVERSION INTEGER,
    ZADJUSTMENTTIMESTAMP TIMESTAMP,

    -- Editor
    ZEDITORLOCALIZEDNAME VARCHAR,
    ZEDITORLOCALIZEDNAME VARCHAR,
    ZADJUSTMENTDATA BLOB,

    -- Relationships
    ZASSETATTRIBUTES INTEGER           -- FK to ZADDITIONALASSETATTRIBUTES
);
```

## Search Database

The `database/search/psi.sqlite` database contains search indexing:

### assets Table

```sql
CREATE TABLE assets (
    uuid_0 INTEGER,                    -- First 64 bits of UUID
    uuid_1 INTEGER,                    -- Last 64 bits of UUID
    date REAL
);

CREATE INDEX assets_uuid ON assets(uuid_0, uuid_1);
```

### groups Table

```sql
CREATE TABLE groups (
    rowid INTEGER PRIMARY KEY,
    category INTEGER,                  -- Category ID (see SearchCategory)
    owning_groupid INTEGER,
    content_string TEXT,               -- Display text
    normalized_string TEXT,            -- Normalized for search
    lookup_identifier TEXT,            -- Internal identifier
    date REAL,
    score INTEGER
);

CREATE INDEX groups_category ON groups(category);
CREATE INDEX groups_content ON groups(content_string);
```

### ga Table (group-asset associations)

```sql
CREATE TABLE ga (
    groupid INTEGER,                   -- FK to groups.rowid
    assetid INTEGER,                   -- Concatenated uuid_0|uuid_1
    data BLOB
);

CREATE INDEX ga_groupid ON ga(groupid);
CREATE INDEX ga_assetid ON ga(assetid);
```

### Search Categories

```python
SEARCH_CATEGORY = {
    2024: "labels",              # ML-detected labels
    2025: "scene",               # Scene classification
    2026: "audio",               # Audio classification (videos)
    2029: "person",              # Named people
    2030: "place",               # Location names
    2031: "keyword",             # User keywords
    2032: "title",               # Photo titles
    2033: "description",         # Photo descriptions
    2034: "album",               # Album names
    2036: "media_type",          # Photo/video/live/etc.
    2037: "text",                # Detected text (OCR)
    2038: "camera",              # Camera make/model
    2039: "holiday",             # Holiday detection
    2040: "activity",            # Activity classification
    2041: "season",              # Season classification
    2042: "venue",               # Venue classification
    2043: "venue_type"           # Type of venue
}
```

## Querying the Database

### Opening the Database

```python
import sqlite3

# Open with read-only mode
db_path = "/path/to/Library.photoslibrary/database/Photos.sqlite"
conn = sqlite3.connect(f"file:{db_path}?mode=ro", uri=True)
cursor = conn.cursor()

# Set query-only pragma for safety
cursor.execute("PRAGMA query_only = ON")
```

### Version Detection Query

```sql
SELECT Z_PLIST FROM Z_METADATA WHERE Z_VERSION = 1;
```

Parse the binary plist to extract `PLModelVersion`.

### Get All Photos

```sql
-- Photos 5
SELECT Z_PK, ZUUID, ZFILENAME, ZDATECREATED, ZFAVORITE, ZHIDDEN
FROM ZGENERICASSET
WHERE ZTRASHEDSTATE = 0;

-- Photos 6+
SELECT Z_PK, ZUUID, ZFILENAME, ZDATECREATED, ZFAVORITE, ZHIDDEN
FROM ZASSET
WHERE ZTRASHEDSTATE = 0;
```

### Get Photo with Details

```sql
-- Photos 6+
SELECT
    a.ZUUID,
    a.ZFILENAME,
    a.ZDATECREATED,
    a.ZFAVORITE,
    aa.ZTITLE,
    aa.ZORIGINALFILENAME,
    aa.ZTIMEZONENAME
FROM ZASSET a
LEFT JOIN ZADDITIONALASSETATTRIBUTES aa ON a.Z_PK = aa.ZASSET
WHERE a.ZUUID = ?;
```

### Get Albums for Photo

```sql
-- Photos 6 (adjust table name for version)
SELECT
    alb.ZUUID,
    alb.ZTITLE,
    alb.ZKIND
FROM ZGENERICALBUM alb
JOIN Z_26ASSETS j ON j.Z_26ALBUMS = alb.Z_PK
JOIN ZASSET a ON a.Z_PK = j.Z_3ASSETS
WHERE a.ZUUID = ?;
```

### Get Keywords for Photo

```sql
-- Photos 6 (adjust column name for version)
SELECT k.ZTITLE
FROM ZKEYWORD k
JOIN Z_1KEYWORDS j ON j.Z_36KEYWORDS = k.Z_PK
JOIN ZADDITIONALASSETATTRIBUTES aa ON aa.Z_PK = j.Z_1ASSETATTRIBUTES
JOIN ZASSET a ON a.Z_PK = aa.ZASSET
WHERE a.ZUUID = ?;
```

### Get Faces in Photo

```sql
-- Photos 8+
SELECT
    df.ZUUID,
    df.ZCENTERX,
    df.ZCENTERY,
    df.ZSIZE,
    p.ZFULLNAME,
    p.ZDISPLAYNAME
FROM ZDETECTEDFACE df
LEFT JOIN ZPERSON p ON p.Z_PK = df.ZPERSONFORFACE
WHERE df.ZASSETFORFACE = ?;
```

## Related Documents

- [[05-version-compatibility]] - Version-specific schema changes
- [[06-organizational-structures]] - How data is organized
- [[08-implementation-guide]] - Implementation examples

---

**Next**: [[03-file-organization]] - File storage organization
