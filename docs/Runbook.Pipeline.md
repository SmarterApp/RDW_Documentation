## Pipeline

**Intended Audience**: This document contains information on how the Pipeline works in the [Reporting Data Warehouse](../README.md) (RDW). Operations and system administration will find it useful.

### Table of Contents

* [Purpose of the Pipeline](#purpose-of-the-pipeline)
* [Script Infrastructure ](#script-infrastructure)
* [Security Regime](#security-regime)
* [System Scripts](#system-scripts)
* [Files and Data Stores](#files-and-data-stores)
* [Pipeline Development Tasks](#pipeline-development-tasks)
* [General Notes on the Pipeline User Interface](#general-notes-on-the-pipeline-user-interface)
* [Diagnosing Problems](#diagnosing-problems)
* [ETS/CDE Exam Script](#ets-script)


### Purpose of the Pipeline

In the RDW Ingest process, documents of various types are submitted to the system, processed,
and stored. In some cases, especially when documents are submitted by third parties, they
can have formatting idiosyncrasies that must be handled in order for the regular processing
of the documents to be successful. The requirements for this special handling can evolve over
time, and so to avoid the need for a new application release each time this handling 
needs to be updated, the Pipeline functionality was introduced. 

The Pipeline allows certain administrators to create and update custom scripts to pre-process the 
incoming documents before they are given to the normal ingestion process. 
These pipeline scripts can be deployed into the live production system to immediately
change how incoming documents are handled. There is a user interface, restricted to these
privilege users, which allows the scripts to be created, edited, tested, versioned, and
deployed to the live system. 

### Script Infrastructure  

Pipeline scripts are currently only available for two of the ingest processors, though this
might be extended in the future. These two are the Exam Processor, which processes
TDT XML files, and the Assessment Package processor, which processes assessment
package CSV files.  

Scripts are written in the Groovy programing language, but are provided with an implicitly
included base script that provides many helper functions, most of which are targeted at
making XML and CSV transformations easy to implement. These helper methods are explained
in detail in the RDW Administrator Guide. In addition, all the capabilities of
the Groovy language are available to be used in the Pipeline scripts. However, this is subject to 
the restrictions of the security regime the scripts run in. This security regime will
be discussed in the following section.

### Security Regime

Pipeline scripts are a very powerful and flexible way to modify the ingest processes that
support them. This power also makes them dangerous, and for that reason, they run in a
restrictive security regime. The regime is pre-configured and currently not available for system
administrators to modify. 

The scripts are allowed to:
- read files from the file system
- import classes (Java and Groovy) that exist in the class path
- parse and format XML and CSV files
- manipulate text, including the use of regular expressions
- output messages to a logging system

The scripts are not allowed to:
- write files to the file system
- create or listen for network connections
- connect to a database
- create new execution threads
- disable the security regime

### System Scripts

The scripts described in previous sections can be called **User Scripts**, because they
are developed by the users of the system, albeit specially privileged ones. In addition to the
Scripts, each pipeline can optionally have one or two **System Scripts**, which are generally 
created by the development team, but could potentially be created or modified by system
administrators. One of these would run before the User Script and the other after, 
so they are called the Pre and Post System Scripts. Typically, these scripts would be used to 
run validation logic on the ingested documents. However, it is possible to have them do 
transformations like the User Scripts do.

Unlike User Scripts, System Scripts do not have a user interface to develop, nor are
they versioned. The system scripts, if they exist, are loaded from the module's resources.
They always run regardless if a User Script has been deployed or not. System Scripts
do not run in the Security Regime that the User Scripts run in. Thus, System Scripts
can access the database and perform any other functions denied to User Scripts. 


### Files and Data Stores

This section will provide a deep dive into where Pipeline code and configuration information
is stored.

#### Files

The System Scripts are loaded from classpath of the relevant ingest processor and have
following naming convention:
```
/scripts/pipelines/<pipeline code>/pre-process.groovy
/scripts/pipelines/<pipeline code>/post-process.groovy
```
... where the pipeline code is `exam` for the Exam Processor Pipeline and `assessment` for the
Assessment Package Processor.

The User Scripts are loaded from the archive service's S3 bucket, with the naming convention:
```
/pipelines/<pipeline code>/user.<version>.groovy
```
 ... where pipeline code is as described above and version is number that increments each
 time a new User Script is published for the Pipeline. All versions of the User Script are kept
 to enable rolling the Pipeline back to a previous version. Existing versions of User Scripts
 cannot be changed through the user interface, but new versions can be published.
 
 Several properties are stored as metadata with the User Scripts stored in S3. They are:
 - pipeline code (exam or assessment)
 - script version
 - username of user who published this version
 - timestamp of the publishing event
 
#### Database

Some additional information about pipelines in stored in the RDW warehouse schema.

The pipeline table contains:
- id -- internal id for pipeline
- code -- pipeline code as described above
- input_type -- type of document the pipeline processes, currently 'xml' or 'csv'
- active_version -- script version currently active for the ingest process, or null for no script

The pipeline_script table contains:
- id -- internal id 
- pipeline_id -- foreign key to pipeline table
- body -- the last saved work-in-progress User Script for this pipeline
- created -- timestamp of the first script save
- updated -- timestamp of the most recent script save
- updated_by -- username of the last user to save changes to the script

The pipeline_test table contains:
- id -- internal id
- pipeline_id -- foreign key to pipeline table
- name -- display name for the test
- example_input -- full text of the document to process when run the test
- expected_output -- full text expected to result from running the script with the example_input
- created -- timestamp of the first save of this test
- updated -- timestamp of the most recent save of this test
- updated_by -- username of the last user to save changes to this test
 
### Pipeline Development Tasks

This section will describe the effects of various Pipeline development tasks on the 
file and data stores described in the previous section. For detail on how to perform
these task, refer to the RDW Administrator Guide.

##### Landing Page
Upon entering the Pipeline User interface and choosing a Pipeline type (Exams or Test Packages), 
the user is taken to a minimal integrated development environment (IDE). It will be displaying the most 
recent work-in-progress script, which is loaded from the from the `pipeline-script` table, 
and a list of existing tests, which is loaded from the `pipeline-test` table. 

If no script has been developed yet for the pipeline, the script window will display a placeholder script
appropriate for the pipeline type, which demonstrates how a real script could be developed. 
The list of tests will be empty.

##### Edit Script
As the script code is edited, it will be automatically compiled every few seconds, and any errors
marked by a red X icon before the line numbers where they are found. Hovering on the icon, will show
the error message. However, the Groovy language is weakly-typed and allows runtime "meta-programing", 
so many situations that would cause compilation errors in other languages will not be flagged by the
Groovy compiler.   

##### Save Script
The script can be saved at any time, whether or not it compiles. This writes the record out to the
`pipeline_script`. It does not publish the Pipeline or activate the new script. Any changes not saved
before the browser session is closed are lost. If the user has unsaved changes and tries to navigate 
away from the IDE page, a warning dialog will provide a second chance to save them.

##### Create Test/Edit Test
To create a new test, select the New Test button. To edit an existing test, select the tile of that test
in the left navigation bar. In either case, the code edit panel is replaced with a form for entering a
description of the test, the test input, and the expected output after the script is run. 

As mentioned previously, the flexibility of the Groovy language also, unfortunately, means some common problems will
not be caught by the compiler. This makes it imperative to maintain a comprehensive set of tests. The
Pipeline UI will not allow a Pipeline to be published until there is at least one test, and all the existing
tests pass.

##### Save Test
Saving the test writes it out to the `pipeline_test` table, including the full text of the test input
and the expected output. As with scripts, any changes not saved at the close of the browser session will be lost.

##### Delete Test
Hover over the test in the left navigation panel will cause a trashcan icon to appear. Selecting this icon,
and then confirming the deletion in the resulting dialog, will cause the `pipeline_test` record to be deleted.
This action cannot be undone.

##### Run Test
Both the test and the script must be saved before the test can be run. If the test passes, then a message
will appear saying that it did. If it fails, then the edit panel will be replaced with the expected output and
the actual output, with differences highlighted. 

##### Run All Tests
From the landing page view, selecting Run Tests button will cause all the tests to be run. The left navigation
panel will summarize which tests passed and which failed, and allow view the various difference reports.

##### Publish Pipeline
Selecting the Publish button from the landing page will cause the Pipeline script to be published.
This writes the script to the s3 archive as well as metadata about it. (Refer to the 
[Files and Data Stores](#files-and-data-stores) section for details about this file.) 

However, before the Publish button is enabled, all changes must be saved, the script must compile without errors, 
and there must be at least one test. In addition, if any tests do not pass, the script will not be published, and the list of
passing and failing tests will be displayed instead. This is the same as what happens with the Run Tests button
is selected.

##### Publishing History
Selecting the ellipsis button next the to the Publish button, and then selecting Publishing History, brings up
the Publishing History views. This displays all the published versions of the Pipeline script and allows viewing,
but not editing, each version of the script. The currently active version, i.e., the one actually being used 
in the ingest process, is indicated in the list of published versions.

The information about the published scripts comes directly from the metadata of the files stored in the s3
archive and the script bodies come from the files themselves. Finally, the active version is found in the `pipeline`
table based on the Pipeline type.

##### Activate Pipeline Version
Activating a Pipeline version can be done during the publish activity. It can also be done by selecting the
version to be activated in the publishing history and selecting the Activate button. In either case, this action
updates the record in the `pipeline` table.

The ingest processor listens for changes in the active version and so within some minutes will begin to use
the new script version to process incoming documents. It is therefore extremely important not to activate 
a new script version, especially on the production system, until it has been fully vetted.

##### Deactivate Pipeline Version
Selecting the currently active version from the publishing history will cause the Activate button to change
to Deactivate. Selecting this Deactivate button will then deactivate the Pipeline altogether. That is, there
will be no active version and so no transformations will be done during the ingest process. 
This action will update the `pipeline` record for the Pipeline, changing its active version field to null.

The ingest processor's listener will also detect a deactivation, and so within in a few minutes will begin
running without a User Script.

### General Notes on the Pipeline User Interface

The user interface does not provide any special handling for concurrent edits in a
pipeline. If mulitple users simultaneously edit scripts or tests, the last to save will
overwrite the changes of the others. However, publishing a script does not cause the older one
to be overwritten. Each time the script is published, a new version is created in the s3 archive
and all the versions are kept indefinitely. There is no way through the user interface to purge
obsolete script versions. This would have to be done manually by a sysadmin with delete
privileges on the s3 bucket.

Also note that nothing in the user interface tasks affects the System Scripts, `pre-process.groovy`
and `post-process.groovy`. If these scripts are present, they will automatically be run
during the ingest process even if the Pipeline has been deactivated. Deactivating only
stop in User Script from running as part of the Pipeline.     

### Diagnosing Problems

During the development process, problems will appear in the user interface, and must be corrected before
the pipeline can be published. When the pipeline becomes part of the ingest process, any problems will
be sent to the log file for the particular service that holds the ingest processor. For example, the exam
processor has its own service, so its problems will appear in the exam-processor-deployment logs. The assessment
package processor is part of the package processor, so its problems will appear in the package-processor-deployment
logs.

The Pipeline development user interface does not allow a Pipeline to be published unless it compiles and
passes tests. However, the UI cannot guarantee that the tests are comprehensive, so the most likely problem to
see in the ingest process is a runtime error. 

Runtime errors appear in the logs as:

```
... WARN 8 --- [ EXAM.default-1] o.o.r.i.common.script.PipelineProcessor  : Pipeline script failure: Error in user.1.groovy at line 11: ArithmeticException: Division by zero

```

The file name `user.1.groovy` shows that the problem occurred in the User script, version 1. 
The best practice for correcting this problem would then be:
1. Add a test in the Pipeline development user interface that duplicates this problem
2. Fix the Pipeline script so that the test passes
3. Publish and activate the new pipeline
4. Reprocess the failed document


Other problem types will appear in a similar way in the logs, but should be rare or non-existent. For example a compilation
error would appear as: 
```
... WARN 34979 --- [ EXAM.default-1] o.o.r.i.common.script.PipelineProcessor  : Failed to compile Exam pipeline script: startup failed:
user.2.groovy: 4: expecting ''', found '\n' @ line 4, column 16.
   println('hello)
                  ^ 
```

A script with a compilation error should not have been published at all, so this would indicate that it had been
changed outside of the normal user interface.

If the currently active script cannot be loaded from the archive, this would appear in the logs as:

```
... WARN 36557 --- [ EXAM.default-2] o.o.r.i.common.script.PipelineProcessor  : Failed to load Exam pipeline: Unable to build pipeline. There is no published pipeline with code "exam" and version "3"
```  

When a document is processed by any ingest process, an entry is created in the `import` table. If there is a Pipeline
failure during the ingest, this record will have a status of -7 (PIPELINE_FAILURE), so these records will be easy
to find an reprocess once the problem with the Pipeline is corrected.

<a name="ets-script"></a>
### ETS/CDE Exam Script

Starting in school year 2017-18 there were inconsistencies in TRT data provided by ETS for CDE. Making the changes in ETS was not feasible so an XSLT solution was added to RDW. The pipeline functionality replaces this XSLT solution, and this script is a drop-in replacement:
```groovy
// Enable XML extensions to simplify processing the XML document
enable 'xml'

// This rule removes leading 10 from item bank key
transform '//Item' by { item ->
    if (item.bankKey.startsWith('10')) {
        item.bankKey = item.bankKey.substring(2)
    }
}

transform '//Response' by { response ->
    def text = response.text

    if (text.contains('choiceInteraction_1') && text.contains('choiceInteraction_2')) {
        // This rule converts EBSR multiple-choice and multiple-select responses to the expected format:
        //    <itemResponse>
        //      <response id="EBSR1">
        //        <value>A</value>
        //      </response>
        //      <response id="EBSR2">
        //        <value>C</value>
        //      </response>
        //    </itemResponse>
        response.text = text
                .replaceAll(~/choiceInteraction_(\d).RESPONSE/, 'EBSR$1')
                .replaceAll(~/choiceInteraction_\d-choice-(\w)/, '$1')

    } else if (text.contains('choiceInteraction_1')) {
        // This rule converts IAT multiple-choice and multiple-select responses to the expected format ("A,C,D")
        def matches = text =~ /choiceInteraction_1-choice-(\w)/
        if (matches.count > 0) {
            response.text = matches.collect { it[1] }.join(',')
        }

    } else if (text.contains('matchInteraction')) {
        // This rule converts Match Interaction (MI) responses to the expected format:
        //    <itemResponse>
        //      <response id="RESPONSE">
        //        <value>1 a</value>
        //        <value>2 b</value>
        //        <value>3 a</value>
        //        <value>4 b</value>
        //      </response>
        //    </itemResponse>
        response.text = text
                .replaceAll(~/matchInteraction_\d.RESPONSE/, 'RESPONSE')
                .replaceAll(~/matchInteraction_\d-(\d)\W*matchInteraction_\d-(\w)/, '$1 $2')

    } else if (text.contains('hotTextInteraction_')) {
        //  This rule converts Hot Text Interaction (HTQ) responses to the expected format:
        //    <itemResponse>
        //      <response id="1">
        //        <value>2</value>
        //        <value>4</value>
        //      </response>
        //    </itemResponse>
        response.text = text
                .replaceAll(~/hotTextInteraction_(\d).RESPONSE/, '$1')
                .replaceAll(~/hotTextInteraction_\d-hottext-(\d)/, '$1')

    } else if ( text.contains('equationInteraction_') ||
                text.contains('tableInteraction_') ||
                text.contains('gridInteraction_') ||
                text.contains('textEntryInteraction_')) {
        // This rule extracts the content out of embedded value tags.
        response.text = text
                .replaceAll(~/(?s).+<value>(.+)<\/value>.+/, '$1')
                .unescapeHtmlTags()
    }
}

outputXml
```
NOTE: although this script was tested in development, it is critical that proper tests are used to validate the functionality.
