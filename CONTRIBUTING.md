# Contributing

**Inteneded Audience**: this document describes the conventions used in this RDW project.

## Contributing to Documentation
* Please keep documentation **consistent**.
* Know that many different users will read this documentation from developers to operations and system administrators to application support. So make it clear **what** is being documented and **who** may find it useful.
* **Diagrams**. Some of the diagrams included in the document use [draw.io](www.draw.io). The source documents are xml files. Please use the convention of naming the source xml file the same as the image (png) file. For example `cool-picture.xml` would be the draw.io source for `cool-picture.png`.

## TODO
Bored? Here's some stuff you can do ...

* Runbook. 
    * It needs cleaning up: it should be mostly configuration. Move monitoring into Monitoring.md and troubleshooting into Troubleshooting.md.
    * How to deal with "common" properties vs. "uncommon" settings?
    * Import Service chapter
    * Package Processor chapter
    * Exam Processor chapter
    * Migrate Reporting chapter - there is a lot of information here, it needs organization and separation (monitoring, troubleshooting)
    * Migrate OLAP chapter
    * Group Processor chapter
    * Task Service chapter
* Troubleshooting - perhaps two: flowchart-style "what's the problem", and a scenario-based (problem specific) guide 
* Reporting. All the work done so far has focused on ingest processes. We need to document the reporting app, admin app and related processes.
