# Metadata Formats

**Related**: [[00-index]] | [[02-database-schema]] | [[03-file-organization]]

## Overview

This document describes the various metadata formats used in `.photoslibrary`, including binary property lists, timestamps, location data, and adjustment information.

## Date and Time Formats

### Core Data Timestamps

All dates in the SQLite database use Core Data timestamp format:

**Definition**: Seconds since January 1, 2001 00:00:00 UTC (Core Data epoch)

```python
from datetime import datetime, timezone

# Core Data epoch
CORE_DATA_EPOCH = datetime(2001, 1, 1, tzinfo=timezone.utc)

def cd_timestamp_to_datetime(timestamp):
    """
    Convert Core Data timestamp to Python datetime.

    Args:
        timestamp: Float/int seconds since 2001-01-01

    Returns:
        datetime object in UTC
    """
    if timestamp is None:
        return None
    return datetime.fromtimestamp(
        timestamp + CORE_DATA_EPOCH.timestamp(),
        tz=timezone.utc
    )

def datetime_to_cd_timestamp(dt):
    """
    Convert Python datetime to Core Data timestamp.

    Args:
        dt: datetime object

    Returns:
        Float seconds since 2001-01-01
    """
    if dt is None:
        return None
    return dt.timestamp() - CORE_DATA_EPOCH.timestamp()
```

### Example Conversions

```python
# Database value: 694656000.0
# Represents: 2023-01-01 00:00:00 UTC

timestamp = 694656000.0
dt = cd_timestamp_to_datetime(timestamp)
# dt = datetime(2023, 1, 1, 0, 0, 0, tzinfo=timezone.utc)
```

### Timezone Information

Photos store timezone data separately from timestamps:

**ZADDITIONALASSETATTRIBUTES**:
- `ZTIMEZONEOFFSET`: Offset in seconds from UTC
- `ZINFERREDTIMEZONEOFFSET`: Inferred offset (when EXIF missing)
- `ZTIMEZONENAME`: Timezone name (e.g., "America/Los_Angeles")

```python
def apply_timezone(dt, timezone_offset, timezone_name=None):
    """
    Apply timezone to UTC datetime.

    Args:
        dt: datetime in UTC
        timezone_offset: Offset in seconds
        timezone_name: Timezone name (optional)

    Returns:
        datetime in local timezone
    """
    from datetime import timedelta
    import pytz

    if timezone_name:
        try:
            tz = pytz.timezone(timezone_name)
            return dt.astimezone(tz)
        except:
            pass

    # Fallback to offset
    if timezone_offset is not None:
        offset = timedelta(seconds=timezone_offset)
        return dt + offset

    return dt
```

## Location Data

### Coordinate Storage

GPS coordinates stored in asset table:

- `ZASSET.ZLATITUDE`: Latitude (-90.0 to 90.0)
- `ZASSET.ZLONGITUDE`: Longitude (-180.0 to 180.0)

**Null Sentinel**: `-180.0` indicates no location data

```python
def parse_coordinates(latitude, longitude):
    """
    Parse coordinates, handling null sentinels.

    Args:
        latitude: Database latitude value
        longitude: Database longitude value

    Returns:
        Tuple of (lat, lon) or (None, None) if invalid
    """
    if latitude == -180.0 or longitude == -180.0:
        return (None, None)
    if latitude is None or longitude is None:
        return (None, None)

    # Validate ranges
    if not (-90 <= latitude <= 90):
        return (None, None)
    if not (-180 <= longitude <= 180):
        return (None, None)

    return (latitude, longitude)
```

### Reverse Geocoding Data

Reverse geocoded place names stored in binary plist:

**Location**: `ZADDITIONALASSETATTRIBUTES.ZREVERSELOCATIONDATA`
**Format**: Binary property list (bplist)

#### Plist Structure

```python
{
    'placeNames': [
        'Country',           # Index 0
        'State/Province',    # Index 1
        'City',             # Index 2
        'Sub-locality',     # Index 3
        'Street Address',   # Index 4
        'Area of Interest'  # Index 5
    ]
}
```

#### Parsing Example

```python
import plistlib

def parse_reverse_location(blob_data):
    """
    Parse reverse location binary plist.

    Args:
        blob_data: ZREVERSELOCATIONDATA blob (bytes)

    Returns:
        Dict with location components
    """
    if not blob_data:
        return {}

    try:
        plist = plistlib.loads(blob_data)
        place_names = plist.get('placeNames', [])

        return {
            'country': place_names[0] if len(place_names) > 0 else None,
            'state_province': place_names[1] if len(place_names) > 1 else None,
            'city': place_names[2] if len(place_names) > 2 else None,
            'sub_locality': place_names[3] if len(place_names) > 3 else None,
            'street': place_names[4] if len(place_names) > 4 else None,
            'area_of_interest': place_names[5] if len(place_names) > 5 else None,
        }
    except Exception as e:
        return {}
```

#### Example Data

```
Country: "United States"
State/Province: "California"
City: "San Francisco"
Sub-locality: "Mission District"
Street: "Valencia Street"
Area of Interest: "Dolores Park"
```

## Adjustment/Edit Data

### Edit Metadata Location

Edit information stored in:
1. **File**: `resources/renders/{first_char}/{UUID}.plist`
2. **Database**: `ZUNMANAGEDADJUSTMENT` table (reference only)

### Adjustment Plist Format

Binary property list with structure:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN">
<plist version="1.0">
<dict>
    <key>adjustmentBaseVersion</key>
    <integer>0</integer>

    <key>adjustmentFormatIdentifier</key>
    <string>com.apple.photos</string>

    <key>adjustmentFormatVersion</key>
    <string>1.4</string>

    <key>adjustmentTimestamp</key>
    <date>2023-06-15T14:30:00Z</date>

    <key>adjustmentData</key>
    <data>
    [Base64 encoded binary data or zlib-compressed plist]
    </data>

    <key>editorBundleID</key>
    <string>com.apple.Photos</string>
</dict>
</plist>
```

### Key Fields

| Field | Type | Description |
|-------|------|-------------|
| `adjustmentBaseVersion` | Integer | Base version number |
| `adjustmentFormatIdentifier` | String | Format identifier (e.g., `com.apple.photos`) |
| `adjustmentFormatVersion` | String | Version of adjustment format |
| `adjustmentTimestamp` | Date | When edit was made |
| `adjustmentData` | Data | Actual edit data (may be compressed) |
| `editorBundleID` | String | Bundle ID of editing application |

### Adjustment Data Formats

The `adjustmentData` can be in several formats:

#### Format 1: Nested Plist

```python
{
    'formatVersion': '1.4',
    'versionInfo': {
        'appVersion': '8.0',
        'buildNumber': '3511.8.200',
        'platform': 'macOS',
        'schemaRevision': 1
    },
    'metadata': {
        'orientation': 1,
        'masterWidth': 4032,
        'masterHeight': 3024
    },
    'adjustments': [
        {
            'identifier': 'Crop',
            'settings': {
                'inputCropRect': [0.1, 0.1, 0.8, 0.8],
                'constraintWidth': 1.0,
                'constraintHeight': 1.0
            },
            'enabled': True
        },
        {
            'identifier': 'Exposure',
            'settings': {
                'inputEV': 0.5
            },
            'enabled': True
        },
        {
            'identifier': 'SmartTone',
            'settings': {
                'statistics': { ... },
                'auto': True
            },
            'enabled': True
        }
    ]
}
```

#### Format 2: Zlib Compressed

Data may be zlib-compressed:

```python
import zlib
import plistlib

def parse_adjustment_data(data_bytes):
    """
    Parse adjustment data, handling compression.

    Args:
        data_bytes: adjustmentData bytes from plist

    Returns:
        Parsed adjustment dict or None
    """
    if not data_bytes:
        return None

    # Try parsing as plist directly
    try:
        return plistlib.loads(data_bytes)
    except:
        pass

    # Try decompressing first
    try:
        decompressed = zlib.decompress(data_bytes)
        return plistlib.loads(decompressed)
    except:
        pass

    return None
```

### Common Adjustment Types

| Identifier | Description | Settings |
|------------|-------------|----------|
| `Crop` | Crop adjustment | `inputCropRect`, `constraintWidth/Height` |
| `Exposure` | Exposure adjustment | `inputEV` |
| `SmartTone` | Auto tone | `statistics`, `auto` |
| `SmartColor` | Auto color | `statistics`, `auto` |
| `Highlights` | Highlight adjustment | `inputAmount` |
| `Shadows` | Shadow adjustment | `inputAmount` |
| `Contrast` | Contrast adjustment | `inputAmount` |
| `Brightness` | Brightness adjustment | `inputAmount` |
| `Saturation` | Saturation adjustment | `inputAmount` |
| `Vibrance` | Vibrance adjustment | `inputAmount` |
| `Temperature` | White balance temp | `inputAmount` |
| `Tint` | White balance tint | `inputAmount` |
| `Sharpen` | Sharpening | `inputAmount` |
| `Definition` | Definition adjustment | `inputAmount` |
| `NoiseReduction` | Noise reduction | `inputAmount` |
| `Effect` | Filter effect | `filterIdentifier` |
| `RedEye` | Red-eye removal | `corrections` array |
| `WhiteBalance` | White balance | `temperature`, `tint` |

### Adjustment Settings Examples

#### Crop

```python
{
    'identifier': 'Crop',
    'settings': {
        'inputCropRect': [0.05, 0.1, 0.9, 0.85],  # [x, y, width, height] normalized
        'constraintWidth': 1.0,
        'constraintHeight': 0.75,  # 4:3 aspect ratio
        'inputAngle': 0.0          # Rotation in radians
    },
    'enabled': True
}
```

#### Exposure and Color

```python
{
    'identifier': 'Exposure',
    'settings': {
        'inputEV': 0.5  # +0.5 EV stops
    },
    'enabled': True
}

{
    'identifier': 'Saturation',
    'settings': {
        'inputAmount': 0.2  # +20% saturation
    },
    'enabled': True
}
```

#### Filter Effects

```python
{
    'identifier': 'Effect',
    'settings': {
        'filterIdentifier': 'CIPhotoEffectMono',  # B&W filter
        'intensity': 1.0
    },
    'enabled': True
}
```

Common filter identifiers:
- `CIPhotoEffectMono`: Black & White
- `CIPhotoEffectTonal`: Tonal
- `CIPhotoEffectNoir`: Noir
- `CIPhotoEffectFade`: Fade
- `CIPhotoEffectChrome`: Chrome
- `CIPhotoEffectProcess`: Process
- `CIPhotoEffectTransfer`: Transfer
- `CIPhotoEffectInstant`: Instant
- `CIPhotoEffectVivid`: Vivid
- `CIPhotoEffectVividWarm`: Vivid Warm
- `CIPhotoEffectVividCool`: Vivid Cool
- `CIPhotoEffectDramatic`: Dramatic
- `CIPhotoEffectDramaticWarm`: Dramatic Warm
- `CIPhotoEffectDramaticCool`: Dramatic Cool

### Parsing Complete Edit Info

```python
def parse_edit_plist(plist_path):
    """
    Parse complete edit plist file.

    Args:
        plist_path: Path to UUID.plist file

    Returns:
        Dict with edit information
    """
    with open(plist_path, 'rb') as f:
        plist = plistlib.load(f)

    adjustment_data = parse_adjustment_data(
        plist.get('adjustmentData', b'')
    )

    return {
        'format_identifier': plist.get('adjustmentFormatIdentifier'),
        'format_version': plist.get('adjustmentFormatVersion'),
        'timestamp': plist.get('adjustmentTimestamp'),
        'editor_bundle_id': plist.get('editorBundleID'),
        'adjustments': adjustment_data.get('adjustments', []) if adjustment_data else []
    }
```

## Binary Plist Utilities

### Reading Binary Plists

```python
import plistlib

def read_bplist(blob_or_path):
    """
    Read binary plist from blob or file path.

    Args:
        blob_or_path: bytes blob or file path string

    Returns:
        Parsed plist dict
    """
    if isinstance(blob_or_path, (bytes, bytearray)):
        return plistlib.loads(blob_or_path)
    else:
        with open(blob_or_path, 'rb') as f:
            return plistlib.load(f)
```

### Detecting Plist Format

```python
def detect_plist_format(data):
    """
    Detect plist format.

    Args:
        data: bytes data

    Returns:
        'binary', 'xml', or 'unknown'
    """
    if data.startswith(b'bplist'):
        return 'binary'
    elif data.startswith(b'<?xml') or data.startswith(b'<plist'):
        return 'xml'
    else:
        return 'unknown'
```

## UUID Formats

### Standard UUID

UUIDs in database are standard format:
- Format: `XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX`
- Case: Usually uppercase
- Example: `2E3F8B9A-1234-5678-90AB-CDEF12345678`

### UUID Validation

```python
import re
import uuid

def validate_uuid(uuid_string):
    """
    Validate UUID format.

    Args:
        uuid_string: UUID string

    Returns:
        True if valid, False otherwise
    """
    try:
        uuid.UUID(uuid_string)
        return True
    except:
        return False

# Alternative: regex validation
UUID_PATTERN = re.compile(
    r'^[0-9A-Fa-f]{8}-[0-9A-Fa-f]{4}-[0-9A-Fa-f]{4}-[0-9A-Fa-f]{4}-[0-9A-Fa-f]{12}$'
)

def validate_uuid_regex(uuid_string):
    return bool(UUID_PATTERN.match(uuid_string))
```

## Fingerprints and Hashes

### Master Fingerprint (Photos 5-9)

**Field**: `ZADDITIONALASSETATTRIBUTES.ZMASTERFINGERPRINT`
**Format**: Base64-encoded hash string

```python
import base64

def decode_fingerprint(fingerprint_string):
    """
    Decode master fingerprint.

    Args:
        fingerprint_string: Base64 fingerprint

    Returns:
        Decoded bytes
    """
    if not fingerprint_string:
        return None
    try:
        return base64.b64decode(fingerprint_string)
    except:
        return None
```

### Original Stable Hash (Photos 10+)

**Field**: `ZADDITIONALASSETATTRIBUTES.ZORIGINALSTABLEHASH`
**Format**: Integer hash value

Purpose: Stable identifier for duplicate detection across library updates.

### Resource Fingerprint

**Field**: `ZINTERNALRESOURCE.ZFINGERPRINT`
**Format**: String hash

Identifies specific resource file version.

## UTI (Uniform Type Identifier)

### Common UTIs

```python
UTI_MAP = {
    # Images
    'public.jpeg': {'ext': 'jpg', 'mime': 'image/jpeg'},
    'public.heic': {'ext': 'heic', 'mime': 'image/heic'},
    'public.heif': {'ext': 'heif', 'mime': 'image/heif'},
    'public.png': {'ext': 'png', 'mime': 'image/png'},
    'public.tiff': {'ext': 'tiff', 'mime': 'image/tiff'},
    'com.compuserve.gif': {'ext': 'gif', 'mime': 'image/gif'},

    # RAW Formats
    'com.canon.cr2-raw-image': {'ext': 'cr2', 'mime': 'image/x-canon-cr2'},
    'com.canon.cr3-raw-image': {'ext': 'cr3', 'mime': 'image/x-canon-cr3'},
    'com.nikon.raw-image': {'ext': 'nef', 'mime': 'image/x-nikon-nef'},
    'com.nikon.nrw-raw-image': {'ext': 'nrw', 'mime': 'image/x-nikon-nrw'},
    'com.sony.raw-image': {'ext': 'arw', 'mime': 'image/x-sony-arw'},
    'com.sony.sr2-raw-image': {'ext': 'sr2', 'mime': 'image/x-sony-sr2'},
    'com.fuji.raw-image': {'ext': 'raf', 'mime': 'image/x-fuji-raf'},
    'com.olympus.raw-image': {'ext': 'orf', 'mime': 'image/x-olympus-orf'},
    'com.panasonic.raw-image': {'ext': 'rw2', 'mime': 'image/x-panasonic-rw2'},
    'com.pentax.raw-image': {'ext': 'dng', 'mime': 'image/x-adobe-dng'},
    'com.adobe.raw-image': {'ext': 'dng', 'mime': 'image/x-adobe-dng'},

    # Video
    'com.apple.quicktime-movie': {'ext': 'mov', 'mime': 'video/quicktime'},
    'public.mpeg-4': {'ext': 'mp4', 'mime': 'video/mp4'},
    'public.avi': {'ext': 'avi', 'mime': 'video/x-msvideo'},
}
```

### UTI Utilities

```python
def uti_to_extension(uti):
    """
    Convert UTI to file extension.

    Args:
        uti: Uniform Type Identifier string

    Returns:
        File extension (without dot) or None
    """
    return UTI_MAP.get(uti, {}).get('ext')

def uti_to_mime(uti):
    """
    Convert UTI to MIME type.

    Args:
        uti: Uniform Type Identifier string

    Returns:
        MIME type string or None
    """
    return UTI_MAP.get(uti, {}).get('mime')
```

## Orientation Values

### EXIF Orientation

Stored in `ZASSET.ZORIENTATION` and `ZADDITIONALASSETATTRIBUTES.ZORIGINALORIENTATION`:

| Value | Description |
|-------|-------------|
| 1 | Normal (0° rotation) |
| 2 | Flipped horizontally |
| 3 | Rotated 180° |
| 4 | Flipped vertically |
| 5 | Rotated 90° CCW and flipped horizontally |
| 6 | Rotated 90° CW |
| 7 | Rotated 90° CW and flipped horizontally |
| 8 | Rotated 90° CCW (270° CW) |

### Applying Orientation

```python
from PIL import Image

ORIENTATION_TRANSFORMS = {
    1: lambda img: img,
    2: lambda img: img.transpose(Image.FLIP_LEFT_RIGHT),
    3: lambda img: img.rotate(180),
    4: lambda img: img.transpose(Image.FLIP_TOP_BOTTOM),
    5: lambda img: img.transpose(Image.FLIP_LEFT_RIGHT).rotate(90, expand=True),
    6: lambda img: img.rotate(270, expand=True),
    7: lambda img: img.transpose(Image.FLIP_LEFT_RIGHT).rotate(270, expand=True),
    8: lambda img: img.rotate(90, expand=True),
}

def apply_orientation(image, orientation):
    """
    Apply EXIF orientation to image.

    Args:
        image: PIL Image object
        orientation: EXIF orientation value (1-8)

    Returns:
        Transformed PIL Image
    """
    transform = ORIENTATION_TRANSFORMS.get(orientation)
    if transform:
        return transform(image)
    return image
```

## Database Metadata

### Z_METADATA Table

Extracting model version:

```python
def get_model_version(conn):
    """
    Get Photos library model version.

    Args:
        conn: SQLite database connection

    Returns:
        Integer model version
    """
    cursor = conn.cursor()
    cursor.execute("SELECT Z_PLIST FROM Z_METADATA WHERE Z_VERSION = 1")
    row = cursor.fetchone()

    if not row:
        return None

    plist = plistlib.loads(row[0])
    return plist.get('PLModelVersion')
```

## Related Documents

- [[02-database-schema]] - Database structure
- [[03-file-organization]] - File organization
- [[05-version-compatibility]] - Version differences
- [[08-implementation-guide]] - Implementation examples

---

**Next**: [[05-version-compatibility]] - Version compatibility information
