# WGMS report files

WGMS Retrieval writes a CSV for each mission and report with retrieved records. A report with no records does not get an empty CSV. The columns come from the fields in WGMS records, so they can differ between reports.

| File | How it is created | What to know |
| --- | --- | --- |
| Current-mission report CSV | Written during active-mission retrieval. | It can gain records after later requests or rechecks. When the mission is finalized, the file moves to past-mission output. |
| Past-mission report CSV | Moved there by active-mission finalization or placed there by an explicit completed-mission rebuild publication. | It represents output for a completed mission, but its presence alone does not establish that the measurements are ready for analysis. |
| Rebuild candidate CSV | Created in a separate workspace after all of the rebuild's planned requests succeed. | It can be shared for review. It does not change past-mission output unless separately published. |

## Repeated and changed records

The same WGMS record can appear in more than one request. The extractor keeps one CSV row for each distinct full record within a mission and report, even if an identical record was returned again with its JSON fields in a different order. If a value changes, the record is distinct. Active-mission output can therefore retain an older row alongside a changed version; it does not identify corrections or remove records that WGMS stops returning.

A completed-mission rebuild uses separate state to create new candidate files. These can be compared with existing past-mission files before anyone chooses to publish them.

## Quality and availability

The extractor validates the structure of WGMS responses, but it does not assess the scientific quality of measurements. It requests mission time windows without independently checking each returned record's timestamp against those bounds. Check the fields and, where present, timestamps in a particular file before using it.

Some report files are routed to a private output area. A generated CSV is not, by itself, evidence that the file is publicly accessible.
