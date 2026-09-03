# Sistema de Control de Inventario y Ventas para Accesorios Electrónicos - "Caseritos"

## 1. Descripción del Sistema

El **Sistema de Control para Accesorios Electrónicos ("Caseritos")** es una solución integral diseñada para gestionar el ciclo de vida completo de los productos comercializados en la tienda: desde la adquisición y recepción con proveedores, pasando por el control de inventarios por volumen y seriados (IMEI), hasta la venta final al cliente.

### Requerimientos Clave:
* **Gestión Dual de Inventario:**
  * **Accesorios, Fundas y Audífonos:** Se gestionan mediante control de stock por **volumen**.
  * **Teléfonos Celulares:** Al ser productos de alto valor con trazabilidad y garantía obligatoria, se gestionan de manera individual mediante su número de serie único (**IMEI**).
* **Gestión de Compras:** Registro de órdenes vinculadas a un **proveedor único por compra**, incrementando automáticamente el stock y generando registros individuales de IMEI para teléfonos.
* **Ventas Dinámicas y Descuentos:**
  * Soporte para **precios dinámicos** y **descuentos por volumen** (ej. precios mayoreo según cantidad).
  * Fijación y congelamiento del precio final negociado directamente en la línea de venta (`DETALLE_VENTA`).
  * Selección explícita de IMEI disponible para la venta de teléfonos.
  * Disminución automática de stock e historial de transacciones según método de pago.

---

## 2. Suposiciones del Negocio

1. **Proveedores por Compra:** Cada orden de compra se vincula de manera obligatoria a un único proveedor independiente.
2. **Precios Variables y Descuentos por Producto:** Los productos no manejan un precio rígido de venta. Tienen un `precio_base` referencial, pero el precio final se calcula dinámicamente mediante reglas de mayoreo (`DESCUENTO_VOLUMEN`) o negociación directa, congelándose en el detalle de la venta.
3. **Trazabilidad de Seriados (IMEI):** Toda unidad física de teléfono se registra obligatoriamente con su número de IMEI único desde su entrada (`DETALLE_COMPRA`) hasta su salida (`DETALLE_VENTA` / estado final).

---

## 3. Modelo de Datos (Entidades y Atributos)

### A. Catálogo e Inventario

#### `CATEGORIA`
Define la clasificación principal de los productos comercializados.

| Atributo | Tipo de Dato | Modificadores | Descripción |
| :--- | :--- | :--- | :--- |
| `id_categoria` | `INTEGER` | `PRIMARY KEY AUTOINCREMENT` | Identificador único de la categoría |
| `nombre` | `VARCHAR(50)` | `NOT NULL`, `CHECK (nombre IN ('Teléfonos', 'Fundas', 'Audífonos', 'Accesorios'))` | Nombre de la categoría |
| `descripcion` | `TEXT` | `NULL` | Descripción opcional |

#### `PRODUCTO`
Catálogo general de artículos ofrecidos en la tienda.

| Atributo | Tipo de Dato | Modificadores | Descripción |
| :--- | :--- | :--- | :--- |
| `id_producto` | `INTEGER` | `PRIMARY KEY AUTOINCREMENT` | Identificador único del producto |
| `id_categoria` | `INTEGER` | `FOREIGN KEY → CATEGORIA(id_categoria)` | Categoría a la que pertenece |
| `nombre` | `VARCHAR(100)` | `NOT NULL` | Nombre comercial del producto |
| `marca` | `VARCHAR(50)` | `NOT NULL` | Marca del fabricante |
| `modelo` | `VARCHAR(50)` | `NULL` | Modelo específico |
| `precio_base` | `DECIMAL(10,2)`| `NOT NULL`, `CHECK (precio_base >= 0)` | Precio de lista referencial |
| `stock_disponible`| `INTEGER` | `DEFAULT 0`, `CHECK (stock_disponible >= 0)` | Cantidad global disponible (para accesorios) |

#### `DESCUENTO_VOLUMEN`
Reglas dinámicas de mayoreo por volumen de compra.

| Atributo | Tipo de Dato | Modificadores | Descripción |
| :--- | :--- | :--- | :--- |
| `id_descuento` | `INTEGER` | `PRIMARY KEY AUTOINCREMENT` | Identificador único del descuento |
| `id_producto` | `INTEGER` | `FOREIGN KEY → PRODUCTO(id_producto)` | Producto aplicable |
| `cantidad_minima`| `INTEGER` | `NOT NULL`, `CHECK (cantidad_minima > 1)` | Cantidad a partir de la cual aplica el precio |
| `precio_especial`| `DECIMAL(10,2)`| `NOT NULL`, `CHECK (precio_especial >= 0)` | Precio unitario con el descuento aplicado |

#### `ITEM_SERIADO`
Trazabilidad individualizada por IMEI para teléfonos y artículos con garantía.

| Atributo | Tipo de Dato | Modificadores | Descripción |
| :--- | :--- | :--- | :--- |
| `id_item` | `INTEGER` | `PRIMARY KEY AUTOINCREMENT` | Identificador único del ítem físico |
| `id_producto` | `INTEGER` | `FOREIGN KEY → PRODUCTO(id_producto)` | Producto asociado |
| `id_detalle_compra`| `INTEGER` | `FOREIGN KEY → DETALLE_COMPRA(id_detalle_compra)` | Referencia del lote/compra de ingreso |
| `imei_serial` | `VARCHAR(50)` | `UNIQUE`, `NOT NULL` | Código IMEI o número de serie |
| `estado` | `VARCHAR(20)` | `DEFAULT 'disponible'`, `CHECK (estado IN ('disponible', 'vendido', 'en_garantia', 'defectuoso'))` | Estado actual de la unidad |

---

### B. Registro de Compras (Entradas de Stock)

#### `PROVEEDOR`
Registro de empresas o distribuidores de suministros.

| Atributo | Tipo de Dato | Modificadores | Descripción |
| :--- | :--- | :--- | :--- |
| `id_proveedor` | `INTEGER` | `PRIMARY KEY AUTOINCREMENT` | Identificador del proveedor |
| `nit_ruc` | `VARCHAR(20)` | `UNIQUE`, `NOT NULL` | Número de identificación tributaria / RUC |
| `razon_social` | `VARCHAR(100)` | `NOT NULL` | Nombre o razón social legal |
| `telefono` | `VARCHAR(20)` | `NULL` | Teléfono de contacto |
| `email` | `VARCHAR(150)` | `NULL` | Correo electrónico de contacto |

#### `COMPRA`
Cabecera de la orden o transacción de abastecimiento.

| Atributo | Tipo de Dato | Modificadores | Descripción |
| :--- | :--- | :--- | :--- |
| `id_compra` | `INTEGER` | `PRIMARY KEY AUTOINCREMENT` | Identificador de la compra |
| `id_proveedor` | `INTEGER` | `FOREIGN KEY → PROVEEDOR(id_proveedor)`, `NOT NULL` | Proveedor emisor |
| `fecha_compra` | `TIMESTAMP` | `DEFAULT CURRENT_TIMESTAMP` | Fecha y hora de la transacción |
| `numero_nota_venta`| `VARCHAR(50)`| `NULL` | Número de factura/nota emitida por proveedor |
| `monto_total` | `DECIMAL(10,2)`| `NOT NULL`, `CHECK (monto_total >= 0)` | Monto total acumulado de la compra |

#### `DETALLE_COMPRA`
Entidad asociativa entre `COMPRA` y `PRODUCTO` para detallar las unidades adquiridas.

| Atributo | Tipo de Dato | Modificadores | Descripción |
| :--- | :--- | :--- | :--- |
| `id_detalle_compra`| `INTEGER` | `PRIMARY KEY AUTOINCREMENT` | Identificador de la línea |
| `id_compra` | `INTEGER` | `FOREIGN KEY → COMPRA(id_compra)` | Compra a la que pertenece |
| `id_producto` | `INTEGER` | `FOREIGN KEY → PRODUCTO(id_producto)` | Producto comprado |
| `cantidad` | `INTEGER` | `NOT NULL`, `CHECK (cantidad > 0)` | Unidades compradas |
| `costo_unitario`| `DECIMAL(10,2)`| `NOT NULL`, `CHECK (costo_unitario >= 0)` | Costo de adquisición por unidad |

---

### C. Registro de Ventas (Salidas de Stock)

#### `VENTA`
Cabecera del comprobante / factura emitida al cliente.

| Atributo | Tipo de Dato | Modificadores | Descripción |
| :--- | :--- | :--- | :--- |
| `id_venta` | `INTEGER` | `PRIMARY KEY AUTOINCREMENT` | Identificador de la factura/venta |
| `fecha_venta` | `TIMESTAMP` | `DEFAULT CURRENT_TIMESTAMP` | Fecha y hora de emisión |
| `monto_total` | `DECIMAL(10,2)`| `NOT NULL`, `CHECK (monto_total >= 0)` | Total cobrado |
| `metodo_pago` | `VARCHAR(20)` | `NOT NULL`, `CHECK (metodo_pago IN ('efectivo', 'tarjeta', 'transferencia'))` | Forma de pago utilizada |

#### `DETALLE_VENTA`
Línea de detalle que congela el precio pactado y relaciona productos o IMEI específicos vendibles.

| Atributo | Tipo de Dato | Modificadores | Descripción |
| :--- | :--- | :--- | :--- |
| `id_detalle_venta`| `INTEGER` | `PRIMARY KEY AUTOINCREMENT` | Identificador del detalle |
| `id_venta` | `INTEGER` | `FOREIGN KEY → VENTA(id_venta)` | Transacción de venta |
| `id_producto` | `INTEGER` | `FOREIGN KEY → PRODUCTO(id_producto)` | Producto vendido |
| `id_item` | `INTEGER` | `FOREIGN KEY → ITEM_SERIADO(id_item)`, `NULL` | Ítem físico/IMEI vendido (si aplica) |
| `cantidad` | `INTEGER` | `NOT NULL`, `CHECK (cantidad > 0)` | Cantidad vendida (siempre 1 para seriados) |
| `precio_unitario`| `DECIMAL(10,2)`| `NOT NULL`, `CHECK (precio_unitario >= 0)` | Precio acordado/congelado de la transacción |
| `subtotal` | `DECIMAL(10,2)`| `NOT NULL`, `CHECK (subtotal >= 0)` | Resultado de `cantidad * precio_unitario` |

---

## 4. Script DDl en SQL (ANSI Standard / PostgreSQL / SQLite Compatible)

```sql
-- TABLA: CATEGORIA
CREATE TABLE CATEGORIA (
    id_categoria INTEGER PRIMARY KEY AUTOINCREMENT,
    nombre VARCHAR(50) NOT NULL CHECK (nombre IN ('Teléfonos', 'Fundas', 'Audífonos', 'Accesorios')),
    descripcion TEXT
);

-- TABLA: PRODUCTO
CREATE TABLE PRODUCTO (
    id_producto INTEGER PRIMARY KEY AUTOINCREMENT,
    id_categoria INTEGER NOT NULL,
    nombre VARCHAR(100) NOT NULL,
    marca VARCHAR(50) NOT NULL,
    modelo VARCHAR(50),
    precio_base DECIMAL(10,2) NOT NULL CHECK (precio_base >= 0),
    stock_disponible INTEGER DEFAULT 0 CHECK (stock_disponible >= 0),
    FOREIGN KEY (id_categoria) REFERENCES CATEGORIA(id_categoria)
);

-- TABLA: DESCUENTO_VOLUMEN
CREATE TABLE DESCUENTO_VOLUMEN (
    id_descuento INTEGER PRIMARY KEY AUTOINCREMENT,
    id_producto INTEGER NOT NULL,
    cantidad_minima INTEGER NOT NULL CHECK (cantidad_minima > 1),
    precio_especial DECIMAL(10,2) NOT NULL CHECK (precio_especial >= 0),
    FOREIGN KEY (id_producto) REFERENCES PRODUCTO(id_producto)
);

-- TABLA: PROVEEDOR
CREATE TABLE PROVEEDOR (
    id_proveedor INTEGER PRIMARY KEY AUTOINCREMENT,
    nit_ruc VARCHAR(20) UNIQUE NOT NULL,
    razon_social VARCHAR(100) NOT NULL,
    telefono VARCHAR(20),
    email VARCHAR(150)
);

-- TABLA: COMPRA
CREATE TABLE COMPRA (
    id_compra INTEGER PRIMARY KEY AUTOINCREMENT,
    id_proveedor INTEGER NOT NULL,
    fecha_compra TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    numero_nota_venta VARCHAR(50),
    monto_total DECIMAL(10,2) NOT NULL CHECK (monto_total >= 0),
    FOREIGN KEY (id_proveedor) REFERENCES PROVEEDOR(id_proveedor)
);

-- TABLA: DETALLE_COMPRA
CREATE TABLE DETALLE_COMPRA (
    id_detalle_compra INTEGER PRIMARY KEY AUTOINCREMENT,
    id_compra INTEGER NOT NULL,
    id_producto INTEGER NOT NULL,
    cantidad INTEGER NOT NULL CHECK (cantidad > 0),
    costo_unitario DECIMAL(10,2) NOT NULL CHECK (costo_unitario >= 0),
    FOREIGN KEY (id_compra) REFERENCES COMPRA(id_compra),
    FOREIGN KEY (id_producto) REFERENCES PRODUCTO(id_producto)
);

-- TABLA: ITEM_SERIADO (IMEI)
CREATE TABLE ITEM_SERIADO (
    id_item INTEGER PRIMARY KEY AUTOINCREMENT,
    id_producto INTEGER NOT NULL,
    id_detalle_compra INTEGER NOT NULL,
    imei_serial VARCHAR(50) UNIQUE NOT NULL,
    estado VARCHAR(20) DEFAULT 'disponible' CHECK (estado IN ('disponible', 'vendido', 'en_garantia', 'defectuoso')),
    FOREIGN KEY (id_producto) REFERENCES PRODUCTO(id_producto),
    FOREIGN KEY (id_detalle_compra) REFERENCES DETALLE_COMPRA(id_detalle_compra)
);

-- TABLA: VENTA
CREATE TABLE VENTA (
    id_venta INTEGER PRIMARY KEY AUTOINCREMENT,
    fecha_venta TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    monto_total DECIMAL(10,2) NOT NULL CHECK (monto_total >= 0),
    metodo_pago VARCHAR(20) NOT NULL CHECK (metodo_pago IN ('efectivo', 'tarjeta', 'transferencia'))
);

-- TABLA: DETALLE_VENTA
CREATE TABLE DETALLE_VENTA (
    id_detalle_venta INTEGER PRIMARY KEY AUTOINCREMENT,
    id_venta INTEGER NOT NULL,
    id_producto INTEGER NOT NULL,
    id_item INTEGER,
    cantidad INTEGER NOT NULL CHECK (cantidad > 0),
    precio_unitario DECIMAL(10,2) NOT NULL CHECK (precio_unitario >= 0),
    subtotal DECIMAL(10,2) NOT NULL CHECK (subtotal >= 0),
    FOREIGN KEY (id_venta) REFERENCES VENTA(id_venta),
    FOREIGN KEY (id_producto) REFERENCES PRODUCTO(id_producto),
    FOREIGN KEY (id_item) REFERENCES ITEM_SERIADO(id_item)
);
