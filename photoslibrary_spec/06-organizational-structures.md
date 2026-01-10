# Organizational Structures

**Related**: [[00-index]] | [[02-database-schema]] | [[07-special-formats]]

## Overview

Photos organizes assets using albums, folders, moments, keywords, and people. This document describes how these organizational structures are implemented.

## Albums

### Album Types

Albums are stored in the `ZGENERICALBUM` table with different `ZKIND` values:

```python
ALBUM_KINDS = {
    2: "user_album",           # Regular user-created album
    1500: "smart_album",       # Smart albums (deprecated)
    1501: "special_location",  # Special location albums
    1502: "special_timeline",  # Special timeline albums
    1505: "shared_album",      # iCloud shared albums
    1506: "import_session",    # Import sessions
    1507: "legacy_album",      # Legacy album types
    1508: "project",           # Projects (books, calendars, slideshows)
    1509: "special_purpose",   # Special purpose albums
    1510: "trip",             # Trip albums (Photos 11+)
    1552: "memories",         # Memories/moments
    3571: "user_library",     # User library container
    3999: "root_folder",      # Root folder container
    4000: "folder"            # User-created folders
}
```

### User Albums (ZKIND = 2)

Regular user-created albums:

```sql
SELECT
    ZUUID,
    ZTITLE,
    ZCREATIONDATE,
    ZCACHEDCOUNT,
    ZPARENTFOLDER
FROM ZGENERICALBUM
WHERE ZKIND = 2
AND ZTRASHEDSTATE = 0
ORDER BY ZTITLE;
```

**Properties**:
- `ZTITLE`: Album name
- `ZCREATIONDATE`: When created
- `ZPARENTFOLDER`: FK to parent folder (NULL if in root)
- `ZCACHEDCOUNT`: Number of photos
- `ZCUSTOMSORTKEY`: Custom sort field
- `ZCUSTOMSORTASCENDING`: Sort direction

### Shared Albums (ZKIND = 1505)

iCloud shared albums:

```sql
SELECT
    ZUUID,
    ZTITLE,
    ZCLOUDOWNERFIRSTNAME,
    ZCLOUDOWNERLASTNAME,
    ZCLOUDGUID
FROM ZGENERICALBUM
WHERE ZKIND = 1505
AND ZTRASHEDSTATE = 0;
```

**Properties**:
- `ZCLOUDOWNERFIRSTNAME`: Owner first name
- `ZCLOUDOWNERLASTNAME`: Owner last name
- `ZCLOUDOWNERHASHEDPERSONID`: Owner identifier
- `ZCLOUDGUID`: iCloud identifier
- `ZCLOUDLOCALSTATE`: Sync state

**Note**: Shared albums cannot be in folders (`ZPARENTFOLDER` is always NULL).

### Import Sessions (ZKIND = 1506)

Each import creates a session:

```sql
SELECT
    ZUUID,
    ZTITLE,
    ZSTARTDATE,
    ZENDDATE,
    ZCACHEDCOUNT
FROM ZGENERICALBUM
WHERE ZKIND = 1506
ORDER BY ZSTARTDATE DESC;
```

**Properties**:
- `ZSTARTDATE`: First photo import time
- `ZENDDATE`: Last photo import time
- Automatic title generation

### Projects (ZKIND = 1508)

Books, calendars, slideshows, and other projects:

```sql
SELECT
    ZUUID,
    ZTITLE,
    ZPROJECTDOCUMENTTYPE,
    ZPROJECTEXTENSIONIDENTIFIER
FROM ZGENERICALBUM
WHERE ZKIND = 1508;
```

**Project Types** (ZPROJECTDOCUMENTTYPE):
- `book`: Photo book
- `calendar`: Calendar
- `slideshow`: Slideshow
- `card`: Greeting card

## Folders

### Folder Hierarchy

Folders organize albums into a hierarchy:

```sql
SELECT
    ZUUID,
    ZTITLE,
    ZPARENTFOLDER,
    ZKIND
FROM ZGENERICALBUM
WHERE ZKIND IN (3999, 4000)
ORDER BY ZTITLE;
```

**Folder Kinds**:
- `ZKIND = 3999`: Root folder (top-level container)
- `ZKIND = 4000`: User folder

### Building Folder Tree

```python
def build_folder_tree(conn):
    """
    Build complete folder hierarchy.

    Args:
        conn: Database connection

    Returns:
        Dict representing folder tree
    """
    cursor = conn.cursor()

    # Get all folders
    cursor.execute("""
        SELECT Z_PK, ZUUID, ZTITLE, ZPARENTFOLDER, ZKIND
        FROM ZGENERICALBUM
        WHERE ZKIND IN (3999, 4000)
    """)

    folders = {}
    for pk, uuid, title, parent_pk, kind in cursor.fetchall():
        folders[pk] = {
            'uuid': uuid,
            'title': title or 'Root',
            'parent_pk': parent_pk,
            'kind': kind,
            'children': [],
            'albums': []
        }

    # Build tree
    root = None
    for pk, folder in folders.items():
        if folder['kind'] == 3999:  # Root folder
            root = folder
        elif folder['parent_pk'] in folders:
            folders[folder['parent_pk']]['children'].append(folder)

    # Add albums to folders
    cursor.execute("""
        SELECT Z_PK, ZUUID, ZTITLE, ZPARENTFOLDER, ZKIND
        FROM ZGENERICALBUM
        WHERE ZKIND = 2
    """)

    for pk, uuid, title, parent_pk, kind in cursor.fetchall():
        album = {
            'uuid': uuid,
            'title': title,
            'kind': kind
        }

        if parent_pk and parent_pk in folders:
            folders[parent_pk]['albums'].append(album)
        elif root:
            root['albums'].append(album)

    return root
```

### Getting Album Path

```python
def get_album_path(conn, album_pk):
    """
    Get full folder path for album.

    Args:
        conn: Database connection
        album_pk: Album Z_PK

    Returns:
        List of folder names from root to album
    """
    cursor = conn.cursor()

    path = []
    current_pk = album_pk

    while current_pk:
        cursor.execute("""
            SELECT ZTITLE, ZPARENTFOLDER, ZKIND
            FROM ZGENERICALBUM
            WHERE Z_PK = ?
        """, (current_pk,))

        row = cursor.fetchone()
        if not row:
            break

        title, parent_pk, kind = row

        # Don't include root folder
        if kind != 3999:
            path.insert(0, title)

        current_pk = parent_pk

    return path
```

## Album-Asset Relationships

### Join Table Structure

Photos are linked to albums via version-specific join tables (see [[05-version-compatibility]]):

**Photos 5-6**: `Z_26ASSETS`
**Photos 7**: `Z_27ASSETS`
**Photos 8-9.5**: `Z_28ASSETS`
**Photos 9.6**: `Z_29ASSETS`
**Photos 10**: `Z_30ASSETS`
**Photos 11**: `Z_32ASSETS`
**Photos 11.1**: `Z_33ASSETS`

### Getting Photos in Album

```python
def get_photos_in_album(conn, album_uuid, asset_table, join_table):
    """
    Get all photos in an album.

    Args:
        conn: Database connection
        album_uuid: Album UUID
        asset_table: Asset table name (ZASSET or ZGENERICASSET)
        join_table: Join table name (e.g., Z_26ASSETS)

    Returns:
        List of photo records
    """
    # Determine column names from join table structure
    cursor = conn.cursor()
    cursor.execute(f"PRAGMA table_info({join_table})")
    columns = [col[1] for col in cursor.fetchall()]

    # Find album and asset column names
    album_col = [c for c in columns if 'ALBUMS' in c][0]
    asset_col = [c for c in columns if 'ASSETS' in c and c != album_col][0]
    sort_col = [c for c in columns if 'FOK' in c][0] if any('FOK' in c for c in columns) else None

    # Build query
    order_clause = f"ORDER BY j.{sort_col}" if sort_col else ""

    query = f"""
        SELECT
            a.ZUUID,
            a.ZFILENAME,
            a.ZDATECREATED,
            j.{sort_col} as sort_order
        FROM {asset_table} a
        JOIN {join_table} j ON j.{asset_col} = a.Z_PK
        JOIN ZGENERICALBUM alb ON alb.Z_PK = j.{album_col}
        WHERE alb.ZUUID = ?
        {order_clause}
    """

    cursor.execute(query, (album_uuid,))
    return cursor.fetchall()
```

### Getting Albums for Photo

```python
def get_albums_for_photo(conn, photo_uuid, asset_table, join_table):
    """
    Get all albums containing a photo.

    Args:
        conn: Database connection
        photo_uuid: Photo UUID
        asset_table: Asset table name
        join_table: Join table name

    Returns:
        List of album records
    """
    cursor = conn.cursor()
    cursor.execute(f"PRAGMA table_info({join_table})")
    columns = [col[1] for col in cursor.fetchall()]

    album_col = [c for c in columns if 'ALBUMS' in c][0]
    asset_col = [c for c in columns if 'ASSETS' in c and c != album_col][0]

    query = f"""
        SELECT
            alb.ZUUID,
            alb.ZTITLE,
            alb.ZKIND
        FROM ZGENERICALBUM alb
        JOIN {join_table} j ON j.{album_col} = alb.Z_PK
        JOIN {asset_table} a ON a.Z_PK = j.{asset_col}
        WHERE a.ZUUID = ?
        ORDER BY alb.ZTITLE
    """

    cursor.execute(query, (photo_uuid,))
    return cursor.fetchall()
```

## Keywords

### Keyword Structure

Keywords are tags assigned to photos:

```sql
SELECT Z_PK, ZTITLE, ZUUID
FROM ZKEYWORD
ORDER BY ZTITLE;
```

### Keyword-Photo Relationships

Keywords link to `ZADDITIONALASSETATTRIBUTES` via version-specific join table:

```python
def get_keywords_for_photo(conn, asset_pk, keyword_join_col):
    """
    Get keywords for a photo.

    Args:
        conn: Database connection
        asset_pk: Asset Z_PK
        keyword_join_col: Version-specific keyword column (e.g., Z_37KEYWORDS)

    Returns:
        List of keyword titles
    """
    query = f"""
        SELECT k.ZTITLE
        FROM ZKEYWORD k
        JOIN Z_1KEYWORDS j ON j.{keyword_join_col} = k.Z_PK
        JOIN ZADDITIONALASSETATTRIBUTES aa ON aa.Z_PK = j.Z_1ASSETATTRIBUTES
        WHERE aa.ZASSET = ?
        ORDER BY k.ZTITLE
    """

    cursor = conn.cursor()
    cursor.execute(query, (asset_pk,))
    return [row[0] for row in cursor.fetchall()]
```

### Getting Photos for Keyword

```python
def get_photos_for_keyword(conn, keyword_title, asset_table, keyword_join_col):
    """
    Get all photos with a specific keyword.

    Args:
        conn: Database connection
        keyword_title: Keyword text
        asset_table: Asset table name
        keyword_join_col: Keyword column name

    Returns:
        List of photo UUIDs
    """
    query = f"""
        SELECT a.ZUUID
        FROM {asset_table} a
        JOIN ZADDITIONALASSETATTRIBUTES aa ON aa.ZASSET = a.Z_PK
        JOIN Z_1KEYWORDS j ON j.Z_1ASSETATTRIBUTES = aa.Z_PK
        JOIN ZKEYWORD k ON k.Z_PK = j.{keyword_join_col}
        WHERE k.ZTITLE = ?
        ORDER BY a.ZDATECREATED
    """

    cursor = conn.cursor()
    cursor.execute(query, (keyword_title,))
    return [row[0] for row in cursor.fetchall()]
```

## People and Faces

### Person Hierarchy

People (named individuals) contain multiple face detections:

```
ZPERSON (Person/Name)
  └── ZDETECTEDFACE (Individual face detection)
       └── ZASSET (Photo containing face)
```

### Getting All People

```sql
SELECT
    Z_PK,
    ZPERSONUUID,
    ZFULLNAME,
    ZDISPLAYNAME,
    ZFACECOUNT,
    ZTYPE,
    ZMANUALORDER
FROM ZPERSON
WHERE ZFACECOUNT > 0
ORDER BY ZMANUALORDER, ZFULLNAME;
```

**Person Types**:
- `ZTYPE = 1`: Favorite person
- `ZTYPE = -1` or `0`: Normal person
- `ZMANUALORDER`: Sort order for favorites

### Getting Faces for Person

```python
def get_faces_for_person(conn, person_uuid, face_asset_fk):
    """
    Get all face detections for a person.

    Args:
        conn: Database connection
        person_uuid: Person UUID
        face_asset_fk: Version-specific asset FK column name

    Returns:
        List of face records with photo info
    """
    # Determine person FK column
    cursor = conn.cursor()
    cursor.execute("PRAGMA table_info(ZDETECTEDFACE)")
    columns = [col[1] for col in cursor.fetchall()]

    if 'ZPERSONFORFACE' in columns:
        person_fk = 'ZPERSONFORFACE'
    else:
        person_fk = 'ZPERSON'

    # Determine asset table
    cursor.execute("SELECT name FROM sqlite_master WHERE type='table' AND name IN ('ZASSET', 'ZGENERICASSET')")
    asset_table = cursor.fetchone()[0]

    query = f"""
        SELECT
            df.ZUUID,
            df.ZCENTERX,
            df.ZCENTERY,
            df.ZSIZE,
            df.ZQUALITYMEASURE,
            a.ZUUID as photo_uuid,
            a.ZFILENAME
        FROM ZDETECTEDFACE df
        JOIN ZPERSON p ON p.Z_PK = df.{person_fk}
        JOIN {asset_table} a ON a.Z_PK = df.{face_asset_fk}
        WHERE p.ZPERSONUUID = ?
        ORDER BY df.ZQUALITYMEASURE DESC
    """

    cursor.execute(query, (person_uuid,))
    return cursor.fetchall()
```

### Getting People in Photo

```python
def get_people_in_photo(conn, photo_uuid, face_person_fk, face_asset_fk):
    """
    Get all named people in a photo.

    Args:
        conn: Database connection
        photo_uuid: Photo UUID
        face_person_fk: Person FK column name
        face_asset_fk: Asset FK column name

    Returns:
        List of person records with face positions
    """
    # Determine asset table
    cursor = conn.cursor()
    cursor.execute("SELECT name FROM sqlite_master WHERE type='table' AND name IN ('ZASSET', 'ZGENERICASSET')")
    asset_table = cursor.fetchone()[0]

    query = f"""
        SELECT
            p.ZPERSONUUID,
            p.ZFULLNAME,
            p.ZDISPLAYNAME,
            df.ZCENTERX,
            df.ZCENTERY,
            df.ZSIZE
        FROM ZPERSON p
        JOIN ZDETECTEDFACE df ON df.{face_person_fk} = p.Z_PK
        JOIN {asset_table} a ON a.Z_PK = df.{face_asset_fk}
        WHERE a.ZUUID = ?
    """

    cursor.execute(query, (photo_uuid,))
    return cursor.fetchall()
```

### Face Position Coordinates

Face positions are normalized (0.0 to 1.0):
- `ZCENTERX`: Center X coordinate (0.0 = left, 1.0 = right)
- `ZCENTERY`: Center Y coordinate (0.0 = top, 1.0 = bottom)
- `ZSIZE`: Face size (diameter as fraction of image)

```python
def get_face_rect(centerx, centery, size, image_width, image_height):
    """
    Convert normalized face coordinates to pixel rectangle.

    Args:
        centerx, centery, size: Normalized coordinates from database
        image_width, image_height: Image dimensions in pixels

    Returns:
        Dict with pixel coordinates: {x, y, width, height}
    """
    # Convert to pixels
    center_x_px = centerx * image_width
    center_y_px = centery * image_height

    # Size is diameter; calculate radius
    radius = (size * min(image_width, image_height)) / 2

    # Calculate rectangle
    return {
        'x': int(center_x_px - radius),
        'y': int(center_y_px - radius),
        'width': int(radius * 2),
        'height': int(radius * 2)
    }
```

## Moments

### Moment Structure

Moments are automatic groupings of photos by time and location:

```sql
SELECT
    Z_PK,
    ZUUID,
    ZTITLE,
    ZSUBTITLE,
    ZSTARTDATE,
    ZENDDATE,
    ZAPPROXIMATELATITUDE,
    ZAPPROXIMATELONGITUDE,
    ZCACHEDCOUNT
FROM ZMOMENT
ORDER BY ZSTARTDATE DESC;
```

### Getting Photos in Moment

```python
def get_photos_in_moment(conn, moment_uuid, asset_table):
    """
    Get photos in a moment.

    Args:
        conn: Database connection
        moment_uuid: Moment UUID
        asset_table: Asset table name

    Returns:
        List of photo records
    """
    query = f"""
        SELECT
            a.ZUUID,
            a.ZFILENAME,
            a.ZDATECREATED
        FROM {asset_table} a
        JOIN ZMOMENT m ON m.Z_PK = a.ZMOMENT
        WHERE m.ZUUID = ?
        ORDER BY a.ZDATECREATED
    """

    cursor = conn.cursor()
    cursor.execute(query, (moment_uuid,))
    return cursor.fetchall()
```

## Search and Labels

### Machine Learning Labels

ML labels stored in search database (`psi.sqlite`):

```python
def get_labels_for_photo(psi_conn, photo_uuid):
    """
    Get ML labels for photo from search database.

    Args:
        psi_conn: Connection to psi.sqlite
        photo_uuid: Photo UUID

    Returns:
        List of label strings
    """
    # Convert UUID to split integers
    uuid_obj = uuid.UUID(photo_uuid)
    uuid_int = uuid_obj.int
    uuid_0 = (uuid_int >> 64) & 0xFFFFFFFFFFFFFFFF
    uuid_1 = uuid_int & 0xFFFFFFFFFFFFFFFF

    query = """
        SELECT DISTINCT g.content_string
        FROM groups g
        JOIN ga ON ga.groupid = g.rowid
        JOIN assets a ON (a.uuid_0 || a.uuid_1) = CAST(ga.assetid AS TEXT)
        WHERE a.uuid_0 = ? AND a.uuid_1 = ?
        AND g.category = 2024
        ORDER BY g.content_string
    """

    cursor = psi_conn.cursor()
    cursor.execute(query, (uuid_0, uuid_1))
    return [row[0] for row in cursor.fetchall()]
```

### Search Categories

Common search categories (see [[02-database-schema#Search Categories]]):
- `2024`: ML labels
- `2029`: People
- `2030`: Places
- `2031`: Keywords
- `2032`: Titles
- `2036`: Media types

## Trash

### Trashed Items

Photos and albums can be in trash:

```sql
-- Trashed photos
SELECT ZUUID, ZFILENAME, ZTRASHEDDATE
FROM ZASSET
WHERE ZTRASHEDSTATE = 1
ORDER BY ZTRASHEDDATE DESC;

-- Trashed albums
SELECT ZUUID, ZTITLE, ZTRASHEDDATE
FROM ZGENERICALBUM
WHERE ZTRASHEDSTATE = 1
ORDER BY ZTRASHEDDATE DESC;
```

**States**:
- `ZTRASHEDSTATE = 0`: Not in trash
- `ZTRASHEDSTATE = 1`: In trash

Items in trash are soft-deleted and can be restored.

## Hidden Photos

Photos can be hidden:

```sql
SELECT ZUUID, ZFILENAME
FROM ZASSET
WHERE ZHIDDEN = 1
AND ZTRASHEDSTATE = 0;
```

Hidden photos don't appear in main library views but are still searchable.

## Related Documents

- [[02-database-schema]] - Database tables
- [[05-version-compatibility]] - Version differences
- [[07-special-formats]] - Special file types
- [[08-implementation-guide]] - Implementation examples

---

**Next**: [[07-special-formats]] - Special file types and conventions
