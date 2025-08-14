```
// backend/src/controllers/customerController.js
import db from '../config/db.js';

export const createCustomer = async (req, res) => {
  const { name, email, identification_number = '', address = null, phone = null } = req.body;

  if (!name || !email) {
    return res.status(400).json({ success: false, error: 'Nombre y email son requeridos' });
  }

  try {
    const [result] = await db.execute(
      `INSERT INTO customers (name, email, identification_number, address, phone)
       VALUES (?, ?, ?, ?, ?)`,
      [name, email, identification_number, address, phone]
    );

    res.status(201).json({
      success: true,
      message: 'Cliente creado',
      data: {
        customer_id: result.insertId,
        name, email, identification_number, address, phone
      }
    });
  } catch (error) {
    console.error('Error creando cliente:', error);
    res.status(500).json({ success: false, error: 'Error de base de datos: ' + error.message });
  }
};

export const getAllCustomers = async (_req, res) => {
  try {
    const [rows] = await db.query(
      `SELECT customer_id, name, email, identification_number, address, phone
       FROM customers
       ORDER BY customer_id DESC`
    );
    res.json({ success: true, data: rows });
  } catch (error) {
    console.error('Error obteniendo clientes:', error);
    res.status(500).json({ success: false, error: 'Database error' });
  }
};

export const updateCustomer = async (req, res) => {
  const { id } = req.params;
  const { name, email, identification_number = '', address = null, phone = null } = req.body;

  if (!name || !email) {
    return res.status(400).json({ success: false, error: 'Nombre y email son requeridos' });
  }

  try {
    const [result] = await db.execute(
      `UPDATE customers
       SET name = ?, email = ?, identification_number = ?, address = ?, phone = ?
       WHERE customer_id = ?`,
      [name, email, identification_number, address, phone, id]
    );

    if (result.affectedRows === 0) {
      return res.status(404).json({ success: false, error: 'Cliente no encontrado' });
    }

    res.json({ success: true, message: 'Cliente actualizado' });
  } catch (error) {
    console.error('Error actualizando cliente:', error);
    res.status(500).json({ success: false, error: 'Error de base de datos: ' + error.message });
  }
};

export const deleteCustomer = async (req, res) => {
  const { id } = req.params;
  try {
    const [result] = await db.execute('DELETE FROM customers WHERE customer_id = ?', [id]);
    if (result.affectedRows === 0) {
      return res.status(404).json({ success: false, error: 'Cliente no encontrado' });
    }
    res.json({ success: true, message: 'Cliente eliminado' });
  } catch (error) {
    console.error('Error eliminando cliente:', error);
    res.status(500).json({ success: false, error: 'Error de base de datos: ' + error.message });
  }
};


```
```

```
