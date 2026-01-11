# Version Compatibility

**Related**: [[00-index]] | [[02-database-schema]] | [[08-implementation-guide]]

## Overview

The `.photoslibrary` format has evolved significantly across macOS versions. This document describes version detection and key differences between versions.

## Version Detection

### Step 1: Check Database Version

Query the database to determine Photos version:

```python
import sqlite3
import plistlib

def detect_photos_version(library_path):
    """
    Detect Photos version from library.

    Args:
        library_path: Path to .photoslibrary bundle

    Returns:
        Dict with version information
    """
    import os

    db_path = os.path.join(library_path, "database", "Photos.sqlite")

    # Check if modern database exists
    if not os.path.exists(db_path):
        # Check for legacy photos.db
        legacy_path = os.path.join(library_path, "database", "photos.db")
        if os.path.exists(legacy_path):
            return detect_legacy_version(legacy_path)
        return None

    # Open database
    conn = sqlite3.connect(f"file:{db_path}?mode=ro", uri=True)
    cursor = conn.cursor()

    # Get model version from Z_METADATA
    cursor.execute("SELECT Z_PLIST FROM Z_METADATA WHERE Z_VERSION = 1")
    row = cursor.fetchone()

    if not row:
        conn.close()
        return None

    # Parse binary plist
    plist = plistlib.loads(row[0])
    model_version = plist.get('PLModelVersion')

    conn.close()

    # Map model version to Photos version
    version_info = map_model_version(model_version)
    version_info['model_version'] = model_version

    return version_info
```

### Step 2: Map Model Version

```python
def map_model_version(model_version):
    """
    Map model version to Photos version info.

    Args:
        model_version: PLModelVersion integer

    Returns:
        Dict with version details
    """
    if model_version is None:
        return {'photos_version': 'Unknown', 'macos_version': 'Unknown'}

    # Version ranges
    if 13000 <= model_version <= 13999:
        return {
            'photos_version': 5,
            'photos_version_str': 'Photos 5',
            'macos_version': '10.15',
            'macos_name': 'Catalina'
        }
    elif 14000 <= model_version <= 14999:
        return {
            'photos_version': 6,
            'photos_version_str': 'Photos 6',
            'macos_version': '11',
            'macos_name': 'Big Sur'
        }
    elif 15000 <= model_version <= 15999:
        return {
            'photos_version': 7,
            'photos_version_str': 'Photos 7',
            'macos_version': '12',
            'macos_name': 'Monterey'
        }
    elif 16000 <= model_version <= 16999:
        return {
            'photos_version': 8,
            'photos_version_str': 'Photos 8',
            'macos_version': '13',
            'macos_name': 'Ventura'
        }
    elif 17000 <= model_version <= 17599:
        return {
            'photos_version': 9,
            'photos_version_str': 'Photos 9',
            'macos_version': '14.0-14.5',
            'macos_name': 'Sonoma'
        }
    elif 17600 <= model_version <= 17999:
        return {
            'photos_version': 9,
            'photos_version_str': 'Photos 9.6',
            'macos_version': '14.6+',
            'macos_name': 'Sonoma'
        }
    elif 18000 <= model_version <= 18200:
        return {
            'photos_version': 10,
            'photos_version_str': 'Photos 10 Beta',
            'macos_version': '15.0 beta',
            'macos_name': 'Sequoia Beta'
        }
    elif 18201 <= model_version <= 18999:
        return {
            'photos_version': 10,
            'photos_version_str': 'Photos 10',
            'macos_version': '15',
            'macos_name': 'Sequoia'
        }
    elif 19063 <= model_version <= 19319:
        return {
            'photos_version': 11,
            'photos_version_str': 'Photos 11',
            'macos_version': '16.0 / 26.0',
            'macos_name': 'Tahoe'
        }
    elif 19320 <= model_version <= 19999:
        return {
            'photos_version': 11,
            'photos_version_str': 'Photos 11.1',
            'macos_version': '26.1',
            'macos_name': 'Tahoe'
        }
    else:
        return {
            'photos_version': 'Unknown',
            'photos_version_str': f'Unknown (model {model_version})',
            'macos_version': 'Unknown'
        }
```

### Legacy Version Detection

```python
def detect_legacy_version(photos_db_path):
    """
    Detect version from legacy photos.db (Photos 2-4).

    Args:
        photos_db_path: Path to photos.db

    Returns:
        Dict with version information
    """
    conn = sqlite3.connect(f"file:{photos_db_path}?mode=ro", uri=True)
    cursor = conn.cursor()

    # Check LiGlobals table for version
    try:
        cursor.execute("SELECT value FROM LiGlobals WHERE keyPath = 'databaseVersion'")
        row = cursor.fetchone()
        if row:
            db_version = int(row[0])

            if db_version <= 2999:
                photos_version = 2
                macos = '10.12'
                name = 'Sierra'
            elif 3000 <= db_version <= 3999:
                photos_version = 3
                macos = '10.13'
                name = 'High Sierra'
            elif 4000 <= db_version <= 4999:
                photos_version = 4
                macos = '10.14'
                name = 'Mojave'
            else:
                photos_version = 'Unknown'
                macos = 'Unknown'
                name = 'Unknown'

            conn.close()
            return {
                'photos_version': photos_version,
                'photos_version_str': f'Photos {photos_version}',
                'macos_version': macos,
                'macos_name': name,
                'database_version': db_version,
                'legacy': True
            }
    except:
        pass

    conn.close()
    return None
```

## Version Comparison Table

| Photos | macOS | Model Version | DB Version | Asset Table | Album Join | Keyword Join |
|--------|-------|---------------|------------|-------------|------------|--------------|
| 2 | 10.12 Sierra | N/A | 2622 | photos.db | N/A | N/A |
| 3 | 10.13 High Sierra | N/A | 3301 | photos.db | N/A | N/A |
| 4 | 10.14 Mojave | N/A | 4025 | photos.db | N/A | N/A |
| 5 | 10.15 Catalina | 13000-13999 | 6000/5001 | ZGENERICASSET | Z_26ASSETS | Z_37KEYWORDS |
| 6 | 11 Big Sur | 14000-14999 | 6000 | ZASSET | Z_26ASSETS | Z_36KEYWORDS |
| 7 | 12 Monterey | 15000-15999 | 6000 | ZASSET | Z_27ASSETS | Z_38KEYWORDS |
| 8 | 13 Ventura | 16000-16999 | 6000 | ZASSET | Z_28ASSETS | Z_40KEYWORDS |
| 9 | 14.0-14.5 Sonoma | 17000-17599 | 6000 | ZASSET | Z_28ASSETS | Z_40KEYWORDS |
| 9.6 | 14.6+ Sonoma | 17600-17999 | 6000 | ZASSET | Z_29ASSETS | Z_41KEYWORDS |
| 10β | 15.0β Sequoia | 18000-18200 | 6000 | ZASSET | Z_31ASSETS | Z_48KEYWORDS |
| 10 | 15 Sequoia | 18201-18999 | 6000 | ZASSET | Z_30ASSETS | Z_47KEYWORDS |
| 11 | 16/26 Tahoe | 19063-19319 | 6000 | ZASSET | Z_32ASSETS | Z_51KEYWORDS |
| 11.1 | 26.1 Tahoe | 19320-19999 | 6000 | ZASSET | Z_33ASSETS | Z_52KEYWORDS |

## Key Schema Differences

### Photos 5 (Catalina)

**Introduced**:
- Modern Photos.sqlite format
- ZGENERICASSET table for assets
- Search database (psi.sqlite)
- ML labels and computer vision
- ZMASTERFINGERPRINT for deduplication

**Asset Table**: `ZGENERICASSET`
**Join Tables**: `Z_26ASSETS`, `Z_37KEYWORDS`

### Photos 6 (Big Sur)

**Changes**:
- Asset table renamed to `ZASSET`
- Improved cloud support
- Join table numbers updated: `Z_26ASSETS`, `Z_36KEYWORDS`

**Breaking Changes**:
- Code must check for `ZASSET` vs `ZGENERICASSET`
- Keyword join table column renamed

### Photos 7 (Monterey)

**Changes**:
- Added `ZIMPORTEDBYDISPLAYNAME` and `ZIMPORTEDBYBUNDLEIDENTIFIER`
- HDR column: `ZCUSTOMRENDEREDVALUE` → `ZHDRTYPE`
- Syndication support added
- Shared library enhancements
- Join tables: `Z_27ASSETS`, `Z_38KEYWORDS`

**Breaking Changes**:
- HDR detection column changed

### Photos 8 (Ventura)

**Changes**:
- Face foreign keys renamed:
  - `ZDETECTEDFACE.ZPERSON` → `ZPERSONFORFACE`
  - `ZDETECTEDFACE.ZASSET` → `ZASSETFORFACE`
- Join tables: `Z_28ASSETS`, `Z_40KEYWORDS`
- Enhanced shared library support

**Breaking Changes**:
- Face relationship queries must use new column names

### Photos 9 (Sonoma)

**Changes**:
- Continued schema evolution
- Join tables remain `Z_28ASSETS`, `Z_40KEYWORDS`

### Photos 9.6 (Sonoma 14.6+)

**Changes**:
- Join tables: `Z_29ASSETS`, `Z_41KEYWORDS`

**Breaking Changes**:
- Album-asset relationships incompatible with earlier Photos 9

### Photos 10 Beta (Sequoia Beta)

**Changes**:
- Join tables: `Z_31ASSETS`, `Z_48KEYWORDS`
- `ZORIGINALSTABLEHASH` replaces `ZMASTERFINGERPRINT`
- `ZADJUSTMENTSSTATE` replaces `ZHASADJUSTMENTS`

**Breaking Changes**:
- Fingerprint mechanism changed
- Adjustment detection column changed

### Photos 10 (Sequoia)

**Changes**:
- Join tables: `Z_30ASSETS`, `Z_47KEYWORDS`
- Final schema adjustments from beta

### Photos 11 (Tahoe)

**Changes**:
- Join tables: `Z_32ASSETS`, `Z_51KEYWORDS`
- macOS versioning changed to 26.x for Apple Silicon

### Photos 11.1 (Tahoe 26.1)

**Changes**:
- Join tables: `Z_33ASSETS`, `Z_52KEYWORDS`

## Version-Adaptive Queries

### Detecting Asset Table Name

```python
def get_asset_table_name(conn):
    """
    Detect correct asset table name for this library version.

    Args:
        conn: Database connection

    Returns:
        'ZGENERICASSET' or 'ZASSET'
    """
    cursor = conn.cursor()

    # Check if ZASSET exists
    cursor.execute("""
        SELECT name FROM sqlite_master
        WHERE type='table' AND name='ZASSET'
    """)

    if cursor.fetchone():
        return 'ZASSET'
    else:
        return 'ZGENERICASSET'
```

### Detecting Join Table Names

```python
def get_album_join_table(conn):
    """
    Detect album-asset join table name.

    Args:
        conn: Database connection

    Returns:
        Table name (e.g., 'Z_26ASSETS')
    """
    cursor = conn.cursor()

    # Find table matching pattern Z_[0-9]+ASSETS
    cursor.execute("""
        SELECT name FROM sqlite_master
        WHERE type='table' AND name LIKE 'Z_%ASSETS'
        AND name NOT LIKE 'Z_RT_%'
        ORDER BY name
    """)

    tables = cursor.fetchall()

    # Return the first matching table
    # Usually there's only one
    if tables:
        return tables[0][0]

    return None

def get_keyword_join_column(conn):
    """
    Detect keyword join column name in Z_1KEYWORDS table.

    Args:
        conn: Database connection

    Returns:
        Column name (e.g., 'Z_37KEYWORDS')
    """
    cursor = conn.cursor()

    # Get column names from Z_1KEYWORDS
    cursor.execute("PRAGMA table_info(Z_1KEYWORDS)")
    columns = cursor.fetchall()

    # Find column matching pattern Z_[0-9]+KEYWORDS
    for col in columns:
        col_name = col[1]
        if col_name != 'Z_1ASSETATTRIBUTES' and 'KEYWORDS' in col_name:
            return col_name

    return None
```

### Version-Adaptive Asset Query

```python
def query_assets_adaptive(conn):
    """
    Query assets with version-adaptive table name.

    Args:
        conn: Database connection

    Returns:
        List of asset rows
    """
    asset_table = get_asset_table_name(conn)

    query = f"""
        SELECT
            Z_PK,
            ZUUID,
            ZFILENAME,
            ZDATECREATED,
            ZFAVORITE,
            ZHIDDEN
        FROM {asset_table}
        WHERE ZTRASHEDSTATE = 0
    """

    cursor = conn.cursor()
    cursor.execute(query)
    return cursor.fetchall()
```

### Face Query with Version Handling

```python
def query_faces_adaptive(conn, asset_pk):
    """
    Query faces with version-adaptive column names.

    Args:
        conn: Database connection
        asset_pk: Asset primary key

    Returns:
        List of face rows with person info
    """
    cursor = conn.cursor()

    # Detect which column names to use
    cursor.execute("PRAGMA table_info(ZDETECTEDFACE)")
    columns = [col[1] for col in cursor.fetchall()]

    if 'ZPERSONFORFACE' in columns:
        # Photos 8+
        person_fk = 'ZPERSONFORFACE'
        asset_fk = 'ZASSETFORFACE'
    else:
        # Photos 5-7
        person_fk = 'ZPERSON'
        asset_fk = 'ZASSET'

    query = f"""
        SELECT
            df.ZUUID,
            df.ZCENTERX,
            df.ZCENTERY,
            df.ZSIZE,
            p.ZFULLNAME,
            p.ZDISPLAYNAME
        FROM ZDETECTEDFACE df
        LEFT JOIN ZPERSON p ON p.Z_PK = df.{person_fk}
        WHERE df.{asset_fk} = ?
    """

    cursor.execute(query, (asset_pk,))
    return cursor.fetchall()
```

## Backwards Compatibility

### Reading Older Libraries

Libraries created in older versions can be read by newer Photos versions:
- Photos automatically migrates schema on first open
- Migration is one-way (cannot downgrade)
- Backup before opening in newer Photos

### Reading Newer Libraries

Older Photos versions **cannot** read newer library formats:
- Incompatible schema
- Missing tables/columns
- Photos will refuse to open or show error

### Cross-Version Access

For read-only access from external tools:
- Use version detection
- Adapt queries to detected version
- Handle missing columns gracefully
- Don't assume specific table names

## Migration Notes

### Schema Migrations

When Photos upgrades a library:
1. Creates backup in `~/Pictures` (optional)
2. Updates database schema
3. Migrates data to new structures
4. Updates model version in Z_METADATA
5. May regenerate derivatives

### Post-Migration

After migration:
- Original library incompatible with older Photos
- File structure remains mostly same
- Database schema updated
- Some columns renamed/added/removed

## Version-Specific Features

| Feature | Minimum Version |
|---------|-----------------|
| Photos.sqlite format | Photos 5 |
| Machine learning labels | Photos 5 |
| Search database (psi.sqlite) | Photos 5 |
| Import source tracking | Photos 7 |
| Shared libraries | Photos 7 |
| Stable hash fingerprints | Photos 10 |
| New adjustment state | Photos 10 |

## Best Practices

### For Library Readers

1. **Always detect version first**
2. **Use adaptive queries** based on detected version
3. **Handle missing columns** gracefully
4. **Test against multiple versions**
5. **Document supported version range**

### For Library Writers

1. **Never modify library database**
2. **Only read data**
3. **Don't assume schema**
4. **Handle version mismatches**
5. **Warn users about compatibility**

### Example Version-Aware Initialization

```python
class PhotosLibrary:
    def __init__(self, library_path):
        self.library_path = library_path
        self.version_info = detect_photos_version(library_path)

        if not self.version_info:
            raise ValueError("Cannot detect Photos version")

        self.conn = self._open_database()
        self._detect_schema()

    def _open_database(self):
        db_path = os.path.join(self.library_path, "database", "Photos.sqlite")
        conn = sqlite3.connect(f"file:{db_path}?mode=ro", uri=True)
        conn.execute("PRAGMA query_only = ON")
        return conn

    def _detect_schema(self):
        """Detect version-specific schema details."""
        self.asset_table = get_asset_table_name(self.conn)
        self.album_join_table = get_album_join_table(self.conn)
        self.keyword_join_column = get_keyword_join_column(self.conn)

        # Detect face column names
        cursor = self.conn.cursor()
        cursor.execute("PRAGMA table_info(ZDETECTEDFACE)")
        columns = [col[1] for col in cursor.fetchall()]

        if 'ZPERSONFORFACE' in columns:
            self.face_person_fk = 'ZPERSONFORFACE'
            self.face_asset_fk = 'ZASSETFORFACE'
        else:
            self.face_person_fk = 'ZPERSON'
            self.face_asset_fk = 'ZASSET'

    def get_photos(self):
        """Get all photos using detected schema."""
        query = f"""
            SELECT Z_PK, ZUUID, ZFILENAME
            FROM {self.asset_table}
            WHERE ZTRASHEDSTATE = 0
        """
        cursor = self.conn.cursor()
        cursor.execute(query)
        return cursor.fetchall()
```

## Related Documents

- [[02-database-schema]] - Complete schema reference
- [[08-implementation-guide]] - Implementation examples
- [[00-index]] - Overview

---

**Next**: [[06-organizational-structures]] - Albums, folders, and organization
