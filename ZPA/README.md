# POC implementation of ZScaler Private Access using Azure Monitor Agent (From Feb 2024)

This implementation has rough edges, but gets to the point of Log Analytics ingestion of ZPA events in a(n over-) schematized table, with excellent performance, while allowing reuse of existing content.

Primary instruction doc is [here](ZScaler%20Private%20Access%20-%20Build%20Process.md).

## Future

This is an interim solution to solve a performance problem at large scale with the MMA-based one-column version which required query-time parsing of every message. (Scaled really well until it suddenly didn't)

An ASIM-compatible revision of the ZPA solution *is* expected in the future. (This solution could similarly be upgraded to ASIM but that's not in scope for this noodle.)

## Table Design

I took a naive approach and simply converted every data type I could see to a column. There were scripts and spreadsheets involved in doing this which I won't share here - maybe in another solution.

Table: `ZPA_Extended_CL` replaces `ZPA_CL` from the official MMA-based solution. ZPAExtended_CL has many (over a hundred) defined columns, whereas ZPA_CL had only TimeGenerated and RawMessage (or similar), 

Function: `ZPAExtended` is designed to plug into or replace `ZPAEvent`, so that the existing rules/detections are mapped to the new schema.


## Differences in detection

Some schema changes allow detections to be more effective than before (bugs?) so alert volume may increase initially. Watch for that.
