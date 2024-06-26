# Documentation

## Overview(Updated Version)

This project implements functionality to generate and download an Excel file using data from an HTML table. The solution consists of JavaScript code on the frontend to gather the data and ASP.NET MVC code on the backend to process the data and generate the Excel file using the EPPlus library.

## JavaScript

### Function: `DownloadExcelFile()`

This function collects data from an HTML table, serializes it into JSON, and submits it to the server to generate an Excel file.

#### Steps:

1. **Variable Initialization**:
   - `postArrayUI` is initialized as an empty array to hold the selected data.

2. **Table Row Iteration**:
   - Iterate through each row of the table (`#dataTable`), excluding the header row.
   - Check if the checkbox (`.singleCheck`) in the row is checked.
   - If checked, collect `ContactNo` and `SmsText` from the row and push an object containing these into `postArrayUI`.

3. **Validation**:
   - If `postArrayUI` is empty, display an error message using SweetAlert and exit the function.

4. **Form Creation and Submission**:
   - Create a hidden form with a POST method targeting `/NC/AttendanceNotice/GenerateExcel`.
   - Add the serialized `postArrayUI` as a hidden input field.
   - Append the form to the body, submit it, and then remove it.

#### JavaScript Code:
```javascript
function DownloadExcelFile() {
    var postArrayUI = [];
    // Iterate through each row of the table
    $("#dataTable tr").not(":first").each(function () {
        var isChked = $(this).find(".singleCheck").is(':checked');
        if (isChked) {
            var singleObj = {
                ContactNo: $(this).find("td:eq(7)").text(),
                SmsText: $(this).find("td:eq(5)").text()
            };
            postArrayUI.push(singleObj);
        }
    });

    if (postArrayUI.length === 0) {
        swal('Sorry!!', 'No data found', 'error');
        return false;
    }

    var urlToCall = "/NC/AttendanceNotice/GenerateExcel";
    var form = $('<form method="POST" action="' + urlToCall + '">');
    form.append($('<input type="hidden" name="postArrayUI" value=\'' + JSON.stringify(postArrayUI) + '\'>'));
    $('body').append(form);
    form.submit();
    form.remove();
}
```

### Handling of Contact Numbers

In the previous version, while downloading the Excel file, contact numbers were displayed in scientific notation even though they were intended to be treated as text. The updated code ensures that the contact numbers retain their original format and do not convert to scientific notation.


<img width="948" alt="2" src="https://github.com/zahirulislam478/Documentation-on-NC-Excel/assets/35406920/6f6cfe16-a17c-47af-a87e-2aa1376afd91">


## C# (ASP.NET MVC)

### Endpoint: `/NC/AttendanceNotice/GenerateExcel`

This endpoint handles the request to generate an Excel file from the submitted data.

### Action Method: `GenerateExcel(string postArrayUI)`

This method is responsible for generating the Excel file from the data received in the `postArrayUI` parameter.

#### Steps:

1. **Deserialize JSON**:
   - Deserializes the input JSON string (`postArrayUI`) into a list of `GetDataFromUIViewModel`.

2. **Excel Package Creation**:
   - Creates a new Excel package and adds a worksheet named "Sheet1".

3. **DataTable Conversion**:
   - Converts the list to a DataTable.
   - Filters the DataTable to include only "ContactNo" and "SmsText" columns.

4. **Worksheet Population**:
   - Calls `AddHeaderRow()` to add a header row to the worksheet.
   - Calls `AddDataRows()` to populate the worksheet with data rows.

5. **File Creation and Return**:
   - Saves the Excel file to a memory stream.
   - Returns the file content as a downloadable file with the specified content type and filename.

6. **Helper Methods**:
   - `RenameColumn(DataTable table, string oldName, string newName)`: Renames a column in the DataTable from `oldName` to `newName`.
   - `AddHeaderRow(ExcelWorksheet worksheet, DataTable table)`: Adds a header row to the worksheet with styled header cells.
   - `AddDataRows(ExcelWorksheet worksheet, DataTable table)`: Adds data rows to the worksheet and styles the cells.

## Helper Methods

### `RenameColumn(DataTable table, string oldName, string newName)`

Renames a column in the DataTable from `oldName` to `newName`.

#### Parameters:

- `table`: The DataTable in which the column needs to be renamed.
- `oldName`: The current name of the column to be renamed.
- `newName`: The new name to assign to the column.

```csharp 
[HttpPost]
public ActionResult GenerateExcel(string postArrayUI)
{
    try
    {
        // Deserialize the input JSON string to a List of GetDataFromUIViewModel
        var data = JsonConvert.DeserializeObject<List<GetDataFromUIViewModel>>(postArrayUI);

        string filename = "Attendance Notice List.xlsx";
        string contentType = "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet";

        using (var excel = new ExcelPackage())
        {
            var worksheet = excel.Workbook.Worksheets.Add("Sheet1");

            // Convert list to DataTable
            DataTable dataTable = ListExtensions.ToDataTable(data);
            dataTable.TableName = "Selected";

            // Select only necessary columns
            var view = new DataView(dataTable);
            DataTable filteredTable = view.ToTable(false, "ContactNo", "SmsText");

            // Rename columns
            RenameColumn(filteredTable, "ContactNo", "Mobile No.");
            RenameColumn(filteredTable, "SmsText", "Message Text");

            // Add header row
            AddHeaderRow(worksheet, filteredTable);

            // Add data rows
            AddDataRows(worksheet, filteredTable);

            // Autofit all columns
            worksheet.Cells.AutoFitColumns();

            // Save the Excel file to a memory stream
            using (var memoryStream = new MemoryStream())
            {
                excel.SaveAs(memoryStream);
                memoryStream.Position = 0;

                var fileContent = memoryStream.ToArray();
                return File(fileContent, contentType, filename);
            }
        }
    }
    catch (Exception ex)
    {
        ex.ToTextFileLog(); // Custom logging method
        ex.ToMssqlLog();    // Custom logging method
        throw ex;
    }
}

private void RenameColumn(DataTable table, string oldName, string newName)
{
    if (table.Columns.Contains(oldName))
    {
        table.Columns[oldName].ColumnName = newName;
    }
}

private void AddHeaderRow(ExcelWorksheet worksheet, DataTable table)
{
    int colIndex = 1;

    foreach (DataColumn column in table.Columns)
    {
        var cell = worksheet.Cells[1, colIndex];

        // Setting the background color of header cells to Gray
        cell.Style.Fill.PatternType = ExcelFillStyle.Solid;
        cell.Style.Fill.BackgroundColor.SetColor(Color.LightGray);

        // Setting borders of header cells
        cell.Style.Border.Bottom.Style = ExcelBorderStyle.Thin;
        cell.Style.Border.Top.Style = ExcelBorderStyle.Thin;
        cell.Style.Border.Left.Style = ExcelBorderStyle.Thin;
        cell.Style.Border.Right.Style = ExcelBorderStyle.Thin;

        // Setting the value of header cell
        cell.Value = column.ColumnName;

        colIndex++;
    }
}

private void AddDataRows(ExcelWorksheet worksheet, DataTable table)
{
    int rowIndex = 2;

    foreach (DataRow row in table.Rows)
    {
        int colIndex = 1;

        foreach (DataColumn column in table.Columns)
        {
            var cell = worksheet.Cells[rowIndex, colIndex];
            cell.Value = row[column.ColumnName].ToString();

            // Setting borders of data cells
            cell.Style.Border.Bottom.Style = ExcelBorderStyle.Thin;
            cell.Style.Border.Top.Style = ExcelBorderStyle.Thin;
            cell.Style.Border.Left.Style = ExcelBorderStyle.Thin;
            cell.Style.Border.Right.Style = ExcelBorderStyle.Thin;

            colIndex++;
        }
        rowIndex++;
    }
}
```


## Overview(Previous Version)

The previous version relied solely on client-side JavaScript and third-party plugins for Excel file generation, while the updated approach integrates server-side processing for more sophisticated validation, error handling, and improved user interaction.

# Function: DownloadExcelFile()

This function collects data from an HTML table, constructs a new hidden table with the selected data, and then uses the jquery-table2excel library to download the data as an Excel file.

## Table Row Iteration

1. Iterate through each row of the table (#dataTable), excluding the header row.
2. For each row:
    - Extract `contactNo` from the 8th cell (`td:eq(7)`).
    - Extract `smsText` from the 6th cell (`td:eq(5)`).
    - Check if the checkbox (`.singleCheck`) in the row is checked.
    - If checked, create a new row (`<tr>`) and append it to the hidden table (#DownloadSMSTable).

## Excel File Download

- Use the table2excel plugin to convert the hidden table to an Excel file and trigger a download with the filename `SendSMSFile.xls`.

## JavaScript Code

```javascript
function DownloadExcelFile() {
    $("#dataTable tr").not(":first").each(function () {
        var contactNo = $(this).find("td:eq(7)").text();
        var smsText = $(this).find("td:eq(5)").text();
        if ($(this).find(".singleCheck").prop('checked') == true) {
            tr = $('<tr/>');
            tr.append("<td>" + contactNo + "</td>");
            tr.append("<td>" + smsText + "</td>");
            $('#DownloadSMSTable').append(tr);
        }
    });
    $("#DownloadSMSTable").table2excel({
        exclude: '.exclude',
        filename: 'SendSMSFile.xls'
    });
};
```

### Handling of Contact Numbers

In the updated version, the `ContactNo` column is handled as text to avoid displaying contact numbers in scientific notation. This is achieved by explicitly setting the format for the `ContactNo` cells

<img width="943" alt="1" src="https://github.com/zahirulislam478/Documentation-on-NC-Excel/assets/35406920/5a47661b-b048-4413-9de4-320e0f3f9453">


# Differences Between the Two Documentations

## 1. Approach and Methodology

### Previous Version (Using table2excel Library)
- **Front End Only:** Data gathering and Excel file generation occur entirely on the client side.
- **Simplicity:** Relies on client-side JavaScript and a third-party plugin (`table2excel`) for file creation.

### Updated Version (Using Server-Side Processing)
- **Client-Server Interaction:** Data is sent to the server for processing and Excel generation.
- **Enhanced Security:** Utilizes server-side capabilities to handle data securely and perform complex operations.

## 2. Data Handling

### Previous Version
- **Direct DOM Manipulation:** Collects data directly from the HTML table on the client side.
- **Limited Processing:** Basic data handling capabilities within the constraints of client-side scripting.

### Updated Version
- **Serialization and Server-Side Processing:** Serializes data, sends it to the server, and processes it with ASP.NET MVC.
- **Flexibility:** Supports more advanced data manipulation and validation on the server.

## 3. Excel File Generation

### Previous Version
- **Using table2excel Plugin:** Converts a hidden table to an Excel file using a client-side library.
- **User Experience:** Offers immediate file download without server interaction.

### Updated Version
- **EPPlus Library:** Generates Excel files on the server with EPPlus for greater control and customization.
- **Scalability:** Handles larger datasets and more complex file generation requirements.

## 4. Validation and Error Handling

### Previous Version
- **Limited Validation:** Basic checks for data presence before file generation.

### Updated Version
- **Enhanced Error Handling:** Validates data before submission and handles exceptions robustly on the server.

## 5. Code Structure and Documentation

### Previous Version
- **Client-Side Focus:** Emphasizes client-side JavaScript and DOM manipulation.
- **Simpler Documentation:** Describes a straightforward client-side workflow.

### Updated Version
- **Comprehensive Documentation:** Covers both client-side and server-side workflows, including data serialization and server handling.
- **Best Practices:** Includes detailed explanations of methods, parameters, and error handling practices.

## Summary

The previous version utilizes client-side scripting and a third-party library (`table2excel`) for Excel file generation, while the updated version implements a more sophisticated approach with server-side processing using ASP.NET MVC and the EPPlus library. This update enhances security, scalability, and flexibility for handling data and generating Excel files, supported by comprehensive documentation of both client-side and server-side workflows.

