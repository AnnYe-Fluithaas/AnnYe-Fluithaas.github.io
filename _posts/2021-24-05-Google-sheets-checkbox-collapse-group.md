---
layout: post
title:  "Checkbox toggle to expand/collapse a Group, in Google Sheets"
date:   2021-05-24 16:34:00 +0100
# categories: google-sheets
---

I have a spreadsheet where I have a list of items, and then several columns of additional meta-data about those items. I could group the 3 meta-data 
columns into a group, which I collapse manually, but I wanted to have the expand-collapse functionality from a checkbox. To do this, I edited
the script for the Google Sheet.



```
function onEdit(e) {
  var activeSheet = SpreadsheetApp.getActiveSheet();
  if (activeSheet.getName() === "MySheet") {
    onEditOfGameSheet(e, activeSheet);
  }
}

function onEditOfGameSheet(e, activeSheet) {
  var row = e.range.getRow();
  var col = e.range.getColumn();

  // Using a named range. You can check/edit the value of the named range used, 
  // in the spreadsheet > Data > Named ranges
  var infoFilterRange = activeSheet.getRange('InfoFilter');

  //Checking that we are selecting the filter
  if (isRowAndColumnWithinRange(row, col, infoFilterRange)) {
    filterinfo(e, activeSheet);
  }
}

function isRowAndColumnWithinRange(row, column, range) {
  return column >= range.getColumn()
        && column <= range.getLastColumn() 
        && row >= range.getRow() 
        && row <= range.getLastRow();
}

function filterinfo(e, activeSheet) {
  var group = activeSheet.getColumnGroup(3, 1);
  if(e.range.getValue() === true){
    group.expand();
  }
  else {
    group.collapse();
  }
}
```
