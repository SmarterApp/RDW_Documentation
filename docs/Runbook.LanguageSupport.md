## Runbook - Language Support

**Intended Audience**: This document contains information for configuring language support for the UI applications and generated reports in the [Reporting Data Warehouse](../README.md) (RDW). Additional information is available in the main [Runbook](Runbook.md). Operations and system administration may find it useful.

### Language Installation
By default all UI and report-processor applications ship with English as an available, embedded language. Installing additional languages requires both configuration changes and translation message files to be made available to the applications.

#### Reporting Webapp UI Language Installation
Adding an available language to the reporting webapp UI involves two steps:
1. Provide a translation JSON file containing the additional language's webapp translations.  
The English source of the messages required can be found [here](https://github.com/SmarterApp/RDW_Reporting/blob/develop/webapp/src/main/webapp/src/assets/i18n/en.json).
A translated version for a particular language should be named using the ISO-standard two-character language code and be placed in the external configuration repository at the location configured below.
(Example: `es.json` for Spanish language translations.)
2. Configuring the application to announce the both the presence and location of additional languages.
    * The `app.ui-languages` property contains a list of languages that are available to the system in addition to English.
    Additional languages should be specified using their ISO-standard two-character language code.
    * The `tenant.translation-location` property contains the URL-prefix used to retrieve advertised language files.
    * The example below registers Spanish as an additional UI language and specifies that `es.json` can be found in the
    configuration repository at `/i18n/es.json` on the `master` branch.
    
**rdw-reporting-webapp.yml**
```yaml
reporting:
  # en is always available as an embedded language
  ui-languages:
    - es

  translation-location: "binary-${spring.cloud.config.uri}/*/*/master/i18n/"
```

#### Report Processor Language Installation
Adding an available language to use when generating PDF reports is slightly different from adding a UI language above, but has the same requirements of providing a translation file and configuring the webapp UI to announce a languages availability:
1. Provide a translation .properties file containing the additional language's report translations.
   The English source of the messages required can be found [here.](https://github.com/SmarterApp/RDW_Reporting/blob/develop/common/src/main/resources/messages.properties)
   A translated version for a particular language should be named using the ISO-standard two-character language code and be placed in the external configuration repository at the location configured below.
   (Example: `messages_es.properties` for Spanish language translations.)
2. Configuring the webapp UI application to announce the presence of an additional report-processor language.
    * The `app.report-languages` property contains a list of languages that are available for PDF reports in addition to English.
    * The example below registers Spanish as an additional report-processor language

**rdw-reporting-webapp.yml**
```yaml
reporting:
  # en is always available as an embedded language
  report-languages:
    - es
```
3. Configuring the report-processor application to configure the location of additional translation properties files.
    * The `tenant.translation-location` property contains the URL-prefix used to retrieve advertised language files.
    * The example below specifies that Spanish translation properties can be found in the configuration 
    repository at `/i18n/messages_es.properties` on the `master` branch.

**rdw-reporting-report-processor.yml**
```yaml
reporting:
  translation-location: "binary-${spring.cloud.config.uri}/*/*/master/i18n/"
```
