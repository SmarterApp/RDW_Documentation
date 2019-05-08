## Tools

### Subject Info Workbook

The subject info workbook, `SubjectInfo.xlsm` facilitates the collection of subject metadata needed to generate the subject XML file.
The subject XML file is loaded into RDW, see [Subject Configuration](../docs/Runbook.SystemConfiguration.md#subjects).

To define a new subject, make a copy of the workbook; the convention is to name the file using the subject code.
For example, if the new subject is Physics, make a copy called `SubjectInfo.Physics.xlsm`.
Open the newly created file using a recent (>2016) version of Excel; be sure to enable macros when it asks.
The workbook has inline comments and notes to guide the user. Follow the directions, collect the data.
Once complete, press the XML button to copy the generated XML to the clipboard. Use your favorite XML editor to save it.

#### Contributing

* To edit the workbook, open it in Excel and open the VBA editor (Tools, Macro, Visual Basic Editor).
* The worksheets are protected, they must be unprotected to edit (Review, Unprotect Sheet). When you're done working turn protection back on allowing only:
    * `Select locked cells`
    * `Select unlocked cells`
    * `Format rows` - so the macro can show/hide rows
* Changes to the template workbook may need to be repeated in copies.
* Things to work on:
    * ( ) Improve instructions.
    * ( ) Collect and emit ReportGrades data. This has not been done because no subjects other than the predefined Smarter Balanced ELA and Math support the printed reports (ISRs).
