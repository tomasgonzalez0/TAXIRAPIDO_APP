# CODIGO CORREGIDO

**Nota:** Recordar cambiar en SCRIPT 00 la ruta de los archivos por su ruta propia. Ademas, al momento de correr un script sea en SYSTEM o en TAXIRAPIDO_APP solo estar conectado a ese esquema ya que puede generar conflictos. Al momento de correr SCRIPT 00 deben de crear una nueva conexion para el usuario TAXIRAPIDO_APP con contraseña TaxiApp2024

---

## 🗺️ Guía de ejecución — 3 scripts, 2 conexiones

### SCRIPT 00 → Conexión: SYSTEM
Limpia todo, crea tablespaces, crea los 4 usuarios (TAXIRAPIDO_APP, OPERADOR, ADMINISTRADOR, CONDUCTOR). Desde SYSTEM porque CREATE USER es privilegio exclusivo de DBA.

### SCRIPT 01 → Conexión: TAXIRAPIDO_APP
Crea tablas, secuencias, triggers, datos de prueba, vistas y roles. Todo en el esquema correcto, no en SYS.

### SCRIPT 02 → Conexión: SYSTEM (de nuevo)
Hace GRANT ROL_OPERADOR TO OPERADOR y GRANT ROL_ADMINISTRADOR TO ADMINISTRADOR. Luego muestra todas las verificaciones y el reporte ejecutivo integrado.

---

# SCRIPT 00
```
-- =============================================================================
-- SCRIPT 00 — SETUP INICIAL Y LIMPIEZA
-- CONEXIÓN REQUERIDA: SYSTEM
-- Servicio: XEPDB1 (Oracle XE 21c) o FREE (Oracle 23c Free)
-- =============================================================================
-- QUÉ HACE ESTE SCRIPT:
--   1. Limpia ejecuciones anteriores
--   2. Crea tablespaces (requiere DBA)
--   3. Crea TAXIRAPIDO_APP — usuario dueño del modelo
--   4. Crea OPERADOR, ADMINISTRADOR, CONDUCTOR — usuarios de negocio
--      NOTA: CREATE USER es privilegio exclusivo de DBA.
--      Por eso los 4 usuarios se crean aquí, desde SYSTEM.
-- =============================================================================

-- -------------------------------------------------------
-- PARTE A: LIMPIEZA TOTAL
-- -------------------------------------------------------
BEGIN
    FOR u IN (SELECT username FROM dba_users
              WHERE username IN ('TAXIRAPIDO_APP','OPERADOR','ADMINISTRADOR','CONDUCTOR')) LOOP
        EXECUTE IMMEDIATE 'DROP USER ' || u.username || ' CASCADE';
    END LOOP;
END;
/

BEGIN
    FOR r IN (SELECT role FROM dba_roles
              WHERE role IN ('ROL_OPERADOR','ROL_ADMINISTRADOR')) LOOP
        EXECUTE IMMEDIATE 'DROP ROLE ' || r.role;
    END LOOP;
END;
/

BEGIN
    FOR t IN (SELECT tablespace_name FROM dba_tablespaces
              WHERE tablespace_name IN ('TS_OPERACIONES','TS_ADMIN','TS_TEMP_TAXI')) LOOP
        BEGIN
            EXECUTE IMMEDIATE 'DROP TABLESPACE ' || t.tablespace_name
                || ' INCLUDING CONTENTS AND DATAFILES CASCADE CONSTRAINTS';
        EXCEPTION WHEN OTHERS THEN NULL;
        END;
    END LOOP;
END;
/

-- -------------------------------------------------------
-- PARTE B: TABLESPACES
-- Separación física: operaciones vs. administración
-- -------------------------------------------------------
CREATE TABLESPACE TS_OPERACIONES
DATAFILE 'D:\Oracle\oradata\TAXIRAPIDO\ts_operaciones01.dbf'
SIZE 200M AUTOEXTEND ON NEXT 20M MAXSIZE 2G
EXTENT MANAGEMENT LOCAL SEGMENT SPACE MANAGEMENT AUTO;

CREATE TABLESPACE TS_ADMIN
DATAFILE 'D:\Oracle\oradata\TAXIRAPIDO\ts_admin01.dbf'
SIZE 100M AUTOEXTEND ON NEXT 10M MAXSIZE 1G
EXTENT MANAGEMENT LOCAL SEGMENT SPACE MANAGEMENT AUTO;

CREATE TEMPORARY TABLESPACE TS_TEMP_TAXI
TEMPFILE 'D:\Oracle\oradata\TAXIRAPIDO\ts_temp01.dbf'
SIZE 100M AUTOEXTEND ON NEXT 10M MAXSIZE 500M;

-- -------------------------------------------------------
-- PARTE C: USUARIO DUEÑO DEL MODELO DE NEGOCIO
-- -------------------------------------------------------
CREATE USER TAXIRAPIDO_APP
IDENTIFIED BY TaxiApp2024
DEFAULT TABLESPACE TS_OPERACIONES
TEMPORARY TABLESPACE TS_TEMP_TAXI
QUOTA UNLIMITED ON TS_OPERACIONES
QUOTA UNLIMITED ON TS_ADMIN;

GRANT CREATE SESSION            TO TAXIRAPIDO_APP;
GRANT CREATE TABLE              TO TAXIRAPIDO_APP;
GRANT CREATE SEQUENCE           TO TAXIRAPIDO_APP;
GRANT CREATE TRIGGER            TO TAXIRAPIDO_APP;
GRANT CREATE VIEW               TO TAXIRAPIDO_APP;
GRANT CREATE ROLE               TO TAXIRAPIDO_APP;
GRANT GRANT ANY OBJECT PRIVILEGE TO TAXIRAPIDO_APP;

-- Acceso al diccionario de datos para verificaciones del taller
GRANT SELECT ON DBA_TABLESPACES TO TAXIRAPIDO_APP;
GRANT SELECT ON DBA_DATA_FILES  TO TAXIRAPIDO_APP;
GRANT SELECT ON DBA_SEGMENTS    TO TAXIRAPIDO_APP;
GRANT SELECT ON DBA_USERS       TO TAXIRAPIDO_APP;
GRANT SELECT ON DBA_ROLES       TO TAXIRAPIDO_APP;
GRANT SELECT ON DBA_ROLE_PRIVS  TO TAXIRAPIDO_APP;
GRANT SELECT ON DBA_TAB_PRIVS   TO TAXIRAPIDO_APP;
GRANT SELECT ON DBA_TS_QUOTAS   TO TAXIRAPIDO_APP;

-- -------------------------------------------------------
-- PARTE D: USUARIOS DE NEGOCIO
-- CREATE USER requiere DBA → se crean aquí desde SYSTEM
-- Los GRANT de permisos sobre objetos del esquema
-- se hacen en el SCRIPT 02 (una vez que los objetos existen)
-- -------------------------------------------------------

-- Operador: gestiona la operación diaria (despacho)
CREATE USER OPERADOR
IDENTIFIED BY Operador2024
DEFAULT TABLESPACE TS_OPERACIONES
TEMPORARY TABLESPACE TS_TEMP_TAXI
QUOTA 100M ON TS_OPERACIONES;

GRANT CREATE SESSION TO OPERADOR;

-- Administrador: gerencia, acceso total
CREATE USER ADMINISTRADOR
IDENTIFIED BY Admin2024
DEFAULT TABLESPACE TS_ADMIN
TEMPORARY TABLESPACE TS_TEMP_TAXI
QUOTA UNLIMITED ON TS_ADMIN
QUOTA 50M ON TS_OPERACIONES;

GRANT CREATE SESSION TO ADMINISTRADOR;

-- Conductor: acceso mínimo — solo sus propios viajes
CREATE USER CONDUCTOR
IDENTIFIED BY Conductor2024
DEFAULT TABLESPACE TS_OPERACIONES
TEMPORARY TABLESPACE TS_TEMP_TAXI
QUOTA 0 ON TS_OPERACIONES;

GRANT CREATE SESSION TO CONDUCTOR;

-- -------------------------------------------------------
-- VERIFICACIÓN: confirmar que los 4 usuarios existen
-- -------------------------------------------------------
SELECT USERNAME, DEFAULT_TABLESPACE, TEMPORARY_TABLESPACE, ACCOUNT_STATUS
FROM DBA_USERS
WHERE USERNAME IN ('TAXIRAPIDO_APP','OPERADOR','ADMINISTRADOR','CONDUCTOR')
ORDER BY USERNAME;

SELECT TABLESPACE_NAME, STATUS, CONTENTS
FROM DBA_TABLESPACES
WHERE TABLESPACE_NAME LIKE 'TS_%'
ORDER BY TABLESPACE_NAME;

-- =============================================================================
-- ✅ SCRIPT 00 COMPLETADO
-- Siguiente: conectarse como TAXIRAPIDO_APP y ejecutar SCRIPT 01
-- =============================================================================
```
# SCRIPT 01
```
-- =============================================================================
-- SCRIPT 01 — MODELO DE NEGOCIO TAXIRÁPIDO
-- CONEXIÓN REQUERIDA: TAXIRAPIDO_APP / TaxiApp2024
-- Servicio: XEPDB1 (Oracle XE 21c) o FREE (Oracle 23c Free)
-- =============================================================================
-- QUÉ HACE ESTE SCRIPT:
--   BLOQUE 1 → ARQUITECTURA: tablas en tablespaces correctos
--   BLOQUE 2 → SECUENCIAS:   generación automática de IDs + triggers
--   BLOQUE 3 → DATOS:        inserción de registros de prueba
--   BLOQUE 4 → VISTAS:       consultas seguras por rol
-- Los GRANTS a usuarios de negocio van en SCRIPT 02 (desde SYSTEM)
-- porque GRANT ANY OBJECT PRIVILEGE no alcanza para GRANT CREATE SESSION
-- ni para administrar cuotas de otros usuarios.
-- =============================================================================


-- =============================================================================
-- BLOQUE 1: ARQUITECTURA DE ORACLE
-- Autor: ARTEAGA GARCIA CARLOS ANDRES
-- =============================================================================

CREATE TABLE CONDUCTORES (
    ID_CONDUCTOR    NUMBER          PRIMARY KEY,
    NOMBRE          VARCHAR2(100)   NOT NULL,
    LICENCIA        VARCHAR2(20)    UNIQUE NOT NULL,
    TELEFONO        VARCHAR2(15),
    USUARIO_ORACLE  VARCHAR2(50),
    ESTADO          VARCHAR2(20)    DEFAULT 'ACTIVO',
    FECHA_REGISTRO  DATE            DEFAULT SYSDATE
) TABLESPACE TS_OPERACIONES;
-- USUARIO_ORACLE almacena el nombre del usuario Oracle del conductor,
-- lo que permite que la VISTA_VIAJES_CONDUCTOR filtre por sesión activa.

CREATE TABLE VEHICULOS (
    ID_VEHICULO   NUMBER       PRIMARY KEY,
    PLACA         VARCHAR2(10) UNIQUE NOT NULL,
    MARCA         VARCHAR2(50),
    MODELO        VARCHAR2(50),
    ANIO          NUMBER(4),
    ID_CONDUCTOR  NUMBER,
    FOREIGN KEY (ID_CONDUCTOR) REFERENCES CONDUCTORES(ID_CONDUCTOR)
) TABLESPACE TS_OPERACIONES;

CREATE TABLE VIAJES (
    ID_VIAJE      NUMBER         PRIMARY KEY,
    ID_CONDUCTOR  NUMBER         NOT NULL,
    ID_VEHICULO   NUMBER         NOT NULL,
    FECHA_HORA    DATE           DEFAULT SYSDATE,
    ORIGEN        VARCHAR2(200),
    DESTINO       VARCHAR2(200),
    DISTANCIA_KM  NUMBER(6,2),
    TARIFA        NUMBER(10,2),
    ESTADO        VARCHAR2(20),
    FOREIGN KEY (ID_CONDUCTOR) REFERENCES CONDUCTORES(ID_CONDUCTOR),
    FOREIGN KEY (ID_VEHICULO)  REFERENCES VEHICULOS(ID_VEHICULO)
) TABLESPACE TS_OPERACIONES;

CREATE TABLE FACTURAS (
    ID_FACTURA     NUMBER        PRIMARY KEY,
    ID_VIAJE       NUMBER        NOT NULL,
    FECHA_EMISION  DATE          DEFAULT SYSDATE,
    MONTO_TOTAL    NUMBER(10,2),
    IMPUESTOS      NUMBER(10,2),
    METODO_PAGO    VARCHAR2(20),
    FOREIGN KEY (ID_VIAJE) REFERENCES VIAJES(ID_VIAJE)
) TABLESPACE TS_ADMIN;
-- Facturas en TS_ADMIN: separación física de datos financieros

-- Verificación de arquitectura
SELECT TABLE_NAME AS "Tabla", TABLESPACE_NAME AS "Almacenada en"
FROM USER_TABLES
WHERE TABLE_NAME IN ('CONDUCTORES','VEHICULOS','VIAJES','FACTURAS')
ORDER BY TABLESPACE_NAME, TABLE_NAME;

SELECT TABLESPACE_NAME AS "Tablespace",
       ROUND(BYTES/1024/1024,2) AS "Tamaño (MB)",
       ROUND(MAXBYTES/1024/1024,2) AS "Máximo (MB)",
       AUTOEXTENSIBLE AS "Autoextensible"
FROM DBA_DATA_FILES
WHERE TABLESPACE_NAME IN ('TS_OPERACIONES','TS_ADMIN')
ORDER BY TABLESPACE_NAME;


-- =============================================================================
-- BLOQUE 2: SECUENCIAS + TRIGGERS
-- Autor: CORREA TORRES BRAYAN ALEXIS
-- =============================================================================

-- Conductores: hasta 999.999, cache 20 para contrataciones en lote
CREATE SEQUENCE SEQ_CONDUCTORES
START WITH 1    INCREMENT BY 1
MINVALUE 1      MAXVALUE 999999
NOCYCLE         CACHE 20;

-- Vehículos: flota moderada, cache pequeño
CREATE SEQUENCE SEQ_VEHICULOS
START WITH 1    INCREMENT BY 1
MINVALUE 1      MAXVALUE 999999
NOCYCLE         CACHE 10;

-- Viajes: ~3.000/día, rango amplio, cache grande para rendimiento
CREATE SEQUENCE SEQ_VIAJES
START WITH 1000 INCREMENT BY 1
MINVALUE 1      MAXVALUE 99999999
NOCYCLE         CACHE 50;

-- Facturas: ORDER garantiza consecutividad legal obligatoria
CREATE SEQUENCE SEQ_FACTURAS
START WITH 100000 INCREMENT BY 1
MINVALUE 100000   MAXVALUE 99999999
NOCYCLE           CACHE 30
ORDER;

-- Triggers: el ID se asigna automáticamente sin que el operador
-- tenga que escribir NEXTVAL en cada INSERT.
CREATE OR REPLACE TRIGGER TRG_CONDUCTORES_ID
BEFORE INSERT ON CONDUCTORES FOR EACH ROW
BEGIN
    IF :NEW.ID_CONDUCTOR IS NULL THEN
        :NEW.ID_CONDUCTOR := SEQ_CONDUCTORES.NEXTVAL;
    END IF;
END;
/

CREATE OR REPLACE TRIGGER TRG_VEHICULOS_ID
BEFORE INSERT ON VEHICULOS FOR EACH ROW
BEGIN
    IF :NEW.ID_VEHICULO IS NULL THEN
        :NEW.ID_VEHICULO := SEQ_VEHICULOS.NEXTVAL;
    END IF;
END;
/

CREATE OR REPLACE TRIGGER TRG_VIAJES_ID
BEFORE INSERT ON VIAJES FOR EACH ROW
BEGIN
    IF :NEW.ID_VIAJE IS NULL THEN
        :NEW.ID_VIAJE := SEQ_VIAJES.NEXTVAL;
    END IF;
END;
/

CREATE OR REPLACE TRIGGER TRG_FACTURAS_ID
BEFORE INSERT ON FACTURAS FOR EACH ROW
BEGIN
    IF :NEW.ID_FACTURA IS NULL THEN
        :NEW.ID_FACTURA := SEQ_FACTURAS.NEXTVAL;
    END IF;
END;
/

-- Verificación de secuencias
SELECT SEQUENCE_NAME AS "Secuencia", MIN_VALUE AS "Mín", MAX_VALUE AS "Máx",
       LAST_NUMBER AS "Último Nro", CACHE_SIZE AS "Cache"
FROM USER_SEQUENCES WHERE SEQUENCE_NAME LIKE 'SEQ_%'
ORDER BY SEQUENCE_NAME;


-- =============================================================================
-- BLOQUE 3: INSERCIÓN DE DATOS DE PRUEBA
-- Los triggers asignan IDs automáticamente — no se especifica ID en el INSERT
-- =============================================================================

INSERT INTO CONDUCTORES (NOMBRE, LICENCIA, TELEFONO, USUARIO_ORACLE, ESTADO)
VALUES ('Juan Pérez Gómez',    'LIC-2024-001', '3001234567', 'CONDUCTOR',  'ACTIVO');

INSERT INTO CONDUCTORES (NOMBRE, LICENCIA, TELEFONO, USUARIO_ORACLE, ESTADO)
VALUES ('María López Castro',  'LIC-2024-002', '3107654321', 'CONDUCTOR2', 'ACTIVO');

INSERT INTO CONDUCTORES (NOMBRE, LICENCIA, TELEFONO, USUARIO_ORACLE, ESTADO)
VALUES ('Carlos Ramírez Silva','LIC-2024-003', '3209876543', 'CONDUCTOR3', 'ACTIVO');

INSERT INTO VEHICULOS (PLACA, MARCA, MODELO, ANIO, ID_CONDUCTOR)
VALUES ('ABC123', 'Toyota',    'Corolla', 2023, 1);

INSERT INTO VEHICULOS (PLACA, MARCA, MODELO, ANIO, ID_CONDUCTOR)
VALUES ('DEF456', 'Chevrolet', 'Spark',   2022, 2);

INSERT INTO VEHICULOS (PLACA, MARCA, MODELO, ANIO, ID_CONDUCTOR)
VALUES ('GHI789', 'Renault',   'Logan',   2024, 3);

INSERT INTO VIAJES (ID_CONDUCTOR,ID_VEHICULO,ORIGEN,DESTINO,DISTANCIA_KM,TARIFA,ESTADO)
VALUES (1,1,'Centro de Medellín','Aeropuerto José María Córdova',25.5,45000,'COMPLETADO');

INSERT INTO VIAJES (ID_CONDUCTOR,ID_VEHICULO,ORIGEN,DESTINO,DISTANCIA_KM,TARIFA,ESTADO)
VALUES (2,2,'Laureles','Centro Comercial El Tesoro',12.3,22000,'COMPLETADO');

INSERT INTO VIAJES (ID_CONDUCTOR,ID_VEHICULO,ORIGEN,DESTINO,DISTANCIA_KM,TARIFA,ESTADO)
VALUES (3,3,'Envigado','Universidad de Antioquia',18.7,32000,'EN_CURSO');

INSERT INTO VIAJES (ID_CONDUCTOR,ID_VEHICULO,ORIGEN,DESTINO,DISTANCIA_KM,TARIFA,ESTADO)
VALUES (1,1,'El Poblado','Estadio Atanasio Girardot',15.2,28000,'EN_CURSO');

INSERT INTO FACTURAS (ID_VIAJE,MONTO_TOTAL,IMPUESTOS,METODO_PAGO)
VALUES (1000,45000,7200,'EFECTIVO');

INSERT INTO FACTURAS (ID_VIAJE,MONTO_TOTAL,IMPUESTOS,METODO_PAGO)
VALUES (1001,22000,3520,'TARJETA');

COMMIT;

-- Inserción masiva: simula un día de operaciones
BEGIN
    FOR i IN 1..10 LOOP
        INSERT INTO VIAJES (ID_CONDUCTOR,ID_VEHICULO,ORIGEN,DESTINO,
                            DISTANCIA_KM,TARIFA,ESTADO)
        VALUES (
            MOD(i,3)+1, MOD(i,3)+1,
            'Barrio '||i||' - Medellín',
            'Destino '||i||' - Medellín',
            ROUND(DBMS_RANDOM.VALUE(5,50),2),
            ROUND(DBMS_RANDOM.VALUE(10000,80000),-3),
            CASE WHEN MOD(i,2)=0 THEN 'COMPLETADO' ELSE 'EN_CURSO' END
        );
    END LOOP;
    COMMIT;
END;
/

-- Verificar datos insertados
SELECT ID_CONDUCTOR, NOMBRE, USUARIO_ORACLE, ESTADO FROM CONDUCTORES;
SELECT ID_VEHICULO, PLACA, MARCA, MODELO FROM VEHICULOS;
SELECT COUNT(*) AS "Total Viajes" FROM VIAJES;
SELECT ID_FACTURA, ID_VIAJE, MONTO_TOTAL, METODO_PAGO FROM FACTURAS;


-- =============================================================================
-- BLOQUE 4: VISTAS
-- Autor: PATIÑO JARAMILLO CRISTIAN ALBIERY
-- NOTA: Las vistas se crean ANTES de que SCRIPT 02 haga los GRANTS,
-- así cuando lleguen los GRANT los objetos ya existen.
-- =============================================================================

-- Vista 1: Viajes en curso — para operadores de despacho
CREATE OR REPLACE VIEW VISTA_VIAJES_ACTIVOS AS
SELECT V.ID_VIAJE                                   AS "ID Viaje",
       C.NOMBRE                                     AS "Conductor",
       C.TELEFONO                                   AS "Teléfono",
       VH.PLACA                                     AS "Placa",
       VH.MARCA||' '||VH.MODELO                    AS "Vehículo",
       V.ORIGEN                                     AS "Origen",
       V.DESTINO                                    AS "Destino",
       V.DISTANCIA_KM                               AS "Distancia (Km)",
       V.TARIFA                                     AS "Tarifa ($)",
       TO_CHAR(V.FECHA_HORA,'DD/MM/YYYY HH24:MI')  AS "Fecha y Hora",
       V.ESTADO                                     AS "Estado"
FROM VIAJES V
JOIN CONDUCTORES C  ON V.ID_CONDUCTOR = C.ID_CONDUCTOR
JOIN VEHICULOS   VH ON V.ID_VEHICULO  = VH.ID_VEHICULO
WHERE V.ESTADO = 'EN_CURSO';

-- Vista 2: Viajes del conductor conectado
-- Filtra por USUARIO_ORACLE = SESSION_USER (usuario de sesión activa)
CREATE OR REPLACE VIEW VISTA_VIAJES_CONDUCTOR AS
SELECT V.ID_VIAJE                                   AS "ID Viaje",
       V.ORIGEN                                     AS "Origen",
       V.DESTINO                                    AS "Destino",
       V.DISTANCIA_KM                               AS "Distancia (Km)",
       V.TARIFA                                     AS "Tarifa ($)",
       TO_CHAR(V.FECHA_HORA,'DD/MM/YYYY HH24:MI')  AS "Fecha y Hora",
       V.ESTADO                                     AS "Estado"
FROM VIAJES V
JOIN CONDUCTORES C ON V.ID_CONDUCTOR = C.ID_CONDUCTOR
WHERE UPPER(C.USUARIO_ORACLE) = UPPER(SYS_CONTEXT('USERENV','SESSION_USER'));

-- Vista 3: Reporte financiero — exclusiva para administradores
CREATE OR REPLACE VIEW VISTA_REPORTES_FINANCIEROS AS
SELECT F.ID_FACTURA                                 AS "N° Factura",
       TO_CHAR(F.FECHA_EMISION,'DD/MM/YYYY')        AS "Fecha Emisión",
       C.NOMBRE                                     AS "Conductor",
       V.ORIGEN                                     AS "Origen",
       V.DESTINO                                    AS "Destino",
       V.DISTANCIA_KM                               AS "Distancia (Km)",
       F.MONTO_TOTAL                                AS "Monto Total ($)",
       F.IMPUESTOS                                  AS "IVA ($)",
       (F.MONTO_TOTAL - F.IMPUESTOS)               AS "Subtotal ($)",
       F.METODO_PAGO                                AS "Método de Pago"
FROM FACTURAS F
JOIN VIAJES      V ON F.ID_VIAJE     = V.ID_VIAJE
JOIN CONDUCTORES C ON V.ID_CONDUCTOR = C.ID_CONDUCTOR
ORDER BY F.FECHA_EMISION DESC;

-- Vista 4: Disponibilidad de flota — para asignación de viajes
CREATE OR REPLACE VIEW VISTA_DISPONIBILIDAD_VEHICULOS AS
SELECT VH.ID_VEHICULO                               AS "ID Vehículo",
       VH.PLACA                                     AS "Placa",
       VH.MARCA||' '||VH.MODELO                    AS "Vehículo",
       VH.ANIO                                      AS "Año",
       C.NOMBRE                                     AS "Conductor Asignado",
       C.TELEFONO                                   AS "Teléfono",
       CASE
           WHEN EXISTS (SELECT 1 FROM VIAJES V2
                        WHERE V2.ID_VEHICULO = VH.ID_VEHICULO
                          AND V2.ESTADO = 'EN_CURSO')
           THEN 'OCUPADO' ELSE 'DISPONIBLE'
       END                                          AS "Disponibilidad"
FROM VEHICULOS VH
JOIN CONDUCTORES C ON VH.ID_CONDUCTOR = C.ID_CONDUCTOR
ORDER BY "Disponibilidad", VH.PLACA;

-- Vista 5: Rendimiento por conductor — para gerencia
CREATE OR REPLACE VIEW VISTA_RENDIMIENTO_CONDUCTORES AS
SELECT C.ID_CONDUCTOR                                           AS "ID",
       C.NOMBRE                                                 AS "Conductor",
       C.LICENCIA                                               AS "Licencia",
       C.ESTADO                                                 AS "Estado",
       COUNT(V.ID_VIAJE)                                        AS "Total Viajes",
       NVL(ROUND(SUM(V.TARIFA),2),0)                           AS "Ingresos Totales ($)",
       NVL(ROUND(AVG(V.TARIFA),2),0)                           AS "Tarifa Promedio ($)",
       NVL(ROUND(SUM(V.DISTANCIA_KM),2),0)                    AS "KM Totales",
       COUNT(CASE WHEN V.ESTADO='COMPLETADO' THEN 1 END)        AS "Viajes Completados",
       COUNT(CASE WHEN V.ESTADO='EN_CURSO'   THEN 1 END)        AS "Viajes en Curso"
FROM CONDUCTORES C
LEFT JOIN VIAJES V ON C.ID_CONDUCTOR = V.ID_CONDUCTOR
GROUP BY C.ID_CONDUCTOR, C.NOMBRE, C.LICENCIA, C.ESTADO
ORDER BY "Ingresos Totales ($)" DESC;

--Vista 6: Vista Vehiculo Conductor
CREATE OR REPLACE VIEW VISTA_VEHICULO_CONDUCTOR AS
SELECT VH.ID_VEHICULO, VH.PLACA, VH.MARCA, VH.MODELO, VH.ANIO
FROM VEHICULOS VH
JOIN CONDUCTORES C ON C.ID_CONDUCTOR = VH.ID_CONDUCTOR
WHERE UPPER(C.USUARIO_ORACLE) = UPPER(SYS_CONTEXT('USERENV','SESSION_USER'));

-- Verificación de vistas
SELECT VIEW_NAME AS "Vista Creada"
FROM USER_VIEWS
WHERE VIEW_NAME LIKE 'VISTA_%'
ORDER BY VIEW_NAME;

-- Prueba de vistas (ejecutadas como TAXIRAPIDO_APP)
SELECT * FROM VISTA_VIAJES_ACTIVOS;
SELECT * FROM VISTA_DISPONIBILIDAD_VEHICULOS;
SELECT * FROM VISTA_RENDIMIENTO_CONDUCTORES;
SELECT * FROM VISTA_REPORTES_FINANCIEROS;


-- =============================================================================
-- BLOQUE 5: ROLES (creación y asignación de permisos sobre objetos propios)
-- Autor: GONZALEZ ZAPATA TOMAS
-- TAXIRAPIDO_APP puede crear roles y hacer GRANT sobre sus propios objetos.
-- Lo que NO puede hacer es CREATE USER ni GRANT CREATE SESSION
-- (esos van en SCRIPT 02 desde SYSTEM).
-- =============================================================================

CREATE ROLE ROL_OPERADOR;

-- El operador gestiona la parte operativa — no ve finanzas
GRANT SELECT, INSERT, UPDATE, DELETE ON CONDUCTORES TO ROL_OPERADOR;
GRANT SELECT, INSERT, UPDATE, DELETE ON VEHICULOS   TO ROL_OPERADOR;
GRANT SELECT, INSERT, UPDATE, DELETE ON VIAJES      TO ROL_OPERADOR;
GRANT SELECT ON SEQ_VIAJES       TO ROL_OPERADOR;
GRANT SELECT ON SEQ_CONDUCTORES  TO ROL_OPERADOR;
GRANT SELECT ON SEQ_VEHICULOS    TO ROL_OPERADOR;
GRANT SELECT ON VISTA_VIAJES_ACTIVOS           TO ROL_OPERADOR;
GRANT SELECT ON VISTA_DISPONIBILIDAD_VEHICULOS TO ROL_OPERADOR;

CREATE ROLE ROL_ADMINISTRADOR;

-- El administrador tiene acceso total incluido finanzas
GRANT SELECT, INSERT, UPDATE, DELETE ON CONDUCTORES TO ROL_ADMINISTRADOR;
GRANT SELECT, INSERT, UPDATE, DELETE ON VEHICULOS   TO ROL_ADMINISTRADOR;
GRANT SELECT, INSERT, UPDATE, DELETE ON VIAJES      TO ROL_ADMINISTRADOR;
GRANT SELECT, INSERT, UPDATE, DELETE ON FACTURAS    TO ROL_ADMINISTRADOR;
GRANT SELECT ON SEQ_VIAJES       TO ROL_ADMINISTRADOR;
GRANT SELECT ON SEQ_CONDUCTORES  TO ROL_ADMINISTRADOR;
GRANT SELECT ON SEQ_VEHICULOS    TO ROL_ADMINISTRADOR;
GRANT SELECT ON SEQ_FACTURAS     TO ROL_ADMINISTRADOR;
GRANT SELECT ON VISTA_VIAJES_ACTIVOS           TO ROL_ADMINISTRADOR;
GRANT SELECT ON VISTA_REPORTES_FINANCIEROS     TO ROL_ADMINISTRADOR;
GRANT SELECT ON VISTA_DISPONIBILIDAD_VEHICULOS TO ROL_ADMINISTRADOR;
GRANT SELECT ON VISTA_VIAJES_CONDUCTOR         TO ROL_ADMINISTRADOR;
GRANT SELECT ON VISTA_RENDIMIENTO_CONDUCTORES  TO ROL_ADMINISTRADOR;

-- Permisos individuales para el CONDUCTOR (no usa rol, acceso mínimo)
GRANT SELECT ON VISTA_VIAJES_CONDUCTOR TO CONDUCTOR;
GRANT SELECT ON VISTA_VEHICULO_CONDUCTOR TO CONDUCTOR;

-- Verificación de roles
SELECT GRANTED_ROLE AS ROLE
FROM USER_ROLE_PRIVS
ORDER BY GRANTED_ROLE;

-- =============================================================================
-- ✅ SCRIPT 01 COMPLETADO
-- Siguiente: conectarse como SYSTEM y ejecutar SCRIPT 02
-- =============================================================================
```
# SCRIPT 02
```
-- =============================================================================
-- SCRIPT 02 — GRANTS FINALES Y PRUEBAS DE SEGURIDAD
-- CONEXIÓN REQUERIDA: SYSTEM
-- Servicio: XEPDB1 (Oracle XE 21c) o FREE (Oracle 23c Free)
-- =============================================================================
-- QUÉ HACE ESTE SCRIPT:
--   1. Asigna roles a usuarios de negocio (requiere DBA)
--   2. Verifica el sistema completo de usuarios y permisos
--   3. Muestra el reporte ejecutivo integrado de los 4 temas
-- Por qué desde SYSTEM: GRANT <rol_del_sistema> TO <usuario>
-- requiere privilegios de DBA cuando el rol es externo al esquema.
-- =============================================================================

-- -------------------------------------------------------
-- PARTE A: ASIGNAR ROLES A USUARIOS DE NEGOCIO
-- -------------------------------------------------------

-- OPERADOR: recibe el rol operativo creado por TAXIRAPIDO_APP
GRANT ROL_OPERADOR      TO OPERADOR;

-- ADMINISTRADOR: recibe el rol de administración completo
GRANT ROL_ADMINISTRADOR TO ADMINISTRADOR;

-- -------------------------------------------------------
-- PARTE B: VERIFICACIONES COMPLETAS DEL SISTEMA
-- Autor: GONZALEZ ZAPATA TOMAS (Usuarios)
-- -------------------------------------------------------

-- Ver todos los usuarios del sistema TaxiRápido
SELECT USERNAME          AS "Usuario",
       DEFAULT_TABLESPACE AS "Tablespace Defecto",
       TEMPORARY_TABLESPACE AS "Tablespace Temp",
       ACCOUNT_STATUS    AS "Estado"
FROM DBA_USERS
WHERE USERNAME IN ('TAXIRAPIDO_APP','OPERADOR','ADMINISTRADOR','CONDUCTOR')
ORDER BY USERNAME;

-- Ver roles de negocio existentes
SELECT ROLE AS "Roles del Sistema"
FROM DBA_ROLES
WHERE ROLE LIKE 'ROL_%'
ORDER BY ROLE;

-- Ver qué rol tiene cada usuario
SELECT GRANTEE      AS "Usuario",
       GRANTED_ROLE AS "Rol Asignado"
FROM DBA_ROLE_PRIVS
WHERE GRANTEE IN ('OPERADOR','ADMINISTRADOR','CONDUCTOR')
ORDER BY GRANTEE;

-- Ver permisos a nivel de objeto por usuario
SELECT GRANTEE    AS "Usuario",
       PRIVILEGE  AS "Permiso",
       TABLE_NAME AS "Objeto"
FROM DBA_TAB_PRIVS
WHERE GRANTEE IN ('OPERADOR','ADMINISTRADOR','CONDUCTOR')
  AND OWNER = 'TAXIRAPIDO_APP'
ORDER BY GRANTEE, TABLE_NAME;

-- Ver cuotas de espacio en disco por usuario
SELECT USERNAME,
       TABLESPACE_NAME,
       CASE WHEN MAX_BYTES = -1 THEN 'UNLIMITED'
            ELSE TO_CHAR(ROUND(MAX_BYTES/1024/1024,0)) || ' MB'
       END AS "Cuota Asignada"
FROM DBA_TS_QUOTAS
WHERE USERNAME IN ('TAXIRAPIDO_APP','OPERADOR','ADMINISTRADOR','CONDUCTOR')
ORDER BY USERNAME, TABLESPACE_NAME;

-- -------------------------------------------------------
-- PARTE C: PRUEBAS DE SEGURIDAD — DEMOSTRACIÓN DEL CONTROL DE ACCESO
-- -------------------------------------------------------
-- INSTRUCCIÓN PARA EL VIDEO:
-- Cada bloque de CONNECT a continuación se prueba abriendo
-- una nueva hoja de trabajo en SQL Developer con esa conexión.
-- Los comentarios "DEBE funcionar" / "NO debe funcionar"
-- muestran el comportamiento esperado de cada rol.

-- >> COMO OPERADOR (abrir nueva hoja con conexión OPERADOR)
-- CONNECT OPERADOR/Operador2024@XEPDB1
--
-- SELECT * FROM TAXIRAPIDO_APP.CONDUCTORES WHERE ROWNUM <= 3;
-- → DEBE funcionar: el operador gestiona conductores ✓
--
-- INSERT INTO TAXIRAPIDO_APP.VIAJES
--     (ID_CONDUCTOR,ID_VEHICULO,ORIGEN,DESTINO,TARIFA,ESTADO)
-- VALUES (1,1,'Bello','Centro',20000,'EN_CURSO');
-- → DEBE funcionar: el operador registra viajes ✓
--
-- SELECT * FROM TAXIRAPIDO_APP.FACTURAS;
-- → NO debe funcionar: ORA-00942 (sin acceso a finanzas) ✓
--
-- SELECT * FROM TAXIRAPIDO_APP.VISTA_REPORTES_FINANCIEROS;
-- → NO debe funcionar: ORA-00942 ✓

-- >> COMO ADMINISTRADOR (abrir nueva hoja con conexión ADMINISTRADOR)
-- CONNECT ADMINISTRADOR/Admin2024@XEPDB1
--
-- SELECT * FROM TAXIRAPIDO_APP.FACTURAS WHERE ROWNUM <= 3;
-- → DEBE funcionar: acceso total incluido finanzas ✓
--
-- SELECT * FROM TAXIRAPIDO_APP.VISTA_REPORTES_FINANCIEROS;
-- → DEBE funcionar: reporte financiero completo ✓
--
-- SELECT * FROM TAXIRAPIDO_APP.VISTA_RENDIMIENTO_CONDUCTORES;
-- → DEBE funcionar: ranking de conductores ✓

-- >> COMO CONDUCTOR (abrir nueva hoja con conexión CONDUCTOR)
-- CONNECT CONDUCTOR/Conductor2024@XEPDB1
--
-- SELECT * FROM TAXIRAPIDO_APP.VISTA_VIAJES_CONDUCTOR;
-- → DEBE funcionar: solo devuelve viajes donde USUARIO_ORACLE='CONDUCTOR' ✓
--
-- SELECT * FROM TAXIRAPIDO_APP.VIAJES;
-- → NO debe funcionar: ORA-00942 (sin acceso directo a tabla) ✓
--
-- INSERT INTO TAXIRAPIDO_APP.VIAJES
--     (ID_CONDUCTOR,ID_VEHICULO,ORIGEN,DESTINO,TARIFA,ESTADO)
-- VALUES (1,1,'A','B',5000,'EN_CURSO');
-- → NO debe funcionar: ORA-01031 insufficient privileges ✓

-- -------------------------------------------------------
-- PARTE D: REPORTE EJECUTIVO INTEGRADO
-- Ejecutar como SYSTEM o como TAXIRAPIDO_APP
-- Demuestra que los 4 temas trabajan juntos:
--   · Tablas en tablespaces separados     (Arquitectura)
--   · IDs generados por secuencias        (Secuencias)
--   · Solo acceden usuarios autorizados   (Usuarios)
--   · Información presentada por vistas   (Vistas)
-- -------------------------------------------------------

SELECT 'REPORTE EJECUTIVO - TAXIRÁPIDO S.A.S.' AS "Sistema",
       TO_CHAR(SYSDATE,'DD/MM/YYYY HH24:MI')   AS "Generado"
FROM DUAL;

-- Resumen operativo del día
SELECT
    (SELECT COUNT(*) FROM TAXIRAPIDO_APP.CONDUCTORES
     WHERE ESTADO = 'ACTIVO')                                 AS "Conductores Activos",
    (SELECT COUNT(*) FROM TAXIRAPIDO_APP.VEHICULOS)           AS "Vehículos en Flota",
    (SELECT COUNT(*) FROM TAXIRAPIDO_APP.VIAJES
     WHERE ESTADO = 'EN_CURSO')                               AS "Viajes en Curso",
    (SELECT COUNT(*) FROM TAXIRAPIDO_APP.VIAJES
     WHERE TRUNC(FECHA_HORA) = TRUNC(SYSDATE))                AS "Viajes Registrados Hoy",
    (SELECT NVL(SUM(MONTO_TOTAL),0) FROM TAXIRAPIDO_APP.FACTURAS
     WHERE TRUNC(FECHA_EMISION) = TRUNC(SYSDATE))             AS "Ingresos Hoy ($)"
FROM DUAL;

-- Ranking de conductores por ingresos
SELECT * FROM TAXIRAPIDO_APP.VISTA_RENDIMIENTO_CONDUCTORES;

-- Estado de la flota en tiempo real
SELECT * FROM TAXIRAPIDO_APP.VISTA_DISPONIBILIDAD_VEHICULOS;

-- Viajes activos en este momento
SELECT * FROM TAXIRAPIDO_APP.VISTA_VIAJES_ACTIVOS;

-- Reporte financiero
SELECT * FROM TAXIRAPIDO_APP.VISTA_REPORTES_FINANCIEROS;

-- Estado de secuencias: capacidad restante
SELECT SEQUENCE_NAME                               AS "Secuencia",
       LAST_NUMBER                                 AS "Último ID Generado",
       MAX_VALUE                                   AS "Capacidad Máxima",
       (MAX_VALUE - LAST_NUMBER)                   AS "IDs Disponibles",
       ROUND((LAST_NUMBER/MAX_VALUE)*100,2)        AS "% Usado"
FROM DBA_SEQUENCES
WHERE SEQUENCE_OWNER = 'TAXIRAPIDO_APP'
  AND SEQUENCE_NAME LIKE 'SEQ_%'
ORDER BY SEQUENCE_NAME;

-- Verificación final: tablespaces con su uso actual
SELECT TS.TABLESPACE_NAME  AS "Tablespace",
       TS.STATUS           AS "Estado",
       TS.CONTENTS         AS "Tipo",
       ROUND(NVL(SEG.BYTES_USADOS,0)/1024/1024,2) AS "Usado (MB)"
FROM DBA_TABLESPACES TS
LEFT JOIN (
    SELECT TABLESPACE_NAME, SUM(BYTES) AS BYTES_USADOS
    FROM DBA_SEGMENTS GROUP BY TABLESPACE_NAME
) SEG ON TS.TABLESPACE_NAME = SEG.TABLESPACE_NAME
WHERE TS.TABLESPACE_NAME LIKE 'TS_%'
ORDER BY TS.TABLESPACE_NAME;

-- =============================================================================
-- ✅ TALLER No. 2 COMPLETADO — TAXIRÁPIDO S.A.S.
-- ABI104 - ADMINISTRACIÓN DE BASES DE DATOS - ITM
-- =============================================================================
```
