# Directory Structure

**Related**: [[00-index]] | [[03-file-organization]] | [[04-metadata-formats]]

## Overview

The `.photoslibrary` package is a directory bundle with a well-defined structure. This document describes the complete directory hierarchy and the purpose of each component.

## Root Structure

```
Library Name.photoslibrary/
├── database/                    # Core database files
├── originals/                   # Original imported files
├── resources/                   # Derived files and metadata
├── private/                     # Application-specific private data
└── scopes/                      # Photos 7+: Shared library data
```

## Database Directory

```
database/
├── Photos.sqlite               # Main database (Photos 5+)
├── Photos.sqlite-shm           # Shared memory file (SQLite WAL mode)
├── Photos.sqlite-wal           # Write-ahead log file
├── Photos.sqlite.lock          # Lock file
├── photos.db                   # Legacy database (Photos 2-4)
├── metaSchema.db               # Schema metadata
├── DataModelVersion.plist      # Model version information
├── protection                  # Protection state file
└── search/
    └── psi.sqlite              # Search/ML labels database (Photos 5+)
```

### Database Files

#### Photos.sqlite

The primary database containing all photo metadata, albums, faces, keywords, and organizational information. Uses SQLite format with Write-Ahead Logging (WAL) mode.

**Access Requirements**:
- Read-only access recommended
- Must handle WAL mode (`.sqlite-wal` and `.sqlite-shm` files)
- May be locked by Photos.app

**Key Properties**:
- Format: SQLite 3
- Journal mode: WAL
- Encoding: UTF-8
- Schema: Core Data generated

#### photos.db (Legacy)

Present in Photos 2-4 libraries. Contains similar metadata in older schema. Still present in newer libraries for backwards compatibility and version detection.

#### psi.sqlite

Search index database containing:
- Machine learning labels
- Search terms
- Asset associations with searchable content
- Category mappings

See [[02-database-schema#Search Database]] for schema details.

#### DataModelVersion.plist

Binary property list containing:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN">
<plist version="1.0">
<dict>
    <key>PLModelVersion</key>
    <integer>19320</integer>
</dict>
</plist>
```

Used for version detection. See [[05-version-compatibility#Version Detection]].

## Originals Directory

```
originals/
├── 0/
│   ├── 00xxx-xxx-xxx-xxx-xxxx.jpg
│   ├── 01xxx-xxx-xxx-xxx-xxxx.heic
│   └── 0Fxxx-xxx-xxx-xxx-xxxx.mov
├── 1/
│   └── 1xxxx-xxx-xxx-xxx-xxxx.jpg
├── 2/
├── ...
├── 9/
├── A/
├── B/
├── ...
└── F/
```

### Organization

- Files organized by first character of UUID (hexadecimal: 0-9, A-F)
- Maximum 16 subdirectories
- File naming: `{UUID}.{extension}`
- Extensions match original file format

### File Types

Common extensions:
- **Images**: `.jpg`, `.jpeg`, `.heic`, `.heif`, `.png`, `.gif`, `.tif`, `.tiff`
- **RAW formats**: `.cr2`, `.cr3`, `.nef`, `.arw`, `.dng`, `.raf`, `.orf`, `.rw2`
- **Videos**: `.mov`, `.mp4`, `.m4v`, `.avi`
- **Live Photos**: `.heic` (photo) + `.mov` (video component)

### Special Files

- `.aae` files: Legacy iPhoto/Aperture adjustment sidecar files (rare)

## Resources Directory

```
resources/
├── derivatives/                # Previews and thumbnails
│   ├── 0/
│   │   ├── {UUID}_1_105_c.jpeg
│   │   ├── {UUID}_4_5005_c.jpeg
│   │   └── {UUID}_1_201_a.jpeg
│   ├── 1/
│   ├── ...
│   ├── masters/               # Master derivatives
│   │   ├── 0/
│   │   ├── 1/
│   │   └── ...
│   └── thumbs/                # Thumbnail cache
│       ├── thumbnailConfiguration
│       └── {NNNN}.ithmb
├── renders/                   # Adjustment renders
│   ├── 0/
│   │   ├── {UUID}.plist
│   │   └── {UUID}_1_201_a.jpeg
│   ├── 1/
│   └── ...
├── journals/                  # Change journals
│   ├── Asset-snapshot.plj
│   ├── Asset-change.plj
│   ├── Album.plist
│   ├── Person.plist
│   ├── Keyword.plist
│   └── ...
└── .metadata_never_index     # Spotlight exclusion marker
```

### Derivatives Subdirectory

Contains generated preview images and thumbnails.

**File Naming Convention**:
```
{UUID}_{quality}_{size}_{type}.{extension}
```

Parameters:
- **quality**: `1` (standard), `4` (high quality/master)
- **size**: Pixel dimension codes
  - `105`: Small thumbnail
  - `201`: Medium preview
  - `5005`: Large/master preview
  - Others: Various preview sizes
- **type**:
  - `c`: Color/standard
  - `a`: Adjusted/edited version
- **extension**: Usually `.jpeg`

Examples:
```
2E3F8B9A-1234-5678-90AB-CDEF12345678_1_105_c.jpeg    # Small thumbnail
2E3F8B9A-1234-5678-90AB-CDEF12345678_4_5005_c.jpeg   # High quality preview
2E3F8B9A-1234-5678-90AB-CDEF12345678_1_201_a.jpeg    # Edited medium preview
```

### Thumbs Subdirectory

Legacy thumbnail cache system.

- `.ithmb` files: Bundled thumbnail data
- `thumbnailConfiguration`: Configuration plist

### Renders Subdirectory

Contains adjustment/edit information.

Files:
- `{UUID}.plist`: Binary plist with adjustment data
- `{UUID}_*.jpeg`: Rendered preview of edited image

See [[04-metadata-formats#Adjustment Data]] for plist format.

### Journals Subdirectory

Change tracking and synchronization logs.

File patterns:
- `{Entity}-snapshot.plj`: Snapshot journals
- `{Entity}-change.plj`: Change journals
- `{Entity}.plist`: Entity information

Common entities:
- Asset, Album, Person, Keyword, Folder
- ImportSession, ProjectAlbum, Memory
- DetectedFace, FileSystemVolume
- FetchingAlbum, HistoryToken

## Private Directory

```
private/
├── com.apple.Photos/
│   └── appPrivateData.plist
├── com.apple.photoanalysisd/
│   ├── vision/
│   │   └── faceWorkerState.plist
│   ├── graph/
│   ├── caches/
│   │   ├── vision/
│   │   │   └── clustererState.plist
│   │   └── graph/
│   └── ...
├── com.apple.mediaanalysisd/
│   └── MediaAnalysis/
│       └── vector_database/
├── com.apple.photolibraryd/
├── com.apple.Photos.Migration/
└── com.apple.photomodel/
```

### Purpose

Contains application-specific private data not intended for direct user access.

### Key Subdirectories

#### com.apple.Photos/
Photos app settings and preferences.

#### com.apple.photoanalysisd/
Photo analysis daemon data:
- **vision/**: Face detection and computer vision data
- **graph/**: Knowledge graph data (scene understanding)
- **caches/**: Cached analysis results

Key files:
- `faceWorkerState.plist`: Face detection worker state
- `clustererState.plist`: Face clustering state

#### com.apple.mediaanalysisd/
Media analysis results:
- **vector_database/**: Vector embeddings for similarity search
- ML model outputs

#### com.apple.photolibraryd/
Photo library daemon data.

#### com.apple.photomodel/
Photo model data (machine learning models).

## Scopes Directory (Photos 7+)

```
scopes/
├── cloudsharing/              # Shared albums data
├── syndication/               # Syndicated content
└── momentshared/              # Shared moments
```

### Purpose

Introduced in Photos 7 (macOS 12) to support:
- Shared photo libraries
- Content syndication
- Shared moments/memories

### Structure

Varies by Photos version and usage. Contains additional metadata and file references for shared content.

## Hidden Files

### .metadata_never_index

Empty file in `resources/` directory that tells Spotlight not to index the contents. Prevents duplicate search results since photos are indexed via Photos.app.

## Directory Permissions

Typical permissions:
```
.photoslibrary/               drwx------  (700)
├── database/                 drwx------  (700)
├── originals/                drwx------  (700)
├── resources/                drwx------  (700)
└── private/                  drwx------  (700)
```

Files typically have `rw-------` (600) permissions.

## Size Considerations

Typical space usage:
- **originals/**: Largest - contains all original files
- **resources/derivatives/**: Moderate - previews and thumbnails
- **database/**: Small - typically under 100 MB
- **private/**: Varies - depends on analysis caching

A library with 10,000 photos (average 5 MB each):
- originals: ~50 GB
- derivatives: ~5-10 GB
- database: ~50-100 MB
- Total: ~55-60 GB

## Implementation Notes

### Directory Existence

Not all directories exist in all libraries:
- `scopes/` only in Photos 7+
- `private/com.apple.mediaanalysisd/` only in Photos 8+
- Empty libraries may have minimal directory structure

### Safe Access

When accessing:
1. Check directory existence before traversing
2. Handle missing files gracefully
3. Don't assume write permissions
4. Be aware of active Photos.app locks

### Path Construction

Always use platform-appropriate path separators:
```python
import os
library_path = "/path/to/Library.photoslibrary"
originals_dir = os.path.join(library_path, "originals")
uuid_first_char = uuid[0].upper()
photo_dir = os.path.join(originals_dir, uuid_first_char)
photo_path = os.path.join(photo_dir, f"{uuid}.jpg")
```

## Related Documents

- [[02-database-schema]] - Database structure
- [[03-file-organization]] - File organization details
- [[04-metadata-formats]] - Metadata file formats
- [[05-version-compatibility]] - Version-specific differences

---

**Next**: [[02-database-schema]] - Database schema and tables
