## Runbook - Language Support

**Intended Audience**: This document contains information for configuring language support for the UI applications and generated reports in the [Reporting Data Warehouse](../README.md) (RDW). Additional information is available in the main [Runbook](Runbook.md). Operations and system administration may find it useful.

### Language Installation
By default all reporting applications ship with English as an available, embedded language. Installing additional languages or overriding English display values requires both configuration changes and translation message files to be made available to the applications.

#### Reporting Language Files
Language translation values are represented as JSON files.  Each language is represented by a single `xx.json` file that is named using the language two-character ISO code. (Example: `es.json` for Spanish, `vi.json` for Vietnamese, etc)
Additionally, although the application ships with English as a default embedded language, tenants may install an `en.json` file to override any display text in the reporting application.
The language files must exist in a location that is accessible by the reporting services.  We recommend using the configuration repository as a simple accessible hosting location that also provides change tracking.

#### Reporting Language File Creation and Installation
To create a new language JSON file, it is easiest to start from the existing English values, translate them, and save the result as a new language JSON file.
To retrieve the current translations for a given language, log into the reporting application and make a call to `https://my-reporting-application/api/translations/{xx}?include-ui=true` to download the full language source for a language identified by its two-character code. (Example: `en` for English)<br>
The `?include-ui=true` parameter includes translation messages that only apply to the UI and may be omitted if installing a language that only applies to generated reports (PDF, Aggregate, etc).<br>
The downloaded JSON may then be translated into any language and saved to the configuration repository as a xx.json file. (Example: `/i18n/es.json`)<br>
See below for how to configure the reporting system to use your translated JSON file to provide translation options to the user.<br>

NOTE: A translation JSON file is not required to be "complete." For example, to override just the footer text in English from the default value, you may install an `en.json` file that contains only the new footer message:

**en.json**
```json
{
  "common-ngx": {
    "footer": "© My Organization – Smarter Balanced Assessment Consortium"
  }
}
```

#### Reporting Webapp UI Language Installation
Adding an available language to the reporting webapp UI involves configuring the application to announce both the presence and location of additional languages.
* The `reporting.ui-languages` property contains a list of languages that are available to the system in addition to English.
  Additional languages should be specified using their ISO-standard two-character language code.
* The `reporting.translation-location` property contains the URL-prefix used to retrieve advertised language files.
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
Adding an available language to use when generating reports is slightly different from adding a UI language above, but has the same requirements of providing a translation file and configuring the webapp UI to announce a languages availability.
1. Configuring the webapp UI application to announce the presence of an additional report-processor language.
    * The `reporting.report-languages` property contains a list of languages that are available for reports in addition to English.
    * The example below registers Spanish as an additional report-processor language

**rdw-reporting-webapp.yml**
```yaml
reporting:
  # en is always available as an embedded language
  report-languages:
    - es
```
2. Configuring the report-processor application to configure the location of additional translation properties files.
    * The `reporting.translation-location` property contains the URL-prefix used to retrieve advertised language files.
    * The example below specifies that Spanish translation values can be found in the configuration 
    repository at `/i18n/es.json` on the `master` branch.

**rdw-reporting-report-processor.yml**
```yaml
reporting:
  translation-location: "binary-${spring.cloud.config.uri}/*/*/master/i18n/"
```

#### Language Cross-References
When installing a language for the UI and/or generated reports, you must include a display name for that language in at least English and ideally all other installed languages.<br>
For the purposes of this example, we will assume that both Spanish and Vietnamese are installed as both UI and generated report languages.

1. Add UI language display names
    * When displaying UI languages to the user in the selector they should be displayed in the target language, since they are designed to be selected by a speaker of that language.
    * Install/modify an `en.json` translation JSON that includes the display values for Spanish and Vietnamese *in* Spanish and Vietnamese:

**en.json**
```json
{
  "common-ngx": {
    "languages": {
      "es": "Español",
      "vi": "Tiếng Việt"
    }
  }
}
```

2. Add generated report language display names
    * When displaying generated report languages to the user in the selector they should be displayed in the current UI language, since they are designed to generate reports for distribution to a speaker of the target language.  The UI user may not speak the target report language.
    * Install/modify an `en.json` translation JSON that includes the display values in English.
    * Install/modify an `es.json` translation JSON that includes the display values in Spanish.
    * Install/modify a `vi.json` translation JSON that includes the display values in Vietnamese.

**en.json**
```json
{
  "report-download": {
    "form": {
      "language-option": {
        "es": "Spanish",
        "vi": "Vietnamese"
      }
    }
  }
}
```

**es.json**
```json
{
  "report-download": {
    "form": {
      "language-option": {
        "en": "Inglés",
        "es": "Español",
        "vi": "Vietnamita"
      }
    }
  }
}
```

**vi.json**
```json
{
  "report-download": {
    "form": {
      "language-option": {
        "en": "Anh",
        "es": "ngôn ngữ Tây ban nha",
        "vi": "Tiếng Việt"
      }
    }
  }
}
```