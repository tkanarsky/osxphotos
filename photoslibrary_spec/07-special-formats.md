# Special Formats and File Types

**Related**: [[00-index]] | [[02-database-schema]] | [[03-file-organization]]

## Overview

This document describes special file types and conventions including Live Photos, burst photos, RAW+JPEG pairs, panoramas, screenshots, and other media types.

## Live Photos

### Description

Live Photos capture a short video clip (typically 1.5-3 seconds) along with the still image. They consist of two components:
1. **Still image**: HEIC or JPEG file
2. **Video component**: MOV file

### Database Representation

**Detection**: `ZASSET.ZKINDSUBTYPE = 2`

**File Resources**: Two entries in `ZINTERNALRESOURCE`:

```sql
SELECT
    r.ZDATASTORESUBTYPE,
    r.ZRESOURCETYPE,
    r.ZUNIFORMTYPEIDENTIFIER,
    r.ZDATALENGTH
FROM ZINTERNALRESOURCE r
WHERE r.ZASSET = ?
ORDER BY r.ZDATASTORESUBTYPE;
```

| Component | ZDATASTORESUBTYPE | ZRESOURCETYPE | UTI |
|-----------|-------------------|---------------|-----|
| Still image | 1 | 1 | public.heic or public.jpeg |
| Video | 2 | 9 | com.apple.quicktime-movie |

### File Locations

Both components share the same UUID:

```
originals/{first_char}/
├── {UUID}.heic    # Still image
└── {UUID}.mov     # Video component
```

### Working with Live Photos

```python
def get_live_photo_info(conn, asset_pk):
    """
    Get Live Photo component information.

    Args:
        conn: Database connection
        asset_pk: Asset Z_PK

    Returns:
        Dict with still and video info
    """
    cursor = conn.cursor()

    # Get basic asset info
    cursor.execute("""
        SELECT ZUUID, ZKINDSUBTYPE
        FROM ZASSET
        WHERE Z_PK = ?
    """, (asset_pk,))

    row = cursor.fetchone()
    if not row or row[1] != 2:  # Not a Live Photo
        return None

    uuid = row[0]

    # Get resources
    cursor.execute("""
        SELECT
            ZDATASTORESUBTYPE,
            ZRESOURCETYPE,
            ZUNIFORMTYPEIDENTIFIER,
            ZDATALENGTH
        FROM ZINTERNALRESOURCE
        WHERE ZASSET = ?
        ORDER BY ZDATASTORESUBTYPE
    """, (asset_pk,))

    resources = cursor.fetchall()

    still_resource = next((r for r in resources if r[0] == 1), None)
    video_resource = next((r for r in resources if r[0] == 2), None)

    return {
        'uuid': uuid,
        'type': 'live_photo',
        'still': {
            'uti': still_resource[2] if still_resource else None,
            'size': still_resource[3] if still_resource else None,
            'path': construct_file_path(uuid, still_resource[2]) if still_resource else None
        },
        'video': {
            'uti': video_resource[2] if video_resource else None,
            'size': video_resource[3] if video_resource else None,
            'path': construct_file_path(uuid, 'com.apple.quicktime-movie') if video_resource else None
        }
    }
```

### Extracting Still from Live Photo

The HEIC/JPEG file is a standard image - no special extraction needed.

### Extracting Video from Live Photo

The MOV file can be played directly as a standard QuickTime video.

## Burst Photos

### Description

Burst mode captures rapid sequence of photos. One photo is marked as the "key" photo representing the burst.

### Database Representation

Burst photos share a common burst UUID:

**Identifier**: `ZASSET.ZAVALANCHEUUID`
**Pick Type**: `ZASSET.ZAVALANCHEPICKTYPE`

```python
BURST_PICK_TYPES = {
    0: "not_burst",         # Not in a burst
    2: "not_selected",      # In burst, not selected
    4: "default_pick",      # Default selection
    8: "user_selected",     # User selected
    16: "key_photo"         # Key/representative photo
}
```

### File Storage

Each burst photo is stored individually:

```
originals/{first_char}/
├── {UUID1}.jpg    # Burst photo 1
├── {UUID2}.jpg    # Burst photo 2
├── {UUID3}.jpg    # Burst photo 3 (key photo)
└── ...
```

### Querying Burst Sets

```sql
-- Get all photos in a burst set
SELECT
    ZUUID,
    ZAVALANCHEPICKTYPE,
    ZDATECREATED
FROM ZASSET
WHERE ZAVALANCHEUUID = ?
ORDER BY ZDATECREATED;
```

### Working with Bursts

```python
def get_burst_photos(conn, burst_uuid, asset_table):
    """
    Get all photos in a burst set.

    Args:
        conn: Database connection
        burst_uuid: Burst UUID (ZAVALANCHEUUID)
        asset_table: Asset table name

    Returns:
        List of burst photo dicts
    """
    query = f"""
        SELECT
            ZUUID,
            ZAVALANCHEPICKTYPE,
            ZDATECREATED
        FROM {asset_table}
        WHERE ZAVALANCHEUUID = ?
        ORDER BY ZDATECREATED
    """

    cursor = conn.cursor()
    cursor.execute(query, (burst_uuid,))

    photos = []
    for uuid, pick_type, date_created in cursor.fetchall():
        photos.append({
            'uuid': uuid,
            'pick_type': pick_type,
            'is_key': pick_type == 16,
            'is_selected': pick_type in (4, 8, 16),
            'date_created': date_created
        })

    return photos

def get_burst_key_photo(conn, burst_uuid, asset_table):
    """
    Get the key photo from a burst.

    Args:
        conn: Database connection
        burst_uuid: Burst UUID
        asset_table: Asset table name

    Returns:
        UUID of key photo or None
    """
    query = f"""
        SELECT ZUUID
        FROM {asset_table}
        WHERE ZAVALANCHEUUID = ?
        AND ZAVALANCHEPICKTYPE = 16
        LIMIT 1
    """

    cursor = conn.cursor()
    cursor.execute(query, (burst_uuid,))

    row = cursor.fetchone()
    return row[0] if row else None
```

### Visibility

Non-selected burst photos have `ZVISIBILITYSTATE = 2` and don't appear in main library views.

## RAW + JPEG Pairs

### Description

Many cameras save both RAW and JPEG versions of each photo. Photos treats these as a single asset with two resources.

### Database Representation

Single asset with two resources in `ZINTERNALRESOURCE`:

```sql
SELECT
    r.ZDATASTORESUBTYPE,
    r.ZUNIFORMTYPEIDENTIFIER,
    r.ZDATALENGTH
FROM ZINTERNALRESOURCE r
WHERE r.ZASSET = ?
ORDER BY r.ZDATASTORESUBTYPE;
```

| Component | ZDATASTORESUBTYPE | Example UTI |
|-----------|-------------------|-------------|
| JPEG | 1 | public.jpeg |
| RAW | 17 | com.canon.cr2-raw-image |

### Primary Resource Selection

**Field**: `ZADDITIONALASSETATTRIBUTES.ZORIGINALRESOURCECHOICE`

| Value | Meaning |
|-------|---------|
| 0 or NULL | JPEG is primary (displayed by default) |
| 1 | RAW is primary |

### File Locations

Both files share the same UUID with different extensions:

```
originals/{first_char}/
├── {UUID}.jpg     # JPEG version
└── {UUID}.cr2     # RAW version
```

### Working with RAW+JPEG

```python
def get_raw_jpeg_pair(conn, asset_pk):
    """
    Get RAW and JPEG file information.

    Args:
        conn: Database connection
        asset_pk: Asset Z_PK

    Returns:
        Dict with RAW and JPEG info
    """
    cursor = conn.cursor()

    # Get UUID and original choice
    cursor.execute("""
        SELECT a.ZUUID, aa.ZORIGINALRESOURCECHOICE
        FROM ZASSET a
        LEFT JOIN ZADDITIONALASSETATTRIBUTES aa ON aa.ZASSET = a.Z_PK
        WHERE a.Z_PK = ?
    """, (asset_pk,))

    row = cursor.fetchone()
    if not row:
        return None

    uuid, original_choice = row

    # Get resources
    cursor.execute("""
        SELECT
            ZDATASTORESUBTYPE,
            ZUNIFORMTYPEIDENTIFIER,
            ZDATALENGTH
        FROM ZINTERNALRESOURCE
        WHERE ZASSET = ?
        AND ZDATASTORESUBTYPE IN (1, 17)
        ORDER BY ZDATASTORESUBTYPE
    """, (asset_pk,))

    resources = cursor.fetchall()

    jpeg_resource = next((r for r in resources if r[0] == 1), None)
    raw_resource = next((r for r in resources if r[0] == 17), None)

    if not jpeg_resource or not raw_resource:
        return None  # Not a RAW+JPEG pair

    return {
        'uuid': uuid,
        'jpeg': {
            'uti': jpeg_resource[1],
            'size': jpeg_resource[2],
            'is_primary': original_choice != 1
        },
        'raw': {
            'uti': raw_resource[1],
            'size': raw_resource[2],
            'is_primary': original_choice == 1
        },
        'primary': 'raw' if original_choice == 1 else 'jpeg'
    }
```

## Panoramas

### Description

Panorama photos are wide-format images stitched from multiple photos.

### Database Representation

**Detection**: `ZASSET.ZKINDSUBTYPE = 1`

### File Storage

Stored as single file (usually JPEG or HEIC):

```
originals/{first_char}/
└── {UUID}.jpg     # Panorama
```

### Dimensions

Panoramas typically have:
- Very wide aspect ratio (> 2:1)
- High pixel width (`ZASSET.ZWIDTH` > 4000)

```python
def is_panorama(kind_subtype, width, height):
    """
    Check if photo is a panorama.

    Args:
        kind_subtype: ZKINDSUBTYPE value
        width, height: Pixel dimensions

    Returns:
        True if panorama
    """
    # Explicit panorama subtype
    if kind_subtype == 1:
        return True

    # Wide aspect ratio heuristic
    if width and height:
        aspect = width / height
        if aspect > 2.0 or aspect < 0.5:
            return True

    return False
```

## Screenshots

### Description

Screenshots captured on the device.

### Database Representation

**Detection**: `ZASSET.ZKINDSUBTYPE = 10`

### File Storage

Standard image file storage:

```
originals/{first_char}/
└── {UUID}.png     # Screenshot (typically PNG)
```

## Screen Recordings

### Description

Videos of screen activity.

### Database Representation

**Detection**: `ZASSET.ZKINDSUBTYPE = 103`

### File Storage

Stored as video file:

```
originals/{first_char}/
└── {UUID}.mov     # Screen recording
```

## Slow Motion Videos

### Description

Videos recorded in slow-motion mode.

### Database Representation

**Detection**: `ZASSET.ZKINDSUBTYPE = 101`

### File Storage

```
originals/{first_char}/
└── {UUID}.mov     # Slow-motion video
```

### Playback

Slow-motion videos contain:
- Normal speed sections
- Slow-motion section (marked in video metadata)
- Video player uses metadata to vary playback speed

## Time-Lapse Videos

### Description

Time-lapse videos created from sequence of photos.

### Database Representation

**Detection**: `ZASSET.ZKINDSUBTYPE = 102`

### File Storage

```
originals/{first_char}/
└── {UUID}.mov     # Time-lapse video
```

## Portrait Photos

### Description

Photos with depth information (Portrait mode).

### Database Representation

**Detection**: `ZASSET.ZKINDSUBTYPE = 110`

### File Storage

Portrait photos may include depth map:

```
originals/{first_char}/
└── {UUID}.heic    # Portrait photo with embedded depth
```

The HEIC file contains:
- Primary image
- Depth map (auxiliary image)
- Portrait metadata

### Extracting Depth Map

Depth maps are typically embedded as auxiliary images in HEIC format. Use image processing libraries that support HEIC depth:

```python
# Example using Pillow/PIL
from PIL import Image

def has_depth_map(heic_path):
    """
    Check if HEIC file contains depth map.

    Args:
        heic_path: Path to HEIC file

    Returns:
        True if depth map present
    """
    try:
        img = Image.open(heic_path)
        # Check for depth auxiliary image
        return 'depth' in img.info or 'Depth' in img.info
    except:
        return False
```

## HDR Photos

### Description

High Dynamic Range photos with extended luminance range.

### Database Representation

**Photos 5-6**: `ZADDITIONALASSETATTRIBUTES.ZCUSTOMRENDEREDVALUE = 3`
**Photos 7+**: `ZADDITIONALASSETATTRIBUTES.ZHDRTYPE = 3` or higher

### File Storage

```
originals/{first_char}/
└── {UUID}.heic    # HDR photo
```

HDR information embedded in HEIC file metadata.

## Shared Photos

### Description

Photos shared via iCloud Photo Sharing.

### Database Representation

**Detection**: `ZASSET.ZCLOUDBATCHPUBLISHDATE` is not NULL

**Owner**: `ZASSET.ZCLOUDOWNERHASHEDPERSONID`

```sql
-- Get shared photos
SELECT
    ZUUID,
    ZFILENAME,
    ZCLOUDBATCHPUBLISHDATE,
    ZCLOUDOWNERHASHEDPERSONID
FROM ZASSET
WHERE ZCLOUDBATCHPUBLISHDATE IS NOT NULL;
```

### Shared vs. Local

Shared photos may be:
1. **Owned by user**: User shared them
2. **Owned by others**: Received from shared album

Check `ZCLOUDOWNERHASHEDPERSONID` against user's identifier.

## Referenced Files

### Description

Files that remain in their original location (not imported into library).

### Database Representation

**Detection**: `ZASSET.ZSAVEDASSETTYPE = 10`

**Bookmark**: `ZINTERNALRESOURCE.ZFILESYSTEMBOOKMARK`
**Volume**: `ZINTERNALRESOURCE.ZFILESYSTEMVOLUME`

### File Access

Referenced files require:
1. Resolving bookmark to get current path
2. Checking if external volume is mounted
3. Handling file unavailability

```python
def get_referenced_file_path(conn, asset_pk):
    """
    Get path to referenced file.

    Args:
        conn: Database connection
        asset_pk: Asset Z_PK

    Returns:
        File path or None if unavailable
    """
    cursor = conn.cursor()

    # Get bookmark
    cursor.execute("""
        SELECT
            r.ZFILESYSTEMBOOKMARK,
            r.ZFILESYSTEMVOLUME,
            v.ZNAME as volume_name
        FROM ZINTERNALRESOURCE r
        LEFT JOIN ZFILESYSTEMVOLUME v ON v.Z_PK = r.ZFILESYSTEMVOLUME
        WHERE r.ZASSET = ?
        AND r.ZDATASTORESUBTYPE = 1
        LIMIT 1
    """, (asset_pk,))

    row = cursor.fetchone()
    if not row or not row[0]:
        return None

    bookmark, volume_pk, volume_name = row

    # Resolve bookmark (requires macOS)
    try:
        path = resolve_bookmark(bookmark)  # See [[03-file-organization]]
        return path
    except:
        return None
```

## Media Type Detection

### Complete Detection Function

```python
def detect_media_type(asset_row):
    """
    Detect complete media type from asset row.

    Args:
        asset_row: Dict with ZASSET columns

    Returns:
        String describing media type
    """
    kind = asset_row['ZKIND']
    subtype = asset_row.get('ZKINDSUBTYPE', 0)
    saved_type = asset_row.get('ZSAVEDASSETTYPE', 0)

    # Base type
    if kind == 0:  # Photo
        # Check subtype
        if subtype == 1:
            return 'panorama'
        elif subtype == 2:
            return 'live_photo'
        elif subtype == 10:
            return 'screenshot'
        elif subtype == 100:
            return 'hdr_photo'
        elif subtype == 110:
            return 'portrait'
        else:
            return 'photo'

    elif kind == 1:  # Video
        if subtype == 101:
            return 'slow_motion'
        elif subtype == 102:
            return 'time_lapse'
        elif subtype == 103:
            return 'screen_recording'
        else:
            return 'video'

    return 'unknown'
```

### Media Type Summary Table

| ZKIND | ZKINDSUBTYPE | Type | Description |
|-------|--------------|------|-------------|
| 0 | 0 | Photo | Standard photo |
| 0 | 1 | Panorama | Wide-format photo |
| 0 | 2 | Live Photo | Photo with video |
| 0 | 10 | Screenshot | Screen capture |
| 0 | 100 | HDR | High dynamic range |
| 0 | 110 | Portrait | Portrait mode with depth |
| 1 | 0 | Video | Standard video |
| 1 | 101 | Slow Motion | Slow-motion video |
| 1 | 102 | Time-Lapse | Time-lapse video |
| 1 | 103 | Screen Recording | Screen recording |

## Edited Photos

### Description

Photos with non-destructive edits applied.

### Database Representation

**Photos 5-9**: `ZASSET.ZHASADJUSTMENTS = 1`
**Photos 10+**: `ZASSET.ZADJUSTMENTSSTATE > 0`

### Edit Data

Edit information stored in:
1. **File**: `resources/renders/{first_char}/{UUID}.plist`
2. **Database**: `ZUNMANAGEDADJUSTMENT` table reference

See [[04-metadata-formats#Adjustment Data]] for plist format.

### Rendered Previews

Edited preview in:
```
resources/renders/{first_char}/
└── {UUID}_1_201_a.jpeg    # Edited preview
```

## Related Documents

- [[02-database-schema]] - Database structure
- [[03-file-organization]] - File organization
- [[04-metadata-formats]] - Metadata formats
- [[06-organizational-structures]] - Organization

---

**Next**: [[08-implementation-guide]] - Implementation guidance and examples
