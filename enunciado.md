## Contexto de la empresa

**TaxiRápido S.A.S.** es una empresa de transporte urbano con base en Medellín que opera mediante una plataforma digital propia. La empresa cuenta con una flota activa de vehículos asignados a conductores registrados, gestiona cientos de viajes diarios entre distintos puntos de la ciudad, y emite facturas electrónicas por cada servicio prestado. En su estructura interna participan tres perfiles operativos: operadores de despacho, quienes coordinan los viajes en tiempo real; administradores, quienes supervisan las finanzas y el rendimiento de la flota; y los propios conductores, quienes necesitan consultar su historial de servicios.

## Problemática actual

Hasta hace poco, TaxiRápido operaba con hojas de cálculo compartidas y una base de datos sin estructura de acceso definida. Esto generaba tres problemas críticos y documentados:

* **Primero**, cualquier persona con acceso al sistema podía consultar o modificar la información financiera de la empresa, incluyendo facturas y montos de recaudo, lo que representaba un riesgo de fraude interno y una violación a las políticas de auditoría.
* **Segundo**, los IDs de viajes y facturas se asignaban manualmente o con scripts ad hoc, lo que prodría duplicados, saltos en la numeración y, en el caso de las facturas, incumplimientos con los requisitos de consecutividad exigidos por la DIAN.
* **Tercero**, los conductores accedían a la tabla completa de viajes para consultar los suyos propios, exponiéndose a información de colegas, lo que generó conflictos internos y quejas formales ante la dirección.

La empresa decide entonces construir un sistema de base de datos robusto sobre Oracle que resuelva estos tres frentes de forma simultánea e integrada.

## Objetivo del sistema

Diseñar e implementar en Oracle una base de datos transaccional para TaxiRápido que:

* **Garantice que cada usuario solo acceda a la información que le corresponde** según su rol dentro de la empresa.
* **Genere identificadores únicos, automáticos y consecutivos** para viajes y facturas, sin intervención manual.
* **Ofrezca vistas especializadas** que presenten la información adecuada a cada perfil sin exponer las tablas base.
* **Organice los datos en una arquitectura de almacenamiento separada** que diferencie físicamente la información operativa de la financiera.

## Integración de los 4 conceptos

**Usuarios, Control de acceso y seguridad (Tomás):** El sistema define cuatro usuarios Oracle con privilegios diferenciados: TAXIRAPIDO_APP es el dueño del modelo de datos; OPERADOR recibe el rol ROL_OPERADOR que le permite gestionar conductores, vehículos y viajes, pero le bloquea completamente el acceso a la tabla FACTURAS; ADMINISTRADOR recibe ROL_ADMINISTRADOR con acceso total incluyendo finanzas; y CONDUCTOR tiene permisos mínimos, únicamente sobre las vistas que filtran su propia información de sesión. Esta separación hace que el problema de acceso irrestricto quede resuelto de raíz: no es una restricción de aplicación, sino una restricción a nivel de motor de base de datos.

**Secuencias Generación automática de IDs (Brayan)** Se crean cuatro secuencias con parámetros ajustados a la realidad operativa: SEQ_VIAJES arranca en 1000 con cache 50 para soportar el volumen diario de despachos; SEQ_FACTURAS arranca en 100000 con el atributo ORDER activado, garantizando la consecutividad legal que exige la DIAN. Los triggers BEFORE INSERT en cada tabla asignan el ID automáticamente, eliminando por completo la posibilidad de duplicados o intervención manual.

**Vistas Abstracción y consultas seguras (Cristian)** Se crean seis vistas que actúan como la "ventana controlada" de cada rol: VISTA_VIAJES_ACTIVOS muestra solo los viajes en curso para los operadores; VISTA_REPORTES_FINANCIEROS consolida facturas con conductor y viaje para los administradores; VISTA_VIAJES_CONDUCTOR usa SYS_CONTEXT('USERENV','SESSION_USER') para filtrar automáticamente los viajes del conductor conectado, sin que este pueda ver los de otro; y VISTA_RENDIMIENTO_CONDUCTORES genera el ranking de ingresos para gerencia. Las tablas base nunca se exponen directamente a los usuarios de negocio.

**Arquitectura Oracle Organización física del almacenamiento (Carlos)** Se crean tres tablespaces: TS_OPERACIONES aloja las tablas de conductores, vehículos y viajes; TS_ADMIN aloja exclusivamente la tabla de facturas, separando físicamente los datos financieros; TS_TEMP_TAXI es el espacio temporal compartido. Esta separación no es solo organizativa: permite aplicar políticas de backup diferenciadas, controlar cuotas de disco por usuario y cumplir requerimientos de auditoría que exigen aislar la información contable.

## Justificación técnica

Oracle es la plataforma adecuada para este caso por razones concretas que el código mismo demuestra. El manejo de **roles y privilegios a nivel de motor** garantiza que las restricciones de acceso no dependan de la aplicación cliente, sino del propio gestor. Las **secuencias con el atributo ORDER** son un mecanismo nativo que ninguna alternativa open source ofrece con la misma garantía de consecutividad en entornos concurrentes. Las **vistas con contexto de sesión (SYS_CONTEXT)** permiten implementar seguridad a nivel de fila sin necesidad de Virtual Private Database adicional. Y la gestión de **tablespaces con autoextensión y cuotas por usuario** da a la empresa control granular sobre el crecimiento del almacenamiento, algo crítico cuando se proyecta escalar de 3 conductores a cientos.

El resultado es un sistema donde un operador puede despachar viajes sin ver un solo peso de facturación, un conductor consulta únicamente sus propios registros, y el administrador tiene visibilidad total — todo esto garantizado por Oracle, no por buenas intenciones de código.
