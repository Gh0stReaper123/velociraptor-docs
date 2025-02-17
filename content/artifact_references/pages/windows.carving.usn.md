---
title: Windows.Carving.USN
hidden: true
tags: [Client Artifact]
---

Carve URN Journal records from the disk.

The USN journal is a very important source of information about when
and how files were manipulated on the filesystem. However, typically
the journal is rotated within a few days.

This artifact carves out USN journal entries from the raw disk. This
might recover older entries which have since been rotated from the
journal file.

## Notes

1. Like all carving, USN carving is not very reliable. You
   would tend to use it to corroborate an existing theory or to
   discover new leads.

2. This artifact takes a long time to complete - you should
   probably increase the collection timeout past 10 minutes (usually
   more than an hour).

3. The reassembled FullPath is derived from the MFTId referenced in
   the USN record. Bear in mind that this might be out of date and
   inaccurate.

4. If you need to carve from a standalone file (e.g. collection from
   `Windows.KapeFiles.Targets`) you should use the
   Windows.Carving.USNFiles artifact instead.


```yaml
name: Windows.Carving.USN
description: |
  Carve URN Journal records from the disk.

  The USN journal is a very important source of information about when
  and how files were manipulated on the filesystem. However, typically
  the journal is rotated within a few days.

  This artifact carves out USN journal entries from the raw disk. This
  might recover older entries which have since been rotated from the
  journal file.

  ## Notes

  1. Like all carving, USN carving is not very reliable. You
     would tend to use it to corroborate an existing theory or to
     discover new leads.

  2. This artifact takes a long time to complete - you should
     probably increase the collection timeout past 10 minutes (usually
     more than an hour).

  3. The reassembled FullPath is derived from the MFTId referenced in
     the USN record. Bear in mind that this might be out of date and
     inaccurate.

  4. If you need to carve from a standalone file (e.g. collection from
     `Windows.KapeFiles.Targets`) you should use the
     Windows.Carving.USNFiles artifact instead.

parameters:
  - name: DriveToScan
    default: "C:"
  - name: FileRegex
    description: "Regex search over File Name"
    default: "."
    type: regex
  - name: DateAfter
    type: timestamp
    description: "search for events after this date. YYYY-MM-DDTmm:hh:ssZ"
  - name: DateBefore
    type: timestamp
    description: "search for events before this date. YYYY-MM-DDTmm:hh:ssZ"

export: |
  -- Profile to parse the USN record
  LET USNProfile = '''[
            ["USN_RECORD_V2", 4, [
                ["RecordLength", 0, "unsigned long"],
                ["MajorVersion", 4, "unsigned short"],
                ["MinorVersion", 6, "unsigned short"],
                ["FileReferenceNumberSequence", 8, "BitField", {
                    "type": "unsigned long long",
                    "start_bit": 48,
                    "end_bit": 63
                }],
                ["FileReferenceNumberID", 8, "BitField", {
                    "type": "unsigned long long",
                    "start_bit": 0,
                    "end_bit": 48
                }],
                ["ParentFileReferenceNumberSequence", 16, "BitField", {
                    "type": "unsigned long long",
                    "start_bit": 48,
                    "end_bit": 63
                }],
                ["ParentFileReferenceNumberID", 16, "BitField", {
                    "type": "unsigned long long",
                    "start_bit": 0,
                    "end_bit": 48
                }],

                ["Usn", 24, "unsigned long long"],
                ["TimeStamp", 32, "WinFileTime"],
                ["Reason", 40, "Flags", {
                    "type": "unsigned long",
                    "bitmap": {
                        "DATA_OVERWRITE": 0,
                        "DATA_EXTEND": 1,
                        "DATA_TRUNCATION": 2,
                        "NAMED_DATA_OVERWRITE": 4,
                        "NAMED_DATA_EXTEND": 5,
                        "NAMED_DATA_TRUNCATION": 6,
                        "FILE_CREATE": 8,
                        "FILE_DELETE": 9,
                        "EA_CHANGE": 10,
                        "SECURITY_CHANGE": 11,
                        "RENAME_OLD_NAME": 12,
                        "RENAME_NEW_NAME": 13,
                        "INDEXABLE_CHANGE": 14,
                        "BASIC_INFO_CHANGE": 15,
                        "HARD_LINK_CHANGE": 16,
                        "COMPRESSION_CHANGE": 17,
                        "ENCRYPTION_CHANGE": 18,
                        "OBJECT_ID_CHANGE": 19,
                        "REPARSE_POINT_CHANGE": 20,
                        "STREAM_CHANGE": 21,
                        "CLOSE": 31
                    }
                }],
                ["SourceInfo", 44, "Flags", {
                    "type": "unsigned long",
                    "bitmap": {
                        "DATA_MANAGEMENT": 0,
                        "AUXILIARY_DATA": 1,
                        "REPLICATION_MANAGEMENT": 2
                    }
                }],
                ["SecurityId", 48, "unsigned long"],
                ["FileAttributes", 52, "Flags", {
                    "type": "unsigned long",
                    "bitmap": {
                        "READONLY": 0,
                        "HIDDEN": 1,
                        "SYSTEM": 2,
                        "DIRECTORY": 4,
                        "ARCHIVE": 5,
                        "DEVICE": 6,
                        "NORMAL": 7,
                        "TEMPORARY": 8,
                        "SPARSE_FILE": 9,
                        "REPARSE_POINT": 10,
                        "COMPRESSED": 11,
                        "OFFLINE": 12,
                        "NOT_CONTENT_INDEXED": 13,
                        "ENCRYPTED": 14,
                        "INTEGRITY_STREAM": 15,
                        "VIRTUAL": 16,
                        "NO_SCRUB_DATA": 17
                    }
                }],
                ["FileNameLength", 56, "unsigned short"],
                ["FileNameOffset", 58, "unsigned short"],
                ["Filename", "x=>x.FileNameOffset", "String", {
                    encoding: "utf16",
                    length: "x=>x.FileNameLength",
                }]
            ]]]
        '''

sources:
  - precondition:
      SELECT OS From info() where OS = 'windows'

    query: |
        -- firstly set timebounds for performance
        LET DateAfterTime <= if(condition=DateAfter,
             then=DateAfter, else="1600-01-01")
        LET DateBeforeTime <= if(condition=DateBefore,
            then=DateBefore, else="2200-01-01")

        LET Device <= '''\\.\''' + DriveToScan

        -- This rule performs an initial reduction for speed, then we
        -- reduce further using other conditions.
        LET YaraRule = '''rule X {
            strings:
              // First byte is the record length < 255 second byte should be 0-1 (0-512 bytes per record)
              // Version Major and Minor must be 2 and 0
              // D7 01 is the ending of a reasonable WinFileTime
              // Name Offset and Name Length are short ints but should be < 255
              $a = { ?? (00 | 01) 00 00 02 00 00 00 [24] ?? ?? ?? ?? ?? ?? D? 01 [16] ?? 00 3c 00  }
            condition:
              any of them
        }
        '''

        -- Find all the records in the drive.
        LET Hits = SELECT String.Offset AS Offset, parse_binary(
           filename=Device, accessor="ntfs", struct="USN_RECORD_V2",
           profile=USNProfile, offset=String.Offset) AS _Parsed
        FROM yara(files=Device, accessor="ntfs", rules=YaraRule, number=200000000)
        WHERE _Parsed.RecordLength > 60 AND  // Record must be at least 60 bytes
              _Parsed.FileNameLength > 3 AND _Parsed.FileNameLength < 100

        LET FlatHits = SELECT Offset, _Parsed.TimeStamp AS TimeStamp, _Parsed.Filename AS Name,
               _Parsed.FileReferenceNumberID AS MFTId,
               parse_ntfs(device=Device, mft=_Parsed.FileReferenceNumberID) AS MFTEntry,
               _Parsed.ParentFileReferenceNumberID AS ParentMFTId,
               _Parsed.Reason AS Reason
        FROM Hits
        WHERE Name =~ FileRegex AND
              TimeStamp < DateBeforeTime AND
              TimeStamp > DateAfterTime

        SELECT Offset, TimeStamp, Name, MFTId,
               MFTEntry.FullPath AS FullPath,
               ParentMFTId, Reason
        FROM FlatHits

```
