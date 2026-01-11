# File Organization

**Related**: [[00-index]] | [[01-directory-structure]] | [[02-database-schema]] | [[07-special-formats]]

## Overview

This document describes how photo and video files are physically stored within the `.photoslibrary` bundle and how to locate them using database information.

## File Naming and Organization

### UUID-Based System

All files in the library are organized using the asset's UUID from the database:

1. **UUID Format**: Standard UUID with hyphens (e.g., `2E3F8B9A-1234-5678-90AB-CDEF12345678`)
2. **Directory Subdivision**: First character of UUID determines subdirectory
3. **File Name**: UUID with appropriate extension

### Directory Structure

```
originals/
├── 0/
│   └── 00123456-789A-BCDE-F012-3456789ABCDE.jpg
├── 1/
├── 2/
├── ...
├── 9/
├── A/
├── B/
├── C/
├── D/
├── E/
└── F/
```

Maximum 16 subdirectories (hexadecimal: 0-F).

## Locating Original Files

### From Database to File Path

Given a photo UUID from the database:

```python
def get_original_path(library_path, uuid, extension):
    """
    Construct path to original file.

    Args:
        library_path: Path to .photoslibrary bundle
        uuid: UUID from ZASSET.ZUUID
        extension: File extension (from ZINTERNALRESOURCE or ZASSET.ZFILENAME)

    Returns:
        Full path to original file
    """
    first_char = uuid[0].upper()
    file_name = f"{uuid}.{extension}"
    return os.path.join(library_path, "originals", first_char, file_name)
```

### Getting Extension

Extension can be determined from:

1. **ZINTERNALRESOURCE.ZUNIFORMTYPEIDENTIFIER**: Most reliable
2. **ZASSET.ZFILENAME**: Original filename with extension
3. **ZADDITIONALASSETATTRIBUTES.ZORIGINALFILENAME**: Fallback

```python
# UTI to extension mapping (common types)
UTI_EXTENSIONS = {
    "public.jpeg": "jpeg",
    "public.heic": "heic",
    "public.heif": "heif",
    "public.png": "png",
    "public.tiff": "tiff",
    "com.canon.cr2-raw-image": "cr2",
    "com.canon.cr3-raw-image": "cr3",
    "com.nikon.raw-image": "nef",
    "com.sony.arw-raw-image": "arw",
    "com.adobe.raw-image": "dng",
    "public.mpeg-4": "mp4",
    "com.apple.quicktime-movie": "mov",
}
```

### Query Example

```sql
-- Get file paths for an asset
SELECT
    a.ZUUID,
    r.ZUNIFORMTYPEIDENTIFIER,
    r.ZDATASTORESUBTYPE,
    r.ZDATALENGTH
FROM ZASSET a
JOIN ZINTERNALRESOURCE r ON r.ZASSET = a.Z_PK
WHERE a.ZUUID = ?
AND r.ZDATASTORESUBTYPE IN (1, 17)  -- 1=original, 17=RAW
ORDER BY r.ZDATASTORESUBTYPE;
```

## Resource Types

### Original Files (ZDATASTORESUBTYPE = 1)

Primary imported file:
- Location: `originals/{first_char}/{UUID}.{ext}`
- Most common case
- For JPEG, HEIC, PNG, standard formats

### RAW Files (ZDATASTORESUBTYPE = 17)

RAW file in RAW+JPEG pair:
- Location: `originals/{first_char}/{UUID}.{raw_ext}`
- Separate database entry in ZINTERNALRESOURCE
- Same UUID as associated JPEG

Example RAW+JPEG pair:
```
originals/A/A1234567-89AB-CDEF-0123-456789ABCDEF.cr2  (RAW, subtype 17)
originals/A/A1234567-89AB-CDEF-0123-456789ABCDEF.jpg  (JPEG, subtype 1)
```

### Determining Primary Resource

Check `ZADDITIONALASSETATTRIBUTES.ZORIGINALRESOURCECHOICE`:
- `0` or `NULL`: JPEG is primary
- `1`: RAW is primary

```python
def get_primary_resource(resources, original_choice):
    """
    Determine which resource is primary in RAW+JPEG pair.

    Args:
        resources: List of ZINTERNALRESOURCE records
        original_choice: ZADDITIONALASSETATTRIBUTES.ZORIGINALRESOURCECHOICE

    Returns:
        Primary resource record
    """
    if original_choice == 1:
        # RAW is primary
        return [r for r in resources if r['ZDATASTORESUBTYPE'] == 17][0]
    else:
        # JPEG is primary
        return [r for r in resources if r['ZDATASTORESUBTYPE'] == 1][0]
```

## Derivative Files

### Preview and Thumbnail Generation

Photos app generates multiple derivatives for performance:

```
resources/derivatives/
├── {first_char}/
│   ├── {UUID}_1_105_c.jpeg       # Small thumbnail
│   ├── {UUID}_1_201_c.jpeg       # Medium preview
│   ├── {UUID}_4_5005_c.jpeg      # High-quality preview
│   ├── {UUID}_1_201_a.jpeg       # Adjusted medium preview
│   └── ...
```

### Derivative Naming Convention

Format: `{UUID}_{quality}_{size}_{type}.{extension}`

**Quality Codes**:
- `1`: Standard quality
- `4`: High quality/master

**Size Codes** (common):
- `105`: Small thumbnail (~105 pixels)
- `201`: Medium preview (~720 pixels)
- `1006`: Large preview (~1280 pixels)
- `5005`: Extra large/master preview (full resolution or ~3200 pixels)
- `5025`: Alternative large size

**Type Codes**:
- `c`: Color/standard (original version)
- `a`: Adjusted (edited version)

### Locating Derivatives

```python
def get_derivative_path(library_path, uuid, quality=1, size=201, type_code='c'):
    """
    Construct path to derivative file.

    Args:
        library_path: Path to .photoslibrary
        uuid: Asset UUID
        quality: Quality code (1 or 4)
        size: Size code (105, 201, 5005, etc.)
        type_code: Type code ('c' or 'a')

    Returns:
        Path to derivative file
    """
    first_char = uuid[0].upper()
    file_name = f"{uuid}_{quality}_{size}_{type_code}.jpeg"
    return os.path.join(library_path, "resources", "derivatives",
                       first_char, file_name)
```

### Derivative Existence

Not all derivatives exist for all photos:
- Generated on-demand or during import
- May be missing for recently imported photos
- May be deleted to save space (re-generated when needed)
- Check file existence before accessing

## Edited Files

### Full-Size Renders (ZDATASTORESUBTYPE = 3)

When a photo is edited and "Save as New" or non-destructive edits are applied:

- **Database**: New ZINTERNALRESOURCE with `ZDATASTORESUBTYPE = 3`
- **Location**: `originals/{first_char}/{UUID}.{ext}` (different UUID)
- **Separate Asset**: Usually creates new ZASSET entry

### Edit Previews

Located in renders directory:

```
resources/renders/
├── {first_char}/
│   ├── {UUID}.plist              # Edit metadata
│   └── {UUID}_1_201_a.jpeg       # Rendered preview
```

The `.plist` file contains adjustment data. See [[04-metadata-formats#Adjustment Data]].

## Referenced Files

### External Files (ZSAVEDASSETTYPE = 10)

Photos can reference files outside the library:

- **Storage**: File remains in original location
- **Database**: `ZASSET.ZSAVEDASSETTYPE = 10`
- **Bookmark**: `ZINTERNALRESOURCE.ZFILESYSTEMBOOKMARK` contains file reference
- **Volume**: `ZINTERNALRESOURCE.ZFILESYSTEMVOLUME` references external volume

### File System Bookmarks

The `ZFILESYSTEMBOOKMARK` blob is a macOS bookmark:

```python
import Foundation

def resolve_bookmark(bookmark_data):
    """
    Resolve a macOS file bookmark to path.

    Args:
        bookmark_data: ZFILESYSTEMBOOKMARK blob (bytes)

    Returns:
        Resolved file path or None
    """
    ns_data = Foundation.NSData.dataWithBytes_length_(
        bookmark_data, len(bookmark_data)
    )

    url, stale, error = Foundation.NSURL.URLByResolvingBookmarkData_options_relativeToURL_bookmarkDataIsStale_error_(
        ns_data, Foundation.NSURLBookmarkResolutionWithoutUI, None, None, None
    )

    if url:
        return url.path()
    return None
```

**Platform Note**: Bookmark resolution requires macOS APIs.

## Cloud Files

### iCloud Photo Library

For assets in iCloud:

- **Indicator**: `ZASSET.ZCLOUDASSETGUID` is not NULL
- **Local Availability**: Check `ZINTERNALRESOURCE.ZLOCALAVAILABILITY`
  - `1`: File is downloaded locally
  - `0` or `-1`: File is in cloud only
- **Remote Availability**: `ZINTERNALRESOURCE.ZREMOTEAVAILABILITY`

### Downloading Cloud Files

Files not locally available must be downloaded via Photos API or appear as placeholder files:

- macOS may show `.icloud` placeholder files
- Accessing triggers automatic download
- Check `ZLOCALAVAILABILITY` before reading

```python
def is_file_local(resource_row):
    """
    Check if file is available locally.

    Args:
        resource_row: ZINTERNALRESOURCE row dict

    Returns:
        True if file is local, False if cloud-only
    """
    return resource_row['ZLOCALAVAILABILITY'] == 1
```

## Live Photos

### Component Files

Live Photos consist of two files:
1. **Still image**: HEIC or JPEG (primary resource)
2. **Video component**: MOV file

### Locating Components

Both use the same UUID:

```python
def get_live_photo_components(library_path, uuid):
    """
    Get paths to both components of Live Photo.

    Args:
        library_path: Path to .photoslibrary
        uuid: Asset UUID

    Returns:
        Tuple of (still_path, video_path)
    """
    first_char = uuid[0].upper()
    base_dir = os.path.join(library_path, "originals", first_char)

    # Still image (usually HEIC)
    still_path = os.path.join(base_dir, f"{uuid}.heic")
    if not os.path.exists(still_path):
        still_path = os.path.join(base_dir, f"{uuid}.jpg")

    # Video component
    video_path = os.path.join(base_dir, f"{uuid}.mov")

    return (still_path, video_path)
```

### Database Representation

Both components have separate ZINTERNALRESOURCE entries:
- Image: `ZDATASTORESUBTYPE = 1`, `ZRESOURCETYPE = 1`
- Video: `ZDATASTORESUBTYPE = 2`, `ZRESOURCETYPE = 9`

See [[07-special-formats#Live Photos]] for details.

## Burst Photos

### File Organization

Burst photos are stored individually:
- Each photo has its own UUID
- Each stored in originals directory
- Linked via `ZAVALANCHEUUID` (burst set UUID)

### Locating Burst Set

```sql
SELECT ZUUID, ZAVALANCHEPICKTYPE
FROM ZASSET
WHERE ZAVALANCHEUUID = ?
ORDER BY ZUUID;
```

All photos with same `ZAVALANCHEUUID` belong to burst set.

See [[07-special-formats#Burst Photos]] for details.

## Video Files

### Storage

Videos stored same as photos:
- Location: `originals/{first_char}/{UUID}.{ext}`
- Common extensions: `.mov`, `.mp4`, `.m4v`
- **ZASSET.ZKIND = 1** (video)

### Video Duration

Duration available in database:
- `ZASSET.ZDURATION`: Duration in seconds (float)
- `ZASSET.ZVIDEOCPDURATIONVALUE`: Duration in video time units

## File Size Information

### Original File Size

Available in database:
- `ZINTERNALRESOURCE.ZDATALENGTH`: Size in bytes
- `ZADDITIONALASSETATTRIBUTES.ZORIGINALFILESIZE`: Original file size

```python
def get_file_size(asset_pk, conn):
    """
    Get file size from database.

    Args:
        asset_pk: ZASSET.Z_PK
        conn: Database connection

    Returns:
        File size in bytes
    """
    cursor = conn.cursor()
    cursor.execute("""
        SELECT ZDATALENGTH
        FROM ZINTERNALRESOURCE
        WHERE ZASSET = ? AND ZDATASTORESUBTYPE = 1
        LIMIT 1
    """, (asset_pk,))

    row = cursor.fetchone()
    return row[0] if row else None
```

### Derivative Sizes

Derivative files are typically smaller than originals:
- Thumbnails: Few hundred KB
- Previews: 1-5 MB
- Full renders: Similar to original

## File Integrity

### Fingerprints

Files are fingerprinted for deduplication and integrity:

**Photos 5-9**:
- `ZADDITIONALASSETATTRIBUTES.ZMASTERFINGERPRINT`: File hash
- Format: Base64-encoded hash

**Photos 10+**:
- `ZADDITIONALASSETATTRIBUTES.ZORIGINALSTABLEHASH`: Stable hash (integer)
- `ZINTERNALRESOURCE.ZFINGERPRINT`: Resource fingerprint

### Verifying File Integrity

```python
def verify_file_matches_fingerprint(file_path, expected_fingerprint):
    """
    Verify file matches expected fingerprint.

    Note: Fingerprint algorithm is proprietary and not publicly documented.
    This is a placeholder for the concept.
    """
    # Apple's fingerprint algorithm is not publicly known
    # Would need reverse engineering or comparison approaches
    pass
```

## Missing Files

### Handling Missing Originals

Files may be missing due to:
- Cloud-only storage (not downloaded)
- External volume not mounted
- File deleted/moved outside Photos
- Corruption

### Fallback Strategy

When original is unavailable:
1. Check `ZLOCALAVAILABILITY` for cloud status
2. Try derivative files as fallback
3. Check `ZFILESYSTEMVOLUME` for external volume
4. Use highest quality available derivative:
   - Try `{UUID}_4_5005_c.jpeg` (highest quality)
   - Try `{UUID}_1_201_c.jpeg` (medium)
   - Try `{UUID}_1_105_c.jpeg` (thumbnail)

```python
def find_available_file(library_path, uuid, resources):
    """
    Find any available file for asset, trying in order of quality.

    Args:
        library_path: Path to .photoslibrary
        uuid: Asset UUID
        resources: List of ZINTERNALRESOURCE rows

    Returns:
        Path to available file or None
    """
    # Try original
    for resource in resources:
        if resource['ZDATASTORESUBTYPE'] == 1:
            path = construct_original_path(library_path, uuid, resource)
            if os.path.exists(path):
                return path

    # Try derivatives in order of quality
    derivatives = [
        (4, 5005, 'c'),  # Highest quality
        (1, 1006, 'c'),  # Large
        (1, 201, 'c'),   # Medium
        (1, 105, 'c'),   # Small thumbnail
    ]

    for quality, size, type_code in derivatives:
        path = get_derivative_path(library_path, uuid, quality, size, type_code)
        if os.path.exists(path):
            return path

    return None
```

## File Access Best Practices

### Read-Only Access

Always access files read-only:
- Don't modify files in place
- Don't delete files
- Don't move files
- Photos.app manages file lifecycle

### Concurrent Access

Be aware Photos.app may be accessing files:
- Files may be locked
- Handle `EACCES` / `EPERM` errors gracefully
- Don't assume exclusive access

### Performance Considerations

For large libraries:
- Cache file paths after construction
- Use derivatives when full resolution not needed
- Batch database queries for multiple assets
- Consider parallel file access for thumbnails

### Example: Efficient Thumbnail Loading

```python
def load_thumbnails_batch(library_path, uuids, conn):
    """
    Load thumbnails for multiple assets efficiently.

    Args:
        library_path: Path to .photoslibrary
        uuids: List of UUIDs
        conn: Database connection

    Returns:
        Dict mapping UUID to thumbnail path
    """
    result = {}

    # Batch query for metadata
    placeholders = ','.join('?' * len(uuids))
    cursor = conn.cursor()
    cursor.execute(f"""
        SELECT ZUUID, ZUNIFORMTYPEIDENTIFIER
        FROM ZASSET
        WHERE ZUUID IN ({placeholders})
    """, uuids)

    rows = cursor.fetchall()

    # Construct paths in parallel
    for uuid, uti in rows:
        thumb_path = get_derivative_path(library_path, uuid, 1, 105, 'c')
        if os.path.exists(thumb_path):
            result[uuid] = thumb_path

    return result
```

## Platform Considerations

### macOS Specific

- Bookmarks require macOS Foundation framework
- `.photoslibrary` bundle appears as file in Finder
- Extended attributes may be present on files

### Cross-Platform Access

For accessing on non-macOS:
- Directory structure is standard (works on Linux/Windows)
- Bookmark resolution won't work (track bookmarks as blob data only)
- File extensions are lowercase or uppercase depending on source

## Related Documents

- [[01-directory-structure]] - Physical directory layout
- [[02-database-schema]] - Database structure
- [[07-special-formats]] - Special file types
- [[08-implementation-guide]] - Implementation examples

---

**Next**: [[04-metadata-formats]] - Metadata storage formats
