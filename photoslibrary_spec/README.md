# .photoslibrary File Format Specification

Complete technical specification of Apple's Photos library format on macOS.

## What is This?

This specification documents the `.photoslibrary` bundle format used by Apple Photos on macOS. It provides comprehensive technical details sufficient to implement a complete library reader from scratch.

## Specification Documents

Start with the **[[00-index]]** for an overview, then follow these documents:

### Core Documentation

1. **[[00-index.md]]** - Overview and introduction
   - Purpose and conventions
   - Getting started guide
   - Key concepts and terminology

2. **[[01-directory-structure.md]]** - Physical file system layout
   - Root directory structure
   - Database files
   - Originals and resources organization
   - Private and scopes directories

3. **[[02-database-schema.md]]** - SQLite database schema
   - Complete table definitions
   - Column descriptions and data types
   - Foreign key relationships
   - Search database (psi.sqlite)

4. **[[03-file-organization.md]]** - File storage and access
   - UUID-based file naming
   - Locating original files
   - Derivatives and previews
   - Referenced and cloud files

5. **[[04-metadata-formats.md]]** - Metadata storage formats
   - Date/time handling (Core Data timestamps)
   - Location data and reverse geocoding
   - Adjustment/edit plists
   - Binary plist utilities

### Advanced Topics

6. **[[05-version-compatibility.md]]** - Version differences
   - Version detection algorithm
   - Schema changes across Photos versions
   - Version-adaptive queries
   - Migration notes

7. **[[06-organizational-structures.md]]** - Albums, folders, and organization
   - User albums and folders
   - Shared albums
   - Keywords and tags
   - People and faces
   - Moments
   - Search and ML labels

8. **[[07-special-formats.md]]** - Special file types
   - Live Photos
   - Burst photos
   - RAW+JPEG pairs
   - Panoramas, screenshots, time-lapses
   - HDR and Portrait photos
   - Edited photos

### Implementation

9. **[[08-implementation-guide.md]]** - Complete implementation guide
   - Full PhotosLibrary class example
   - Usage examples
   - Performance optimization
   - Error handling
   - Testing
   - Best practices

## Quick Start

### For Readers

If you want to understand the format:
1. Read [[00-index.md]]
2. Skim [[01-directory-structure.md]] to understand the layout
3. Read [[02-database-schema.md]] for the data model

### For Implementers

If you want to build a library reader:
1. Read [[05-version-compatibility.md]] for version detection
2. Study [[02-database-schema.md]] for database queries
3. Reference [[08-implementation-guide.md]] for complete examples
4. Use other documents as needed for specific features

## Coverage

This specification covers:

- **Photos versions**: 2 through 11.1 (Photos 2-4 legacy, Photos 5-11 modern)
- **macOS versions**: 10.12 Sierra through macOS 26.1 Tahoe
- **Database format**: SQLite with Core Data schema
- **File formats**: JPEG, HEIC, RAW, MOV, MP4, and more
- **Special types**: Live Photos, bursts, panoramas, HDR, portraits
- **Organization**: Albums, folders, keywords, people, moments
- **Metadata**: EXIF, location, edits, ML labels

## What You Can Build

With this specification, you can implement:

- **Library readers**: Read photo metadata and files
- **Export tools**: Export photos with metadata
- **Analysis tools**: Analyze library contents
- **Backup tools**: Create backups with metadata
- **Migration tools**: Move photos between libraries
- **Search tools**: Search photos by various criteria
- **Reporting tools**: Generate library statistics

## What You Cannot Do

This specification does **not** cover:

- **Writing/modifying libraries**: Read-only access only
- **Creating new libraries**: Library creation is undocumented
- **iCloud sync protocol**: Cloud sync is not documented
- **Photos.app UI**: Application interface details
- **Image processing**: Edit algorithms not included

## Example Use Cases

### 1. Export Photos with Metadata

```python
from photoslibrary_spec import PhotosLibrary

with PhotosLibrary('/path/to/Library.photoslibrary') as lib:
    for photo in lib.get_all_photos():
        # Get file
        file_path = lib.get_file_path(photo['uuid'])

        # Get metadata
        keywords = lib.get_keywords(photo['uuid'])
        people = lib.get_faces_in_photo(photo['uuid'])

        # Export with metadata...
```

### 2. Find Photos by Person

```python
with PhotosLibrary('/path/to/Library.photoslibrary') as lib:
    # Find all photos with "John"
    for photo in lib.get_all_photos():
        faces = lib.get_faces_in_photo(photo['uuid'])
        if any('John' in face['person_name'] for face in faces):
            print(f"Found: {photo['filename']}")
```

### 3. Generate Library Report

```python
with PhotosLibrary('/path/to/Library.photoslibrary') as lib:
    photos = lib.get_all_photos()
    albums = lib.get_albums()
    people = lib.get_people()

    print(f"Total photos: {len(photos)}")
    print(f"Total albums: {len(albums)}")
    print(f"Named people: {len(people)}")
```

## Technical Details

### Database Access

- **Format**: SQLite 3 with WAL mode
- **Access**: Read-only recommended
- **Safety**: Use `PRAGMA query_only = ON`
- **Versions**: Schema varies by Photos version

### File Organization

- **Structure**: UUID-based with first-character subdivision
- **Originals**: `originals/{first_char}/{UUID}.{ext}`
- **Derivatives**: `resources/derivatives/{first_char}/{UUID}_{quality}_{size}_{type}.jpeg`
- **Edits**: `resources/renders/{first_char}/{UUID}.plist`

### Date/Time Format

All dates use Core Data timestamp:
- **Epoch**: January 1, 2001 00:00:00 UTC
- **Format**: Float seconds since epoch
- **Conversion**: Add Core Data epoch to Unix timestamp

### Location Format

- **Coordinates**: Float latitude/longitude
- **Null sentinel**: -180.0 indicates no location
- **Reverse geocoding**: Binary plist in ZADDITIONALASSETATTRIBUTES

## Version Support

| Photos Version | macOS | Model Version Range | Status |
|----------------|-------|---------------------|--------|
| Photos 5 | 10.15 Catalina | 13000-13999 | ✅ Documented |
| Photos 6 | 11 Big Sur | 14000-14999 | ✅ Documented |
| Photos 7 | 12 Monterey | 15000-15999 | ✅ Documented |
| Photos 8 | 13 Ventura | 16000-16999 | ✅ Documented |
| Photos 9 | 14 Sonoma | 17000-17599 | ✅ Documented |
| Photos 9.6 | 14.6+ Sonoma | 17600-17999 | ✅ Documented |
| Photos 10 | 15 Sequoia | 18201-18999 | ✅ Documented |
| Photos 11 | 16/26 Tahoe | 19063-19319 | ✅ Documented |
| Photos 11.1 | 26.1 Tahoe | 19320-19999 | ✅ Documented |

## Source and Credits

This specification is derived from the [osxphotos](https://github.com/RhetTbull/osxphotos) project, which provides a comprehensive Python implementation for accessing Photos libraries.

### Key Source Files

- `osxphotos/photosdb/photosdb.py` - Main database handler
- `osxphotos/_constants.py` - Version mappings
- `osxphotos/photoinfo.py` - Photo information
- `osxphotos/albuminfo.py` - Album information
- `osxphotos/personinfo.py` - Person/face information

## Document Format

### Obsidian Compatibility

These documents use Obsidian-compatible markdown:
- **Internal links**: `[[document-name]]` syntax
- **Cross-references**: Links between related topics
- **Code blocks**: Syntax-highlighted examples
- **Tables**: Formatted tables for reference data

### Navigation

- Each document links to related documents
- Index provides overview and navigation
- Implementation guide ties everything together

## Contributing

To improve this specification:
1. Identify gaps or inaccuracies
2. Test against real Photos libraries
3. Document findings with examples
4. Submit updates with version information

## License

This specification documents Apple's proprietary format for educational and interoperability purposes. Implementation is at your own risk.

## Disclaimer

This is a reverse-engineered specification. Apple may change the format without notice. Always:
- Access libraries read-only
- Test thoroughly before deployment
- Handle version differences
- Backup before any operations
- Respect user privacy

---

## Document Index

### Quick Reference

- **Getting Started**: [[00-index.md]]
- **Directory Layout**: [[01-directory-structure.md]]
- **Database Tables**: [[02-database-schema.md]]
- **File Locations**: [[03-file-organization.md]]
- **Metadata Formats**: [[04-metadata-formats.md]]
- **Version Detection**: [[05-version-compatibility.md]]
- **Albums & People**: [[06-organizational-structures.md]]
- **Special Types**: [[07-special-formats.md]]
- **Implementation**: [[08-implementation-guide.md]]

### By Topic

**Database**:
- Schema: [[02-database-schema.md]]
- Queries: [[08-implementation-guide.md]]
- Versions: [[05-version-compatibility.md]]

**Files**:
- Organization: [[03-file-organization.md]]
- Directory structure: [[01-directory-structure.md]]
- Special formats: [[07-special-formats.md]]

**Features**:
- Albums: [[06-organizational-structures.md]]
- People/Faces: [[06-organizational-structures.md]]
- Live Photos: [[07-special-formats.md]]
- Edits: [[04-metadata-formats.md]]

**Implementation**:
- Complete guide: [[08-implementation-guide.md]]
- Version handling: [[05-version-compatibility.md]]
- Examples: [[08-implementation-guide.md]]

---

**Ready to start?** Open [[00-index.md]] to begin exploring the specification.
