---
title: Windows.Sys.PhysicalMemoryRanges
hidden: true
tags: [Client Artifact]
---

List Windows physical memory ranges.

```yaml
name: Windows.Sys.PhysicalMemoryRanges
description: List Windows physical memory ranges.
reference:
  - https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/content/wdm/ns-wdm-_cm_resource_list

parameters:
  - name: physicalMemoryKey
    default: HKEY_LOCAL_MACHINE\HARDWARE\RESOURCEMAP\System Resources\Physical Memory\.Translated

export: |
  LET Profile = '''
      [
        ["CM_RESOURCE_LIST", 0, [
          ["Count", 0, "uint32"],
          ["List", 4, "CM_FULL_RESOURCE_DESCRIPTOR"]
        ]],
        ["CM_FULL_RESOURCE_DESCRIPTOR", 0, [
           ["PartialResourceList", 8, "CM_PARTIAL_RESOURCE_LIST"]
        ]],

        ["CM_PARTIAL_RESOURCE_LIST", 0, [
           ["Version", 0, "uint16"],
           ["Revision", 2, "uint16"],
           ["Count", 4, "uint32"],
           ["PartialDescriptors", 8, "Array", {
              "type": "CM_PARTIAL_RESOURCE_DESCRIPTOR",
              "count": "x=>x.Count"
           }]
        ]],

        ["CM_PARTIAL_RESOURCE_DESCRIPTOR", 20, [
           ["Type", 0, "char"],
           ["ShareDisposition", 1, "char"],
           ["Flags",2, "uint16"],
           ["Start",4, "int64"],
           ["Length",12, "uint32"]
        ]]
      ]
  '''

sources:
  - precondition:
      SELECT OS From info() where OS = 'windows'
    query: |
      SELECT * FROM foreach(
         row={SELECT Data from stat(filename=physicalMemoryKey, accessor="registry")},
         query={
            SELECT * FROM foreach(
               row=parse_binary(
                  filename=Data.value,
                  accessor="data",
                  profile=Profile,
                  struct="CM_RESOURCE_LIST").List.PartialResourceList.PartialDescriptors,
               query={
                  SELECT Type,
                         format(format="%#0x", args=Start) AS Start,
                         format(format="%#0x", args=Length) AS Length
                  FROM scope()
              })
      })

```
