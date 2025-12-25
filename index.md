---
layout: default
---

## Software

I'm not the author of all of these projects; in some cases they are customized versions
or friendly forks of existing utilities.

* [Aperture Library Extractor](https://github.com/Jachimo/aplib-extractor) - A hacky fork of
  [Hubert Figuière's aplib-extractor](https://github.com/hfiguiere/aplib-extractor), built to
  assist in migrating my 60,000+ images, with their metadata, out of the Apple Aperture
  Library format and into plain image-and-sidecar-in-directories for use with other DAM tools.
* [AppleDouble to XMP](https://github.com/Jachimo/experiments/tree/main/apple_metadata) -
  One-off tool to convert Apple Finder metadata stored in "AppleDouble" files (those mysterious
  "._filename" files that appear when a Mac interacts with a non-HFS filesystem) and convert
  them—particularly the colored "Label" flags—to XMP files that can be used in a DAM workflow.
* [Taky](https://github.com/Jachimo/taky) - My fork of tkuester's well-known
  [simple Python TAK server](https://github.com/tkuester/taky).  Contains a few arguable
  improvements, including the addition of Oracle Cloud Infrastructure (OCI) Object Storage
  as a backend database for the server.
* [CoT Tools](https://github.com/Jachimo/cot-tools) - Test utilities for working with the de facto
  standard 'Cursor on Target' (CoT) format developed by Mitre for tactical data interchange, on
  UDP multicast networks.
* [Obsidian Tagging Tools](https://github.com/Jachimo/obstagtools) - Various tools
  for working with [Obsidian](https://obsidian.md/) documents, including batch tagging,
  filtered exports, taxonomies, and a very rough attempt at automatic tagging.
* [MBOX to CouchDB](https://github.com/Jachimo/mbox-to-couchdb) - Import email messages from a Unix-style
  MBOX file into CouchDB for further analysis.
* [.eml to MBOX](https://github.com/Jachimo/emlToMbox) - Add one or more individual
  MIME email files (.eml), such as those produced by Apple Mail and Microsoft Outlook,
  to a new or existing Unix MBOX file.
* [MIME Wrapper](https://github.com/Jachimo/mimewrapper) - Takes an
  arbitrary file and 'wraps' it in the RFC 2045 MIME (".eml") container format, using specified
  values from a sidecar file to set whatever headers you'd like.  Allows easy storage/retrieval/archiving
  of basically any file, with accompanying metadata, using any email server as the backend.
* [Adium to .eml](https://github.com/Jachimo/adiumtoeml) - Inspired by "iChat to .eml"
  (see below), this performs a similar function for chat logs created by the Mac OS X
  "Adium" messaging client, which I used heavily in the early 2000s.
* [MBOX Deduplicator](https://github.com/Jachimo/mbox_dedupe) - A rather trivial
  little utility to take an .mbox file and deduplicate it based on `Message-ID`
  header values.  It's basic but it works, and handy if you're working with other
  email tools and happen to (hypothetically) export some messages into the same
  destination file more than once.
* [Mailbox Lightener](https://github.com/Jachimo/mailbox_lightener) - Processes a
  saved .mbox (Unix mailbox) file and strips it down to a single `text/plain` part
  per message, discarding HTML formatting, attachments, and extraneous headers; good
  for producing machine-readable versions of email archives for analysis.
* [iChat to .eml](https://github.com/Jachimo/ichat_to_eml) - Converts saved
  iChat conversations into MIME text files (which typically have the .eml extension)
  so they can be imported into mail programs and searched/read/archived like
  email. Neither my fork or the upstream version have been maintained recently.

## Hardware

TBD
