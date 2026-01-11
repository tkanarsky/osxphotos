# Known Gaps and Limitations

**Related**: [[00-index]] | [[02-database-schema]] | [[05-version-compatibility]]

## Overview

This document catalogs known gaps, uncertainties, and incomplete areas in the reverse-engineered .photoslibrary format specification. These represent areas where the format is not fully understood or where proprietary algorithms prevent complete documentation.

## Status Legend

- ✅ **Fully Documented**: Complete understanding and implementation possible
- ⚠️ **Partially Documented**: Basic functionality understood, some details missing
- ❌ **Incomplete**: Significant gaps in understanding
- 🔒 **Proprietary**: Relies on private APIs or algorithms that cannot be reverse-engineered

## Major Gaps

### 1. Shared iCloud Library (Photos 8+) 🔒❌

**Status**: Experimental/Guessed Implementation

**Issue**: The distinction between shared moments (Messages sharing) and shared iCloud libraries is unclear.

```python
# From osxphotos codebase - marked as TODO
# "this is a total guess right now. I think that ZSHARE holds information
# about both shared moments and shared iCloud Library
# The foreign key for shared moments appears to be ZASSET.ZMOMENTSHARE
# but I don't know the key for shared iCloud Libraries
# I'm guessing it's ZASSET.ZSHARESCOPE but I don't know for sure"
```

**Unknown**:
- Correct foreign key relationship for shared libraries vs. shared moments
- Participant management in shared libraries
- Sync state transitions
- Sharing permissions and rules
- How Photos distinguishes between share types

**Database Tables Affected**:
- `ZSHARE` - Purpose of many columns unclear
- `ZASSET.ZSHARESCOPE` - Relationship uncertain
- `ZASSET.ZMOMENTSHARE` - Relationship to moments vs. libraries

**Workaround**: Limited support for shared library detection, may be inaccurate.

### 2. Syndication / "Shared with You" (Photos 7+) ⚠️

**Status**: Minimally Implemented

**Available**:
- `ZASSET.ZSYNDICATIONSTATE` - Basic state field
- Detection via model version >= 15323 (Photos 7)

**Unknown**:
- Complete state enumeration for `ZSYNDICATIONSTATE`
- `ZSYNDICATIONHISTORY` blob format
- Syndication identifier meaning and generation
- How syndicated items are downloaded/cached
- Expiration and cleanup of syndicated content

**File System**:
- `scopes/syndication/` directory structure not documented
- Syndication metadata format unknown

**Impact**: Can detect if photo is syndicated but cannot fully understand syndication lifecycle.

### 3. Face Recognition and Clustering 🔒

**Status**: Face Detection Documented, Clustering Proprietary

**Fully Documented**:
- Face bounding boxes (`ZCENTERX`, `ZCENTERY`, `ZSIZE`)
- Face-to-person associations
- Person naming

**Unknown/Proprietary**:
- Face clustering algorithm (how faces are grouped into people)
- Face recognition training data and models
- Quality metrics meaning:
  - `ZQUALITYMEASURE` - Values and thresholds unclear
  - `ZBLURSCORE` - Blur detection algorithm
- Face feature detection (removed in later versions):
  - Eye coordinates (removed after Photos 4)
  - Mouth coordinates (removed after Photos 4)
  - Eye closed detection (never fully implemented)

**Version-Specific Changes**:

| Feature | Photos 4 | Photos 5-8 | Photos 9+ | Purpose |
|---------|----------|-----------|----------|---------|
| Eye/Mouth coords | ✅ | ❌ | ❌ | Unknown why removed |
| Bald detection | ❌ | ✅ | ❌? | Unknown if still present |
| `ZMASTERIDENTIFIER` | ❌ | ✅ | ✅ | Purpose unclear |

**Database Gaps**:
```sql
-- Purpose of these columns not documented
ZDETECTEDFACE.ZMASTERIDENTIFIER     -- What does this identify?
ZDETECTEDFACE.ZBLURSCORE            -- Threshold values?
ZDETECTEDFACE.ZQUALITYMEASURE       -- How is quality calculated?
ZPERSON.ZVERIFIEDTYPE               -- Verification meaning?
```

### 4. Spotlight Search Indices 🔒

**Status**: SQLite Database Documented, Binary Indices Not

**Documented**:
- `database/search/psi.sqlite` structure
- Asset-to-label associations
- Search categories

**Not Documented**:
```
database/search/Spotlight/NSFileProtectionCompleteUntilFirstUserAuthentication/
└── index.spotlightV3/
    ├── 0.indexScores           # Binary format unknown
    ├── dbStr-3.map.buckets     # Binary format unknown
    ├── dbStr-3.map.offsets     # Binary format unknown
    ├── live.0.indexBigDates    # Binary format unknown
    ├── live.0.indexIds         # Binary format unknown
    ├── live.0.indexPositionTable  # Binary format unknown
    ├── live.0.indexScores      # Binary format unknown
    └── live.0.indexTermIds     # Binary format unknown
```

**Impact**: Can read ML labels from psi.sqlite but cannot understand or rebuild Spotlight indices.

**Unknown**:
- Binary index format
- Index serialization
- Index update mechanism
- How indices relate to psi.sqlite

### 5. Photo Quality and Aesthetic Scores 🔒

**Status**: Fields Documented, Algorithms Proprietary

**Available Scores** (27 total):
```sql
SELECT
    ZOVERALLAESTHETICSCORE,      -- Overall aesthetic quality
    ZCURATIONSSCORE,              -- Curation ranking
    ZPROMOTIONSCORE,              -- Promotion likelihood
    ZHIGHLIGHTVISIBILITYSCORE,    -- Highlight visibility
    ZAUTOPLAYSUGGESTIONSCORE,     -- Autoplay suggestion
    ZWELLFRAMEDSUBJECTSCORE,      -- Subject framing quality
    ZWELLTIMEDSHOTSCORE,          -- Shot timing quality
    ZPLEASANTLIGHTINGSCORE,       -- Lighting quality
    ZPLEASANTREFLECTIONSSCORE,    -- Reflection quality
    ZHARMONIOUSCOLORSCORE,        -- Color harmony
    ZLIVELYCOLORSCORE,            -- Color vibrancy
    ZPLEASANTPERSPECTIVESCORE,    -- Perspective quality
    ZPLEASANTPATTERNSCORE,        -- Pattern quality
    ZIMMERSIVENESSCORE,           -- Immersiveness
    ZPLEASANTCOMPOSITIONSCORE,    -- Composition quality
    ZINTERESTINGSUBJECTSCORE,     -- Subject interest
    ZPLEASANTSYMMETRYSCORE,       -- Symmetry quality
    ZPLEASANTPOSTPROCESSINGSCORE, -- Post-processing quality
    ZSHARPLYFOCUSEDSUBJECTSCORE,  -- Focus quality
    ZTASTEFULLYBLURREDSCORE,      -- Bokeh quality
    ZNOISYSCORE,                  -- Noise level
    ZFAILURESCORE,                -- Failure detection
    ZPLEASANTCAMERATILTSCORE,     -- Camera tilt quality
    ZLOWLIGHT,                    -- Low light detection
    -- Photos 5-10 only:
    ZBEHAVIORALSCORE,             -- User interaction (removed Photos 11)
    ZINTERACTIONSCORE,            -- Interaction level (removed Photos 11)
    -- Plus ~2 more version-specific scores
FROM ZCOMPUTEDASSETATTRIBUTES;
```

**Unknown**:
- Calculation algorithms for all scores
- Score ranges and meanings
- Thresholds for quality levels
- Training data for ML models
- Why behavioral/interaction scores removed in Photos 11

**Impact**: Can read scores but cannot replicate scoring or understand exact meaning.

### 6. Burst Photo Selection Algorithm ⚠️

**Status**: Selection Flags Documented, Algorithm Unknown

**Known**:
```python
ZAVALANCHEPICKTYPE values:
    0:  not_burst
    2:  not_selected
    4:  default_pick
    8:  user_selected
    16: key_photo
    32: unknown (marked as BURST_UNKNOWN in codebase)
```

**Unknown**:
- What does value 32 represent?
- How does Photos automatically select the "best" photo in burst?
- What criteria determine the key photo?
- Quality metrics used for selection

**From osxphotos code**:
```python
BURST_UNKNOWN = 0b100000  # 32
# "this is almost always set with BURST_DEFAULT_PICK and never if
# BURST_DEFAULT_PICK is not set. I think this has something to do
# with what algorithm Photos used to pick the default image"
```

**Impact**: Can identify burst selections but cannot replicate selection algorithm.

### 7. iCloud Sync States and Transitions ⚠️

**Status**: Field Values Known, State Machine Unknown

**Available Fields**:
```sql
ZASSET.ZCLOUDLOCALSTATE          -- Local sync state
ZASSET.ZACTIVE                   -- Active in cloud?
ZASSET.ZCLOUDISDUPLICATE         -- Duplicate detection
ZASSET.ZCLOUDBATCHPUBLISHDATE    -- Publish date for sharing
ZCLOUDMASTER.ZCLOUDLOCALSTATE    -- Master sync state
ZSHARE.ZCLOUDLOCALSTATE          -- Share sync state
```

**Unknown**:
- Valid state values and meanings
- State transition diagram
- Error states and recovery
- How conflicts are resolved
- Duplicate detection algorithm
- When and why states change

**Impact**: Can read current state but cannot predict behavior or understand sync process.

### 8. Moments and Automatic Grouping ❌

**Status**: Database Schema Known, Algorithm Proprietary

**Available**:
```sql
ZMOMENT table:
    ZUUID, ZTITLE, ZSUBTITLE
    ZSTARTDATE, ZENDDATE
    ZAPPROXIMATELATITUDE, ZAPPROXIMATELONGITUDE
    ZCACHEDCOUNT
```

**Unknown**:
- How Photos automatically groups photos into moments
- Time window calculation
- Location clustering algorithm
- Title and subtitle generation
- Moment merging and splitting rules

**From osxphotos**:
```python
# moment_info not implemented for PhotosDB (#1496)
# Moments counted by iterating photos - no proper API
```

**Impact**: Can read existing moments but cannot create or understand grouping logic.

### 9. Edit/Adjustment Rendering 🔒

**Status**: Adjustment Data Documented, Rendering Unknown

**Known**:
- Adjustment plist format (see [[04-metadata-formats]])
- Adjustment types and parameters
- Edit history

**Unknown**:
- How Photos renders adjustments to generate previews
- Image processing pipeline
- Filter implementation details
- `SmartTone` and `SmartColor` auto-adjustment algorithms
- Rendered derivative generation algorithm

**Files**:
```
resources/renders/{first_char}/
├── {UUID}.plist              # Documented ✅
└── {UUID}_1_201_a.jpeg      # Generation algorithm unknown 🔒
```

**Impact**: Can read what edits were made but cannot recreate edited images without Photos.

### 10. RAW + JPEG Pairing Logic ⚠️

**Status**: Basic Pairing Documented, Details Missing

**Known**:
- RAW files have `ZDATASTORESUBTYPE = 17`
- `ZORIGINALRESOURCECHOICE` indicates which is primary (0=JPEG, 1=RAW)

**Unknown**:
- How Photos decides which camera RAW formats to support
- RAW processing pipeline
- Why subtype value is specifically `16` or `17`
- Are there other subtype values used?
- Thumbnail extraction from RAW files

**Database**:
```sql
-- From code comments:
-- "RKVersion.subType, RAW always appears to be 16"
-- Why 16? Are there other values?
```

**Impact**: Can locate RAW files but don't understand full RAW handling logic.

### 11. Depth Maps and Portrait Mode ⚠️

**Status**: Detection Documented, Extraction Incomplete

**Known**:
- Portrait photos have `ZKINDSUBTYPE = 110`
- Depth state in `ZADDITIONALASSETATTRIBUTES.ZDEPTHSTATE`

**Unknown**:
- How to extract depth maps from HEIC files reliably
- Depth map format specification
- Portrait effect application algorithm
- How Photos uses depth for effects

**Impact**: Can detect portrait mode but depth extraction requires external libraries.

### 12. File Derivative Naming Convention ⚠️

**Status**: Pattern Known, Meaning Partially Unknown

**Format**: `{UUID}_{quality}_{size}_{type}.{ext}`

**Known**:
```
Quality: 1 (standard), 4 (high quality)
Size: 105, 201, 1006, 5005, 5025 (appear to be pixel dimensions)
Type: c (color/standard), a (adjusted)
```

**Unknown**:
- Exact meaning of size codes (105 = 105 pixels? on which dimension?)
- Are there other type codes besides 'c' and 'a'?
- When are different quality levels generated?
- Size selection algorithm

**Impact**: Can locate common derivatives but may miss non-standard ones.

### 13. Version-Specific Column Removals ❌

**Status**: Changes Documented, Reasons Unknown

**Examples**:

```sql
-- Photos 10+ removed:
ZADDITIONALASSETATTRIBUTES.ZMASTERFINGERPRINT
-- Replaced by: ZORIGINALSTABLEHASH
-- Why? Different algorithm? Format change?

-- Photos 11+ removed:
ZCOMPUTEDASSETATTRIBUTES.ZBEHAVIORALSCORE
ZCOMPUTEDASSETATTRIBUTES.ZINTERACTIONSCORE
-- Why removed? Privacy? Different scoring system?

-- Photos 7+ changed:
ZADDITIONALASSETATTRIBUTES.ZCUSTOMRENDEREDVALUE (HDR)
-- Replaced by: ZHDRTYPE
-- Why? Format change? More HDR types?

-- Photos 8+ changed:
ZDETECTEDFACE.ZPERSON → ZPERSONFORFACE
ZDETECTEDFACE.ZASSET → ZASSETFORFACE
-- Why rename? Core Data requirement?
```

**Impact**: Must handle version differences but don't understand underlying reasons.

### 14. Proprietary Fingerprinting 🔒

**Status**: Field Exists, Algorithm Proprietary

**Used For**:
- Duplicate detection
- File integrity verification
- Library merging

**Available**:
```sql
-- Photos 5-9:
ZADDITIONALASSETATTRIBUTES.ZMASTERFINGERPRINT  -- Base64 hash

-- Photos 10+:
ZADDITIONALASSETATTRIBUTES.ZORIGINALSTABLEHASH  -- Integer hash

-- All versions:
ZINTERNALRESOURCE.ZFINGERPRINT  -- Resource fingerprint
```

**Unknown**:
- Hashing algorithm
- Input data (just file bytes? includes metadata?)
- Why algorithm changed in Photos 10
- How to compute fingerprints

**From osxphotos**:
```python
# Uses private CloudPhotoLibrary framework
# Cannot be implemented without Apple's APIs
```

**Impact**: Cannot compute fingerprints for deduplication without Apple frameworks.

### 15. Media Analysis Framework 🔒

**Status**: Results Readable, Processing Proprietary

**Location**: `private/com.apple.mediaanalysisd/MediaAnalysis/`

**Available Results**:
- ML classifications
- Scene detection
- Object detection
- Text detection (OCR)

**Unknown**:
- ML models used
- Training data
- Confidence thresholds
- Processing pipeline
- Vector database format (`vector_database/`)

**Impact**: Can read results but cannot replicate analysis or understand model decisions.

### 16. Database Column Purpose Unknown

**Columns with unclear purpose**:

```sql
-- ZSHARE table:
ZAUTOSHAREPOLICY              -- Auto-share rules?
ZPARTICIPANTCLOUDUPDATESTATE  -- Participant sync state?
ZPREVIEWSTATE                 -- Preview generation state?
ZSCOPESYNCINGSTATE            -- Scope sync state?
ZEXITSOURCE, ZEXITTYPE        -- How sharing ended?
ZFORCESYNCATTEMPTED           -- Force sync meaning?

-- ZASSET table:
ZVISIBILITYSTATE              -- Visibility levels beyond 0/2?
ZPACKEDACCEPTABLECROPRECT     -- Crop suggestions?
ZPACKEDPREFERREDCROPRECT      -- Preferred crop?

-- ZDETECTEDFACE table:
ZMASTERIDENTIFIER             -- Master face?
ZCONFIRMEDFACECROPGENERATIONSTATE  -- Crop generation?
ZTRAININTYPE                  -- Training state?

-- ZMOMENT table:
ZTITLE, ZSUBTITLE             -- Generation algorithm?
```

**Impact**: Fields exist but purpose unclear, may affect some features.

### 17. Search Category Version Differences ⚠️

**Status**: Categories Known, Migration Path Unclear

**Issue**: Search categories changed between versions:

```python
# Photos 5-7 categories differ from Photos 8+ categories
# Some categories renumbered
# New categories added in Photos 8:
#   - CAMERA
#   - TEXT_FOUND
#   - DETECTED_TEXT

# Migration of old searches to new categories unknown
```

**Impact**: May misinterpret search results when reading libraries from different versions.

### 18. Incomplete PhotoKit Integration ⚠️

**From osxphotos code TODOs**:

```python
# Line 566 (photokit.py):
# "if user has selected use RAW as original, this returns the RAW"
# "may not work for all cases"

# Line 824 (photokit.py):
# "doesn't work for slow-mo videos"

# Line 902 (photoinfo.py):
# "need reliable way to get original UTI for shared"
```

**Impact**: Some edge cases not fully handled in library access.

## Minor Gaps

### Uncatalogued File Extensions

Some RAW formats may not be in UTI map:
- Pentax PEF
- Sigma X3F
- Phase One IIQ
- Hasselblad 3FR

**Workaround**: Use generic RAW detection or external libraries.

### Journal File Format

```
resources/journals/
├── Asset-snapshot.plj
├── Asset-change.plj
└── *.plist
```

**Status**: Presence documented, format not specified
**Impact**: Cannot interpret change journals

### Thumbnail Cache Format

```
resources/derivatives/thumbs/
├── thumbnailConfiguration
└── *.ithmb
```

**Status**: Legacy format, not documented
**Impact**: Must regenerate thumbnails, cannot use cache

## Workarounds and Limitations

### For Implementers

1. **Shared iCloud Libraries**: Assume experimental, test thoroughly
2. **Face Clustering**: Use existing associations, don't attempt to cluster
3. **Fingerprints**: Use external hashing (MD5, SHA256) for deduplication
4. **Scores**: Treat as relative rankings, not absolute values
5. **Edits**: Store adjustment data but don't attempt to render
6. **Moments**: Use existing moments, don't attempt to create new ones
7. **Search Indices**: Use psi.sqlite only, ignore binary indices
8. **Cloud Sync**: Read current state only, don't predict transitions

### Testing Requirements

Due to these gaps, implementers should:
- Test against multiple Photos versions (5-11+)
- Test with and without iCloud enabled
- Test shared libraries and albums
- Test all special media types
- Handle missing/unknown data gracefully
- Validate against Photos.app as ground truth

## Future Work

Areas where additional reverse engineering could help:

1. **Priority High**:
   - Shared iCloud library relationships
   - Face clustering algorithm insights
   - Cloud sync state machine

2. **Priority Medium**:
   - Moment grouping heuristics
   - Burst selection criteria
   - Score calculation approximations

3. **Priority Low**:
   - Binary search index format
   - Derivative naming conventions
   - Journal file format

## Conclusion

While this specification provides comprehensive coverage of the .photoslibrary format for most use cases, certain advanced features remain incompletely understood. These gaps primarily affect:

- Cloud synchronization features
- Advanced photo analysis (faces, aesthetics)
- Proprietary algorithms (rendering, scoring)
- Binary file formats (search indices, journals)

For most read-only library access use cases, the documented portions are sufficient. Features marked as proprietary 🔒 require Apple's frameworks and cannot be reverse-engineered.

## Related Documents

- [[00-index]] - Overview
- [[02-database-schema]] - Database reference
- [[05-version-compatibility]] - Version handling
- [[08-implementation-guide]] - Implementation examples

---

**Status**: Living document - will be updated as new information becomes available.
