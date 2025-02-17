---
title: Windows.System.Powershell.ModuleAnalysisCache
hidden: true
tags: [Client Artifact]
---



```yaml
name: Windows.System.Powershell.ModuleAnalysisCache

reference:
 - https://github.com/PowerShell/PowerShell/blob/281b437a65360ae869d40f3766a1f2bbba786e5e/src/System.Management.Automation/engine/Modules/AnalysisCache.cs#L649

parameters:
  - name: GlobLookup
    default: C:\Windows\System32\config\systemprofile\AppData\Local\Microsoft\Windows\PowerShell\ModuleAnalysisCache

sources:
  - query: |
      LET Profile = '
       [
         ["Header", 0, [
           ["Signature", 0, "String", {"length": 13}],
           ["CountOfEntries", 14, "uint32"],
           ["Entries", 18, "Array",
                 {"type": "Entry", "count": "x => x.CountOfEntries"}]
         ]],

         ["Entry", "x=>x.Func.SizeOf + x.ModuleLength + 20", [
           ["Offset", 0, "Value", {"value": "x => x.StartOf"}],
           ["TimestampTicks", 0, "uint64"],
           ["ModuleLength", 8, "uint32"],
           ["ModuleName", 12, "String", {"length": "x => x.ModuleLength"}],
           ["CommandCount", "x => x.ModuleLength + 12", "uint32"],
           ["Func", "x => x.ModuleLength + 16", "Array",
                  {"type": "FunctionInfo", "count": "x => x.CommandCount"}],
           ["CountOfTypes", "x => x.Func.EndOf", "uint32"]
         ]],

         ["FunctionInfo", "x => x.NameLen + 8", [
           ["NameLen", 0, "uint32"],
           ["Name", 4, "String", {"length": "x => x.NameLen"}],
           ["Count", "x => x.NameLen + 4", "uint32"]
         ]]
       ]
      '
      LET parsed = SELECT FullPath,
         parse_binary(filename=FullPath, profile=Profile, struct="Header") AS Header
      FROM glob(globs=GlobLookup)

      SELECT * FROM foreach(row=parsed,
      query={
         SELECT * FROM foreach(row=Header.Entries,
         query={
            SELECT FullPath, ModuleName,
                  timestamp(epoch=TimestampTicks/10000000 - 62136892800) AS Timestamp,
                  Func.Name AS Functions
            FROM scope()
         })
      })

```
