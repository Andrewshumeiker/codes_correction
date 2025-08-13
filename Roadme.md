Voy a explicarte exactamente dónde hacer cada cambio en tus archivos, línea por línea. Sigue estas instrucciones cuidadosamente:
```
import express from 'express';
import bodyParser from 'body-parser';
import dotenv from 'dotenv';
import cors from 'cors';
import customerRoutes from './routes/customerRoutes.js';
import reportRoutes from './routes/reportRoutes.js';
import  csvUpload  from './utils/uploadMiddleware.js'; // Cambio aquí
import { loadCSVData } from './services/csvLoader.js';
import db from './config/db.js';

dotenv.config();
const app = express();
const PORT = process.env.PORT || 3000;

// Configuración de CORS actualizada
const allowedOrigins = [
  'http://localhost:5500',
  'http://127.0.0.1:5500',
  'http://localhost:8080',
  'http://127.0.0.1:8080',
  'http://localhost:3000', // Añadir para React/Vue
  'http://127.0.0.1:3000'  // Añadir para React/Vue
];

if (process.env.FRONTEND_URL) {
  allowedOrigins.push(process.env.FRONTEND_URL);
}

const corsOptions = {
  origin: allowedOrigins,
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  credentials: true,
  optionsSuccessStatus: 200
};

app.use(cors(corsOptions));
app.use((req, res, next) => {
  console.log(`[${new Date().toISOString()}] ${req.method} ${req.url}`);
  next();
});
app.use(bodyParser.json());

// RUTAS
app.use('/api/customers', customerRoutes);
app.use('/api/reports', reportRoutes);

// Endpoint para carga de CSV (CORREGIDO)
// Reemplaza el endpoint de carga CSV con:
app.post('/api/upload-csv', csvUpload, async (req, res) => {
  try {
    if (!req.files || !req.files.csv) {
      return res.status(400).json({
        success: false,
        error: 'No se subió ningún archivo CSV'
      });
    }
    
    const csvString = req.files.csv.data.toString('utf8');
    const result = await loadCSVData(csvString);
    
    // Respuesta simplificada sin strictContentLength
    res.setHeader('Content-Type', 'application/json');
    res.write(JSON.stringify({
      success: true,
      message: result.message
    }));
    res.end();
  } catch (error) {
    console.error('Error en carga CSV:', error);
    
    // Respuesta de error simplificada
    res.setHeader('Content-Type', 'application/json');
    res.status(500);
    res.write(JSON.stringify({
      success: false,
      error: 'Error procesando CSV: ' + error.message
    }));
    res.end();
  }
});

// Servir el frontend (si está en la misma instancia)
app.use(express.static('../frontend/public'));

// Ruta de prueba para verificar el backend
app.get('/api/health', (req, res) => {
  res.json({
    success: true,
    message: 'Backend is running',
    timestamp: new Date()
  });
});

// Manejar rutas no encontradas
app.use((req, res) => {
  res.status(404).json({
    success: false,
    error: 'Ruta no encontrada'
  });
});

// Manejo de errores global
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json({
    success: false,
    error: 'Error interno del servidor'
  });
});

// Iniciar el servidor
app.listen(PORT, () => {
  console.log(`Servidor backend corriendo en http://localhost:${PORT}`);
});

// Probar la conexión a la base de datos
db.getConnection()
  .then(connection => {
    console.log('✅ Conexión a MySQL exitosa!');
    connection.release();
  })
  .catch(error => {
    console.error('❌ Error de conexión a MySQL:', error.message);
  });

```
```

// Reemplaza TODO el contenido de csvLoader.js con este código:
import db from '../config/db.js';
import { parse } from 'csv-parse/sync';

export async function loadCSVData(csvString) {
  try {
    const records = parse(csvString, {
      columns: true,
      skip_empty_lines: true,
      delimiter: ',',
      trim: true,
      skip_records_with_error: true,
      cast: (value, context) => {
        if (!value || value.trim() === '') return null;
        
        if (context.column.includes('Monto') || 
            context.column.includes('Total') || 
            context.column.includes('amount')) {
          return parseFloat(value.replace(/[^0-9.-]/g, ''));
        }
        
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

    const connection = await db.getConnection();
    await connection.beginTransaction();

    try {
      // 1. Cargar clientes
      const customerMap = new Map();
      for (const record of records) {
        const email = record['Correo Electrónico'];
        if (!customerMap.has(email)) {
          const [result] = await connection.execute(
            `INSERT INTO customers (name, email, identification_number, address, phone)
             VALUES (?, ?, ?, ?, ?)
             ON DUPLICATE KEY UPDATE
               name = VALUES(name),
               identification_number = VALUES(identification_number),
               address = VALUES(address),
               phone = VALUES(phone)`,
            [
              record['Nombre del Cliente'],
              email,
              record['Número de Identificación'],
              record['Dirección'],
              record['Teléfono']
            ]
          );
          customerMap.set(email, result.insertId || result.affectedRows);
        }
      }

      // 2. Cargar plataformas
      const platformMap = new Map();
      const uniquePlatforms = [...new Set(records.map(r => r['Plataforma Utilizada']))];
      for (const platformName of uniquePlatforms) {
        const [result] = await connection.execute(
          `INSERT INTO platforms (name) VALUES (?) 
           ON DUPLICATE KEY UPDATE name = VALUES(name)`,
          [platformName]
        );
        platformMap.set(platformName, result.insertId);
      }

      // 3. Cargar facturas
      const invoiceMap = new Map();
      for (const record of records) {
        const invoiceNumber = record['Número de Factura'];
        if (!invoiceMap.has(invoiceNumber)) {
          const customerEmail = record['Correo Electrónico'];
          const customerId = customerMap.get(customerEmail);
          
          const [invoiceResult] = await connection.execute(
            `INSERT INTO invoices (
              invoice_number, 
              customer_id, 
              billing_period, 
              total_amount, 
              status
            ) VALUES (?, ?, ?, ?, ?)
            ON DUPLICATE KEY UPDATE
              customer_id = VALUES(customer_id),
              total_amount = VALUES(total_amount)`,
            [
              invoiceNumber,
              customerId,
              record['Período Facturado'],
              parseFloat(record['Total Factura']),
              'pending'
            ]
          );
          
          const invoiceId = invoiceResult.insertId || (
            await connection.query(
              'SELECT invoice_id FROM invoices WHERE invoice_number = ?',
              [invoiceNumber]
            )
          )[0][0].invoice_id;
          
          invoiceMap.set(invoiceNumber, invoiceId);
        }
      }

      // 4. Cargar transacciones
      for (const record of records) {
        const platformId = platformMap.get(record['Plataforma Utilizada']);
        const invoiceId = invoiceMap.get(record['Número de Factura']);
        
        await connection.execute(
          `INSERT INTO transactions (
            transaction_id,
            invoice_id,
            platform_id,
            amount,
            transaction_date,
            status
          ) VALUES (?, ?, ?, ?, ?, ?)
          ON DUPLICATE KEY UPDATE
            amount = VALUES(amount),
            status = VALUES(status)`,
          [
            record['ID de Transacción'],
            invoiceId,
            platformId,
            parseFloat(record['Monto']),
            new Date(record['Fecha de Transacción']),
            record['Estado']
          ]
        );
      }

      await connection.commit();
      return { 
        success: true, 
        message: `${records.length} registros procesados correctamente` 
      };
    } catch (error) {
      await connection.rollback();
      console.error('Error en transacción:', error);
      throw new Error(`Error en base de datos: ${error.message}`);
    } finally {
      connection.release();
    }
  } catch (error) {
    console.error('Error en loadCSVData:', error);
    throw new Error(`Error al procesar CSV: ${error.message}`);
  }
}
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
