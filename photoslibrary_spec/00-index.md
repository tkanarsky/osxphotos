# .photoslibrary File Format Specification

**Version**: 1.0
**Date**: January 2026
**Source**: Based on osxphotos implementation

## Overview

The `.photoslibrary` format is a package bundle directory structure used by Apple's Photos application on macOS to store photos, videos, metadata, and organizational information. This specification documents the complete format from Photos 2 (macOS 10.12) through Photos 11.1 (macOS 26.1/Tahoe).

## Purpose

This specification provides comprehensive technical documentation sufficient to implement a complete parser and reader for `.photoslibrary` files. It covers:

- Physical directory structure
- SQLite database schemas
- File organization conventions
- Metadata storage formats
- Version compatibility across macOS releases
- Special file types and their semantics

## Specification Structure

This specification is organized into the following documents:

1. **[[01-directory-structure]]** - Physical directory layout and file system organization
2. **[[02-database-schema]]** - SQLite database schema, tables, columns, and relationships
3. **[[03-file-organization]]** - How photo and video files are stored and organized
4. **[[04-metadata-formats]]** - Metadata storage including EXIF, edits, and binary formats
5. **[[05-version-compatibility]]** - Version differences across Photos releases
6. **[[06-organizational-structures]]** - Albums, folders, moments, and hierarchies
7. **[[07-special-formats]]** - Special file types (Live Photos, RAW, bursts, etc.)
8. **[[08-implementation-guide]]** - Implementation guidance with code examples
9. **[[09-known-gaps]]** - Known gaps and limitations in reverse-engineered knowledge

## Key Concepts

### Package Bundle Structure

The `.photoslibrary` is a macOS package bundle - a directory that appears as a single file in Finder but contains a structured hierarchy of files and subdirectories.

### Core Components

1. **Photos.sqlite** - Primary SQLite database containing all metadata
2. **originals/** - Original imported photo and video files
3. **resources/** - Derived files (thumbnails, previews, edits)
4. **database/search/psi.sqlite** - Search and ML labels database
5. **private/** - Application-specific private data

### Version Evolution

The format has evolved significantly across macOS versions:
- **Photos 2-4**: Legacy `photos.db` format
- **Photos 5**: Introduction of `Photos.sqlite`, ZGENERICASSET table
- **Photos 6+**: Renamed to ZASSET table, improved cloud support
- **Photos 7+**: Shared libraries, syndication support
- **Photos 10+**: New fingerprinting, adjustment state changes
- **Photos 11+**: Latest schema with Z_33ASSETS relationship table

## Target Audience

This specification is intended for:

- Developers implementing Photos library parsers
- Backup and migration tool developers
- Digital asset management system developers
- Forensic analysis tools
- AI assistants implementing library access code

## Conventions

### Data Types

- **Timestamp**: Core Data timestamp (seconds since January 1, 2001 00:00:00 UTC)
- **UUID**: Standard 128-bit UUID, typically uppercase with hyphens
- **FK**: Foreign key reference to another table's primary key
- **Binary Plist**: Apple binary property list format
- **UTI**: Uniform Type Identifier (e.g., `public.jpeg`, `public.heic`)

### Notation

- `Z_PK` - Primary key (Core Data convention)
- `Z[TABLE]` - Core Data table naming convention
- `Z_[X]ASSETS` - Version-specific join table (X varies by version)
- `-180.0` - Sentinel value indicating null for latitude/longitude

### File Paths

All paths in this specification are relative to the `.photoslibrary` bundle root unless otherwise specified.

## Getting Started

For a quick implementation:

1. Start with **[[05-version-compatibility]]** to detect the Photos version
2. Read **[[02-database-schema]]** to understand the core tables
3. Reference **[[03-file-organization]]** to locate photo files
4. Use **[[08-implementation-guide]]** for code examples

## Terminology

| Term | Definition |
|------|------------|
| **Asset** | A photo or video in the library |
| **Master** | Original file as imported |
| **Derivative** | Generated preview, thumbnail, or edited version |
| **Moment** | Automatically grouped collection of photos by time/location |
| **Album** | User-created collection of photos |
| **Folder** | Container for albums and other folders |
| **Burst** | Sequence of rapid-fire photos (burst mode) |
| **Live Photo** | Photo with associated short video |
| **RAW+JPEG** | Pair of RAW and JPEG versions of same photo |
| **Fingerprint** | Hash identifying unique file content |
| **Core Data** | Apple's object graph persistence framework |

## Important Notes

### Database Access

- The Photos.sqlite database should be accessed read-only
- Photos.app may have the database open; use `PRAGMA query_only = ON`
- Handle write-ahead log (WAL) files: `Photos.sqlite-wal`, `Photos.sqlite-shm`

### Version Detection

Always detect the Photos version before parsing:
1. Check `database/photos.db` (legacy) or `database/Photos.sqlite`
2. Read model version from `Z_METADATA.Z_PLIST` binary plist
3. Map model version to Photos version using ranges

### Cross-Version Compatibility

Table and column names vary by version. Critical differences:
- Asset table: `ZGENERICASSET` (Photos 5) vs `ZASSET` (Photos 6+)
- Join tables: `Z_26ASSETS` through `Z_33ASSETS` (version-dependent)
- Face foreign keys: `ZPERSON`/`ZASSET` vs `ZPERSONFORFACE`/`ZASSETFORFACE`

## Reference Implementation

This specification is derived from the osxphotos project:
- Repository: https://github.com/RhetTbull/osxphotos
- Primary implementation: `osxphotos/photosdb/photosdb.py`
- Version constants: `osxphotos/_constants.py`

## License and Credits

This specification documents the `.photoslibrary` format as implemented by Apple Photos and reverse-engineered by the osxphotos project. It is provided for educational and interoperability purposes.

---

**Next**: Start with [[01-directory-structure]] to understand the physical layout.
