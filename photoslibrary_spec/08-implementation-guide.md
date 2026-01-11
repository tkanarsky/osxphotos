# Implementation Guide

**Related**: [[00-index]] | [[02-database-schema]] | [[05-version-compatibility]]

## Overview

This document provides practical implementation guidance and complete code examples for building a Photos library reader. It demonstrates best practices and common use cases.

## Complete Library Reader Example

### Basic Library Class

```python
import sqlite3
import plistlib
import os
from datetime import datetime, timezone
from pathlib import Path

class PhotosLibrary:
    """
    Read-only Photos library reader.

    Example:
        lib = PhotosLibrary('/path/to/Library.photoslibrary')
        print(f"Photos version: {lib.version_info['photos_version']}")
        photos = lib.get_all_photos()
    """

    def __init__(self, library_path):
        """
        Initialize library reader.

        Args:
            library_path: Path to .photoslibrary bundle

        Raises:
            ValueError: If library cannot be opened
        """
        self.library_path = Path(library_path)

        if not self.library_path.exists():
            raise ValueError(f"Library not found: {library_path}")

        # Detect version
        self.version_info = self._detect_version()
        if not self.version_info:
            raise ValueError("Cannot detect Photos version")

        # Open database
        self.conn = self._open_database()

        # Detect schema
        self._detect_schema()

        # Open search database if available
        self.psi_conn = self._open_search_database()

    def _detect_version(self):
        """Detect Photos version from library."""
        db_path = self.library_path / "database" / "Photos.sqlite"

        if not db_path.exists():
            return None

        conn = sqlite3.connect(f"file:{db_path}?mode=ro", uri=True)
        cursor = conn.cursor()

        try:
            cursor.execute("SELECT Z_PLIST FROM Z_METADATA WHERE Z_VERSION = 1")
            row = cursor.fetchone()

            if not row:
                return None

            plist = plistlib.loads(row[0])
            model_version = plist.get('PLModelVersion')

            conn.close()

            # Map model version to Photos version
            return self._map_model_version(model_version)
        except Exception as e:
            conn.close()
            return None

    def _map_model_version(self, model_version):
        """Map model version to Photos version info."""
        if model_version is None:
            return None

        version_ranges = [
            (13000, 13999, 5, '10.15', 'Catalina'),
            (14000, 14999, 6, '11', 'Big Sur'),
            (15000, 15999, 7, '12', 'Monterey'),
            (16000, 16999, 8, '13', 'Ventura'),
            (17000, 17599, 9, '14.0-14.5', 'Sonoma'),
            (17600, 17999, 9, '14.6+', 'Sonoma'),
            (18000, 18200, 10, '15.0 beta', 'Sequoia Beta'),
            (18201, 18999, 10, '15', 'Sequoia'),
            (19063, 19319, 11, '16/26', 'Tahoe'),
            (19320, 19999, 11, '26.1', 'Tahoe'),
        ]

        for min_v, max_v, photos_v, macos_v, macos_name in version_ranges:
            if min_v <= model_version <= max_v:
                return {
                    'photos_version': photos_v,
                    'macos_version': macos_v,
                    'macos_name': macos_name,
                    'model_version': model_version
                }

        return None

    def _open_database(self):
        """Open Photos database in read-only mode."""
        db_path = self.library_path / "database" / "Photos.sqlite"
        conn = sqlite3.connect(f"file:{db_path}?mode=ro", uri=True)
        conn.row_factory = sqlite3.Row  # Enable column access by name
        conn.execute("PRAGMA query_only = ON")
        return conn

    def _open_search_database(self):
        """Open search database if available."""
        psi_path = self.library_path / "database" / "search" / "psi.sqlite"

        if not psi_path.exists():
            return None

        try:
            conn = sqlite3.connect(f"file:{psi_path}?mode=ro", uri=True)
            conn.row_factory = sqlite3.Row
            conn.execute("PRAGMA query_only = ON")
            return conn
        except:
            return None

    def _detect_schema(self):
        """Detect version-specific schema elements."""
        cursor = self.conn.cursor()

        # Detect asset table name
        cursor.execute("""
            SELECT name FROM sqlite_master
            WHERE type='table' AND name IN ('ZASSET', 'ZGENERICASSET')
        """)
        row = cursor.fetchone()
        self.asset_table = row[0] if row else 'ZASSET'

        # Detect album join table
        cursor.execute("""
            SELECT name FROM sqlite_master
            WHERE type='table' AND name LIKE 'Z_%ASSETS'
            AND name NOT LIKE 'Z_RT_%'
            ORDER BY name
        """)
        row = cursor.fetchone()
        self.album_join_table = row[0] if row else None

        # Detect keyword join column
        cursor.execute("PRAGMA table_info(Z_1KEYWORDS)")
        columns = [col[1] for col in cursor.fetchall()]
        self.keyword_join_col = next(
            (c for c in columns if c != 'Z_1ASSETATTRIBUTES' and 'KEYWORDS' in c),
            None
        )

        # Detect face foreign key names
        cursor.execute("PRAGMA table_info(ZDETECTEDFACE)")
        columns = [col[1] for col in cursor.fetchall()]

        if 'ZPERSONFORFACE' in columns:
            self.face_person_fk = 'ZPERSONFORFACE'
            self.face_asset_fk = 'ZASSETFORFACE'
        else:
            self.face_person_fk = 'ZPERSON'
            self.face_asset_fk = 'ZASSET'

    def close(self):
        """Close database connections."""
        if self.conn:
            self.conn.close()
        if self.psi_conn:
            self.psi_conn.close()

    def __enter__(self):
        """Context manager entry."""
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        """Context manager exit."""
        self.close()

    # Core Data timestamp conversion
    CORE_DATA_EPOCH = datetime(2001, 1, 1, tzinfo=timezone.utc).timestamp()

    def _cd_to_datetime(self, timestamp):
        """Convert Core Data timestamp to datetime."""
        if timestamp is None:
            return None
        return datetime.fromtimestamp(
            timestamp + self.CORE_DATA_EPOCH,
            tz=timezone.utc
        )

    def get_all_photos(self, include_hidden=False, include_trashed=False):
        """
        Get all photos in library.

        Args:
            include_hidden: Include hidden photos
            include_trashed: Include trashed photos

        Returns:
            List of photo dicts
        """
        conditions = []
        if not include_trashed:
            conditions.append("ZTRASHEDSTATE = 0")
        if not include_hidden:
            conditions.append("ZHIDDEN = 0")

        where_clause = f"WHERE {' AND '.join(conditions)}" if conditions else ""

        query = f"""
            SELECT
                a.Z_PK,
                a.ZUUID,
                a.ZFILENAME,
                a.ZKIND,
                a.ZKINDSUBTYPE,
                a.ZDATECREATED,
                a.ZMODIFICATIONDATE,
                a.ZLATITUDE,
                a.ZLONGITUDE,
                a.ZHEIGHT,
                a.ZWIDTH,
                a.ZFAVORITE,
                a.ZHIDDEN,
                aa.ZTITLE,
                aa.ZORIGINALFILENAME
            FROM {self.asset_table} a
            LEFT JOIN ZADDITIONALASSETATTRIBUTES aa ON aa.ZASSET = a.Z_PK
            {where_clause}
            ORDER BY a.ZDATECREATED DESC
        """

        cursor = self.conn.cursor()
        cursor.execute(query)

        photos = []
        for row in cursor.fetchall():
            photos.append({
                'pk': row['Z_PK'],
                'uuid': row['ZUUID'],
                'filename': row['ZFILENAME'],
                'kind': 'photo' if row['ZKIND'] == 0 else 'video',
                'subtype': row['ZKINDSUBTYPE'],
                'date_created': self._cd_to_datetime(row['ZDATECREATED']),
                'date_modified': self._cd_to_datetime(row['ZMODIFICATIONDATE']),
                'latitude': row['ZLATITUDE'] if row['ZLATITUDE'] != -180.0 else None,
                'longitude': row['ZLONGITUDE'] if row['ZLONGITUDE'] != -180.0 else None,
                'height': row['ZHEIGHT'],
                'width': row['ZWIDTH'],
                'favorite': bool(row['ZFAVORITE']),
                'hidden': bool(row['ZHIDDEN']),
                'title': row['ZTITLE'],
                'original_filename': row['ZORIGINALFILENAME']
            })

        return photos

    def get_photo(self, uuid):
        """
        Get single photo by UUID.

        Args:
            uuid: Photo UUID

        Returns:
            Photo dict or None
        """
        query = f"""
            SELECT
                a.Z_PK,
                a.ZUUID,
                a.ZFILENAME,
                a.ZKIND,
                a.ZKINDSUBTYPE,
                a.ZDATECREATED,
                a.ZMODIFICATIONDATE,
                a.ZLATITUDE,
                a.ZLONGITUDE,
                a.ZHEIGHT,
                a.ZWIDTH,
                a.ZFAVORITE,
                a.ZHIDDEN,
                aa.ZTITLE,
                aa.ZORIGINALFILENAME,
                aa.ZTIMEZONENAME,
                aa.ZTIMEZONEOFFSET
            FROM {self.asset_table} a
            LEFT JOIN ZADDITIONALASSETATTRIBUTES aa ON aa.ZASSET = a.Z_PK
            WHERE a.ZUUID = ?
        """

        cursor = self.conn.cursor()
        cursor.execute(query, (uuid,))
        row = cursor.fetchone()

        if not row:
            return None

        return {
            'pk': row['Z_PK'],
            'uuid': row['ZUUID'],
            'filename': row['ZFILENAME'],
            'kind': 'photo' if row['ZKIND'] == 0 else 'video',
            'subtype': row['ZKINDSUBTYPE'],
            'date_created': self._cd_to_datetime(row['ZDATECREATED']),
            'date_modified': self._cd_to_datetime(row['ZMODIFICATIONDATE']),
            'latitude': row['ZLATITUDE'] if row['ZLATITUDE'] != -180.0 else None,
            'longitude': row['ZLONGITUDE'] if row['ZLONGITUDE'] != -180.0 else None,
            'height': row['ZHEIGHT'],
            'width': row['ZWIDTH'],
            'favorite': bool(row['ZFAVORITE']),
            'hidden': bool(row['ZHIDDEN']),
            'title': row['ZTITLE'],
            'original_filename': row['ZORIGINALFILENAME'],
            'timezone_name': row['ZTIMEZONENAME'],
            'timezone_offset': row['ZTIMEZONEOFFSET']
        }

    def get_file_path(self, uuid, uti='public.jpeg'):
        """
        Get file path for photo.

        Args:
            uuid: Photo UUID
            uti: UTI to determine extension

        Returns:
            Path to file
        """
        # UTI to extension mapping
        uti_map = {
            'public.jpeg': 'jpg',
            'public.heic': 'heic',
            'public.png': 'png',
            'com.apple.quicktime-movie': 'mov',
            'public.mpeg-4': 'mp4',
        }

        ext = uti_map.get(uti, 'jpg')
        first_char = uuid[0].upper()

        return self.library_path / "originals" / first_char / f"{uuid}.{ext}"

    def get_albums(self):
        """
        Get all user albums.

        Returns:
            List of album dicts
        """
        query = """
            SELECT
                Z_PK,
                ZUUID,
                ZTITLE,
                ZKIND,
                ZCREATIONDATE,
                ZCACHEDCOUNT,
                ZPARENTFOLDER
            FROM ZGENERICALBUM
            WHERE ZKIND = 2
            AND ZTRASHEDSTATE = 0
            ORDER BY ZTITLE
        """

        cursor = self.conn.cursor()
        cursor.execute(query)

        albums = []
        for row in cursor.fetchall():
            albums.append({
                'pk': row['Z_PK'],
                'uuid': row['ZUUID'],
                'title': row['ZTITLE'],
                'kind': row['ZKIND'],
                'date_created': self._cd_to_datetime(row['ZCREATIONDATE']),
                'photo_count': row['ZCACHEDCOUNT'],
                'parent_folder': row['ZPARENTFOLDER']
            })

        return albums

    def get_photos_in_album(self, album_uuid):
        """
        Get photos in album.

        Args:
            album_uuid: Album UUID

        Returns:
            List of photo dicts
        """
        if not self.album_join_table:
            return []

        # Detect column names
        cursor = self.conn.cursor()
        cursor.execute(f"PRAGMA table_info({self.album_join_table})")
        columns = [col[1] for col in cursor.fetchall()]

        album_col = next((c for c in columns if 'ALBUMS' in c), None)
        asset_col = next((c for c in columns if 'ASSETS' in c and c != album_col), None)

        if not album_col or not asset_col:
            return []

        query = f"""
            SELECT a.ZUUID, a.ZFILENAME
            FROM {self.asset_table} a
            JOIN {self.album_join_table} j ON j.{asset_col} = a.Z_PK
            JOIN ZGENERICALBUM alb ON alb.Z_PK = j.{album_col}
            WHERE alb.ZUUID = ?
            ORDER BY a.ZDATECREATED
        """

        cursor.execute(query, (album_uuid,))

        return [{'uuid': row[0], 'filename': row[1]} for row in cursor.fetchall()]

    def get_keywords(self, photo_uuid):
        """
        Get keywords for photo.

        Args:
            photo_uuid: Photo UUID

        Returns:
            List of keyword strings
        """
        if not self.keyword_join_col:
            return []

        query = f"""
            SELECT k.ZTITLE
            FROM ZKEYWORD k
            JOIN Z_1KEYWORDS j ON j.{self.keyword_join_col} = k.Z_PK
            JOIN ZADDITIONALASSETATTRIBUTES aa ON aa.Z_PK = j.Z_1ASSETATTRIBUTES
            JOIN {self.asset_table} a ON a.Z_PK = aa.ZASSET
            WHERE a.ZUUID = ?
            ORDER BY k.ZTITLE
        """

        cursor = self.conn.cursor()
        cursor.execute(query, (photo_uuid,))

        return [row[0] for row in cursor.fetchall()]

    def get_people(self):
        """
        Get all named people.

        Returns:
            List of person dicts
        """
        query = """
            SELECT
                Z_PK,
                ZPERSONUUID,
                ZFULLNAME,
                ZDISPLAYNAME,
                ZFACECOUNT,
                ZTYPE
            FROM ZPERSON
            WHERE ZFACECOUNT > 0
            ORDER BY ZFULLNAME
        """

        cursor = self.conn.cursor()
        cursor.execute(query)

        people = []
        for row in cursor.fetchall():
            people.append({
                'pk': row['Z_PK'],
                'uuid': row['ZPERSONUUID'],
                'full_name': row['ZFULLNAME'],
                'display_name': row['ZDISPLAYNAME'],
                'face_count': row['ZFACECOUNT'],
                'is_favorite': row['ZTYPE'] == 1
            })

        return people

    def get_faces_in_photo(self, photo_uuid):
        """
        Get faces detected in photo.

        Args:
            photo_uuid: Photo UUID

        Returns:
            List of face dicts with person info
        """
        query = f"""
            SELECT
                p.ZPERSONUUID,
                p.ZFULLNAME,
                df.ZCENTERX,
                df.ZCENTERY,
                df.ZSIZE
            FROM ZPERSON p
            JOIN ZDETECTEDFACE df ON df.{self.face_person_fk} = p.Z_PK
            JOIN {self.asset_table} a ON a.Z_PK = df.{self.face_asset_fk}
            WHERE a.ZUUID = ?
        """

        cursor = self.conn.cursor()
        cursor.execute(query, (photo_uuid,))

        faces = []
        for row in cursor.fetchall():
            faces.append({
                'person_uuid': row[0],
                'person_name': row[1],
                'center_x': row[2],
                'center_y': row[3],
                'size': row[4]
            })

        return faces
```

## Usage Examples

### Example 1: List All Photos

```python
# Open library
with PhotosLibrary('/Users/username/Pictures/Photos Library.photoslibrary') as lib:
    print(f"Photos version: {lib.version_info['photos_version']}")
    print(f"macOS: {lib.version_info['macos_name']}")

    # Get all photos
    photos = lib.get_all_photos()

    print(f"\nTotal photos: {len(photos)}")

    # Print first 10
    for photo in photos[:10]:
        print(f"  {photo['date_created']}: {photo['filename']}")
        if photo['title']:
            print(f"    Title: {photo['title']}")
```

### Example 2: Export Photo with Metadata

```python
import shutil
from pathlib import Path

def export_photo_with_metadata(lib, photo_uuid, export_dir):
    """
    Export photo with all metadata.

    Args:
        lib: PhotosLibrary instance
        photo_uuid: UUID of photo to export
        export_dir: Directory to export to
    """
    export_dir = Path(export_dir)
    export_dir.mkdir(parents=True, exist_ok=True)

    # Get photo info
    photo = lib.get_photo(photo_uuid)
    if not photo:
        print(f"Photo {photo_uuid} not found")
        return

    # Get file path
    file_path = lib.get_file_path(photo['uuid'])

    if not file_path.exists():
        print(f"File not found: {file_path}")
        return

    # Copy file
    dest_path = export_dir / photo['filename']
    shutil.copy2(file_path, dest_path)

    print(f"Exported: {dest_path}")

    # Get metadata
    keywords = lib.get_keywords(photo['uuid'])
    faces = lib.get_faces_in_photo(photo['uuid'])

    # Write metadata file
    metadata = {
        'filename': photo['filename'],
        'date_created': str(photo['date_created']),
        'title': photo['title'],
        'location': {
            'latitude': photo['latitude'],
            'longitude': photo['longitude']
        } if photo['latitude'] else None,
        'dimensions': {
            'width': photo['width'],
            'height': photo['height']
        },
        'keywords': keywords,
        'people': [f['person_name'] for f in faces],
        'favorite': photo['favorite']
    }

    import json
    metadata_path = export_dir / f"{photo['filename']}.json"
    with open(metadata_path, 'w') as f:
        json.dump(metadata, f, indent=2)

    print(f"Metadata: {metadata_path}")

# Usage
with PhotosLibrary('/path/to/Library.photoslibrary') as lib:
    export_photo_with_metadata(lib, 'SOME-UUID-HERE', '/tmp/export')
```

### Example 3: Generate Photo Report

```python
def generate_library_report(library_path):
    """Generate summary report of Photos library."""

    with PhotosLibrary(library_path) as lib:
        print("=" * 60)
        print("PHOTOS LIBRARY REPORT")
        print("=" * 60)

        print(f"\nLibrary: {library_path}")
        print(f"Photos Version: {lib.version_info['photos_version']}")
        print(f"macOS Version: {lib.version_info['macos_version']} ({lib.version_info['macos_name']})")

        # Count photos
        photos = lib.get_all_photos()
        videos = [p for p in photos if p['kind'] == 'video']
        favorites = [p for p in photos if p['favorite']]

        print(f"\n=== MEDIA ===")
        print(f"Total items: {len(photos)}")
        print(f"Photos: {len(photos) - len(videos)}")
        print(f"Videos: {len(videos)}")
        print(f"Favorites: {len(favorites)}")

        # Count special types
        live_photos = [p for p in photos if p['subtype'] == 2]
        panoramas = [p for p in photos if p['subtype'] == 1]
        screenshots = [p for p in photos if p['subtype'] == 10]

        print(f"\nLive Photos: {len(live_photos)}")
        print(f"Panoramas: {len(panoramas)}")
        print(f"Screenshots: {len(screenshots)}")

        # Albums
        albums = lib.get_albums()
        print(f"\n=== ALBUMS ===")
        print(f"Total albums: {len(albums)}")

        # People
        people = lib.get_people()
        print(f"\n=== PEOPLE ===")
        print(f"Named people: {len(people)}")

        if people:
            print("\nTop 10 people:")
            sorted_people = sorted(people, key=lambda p: p['face_count'], reverse=True)
            for person in sorted_people[:10]:
                print(f"  {person['full_name']}: {person['face_count']} faces")

        # Date range
        if photos:
            dates = [p['date_created'] for p in photos if p['date_created']]
            if dates:
                print(f"\n=== TIMELINE ===")
                print(f"Oldest photo: {min(dates)}")
                print(f"Newest photo: {max(dates)}")

        print("\n" + "=" * 60)

# Usage
generate_library_report('/Users/username/Pictures/Photos Library.photoslibrary')
```

### Example 4: Find Photos by Criteria

```python
def find_photos(lib, **criteria):
    """
    Find photos matching criteria.

    Args:
        lib: PhotosLibrary instance
        **criteria: Search criteria (keyword, person, date_from, date_to, favorite, location)

    Returns:
        List of matching photos
    """
    photos = lib.get_all_photos()

    # Filter by keyword
    if 'keyword' in criteria:
        keyword_photos = set()
        for photo in photos:
            keywords = lib.get_keywords(photo['uuid'])
            if criteria['keyword'] in keywords:
                keyword_photos.add(photo['uuid'])

        photos = [p for p in photos if p['uuid'] in keyword_photos]

    # Filter by person
    if 'person' in criteria:
        person_photos = set()
        for photo in photos:
            faces = lib.get_faces_in_photo(photo['uuid'])
            if any(criteria['person'].lower() in (f['person_name'] or '').lower() for f in faces):
                person_photos.add(photo['uuid'])

        photos = [p for p in photos if p['uuid'] in person_photos]

    # Filter by favorite
    if 'favorite' in criteria:
        photos = [p for p in photos if p['favorite'] == criteria['favorite']]

    # Filter by date range
    if 'date_from' in criteria:
        photos = [p for p in photos if p['date_created'] >= criteria['date_from']]

    if 'date_to' in criteria:
        photos = [p for p in photos if p['date_created'] <= criteria['date_to']]

    # Filter by location
    if 'has_location' in criteria:
        if criteria['has_location']:
            photos = [p for p in photos if p['latitude'] is not None]
        else:
            photos = [p for p in photos if p['latitude'] is None]

    return photos

# Usage
from datetime import datetime

with PhotosLibrary('/path/to/Library.photoslibrary') as lib:
    # Find favorite photos with a specific person
    results = find_photos(
        lib,
        person="John",
        favorite=True
    )

    print(f"Found {len(results)} photos")
    for photo in results:
        print(f"  {photo['filename']}")

    # Find photos from 2023
    results = find_photos(
        lib,
        date_from=datetime(2023, 1, 1, tzinfo=timezone.utc),
        date_to=datetime(2024, 1, 1, tzinfo=timezone.utc)
    )

    print(f"\nPhotos from 2023: {len(results)}")
```

## Performance Optimization

### Batch Loading

```python
def get_photos_with_metadata_batch(lib, photo_uuids):
    """
    Load multiple photos with metadata efficiently.

    Args:
        lib: PhotosLibrary instance
        photo_uuids: List of photo UUIDs

    Returns:
        Dict mapping UUID to photo data with metadata
    """
    if not photo_uuids:
        return {}

    # Build placeholders for IN clause
    placeholders = ','.join('?' * len(photo_uuids))

    # Single query for all photos
    query = f"""
        SELECT
            a.ZUUID,
            a.ZFILENAME,
            a.ZDATECREATED,
            aa.ZTITLE
        FROM {lib.asset_table} a
        LEFT JOIN ZADDITIONALASSETATTRIBUTES aa ON aa.ZASSET = a.Z_PK
        WHERE a.ZUUID IN ({placeholders})
    """

    cursor = lib.conn.cursor()
    cursor.execute(query, photo_uuids)

    result = {}
    for row in cursor.fetchall():
        result[row['ZUUID']] = {
            'uuid': row['ZUUID'],
            'filename': row['ZFILENAME'],
            'date_created': lib._cd_to_datetime(row['ZDATECREATED']),
            'title': row['ZTITLE']
        }

    return result
```

### Caching

```python
class CachedPhotosLibrary(PhotosLibrary):
    """Photos library with caching."""

    def __init__(self, library_path):
        super().__init__(library_path)
        self._photo_cache = {}
        self._album_cache = {}

    def get_photo(self, uuid):
        """Get photo with caching."""
        if uuid not in self._photo_cache:
            self._photo_cache[uuid] = super().get_photo(uuid)
        return self._photo_cache[uuid]

    def get_albums(self):
        """Get albums with caching."""
        if not self._album_cache:
            albums = super().get_albums()
            for album in albums:
                self._album_cache[album['uuid']] = album
        return list(self._album_cache.values())
```

## Error Handling

### Robust File Access

```python
def safe_get_file(lib, photo_uuid):
    """
    Safely get photo file with fallbacks.

    Args:
        lib: PhotosLibrary instance
        photo_uuid: Photo UUID

    Returns:
        Path to available file or None
    """
    # Try original
    original = lib.get_file_path(photo_uuid, 'public.heic')
    if original.exists():
        return original

    # Try JPEG alternative
    original = lib.get_file_path(photo_uuid, 'public.jpeg')
    if original.exists():
        return original

    # Try derivatives
    first_char = photo_uuid[0].upper()
    derivatives = [
        f"{photo_uuid}_4_5005_c.jpeg",
        f"{photo_uuid}_1_201_c.jpeg",
        f"{photo_uuid}_1_105_c.jpeg"
    ]

    for deriv in derivatives:
        deriv_path = lib.library_path / "resources" / "derivatives" / first_char / deriv
        if deriv_path.exists():
            return deriv_path

    return None
```

## Testing

### Unit Test Example

```python
import unittest
from pathlib import Path

class TestPhotosLibrary(unittest.TestCase):
    """Test Photos library reader."""

    @classmethod
    def setUpClass(cls):
        """Set up test library."""
        cls.library_path = Path(os.environ.get('TEST_LIBRARY_PATH', '/path/to/test/library'))
        if not cls.library_path.exists():
            raise unittest.SkipTest("Test library not available")

        cls.lib = PhotosLibrary(str(cls.library_path))

    @classmethod
    def tearDownClass(cls):
        """Close library."""
        cls.lib.close()

    def test_version_detection(self):
        """Test version detection."""
        self.assertIsNotNone(self.lib.version_info)
        self.assertIn('photos_version', self.lib.version_info)

    def test_get_photos(self):
        """Test getting photos."""
        photos = self.lib.get_all_photos()
        self.assertIsInstance(photos, list)
        if photos:
            self.assertIn('uuid', photos[0])
            self.assertIn('filename', photos[0])

    def test_get_albums(self):
        """Test getting albums."""
        albums = self.lib.get_albums()
        self.assertIsInstance(albums, list)

    def test_schema_detection(self):
        """Test schema detection."""
        self.assertIn(self.lib.asset_table, ['ZASSET', 'ZGENERICASSET'])
        self.assertIsNotNone(self.lib.album_join_table)

if __name__ == '__main__':
    unittest.main()
```

## Best Practices

### 1. Always Use Read-Only Access

```python
# Good
conn = sqlite3.connect(f"file:{db_path}?mode=ro", uri=True)
conn.execute("PRAGMA query_only = ON")

# Bad - could accidentally modify
conn = sqlite3.connect(db_path)
```

### 2. Handle Version Differences

```python
# Good - version adaptive
asset_table = detect_asset_table(conn)
query = f"SELECT * FROM {asset_table}"

# Bad - assumes version
query = "SELECT * FROM ZASSET"  # Will fail on Photos 5
```

### 3. Use Context Managers

```python
# Good
with PhotosLibrary(library_path) as lib:
    photos = lib.get_all_photos()
# Automatically closed

# Bad
lib = PhotosLibrary(library_path)
photos = lib.get_all_photos()
# Forgot to close
```

### 4. Validate Data

```python
# Good
latitude = row['ZLATITUDE']
if latitude != -180.0 and -90 <= latitude <= 90:
    use_location(latitude)

# Bad - assuming all values are valid
latitude = row['ZLATITUDE']
use_location(latitude)  # May be sentinel value
```

### 5. Handle Missing Files

```python
# Good
file_path = lib.get_file_path(uuid)
if file_path.exists():
    process_file(file_path)
else:
    # Try fallback
    fallback = find_derivative(uuid)

# Bad - assuming file exists
file_path = lib.get_file_path(uuid)
process_file(file_path)  # May crash
```

## Related Documents

- [[00-index]] - Overview
- [[02-database-schema]] - Database reference
- [[03-file-organization]] - File organization
- [[05-version-compatibility]] - Version handling
- [[06-organizational-structures]] - Albums and organization

---

**Complete**: You now have a comprehensive specification of the .photoslibrary format!
