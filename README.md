# Documentation - proyecto-data-automation

This documentation aims to explain the automation process, how it works, some possible errors it may face and how to mitigate them.
Flow diagram

![esquema_flujo](https://github.com/user-attachments/assets/dcc7b72f-d318-490b-aeeb-cb833cea2e71)

The automation consists of 3 steps:
* Mantain the Airtable table automatically updated through the Airtable automations.
* Make use of the Google Sheets App Script extension to extract the table data through the Airtable API.
* Connect the Google Sheets to the Looker Studio report in order to feed the metrics.

## Airtable

"seguimiento_paciente" table.
This "facts" table mantains a close follow up for every pacient en each step of the operation as shown in the following image: 

![Flora _ Proceso Simplificado](https://github.com/user-attachments/assets/70d72ce5-2f84-4fa7-8b5c-3af48133cbc1)

## Automations
I used the Airatable automations feature in order to maintan the "seguimiento_paciente" table updated.

* Creación Pacientes (seguimiento_paciente)

![Captura](https://github.com/user-attachments/assets/2f9feac0-8d90-46e3-b84b-9598e1e103a1)

This automation is responsible for creating a new record in the “seguimiento_paciente” table for each new patient that is created in the “pacientes” table and is not already in the “seguimiento_paciente” table, then records whether this patient has any part of the related process to fill in the relationship fields with the other tables.

* Carga inicial tabla "seguimiento_paciente"

![Captura](https://github.com/user-attachments/assets/5884d0ff-28f0-492c-9b78-eda9dbc07199)

This automation was used only once to perform the “initial load” of the “seguimiento_paciente” table, completing for each patient that is already in the database with their respective fields.

* Actualizacion seguimiento_paciente (consultas - creacion)

![Captura](https://github.com/user-attachments/assets/bcb9d60b-990d-43d9-8756-08e8e6b080f8)

This automation is responsible for updating the fields related to the consultations for each patient. Every time a new query is created in the “consultas” table, the patient it belongs to is searched and its record is updated in the “seguimiento_paciente” table.

* Actualizacion seguimiento_paciente (encuesta_satisfaccion - creacion)

![Captura](https://github.com/user-attachments/assets/fa9fce53-1b24-4117-959f-85c609f22f04)

This automation is responsible for updating the fields related to the surveys for each patient. Each time a new survey is created in the “encuesta_satisfaccion” table, the patient it belongs to is searched for and its record is updated in the “seguimiento_paciente” table.

* Actualizacion seguimiento_paciente (transacciones - creacion)

![Captura](https://github.com/user-attachments/assets/85208e68-7e63-43c6-b726-bc8233c1005e)

This automation is responsible for updating the fields related to transactions for each patient. Every time a new query is created in the “transacciones” table, the patient it belongs to is searched and its record is updated in the “seguimiento_paciente” table.

* Actualizacion seguimiento_paciente (estudios - creacion)

![Captura](https://github.com/user-attachments/assets/654bd5e1-3ba6-4bd2-97b9-9f03342ce375)

This automation is responsible for updating the fields related to the studies for each patient. Each time a new study is created in the “estudios” table, the patient it belongs to is searched for and its record is updated in the “seguimiento_paciente” table.

* Actualizacion seguimiento_paciente (medicamento - creacion)

![Captura](https://github.com/user-attachments/assets/a1844284-825c-49a3-b2b2-55871b7ddd8c)

This automation is responsible for updating the fields related to medications for each patient. Every time a new medication is created in the “medicamento” table, the patient it belongs to is searched and its record is updated in the “seguimiento_paciente” table.

* Actualizacion seguimiento_paciente (tratamiento - creacion)

![Captura](https://github.com/user-attachments/assets/89a3bad9-581e-419f-9648-3310b739f386)

This automation is responsible for updating the fields related to the treatments for each patient. Each time a new treatment is created in the “tratamiento” table, the patient it belongs to is searched and its record is updated in the “seguimiento_paciente” table.

## Google Sheets
### App Script
Using the App Script, data from Airtable tables can be retrieved automatically every 4 hours (this can be modified as explained later in the document) through its API.

Script:

    // Generic function to import data from Airtable to Google Sheets
    function importFromAirtableToSheet(accessToken, baseId, tableName, sheetName) {
      var sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName(sheetName);
      if (!sheet) {
        throw new Error('The sheet with the specified name was not found:' + sheetName);
      }
    
    
      var url = 'https://api.airtable.com/v0/' + baseId + '/' + tableName;
      var headers = {
        'Authorization': 'Bearer ' + accessToken
      };
    
    
      var options = {
        'headers': headers
      };
    
    
      try {
        var offset = '';
        var allRecords = [];
        var columnHeaders = new Set();
    
    
        do {
          var urlWithOffset = url + (offset ? '?offset=' + offset : '');
          var response = UrlFetchApp.fetch(urlWithOffset, options);
          var data = JSON.parse(response.getContentText());
    
    
          allRecords = allRecords.concat(data.records);
    
    
          data.records.forEach(record => {
            Object.keys(record.fields).forEach(key => columnHeaders.add(key));
          });
    
    
          offset = data.offset;
        } while (offset);
    
    
        columnHeaders = Array.from(columnHeaders);
    
    
        sheet.clear();
        sheet.appendRow(columnHeaders);
    
    
        var rows = allRecords.map(record => {
          var row = [];
          columnHeaders.forEach(header => {
            row.push(record.fields[header] || '');
          });
          return row;
        });
    
    
        if (rows.length > 0) {
          sheet.getRange(sheet.getLastRow() + 1, 1, rows.length, columnHeaders.length).setValues(rows);
        }
    
    
        Logger.log('Data successfully imported from Airtable to sheet:' + sheetName);
    
    
      } catch (e) {
        Logger.log('Error importing from Airtable:' + e.message);
        throw new Error('Error importing from Airtable. Check your API configuration and variables.');
      }
    }
    
    
    // Function to import data from all specified tables
    function importAllTables() {
      const accessToken = 'patW9kgrx7emTsR7I.027c0be71acb33358904f22077f1c13de6234bbc9b67f73ed635a525f2ce17db';
      const baseId = 'appJpPv3TAUi3SnWM';
    
    
      const tablesAndSheets = [
        {table: 'pacientes', sheet: 'Pacientes'},
        {table: 'consultas', sheet: 'Consultas'},
        {table: 'ordenes', sheet: 'Ordenes'},
        {table: 'transacciones', sheet: 'Transacciones'},
        {table: 'estudios', sheet: 'Estudios'},
        {table: 'medicamentos', sheet: 'Medicamentos'},
        {table: 'productos', sheet: 'Productos'},
        {table: 'tratamientos', sheet: 'Tratamientos'},
        {table: 'seguimiento_paciente', sheet:'Seguimiento_paciente'}
      ];
    
    
      tablesAndSheets.forEach(pair => {
        if (pair.table && pair.sheet) { // Verify that both parameters are defined
          try {
            importFromAirtableToSheet(accessToken, baseId, pair.table, pair.sheet);
          } catch (error) {
            Logger.log('Error importing table ' + pair.table + ' to the sheet ' + pair.sheet + ': ' + error.message);
          }
        } else {
          Logger.log('Undefined table and sheet pair found:' + JSON.stringify(pair));
        }
      });
    }

### Add tables to the Script
To add more tables to be automatically uploaded to Google Sheets, follow these steps:

* Add new sheet (sheet) to the document (name accordingly).
  
* Acceder al App Script del documento.

![Captura](https://github.com/user-attachments/assets/78bc074e-f63f-4151-94b5-2fc91e7e8cb8)

* Once inside, from the code editor tab, look for the Airtable table dictionary with the names of the document sheets.

![Captura](https://github.com/user-attachments/assets/5144960a-e841-47c5-95ef-10f4eb9104ea)

* Add the name of the Airtable table to be passed as a new entry to the dictionary as follows, along with the name given to the new sheet that will contain that data:
  {table: ‘airtable_table_name’, sheet: ‘sheet_name’}

### Trigger
The function is scheduled to run every 4 hours through the triggers provided by the App Script extension.

To modify the update frequency of Google Sheets, you must access this extension from the same document.

![Captura](https://github.com/user-attachments/assets/2d298f67-d599-41f3-b5f4-bf3aadda4324)

Then in the pop-up window go to the “Triggers” section of the left menu.

![Captura](https://github.com/user-attachments/assets/1eff9ef0-c00c-4cbd-9bab-58da0fc808f5)

Select the trigger to edit and modify the execution frequency, either by hours or minutes.

![Captura](https://github.com/user-attachments/assets/ff84ee41-de6f-42ab-b21e-2a1303bc95d4)

![Captura](https://github.com/user-attachments/assets/87b3b847-5523-4306-906b-76ed114071d0)

## Airtable API
To use the database API it is necessary to follow the following steps:

* Generar token
From the “Developer Center”

![Captura](https://github.com/user-attachments/assets/d8317cd0-0636-464d-bd57-67d37346476d)

* And then following the steps by tapping the “Create token” button

![Captura](https://github.com/user-attachments/assets/fa175112-c556-4fc2-84f6-0aacc9e1f3ae)

* This token used in the script is the one used to access the indicated database

## Looker Studio
Using Looker Studio you can keep updated information from Google Sheets.

![looker_tablero](https://github.com/user-attachments/assets/1f1f4411-a5c0-4b65-9dee-7a002d4dfd54)

The dashboard will re-read data from Google Sheets every 15 minutes.

### Google Sheets Connection

* In board edit mode

![Captura](https://github.com/user-attachments/assets/d9de2c69-46a1-4a6f-a069-a4750ab9456a)

* Access the add data tab.

![Captura](https://github.com/user-attachments/assets/89e21537-7483-4bc2-88a1-a7820e1b11f0)

* Select the “Google Sheets” option.

![Captura](https://github.com/user-attachments/assets/6177b4c4-3dba-406d-a59b-6b28e864b2cb)

* And you finish by selecting the file and spreadsheets that you want to add as data sources.

### Modificación fuentes de datos (Google Sheets)
In case any of the data sources change, either from Airtable adding a new field, or directly in Google Sheets with a new column, the following steps must be followed to update the schema that Looker uses for these sources.

* Access the resources tab and then manage data sources.

![Captura](https://github.com/user-attachments/assets/f9403225-bfb5-4501-a66b-8ead17b7b9b3)

* Select the data source to update and the edit button.

![Captura](https://github.com/user-attachments/assets/b11b0f0f-469c-4772-8ff9-49734b2fc789)

* Once inside, click on the edit connection button.

![Captura](https://github.com/user-attachments/assets/6a921e8e-6be3-4e79-acb3-443258f196af)

* Finally, the reconnection is made with the data source with the reconnect button, and confirming the new fields or columns added.

![Captura](https://github.com/user-attachments/assets/20516d47-645d-42f4-a44c-4645fb72f732)

### Dataset Configuration Error
Sometimes this error message may appear in some dashboard visualizations, due to how Looker is interpreting some Google Sheets fields, commonly date fields.
To resolve this problem, the following steps must be followed:

* Identify which data source the graph uses.

![Captura](https://github.com/user-attachments/assets/1e89bd32-4f0b-417f-9a7d-7ec7481b9696)

* Access the resources tab and then manage data sources.

![Captura](https://github.com/user-attachments/assets/b0c5ad2e-adbf-4559-92b0-e17430f02730)

* The data source is selected and the edit button is selected.

![Captura](https://github.com/user-attachments/assets/f485d4dc-94e0-4dd7-a9bc-5d92450591c5)

* Find the field to modify and select the correct format from the drop-down menu.

![Captura](https://github.com/user-attachments/assets/1fc8258e-b2f3-41aa-9c98-a4d91b91b3c0)
