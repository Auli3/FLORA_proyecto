# Documentation - proyecto-data-automation

This documentation aims to explain the automation process, how it works, some possible errors it may face and how to mitigate them.
Esquema del flujo

The automation consists of 3 steps:
Mantain the Airtable table automatically updated through the Airtable automations.
Make use of the Google Sheets App Script extension to extract the table data through the Airtable API.
Connect the Google Sheets to the Looker Studio report in order to feed the metrics.

## Airtable

"seguimiento_paciente" table.
This "facts" table mantains a close follow up for every pacient en each step of the operation as shown in the following image: 

{imagen}

## Automations
Para mantener la tabla de “seguimiento_paciente” actualizada, se hace uso de las automatizaciones de Airtable, éstas están creadas en el grupo “Seguimiento de pacientes”.








Creación Pacientes (seguimiento_paciente)



Esta automatización se encarga de crear un nuevo registro en la tabla “seguimiento_paciente” para cada nuevo paciente que se cree en la tabla “pacientes” y no esté ya en la tabla “seguimiento_paciente”, seguido registra si este tiene alguna parte del proceso relacionada para rellenar los campos de relación con la otras tablas.











Carga inicial tabla "seguimiento_paciente"



Esta automatización se utilizó solo una vez para realizar la “carga inicial” de la tabla “seguimiento_paciente”, completando para cada paciente que ya está en la base de datos con sus respectivos campos.













Actualizacion seguimiento_paciente (consultas - creacion)


Esta automatización se encarga de actualizar los campos relacionados a las consultas para cada paciente, cada vez que se cree una nueva consulta en la tabla “consultas”, se busca a qué paciente pertenece y se actualiza su registro en la tabla “seguimiento_paciente”.
Actualizacion seguimiento_paciente (encuesta_satisfaccion - creacion)

Esta automatización se encarga de actualizar los campos relacionados a las encuestas para cada paciente, cada vez que se cree una nueva encuesta en la tabla “encuesta_satisfaccion”, se busca a qué paciente pertenece y se actualiza su registro en la tabla “seguimiento_paciente”.
Actualizacion seguimiento_paciente (transacciones - creacion)

Esta automatización se encarga de actualizar los campos relacionados a las transacciones para cada paciente, cada vez que se cree una nueva consulta en la tabla “transacciones”, se busca a qué paciente pertenece y se actualiza su registro en la tabla “seguimiento_paciente”.
Actualizacion seguimiento_paciente (estudios - creacion)

Esta automatización se encarga de actualizar los campos relacionados a los estudios para cada paciente, cada vez que se cree un nuevo estudio en la tabla “estudios”, se busca a qué paciente pertenece y se actualiza su registro en la tabla “seguimiento_paciente”.

Actualizacion seguimiento_paciente (medicamento - creacion)

Esta automatización se encarga de actualizar los campos relacionados a los medicamentos para cada paciente, cada vez que se cree un nuevo medicamento en la tabla “medicamentos”, se busca a qué paciente pertenece y se actualiza su registro en la tabla “seguimiento_paciente”.
Actualizacion seguimiento_paciente (tratamiento - creacion)

Esta automatización se encarga de actualizar los campos relacionados a los tratamientos para cada paciente, cada vez que se cree un nuevo tratamiento en la tabla “tratamientos”, se busca a qué paciente pertenece y se actualiza su registro en la tabla “seguimiento_paciente”.

Google Sheets
App Script
Utilizando el App Script se pueden recuperar los datos de las tablas de Airtable de manera automática cada 4hs (esto se puede modificar como se explica más adelante en el documento) por medio de su API.

Script:
// Función genérica para importar datos desde Airtable a Google Sheets
function importFromAirtableToSheet(accessToken, baseId, tableName, sheetName) {
  var sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName(sheetName);
  if (!sheet) {
    throw new Error('No se encontró la hoja con el nombre especificado: ' + sheetName);
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


    Logger.log('Datos importados correctamente desde Airtable a la hoja: ' + sheetName);


  } catch (e) {
    Logger.log('Error al importar desde Airtable: ' + e.message);
    throw new Error('Error al importar desde Airtable. Verifica la configuración de tu API y variables.');
  }
}


// Función para importar datos de todas las tablas especificadas
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
    if (pair.table && pair.sheet) { // Verificar que ambos parámetros están definidos
      try {
        importFromAirtableToSheet(accessToken, baseId, pair.table, pair.sheet);
      } catch (error) {
        Logger.log('Error al importar la tabla ' + pair.table + ' a la hoja ' + pair.sheet + ': ' + error.message);
      }
    } else {
      Logger.log('Par de tabla y hoja indefinido encontrado: ' + JSON.stringify(pair));
    }
  });
}
Agregar tablas al Script
Para agregar más tablas para que se carguen automáticamente al Google Sheets, se deben seguir los siguientes pasos:

Agregar nuevas hoja (sheet) al documento (nombrar acordemente).
Acceder al App Script del documento.


Una vez dentro desde la pestaña del editor de código buscar el diccionario de tablas de Airtable con los nombres de las hojas del documento.


Agregar como nueva entrada al diccionario de la siguiente manera el nombre de la tabla de Airtable a pasar, junto con el nombre que se le puso a la nueva hoja que contendrá esos datos:
{table: ‘nombre_de_la_tabla_de_airtable’, sheet: ‘nombre_de_la_hoja’}

Trigger
La función está programada para ejecutarse cada 4hs por medio de los triggers que proporciona la extensión de App Script.

Para modificar la frecuencia de actualización del Google Sheets se debe acceder a esta extensión desde el mismo documento.



Luego en la ventana emergente dirigirse a la sección de “Triggers” del menú izquierdo.



Seleccionar el trigger a editar y modificar la frecuencia de ejecución, ya sea por horas o minutos.






Airtable API
Para utilizar la API de la base de datos es necesario seguir los siguientes pasos:

Generar token
Desde el “Centro de desarrolladores”









Y luego siguiendo los pasos tocando el botón “Crear token”



Este token que se utiliza en el script es el que se utiliza para tener acceso a la base de datos indicada


























Looker Studio
Utilizando Looker Studio se puede mantener la información actualizada proveniente del Google Sheets.

El tablero volverá a leer los datos de Google Sheets cada 15 minutos.
Conexion Google Sheets

En el modo edición del tablero



Se accede a la pestaña de agregar datos.


Se selecciona la opción “Google Sheets”


Y se termina seleccionando el archivo y hojas de cálculo que se quieren agregar como fuentes de datos.

Modificación fuentes de datos (Google Sheets)
En caso de que alguna de las fuentes de datos cambien, ya sea desde Airtable agregando un nuevo campo, o directamente en Google Sheets con una nueva columna, se deberán seguir los siguientes pasos para actualizar el esquema que utiliza Looker para estas fuentes.




Se accede a la pestaña de recursos y luego a administrar fuentes de datos.



Se selecciona la fuente de datos a actualizar y el botón editar.


Una vez dentro se hace click en el botón de editar conexión.


Por último se hace la reconexión con la fuente de datos con el botón reconectar, y confirmando los nuevos campos o columnas agregados.

Dataset Configuration Error
En ocasiones puede aparecer este mensaje de error en algunas visualizaciones del tablero, esto debido a como esta interpretando Looker algunos campos del Google Sheets, comúnmente los campos de fechas.

Para resolver este problema se deben seguir los siguientes pasos:

Identificar que fuente de datos utiliza el gráfico.


Se accede a la pestaña de recursos y luego a administrar fuentes de datos.



Se selecciona la fuente de datos y el botón editar.


Se busca el campo a modificar y se selecciona el formato correcto del menu desplegable.

