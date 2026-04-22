DROP DATABASE IF EXISTS panaderia; 
CREATE DATABASE panaderia;
USE panaderia;

-- ######################################################
-- 1. DEFINICIÓN DE TABLAS
-- ######################################################

SET FOREIGN_KEY_CHECKS = 0; 
DROP TABLE IF EXISTS permisos;
SET FOREIGN_KEY_CHECKS = 1;
CREATE TABLE permisos (
    id INT AUTO_INCREMENT PRIMARY KEY, 
    nombre VARCHAR(50) NOT NULL,       -- Aumentado a 50 para nombres descriptivos
    slug VARCHAR(50) NOT NULL,         -- Único: nombre técnico (ej: 'usuarios.crear')
    descripcion VARCHAR(150) NULL,     -- Aumentado a 150 para mayor detalle
    modulo VARCHAR(30) NOT NULL,       -- Para agrupar (ej: 'Ventas', 'Producción')
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    UNIQUE (slug)                      -- El slug no puede repetirse
) ENGINE=InnoDB;

SET FOREIGN_KEY_CHECKS = 0; 
DROP TABLE IF EXISTS roles;
SET FOREIGN_KEY_CHECKS = 1;
CREATE TABLE roles (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nombre VARCHAR(40) NOT NULL,       -- Ej: "Administrador de Panadería"
    slug VARCHAR(40) NOT NULL,         -- Ej: "admin", "panadero", "vendedor"
    descripcion VARCHAR(150) NULL,     -- Explicación de las responsabilidades del rol
    activo BOOLEAN DEFAULT TRUE,       -- Para inhabilitar un rol sin borrarlo
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE (slug)                      -- El identificador técnico debe ser único
) ENGINE=InnoDB;

-- Tabla intermedia (Laravel espera nombres en singular y orden alfabético, pero esto funciona bien)
SET FOREIGN_KEY_CHECKS = 0; 
DROP TABLE IF EXISTS permiso_rol;
SET FOREIGN_KEY_CHECKS = 1;
CREATE TABLE permiso_rol (
    rol_id INT NOT NULL,
    permiso_id INT NOT NULL,
    
    -- Auditoría: ¿Cuándo se asignó este permiso a este rol?
    asignado_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- Llave Primaria Compuesta (Evita duplicados de la misma relación)
    PRIMARY KEY (rol_id, permiso_id),
    
    -- Integridad Referencial con Nombres de Restricción Claros
    CONSTRAINT fk_rol_permiso FOREIGN KEY (rol_id) 
        REFERENCES roles(id) 
        ON UPDATE CASCADE 
        ON DELETE CASCADE,
        
    CONSTRAINT fk_permiso_rol FOREIGN KEY (permiso_id) 
        REFERENCES permisos(id) 
        ON UPDATE CASCADE 
        ON DELETE CASCADE
) ENGINE=InnoDB;

SET FOREIGN_KEY_CHECKS = 0;
DROP TABLE IF EXISTS usuarios;

CREATE TABLE usuarios (
    codigo VARCHAR(10) PRIMARY KEY,
    nombre VARCHAR(40) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    sexo CHAR(1) NOT NULL,
    password VARCHAR(250) NOT NULL, -- Aquí garantizamos que se llame 'password'
    telefono VARCHAR(15) NOT NULL,
    direccion TEXT,
    rol_id INT NOT NULL,
    remember_token VARCHAR(100) NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    CONSTRAINT fk_usuario_rol FOREIGN KEY (rol_id) REFERENCES roles(id)
) ENGINE=InnoDB;

SET FOREIGN_KEY_CHECKS = 1;

-- TABLA: clientes
SET FOREIGN_KEY_CHECKS = 0; 
DROP TABLE IF EXISTS clientes;
SET FOREIGN_KEY_CHECKS = 1;
CREATE TABLE clientes (
    ci VARCHAR(12) NOT NULL,            -- Aumentado a 12 por si incluyen complemento de CI
    nombre VARCHAR(60) NOT NULL,        -- Aumentado para nombres completos
    email VARCHAR(100) NULL,            -- Opcional, pero útil para marketing o facturación
    sexo CHAR(1) NOT NULL,
    telefono VARCHAR(15) NOT NULL,      -- Aumentado para formatos internacionales o celulares
    puntos_lealtad INT DEFAULT 0,       -- Mejora pro: Para fidelizar clientes en la panadería
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    PRIMARY KEY (ci),
    UNIQUE (email),
    CONSTRAINT chk_sexo_cliente CHECK (sexo IN ('M', 'F'))
) ENGINE=InnoDB;

-- Solo usa este bloque para la tabla direcciones
SET FOREIGN_KEY_CHECKS = 0;
DROP TABLE IF EXISTS direcciones;
SET FOREIGN_KEY_CHECKS = 1;

CREATE TABLE direcciones (
    id INT AUTO_INCREMENT PRIMARY KEY,
    titulo VARCHAR(30) NOT NULL,        -- Ej: "Casa", "Oficina"
    avenida_calle VARCHAR(100) NOT NULL,
    barrio VARCHAR(50) NOT NULL,
    nro_casa VARCHAR(10) NULL,
    uv VARCHAR(10) NULL,                -- Específico para Santa Cruz
    manzano VARCHAR(10) NULL,           -- Específico para Santa Cruz
    referencia TEXT NULL,
    geolocalizacion VARCHAR(255) NULL,  -- Capacidad para enlaces de Maps
    cliente_ci VARCHAR(12) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT fk_direccion_cliente FOREIGN KEY (cliente_ci) 
        REFERENCES clientes(ci) 
        ON UPDATE CASCADE 
        ON DELETE CASCADE
) ENGINE=InnoDB;

SET FOREIGN_KEY_CHECKS = 0;
DROP TABLE IF EXISTS pedidos;
SET FOREIGN_KEY_CHECKS = 1;
CREATE TABLE pedidos (
    id INT AUTO_INCREMENT PRIMARY KEY,
    fecha_pedido DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP, -- Cambio a DATETIME para saber la hora exacta
    fecha_entrega DATETIME NOT NULL,
    tipo ENUM('sucursal', 'delivery') NOT NULL,             -- Usamos ENUM para mayor control
    estado ENUM('pendiente', 'preparando', 'listo', 'en camino', 'entregado', 'cancelado') DEFAULT 'pendiente',
    pagado BOOLEAN NOT NULL DEFAULT FALSE,
    metodo_pago ENUM('efectivo', 'qr', 'transferencia') DEFAULT 'efectivo',
    subtotal DECIMAL(12, 2) NOT NULL DEFAULT 0,
    costo_envio DECIMAL(12, 2) DEFAULT 0,                    -- Vital si hay delivery
    total DECIMAL(12, 2) NOT NULL DEFAULT 0,
    observaciones TEXT NULL,                                -- Ej: "No tocar timbre, llamar al llegar"
    cliente_ci VARCHAR(12) NOT NULL,
    direccion_id INT NULL,                                  -- Relación con la dirección específica
    usuario_codigo VARCHAR(10) NULL,                        -- Quién registró el pedido (vendedor)
    
    -- Auditoría
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    -- Llaves Foráneas
    CONSTRAINT fk_pedido_cliente FOREIGN KEY (cliente_ci) 
        REFERENCES clientes(ci) ON UPDATE CASCADE ON DELETE CASCADE,
    CONSTRAINT fk_pedido_direccion FOREIGN KEY (direccion_id) 
        REFERENCES direcciones(id) ON UPDATE CASCADE ON DELETE SET NULL,
    CONSTRAINT fk_pedido_usuario FOREIGN KEY (usuario_codigo) 
        REFERENCES usuarios(codigo) ON UPDATE CASCADE ON DELETE SET NULL
) ENGINE=InnoDB;

SET FOREIGN_KEY_CHECKS = 0;
DROP TABLE IF EXISTS facturas_internas;
SET FOREIGN_KEY_CHECKS = 1;
CREATE TABLE facturas_internas (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nro_factura VARCHAR(20) NOT NULL,      -- Ej: "FAC-0001", útil para control físico
    total DECIMAL(12, 2) NOT NULL,
    puntos_ganados INT DEFAULT 0,          -- Si usas el sistema de lealtad en clientes
    fecha_emision DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP, 
    estado ENUM('valida', 'anulada') DEFAULT 'valida', -- Vital para contabilidad
    motivo_anulacion VARCHAR(150) NULL,    -- Por qué se anuló la factura
    
    usuario_codigo VARCHAR(10) NOT NULL,   -- El cajero/vendedor que emitió
    cliente_ci VARCHAR(12) NOT NULL,       -- CI del cliente (consistente con 12 chars)
    pedido_id INT NULL,                    -- Relación opcional: ¿Esta factura viene de un pedido previo?

    -- Auditoría técnica
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    -- Llaves Foráneas con nombres descriptivos
    CONSTRAINT fk_factura_usuario FOREIGN KEY (usuario_codigo) 
        REFERENCES usuarios(codigo) ON UPDATE CASCADE ON DELETE RESTRICT,
    CONSTRAINT fk_factura_cliente FOREIGN KEY (cliente_ci) 
        REFERENCES clientes(ci) ON UPDATE CASCADE ON DELETE RESTRICT,
    CONSTRAINT fk_factura_pedido FOREIGN KEY (pedido_id) 
        REFERENCES pedidos(id) ON UPDATE CASCADE ON DELETE SET NULL
) ENGINE=InnoDB;

-- 3. PRODUCTOS Y CATEGORÍAS
SET FOREIGN_KEY_CHECKS = 0;
DROP TABLE IF EXISTS categorias;
SET FOREIGN_KEY_CHECKS = 1;
CREATE TABLE categorias (
    id INT AUTO_INCREMENT PRIMARY KEY, 
    nombre VARCHAR(50) NOT NULL,        -- Aumentado para nombres como "Panadería Tradicional Salada"
    slug VARCHAR(50) NOT NULL,          -- Identificador técnico (ej: 'panaderia-dulce')
    descripcion VARCHAR(255) NULL,      -- Aumentado para explicar el uso de la categoría
    imagen VARCHAR(255) NULL,           -- Para mostrar un icono o foto en el sistema de ventas
    activo BOOLEAN DEFAULT TRUE,        -- Para ocultar categorías sin borrarlas
    
    -- Auditoría
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    UNIQUE (nombre),
    UNIQUE (slug)
) ENGINE=InnoDB;

SET FOREIGN_KEY_CHECKS = 0;
DROP TABLE IF EXISTS productos;
SET FOREIGN_KEY_CHECKS = 1;
CREATE TABLE productos (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nombre VARCHAR(100) NOT NULL,       -- Aumentado a 100 para nombres detallados
    descripcion TEXT NULL,              -- Para ingredientes o detalles del producto
    precio_venta DECIMAL(12, 2) NOT NULL DEFAULT 0.00,
    precio_costo DECIMAL(12, 2) NOT NULL DEFAULT 0.00, -- Vital para calcular ganancias
    stock DECIMAL(12, 2) NOT NULL DEFAULT 0.00,        -- Decimal por si vendes por peso (kg)
    stock_minimo DECIMAL(12, 2) NOT NULL DEFAULT 5.00,
    imagen VARCHAR(255) NULL,           -- Ruta de la foto para el catálogo/POS
    es_producido BOOLEAN DEFAULT TRUE,  -- Diferencia entre pan hecho en casa y refrescos comprados
    estado ENUM('activo', 'agotado', 'descontinuado') DEFAULT 'activo',
    
    categoria_id INT NOT NULL,
    
    -- Auditoría
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    -- Llaves Foráneas y Restricciones
    CONSTRAINT fk_producto_categoria FOREIGN KEY (categoria_id) 
        REFERENCES categorias(id) ON UPDATE CASCADE ON DELETE RESTRICT
) ENGINE=InnoDB;

-- 4. TABLAS DE DETALLE (Pivotes)
SET FOREIGN_KEY_CHECKS = 0;
DROP TABLE IF EXISTS detalle_factura;
SET FOREIGN_KEY_CHECKS = 1;
CREATE TABLE detalle_factura (
    factura_interna_id INT NOT NULL,
    producto_id INT NOT NULL,
    cantidad DECIMAL(12, 2) NOT NULL, -- DECIMAL por si vendes por peso (kg)
    precio_unitario DECIMAL(12, 2) NOT NULL, -- Precio al que se vendió en ese momento
    subtotal DECIMAL(12, 2) NOT NULL,        -- cantidad * precio_unitario
    descuento DECIMAL(12, 2) DEFAULT 0.00,   -- Por si hay rebaja por volumen
    total_linea DECIMAL(12, 2) NOT NULL,     -- subtotal - descuento
    
    PRIMARY KEY (factura_interna_id, producto_id),
    
    -- Usamos RESTRICT para productos para no borrar historial de ventas
    CONSTRAINT fk_detalle_factura FOREIGN KEY (factura_interna_id) 
        REFERENCES facturas_internas(id) ON UPDATE CASCADE ON DELETE CASCADE,
        
    CONSTRAINT fk_detalle_producto FOREIGN KEY (producto_id) 
        REFERENCES productos(id) ON UPDATE CASCADE ON DELETE RESTRICT
) ENGINE=InnoDB;

SET FOREIGN_KEY_CHECKS = 0;
DROP TABLE IF EXISTS pedido_producto;
SET FOREIGN_KEY_CHECKS = 1;
CREATE TABLE pedido_producto (
    pedido_id INT NOT NULL,
    producto_id INT NOT NULL,
    cantidad DECIMAL(12, 2) NOT NULL,      -- DECIMAL para productos por kilo (galletas, tortas)
    precio_unitario DECIMAL(12, 2) NOT NULL, -- El precio pactado al momento del pedido
    subtotal DECIMAL(12, 2) NOT NULL,        -- cantidad * precio_unitario
    
    PRIMARY KEY (pedido_id, producto_id),
    
    -- Relaciones
    CONSTRAINT fk_detalle_pedido FOREIGN KEY (pedido_id) 
        REFERENCES pedidos(id) ON UPDATE CASCADE ON DELETE CASCADE,
        
    CONSTRAINT fk_pedido_producto FOREIGN KEY (producto_id) 
        REFERENCES productos(id) ON UPDATE CASCADE ON DELETE RESTRICT
) ENGINE=InnoDB;

-- 5. PRODUCCIÓN Y RECETAS
SET FOREIGN_KEY_CHECKS = 0;
DROP TABLE IF EXISTS nota_salida;
SET FOREIGN_KEY_CHECKS = 1;
CREATE TABLE notas_salida (
    id INT AUTO_INCREMENT PRIMARY KEY,
    fecha_hora DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    motivo ENUM('merma', 'donacion', 'consumo_interno', 'vencimiento', 'otros') NOT NULL,
    descripcion VARCHAR(255) NULL,    -- Aumentado para dar detalles del incidente
    usuario_codigo VARCHAR(10) NOT NULL, -- Quién registra la salida
    
    -- Auditoría
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- Relaciones
    CONSTRAINT fk_nota_usuario FOREIGN KEY (usuario_codigo) 
        REFERENCES usuarios(codigo) ON UPDATE CASCADE ON DELETE RESTRICT
) ENGINE=InnoDB;

SET FOREIGN_KEY_CHECKS = 0;
DROP TABLE IF EXISTS detalle_nota;
SET FOREIGN_KEY_CHECKS = 1;
CREATE TABLE detalle_nota (
    nota_salida_id INT NOT NULL,
    producto_id INT NOT NULL,
    cantidad DECIMAL(12, 2) NOT NULL, -- DECIMAL para soportar unidades (1.00) o peso (0.50 kg)
    
    PRIMARY KEY (nota_salida_id, producto_id),
    
    -- Relaciones
    CONSTRAINT fk_detallenota_nota FOREIGN KEY (nota_salida_id) 
        REFERENCES notas_salida(id) 
        ON UPDATE CASCADE 
        ON DELETE CASCADE,
        
    CONSTRAINT fk_detallenota_producto FOREIGN KEY (producto_id) 
        REFERENCES productos(id) 
        ON UPDATE CASCADE 
        ON DELETE RESTRICT -- IMPORTANTE: No borrar el producto si tiene historial de salida
) ENGINE=InnoDB;


SET FOREIGN_KEY_CHECKS = 0;
DROP TABLE IF EXISTS recetas;
SET FOREIGN_KEY_CHECKS = 1;
CREATE TABLE recetas (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nombre VARCHAR(100) NOT NULL,       -- Ej: "Receta Estándar Pan de Batalla"
    rendimiento_estimado DECIMAL(12, 2) NOT NULL, -- Cuántas unidades o kg rinde
    tiempo_preparacion_min INT NOT NULL, -- Guardar minutos es más fácil para cálculos que el tipo TIME
    instrucciones TEXT NULL,            -- El "paso a paso" para el panadero
    estado ENUM('activa', 'borrador', 'obsoleta') DEFAULT 'activa',
    
    producto_id INT NOT NULL,           -- El producto final que genera esta receta
    
    -- Auditoría
    usuario_codigo VARCHAR(10) NOT NULL, -- Quién creó/autorizó la receta
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    -- Relaciones
    CONSTRAINT fk_receta_producto FOREIGN KEY (producto_id) 
        REFERENCES productos(id) ON UPDATE CASCADE ON DELETE CASCADE,
    CONSTRAINT fk_receta_usuario FOREIGN KEY (usuario_codigo) 
        REFERENCES usuarios(codigo) ON UPDATE CASCADE ON DELETE RESTRICT
) ENGINE=InnoDB;

SET FOREIGN_KEY_CHECKS = 0;

CREATE TABLE detalle_receta (
    id INT AUTO_INCREMENT PRIMARY KEY,
    receta_id INT NOT NULL,
    insumo_id INT NOT NULL,
    cantidad_necesaria DECIMAL(12, 4) NOT NULL, -- Usamos 4 decimales para mayor precisión en gramos
    
    -- Relaciones
    CONSTRAINT fk_detalle_receta_padre FOREIGN KEY (receta_id) 
        REFERENCES recetas(id) ON UPDATE CASCADE ON DELETE CASCADE,
    CONSTRAINT fk_detalle_insumo FOREIGN KEY (insumo_id) 
        REFERENCES insumos(id) ON UPDATE CASCADE ON DELETE RESTRICT
) ENGINE=InnoDB;

SET FOREIGN_KEY_CHECKS = 1;

-- 6. INSUMOS Y RECETAS (DETALLES)
SET FOREIGN_KEY_CHECKS = 0;
DROP TABLE IF EXISTS insumos;
SET FOREIGN_KEY_CHECKS = 1;
CREATE TABLE insumos (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nombre VARCHAR(50) NOT NULL,        -- Aumentado para nombres como "Harina de Trigo 000"
    unidad_medida ENUM('kg', 'gr', 'lt', 'ml', 'unid', 'bolsa') NOT NULL, -- Medidas estandarizadas
    stock_actual DECIMAL(12, 2) NOT NULL DEFAULT 0.00,
    stock_minimo DECIMAL(12, 2) NOT NULL DEFAULT 5.00,
    precio_compra_promedio DECIMAL(12, 2) NOT NULL DEFAULT 0.00, -- Para costeo de recetas
    
    -- Auditoría
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB;

-- 7. PRODUCCIÓN Y PARTICIPACIÓN
SET FOREIGN_KEY_CHECKS = 0;
DROP TABLE IF EXISTS produccione;
SET FOREIGN_KEY_CHECKS = 1;
CREATE TABLE producciones (
    id INT AUTO_INCREMENT PRIMARY KEY,
    lote_codigo VARCHAR(20) NULL,        -- Ej: "PROD-20260415-01" para trazabilidad
    descripcion VARCHAR(150) NULL,      -- Aumentado para notas de producción
    fecha_programada DATE NOT NULL,     -- Cuándo se planeó hacer
    hora_inicio DATETIME NULL,          -- Cuándo empezó realmente el horno
    hora_fin DATETIME NULL,             -- Para calcular cuánto tardó el panadero
    estado ENUM('planificado', 'en_proceso', 'completado', 'fallido') DEFAULT 'planificado',
    cantidad_producida DECIMAL(12, 2) DEFAULT 0, -- Lo que realmente salió del horno
    observaciones_calidad TEXT NULL,    -- Ej: "El pan salió un poco quemado por el horno 2"
    
    receta_id INT NOT NULL,
    usuario_codigo VARCHAR(10) NOT NULL, -- El panadero responsable
    
    -- Auditoría
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- Relaciones
    CONSTRAINT fk_produccion_receta FOREIGN KEY (receta_id) 
        REFERENCES recetas(id) ON UPDATE CASCADE ON DELETE RESTRICT,
    CONSTRAINT fk_produccion_usuario FOREIGN KEY (usuario_codigo) 
        REFERENCES usuarios(codigo) ON UPDATE CASCADE ON DELETE RESTRICT
) ENGINE=InnoDB;

-- Esta tabla registra qué panadero trabajó en qué producción
SET FOREIGN_KEY_CHECKS = 0;
DROP TABLE IF EXISTS participaciones;
SET FOREIGN_KEY_CHECKS = 1;
CREATE TABLE participaciones (
    usuario_codigo VARCHAR(10) NOT NULL, 
    produccion_id INT NOT NULL,
    rol_en_turno ENUM('maestro_panadero', 'ayudante', 'hornero', 'limpieza') DEFAULT 'ayudante',
    horas_trabajadas DECIMAL(4, 2) NULL, -- Útil si pagas por hora o para medir productividad
    observaciones_empleado VARCHAR(150) NULL,
    
    PRIMARY KEY (usuario_codigo, produccion_id),
    
    -- Relaciones
    CONSTRAINT fk_participacion_usuario FOREIGN KEY (usuario_codigo) 
        REFERENCES usuarios(codigo) 
        ON UPDATE CASCADE 
        ON DELETE CASCADE,
        
    CONSTRAINT fk_participacion_produccion FOREIGN KEY (produccion_id) 
        REFERENCES producciones(id) 
        ON UPDATE CASCADE 
        ON DELETE CASCADE
) ENGINE=InnoDB;

-- 8. PROVEEDORES Y COMPRAS
SET FOREIGN_KEY_CHECKS = 0;
DROP TABLE IF EXISTS proveedores;
SET FOREIGN_KEY_CHECKS = 1;
CREATE TABLE proveedores (
    codigo VARCHAR(10) NOT NULL,
    nombre_contacto VARCHAR(60) NOT NULL, -- Persona con la que se habla
    empresa VARCHAR(60) NOT NULL,         -- Razón social (ej: "Molinos S.A.")
    nit VARCHAR(20) NULL,                -- Identificación tributaria (importante en Bolivia)
    telefono VARCHAR(15) NOT NULL,
    email VARCHAR(100) NULL,
    direccion TEXT NULL,
    estado ENUM('activo', 'suspendido', 'inactivo') DEFAULT 'activo',
    
    -- Auditoría
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    PRIMARY KEY (codigo),
    UNIQUE (nit)
) ENGINE=InnoDB;

SET FOREIGN_KEY_CHECKS = 0;
DROP TABLE IF EXISTS nota_compra;
SET FOREIGN_KEY_CHECKS = 1;
CREATE TABLE notas_compra (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nro_comprobante VARCHAR(20) NULL,      -- El nro de factura o recibo que entrega el proveedor
    fecha_pedido DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    fecha_recepcion DATETIME NULL,         -- Cuándo llegó realmente la mercadería
    monto_total DECIMAL(12, 2) NOT NULL DEFAULT 0.00,
    estado ENUM('solicitado', 'recibido', 'cancelado') DEFAULT 'solicitado',
    observaciones TEXT NULL,               -- Ej: "Llegó con 2 bolsas de harina rotas"
    
    usuario_codigo VARCHAR(10) NOT NULL,   -- El encargado que realiza/recibe la compra
    proveedor_codigo VARCHAR(10) NOT NULL, -- A quién le compramos
    
    -- Auditoría
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    -- Relaciones
    CONSTRAINT fk_compra_usuario FOREIGN KEY (usuario_codigo) 
        REFERENCES usuarios(codigo) ON UPDATE CASCADE ON DELETE RESTRICT,
    CONSTRAINT fk_compra_proveedor FOREIGN KEY (proveedor_codigo) 
        REFERENCES proveedores(codigo) ON UPDATE CASCADE ON DELETE RESTRICT
) ENGINE=InnoDB;

SET FOREIGN_KEY_CHECKS = 0;
DROP TABLE IF EXISTS campra_insumo;
SET FOREIGN_KEY_CHECKS = 1;
CREATE TABLE compra_insumo (
    nota_compra_id INT NOT NULL,
    insumo_id INT NOT NULL,
    cantidad DECIMAL(12, 2) NOT NULL,        -- Por si compras por peso (ej: 50.5 kg)
    precio_unitario DECIMAL(12, 2) NOT NULL, -- Precio de compra de hoy
    subtotal DECIMAL(12, 2) NOT NULL,        -- cantidad * precio_unitario
    
    PRIMARY KEY (nota_compra_id, insumo_id),
    
    -- Relaciones
    CONSTRAINT fk_detallenotacompra_nota FOREIGN KEY (nota_compra_id) 
        REFERENCES notas_compra(id) 
        ON UPDATE CASCADE 
        ON DELETE CASCADE,
        
    CONSTRAINT fk_detallenotacompra_insumo FOREIGN KEY (insumo_id) 
        REFERENCES insumos(id) 
        ON UPDATE CASCADE 
        ON DELETE RESTRICT -- Protegemos el insumo si tiene historial de compras
) ENGINE=InnoDB;

SET FOREIGN_KEY_CHECKS = 0;
DROP TABLE IF EXISTS compra_producto;
SET FOREIGN_KEY_CHECKS = 1;
CREATE TABLE compra_producto (
    nota_compra_id INT NOT NULL,
    producto_id INT NOT NULL,
    cantidad INT NOT NULL,               -- Generalmente los productos de reventa son por unidad
    precio_compra_unitario DECIMAL(12, 2) NOT NULL,
    subtotal DECIMAL(12, 2) NOT NULL,    -- cantidad * precio_compra_unitario
    
    PRIMARY KEY (nota_compra_id, producto_id),
    
    -- Relaciones
    CONSTRAINT fk_compraprod_nota FOREIGN KEY (nota_compra_id) 
        REFERENCES notas_compra(id) 
        ON UPDATE CASCADE 
        ON DELETE CASCADE,
        
    CONSTRAINT fk_compraprod_producto FOREIGN KEY (producto_id) 
        REFERENCES productos(id) 
        ON UPDATE CASCADE 
        ON DELETE RESTRICT -- No borrar el producto si hay historial de compra
) ENGINE=InnoDB;

-- ######################################################
-- 2. CARGA DE DATOS INICIALES (MUESTRA)
-- ######################################################
-- 1. ROLES (Tabla 'roles', columnas en minúsculas)
SET FOREIGN_KEY_CHECKS = 0;
TRUNCATE TABLE roles;
SET FOREIGN_KEY_CHECKS = 1;

INSERT INTO roles (id, nombre, slug) VALUES 
(1, 'Administrador', 'admin'), 
(2, 'Cajero', 'cajero'), 
(3, 'Proveedor', 'proveedor'), 
(4, 'Cocinero / Panadero', 'panadero'), 
(5, 'Cliente', 'cliente');

-- He cambiado 'contrasena' por 'password' para que coincida con tu tabla
SET FOREIGN_KEY_CHECKS = 0;
TRUNCATE TABLE usuarios;
SET FOREIGN_KEY_CHECKS = 1;
INSERT INTO usuarios (codigo, nombre, email, sexo, password, telefono, direccion, rol_id) VALUES 
('US0001', 'Nadia Carvajal Ramos', 'nadycarvajal@gmail.com', 'F', '$2y$10$92IXUNpkjO0rOQ5byMi.Ye4oKoEa3Ro9llC/.og/at2.uheWG/igi', '68824368', 'Santa Cruz, Bolivia', 1),
('US0002', 'Josue Luberth', 'josue@mail.com', 'M', '$2y$10$92IXUNpkjO0rOQ5byMi.Ye4oKoEa3Ro9llC/.og/at2.uheWG/igi', '69275363', 'Av. Bush, Calle 5', 1),
('US0003', 'Jhoel Jhonny Heredia', 'jhoel@mail.com', 'M', '$2y$10$92IXUNpkjO0rOQ5byMi.Ye4oKoEa3Ro9llC/.og/at2.uheWG/igi', '62129358', 'Barrio Lujan, Calle 2', 1);

-- 2. CATEGORÍAS (Tabla 'categorias')
SET FOREIGN_KEY_CHECKS = 0;
TRUNCATE TABLE categorias;
SET FOREIGN_KEY_CHECKS = 1;

INSERT INTO categorias (id, nombre, slug, descripcion, activo) VALUES
(1, 'Panes', 'panes', 'Todo tipo de pan de avena, trigo y masa madre', 1), 
(2, 'Galletas', 'galletas', 'Galletas artesanales y snacks dulces', 1),
(3, 'Empanadas', 'empanadas', 'Empanadas saladas y productos rellenos', 1), 
(4, 'Postres y Bollería', 'postres-bolleria', 'Rollos, croissants, brioche y repostería fina', 1), 
(5, 'Bebidas', 'bebidas', 'Suero de kéfir, jugos naturales y otras bebidas', 1),
(6, 'Especiales', 'especiales', 'Línea de pan keto y productos sin gluten', 1);

-- 3. INSUMOS (Tabla 'insumos')
SET FOREIGN_KEY_CHECKS = 0;
TRUNCATE TABLE insumos;
SET FOREIGN_KEY_CHECKS = 1;

INSERT INTO insumos (id, nombre, unidad_medida, stock_actual, stock_minimo) VALUES
(1, 'Harina de avena', 'kg', 100.00, 10.00),
(2, 'Harina de trigo', 'kg', 100.00, 10.00),
(3, 'Harina de trigo integral', 'kg', 100.00, 10.00),
(4, 'Masa madre', 'kg', 50.00, 5.00),
(5, 'Sal', 'gr', 5000.00, 500.00),
(6, 'Linaza molida', 'gr', 2000.00, 200.00),
(7, 'Semillas de girasol', 'gr', 2000.00, 200.00),
(8, 'Semillas de chia', 'gr', 2000.00, 200.00),
(9, 'Semillas de sesamo', 'gr', 2000.00, 200.00),
(10, 'Azúcar', 'gr', 5000.00, 500.00),
(11, 'Mantequilla', 'gr', 2000.00, 200.00),
(12, 'Aceite de oliva', 'ml', 5000.00, 500.00),
(13, 'Queso criollo', 'gr', 3000.00, 300.00),
(14, 'Jamón', 'gr', 2000.00, 200.00),
(15, 'Chocolate chips', 'gr', 2000.00, 200.00),
(16, 'Almendras', 'gr', 1500.00, 150.00),
(17, 'Uvas pasas', 'gr', 1500.00, 150.00),
(18, 'Cúrcuma', 'gr', 1000.00, 100.00),
(19, 'Harina de arroz', 'kg', 50.00, 5.00),
(20, 'Huevos', 'unid', 200.00, 20.00);

-- 4. PRODUCTOS 
SET FOREIGN_KEY_CHECKS = 0;
TRUNCATE TABLE productos;
SET FOREIGN_KEY_CHECKS = 1;

INSERT INTO productos (id, nombre, precio_venta, precio_costo, stock, stock_minimo, categoria_id, es_producido, estado) VALUES
-- Categoria 1: Panes
(1, 'Pan de avena clasico', 25.00, 15.00, 50, 5, 1, 1, 'activo'),
(2, 'Pan de avena con linaza', 25.00, 16.00, 50, 5, 1, 1, 'activo'),
(3, 'Pan de avena con chocolate', 35.00, 22.00, 50, 5, 1, 1, 'activo'),

-- Categoria 2: Galletas
(4, 'Galleta avena con chocolate bolsa grande', 30.00, 18.00, 40, 5, 2, 1, 'activo'),
(5, 'Galleta clasica mezcla de semillas 150gr', 15.00, 8.50, 40, 5, 2, 1, 'activo'),

-- Categoria 3: Empanadas
(6, 'Pizza de empanada queso y jamon', 7.00, 3.50, 60, 10, 3, 1, 'activo'),
(7, 'Empanada de queso criollo', 7.00, 3.00, 60, 10, 3, 1, 'activo'),
(8, 'Empanada queso espinacas', 7.00, 3.20, 60, 10, 3, 1, 'activo'),

-- Categoria 4: Postres y bolleria
(9, 'Rollo de canela 100gr', 12.00, 6.00, 40, 5, 4, 1, 'activo'),
(10, 'Croissant clasico', 10.00, 4.50, 40, 5, 4, 1, 'activo'),
(11, 'Palmeritas de hojaldre', 8.00, 3.80, 40, 5, 4, 1, 'activo'),

-- Categoria 6: Especiales (Keto)
(12, 'Pan keto sin gluten', 8.00, 4.20, 30, 5, 6, 1, 'activo'),

-- Categoria 5: Bebidas
(13, 'Suero de kefir 300ml', 10.00, 5.00, 50, 5, 5, 1, 'activo');

-- 5. RECETAS 
SET FOREIGN_KEY_CHECKS = 0;
TRUNCATE TABLE recetas;
SET FOREIGN_KEY_CHECKS = 1;

INSERT INTO recetas (id, nombre, rendimiento_estimado, tiempo_preparacion_min, producto_id, usuario_codigo, estado) VALUES 
(1, 'Receta Pan de avena clasico', 1.00, 90, 1, 'US0001', 'activa'), 
(2, 'Receta Pan de avena con linaza', 1.00, 90, 2, 'US0001', 'activa'), 
(3, 'Receta Pan de avena con chocolate', 1.00, 90, 3, 'US0001', 'activa'), 
(4, 'Receta Galleta avena chocolate (Bolsa G)', 15.00, 60, 4, 'US0001', 'activa'), 
(5, 'Receta Galleta clasica semillas 150gr', 1.00, 60, 5, 'US0002', 'activa'), 
(6, 'Receta Pizza empanada Q/J', 1.00, 30, 6, 'US0002', 'activa'), 
(7, 'Receta Empanada queso criollo', 1.00, 25, 7, 'US0002', 'activa'), 
(8, 'Receta Empanada queso espinacas', 1.00, 25, 8, 'US0002', 'activa'), 
(9, 'Receta Rollo de canela', 1.00, 45, 9, 'US0003', 'activa'), 
(10, 'Receta Croissant clasico', 1.00, 40, 10, 'US0003', 'activa'), 
(11, 'Receta Palmeritas hojaldre', 1.00, 30, 11, 'US0003', 'activa'), 
(12, 'Receta Pan keto sin gluten', 1.00, 50, 12, 'US0003', 'activa'), 
(13, 'Receta Suero de kefir 300ml', 1.00, 5, 13, 'US0003', 'activa');

-- 1. CONTINUACIÓN DE DETALLE_RECETA (Tabla: detalle_receta)
SET FOREIGN_KEY_CHECKS = 0;
TRUNCATE TABLE detalle_receta;
SET FOREIGN_KEY_CHECKS = 1;

-- Nota: La medida se toma automáticamente de la tabla 'insumos' para evitar errores.
-- Solo insertamos la cantidad necesaria según la unidad base del insumo.

INSERT INTO detalle_receta (receta_id, insumo_id, cantidad_necesaria) VALUES
-- Receta 2: Pan de avena con linaza
(2, 1, 0.3), (2, 4, 0.05), (2, 5, 5.00), (2, 6, 10.00),
-- Receta 3: Pan de avena con chocolate
(3, 1, 0.3), (3, 4, 0.05), (3, 5, 5.00), (3, 15, 20.00),
-- Receta 4: Galleta avena chocolate (Bolsa G)
(4, 1, 0.45), (4, 4, 0.05), (4, 10, 30.00), (4, 11, 30.00), (4, 15, 50.00),
-- Receta 5: Galleta clasica semillas 150gr
(5, 4, 0.05), (5, 1, 0.15), (5, 7, 20.00), (5, 8, 10.00),
-- Receta 6: Pizza empanada Q/J
(6, 2, 0.1), (6, 13, 50.00), (6, 14, 50.00),
-- Receta 7: Empanada queso criollo
(7, 3, 0.08), (7, 13, 50.00),
-- Receta 8: Empanada queso espinacas
(8, 3, 0.08), (8, 13, 30.00),
-- Receta 9: Rollo de canela
(9, 1, 0.1), (9, 19, 0.05), (9, 10, 15.00), (9, 11, 20.00),
-- Receta 10: Croissant clasico
(10, 2, 0.1), (10, 11, 15.00), (10, 10, 10.00),
-- Receta 11: Palmeritas hojaldre
(11, 2, 0.08), (11, 11, 10.00), (11, 10, 5.00), (11, 20, 1.00),
-- Receta 12: Pan keto sin gluten
(12, 19, 0.2),
-- Receta 13: Suero de kefir 300ml
(13, 4, 0.05);

-- 2. PERMISOS (Tabla: permisos)
SET FOREIGN_KEY_CHECKS = 0;
TRUNCATE TABLE permisos;
SET FOREIGN_KEY_CHECKS = 1;

INSERT INTO permisos (id, nombre, slug, descripcion) VALUES
(1, 'Gestionar Usuarios', 'usuarios.index', 'Crear, editar o eliminar usuarios'), 
(2, 'Gestionar Clientes', 'clientes.index', 'Agregar o modificar clientes'),
(3, 'Gestionar Proveedores', 'proveedores.index', 'Agregar o modificar proveedores'), 
(4, 'Gestionar Productos', 'productos.index', 'Agregar, editar o eliminar productos'), 
(5, 'Gestionar Insumos', 'insumos.index', 'Agregar, editar o eliminar insumos'),
(6, 'Gestionar Categorías', 'categorias.index', 'Agregar o editar categorías de productos'), 
(7, 'Gestionar Pedidos', 'pedidos.index', 'Ver, crear o actualizar pedidos de clientes'),
(8, 'Gestionar Facturas', 'facturas.index', 'Emitir y consultar facturas internas'), 
(9, 'Gestionar Producción', 'produccion.index', 'Registrar producción y recetas'),
(10, 'Gestionar Compras', 'compras.index', 'Registrar compras de insumos y productos'), 
(11, 'Ver Reportes', 'reportes.index', 'Acceder a reportes de ventas, stock y producción');
-- 3. ROL_PERMISO (Tabla: permiso_rol)
SET FOREIGN_KEY_CHECKS = 0;
TRUNCATE TABLE permiso_rol;
SET FOREIGN_KEY_CHECKS = 1;

INSERT INTO permiso_rol (rol_id, permiso_id) VALUES
-- Admin: Acceso total
(1,1),(1,2),(1,3),(1,4),(1,5),(1,6),(1,7),(1,8),(1,9),(1,10),(1,11),
-- Cajero: Flujo de ventas y reportes de caja
(2,2),(2,7),(2,8),(2,11),
-- Proveedor/Encargado de Compras: Registro de abastecimiento
(3,10),
-- Cocinero: Inventario de materia prima y producción
(4,5),(4,9),
-- Cliente: Autogestión de pedidos
(5,7);

-- 5. PROVEEDORES Y COMPRAS (Tablas: proveedores, notas_compra, compra_insumo, compra_producto)
-- Limpiamos para evitar duplicados si ya intentaste insertar antes
SET FOREIGN_KEY_CHECKS = 0;
TRUNCATE TABLE proveedores;
SET FOREIGN_KEY_CHECKS = 1;

INSERT INTO proveedores (codigo, nombre_contacto, empresa, nit, telefono, email, direccion, estado) VALUES  
('PRV001', 'Juan Pérez', 'Molinos del Sur', '1020304050', '71236517', 'ventas@molinos.com', 'Parque Industrial, Santa Cruz', 'activo'), 
('PRV002', 'María López', 'Lácteos Santa Cruz', '2030405060', '77234568', 'info@lacteos-sc.bo', 'Av. Banzer km 5', 'activo'), 
('PRV003', 'Carlos Gómez', 'Distribuidora Granos', '3040506070', '61334269', 'carlos@granos.com', 'Barrio Lujan, Calle 3', 'activo');

-- nota compra
SET FOREIGN_KEY_CHECKS = 0;
TRUNCATE TABLE notas_compra;
SET FOREIGN_KEY_CHECKS = 1;

INSERT INTO notas_compra (id, fecha_pedido, fecha_recepcion, monto_total, estado, usuario_codigo, proveedor_codigo) VALUES 
(1, '2025-09-01 10:00:00', '2025-09-03 15:30:00', 975.00, 'recibido', 'US0001', 'PRV001'), 
(2, '2025-09-05 09:00:00', '2025-09-07 11:00:00', 821.00, 'recibido', 'US0002', 'PRV002'), -- Cambiado US0005 por US0002
(3, '2025-09-08 08:30:00', '2025-09-10 14:00:00', 430.00, 'recibido', 'US0003', 'PRV003'); -- Cambiado US0006 por US0003

-- 3. DETALLE: COMPRA DE INSUMOS (Materia Prima)
SET FOREIGN_KEY_CHECKS = 0;
TRUNCATE TABLE compra_insumo;
SET FOREIGN_KEY_CHECKS = 1;

INSERT INTO compra_insumo (nota_compra_id, insumo_id, cantidad, precio_unitario, subtotal) VALUES
(1, 1, 10.00, 20.00, 200.00), (1, 2, 5.00, 15.00, 75.00),
(2, 3, 12.00, 8.00, 96.00), (2, 4, 8.00, 25.00, 200.00),
(3, 1, 5.00, 20.00, 100.00), (3, 5, 7.00, 30.00, 210.00);

SET FOREIGN_KEY_CHECKS = 0;
TRUNCATE TABLE compra_producto;
SET FOREIGN_KEY_CHECKS = 1;

INSERT INTO compra_producto (nota_compra_id, producto_id, cantidad, precio_compra_unitario, subtotal) VALUES
(1, 1, 20, 25.00, 500.00),
(2, 3, 15, 35.00, 525.00),
(3, 5, 10, 12.00, 120.00);

-- 6. PRODUCCIONES (Tabla: producciones)
-- Limpiamos para evitar errores de duplicidad en las pruebas
SET FOREIGN_KEY_CHECKS = 0;
TRUNCATE TABLE producciones;
SET FOREIGN_KEY_CHECKS = 1;

INSERT INTO producciones (lote_codigo, descripcion, fecha_programada, hora_inicio, estado, cantidad_producida, receta_id, usuario_codigo) VALUES 
('LOTE-20250902-01', 'Producción matutina: Pan de avena clásico', '2025-09-02', '2025-09-02 07:00:00', 'completado', 50.00, 1, 'US0001'), 
('LOTE-20250905-02', 'Producción especial: Galleta de avena', '2025-09-05', '2025-09-05 08:00:00', 'completado', 100.00, 4, 'US0001'), 
('LOTE-20250908-03', 'Lote semanal: Croissant Clásico', '2025-09-08', '2025-09-08 06:30:00', 'completado', 30.00, 10, 'US0003');
-- 1. PARTICIPACIONES (Tabla: participaciones)
-- Relaciona qué usuario trabajó en qué producción
-- Limpiamos para evitar errores de llave primaria duplicada
SET FOREIGN_KEY_CHECKS = 0;
TRUNCATE TABLE participaciones;
SET FOREIGN_KEY_CHECKS = 1;

INSERT INTO participaciones (usuario_codigo, produccion_id, rol_en_turno, horas_trabajadas, observaciones_empleado) VALUES 
('US0003', 1, 'ayudante', 4.50, 'Apoyo en amasado'),       -- Jhoel (US0003) en producción 1
('US0002', 1, 'hornero', 4.00, 'Control de temperatura'),  -- Josue (US0002) en producción 1
('US0001', 2, 'maestro_panadero', 5.00, 'Lote de galletas'), -- Nadia (US0001) en producción 2
('US0003', 3, 'ayudante', 3.50, 'Formado de croissants'),  -- Jhoel en producción 3
('US0001', 3, 'maestro_panadero', 6.00, 'Supervisión general'); -- Nadia en producción 3

-- 2. CLIENTES (Tabla: clientes)pma__bookmarkpma__bookmarkpma__bookmark
-- Limpiamos para evitar errores de duplicidad de CI
SET FOREIGN_KEY_CHECKS = 0;
TRUNCATE TABLE clientes;
SET FOREIGN_KEY_CHECKS = 1;

INSERT INTO clientes (ci, nombre, email, sexo, telefono, puntos_lealtad) VALUES  
('13334249', 'Carlos Rivera', 'carlos.rivera@mail.com', 'M', '72012345', 0), 
('7794986', 'Ana Pérez', 'ana.perez@mail.com', 'F', '72123456', 0), 
('69425076', 'Luis Gómez', 'luis.gomez@mail.com', 'M', '72234567', 0);

-- 3. FACTURAS INTERNAS (Tabla: facturas_internas)
-- Limpieza preventiva
SET FOREIGN_KEY_CHECKS = 0;
TRUNCATE TABLE facturas_internas;
SET FOREIGN_KEY_CHECKS = 1;
-- Ajustado a: nro_factura, fecha_emision y usuarios existentes
INSERT INTO facturas_internas (nro_factura, total, fecha_emision, estado, usuario_codigo, cliente_ci) VALUES 
('FAC-0001', 125.00, '2025-09-02 10:30:00', 'valida', 'US0001', '13334249'), 
('FAC-0002', 70.00, '2025-09-05 16:45:00', 'valida', 'US0001', '7794986'), 
('FAC-0003', 105.00, '2025-09-08 11:15:00', 'valida', 'US0003', '69425076');

-- 4. DETALLE DE FACTURA (Tabla: detalle_factura)
SET FOREIGN_KEY_CHECKS = 0;
TRUNCATE TABLE detalle_factura;
SET FOREIGN_KEY_CHECKS = 1;

INSERT INTO detalle_factura (factura_interna_id, producto_id, cantidad, precio_unitario, subtotal, descuento, total_linea) VALUES 
-- Factura 1
(1, 1, 2.00, 25.00, 50.00, 0.00, 50.00), 
(1, 3, 1.00, 35.00, 35.00, 0.00, 35.00), 
(1, 4, 1.00, 40.00, 40.00, 0.00, 40.00),
-- Factura 2
(2, 5, 2.00, 15.00, 30.00, 0.00, 30.00), 
(2, 6, 1.00, 40.00, 40.00, 0.00, 40.00),
-- Factura 3
(3, 3, 2.00, 35.00, 70.00, 0.00, 70.00), 
(3, 7, 5.00, 7.00, 35.00, 0.00, 35.00);

-- ######################################################
-- 3. CONSULTAS 
-- ######################################################

-- 1. Total de ventas por fecha
SELECT 
    DATE(fecha_emision) AS dia, 
    SUM(total) AS total_ventas,
    COUNT(id) AS cantidad_facturas
FROM facturas_internas 
GROUP BY dia 
ORDER BY dia 
LIMIT 0, 1000;

-- 2. Total de ventas por producto
SELECT p.nombre, SUM(df.subtotal) AS total_ventas
FROM detalle_factura df
JOIN productos p ON df.producto_id = p.id
GROUP BY p.nombre
ORDER BY total_ventas DESC;

-- 3. Facturas internas emitidas por cada usuario
SELECT f.usuario_codigo, u.nombre, COUNT(f.id) AS total_facturas 
FROM facturas_internas f
JOIN usuarios u ON f.usuario_codigo = u.codigo 
GROUP BY f.usuario_codigo, u.nombre 
ORDER BY total_facturas DESC;

-- 4. Productos más vendidos en septiembre 2025
SELECT 
    p.id, 
    p.nombre, 
    SUM(df.cantidad) AS total_vendido 
FROM detalle_factura df 
JOIN productos p ON df.producto_id = p.id 
JOIN facturas_internas f ON df.factura_interna_id = f.id 
WHERE MONTH(f.fecha_emision) = 9 AND YEAR(f.fecha_emision) = 2025 
GROUP BY p.id, p.nombre  
ORDER BY total_vendido DESC 
LIMIT 0, 1000;

-- 5. Clientes que más han comprado
SELECT f.cliente_ci, c.nombre, SUM(f.total) AS total_comprado 
FROM facturas_internas f
JOIN clientes c ON f.cliente_ci = c.ci
GROUP BY f.cliente_ci, c.nombre
ORDER BY total_comprado DESC;
 
-- 6. Ventas totales por categoría
SELECT c.nombre AS categoria, SUM(df.subtotal) AS total_ventas
FROM detalle_factura df
JOIN productos p ON df.producto_id = p.id
JOIN categorias c ON p.categoria_id = c.id 
GROUP BY c.nombre
ORDER BY total_ventas DESC;

-- 7. Ventas promedio por cliente
SELECT f.cliente_ci, c.nombre, ROUND(AVG(f.total), 2) AS promedio_gasto 
FROM facturas_internas f
JOIN clientes c ON f.cliente_ci = c.ci
GROUP BY f.cliente_ci, c.nombre
ORDER BY promedio_gasto DESC;

-- 8. Detalle de facturas de un cliente específico (ej. 13334249)
SELECT f.id AS factura_id, p.nombre AS producto, df.cantidad, df.precio_unitario, df.subtotal 
FROM facturas_internas f
JOIN detalle_factura df ON f.id = df.factura_interna_id 
JOIN productos p ON df.producto_id = p.id
WHERE f.cliente_ci = '13334249' 
ORDER BY f.id;

-- 9. Facturas emitidas entre dos fechas
SELECT 
    f.id, 
    f.nro_factura, -- Agregué esto porque es más útil que solo el ID
    f.cliente_ci, 
    f.total, 
    f.fecha_emision 
FROM facturas_internas f 
WHERE DATE(f.fecha_emision) BETWEEN '2025-09-01' AND '2025-09-15' 
ORDER BY f.fecha_emision 
LIMIT 0, 1000;

-- 10. Total de ventas de productos que contienen "Pan de avena"
SELECT p.nombre, SUM(df.subtotal) AS total_ingresos 
FROM detalle_factura df
JOIN productos p ON df.producto_id = p.id
WHERE p.nombre LIKE '%Pan de avena%' 
GROUP BY p.nombre
ORDER BY total_ingresos DESC;

-- %%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%

-- INVENTARIO Y STOCK

-- %%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%

-- 11. Productos con stock bajo su mínimo 
SELECT id, nombre, stock, stock_minimo 
FROM productos 
WHERE stock < stock_minimo 
ORDER BY stock;

-- valores tienen tus productos
SELECT nombre, stock, stock_minimo 
FROM productos;

-- 12. Insumos con stock bajo su mínimo 
SELECT nombre, stock_actual, stock_minimo 
FROM insumos;

-- Ahora corre de nuevo tu consulta:
SELECT id, nombre, stock_actual, stock_minimo 
FROM insumos 
WHERE stock_actual < stock_minimo 
ORDER BY stock_actual;

SELECT id, nombre, stock_actual, stock_minimo 
FROM insumos 
WHERE stock_actual < stock_minimo 
ORDER BY stock_actual;

-- 13. Stock actual de cada producto 
SELECT nombre, stock 
FROM productos 
ORDER BY nombre;

-- 14. Stock actual de cada insumo 
SELECT nombre, stock_actual, unidad_medida 
FROM insumos 
ORDER BY nombre;

-- 15 Insumos Estratégicos (Uso en Recetas)
SELECT dr.insumo_id, i.nombre, COUNT(dr.receta_id) AS veces_usado 
FROM detalle_receta dr
JOIN insumos i ON dr.insumo_id = i.id 
GROUP BY dr.insumo_id, i.nombre
HAVING COUNT(dr.receta_id) > 3
ORDER BY veces_usado DESC;
 
-- 16. Productos que necesitan reposición según ventas recientes 
SELECT 
    p.id, 
    p.nombre, 
    p.stock AS stock_en_panaderia, 
    SUM(df.cantidad) AS unidades_vendidas_septiembre
FROM productos p 
JOIN detalle_factura df ON p.id = df.producto_id  
JOIN facturas_internas f ON df.factura_interna_id = f.id 
WHERE DATE(f.fecha_emision) BETWEEN '2025-09-01' AND '2025-09-15' 
GROUP BY p.id, p.nombre, p.stock 
ORDER BY unidades_vendidas_septiembre DESC;

-- 17. Productos que no se han vendido en un período determinado 
SELECT p.id, p.nombre 
FROM productos p  
WHERE NOT EXISTS (     
    SELECT 1  
    FROM detalle_factura df  
    JOIN facturas_internas f ON df.factura_interna_id = f.id 
    WHERE df.producto_id = p.id  
    AND DATE(f.fecha_emision) BETWEEN '2025-09-01' AND '2025-09-15' 
) 
ORDER BY p.nombre 
LIMIT 0, 1000;
 
-- %%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%

-- PRODUCCIÓN

-- %%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%

-- 18. Producciones terminadas y pendientes 
SELECT 
    id, 
    lote_codigo, -- Agregué este porque es tu identificador de trazabilidad
    descripcion, 
    fecha_programada, 
    hora_inicio, 
    estado  
FROM producciones 
ORDER BY estado DESC, fecha_programada 
LIMIT 0, 1000;

-- 19. Participantes de una producción específica
SELECT p.usuario_codigo, u.nombre
FROM participaciones p
JOIN usuarios u ON p.usuario_codigo = u.codigo 
WHERE p.produccion_id = 1;

-- 20. Tiempo promedio de producción por receta (en minutos)
-- Nombre exacto: tiempo_preparacion_min
SELECT id, nombre, tiempo_preparacion_min AS tiempo_estandar_minutos 
FROM recetas 
ORDER BY tiempo_preparacion_min DESC;

-- 21. Cantidad de unidades producidas por receta (solo terminadas)
-- Nombre exacto: rendimiento_estimado (según tu DESC)
SELECT prd.nombre AS producto, SUM(r.rendimiento_estimado) AS unidades_totales 
FROM producciones p
JOIN recetas r ON p.receta_id = r.id 
JOIN productos prd ON prd.id = r.producto_id 
WHERE p.estado = 'completada' 
GROUP BY prd.nombre;

-- 22. Producciones realizadas en un rango de fechas 
-- Corregido: Usando 'fecha_programada'
SELECT id, lote_codigo, descripcion, fecha_programada, estado
FROM producciones
WHERE fecha_programada BETWEEN '2026-04-01' AND '2026-04-30'
ORDER BY fecha_programada;

DESC detalle_receta;

-- 23. Ingredientes (insumos) más utilizados en la producción real
-- Corregido: El estado es 'completada'
SELECT 
    i.nombre AS insumo, 
    SUM(dr.cantidad_necesaria) AS total_consumido, 
    i.unidad_medida
FROM detalle_receta dr
JOIN insumos i ON dr.insumo_id = i.id
JOIN recetas r ON dr.receta_id = r.id
JOIN producciones p ON p.receta_id = r.id
WHERE p.estado = 'completada'
GROUP BY i.id, i.nombre, i.unidad_medida  
ORDER BY total_consumido DESC 
LIMIT 0, 1000;

-- 24. Recetas más frecuentes en producción 
SELECT r.nombre, COUNT(p.id) AS veces_producida 
FROM producciones p
JOIN recetas r ON p.receta_id = r.id
GROUP BY r.nombre
ORDER BY veces_producida DESC;
DESC recetas;

select * from roles;


-- %%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%

-- COMPRAS Y PROVEEDORES

-- %%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%


SHOW TABLES;

-- 25. Compras de INSUMOS por proveedor
SELECT 
    nc.id AS nota_id, 
    nc.nro_comprobante,
    p.nombre_contacto AS proveedor, 
    p.empresa, 
    i.nombre AS insumo, 
    ci.cantidad, 
    ci.precio_unitario,
    ci.subtotal 
FROM notas_compra nc
JOIN proveedores p ON nc.proveedor_codigo = p.codigo
JOIN compra_insumo ci ON nc.id = ci.nota_compra_id
JOIN insumos i ON ci.insumo_id = i.id
ORDER BY nc.fecha_pedido DESC; -- Nombre exacto de tu tabla

DESC compra_producto;
-- 26. Compras de PRODUCTOS por proveedor (Reventa)
SELECT 
    nc.id AS nota_id, 
    p.nombre_contacto AS proveedor, 
    p.empresa, 
    pr.nombre AS producto, 
    cp.cantidad, 
    cp.precio_compra_unitario, -- Nombre exacto de tu tabla
    cp.subtotal 
FROM notas_compra nc
JOIN proveedores p ON nc.proveedor_codigo = p.codigo
JOIN compra_producto cp ON nc.id = cp.nota_compra_id
JOIN productos pr ON cp.producto_id = pr.id
ORDER BY nc.fecha_pedido DESC; -- Usando el nombre confirmado anteriormente

-- 27. Total gastado en compras por mes (Insumos + Productos)
-- Corregido: nombres de tablas en singular
SELECT mes, SUM(monto) AS total_gastado
FROM (
    -- Parte 1: Gastos en Insumos
    -- Usamos 'fecha_pedido' y 'subtotal' de compra_insumo
    SELECT DATE_FORMAT(nc.fecha_pedido, '%Y-%m') AS mes, ci.subtotal AS monto
    FROM notas_compra nc 
    JOIN compra_insumo ci ON nc.id = ci.nota_compra_id
    
    UNION ALL
    
    -- Parte 2: Gastos en Productos de Reventa
    -- Corregido: 'compra_producto', 'fecha_pedido' y 'subtotal'
    SELECT DATE_FORMAT(nc.fecha_pedido, '%Y-%m') AS mes, cp.subtotal AS monto
    FROM notas_compra nc 
    JOIN compra_producto cp ON nc.id = cp.nota_compra_id
) AS movimientos
GROUP BY mes
ORDER BY mes;

-- 28. Proveedores con mayor frecuencia de compra
SELECT nc.proveedor_codigo, p.empresa, COUNT(nc.id) AS veces_comprado 
FROM notas_compra nc
JOIN proveedores p ON nc.proveedor_codigo = p.codigo
GROUP BY nc.proveedor_codigo, p.empresa 
ORDER BY veces_comprado DESC;

DESC compra_insumo;

-- 29. Seguimiento de Pedidos (Compras que aún no han llegado)
-- Corregido: 'fecha_pedido' y estado 'solicitado'
SELECT 
    id, 
    nro_comprobante,
    fecha_pedido, 
    fecha_recepcion, 
    proveedor_codigo,
    estado
FROM notas_compra
WHERE estado = 'solicitado' 
ORDER BY fecha_pedido;

DESC notas_compra;

-- 30. Ranking de Insumos por volumen de compra
-- Corregido: nombre de tabla a 'compra_insumo'
SELECT 
    i.nombre AS insumo, 
    SUM(ci.cantidad) AS total_unidades, 
    i.unidad_medida 
FROM compra_insumo ci -- Tabla en singular
JOIN insumos i ON ci.insumo_id = i.id 
GROUP BY i.id, i.nombre, i.unidad_medida 
ORDER BY total_unidades DESC;

select * from usuarios;

-- ######################################################
-- PROCEDIMIENTO ALMACENADO PARA AGREGAR PRODUCTO
-- ######################################################

DELIMITER //

-- 1. AGREGAR PRODUCTO (Ajustado a precio_venta y stock_actual)
CREATE PROCEDURE agregar_producto( 
    IN p_nombre VARCHAR(50),
    IN p_precio DECIMAL(12, 2), 
    IN p_categoria_id INTEGER,
    IN p_stock_minimo INTEGER
)
BEGIN
    INSERT INTO productos (nombre, precio_venta, categoria_id, stock_minimo, stock_actual, es_producido, estado) 
    VALUES (p_nombre, p_precio, p_categoria_id, p_stock_minimo, 0, 1, 'activo'); 
END //

-- 2. ELIMINAR PRODUCTO (Por ID es más seguro, pero mantengo nombre por tu requerimiento)
CREATE PROCEDURE eliminar_producto( IN p_nombre VARCHAR(50) )
BEGIN
    DELETE FROM productos WHERE nombre = p_nombre; 
END //

-- 3. ACTUALIZAR PRECIO
CREATE PROCEDURE actualizar_precio_producto( 
    IN p_nombre VARCHAR(50),
    IN p_nuevo_precio DECIMAL(12,2)
)
BEGIN
    UPDATE productos 
    SET precio_venta = p_nuevo_precio 
    WHERE nombre = p_nombre;
END //

-- 4. INSERTAR USUARIO (Ajustado a la estructura de Laravel)
CREATE PROCEDURE insertar_usuario( 
    IN p_codigo VARCHAR(10),
    IN p_nombre VARCHAR(50), 
    IN p_sexo CHAR(1), 
    IN p_password VARCHAR(250), 
    IN p_telefono VARCHAR(15), 
    IN p_rol_id INTEGER
)
BEGIN
    INSERT INTO usuarios (codigo, nombre, sexo, password, telefono, rol_id) 
    VALUES (p_codigo, p_nombre, p_sexo, p_password, p_telefono, p_rol_id); 
END //

-- 5. GENERAR COMPRA DE PRODUCTO (Actualizado a 'compra_producto' y sus IDs)
CREATE PROCEDURE generar_compra_producto( 
    IN p_nota_compra_id INTEGER,
    IN p_producto_id INTEGER, 
    IN p_cantidad INTEGER, 
    IN p_precio DECIMAL(12,2)
)
BEGIN
    DECLARE v_total DECIMAL(12,2);
    SET v_total = p_cantidad * p_precio;
    
    INSERT INTO compra_producto (nota_compra_id, producto_id, cantidad, precio, total)
    VALUES (p_nota_compra_id, p_producto_id, p_cantidad, p_precio, v_total);
END //

DELIMITER ;


--  Triggers

DROP TRIGGER IF EXISTS tr_actualizar_stock_insumo_AI;
DELIMITER //

-- 1. Trigger para Insumos (Se activa al comprar materia prima como harina, azúcar, etc.)
CREATE TRIGGER tr_actualizar_stock_insumo_AI 
AFTER INSERT ON compra_insumo 
FOR EACH ROW 
BEGIN 
    UPDATE insumos  
    SET stock_actual = stock_actual + NEW.cantidad  
    WHERE id = NEW.insumo_id; 
END //

-- 2. Trigger para Productos (Se activa al comprar productos terminados para reventa)
CREATE TRIGGER tr_actualizar_stock_producto_AI
AFTER INSERT ON compra_producto
FOR EACH ROW
BEGIN
    UPDATE productos 
    SET stock = stock + NEW.cantidad 
    WHERE id = NEW.producto_id;
END //

DELIMITER ;

-- ######################################################
-- 3. TRIGGERS (LÓGICA AUTOMÁTICA)
-- ######################################################

DELIMITER //

-- A. Actualizar stock de Insumos al comprar materia prima
CREATE TRIGGER tr_actualizar_stock_insumo_AI
AFTER INSERT ON compra_insumo
FOR EACH ROW
BEGIN
    UPDATE insumos 
    SET stock_actual = stock_actual + NEW.cantidad 
    WHERE id = NEW.insumo_id;
END //

-- B. Actualizar stock de Productos al comprar para reventa
CREATE TRIGGER tr_actualizar_stock_producto_AI
AFTER INSERT ON compra_producto
FOR EACH ROW
BEGIN
    UPDATE productos 
    SET stock_actual = stock_actual + NEW.cantidad 
    WHERE id = NEW.producto_id;
END //

-- C. Lógica de Producción (Resta insumos y aumenta producto terminado)
CREATE TRIGGER tr_produccion_automatica_AI
AFTER INSERT ON producciones
FOR EACH ROW
BEGIN
    DECLARE done INT DEFAULT FALSE;
    DECLARE v_insumo_id INT;
    DECLARE v_cantidad_receta DECIMAL(12,2);
    
    -- Cursor para obtener los ingredientes de la receta
    DECLARE cur_receta CURSOR FOR 
        SELECT insumo_id, cantidad_necesaria FROM detalle_receta WHERE receta_id = NEW.receta_id;
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;

    -- Si la producción se marca como completada
    IF NEW.estado = 'completado' THEN
        OPEN cur_receta;
        read_loop: LOOP
            FETCH cur_receta INTO v_insumo_id, v_cantidad_receta;
            IF done THEN LEAVE read_loop; END IF;
            
            -- Descontar materia prima del inventario
            UPDATE insumos SET stock_actual = stock_actual - v_cantidad_receta WHERE id = v_insumo_id;
        END LOOP;
        CLOSE cur_receta;
        
        -- Aumentar el stock del producto final (Pan, Galletas, etc.)
        UPDATE productos p 
        JOIN recetas r ON p.id = r.producto_id 
        SET p.stock_actual = p.stock_actual + r.rendimiento_estimado
        WHERE r.id = NEW.receta_id;
    END IF;
END //

DELIMITER ;
