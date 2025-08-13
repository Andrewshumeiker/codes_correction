Voy a explicarte exactamente dónde hacer cada cambio en tus archivos, línea por línea. Sigue estas instrucciones cuidadosamente:
```
Error: {"success":false,"error":"Error procesando CSV: res.status(...).strictContentLength is not a function"}
```
});

```
The requested module './utils/uploadMiddleware.js' does not provide an export named 'csvUpload'
    at ModuleJob._instantiate (node:internal/modules/esm/module_job:220:21)
    at async ModuleJob.run (node:internal/modules/esm/module_job:321:5)
    at async onImport.tracePromise.__proto__ (node:internal/modules/esm/loader:644:26)
    at async asyncRunEntryPointWithESMLoader (node:internal/modules/run_main:117:5)

Node.js v22.17.0
coders@p4m2-0510:~/pd-sql-project/pd-sql-project/backend$ npm start

> pd-sql-project@1.0.0 start
> node src/app.js

file:///home/coders/pd-sql-project/pd-sql-project/backend/src/app.js:7
import { csvUpload } from './utils/uploadMiddleware.js'; // Cambio aquí
         ^^^^^^^^^
SyntaxError: The requested module './utils/uploadMiddleware.js' does not provide an export named 'csvUpload'
    at ModuleJob._instantiate (node:internal/modules/esm/module_job:220:21)
    at async ModuleJob.run (node:internal/modules/esm/module_job:321:5)
    at async onImport.tracePromise.__proto__ (node:internal/modules/esm/loader:644:26)
    at async asyncRunEntryPointWithESMLoader (node:internal/modules/run_main:117:5)

Node.js v22.17.0
```
### 1. Archivo: `backend/app.js`

**Cambios necesarios:**

```javascript
// ... código existente ...
import { csvUpload } from './utils/uploadMiddleware.js'; // <-- Esta línea ya existe

// ELIMINAR ESTA LÍNEA:
app.use(fileUpload());  // <-- BORRA ESTA LÍNEA COMPLETAMENTE

// ... código existente ...

// REEMPLAZAR TODO EL BLOQUE DEL ENDPOINT /api/upload-csv CON ESTE CÓDIGO:
app.post('/api/upload-csv', csvUpload, async (req, res) => {
  try {
    // Usar req.files en lugar de req.file
    if (!req.files || !req.files.csv) {
      return res.status(400).json({
        success: false,
        error: 'No se subió ningún archivo CSV'
      });
    }
    
    const csvString = req.files.csv.data.toString('utf8');
    const result = await loadCSVData(csvString);
    
    res.status(200).json({
      success: true,
      message: result.message
    });
  } catch (error) {
    console.error('Error en carga CSV:', error);
    res.status(500).json({
      success: false,
      error: 'Error procesando CSV: ' + error.message
    });
  }
});

// ... el resto del código permanece igual ...
```

### 2. Archivo: `backend/utils/uploadMiddleware.js`

**Reemplazar TODO el contenido del archivo con:**

```javascript
import multer from 'multer';

const upload = multer({
  storage: multer.memoryStorage(),
  fileFilter: (req, file, cb) => {
    const allowedMimes = [
      'text/csv',
      'application/vnd.ms-excel',
      'text/plain',
      'application/octet-stream'
    ];
    
    const isCSV = allowedMimes.includes(file.mimetype) || 
                 file.originalname.toLowerCase().endsWith('.csv');
    
    if (isCSV) {
      cb(null, true);
    } else {
      cb(new Error('Tipo de archivo inválido. Solo se permiten archivos CSV'), false);
    }
  },
  limits: { fileSize: 20 * 1024 * 1024 }
});

// Exportar directamente el middleware
export default upload.single('csv');
```

### 3. Archivo: `frontend/public/js/main.js`

**Cambiar SOLO esta parte del código:**

```javascript
// ... código existente ...

input.onchange = async (e) => {
  const file = e.target.files[0];
  if (!file) return;
  
  const formData = new FormData();
  formData.append('csv', file); // 'csv' debe coincidir con el backend
  
  try {
    loadCsvBtn.disabled = true;
    loadCsvBtn.textContent = 'Cargando...';
    
    // IMPORTANTE: NO agregar headers 'Content-Type'
    const response = await fetch(`${API_BASE_URL}/api/upload-csv`, {
      method: 'POST',
      body: formData
    });
    
    // ... el resto del código permanece igual ...
};

// ... código existente ...
```

### 4. Archivo: `backend/services/csvLoader.js`

**Modificar la función `loadCSVData` así:**

```javascript
export async function loadCSVData(csvString) {
  try {
    const records = parse(csvString, {
      columns: true,
      skip_empty_lines: true,
      delimiter: ',',
      trim: true,
      skip_records_with_error: true,
      cast: (value, context) => {
        // Convertir campos vacíos a null
        if (!value || value.trim() === '') return null;
        
        // Convertir montos a números
        if (context.column.includes('Monto') || 
            context.column.includes('Total') || 
            context.column.includes('amount')) {
          return parseFloat(value.replace(/[^0-9.-]/g, ''));
        }
        
        // Convertir fechas
        if (context.column.includes('Fecha') || 
            context.column.includes('date')) {
          return new Date(value);
        }
        
        return value;
      }
    });

    if (!records || records.length === 0) {
      throw new Error('El archivo CSV está vacío o no contiene datos válidos');
    }

    // ... EL RESTO DEL CÓDIGO PERMANECE IGUAL DESDE AQUÍ ...
    
  } catch (error) {
    console.error('Error en loadCSVData:', error);
    throw new Error(`Error al procesar CSV: ${error.message}`);
  }
}
```

### 5. Archivo: `backend/controllers/customerController.js`

**Añadir manejo de error para duplicados en la función `createCustomer`:**

```javascript
export const createCustomer = async (req, res) => {
  // ... código existente ...
  try {
    // ... código existente ...
  } catch (error) {
    console.error('Error creando cliente:', error);
    
    // AÑADIR ESTE BLOQUE PARA MANEJAR ERRORES DE DUPLICADOS:
    if (error.code === 'ER_DUP_ENTRY') {
      return res.status(409).json({
        success: false,
        error: 'El email ya existe en la base de datos'
      });
    }
    
    res.status(500).json({
      success: false,
      error: 'Error de base de datos: ' + error.message
    });
  }
};
```

### 6. Archivo: `backend/load-data.js`

**Modificar las rutas de los archivos CSV:**

```javascript
// ... código existente ...

async function main() {
  // ... código existente ...
  
  try {
    // ... código existente ...
    
    // REEMPLAZAR ESTAS 4 LÍNEAS:
    const basePath = path.resolve(process.cwd(), 'backend', 'database');
    
    await loadTableFromCsv(connection, path.join(basePath, 'customers.csv'), 'customers');
    await loadTableFromCsv(connection, path.join(basePath, 'platforms.csv'), 'platforms');
    await loadTableFromCsv(connection, path.join(basePath, 'invoices.csv'), 'invoices');
    await loadTableFromCsv(connection, path.join(basePath, 'transactions.csv'), 'transactions');
    
    // ... el resto del código permanece igual ...
};
```

### Pasos adicionales obligatorios:

1. **Elimina `express-fileupload` de tus dependencias:**
   ```bash
   npm uninstall express-fileupload
   ```

2. **Asegúrate de tener estas dependencias instaladas:**
   ```bash
   npm install multer csv-parse
   ```

3. **Verifica que tu archivo `.env` tenga:**
   ```env
   # backend/.env
   FRONTEND_URL=http://localhost:5500
   # Asegúrate que este sea el puerto de tu frontend
   ```

4. **Reinicia completamente tu servidor backend después de hacer estos cambios.**

### Resumen de cambios clave:

1. **Backend:**
   - Eliminado `express-fileupload`
   - Usando solo `multer` para manejo de archivos
   - Corregido `req.file` → `req.files.csv`
   - Mejorado el parser CSV con conversión de tipos
   - Rutas absolutas para archivos CSV

2. **Frontend:**
   - Asegurado que el FormData use el campo 'csv'
   - Eliminado header 'Content-Type' en la petición

3. **General:**
   - Manejo de errores mejorado
   - Validación de tipos de archivo más flexible
   - Conversión robusta de números y fechas

Con estos cambios exactos, tu sistema debería permitir subir y procesar archivos CSV correctamente. ¡Verifica cada cambio cuidadosamente!
