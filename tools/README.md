## Tools

### Subject Info Workbook

The subject definition workbook, `SubjectDef.xlsm` facilitates the collection of subject metadata needed to generate the subject XML file.
The subject XML file is loaded into RDW, see [Subject Configuration](../docs/Runbook.SystemConfiguration.md#subjects).

To define a new subject, make a copy of the workbook; the convention is to name the file using the subject code.
For example, if the new subject is Physics, make a copy called `SubjectDef.Physics.xlsm`.
Open the newly created file using a recent (>2016) version of Excel; be sure to enable macros when it asks.
The workbook has inline comments and notes to guide the user. Follow the directions, collect the data.
Once complete, press the XML button to copy the generated XML to the clipboard. Use your favorite XML editor to save it.

#### Contributing

* To edit the workbook, open it in Excel and open the VBA editor (Tools, Macro, Visual Basic Editor).
* The worksheets are protected, they must be unprotected to edit (Review, Unprotect Sheet). When you're done working turn protection back on allowing only:
    * `Select locked cells`
    * `Select unlocked cells`
    * `Format rows` - so the macro can show/hide rows
* In some of the worksheets, rows are conditionally hidden until specific data is
entered into the visible cells. To edit these rows in development, first unhide them
by selecting the rows above and below the hidden rows, right-clicking, and selecting
`Unhide` from the context menu. After making edits, remember to hide the rows that
were previously hidden.
* The palette of colors is defined in the Reference worksheet. The names of these
colors are used on several of the other worksheets to define drop down selectors.
The range for these color names is defined by a named range `colorlist` in the
Reference worksheet. If colors are added (or removed), this range needs to be
updated. Select `Formula` from the Excel Ribbon, and then select `Define Name`.
Select `colorlist` and adjust the range as needed. Note: only the Reference
worksheet needs to be unprotected for this operation. The worksheets that use the
range can remain protected.
![Range Edit Dialog](images/colorlist_range_edit.png)
* Changes to the template workbook may need to be repeated in copies.
* Things to work on:
    * Improve instructions.
    * ReportGrades. Only Summative overall text is collected because no subjects other than the predefined Smarter Balanced ELA and Math support the printed reports (ISRs).
        * Improve usability.
        * Extend to include claim text.
        * Extend to include ICA and IAB. 


### Assessment Package Files

To load specific assessment packages into RDW, they must be put into a CSV in Tabulator output format.
There are two general field sets for the tabulator output:
* Full. This is the full field set which specifies all the details for all the items in all the packages. This is the required data for Smarter Balanced assessments that take advantage of all the features of RDW.
* Simple (itemless). Some subjects are summative-only and do not need to specify item details. For these subjects a much smaller set of data is needed.
Some fields are optional based on the scoring requirements of a subject.

If item details are specified, the assessment fields must be repeated in every item row.


### Validator

The validator is a command-line utility for validating subject and assessment package files.
To use:
```
$ java -jar rdw-ingest-validator-2.0.0-RELEASE.jar 
Specify at least one subject (-s) or test package (-t) file
usage: Validator
Validator for RDW subject and test package files
 -s,--subject <arg>        subject file
 -t,--test-package <arg>   test package (tabulator) file
You may repeat options multiple times
```
NOTE: this requires java 1.8 or higher.
TODO - Currently the JAR artifact is manually copied into this folder, it should be published in a more automated way. 
