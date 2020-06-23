## Color Palette
**Intended Audience**: This document describes how to adjust color definitions in the 
SBAC Global UI Kit and how to apply these changes to the [Reporting Data Warehouse](../README.md) (RDW).
It does not cover adding or removing colors or changing color names.

#### Changes in SBAC Global UI Kit
The color codes are defined in the [SBAC Global UI Kit](https://github.com/SmarterApp/SBAC-Global-UI-Kit). 

Most of the color palette is defined in `colors.less`, but `sbac-iab-green`, 
`sbac-iab-red`, `sbac-iab-yellow` are defined in `rdw.less`. In the correct file,
find the color definition that needs to be changed, and update its code with the 
desired value. For example, to change the orange color, locate its color definition in
`colors.less`:

```less
...
@purple:                #A5508C;
@orange:                #B15E15;
@cobalt:                #4449BE;
@red:                   #D5124A;
...
```

... and update it with the new color code:
```less
...
@purple:                #A5508C;
@orange:                #FFA500;
@cobalt:                #4449BE;
@red:                   #D5124A;
...
```

Be careful to keep the same syntax. The color name should not change and should be
prefixed with an at sign (@). The color code should be an RGB hex value prefixed with
a pound sign (#) and followed by a semicolon.

The next file to update is `index.html`. This file is for documentation only so it
does not affect the functionality of RDW, but it is useful to keep it synchronized with
the color definitions. Locate the color whose definition need to be changed in the 
html code:

```html
<!-- Reds -->
<div class="col-md-3">
  <div class="panel small">
    <div class="panel-heading">Reds</div>
    ...
    <div class="panel-body orange"><span class="label">.orange</span><span class="hex right">#B15E15</span></div>
    ....
```
... and update the color code text:
```html
<!-- Reds -->
<div class="col-md-3">
  <div class="panel small">
    <div class="panel-heading">Reds</div>
    ...
    <div class="panel-body orange"><span class="label">.orange</span><span class="hex right">#FFA500</span></div>
    ....
```

The last change is to increment the version in `package.json`:

```json
{
  "name": "@sbac/sbac-ui-kit",
  "version": "0.0.33",
  "description": "Common web styling and assets for SBAC projects",
...
```

... should change to: 
```json
{
  "name": "@sbac/sbac-ui-kit",
  "version": "0.0.34",
  "description": "Common web styling and assets for SBAC projects",
  ...
```

The changes can now be pushed out to the git repository, where the definitions will
be automatically built into CSS style files available to RDW. Note that the build 
automatically derives several ancillary color definitions for each base color.
For example, the foreground color used for text appearing on top of the base color is
automatically defined as white or black, whichever gives the better contrast. In 
addition, other colors used for drawing a border for the base color or a contrasting
column header are also generated. There should be no need to modify these generated
color definitions.

#### Changes in RDW
   
In `package.json`, which is located in the [RDW Reporting](https://github.com/SmarterApp/RDW_Reporting)
project, in the `webapp` module, locate the `sbac-ui-kit` entry in the dependencies: 
```json
"dependencies": {
  ... 
  "@sbac/sbac-ui-kit": "0.0.33",
  ...
}
```

Adjust this version to match what was created in the SBAC Global UI Kit project:
```json
"dependencies": {
  ... 
  "@sbac/sbac-ui-kit": "0.0.34",
  ...
}
```

When you rebuild the webapp module, the new style definitions will immediately be available
to the entire UI project. However, one more change is necessary to make these definitions
available to the PDF generator that produces the printed reports. The `fragments.html` file
contains a large section of minified CSS styles, which must be updated with the entire contents
of  `sbac-ui-kit-wkhtmltopdf.min.css`, which was updated in the previous section
when the version was incremented and the webapp module rebuilt. Locate the file in the root
of the RDW Reporting project at:
`./webapp/node_modules/@sbac/sbac-ui-kit/dist/css/sbac-ui-kit-wkhtmltopdf.min.css`

... and copy the entire contents of this file. Next, locate `fragments.html` at:
`./report-processor/src/main/resources/templates/fragments.html`

In `fragments.html`, find the previous minified CSS definitions in the `styles-libraries` div:
```html
<div th:fragment="styles-libraries">

    <style>
/*!
 * Bootstrap v3.4.1 (https://getbootstrap.com/)
 * Copyright 2011-2019 Twitter, Inc.
 * Licensed under MIT (https://github.com/twbs/bootstrap/blob/master/LICENSE)
 *//*! normalize.css v3.0.3 | MIT License | github.com/necolas/normalize.css */
.badge,b,dt,kbd kbd,label,optgroup,strong{font-weight:700}.label,audio,canvas,progress,sub,sup,video{vertical-align:baseline}.dropdown .dropdown-toggle .caret,.fa,
...
   </style>
...
</div>
```

Replace the entire contents between `<style>` and `</style>` by pasting in the content
copied from `sbac-ui-kit-wkhtmltopdf.min.css`. 

Finally, rebuild the whole RDW Reporting project. The color definition changes will now be applied
to both the UI and the printed reports. 

#### Change in Subject Definition Workbook
The Subject Definition Workbook is located in this project, [RDW](https://github.com/SmarterApp/RDW),
in the file `SubjectDef.xlsm` which is under the `tools` folder. The `Reference` tab 
of this worksheet contains a list of colors defined in the SBAC Global UI Kit. 
This list is used only for reference when choosing appropriate colors for new 
subject definitions and does not have any impact on the generated subject files.

To change a color definition, first unprotect the sheet for the `Reference` tab by
right-clicking on the tab and selecting `Unprotect Sheet ...`. Then locate the color
in the list and update its color code. For example, for the color orange:

![Original](wksht-colors1.png)

.... first, update the code:

![Updated hex code](wksht-colors2.png)

Next, to update the color swatch, right click its cell, select `Format Cells ...`, then select
`More Colors ...`, and then `RGB Sliders`. In this window, update the `Hex Color` with
the new color code:

![Color picker](wksht-colors3.png)

Select OK, and the color swatch should be updated:

![Completed](wksht-colors4.png)

Finally, protect the `Reference` tab again. If prompted, do *not* change the protection
options for the tab and do not enter a password. Save the worksheet and commit your changes
to the project.
