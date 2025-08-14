# codes_correction
SELECT COUNT(*) FROM customers;
SELECT * FROM customers ORDER BY id DESC LIMIT 10;

# error
```
import db from '../config/db.js';
import { parse } from 'csv-parse/sync';

export async function loadCSVData(csvString) {
  const records = parse(csvString, {
    columns: true,
    skip_empty_lines: true,
    delimiter: ',',
    trim: true,
    skip_records_with_error: true
  });

  if (records.length === 0) {
    throw new Error('El CSV está vacío o no tiene formato válido');
  }

  // Normalizar datos: reemplazar undefined por cadena vacía
  const normalizedRecords = records.map(row => {
    return {
      name: row.name ?? '',
      email: row.email ?? '',
      phone: row.phone ?? '',
      address: row.address ?? '',
      identification_number: row.identification_number ?? '',
      role: row.role ?? ''
    };
  });

  const insertQuery = `
    INSERT INTO customers
      (name, email, phone, address, identification_number, role)
    VALUES (?, ?, ?, ?, ?, ?)
  `;

  // Insertar fila por fila
  for (const record of normalizedRecords) {
    await db.execute(insertQuery, [
      record.name,
      record.email,
      record.phone,
      record.address,
      record.identification_number,
      record.role
    ]);
  }

  return { message: `Se insertaron ${normalizedRecords.length} registros correctamente` };
}

```
# ddl.sql
```
CREATE DATABASE IF NOT EXISTS pd_andres_covaleda_gosling;
USE pd_andres_covaleda_gosling;

-- Tabla de clientes
CREATE TABLE IF NOT EXISTS customers (
    customer_id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    identification_number VARCHAR(50) NOT NULL,
    address TEXT,
    phone VARCHAR(50)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Tabla de plataformas
CREATE TABLE IF NOT EXISTS platforms (
    platform_id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50) NOT NULL UNIQUE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Tabla de facturas
CREATE TABLE IF NOT EXISTS invoices (
    invoice_id INT AUTO_INCREMENT PRIMARY KEY,
    invoice_number VARCHAR(50) NOT NULL UNIQUE,
    customer_id INT NOT NULL,
    billing_period VARCHAR(20) NOT NULL,
    total_amount DECIMAL(12, 2) NOT NULL,
    status ENUM('pending', 'partial', 'paid') DEFAULT 'pending',
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Tabla de transacciones
CREATE TABLE IF NOT EXISTS transactions (
    transaction_id VARCHAR(50) PRIMARY KEY,
    invoice_id INT NOT NULL,
    platform_id INT NOT NULL,
    amount DECIMAL(12, 2) NOT NULL,
    transaction_date DATETIME NOT NULL,
    status ENUM('Pendiente', 'Completada', 'Fallida') NOT NULL,
    FOREIGN KEY (invoice_id) REFERENCES invoices(invoice_id),
    FOREIGN KEY (platform_id) REFERENCES platforms(platform_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- Índices para mejorar rendimiento
CREATE INDEX idx_invoices_customer ON invoices(customer_id);
CREATE INDEX idx_transactions_invoice ON transactions(invoice_id);
CREATE INDEX idx_transactions_platform ON transactions(platform_id);

```

# csvLoader.js
```
// backend/src/services/csvLoader.js
import db from '../config/db.js';
import { parse } from 'csv-parse/sync';

export async function loadCSVData(csvString) {
  const records = parse(csvString, {
    columns: true,
    skip_empty_lines: true,
    delimiter: ',',
    trim: true,
    skip_records_with_error: true
  });

  if (records.length === 0) {
    throw new Error('El archivo CSV está vacío o no contiene datos válidos');
  }

  const connection = await db.getConnection();
  await connection.beginTransaction();

  try {
    // 1. Cargar clientes (existente)
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

    // 2. Cargar plataformas (existente)
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

    // 3. Cargar facturas (NUEVO)
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
            record['Período Facturado'], // Nueva columna requerida
            parseFloat(record['Total Factura']), // Nueva columna
            'pending' // Valor por defecto
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

    // 4. Cargar transacciones (NUEVO)
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
          record['ID de Transacción'], // Nueva columna
          invoiceId,
          platformId,
          parseFloat(record['Monto']), // Nueva columna
          new Date(record['Fecha de Transacción']), // Nueva columna
          record['Estado'] // Nueva columna
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
    
    // Registrar error detallado
    console.error('Error en transacción:', error);
    
    throw new Error(`Error en base de datos: ${error.message}`);
  } finally {
    connection.release();
  }
}
```
# db.js
```
ALTER TABLE customers 
MODIFY identification_number VARCHAR(50) DEFAULT '';
```
# app.js
```
// backend/src/App.js
import express from 'express';
import bodyParser from 'body-parser';
import dotenv from 'dotenv';
import cors from 'cors';
import customerRoutes from './routes/customerRoutes.js';
import reportRoutes from './routes/reportRoutes.js';
import { csvUpload } from './utils/uploadMiddleware.js';
import { loadCSVData } from './services/csvLoader.js';
import db from './config/db.js';

dotenv.config();
const app = express();
const PORT = process.env.PORT || 3000;

// Configuración de CORS
const corsOptions = {
  origin: [
    'http://localhost:5500',
    'http://127.0.0.1:5500',
    'http://localhost:8080',
    'http://127.0.0.1:8080'
  ],
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  credentials: true,
  optionsSuccessStatus: 200
};

app.use(cors(corsOptions));
app.use(bodyParser.json());

// Rutas
app.use('/api/customers', customerRoutes);
app.use('/api/reports', reportRoutes);

// Endpoint para carga de CSV
app.post('/api/upload-csv', csvUpload, async (req, res) => {
  try {
    if (!req.file) {
      return res.status(400).json({
        success: false,
        error: 'No se subió ningún archivo CSV'
      });
    }

    const csvString = req.file.buffer.toString('utf8');
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

// Servir frontend
app.use(express.static('../frontend/public'));

// Ruta de prueba
app.get('/api/health', (req, res) => {
  res.json({ success: true, message: 'Backend is running', timestamp: new Date() });
});

// Manejo de rutas no encontradas
app.use((req, res) => {
  res.status(404).json({ success: false, error: 'Ruta no encontrada' });
});

// Manejo de errores global
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json({ success: false, error: 'Error interno del servidor' });
});

// Iniciar servidor
app.listen(PORT, () => {
  console.log(`Servidor backend corriendo en http://localhost:${PORT}`);
});

// Probar conexión a DB
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
import mysql from 'mysql2/promise';
import dotenv from 'dotenv';

dotenv.config();

console.log('Configurando conexión a DB:', {
  host: process.env.DB_HOST,
  user: process.env.DB_USER,
  database: process.env.DB_NAME
});

const pool = mysql.createPool({
  host: process.env.DB_HOST || 'localhost',
  user: process.env.DB_USER || 'root',
  password: process.env.DB_PASSWORD || 'Qwe.123*',
  database: process.env.DB_NAME || 'pd_andres_covaleda_gosling',
  waitForConnections: true,
  connectionLimit: 10,
  queueLimit: 0
});
// Agregar prueba de conexión
pool.getConnection()
  .then(connection => {
    console.log(' Connected to MySQL database!');
    connection.release();
  })
  .catch(error => {
    console.error(' Database connection failed:', error.message);
    console.log('Please check:');
    console.log('1. Is MySQL running?');
    console.log('2. Check DB credentials in .env');
    console.log('3. Verify database exists');
    process.exit(1); // Salir si no hay conexión a DB
  });
export default pool;
```
# customerController.js
```
import db from '../config/db.js';

// CREATE
export const createCustomer = async (req, res) => {
  const { name, email } = req.body;
  
  if (!name || !email) {
    return res.status(400).json({ 
      success: false,
      error: 'Nombre y email son requeridos' 
    });
  }

  try {
    const [result] = await db.execute(
      'INSERT INTO customers (name, email) VALUES (?, ?)',
      [name, email]
    );
    
    res.status(201).json({
      success: true,
      message: 'Cliente creado',
      data: {
        id: result.insertId,
        name,
        email
      }
    });
  } catch (error) {
    console.error('Error creando cliente:', error);
    res.status(500).json({
      success: false,
      error: 'Error de base de datos: ' + error.message
    });
  }
};

// READ
// En todos los controladores, manejar errores y siempre responder
export const getAllCustomers = async (req, res) => {
  try {
    const [customers] = await db.query('SELECT * FROM customers');
    // Siempre enviar estructura JSON válida
    res.json({
      success: true,
      data: customers
    });
  } catch (error) {
    // Respuesta de error también en formato JSON
    res.status(500).json({
      success: false,
      error: "Database error"
    });
  }
};

// UPDATE
export const updateCustomer = async (req, res) => {
  const { id } = req.params;
  const { name, email } = req.body;

  if (!name || !email) {
    return res.status(400).json({
      success: false,
      error: 'Nombre y email son requeridos'
    });
  }

  try {
    const [result] = await db.execute(
      'UPDATE customers SET name = ?, email = ? WHERE customer_id = ?',
      [name, email, id]
    );
    
    if (result.affectedRows === 0) {
      return res.status(404).json({
        success: false,
        error: 'Cliente no encontrado'
      });
    }
    
    res.status(200).json({
      success: true,
      message: 'Cliente actualizado',
      data: { id, name, email }
    });
  } catch (error) {
    console.error('Error actualizando cliente:', error);
    res.status(500).json({
      success: false,
      error: 'Error de base de datos: ' + error.message
    });
  }
};

// DELETE
export const deleteCustomer = async (req, res) => {
  const { id } = req.params;

  try {
    const [result] = await db.execute(
      'DELETE FROM customers WHERE customer_id = ?',
      [id]
    );
    
    if (result.affectedRows === 0) {
      return res.status(404).json({
        success: false,
        error: 'Cliente no encontrado'
      });
    }
    
    res.status(200).json({
      success: true,
      message: 'Cliente eliminado'
    });
  } catch (error) {
    console.error('Error eliminando cliente:', error);
    res.status(500).json({
      success: false,
      error: 'Error de base de datos: ' + error.message
    });
  }
};
```
# reportController.js
```
import db from '../config/db.js';

// 1. Total pagado por cliente
export const getTotalPaidByCustomer = async (req, res) => {
  try {
    const [results] = await db.query(`
      SELECT 
        c.customer_id,
        c.name,
        c.email,
        SUM(t.amount) AS total_paid
      FROM customers c
      JOIN invoices i ON c.customer_id = i.customer_id
      JOIN transactions t ON i.invoice_id = t.invoice_id
      WHERE t.status = 'Completada'
      GROUP BY c.customer_id
    `);
    res.json(results);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

// 2. Facturas pendientes con cliente y transacción
export const getPendingInvoices = async (req, res) => {
  try {
    const [results] = await db.query(`
      SELECT 
        i.invoice_id,
        i.invoice_number,
        i.total_amount,
        i.status AS invoice_status,
        c.name AS customer_name,
        c.email,
        t.transaction_id,
        t.amount AS transaction_amount,
        t.status AS transaction_status
      FROM invoices i
      JOIN customers c ON i.customer_id = c.customer_id
      LEFT JOIN transactions t ON i.invoice_id = t.invoice_id
      WHERE i.status IN ('pending', 'partial')
    `);
    res.json(results);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

// 3. Transacciones por plataforma
export const getTransactionsByPlatform = async (req, res) => {
  const { platform } = req.query;
  if (!platform) {
    return res.status(400).json({ error: "Platform name is required" });
  }

  try {
    const [results] = await db.query(`
      SELECT 
        t.transaction_id,
        t.amount,
        t.transaction_date,
        t.status,
        p.name AS platform_name,
        c.name AS customer_name,
        i.invoice_number
      FROM transactions t
      JOIN platforms p ON t.platform_id = p.platform_id
      JOIN invoices i ON t.invoice_id = i.invoice_id
      JOIN customers c ON i.customer_id = c.customer_id
      WHERE p.name = ?
    `, [platform]);
    res.json(results);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};
```
# auth.js
```
// backend/src/middleware/auth.js
import jwt from 'jsonwebtoken';

export const authenticate = (req, res, next) => {
  const token = req.headers.authorization?.split(' ')[1];
  
  if (!token) return res.status(401).json({ error: 'Acceso no autorizado' });

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    res.status(401).json({ error: 'Token inválido' });
  }
};
```
# csvLoader.js
```
// backend/src/services/csvLoader.js
import db from '../config/db.js';
import { parse } from 'csv-parse/sync';

export async function loadCSVData(csvString) {
  const records = parse(csvString, {
    columns: true,
    skip_empty_lines: true,
    delimiter: ',',
    trim: true,
    skip_records_with_error: true
  });

  const connection = await db.getConnection();
  try {
    await connection.beginTransaction();

    // Mapas para evitar búsquedas repetidas
    const customerMap = new Map();
    const platformMap = new Map();

    // ======================
    // 1. Cargar clientes
    // ======================
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

        let customerId;
        if (result.insertId && result.insertId > 0) {
          customerId = result.insertId; // Nuevo cliente
        } else {
          const [[existing]] = await connection.query(
            'SELECT customer_id FROM customers WHERE email = ?',
            [email]
          );
          customerId = existing.customer_id; // Cliente existente
        }

        customerMap.set(email, customerId);
      }
    }

    // ======================
    // 2. Cargar plataformas
    // ======================
    for (const record of records) {
      const platformName = record['Plataforma'];
      if (!platformMap.has(platformName)) {
        const [result] = await connection.execute(
          `INSERT INTO platforms (name)
           VALUES (?)
           ON DUPLICATE KEY UPDATE
             name = VALUES(name)`,
          [platformName]
        );

        let platformId;
        if (result.insertId && result.insertId > 0) {
          platformId = result.insertId; // Nueva plataforma
        } else {
          const [[existing]] = await connection.query(
            'SELECT platform_id FROM platforms WHERE name = ?',
            [platformName]
          );
          platformId = existing.platform_id; // Plataforma existente
        }

        platformMap.set(platformName, platformId);
      }
    }

    // ======================
    // 3. Cargar facturas y transacciones
    // ======================
    for (const record of records) {
      const customerId = customerMap.get(record['Correo Electrónico']);
      const platformId = platformMap.get(record['Plataforma']);

      // Insertar/actualizar factura
      const [invoiceResult] = await connection.execute(
        `INSERT INTO invoices (invoice_number, customer_id, platform_id, invoice_date, total_amount)
         VALUES (?, ?, ?, ?, ?)
         ON DUPLICATE KEY UPDATE
           customer_id = VALUES(customer_id),
           platform_id = VALUES(platform_id),
           invoice_date = VALUES(invoice_date),
           total_amount = VALUES(total_amount)`,
        [
          record['Número de Factura'],
          customerId,
          platformId,
          record['Fecha de Factura'],
          record['Total']
        ]
      );

      let invoiceId;
      if (invoiceResult.insertId && invoiceResult.insertId > 0) {
        invoiceId = invoiceResult.insertId; // Nueva factura
      } else {
        const [[existingInvoice]] = await connection.query(
          'SELECT invoice_id FROM invoices WHERE invoice_number = ?',
          [record['Número de Factura']]
        );
        invoiceId = existingInvoice.invoice_id; // Factura existente
      }

      // Insertar transacción asociada a la factura
      await connection.execute(
        `INSERT INTO transactions (invoice_id, payment_method, payment_date, amount)
         VALUES (?, ?, ?, ?)
         ON DUPLICATE KEY UPDATE
           payment_method = VALUES(payment_method),
           payment_date = VALUES(payment_date),
           amount = VALUES(amount)`,
        [
          invoiceId,
          record['Método de Pago'],
          record['Fecha de Pago'],
          record['Monto Pagado']
        ]
      );
    }

    await connection.commit();
    return { success: true, message: 'Datos cargados y actualizados correctamente' };
  } catch (error) {
    await connection.rollback();
    throw error;
  } finally {
    connection.release();
  }
}

```
reportRoutes.js
```
import express from 'express';
import {
  getTotalPaidByCustomer,
  getPendingInvoices,
  getTransactionsByPlatform
} from '../controllers/reportController.js';

const router = express.Router();

// Consulta 1: Total pagado por cliente
router.get('/total-paid', getTotalPaidByCustomer);

// Consulta 2: Facturas pendientes
router.get('/pending-invoices', getPendingInvoices);

// Consulta 3: Transacciones por plataforma
router.get('/transactions-by-platform', getTransactionsByPlatform);

export default router;
```
# csvLoader.js
```

            invoiceNumber,
            customerId,
            record['Período Facturado'], // Nueva columna requerida
            parseFloat(record['Total Factura']), // Nueva columna
            'pending' // Valor por defecto
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

    // 4. Cargar transacciones (NUEVO)
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
          record['ID de Transacción'], // Nueva columna
          invoiceId,
          platformId,
          parseFloat(record['Monto']), // Nueva columna
          new Date(record['Fecha de Transacción']), // Nueva columna
          record['Estado'] // Nueva columna
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
    
    // Registrar error detallado
    console.error('Error en transacción:', error);
    
    throw new Error(`Error en base de datos: ${error.message}`);
  } finally {
    connection.release();
  }
}
```
# uploadMiddleware.js
```
import multer from 'multer';

const storage = multer.memoryStorage();
const upload = multer({
  storage,
  fileFilter: (req, file, cb) => {
    const allowedMimes = [
      'text/csv',
      'application/vnd.ms-excel',
      'application/csv',
      'text/x-csv',
      'application/x-csv'
    ];
    
    if (allowedMimes.includes(file.mimetype)) {
      cb(null, true);
    } else {
      cb(new Error('Tipo de archivo inválido. Solo se permiten archivos CSV'), false);
    }
  },
  limits: { fileSize: 20 * 1024 * 1024 } // 20MB
});

export const csvUpload = upload.single('csv');
```
# app.js
```
import express from 'express';
import bodyParser from 'body-parser';
import fileUpload from 'express-fileupload';
import dotenv from 'dotenv';
import cors from 'cors';
import customerRoutes from './routes/customerRoutes.js';
import reportRoutes from './routes/reportRoutes.js';
import { csvUpload } from './utils/uploadMiddleware.js';
import { loadCSVData } from './services/csvLoader.js';
import db from './config/db.js';

dotenv.config();
const app = express();
const PORT = process.env.PORT || 3000;

// Configuración de CORS
const corsOptions = {
  origin: [
    'http://localhost:5500',
    'http://127.0.0.1:5500',
    'http://localhost:8080',
    'http://127.0.0.1:8080'
  ],
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  credentials: true,
  optionsSuccessStatus: 200
};

app.use(cors(corsOptions));

// Middleware para parsear JSON
app.use(bodyParser.json());

// Middleware para manejar carga de archivos
app.use(fileUpload());

// Rutas
app.use('/api/customers', customerRoutes);
app.use('/api/reports', reportRoutes);

// Endpoint para carga de CSV
app.post('/api/upload-csv', csvUpload, async (req, res) => {
  try {
    if (!req.file) {
      return res.status(400).json({
        success: false,
        error: 'No se subió ningún archivo CSV'
      });
    }
    
    const csvString = req.file.buffer.toString('utf8');
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
# load-data.js
```
import fs from 'fs';
import { parse } from 'csv-parse/sync';
import mysql from 'mysql2/promise';
import dotenv from 'dotenv';
import path from 'path';

// Cargar variables de entorno desde backend/.env
dotenv.config({ path: path.resolve(process.cwd(), 'backend', '.env') });

async function loadTableFromCsv(connection, filePath, tableName) {
  try {
    const csvData = fs.readFileSync(filePath, 'utf8');
    const records = parse(csvData, {
      columns: true,
      skip_empty_lines: true,
      bom: true
    });

    if (records.length === 0) {
      console.log(`⚠️  Archivo vacío: ${filePath}`);
      return;
    }

    const columns = Object.keys(records[0]);
    const placeholders = columns.map(() => '?').join(',');
    
    for (const record of records) {
      const values = columns.map(col => record[col]);
      const query = `INSERT INTO ${tableName} (${columns.join(',')}) VALUES (${placeholders})`;
      await connection.execute(query, values);
    }

    console.log(` ${tableName}: ${records.length} registros insertados`);
    return true;
  } catch (error) {
    console.error(` Error en ${tableName}: ${error.message}`);
    return false;
  }
}

async function main() {
  console.log(' Credenciales usadas:', {
    host: process.env.DB_HOST,
    user: process.env.DB_USER,
    database: process.env.DB_NAME
  });

  try {
    const connection = await mysql.createConnection({
      host: process.env.DB_HOST || 'localhost',
      user: process.env.DB_USER || 'root',
      password: process.env.DB_PASSWORD || '',
      database: process.env.DB_NAME || 'pd_andrew_vega_clan',
      multipleStatements: true
    });

    console.log('🔌 Conexión a MySQL exitosa!');
    
    // Cargar datos en orden correcto
    await loadTableFromCsv(connection, 'database/customers.csv', 'customers');
    await loadTableFromCsv(connection, 'database/platforms.csv', 'platforms');
    await loadTableFromCsv(connection, 'database/invoices.csv', 'invoices');
    await loadTableFromCsv(connection, 'database/transactions.csv', 'transactions');
    
    await connection.end();
    console.log(' Carga de datos completada!');
  } catch (error) {
    console.error(' Error de conexión:', error.message);
    console.log('Posibles soluciones:');
    console.log('1. Verifica que MySQL esté corriendo: sudo service mysql status');
    console.log('2. Confirma tus credenciales en backend/.env');
    console.log('3. Prueba conectarte manualmente: mysql -u root -p');
    console.log(`4. Crea la base de datos manualmente: CREATE DATABASE ${process.env.DB_NAME || 'pd_andrew_vega_clan'}`);
  }
}

main();

```
# main.js
```
// frontend/js/main.js
document.addEventListener('DOMContentLoaded', () => {
  const customerForm = document.getElementById('customerForm');
  const customerTable = document.querySelector('#customerTable tbody');
  const loadCsvBtn = document.getElementById('loadCsvBtn');
  
  const API_BASE_URL = 'http://localhost:3000';
  
  fetchCustomers();
  
  customerForm.addEventListener('submit', async (e) => {
    e.preventDefault();
    const customer = {
      id: document.getElementById('customerId').value,
      name: document.getElementById('name').value,
      email: document.getElementById('email').value
    };
    const method = customer.id ? 'PUT' : 'POST';
    const url = customer.id 
      ? `${API_BASE_URL}/api/customers/${customer.id}` 
      : `${API_BASE_URL}/api/customers`;
    
    try {
      const response = await fetch(url, {
        method,
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(customer)
      });
      const result = await response.json();
      customerForm.reset();
      document.getElementById('customerId').value = '';
      await fetchCustomers();
      alert(result.message || 'Operación exitosa');
    } catch (error) {
      console.error('Error en el formulario:', error);
      alert('Error: ' + error.message);
    }
  });

  // Cargar CSV
  loadCsvBtn.addEventListener('click', () => {
    const input = document.createElement('input');
    input.type = 'file';
    input.accept = '.csv';
    
    input.onchange = async (e) => {
      const file = e.target.files[0];
      if (!file) return;
      const formData = new FormData();
      formData.append('csv', file);

      try {
        loadCsvBtn.disabled = true;
        loadCsvBtn.textContent = 'Cargando...';

        const response = await fetch(`${API_BASE_URL}/api/upload-csv`, {
          method: 'POST',
          body: formData
        });

        const result = await response.json();
        alert(result.message || 'Datos cargados exitosamente');
        fetchCustomers();
      } catch (error) {
        console.error('Error cargando CSV:', error);
        alert('Error: ' + error.message);
      } finally {
        loadCsvBtn.disabled = false;
        loadCsvBtn.textContent = 'Cargar CSV';
      }
    };

    input.click();
  });

  async function fetchCustomers() {
    try {
      const response = await fetch(`${API_BASE_URL}/api/customers`);
      const result = await response.json();
      if (result.success && result.data) renderCustomers(result.data);
    } catch (error) {
      console.error('Error obteniendo clientes:', error);
      alert('Error: ' + error.message);
    }
  }

  function renderCustomers(customers) {
    customerTable.innerHTML = '';
    customers.forEach(customer => {
      const row = document.createElement('tr');
      row.innerHTML = `
        <td>${customer.customer_id}</td>
        <td>${customer.name}</td>
        <td>${customer.email}</td>
        <td class="actions">
          <button onclick="editCustomer(${customer.customer_id}, '${escapeString(customer.name)}', '${escapeString(customer.email)}')">Editar</button>
          <button onclick="deleteCustomer(${customer.customer_id})">Eliminar</button>
        </td>
      `;
      customerTable.appendChild(row);
    });
  }

  function escapeString(str) {
    return str.replace(/'/g, "\\'").replace(/"/g, '\\"');
  }

  window.editCustomer = (id, name, email) => {
    document.getElementById('customerId').value = id;
    document.getElementById('name').value = name;
    document.getElementById('email').value = email;
  };

  window.deleteCustomer = async (id) => {
    if (confirm('¿Estás seguro de eliminar este cliente?')) {
      try {
        const response = await fetch(`${API_BASE_URL}/api/customers/${id}`, { method: 'DELETE' });
        const result = await response.json();
        fetchCustomers();
        alert(result.message || 'Cliente eliminado');
      } catch (error) {
        alert('Error: ' + error.message);
      }
    }
  };
});

```
```
