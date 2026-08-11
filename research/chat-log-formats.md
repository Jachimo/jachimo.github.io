---
layout: default
rfcdate: Fri Mar 27 16:21:06 2026 -0400
date: 2026-03-27
title: Chat Log Formats
---

## Introduction

This page summarizes the information I've found while researching file
formats used by various instant messaging (IM) client programs to save
chat data (chat logs) to disk.

Much is owed to Github user "Kadin2048", on whose [2021 Gist][] this
page was originally based.

[2021 Gist]: https://gist.github.com/kadin2048/ffe811e56c8e8fb6ceb8bade09439341

Corrections and additional information are requested.


### Summary

| Client                        | File Extension | File Type        |   Self Contained?   |
|-------------------------------|----------------|------------------|---------------------|
| AOL Instant Messenger for Mac | *none*         | Flat text        |      Yes            |
| CenterIM                      | *none*         | Flat text        |      No             |
| IBM Sametime Connect          | .html          | HTML Document    |     No              |
| Pidgin                        | .html          | HTML Document    |     Yes             |
| Adium (Old Style)             | .AdiumHTMLLog  | HTML Fragment    | Yes (with filename) |
| Adium (ULF)                   | .chatlog       | XML Document     |      Yes            |
| iChat                         | .chat          | typedstream data |    Yes              |
| Apple Messages (file-based)   | .ichat         | Binary PList     |      Yes            |
| Apple Messages (chat.db)      | .db            | SQLite Database  |     Yes             |
| iOS Messages (sms.db)         | .db            | SQLite Database  |     Yes             |

Note that "Self Contained" refers to whether the logs contain enough
information to completely reconstruct a chat conversation without
additional information from outside the log files themselves; it does
*not* mean that attachments are necessarily included.

For instance, IBM Sametime Connect logs, stored as `.html` files, are
not self-contained because they do not contain the far-side username
(the other person that the chat is with) in the logs themselves; this
information has to be reconstructed from metadata files and/or the
directory that the file resides in.


## AOL Instant Messenger for Mac

At the height of its popularity in the late 90s and early 00s, AOL offered official client applications for AOL IM on several platforms, including Classic (pre-OS X) MacOS.  My earliest IM logs are from this software and are still easily readable today, owing to the relatively simple plain-text format.

Although I do not have a working Classic Mac system to run the software and check, I believe that logs were saved to a folder inside `Macintosh HD:System Folder:Preferences`, and were grouped into subfolders by local account name.

The files are named according to the pattern `[Screen Name] IM Log` where [Screen Name] is the remote party's AOL IM screen name.  Note that there is no file extension as part of the name, as extensions were not typically used on Classic MacOS (in favor of filesystem Creator and Type metadata, stored in the file's Resource Fork and often lost over time if the files were ever copied to a non-HFS filesystem).

File type and creator were set to `TXEX/txtt` on the oldest HFS backups I was able to locate and analyze.

Each logfile contains multiple conversations with the same remote screenname.  The maximum size of a log file is uncertain; none of mine are larger than about 500kB.  The text encoding used is also uncertain.  Assuming the files have not been converted or re-saved using a editor that silently converts them, line separators are 0x0D (Carriage Return) rather than the now-common Unix-style 0x0A (Line Feed) or the Windows-standard 0x0D 0x0A (CR/LF).  Many logfiles I've found contain odd high-ASCII values, but they are not clearly in any particular encoding.

Each conversation within the file begins with a date and time stamp (usually formatted either as "MM/DD/YY hh:MM AP" or "mm/DD/YY HH MM") and a Carriage Return (0x0D).

Each message within the conversation begins on a new line and starts with the AOL screen name of the party sending the message, followed by a colon (0x3A), a tab (0x09), and then the message, terminated by a 0x0D.  However, messages can span multiple lines (by virtue of containing Carriage Return characters), and copying and pasting messages (something I did often, apparently) can result in what are effectively *nested* messages.

The following is a legal chat log containing several edge cases, shown with the following substitutions for display purposes:

* [CR] for 0x0D (ASCII Carriage Return)
* [SOH] for 0x01 (ASCII Start of Header, appears to be used as a placeholder character, perhaps for graphical smileys?)
* [C2] for 0xC2 (Unknown Character; represents Unicode ¬ NOT SIGN U+00AC in MacRoman charset)
* [A7] for 0xA7 (Unknown Character; represents Unicode ß LATIN SMALL LETTER SHARP S U+00DF in MacRoman charset)
* [EOF] for the logical end of the file
* Horizontal Tab (0x09) characters are left unconverted, since they are the same in UTF-8

"TheirAIMName IM Log":

    1/18/2000 22 58[CR]
    MyAIMName:	this is a legal message:[CR]
    MyAIMName:	[SOH][CR]
    MyAIMName:	and so is the following one[CR]
    MyAIMName:	we can be [SOH] and [SOH] ...[CR]
    TheirAIMName:	anyway i need to go[CR]
    TheirAIMName:	k[CR]
    MyAIMName:	back[CR]
    [CR]
    Auto response from TheirAIMName:[CR]
    TheirAIMName:	i'm checking my negatives right now.. be right back[CR]
    [CR]
    MyAIMName:	k[CR]
    TheirAIMName:	i'm back[CR]
    [CR]
    7/19/00 19 08[CR]
    TheirAIMName:	hello[CR]
    MyAIMName:	hi[CR]
    MyAIMName:	what's goin on>[CR]
    MyAIMName:	?[CR]
    TheirAIMName:	ah[CR]
    TheirAIMName:	not much[CR]
    MyAIMName:	and what's your nick?[CR]
    [CR]
    TheirAIMName:	OldName82[CR]
    TheirAIMName:	 Hunter [C2][A7]t@rcing.Boxers4KI 01:29:35[CR]
    Bet shes gonna shella alotta dough for that operation[CR]
    MyAIMName:	lol[EOF]

Note that the files *do not* generally end with a newline.


## CenterIM

Official page: https://github.com/petrpavlu/centerim5

CenterIM is a text-mode, multi-protocol instant messaging client using
libpurple on the back end.  This means it supports all the same
protocols that other libpurple-based clients, such as Pidgin, do.
However, its logs are written in a different format than Pidgin's.

CenterIM's logs are stored inside the `~/.centerim` directory as text
files (seemingly ASCII, but other character sets may be supported)
inside specially-named subdirectories.

A directory is created for each chat partner, prefixed with a letter
indicating the protocol in use.  E.g. an AOL Instant Messenger chat
with a user named "joecool" would be stored in a directory named
"ajoecool", with the prefix "a" indicating AOL IM.

| Protocol                      | Directory Prefix |
|-------------------------------|----------------|
| AOL Instant Messenger (AOLIM) | a              |
| Microsoft Messenger (MSN)     | m              |
| Jabber / Google Chat          | j              |

Inside each directory is a file named "history", holding the actual messages.

Each message begins with an ASCII form feed character (hex 0C) on a line by itself, followed by the string "IN" or "OUT" depending on whether the message is incoming or outgoing (from the perspective of the client program), the string "MSG" on a line by itself, then two Unix timestamps (seemingly always the same; I'm not sure why it's repeated) on lines by themselves, and then the message text on the last line.

Example (with ASCII Form Feeds represented as "[FF]"):

    [FF]
    IN
    MSG
    1196779655
    1196779655
    hey there! thanks for the invite, but my company holiday party is that day, so I can't make it. Next time?
    [FF]
    OUT
    MSG
    1196782478
    1196782478
    hey no problem, just wanted to let you know you're invited if you were in town

Note that nowhere in the actual log file are either the near- or far-side account names given.  The far-side account name can be determined from the name of the enclosing directory, but the near-side name has to be supplied by the user in some other fashion.


### CenterIM to XML ULF in Python

A description of a CenterIM log file parser written in Python can
be found in [this 2007 blog post](http://kadin.sdf-us.org/blog/technology/centerim-converter.html)
(as long as it stays online).


### CenterIM to XML ULF in Java

The same author also created a Java utility to convert CenterIM logs
to XML with the 'Unified Logging Format' schema; it can be found [in
this Gist](https://gist.github.com/kadin2048/1e6a1f1204b56d08e6612f9b33dc44f7).

The actual message-parsing loop:

    while ( (line = brSourceFile.readLine()) != null ) {
        if ( line.indexOf("\f") != -1 ) {
            // If we're looking at a formfeed, skip it by re-running the loop
            continue;
        }
        xmlout.writeEntity("message");
        msgs++; // Increment the message counter
        
        String direction = line; // First real line should be "IN" or "OUT"
        xmlout.writeAttribute("direction", direction);
        
        if (direction.indexOf("IN") != -1) {
        	// If it's an incoming message...
        	xmlout.writeAttribute("sender", farEndName);
        } else {
        	// If it's an outgoing message...
        	xmlout.writeAttribute("sender", sNearEndName);
        }
        
        brSourceFile.readLine(); // Then is the string "MSG", we skip it
        
        String timestamp = brSourceFile.readLine(); // Next should be timestamp
        if ( !bNoUnixDate.getValue() ) {
        	xmlout.writeAttribute("unixtime", timestamp);  // Write it unconverted
        }
        unixdate = Integer.parseInt(timestamp);
        javadate = (long) unixdate * 1000; // Java uses ms, Unix uses secs
        msgdate = new Date(javadate); // Convert the long to a date object
        xmlout.writeAttribute("time", df.format(msgdate));
        
        brSourceFile.readLine(); // Then skip the second, redundant timestamp
        String message = brSourceFile.readLine(); // Then read the message
        xmlout.writeText(message); // write the message out
        
        xmlout.endEntity(); // close the </message>
    }


## IBM Sametime Connect (HTML)

Note that this information pertains specifically to Sametime Connect
versions 7.x for Windows XP, which was in use circa 2005-2006 and
maybe later. It *does not* apply to the Sametime client application
embedded in Lotus Notes.

Unlike some other clients, Sametime Connect does not log by default;
the feature has to be explicitly enabled by the user.

As of 2021, [available public documentation from IBM and HCL Software
(the current owner of Sametime)](https://help.hcltechsw.com/sametime/11.5/connectclient/t_st8_uim_chat_hist.html)
does not specify the storage location of log files.  It is presumably
somewhere in the `\Application Data\` directory.

Within the logs directory is a `personfolders.xml` file, and
subdirectories for each conversation partner, named according to their
Sametime/Notes "contactID" (typically an email address).  Inside of
each of these directories is a series of subdirectories named by date
as `YYYYMMDD`, and a file named `ChatHistory.properties`.  These
by-date subdirectories hold the conversation logs, one HTML file per
day (so, one per directory).  The HTML files are named using the user
portion of their contactID (for "joecool@somecompany.example" this
would be "joecool") followed by "log.html" (e.g. producing
"joecoollog.html").

Directory structure:

    [Sametime Logs Folder]
      personfolders.xml
      [contactID]
        ChatHistory.xml
        [YYYYMMDD]
          [contactIDuser]log.html


### Sametime Metadata Sidecar Files

Neither the personfolders.xml or ChatHistory.properties files are
needed for reconstruction of usable chat logs, although they both
contain metadata that may be of interest to the archivist.

"personfolders.xml":

    <?xml version="1.0" encoding="UTF-8"?>
    <folders>
      <folder communityId="Sametime_0000000003593.0000000037" displayName="Joe Cool" folderPath="joe.b.cool@us.bigco.example" id="joe.b.cool@us.bigco.example" isExternal="false"/>
      [<folder ... /> elements repeat]
    </folders>
    

Note that the displayName attribute is not always filled-in; on many
of my folders the attribute is null.

"ChatHistory.properties":

    #Chat History Properties
    #Mon Aug 14 13:24:35 EDT 2006
    serverID=0 09118812
    

Tested versions of Sametime Connect produced complete, well-formed HTML
documents, one per chat partner per day.  However, similar to CenterIM
and some other clients, the far-side username is not explicitly
provided in the log itself.  Although a `<meta>` element in the
document `head` does provide an attribute `sametime:initiator`, this
does not seem to be guaranteed to be the remote-side party's username
(although in many instances it is, particularly if the first message
in that day's logs is from the remote side).

One way to reliably get the remote username is to consult the folder
hierarchy in which the logfile is found.  Typically the grandparent
directory of the log file will be named using the contactID.

A utility to convert Sametime HTML logs into RFC-compliant MIME
`multipart/mixed` email messages (for archiving in an IMAP mailbox) can
be found at [this Gist](https://gist.github.com/kadin2048/6d77ab7471590eedcc65).


## Pidgin

Official page: https://www.pidgin.im/

Pidgin, originally known as Gaim, is a multi-protocol IM client for
Linux and Windows.  Pidgin logs are well-formed HTML and include the
far-side (conversation partner's) account name inside the HTML `title`
element, making them "self-contained".

Github user "gabebw" wrote a converter from Pidgin to Adium log format
(in Ruby): [pidgin2adium](https://github.com/gabebw/pidgin2adium)

Github user "Kadin2048" [wrote a converter which converts from Pidgin
to MIME `.eml` files][pidgintoeml], a relatively easy task since they
are already well-formed HTML, and just need to be wrapped correctly
and paired with the right headers.

[pidgintoeml]: https://gist.github.com/kadin2048/ad0feef9f4f4230cd207907eceb17452

## Adium

Adium was an OS X native, multi-protocol IM client with a nice UI and
a variety of advanced features, including end-to-end encryption
(E2EE), which wouldn't become standard on most messaging systems until
years later.  (And it had a cute mascot.)

The official development site is/was: https://github.com/adium/adium

Adium came with a standalone 'Chat Transcript Viewer' program,
documented here: https://adium.im/help/pgs/Messaging-TranscriptViewer.html

Adium logs are usually stored in `~/Library/Application Support/Adium
2.0/Users/Default/Logs`, although this may vary if you had multiple
users configured (uncommon).

Inside the Logs folder are subfolders for each local messaging
account, named according to the protocol/service, a dot, and then the
account username.  Inside each account subfolder are further
subfolders for each remote conversation partner, by username.  Actual
logs are stored inside, one per conversation.

Adium Folder Hierarchy:

    ~/Library/Application Support/Adium 2.0/Users/Default/Logs/
      <protocol>.<local_username>
        <remote_username>
          <remote_username> (<datestamp>).<extension>

Adium used two distinct log file formats over its lifespan.  The first
was an HTML-based format saved with the file extension
".AdiumHTMLLog", and then the later was a semi-standardized, XML-based
format with the extension ".chatlog".  (This *may* have corresponded
to Adium 1.x versus 2.x.)


### Adium HTML Logs (.AdiumHTMLLog - Fragmentary Format)

Adium HTML-based logs with extension ".AdiumHTMLLog" are *not*
well-formed HTML, but rather a series of `<div>` and `<span>` elements
(aka "tag soup"), with each message on one line.  The file can be
mated to a header and footer to form a complete HTML document for
display, with the file contents inserted inside the HTML `<body>`
element.

Example, showing a short conversation between users "someguy" (remote) and "myusername" (local):

    <div class="receive"><span class="timestamp">12:41:38</span> <span class="sender">someguy: </span><pre class="message">hey</pre></div>
    <div class="send"><span class="timestamp">12:45:02</span> <span class="sender">myusername: </span><pre class="message">whats up?</pre></div>
    <div class="receive"><span class="timestamp">12:45:06</span> <span class="sender">someguy: </span><pre class="message">sorry</pre></div>
    <div class="send"><span class="timestamp">12:46:08</span> <span class="sender">myusername: </span><pre class="message">oh, no problem</pre></div>
    <div class="receive"><span class="timestamp">12:46:58</span> <span class="sender">someguy: </span><pre class="message">no biggie</pre></div>

Files do not contain any headers, but begin with the first `<div>`, and end with a newline.


### Adium XML Logs (.chatlog - Unified Logging Format)

Adium switched to an XML-based log schema referred to as the "Unified
Logging Format" (ULF) sometime around May 2005.  Despite the name, the
format never seemed to get wide adoption by any client other than
Adium.

This is a bit unfortunate, as it's a very nice archival format
compared to most others.

The ULF logs use the extension ".chatlog" and begin with an XML
declaration, specifying XML 1.0 and UTF-8.  This is followed by the
top-level `<chat>` element, which can contain one or more `<message>`
elements, featuring a number of possible attributes, and containing
the HTML-formatted message text.

Example of a ULF log containing two messages (indents and extra LFs added for clarity):

    <?xml version="1.0" encoding="UTF-8" ?>
    <chat xmlns="http://purl.org/net/ulf/ns/0.4-02" account="someguy" service="AIM">
      <message sender="someguy" time="2005-05-29T12:06:51-05:00">
        <div>
          <span style="background-color: #ffffff; color: #000093; font-family: Verdana; font-size: 10pt;">hey</span>
        </div>
      </message>
      <message sender="myusername" time="2005-05-29T12:07:13-05:00">
        <div>
          <span style="background-color: #acb5bf; color: #000000; font-family: Verdana; font-size: 10pt;">whats up?</span>
        </div>
      </message>
    </chat>

To get browser-renderable HTML from the XML logs, one method is via an
XML stylesheet and `libxslt`.  An example stylesheet can be found
[here](https://matthieu.yiptong.ca/2009/12/18/converting-adium-xml-chat-logs-to-html-format/).

The neat-looking ["Log2Log" program][log2log] (apparently abandoned as
of 2021?) also supported ULF XML.

[log2log]: https://log2log.sourceforge.io/

If you want to archive your chats using an email server (e.g. Gmail),
[this Python script][adiumemlgist] by Kadin2048 is one option (noted
by its author as "definitely not elegant, but it worked for me").

[adiumemlgist]: https://gist.github.com/kadin2048/8db8767686dfe93fe045


## Apple iChat.app

Apple iChat.app, originally called "iChat AV" (because it included
early voice and video conferencing support) was Apple's own OS X
native IM client, released in 2002.  It supported a number of
protocols, including AOL IM and direct peer-to-peer connections via
Rendezvous.

Originally, iChat.app logs were stored in "~/Documents/iChats", rather
than in the Library as with later versions. These logs usually have
the extension `.chat`.

The iChat.app logs are NeXT-style "typedstream" files.  As reported by
the `file` utility on a modern (2024) Mac:

    NeXT/Apple typedstream data, big endian, version 4, system 1000

[This StackOverflow question][so-ichat] confirms that these files are
different from the binary plists (as used by the ".ichat" files
produced by the newer Messages application), but NeXT-style
TypedStreams.

[so-ichat]: https://stackoverflow.com/questions/4371588/is-there-a-way-to-read-in-files-in-typedstream-format

A variety of libraries have been written to decode them, in addition
to the official Mac OS X APIs. One Python library is
[python-typedstream](https://causlayer.orgs.hk/dgelessus/python-typedstream).

Using "pytypedstream" (from `typedstream-0.0.1.dev0` in
python-typedstream), a ".chat" file from Apr 2005 can be decoded into:

    type b'@': NSArray, 8 elements:
    	NSString('AIM')
    	NSString('')
    	NSMutableArray, 2 elements:
    		object of class InstantMessage v0, extends NSObject v0, contents:
    			type b'@': object of class Presentity v0, extends Person v0, extends NSObject v0, contents:
    				type b'@': NSString('AIM')
    				type b'@': NSString('myusername')
    			type b'@': <NSDate: 2005-04-18 00:22:24.936263+00:00>
    			type b'@': object of class NSAttributedString v0, extends NSObject v0, contents:
    				type b'@': NSString('hey')
    				group:
    					type b'i': 1
    					type b'I': 3
    				type b'@': NSDictionary, 3 entries:
    					NSString('NSColor'): <NSColor CALIBRATED_RGBA: 0.0, 0.0, 0.0, 1.0>
    					NSString('NSBackgroundColor'): <NSColor CALIBRATED_RGBA: 0.6745098233222961, 0.7098039388656616, 0.7490196228027344, 1.0>
    					NSString('NSFont'): NSFont(name='Verdana', size=10.0, flags_unknown=(0x00, 0x01, 0x00, 0x00))
    			type b'I': 5
    		object of class InstantMessage v0, extends NSObject v0, contents:
    			type b'@': None
    			type b'@': <NSDate: 2005-04-18 00:30:36.754579+00:00>
    			type b'@': object of class NSAttributedString v0, extends NSObject v0, contents:
    				type b'@': NSString('You left the chat.')
    				group:
    					type b'i': 1
    					type b'I': 18
    				type b'@': NSDictionary, 1 entry:
    					NSString('NSFont'): NSFont(name='Verdana', size=11.0, flags_unknown=(0x00, 0x01, 0x00, 0x00))
    			type b'I': 1
    	NSMutableArray, 1 element:
    		object of class Presentity v0, extends Person v0, extends NSObject v0, contents:
    			type b'@': NSString('AIM')
    			type b'@': NSString('theirusername')
    	NSNumber, type b'c': 0
    	NSNumber, type b'i': 2
    	NSString('')
    	NSString('')

Note that NeXT-style TypedStreams are *also* used inside the newer
SQLite databases used by Apple's Messages.app (OS X) and Messages
(iOS), as one of the possible formats that the message-data BLOB can
be serialized with.


### Logorrhea

The OS X application Logorrhea, last updated (as of 2021) in 2006, was
created to parse and view iChat logs and is able to convert them out
of the binary formats: http://spiny.com/logorrhea/

The actual parsing logic, written in Objective C++ and taken from the
file "Chat.mm", appears to be:

    - (void) loadContents
    {
    	if (chatContents == nil)
    	{
    		NSData *chatLog = [[NSData alloc] initWithContentsOfMappedFile:myPath];
    		if ([myPath hasSuffix:@".ichat"]) // check for tiger-style chat transcript
    		{
    			NS_DURING
    				chatContents = [[NSKeyedUnarchiver unarchiveObjectWithData:chatLog] retain];
    			NS_HANDLER
    				NSLog(@"Caught exception from NSKeyedUnarchiver - %@", [localException reason]);
    				chatContents = nil;
    			NS_ENDHANDLER
    			[chatLog release];
    		}
    		else
    		{
    			NS_DURING
    				chatContents = [[NSUnarchiver unarchiveObjectWithData:chatLog] retain];
    			NS_HANDLER
    				NSLog(@"Caught exception from NSUnarchiver - %@", [localException reason]);
    				chatContents = nil;
    			NS_ENDHANDLER
    			[chatLog release];
    		}
		
    		if (![chatContents isKindOfClass:[NSArray class]])
    		{
    			[chatContents release];
    			chatContents = nil;
    		}
    
    		if (chatContents != nil)
    		{
    			for (unsigned int i=0; i < [chatContents count]; i++)
    			{
    				id obj = [chatContents objectAtIndex:i];
    				if ([obj isKindOfClass:[NSArray class]])
    				{
    					instantMessages = [obj retain];
    					break;
    				}
    			}
    		}
    	}
    }

Since this uses the Mac OS X APIs to deserialize the TypedStreams,
it's not especially useful on any other platform.


### iChat to Adium ULF Converter

An iChat log converter was supplied with the third-party Adium
multi-protocol IM client, in order to ease the transition for iChat
users. (As a result, anyone who migrated from iChat to Adium in the
early 2000s probably already has their iChat logs migrated to, and
stored with, their Adium logs as ULF XML files.)

From the file [InstantMessage.m](https://github.com/adium/adium/blob/master/Source/Logorrhea/InstantMessage.m):

    - (id)initWithCoder:(NSCoder *)decoder
    {
    	if ([decoder allowsKeyedCoding])
    	{
    		sender = [[decoder decodeObjectForKey:@"Sender"] retain];
    		text = [[decoder decodeObjectForKey:@"MessageText"] retain];
    		date = [[decoder decodeObjectForKey:@"Time"] retain];
    		flags = [decoder decodeInt32ForKey:@"Flags"];
    	}
    	else
    	{
    		sender = [[decoder decodeObject] retain];
    		date = [[decoder decodeObject] retain];
    		text = [[decoder decodeObject] retain];
    		[decoder decodeValueOfObjCType:@encode(unsigned) at:&flags];
    	}
    
    	return self;
    }

Again, this depends on the Mac OS X API, so it's not useful for
general-purpose log conversion or forensic analysis on other platforms.


## Apple Messages.app (OS X)

Messages is the modern OS X descendent of iChat.app, designed
primarily for Apple's proprietary iMessage service (including SMS
messages sent via an iPhone).  At one point it also supported Jabber.

### Storage Locations

Messages.app on OS X stores message history in two places:

* An SQLite database named `chat.db`, stored in `~/Library/Messages/`
* Individual `.ichat` files stored (at least with Messages v11.0, Mac
  OS 10.13.6) in
  `~/Library/Containers/com.apple.iChat/Data/Library/Messages/Archive/[YYYY-MM-DD]/`,
  where `[YYYY-MM-DD]` are a series of folders named by date in that
  format.

Explanations seem to vary as to why messages end up logged in one
place versus the other.  It *seems* to be the case that the SQLite
database is what's synchronized when using iCloud sync for Messages,
so this is presumably the canonical message repository.

The original author of this document noted:

> This raises the question of why, then, do the ".ichat" files inside
> the com.apple.ichat hierarchy exist at all?  I have yet to find an
> adequate answer.  But at least on my Mac OS 10.13.6 machine, the
> directories seem to contain *all* conversations (not just closed
> conversations, or local conversations) from mid-2018 to the present
> day.

Newer versions of Messages.app seem to have moved away from the
individual-file logs entirely, still use the same Binary Plist format
described below in some places (as BLOBs) within the database.


### Binary PList File Format

Despite the file extension of ".ichat", Messages.app uses a different file
format for its logs than iChat.app does.  

Unlike iChat's logs, which use NeXT "typedstreams" serialization,
Messages' ".ichat" files use the Apple-created Binary Property List
(BPList) format.

An easy test to determine whether a file uses the old or new format,
if you have a Mac OS X system available, is to attempt to parse it
with the Apple-supplied `plutil` program.  Parsing an old textstream
log file will yield:

    $ plutil -convert xml1 testaccount.chat
    testaccount.chat: Property List error: Unexpected character  at line 1 / JSON error: JSON text did not start with array or object and option to allow fragments not set.

In contrast, parsing a newer BPList log will result in no output, and
the *in-place translation* of the binary PList file into an XML-based
PList.  (You may notice the file suddenly increase in size in the
Finder, due to the XML overhead.)

Files using the more-modern `NSKeyedUnarchiver` format should be
parsable with Python, using
https://pypi.org/project/NSKeyedUnArchiver/.


### SQLite Chat Database

Recent versions of Messages.app have moved to a centralized SQLite
database for message history, presumably for ease in cross-device
synchronization via iCloud.

The preferred location the database on Mac OS X systems appears to be
`~/Library/Messages/chat.db`.

Although officially undocumented, the information-forensics community
has studied the schema of this database quite thoroughly.  It seems to
have changed over time, although usually in backwards-compatible ways.

If you want, [you can load a `chat.db` file into PANDAS with Python 3][chatdbpandas]
and poke at it. 

[chatdbpandas]: https://towardsdatascience.com/heres-how-you-can-access-your-entire-imessage-history-on-your-mac-f8878276c6e9

One frequent "gotcha" is that Apple changed the way dates are stored
in the database at least two times:

* In Mac OS X versions before High Sierra (v. 10.13, released
  September 2017), the date column is an seconds-since-epoch but,
  unlike the Unix standard of counting the seconds from 1970–01–01, it
  is counting the seconds from 2001–01–01.
* Beginning in Mac OS X High Sierra, it’s the same thing but the date
  format is now *much* more granular: it's a count of nanoseconds
  since 2001-01-01. So now we need to divide by 1,000,000,000.


## iOS Messages (iOS)

The iOS "Messages" app seems similar in its chat/message logging
behavior to the OS X "Messages.app", likely because the two are kept
in sync via Apple's iCloud service, and it's easier for them to use
the same (or at least very closely related) database schemas than have
to convert back and forth on each sync.

Most tools designed to parse the OS X `chat.db` database, also tend to
be able to parse the iOS `sms.db` database, although getting a copy of
the `sms.db` file from an iOS device generally requires making a
backup of the device to a Mac, then extracting the database file from
it.  (Some tools are available that will parse an iOS device backup
*directly*, but very few appear capable of grabbing it directly from
the device without a backup being made first.)


## References

* Github User "Kadin2048", "[Instant Messaging Client Log
  Formats](https://gist.github.com/kadin2048/ffe811e56c8e8fb6ceb8bade09439341)"
  (Gist, dated 2021-11-03) — primary source for this page.
* Github User "Yortos",
  "[imessage-analysis v3.3](https://github.com/yortos/imessage-analysis)"
  (Github Repository, retrieved 2026-03-27) — information on modern
  iMessage/Messages.app database schema.
