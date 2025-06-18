# IIS DCR Transform Workbook

In which we learn how to build Workbooks, and some of the features and limitations (next time I'd probably try to do it in one page of KQL rather than multiple steps... there ya go).

# Wot It Does

This workbook helps construct an appropriate `transformKql` statement for use in a Custom Text Data Collection Rule.

The trick is, it doesn't target a custom text table schema, though, it targets 2 _official_ schemas!
- [W3CIISLog](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/w3ciislog) and
- [ASimWebSessionLogs](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/asimwebsessionlogs)

## Um why?

_"But, you idiot, these are solved problems!"_ you might exclaim, perhaps rudely? 

To which I would respond _"where's your X-Forwarded-For header, buddy? Where is it? **Anywhere**? No? Well, quiet now, aren't you?!"_ and things would probably devolve from there.

Two things I wanted to try to do here, and two things are done! Perhaps badly; we'll see.

1. Hack X-F-F into an existing W3CIISLog field to avoid having to use a custom log table for the whole thing
2. "Native" IIS ASIM log collection

I've assumed that DCR creation is a Solved Problem, and that the fiddly, hard, gnarly bit is working out the transformKql part.

# Setup

You will need:
- **#Fields:** A copy of "the" #Fields directive from the top of an IIS log
- **One Log:** Optionally but recommended, one (1) (0+1) sample log line demonstrating use of these fields
- A view on which table you're going to target
- An rough understanding of how to set up a custom text log ingestion DCR and apply the transform to it

Last bit is hardest and unfortunately not solved by this workbook, at least not now. (Note to self: post sample DCR template so at least prior art is visible).

## Using it

### Provide the #Fields and a log line

Just drop your #Fields line into the first parameter, and your log line into the next part. Examples are provided, just select-all and paste over them.

This will show you the Workbook query's interpretation of the fields and (critically!) the order of the fields for the log you just posted.

> Aside: Yes, IIS sites and apps can have different log settings. Ick. And this will/may/might once change the index of the items, which will try to put the wrong values into the wrong colums and - bluntly - that's your problem! All I've got for you are that you could:
> - standardize the log format 
> - or have one DCR per site/sub-site/log configuration.

> Aside aside: If AMA did flexible W3C field parsing to columns _including extended columns_, we wouldn't be having this little chat! And at this point, the transform is post-upload, so we're just sending blobs of data without a proper schema and then splitting them... Look, I know, it sounds dumb, but it doesn't even natively do CSV so I don't know what you're expecting from it. Oh and if you change your log format, you need to change the DCR as well, and there'll almost always be some nasty overlap where one of them is out of date. Fair warning.

### Output

You then get to pick your output table: `W3CIISLog` or `ASimWebSessionLogs`

- If you pick `W3CIISLog` (the traditional table) then you also get to pick which column in which to jam X-Forwarded-For (or un-set, un-set is OK, but... that's why you're here, right?) - with the `Description` and `Confidence` columns offered for your selection. 

> Snark: If you think there are better fields to misuse, go ahead and edit the code to add and use them!

- If you pick `ASimWebSessionLogs` it's mostly on rails.

### Select all the merged columns

You'll see a table with tickboxes combining the indexes with the output schema - tick the box at the top left to select everything. If you want to exclude something, you can probably do that too using the tickboxes.

Anything which doesn't have either an index or an output column will be ignored by the next step, so check you've got what you came for, and where it's going.

### Click the transformKql statement

Ticking the box regenerates the `transformKql` step on the right, so click that next to (phew) run the query against the log sample and show you an output sample.

If this looks correct, wahey! You can copy the `transformKql` from the bottom panel for use in your DCR.

# Editing a DCR

This part seemed harder/more fiddly than I naively expected.

- We know we want custom text input
- We know we want to target an actual schematized table for output

So it should be just a matter of changing the `Custom-YourName_CL` output stream to be e.g. `Microsoft-ASimWebSessionLogs` and dropping in the `transformKql` in the same section.

But you might only be able to do that easily via a template as a starting point. 

(I tried the _Custom Text Logs via AMA_ connector for a custom log, and it created a table for me (aww) which I instantly wanted to ignore, so used [Azure Regedit](https://tristank.substack.com/i/160556249/how-to-azure-regedit) to edit the DCR as above... Now I have an empty `ASimWebSessionLogs_CL` table floating around which I have to delete, but the DCR does work fine to the real ASimWebSessionLogs table... At least so far...)

We'll see!...

# One last note on X-F-F
`x-forwarded-for` is often _experienced_ as a single IP address, but _defined_ as a comma-separated list, so just watch for that assumption. It's not safe to simply push x-f-f into cIP unprocessed... so that's left as an exercise for Query time.
