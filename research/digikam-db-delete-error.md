---
layout: default
date: 2026-08-10
title: DigiKam Tag Deletion ERRNO 1451
---
# DigiKam Tag Deletion: ERRORNO 1451

Or, in which I spend a bunch of time and a bunch of my employer's Claude
credits chasing down an annoyance in DigiKam's tag list.

## Environment

- OS: "Pop_OS 22.04" (Ubuntu 22.04), AMD64 architecture
- DigiKam: v9.1.0 AppImage
- DB Schema: migration V16 -> V17 performed 2026-07-26 with DigiKam 9.1.0
- Database server: MariaDB 10.6.23, remote host
- Database name: `digikam` (all four DigiKam databases on the same server)
- Collection storage: NFS 4.0 share from a NAS

## Symptoms

Attempting to delete certain tags inside DigiKam fails: the tag
appears to be deleted, but reappears on the next launch of DigiKam.
In the database, the DELETE operation fails and the row is shown again
on the next restart, because it was never deleted in the first place.

This can be seen in the DigiKam debug log
(`QT_LOGGING_RULES=digikam*=true`):

```
Digikam::BdEngineBackendPrivate::debugOutputFailedQuery: Failure executing query:
 "DELETE FROM Tags WHERE id=?;"
Error messages: "QMYSQL: Unable to execute statement" "Cannot delete or update a parent row: a foreign key constraint fails (`digikam`.`ImageTagProperties`, CONSTRAINT `ImageTagProperties_Tags` FOREIGN KEY (`tagid`) REFERENCES `Tags` (`id`) ON DELETE CASCADE ON UPDATE CASCADE)" "1451" 2
Bound values:  QList(QVariant(int, 433))
Digikam::BdEngineBackend::execDBAction: Error while executing DBAction [ "DeleteTag" ] Statement [ "DELETE FROM Tags WHERE id=:tagID;" ]
```

## Diagnosis

After looking at the logs, it became clear that _only_ tags whose rows
referenced `ImageTagProperties` would fail to delete. (Some tags
referenced only `ImageTags` and `TagProperties` and deleted fine.  I'm
still unsure why some tags use one vs. the other.)  

All three foreign keys referencing `*Tags` are declared with cascading
updates and deletes:

```sql
-- information_schema.KEY_COLUMN_USAGE + REFERENTIAL_CONSTRAINTS
ImageTagProperties.tagid  ImageTagProperties_Tags  Tags(id)  ON DELETE CASCADE ON UPDATE CASCADE
TagProperties.tagid       TagProperties_Tags       Tags(id)  ON DELETE CASCADE ON UPDATE CASCADE
ImageTags.tagid           ImageTags_Tags           Tags(id)  ON DELETE CASCADE ON UPDATE CASCADE
```

The relevant `SHOW ENGINE INNODB STATUS` output:

```
2026-08-08 01:20:52 0x7ff3440de640 Transaction:
TRANSACTION 7882970, ACTIVE 1 sec updating or deleting
mysql tables in use 6, locked 6
...
DELETE FROM Tags WHERE id IN (429,430,431,432,433,617)
Foreign key constraint fails for table `digikam`.`Tags`:

CONSTRAINT `ImageTagProperties_Tags` FOREIGN KEY (`tagid`) REFERENCES `Tags` (`id`) ON DELETE CASCADE ON UPDATE CASCADE
...
But the referencing table `digikam`.`ImageTagProperties`
or its .ibd file or the required index does not currently exist!
```

This was admittedly a bit opaque to me, but it's the sort of thing my
friend Claude excels at:

> InnoDB's data dictionary believed the `ImageTagProperties` table (or
> its secondary index) does not exist, even though
> `information_schema.TABLES` reported it as an existing InnoDB
> table. The FK metadata/index state for that constraint was therefore
> broken; the server treated the cascading FK as non-cascading
> (really, as referencing a missing object) and returned `Cannot
> delete or update a parent row ... 1451`.

This seemed plausible enough, but it raised the question of where this
issue would have come from.  While I have done some shifty,
warranty-voiding stuff to my DigiKam installation in the past, I
hadn't done anything to _that_ part of the database.

Although it presents as a missing/incorrect FK, the source of the
problem _seems_ to be related to a DB schema change introduced in
v9.1.0.

### V16/V17 Schema Update

Next, I checked the MariaDB logs on my "enterprise" (aka: always-on PC
in the basement) database server, looking for when the
`ImageTagProperties` table had last been changed. And, lo and behold,
the log showed an `ALTER TABLE` warning restricted to
`ImageTagProperties`:

```
Jul 26 12:56:16 myserver mariadbd[1976]: 2026-07-26 12:56:16 6898 [Warning] InnoDB: In ALTER TABLE `digikam`.`ImageTagProperties` has or is referenced in foreign key constraints which are not compatible with the new table definition.
```

How interesting! The date, 2026-07-26, roughly matched my recollection
of installing the last DigiKam update. (DigiKam 9.1.0 was released
2026-06-07, so this at least doesn't involve any temporal-causality
violations.)

And the [release notes for 9.1.0][ra] do mention database work and a
V16 to V17 schema update:

> "This release focuses on database migration ..." "The database
> schema has been updated to support time zones ... Significant
> improvements have been made to MariaDB migration".

[ra]: https://www.digikam.org/news/2026-06-07-9.1.0_release_announcement/

This led me to [KDE commit `3b256067`][kdecom], which includes a
"database update to add Indexes and timezone field to V17". And a
quick search for table alterations led to the file
`dbconfig.xml.cmake.in` and  the line: `ALTER TABLE
ImageTagProperties MODIFY COLUMN property VARCHAR(255)…`
  
[kdecom]: https://invent.kde.org/graphics/digikam/-/commit/3b256067d8938838522c16ab7df05fe5c03f21c2

This would explain the warning in the logs from Jul 26 at 12:56:16.
The migration script changed `ImageTagProperties.property` to a
VARCHAR (from something else), MariaDB warned that this would break
some foreign key constraints but did it anyway, and now we're getting
FK errors.

### Related Issues
Not a complete list, but these popped on a quick "am I reinventing the
wheel?" search:

- [519390 DB: ImageTagProperties.property should be VARCHAR, not
  TEXT](https://bugs.kde.org/show_bug.cgi?id=519390) — schema change,
  RESOLVED/FIXED in 9.1.0.
- [514970 Create index on
  ImageTagProperties.property](https://bugs.kde.org/show_bug.cgi?id=514970)
  — index change, RESOLVED/FIXED in 9.1.0.
- [519443 UpdateSchemaFromV16ToV17 not implemented for
  QMYSQL/MariaDB](https://bugs.kde.org/show_bug.cgi?id=519443) —
  User reported this exact migration step failing on
  MySQL/MariaDB; upstream closed as user-environment.

## Proposed Solution
All of this is very nice, but what to do about it?

1. **Backup first.**

   ```bash
   mysqldump --single-transaction --routines --triggers --events \
     -h mydbserver -u digikam -p digikam > digikam-before-tag-repair.sql
   # 4.5 GB dump; DB size sample: 300,162 Images rows, 364,121 ImageTags rows
   ```

2. **Rebuild the broken foreign key.** The FK metadata is
   inconsistent, so drop and recreate it:

   ```sql
   ALTER TABLE ImageTagProperties DROP FOREIGN KEY ImageTagProperties_Tags;
   ALTER TABLE ImageTagProperties
     ADD CONSTRAINT ImageTagProperties_Tags
     FOREIGN KEY (tagid) REFERENCES Tags(id)
     ON DELETE CASCADE ON UPDATE CASCADE;
   ```

   Bonus point: Inspect `information_schema.REFERENTIAL_CONSTRAINTS`
   to confirm `DELETE_RULE=CASCADE`, `UPDATE_RULE=CASCADE`.

3. **Delete the affected tags.** Just blow them away:

   ```sql
   DELETE FROM Tags WHERE id IN (429,430,431,432,433,617);
   ```
   
   These values came from the original error messages, plus inspection
   of the DB to find a few others that I hadn't stumbled on.  Your
   values will, of course, be different.

4. **Verify:** the offending tags are gone, and their `ImageTags` rows
   (in my case, 144 rows) were removed automatically by the cascade.

## Recommendations
If you encounter magically un-deletable tags and think this might be
the issue:

- Check the server error log for the `InnoDB: In ALTER
  TABLE... ImageTagProperties has or is referenced in foreign key
  constraints which are not compatible with the new table definition`
  warning; if present, the FK metadata should probably be rebuilt as
  above.
- Verify `SHOW ENGINE INNODB STATUS` during a failing DELETE. Look for
  messages with "referencing table… does not currently exist" phrasing.
- If `ImageTagProperties_Tags` is intact, then the `1451` error is a
  real orphan-row problem (though one that shouldn't happen with
  cascading FKs) and probably requires drilling into the
  `ImageTagProperties`/`TagProperties`/`ImageTags` rows, looking for `tagid`
  not present in `Tags`.
- Don't confuse this with a separate (but symptomatically similar)
  issue where DigiKam _successfully_ deletes a tag from the DB, but then
  _reinserts_ it later during a metadata synchronization.  (I've run
  into that one too, and will write a note about it at a later date.)
  Ruling that out requires looking at the logs while attempting the
  delete.

## Caveats

The root-cause attribution (V16 to V17 migration) rests on a certain
amount of circumstantial evidence, mainly my recollection of the last
update date, the MariaDB error-log timestamps matching an
`ImageTagProperties` ALTER during the same period, and the release
notes pointing to commits from 9.1.0 that appeared to have the same
ALTERs.

If pressed, I'd have to back away from saying that it's _proven_,
because I didn't have query logging enabled on the DB, and I'm not
planning on going back and doing a thorough test.  It's possible,
perhaps, that something else I did in that same period of time screwed
up the database… but I can't think of what that would plausibly have
been.

The FK-consistency issue was directly observed, and the fix noted
above seems to have resolved the issue to my satisfaction (after
several days of heavy use, it hasn't reoccurred).
