---
title: Windows.Forensics.CertUtil
hidden: true
tags: [Client Artifact]
---

The Windows Certutil binary is capable of downloading arbitrary
files. Attackers typically use it to fetch tools undetected using
Living off the Land (LOL) techniques.

Certutil maintains a cache of the downloaded files and this contains
valuable metadata. This artifact parses this metadata to establish
what was downloaded and when.


```yaml
name: Windows.Forensics.CertUtil
description: |
  The Windows Certutil binary is capable of downloading arbitrary
  files. Attackers typically use it to fetch tools undetected using
  Living off the Land (LOL) techniques.

  Certutil maintains a cache of the downloaded files and this contains
  valuable metadata. This artifact parses this metadata to establish
  what was downloaded and when.

reference:
  - https://u0041.co/blog/post/3

parameters:
  - name: URLWhitelist
    type: csv
    default: |
      URL
      http://sf.symcd.com
      http://oneocsp.microsoft.com
      http://certificates.godaddy.com
      http://ocsp.pki.goog
      http://repository.certum.pl
      http://www.microsoft.com
      http://ocsp.verisign.com
      http://ctldl.windowsupdate.com
      http://ocsp.sectigo.com
      http://ocsp.usertrust.com
      http://ocsp.comodoca.com
      http://cacerts.digicert.com
      http://ocsp.digicert.com
  - name: MetadataGlobUser
    default: C:/Users/*/AppData/LocalLow/Microsoft/CryptnetUrlCache/MetaData/*
  - name: MetadataGlobSystem
    default: C:/Windows/*/config/systemprofile/AppData/LocalLow/Microsoft/CryptnetUrlCache/MetaData/*
  - name: AlsoUpload
    type: bool

sources:
  - query: |
      LET Profile = '[
        ["Header", 0, [
          ["UrlSize", 12, "uint32"],
          ["HashSize", 100, "uint32"],
          ["DownloadTime", 16, "uint64"],
          ["FileSize", 112, "uint32"],
          ["URL", 116, "String", {
              "encoding": "utf16",
              "length": "x=>x.UrlSize"
          }],
          ["Hash", "x=>x.UrlSize + 116", "String", {
              "encoding": "utf16",
              "length": "x=>x.HashSize"
          }]
        ]]
      ]'

      -- Build a whitelist regex
      LET URLRegex <= "^" + join(array=URLWhitelist.URL, sep="|")
      LET Files = SELECT FullPath,

          -- Parse each metadata file.
          parse_binary(filename=FullPath,
                       profile=Profile,
                       struct="Header") AS Header,

                       -- The content is kept in the Content directory.
                       regex_replace(re="MetaData",
                                     replace="Content",
                                     source=FullPath) AS Content
      FROM glob(globs=[MetadataGlobUser,MetadataGlobSystem])

      SELECT * FROM foreach(row=Files,
      query={
          SELECT FullPath AS _MetadataFile,
               if(condition=AlsoUpload, then=upload(file=FullPath)) AS _MetdataUpload,
               if(condition=AlsoUpload, then=upload(file=Content)) AS _Upload,
               Header.URL AS URL,
               Header.FileSize AS FileSize,
               regex_replace(re='"', replace="", source=Header.Hash) AS Hash,
               timestamp(winfiletime=Header.DownloadTime) AS DownloadTime
          FROM scope()
          WHERE NOT URL =~ URLRegex
      })

```
