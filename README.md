# Casco-minero-ESP32
/**
 * Casco Inteligente Minero · Subida de lecturas a Google Sheets (se puede bajar como CSV)
 *
 * PASOS:
 * 1. Crea una hoja de cálculo en Google Sheets (ej. "Casco Minero - Lecturas").
 * 2. Menú Extensiones → Apps Script. Pega este código y pon tu CLIENT_ID abajo.
 * 3. Implementar → Nueva implementación → Tipo: Aplicación web
 *      · Ejecutar como: Yo
 *      · Quién tiene acceso: Cualquier persona
 * 4. Copia la URL que termina en /exec y pégala en CFG.cloudUrl del index.html.
 * 5. Para el CSV: en la hoja, Archivo → Descargar → Valores separados por comas (.csv).
 */

const CLIENT_ID = 'PEGA_AQUI_TU_CLIENT_ID.apps.googleusercontent.com';

// Opcional: solo estos correos pueden subir datos. Déjalo vacío [] para permitir cualquier cuenta Google.
const CORREOS_PERMITIDOS = [];

const HOJA = 'Lecturas';
const COLUMNAS = ['fecha_hora','turno','casco','operador','temp','hum','ch4','lel','glp','humo',
                  'co2','nh3','nox','nivel','gas'];

function doPost(e) {
  try {
    const body = JSON.parse(e.postData.contents);

    // Verifica que el token realmente venga de Google y de TU aplicación
    const resp = UrlFetchApp.fetch(
      'https://oauth2.googleapis.com/tokeninfo?id_token=' + encodeURIComponent(body.token),
      { muteHttpExceptions: true }
    );
    const info = JSON.parse(resp.getContentText());
    if (info.aud !== CLIENT_ID) return salida({ ok: false, error: 'token inválido' });
    if (CORREOS_PERMITIDOS.length && CORREOS_PERMITIDOS.indexOf(info.email) < 0)
      return salida({ ok: false, error: 'correo no autorizado' });

    const ss = SpreadsheetApp.getActiveSpreadsheet();
    let sh = ss.getSheetByName(HOJA);
    if (!sh) {
      sh = ss.insertSheet(HOJA);
      sh.appendRow(COLUMNAS.concat(['enviado_por']));
      sh.setFrozenRows(1);
    }

    const filas = (body.rows || []).map(function (r) {
      return COLUMNAS.map(function (c) { return r[c] === undefined || r[c] === null ? '' : r[c]; })
             .concat([info.email]);
    });
    if (filas.length) {
      sh.getRange(sh.getLastRow() + 1, 1, filas.length, COLUMNAS.length + 1).setValues(filas);
    }
    return salida({ ok: true, n: filas.length });
  } catch (err) {
    return salida({ ok: false, error: String(err) });
  }
}

function doGet() {
  return salida({ ok: true, servicio: 'Casco Inteligente Minero' });
}

function salida(obj) {
  return ContentService.createTextOutput(JSON.stringify(obj))
    .setMimeType(ContentService.MimeType.JSON);
}
