# Sistema web de gestión de notas

## Especificación funcional y técnica --- versión 1.0

**Estado:** especificación inicial para comenzar el análisis y el
desarrollo.\
**Tipo de producto:** aplicación web multiusuario (SaaS) para docentes.\
**Idioma y contexto académico inicial:** español; escuelas y ciclos
lectivos de Argentina.\
**Decisión de producto:** cualquier docente podrá crear una cuenta y
administrar sus propios cursos, alumnos y calificaciones. No se
presupone que la escuela tenga una cuenta ni que exista una
administración institucional central.

------------------------------------------------------------------------

# 1. Objetivo y alcance

Desarrollar una aplicación web que permita a cada docente:

-   Administrar sus escuelas, materias y asignaciones.
-   Registrar alumnos y matricularlos en una o varias asignaciones.
-   Crear exámenes y cargar notas y recuperatorios.
-   Obtener automáticamente las condiciones TEA, TEP y TED por
    cuatrimestre y la condición RITE.
-   Consultar gráficos de resultados y exportar información.
-   Consultar el historial de modificaciones relevantes.

La aplicación será multiusuario. Cada docente tendrá un espacio de
trabajo independiente. Un docente no podrá consultar ni modificar
información de otro docente.

## 1.1. Fuera del alcance de la primera versión

Salvo que se decida expresamente lo contrario, la primera versión no
incluirá:

-   Cuentas para alumnos, familias, directivos ni preceptores.
-   Acceso institucional compartido entre varios docentes.
-   Sincronización con sistemas oficiales de gestión escolar.
-   Aplicación móvil nativa.
-   Notificaciones por WhatsApp o SMS.
-   Firma digital de boletines o actas oficiales.
-   Cálculo de asistencia general: el estado `ausente` se utiliza
    únicamente para una instancia de evaluación.

La arquitectura no debe impedir incorporar estas funciones en el futuro.

# 2. Conceptos y definiciones

-   **Docente:** usuario autenticado que administra sus propios datos.
-   **Escuela:** institución asociada al espacio de trabajo de un
    docente. Es un dato de texto administrado por ese docente, no un
    catálogo global.
-   **Materia:** materia asociada al espacio de trabajo de un docente.
    También es un dato de texto administrado por ese docente.
-   **Asignación:** combinación de docente, escuela, curso, materia y
    ciclo lectivo.
-   **Alumno:** registro de una persona dentro del espacio de trabajo de
    un docente. No es un usuario del sistema.
-   **Matrícula:** inscripción de un alumno en una asignación. Un mismo
    alumno puede tener varias matrículas.
-   **Cuatrimestre:** uno de los dos períodos de una asignación.
-   **Examen:** evaluación perteneciente a un cuatrimestre.
-   **Resultado:** situación de un alumno frente a un examen y, cuando
    corresponda, su recuperatorio.
-   **Baja de matrícula:** deja de considerar al alumno en una
    asignación, pero conserva sus antecedentes.
-   **Anonimización:** eliminación irreversible de los datos
    identificatorios de un alumno en el espacio de trabajo del docente.

# 3. Requisitos funcionales

## RF-01. Cuentas de docentes

1.  Cualquier docente puede registrarse con nombre, correo electrónico y
    contraseña.
2.  El correo debe verificarse antes de habilitar el uso normal de la
    cuenta.
3.  El docente puede iniciar y cerrar sesión.
4.  Debe existir recuperación segura de contraseña.
5.  Cada docente administra exclusivamente sus escuelas, materias,
    asignaciones, alumnos, matrículas, exámenes y resultados.
6.  El sistema no tendrá un registro público de alumnos ni permitirá que
    un docente busque alumnos de otros docentes.
7.  El docente podrá solicitar la eliminación de su cuenta. Antes de
    ejecutarla, el sistema debe explicar qué datos se eliminarán, cuáles
    se conservarán por obligaciones legales o de auditoría y qué
    sucederá con sus asignaciones.

## RF-02. Escuelas, materias y asignaciones

1.  El docente puede crear y reutilizar escuelas y materias que ya haya
    utilizado.
2.  Escuela y materia se almacenan como registros pertenecientes al
    docente. No existe un catálogo global obligatorio.
3.  Una asignación se identifica por:
    -   docente;
    -   escuela;
    -   curso (texto libre, por ejemplo `3.º A`);
    -   materia;
    -   ciclo lectivo.
4.  No se permiten dos asignaciones del mismo docente con la misma
    combinación de esos cinco campos.
5.  Cada asignación tiene dos cuatrimestres.
6.  Las reglas académicas se configuran por asignación y deben conservar
    una versión o copia histórica para que un cambio posterior no altere
    silenciosamente períodos ya evaluados.
7.  El dashboard muestra una tarjeta por asignación del ciclo lectivo
    seleccionado, con escuela, curso, materia y cantidad de matrículas
    activas.
8.  La cantidad de alumnos se calcula contando las matrículas activas de
    la asignación. No se mantiene como contador independiente
    persistido.
9.  El docente puede consultar asignaciones de ciclos anteriores.

## RF-03. Alumnos y matrículas

### Registro de alumnos

1.  El docente puede crear alumnos manualmente o importarlos mediante
    CSV.
2.  Los datos identificatorios mínimos son apellido y nombre. No se
    solicita DNI.
3.  Existe un único registro de alumno por persona dentro del espacio de
    trabajo de un docente.
4.  La coincidencia de apellido y nombre no demuestra identidad: si
    coincide con un registro existente, el sistema debe pedir
    confirmación antes de reutilizarlo.
5.  Debe permitirse registrar homónimos como personas distintas.
6.  Los listados se ordenan por apellido y nombre con reglas de
    ordenamiento en español, contemplando tildes y `ñ`.
7.  Modificar el nombre de una persona actualiza su representación en
    todas las asignaciones del docente. Antes de confirmar, se informa
    en cuántas asignaciones aparece.

### Matrículas

1.  Una matrícula vincula un alumno con una asignación.
2.  Un alumno puede tener matrículas en distintas asignaciones del mismo
    docente.
3.  No puede haber dos matrículas activas del mismo alumno en una misma
    asignación.
4.  La matrícula contiene `fecha_alta` y `fecha_baja` (esta última
    nullable).
5.  Dar de baja una matrícula:
    -   establece `fecha_baja`;
    -   conserva resultados e historial;
    -   excluye al alumno de listados activos, contadores, cálculos y
        gráficos actuales de esa asignación;
    -   no afecta sus matrículas en otras asignaciones.
6.  Debe poder reactivarse una matrícula dada de baja. La reactivación
    se registra en el historial y requiere definir una nueva
    `fecha_alta` efectiva.
7.  Al crear una asignación de otra materia para un curso que el docente
    ya administra, el sistema ofrece copiar la matrícula de una
    asignación de origen.
8.  La pantalla de copia muestra todos los alumnos de origen
    seleccionados inicialmente y permite desmarcar alumnos.
9.  Copiar matrícula crea nuevas matrículas para los alumnos
    seleccionados. No duplica personas ni copia notas, exámenes ni
    resultados.
10. Al incorporar un alumno a una asignación que ya tiene exámenes, el
    docente debe indicar si:
    -   ingresó después de las evaluaciones ya realizadas; o
    -   pertenecía al curso desde el inicio y se lo está registrando
        tarde.
11. En el primer caso, los exámenes anteriores a su ingreso quedan
    `ausente`.
12. En el segundo caso, la `fecha_alta` efectiva se fija al inicio del
    cuatrimestre y los exámenes anteriores quedan `pendiente`.
13. La operación debe mostrar una vista previa de los resultados que se
    crearán antes de confirmar.

### Anonimización

1.  El docente puede solicitar la anonimización irreversible de una
    persona de su espacio de trabajo.
2.  La acción requiere confirmación explícita y una explicación de sus
    consecuencias.
3.  Se eliminan apellido, nombre y cualquier otro dato identificatorio
    almacenado por la aplicación.
4.  Se conserva un identificador interno no reversible a la identidad
    original y los resultados académicos históricos que deban
    conservarse.
5.  Una persona anonimizada no aparece en listados, búsquedas,
    exportaciones nominales, contadores, cálculos ni gráficos.
6.  Sus matrículas se consideran inactivas a efectos operativos.
7.  El historial no debe almacenar nombres: debe referenciar
    identificadores internos.
8.  La anonimización no puede deshacerse desde la aplicación.
9.  Debe existir un registro de anonimización independiente de los
    backups ordinarios, para reaplicar las operaciones al restaurar una
    copia anterior.
10. La política de retención de backups debe tener un plazo definido. La
    organización debe documentar cuándo puede considerarse efectiva la
    eliminación de datos en todas las copias.

## RF-04. Exámenes y resultados

1.  Cada examen pertenece a un único cuatrimestre.
2.  Un examen tiene, como mínimo, nombre o descripción y fecha.
3.  Al crear un examen, el sistema genera un resultado inicial
    `pendiente` para cada matrícula activa que corresponda.
4.  Cada resultado tiene una instancia de examen y una instancia de
    recuperatorio opcional.
5.  Cada instancia puede tener uno de estos estados:
    -   `pendiente`: todavía no hay resultado definitivo;
    -   `calificada`: existe una nota entera de 1 a 10;
    -   `ausente`: el alumno no se presentó; no se almacena una nota
        numérica.
6.  Para el recuperatorio se admite además `no_habilitado`, que
    significa que no corresponde o no fue asignado. Este estado no
    bloquea el cierre.
7.  El recuperatorio solo puede calificarse si el docente lo habilitó.
    La aplicación debe impedir combinaciones inconsistentes.
8.  Una nota calificada debe ser un entero entre 1 y 10.
9.  Un campo vacío no se interpreta automáticamente como 1: mientras no
    se confirme una calificación, el estado es `pendiente`. Para cargar
    un 1, el docente debe ingresar explícitamente `1`.
10. `ausente` aporta cero al cálculo académico, pero no se guarda como
    nota numérica.
11. El docente puede modificar resultados mientras el cuatrimestre esté
    abierto.
12. Toda modificación de un resultado debe generar un evento de
    historial con usuario, fecha, valor anterior, valor nuevo y, cuando
    corresponda, motivo.
13. Si se modifica la fecha de un examen, el sistema no debe cambiar
    estados existentes silenciosamente. Debe mostrar el impacto y pedir
    confirmación para cualquier actualización masiva.
14. La eliminación de un examen no será física si tiene resultados
    asociados. En la primera versión se recomienda permitir
    archivarlo/anularlo, conservando resultados e historial y
    excluyéndolo de los cálculos solo mediante una operación explícita y
    auditada.

## RF-05. Cierre y reapertura de cuatrimestre

1.  Un cuatrimestre puede estar `abierto` o `cerrado`.
2.  El cierre se bloquea si existe algún examen pendiente para una
    matrícula activa que corresponda al período.
3.  También se bloquea si existe un recuperatorio habilitado que
    continúa pendiente.
4.  Un recuperatorio `no_habilitado`, `calificado` o `ausente` no
    bloquea por sí mismo el cierre.
5.  Un cuatrimestre sin exámenes puede cerrarse, siempre que el sistema
    muestre que no hay evaluaciones cargadas y el docente confirme.
6.  Al cerrar, se registra quién y cuándo realizó la operación.
7.  Un cuatrimestre cerrado no permite modificar exámenes, matrículas ni
    resultados que afecten su información académica.
8.  Para modificar información se debe reabrir el cuatrimestre.
9.  La reapertura requiere motivo y registra quién, cuándo y por qué.
10. Luego de las modificaciones, el docente debe volver a cerrarlo
    explícitamente.
11. El cierre y la reapertura son operaciones transaccionales: no pueden
    quedar parcialmente aplicadas.

## RF-06. Cálculos académicos

Los cálculos se realizan en el backend al consultar la información. No
se persisten como valores definitivos.

### Nota efectiva por examen

Para cada alumno y examen:

1.  Si el recuperatorio está `calificada`, la nota efectiva es la nota
    del recuperatorio.
2.  Si no, si el examen está `calificada`, la nota efectiva es la nota
    del examen.
3.  Si el examen está `ausente`, la nota efectiva es cero.
4.  Si el examen está `pendiente`, no existe nota efectiva definitiva.
5.  Un recuperatorio `ausente` no reemplaza la nota del examen.
6.  Un recuperatorio `no_habilitado` o `pendiente` tampoco reemplaza la
    nota del examen.
7.  Para evitar ambigüedad, en la primera versión un recuperatorio solo
    puede habilitarse para un examen originalmente calificado. La
    posibilidad de recuperar una ausencia queda fuera de alcance hasta
    que se defina la regla institucional.

### Condición del cuatrimestre

Definiciones:

-   `C`: cantidad de resultados con nota efectiva calificada. Incluye
    examen calificado y recuperatorio calificado; no incluye ausentes.
-   `n`: cantidad de resultados con examen calificado o ausente. No
    incluye pendientes.
-   `S`: suma de notas efectivas; los ausentes aportan cero.

Reglas:

``` text
Si C == 0:
    TED

Si 2 * S > 13 * n:
    TEA

En cualquier otro caso:
    TEP
```

El umbral es estricto: un promedio exactamente igual a 6,50 corresponde
a TEP. Un promedio superior a 6,50 corresponde a TEA.

Un cuatrimestre con todos los alumnos pendientes no tiene resultados
evaluables y se clasifica como TED según esta regla. La interfaz debe
distinguir visualmente esta situación de un cuatrimestre con
evaluaciones efectivamente realizadas.

### Condición RITE

Solo se consideran cuatrimestres con `C > 0`.

-   Si ambos cuatrimestres tienen `C > 0`, se aplica:

``` text
S1 * n2 + S2 * n1 > 13 * n1 * n2
```

-   Si solo uno tiene `C > 0`, se aplica a ese cuatrimestre:

``` text
2 * S > 13 * n
```

-   Si ninguno tiene `C > 0`, la condición es TED.

**Interpretación adoptada:** cuando ambos cuatrimestres tienen
resultados calificables, cada promedio cuatrimestral tiene el mismo
peso, independientemente de la cantidad de exámenes de cada período.
Esta regla debe ser confirmada por el responsable académico antes de
producción.

### Precisión numérica

1.  Las decisiones TEA/TEP/TED se calculan con enteros, sin redondeos
    intermedios.
2.  Para mostrar promedios se utiliza redondeo decimal half-up a dos
    decimales.
3.  El redondeo visual nunca modifica la condición calculada.

## RF-07. Gráficos

1.  Se muestra un gráfico de barras con el promedio de nota efectiva por
    examen.
2.  Solo se consideran resultados con nota efectiva calificada.
3.  Las ausencias y los resultados pendientes se excluyen del promedio.
4.  Se omite la barra cuando no hay notas calificadas para ese examen.
5.  Cada barra muestra la cantidad de notas consideradas (`n` del
    gráfico), no la cantidad total de matriculados.
6.  El promedio se calcula con precisión completa y se redondea
    únicamente para mostrar.
7.  Los filtros deben permitir seleccionar asignación y cuatrimestre.
8.  Los gráficos deben respetar las mismas reglas de autorización y
    anonimización que el resto del sistema.

## RF-08. Importación y exportación

### Importación CSV

1.  Debe existir una plantilla descargable con columnas y formato
    documentados.
2.  Se validan tamaño, codificación, encabezados y tipos de datos.
3.  Se muestra una vista previa antes de importar.
4.  Se informan filas válidas, inválidas y posibles duplicados.
5.  La importación no debe crear duplicados silenciosamente.
6.  El docente puede cancelar la operación o confirmar las filas válidas
    según una política explícita.
7.  La operación debe ser transaccional o informar claramente qué filas
    se procesaron y cuáles no.

### Exportación

1.  Se permite exportar información de una asignación y cuatrimestre a
    Excel y PDF.
2.  La exportación respeta filtros, bajas y anonimización.
3.  Debe indicarse fecha de generación y período exportado.
4.  Las celdas de Excel se neutralizan para evitar inyección de fórmulas
    cuando el contenido comienza con `=`, `+`, `-` o `@`.
5.  Los archivos generados no deben quedar publicados mediante URLs
    predecibles ni ser accesibles por otros docentes.
6.  Los archivos temporales se eliminan según una política de retención
    definida.

## RF-09. Dashboard

1.  Muestra una tarjeta por asignación del ciclo lectivo seleccionado.
2.  Cada tarjeta muestra escuela, curso, materia y cantidad de
    matrículas activas.
3.  La cantidad se obtiene contando matrículas activas y alumnos no
    anonimizados.
4.  El docente puede cambiar el ciclo lectivo consultado.
5.  Desde cada tarjeta se accede al detalle de la asignación.
6.  Un docente con asignaciones en varias escuelas ve todas sus
    asignaciones, sin mezclar alumnos ni resultados.

# 4. Reglas de negocio y decisiones explícitas

  -----------------------------------------------------------------------
  Tema                                Regla
  ----------------------------------- -----------------------------------
  Nota válida                         Entero entre 1 y 10

  Campo de nota vacío                 Permanece pendiente; no equivale a
                                      1

  Ausencia                            Aporta cero al cálculo, sin guardar
                                      nota numérica

  Recuperatorio calificado            Reemplaza la nota original, aunque
                                      sea menor

  Recuperatorio ausente               No reemplaza la nota original

  Recuperatorio no habilitado         No reemplaza la nota original y no
                                      bloquea el cierre

  Pendiente                           Bloquea el cierre si corresponde a
                                      una evaluación o recuperatorio
                                      habilitado

  Ingresante posterior a un examen    Ausente en ese examen

  Carga tardía de alumno que estaba   Pendiente en los exámenes
  desde el inicio                     anteriores

  Baja de matrícula                   Conserva historial, pero excluye de
                                      resultados actuales

  Alumno anonimizado                  Se excluye de listados, cálculos,
                                      gráficos y exportaciones nominales

  Umbral TEA                          Estrictamente mayor que 6,50

  Cálculo                             Enteros; redondeo half-up solo para
                                      presentación

  RITE con dos cuatrimestres          Promedio de ambos promedios con
                                      igual peso
  -----------------------------------------------------------------------

# 5. Modelo de datos propuesto

El siguiente modelo es conceptual. Los tipos, índices, claves foráneas y
restricciones exactas se definirán al elegir el motor de base de datos.

``` mermaid
erDiagram
    DOCENTE ||--o{ ESCUELA : administra
    DOCENTE ||--o{ MATERIA : administra
    DOCENTE ||--o{ ALUMNO : administra
    DOCENTE ||--o{ ASIGNACION : tiene

    ESCUELA ||--o{ ASIGNACION : identifica
    MATERIA ||--o{ ASIGNACION : identifica

    ASIGNACION ||--|{ CUATRIMESTRE : contiene
    ASIGNACION ||--o{ MATRICULA : incluye
    ALUMNO ||--o{ MATRICULA : se_inscribe

    CUATRIMESTRE ||--o{ EXAMEN : contiene
    EXAMEN ||--o{ RESULTADO_EXAMEN : evalua
    MATRICULA ||--o{ RESULTADO_EXAMEN : recibe

    DOCENTE ||--o{ SESION : inicia
    DOCENTE ||--o{ EVENTO_AUDITORIA : genera
    DOCENTE ||--o{ TOKEN_RECUPERACION : solicita

    RESULTADO_EXAMEN ||--o{ EVENTO_AUDITORIA : registra

    DOCENTE {
        uuid id PK
        string email UK
        string password_hash
        datetime email_verificado_en
        datetime creado_en
        datetime eliminado_en
    }

    ESCUELA {
        uuid id PK
        uuid docente_id FK
        string nombre
    }

    MATERIA {
        uuid id PK
        uuid docente_id FK
        string nombre
    }

    ALUMNO {
        uuid id PK
        uuid docente_id FK
        string apellido
        string nombre
        datetime anonimizado_en
    }

    ASIGNACION {
        uuid id PK
        uuid docente_id FK
        uuid escuela_id FK
        uuid materia_id FK
        string curso
        int ciclo_lectivo
        json reglas
    }

    CUATRIMESTRE {
        uuid id PK
        uuid asignacion_id FK
        int numero
        string estado
        datetime cerrado_en
    }

    MATRICULA {
        uuid id PK
        uuid asignacion_id FK
        uuid alumno_id FK
        date fecha_alta
        date fecha_baja
    }

    EXAMEN {
        uuid id PK
        uuid cuatrimestre_id FK
        string nombre
        date fecha
        boolean anulado
    }

    RESULTADO_EXAMEN {
        uuid id PK
        uuid examen_id FK
        uuid matricula_id FK
        string estado_examen
        int nota_examen
        string estado_recuperatorio
        int nota_recuperatorio
    }

    SESION {
        uuid id PK
        uuid docente_id FK
        string token_hash
        datetime expira_en
        datetime revocada_en
    }

    TOKEN_RECUPERACION {
        uuid id PK
        uuid docente_id FK
        string token_hash
        datetime expira_en
        datetime usado_en
    }

    EVENTO_AUDITORIA {
        uuid id PK
        uuid docente_id FK
        uuid entidad_id
        string entidad_tipo
        string accion
        json valor_anterior
        json valor_nuevo
        string motivo
        datetime creado_en
    }
```

## 5.1. Restricciones de integridad mínimas

-   `UNIQUE (docente_id, escuela_id, curso, materia_id, ciclo_lectivo)`
    en asignaciones.
-   `UNIQUE (examen_id, matricula_id)` en resultados.
-   Una sola matrícula activa por `(asignacion_id, alumno_id)`.
-   `UNIQUE (asignacion_id, numero)` en cuatrimestres.
-   `nota_examen` solo puede tener valor si
    `estado_examen = calificada`.
-   `nota_recuperatorio` solo puede tener valor si
    `estado_recuperatorio = calificada`.
-   Las notas deben estar entre 1 y 10.
-   La matrícula, asignación, examen y alumno relacionados deben
    pertenecer al mismo docente.
-   No se deben permitir referencias entre entidades de distintos
    espacios de trabajo.
-   Los registros históricos no se eliminan físicamente desde la
    aplicación.

**Importante:** las restricciones de unicidad y pertenencia deben
protegerse en la base de datos y no únicamente con validaciones del
backend. Para las relaciones entre entidades del mismo docente se
evaluarán claves foráneas compuestas o mecanismos equivalentes.

# 6. Seguridad, privacidad y cumplimiento

## 6.1. Objetivo de seguridad

Utilizar OWASP ASVS 5.0.0, nivel 2, como referencia de verificación. No
se debe afirmar que el sistema cumple ASVS por incluir una lista de
medidas: el cumplimiento requiere implementar, probar y documentar los
controles aplicables.

## 6.2. Autenticación

1.  Contraseñas almacenadas con Argon2id.
2.  Parámetros iniciales mínimos: `m=19 MiB`, `t=2`, `p=1`, sujetos a
    revisión según el hardware disponible y las recomendaciones
    vigentes.
3.  Salt único por contraseña, administrado por la biblioteca
    criptográfica.
4.  Longitud mínima de 12 caracteres y comprobación contra contraseñas
    conocidas como comprometidas.
5.  Segundo factor TOTP obligatorio antes de habilitar el acceso
    completo a la aplicación.
6.  Códigos de recuperación de un solo uso, almacenados de forma segura.
7.  Recuperación de contraseña mediante token aleatorio, de un solo uso,
    con vencimiento y almacenamiento hasheado.
8.  El restablecimiento de contraseña invalida sesiones existentes.
9.  El restablecimiento o reemplazo del segundo factor requiere un
    procedimiento reforzado y queda auditado.
10. Límites de intentos por IP y por cuenta.
11. Respuestas equivalentes ante usuario inexistente y contraseña
    incorrecta.
12. Aviso por correo ante inicio de sesión desde un dispositivo o
    contexto nuevo, sin incluir secretos.

## 6.3. Sesiones y autorización

1.  Sesiones del lado servidor, identificador aleatorio y cookie
    `HttpOnly`, `Secure` y `SameSite` apropiado.
2.  Expiración por inactividad y expiración absoluta.
3.  Regeneración del identificador al autenticar y ante cambios de
    privilegios.
4.  Revocación al cerrar sesión, cambiar contraseña o desactivar la
    cuenta.
5.  Posibilidad de cerrar todas las sesiones.
6.  Denegación por defecto.
7.  El identificador del docente se obtiene de la sesión autenticada;
    nunca se confía en un `docente_id` enviado por el cliente.
8.  Toda consulta y operación se limita al espacio de trabajo del
    docente autenticado.
9.  Pruebas automatizadas de acceso cruzado para lecturas, escrituras,
    historial, exportaciones y archivos.

## 6.4. Aplicación web y API

1.  Validación de entradas en servidor.
2.  Consultas parametrizadas y ORM utilizado de forma segura.
3.  Escape de salida según el contexto.
4.  Protección CSRF en operaciones que modifican datos cuando se
    utilizan cookies de sesión.
5.  CSP restrictiva, HSTS, `X-Content-Type-Options`, `frame-ancestors` y
    `Referrer-Policy`.
6.  CORS deshabilitado si no es necesario o limitado a orígenes
    explícitos.
7.  Límites de tamaño, frecuencia y complejidad de las peticiones.
8.  Errores sin trazas internas ni secretos en producción.

## 6.5. Base de datos e infraestructura

1.  HTTPS obligatorio.
2.  Base de datos no expuesta públicamente.
3.  Usuario de base de datos con privilegios mínimos.
4.  Secretos fuera del repositorio y separados por entorno.
5.  Backups cifrados, con retención definida y pruebas periódicas de
    restauración.
6.  Registro de anonimización conservado fuera del conjunto de backups
    que podría restaurarse.
7.  Al restaurar, se reaplican las anonimaciones posteriores a la fecha
    del backup antes de habilitar el sistema.
8.  Logs sin contraseñas, tokens, notas innecesarias ni datos
    identificatorios de alumnos.
9.  Dependencias auditadas y actualizadas periódicamente.
10. Monitoreo de errores, disponibilidad y eventos de seguridad.

## 6.6. Auditoría

1.  Los cambios de notas, recuperatorios, matrículas, cierres,
    reaperturas, anonimización y operaciones sensibles generan eventos.
2.  Cada evento identifica actor, fecha, entidad, acción y valores
    anteriores y nuevos cuando corresponda.
3.  Los eventos no tienen endpoints de edición ni eliminación.
4.  El acceso de la aplicación a la tabla de auditoría debe permitir
    insertar y consultar, pero no modificar ni borrar eventos.
5.  El historial referencia identificadores internos, nunca nombres de
    alumnos.
6.  Se registra el motivo de reaperturas y de operaciones sensibles.

## 6.7. Privacidad y normativa

1.  Recoger únicamente los datos necesarios.
2.  Publicar una política de privacidad clara.
3.  Definir finalidad, base jurídica, plazos de conservación, canales de
    ejercicio de derechos y procedimiento de incidentes.
4.  Determinar contractualmente quién es responsable del tratamiento y
    quién actúa como encargado. No se presume que el proveedor sea
    automáticamente responsable de todos los datos.
5.  Evaluar el tratamiento de datos de menores y la intervención de las
    instituciones o representantes que corresponda.
6.  Verificar las obligaciones de inscripción y demás exigencias
    aplicables bajo la Ley 25.326 y normativa complementaria con
    asesoramiento profesional.
7.  Antes de ofrecer el servicio públicamente, realizar una revisión
    legal y de seguridad.

# 7. Requisitos no funcionales

1.  **Aislamiento:** ningún docente puede acceder a datos de otro,
    incluso manipulando IDs.
2.  **Integridad:** las operaciones que modifican varias entidades deben
    ejecutarse en transacciones.
3.  **Consistencia:** los cálculos deben implementarse una sola vez en
    el backend y reutilizarse en reportes y dashboard.
4.  **Trazabilidad:** las modificaciones relevantes deben quedar
    registradas.
5.  **Usabilidad:** carga de notas rápida, con navegación por teclado y
    validaciones visibles.
6.  **Accesibilidad:** contraste adecuado, etiquetas asociadas a
    controles y uso sin depender exclusivamente del color.
7.  **Rendimiento:** definir objetivos medibles antes de producción;
    como punto de partida, las consultas habituales deberían responder
    en menos de 500 ms en condiciones normales, excluyendo
    exportaciones.
8.  **Compatibilidad:** navegadores modernos de escritorio y
    dispositivos móviles.
9.  **Mantenibilidad:** módulos separados por dominio, lógica de negocio
    independiente de la interfaz y pruebas automatizadas.
10. **Observabilidad:** logs estructurados, métricas y alertas sin
    exponer datos personales.

# 8. Criterios de aceptación y pruebas

## 8.1. Cálculos

-   [ ] 6,50 produce TEP.
-   [ ] Un promedio superior a 6,50 produce TEA.
-   [ ] Un promedio inferior a 6,50 produce TEP.
-   [ ] Recuperatorio menor que el examen reemplaza la nota original.
-   [ ] Recuperatorio calificado reemplaza la nota original.
-   [ ] Recuperatorio ausente no reemplaza la nota original.
-   [ ] Examen ausente aporta cero.
-   [ ] Todos los resultados ausentes producen TED.
-   [ ] Todos los resultados pendientes producen TED y se distinguen
    visualmente de un período evaluado.
-   [ ] RITE con ambos cuatrimestres aplica la fórmula definida.
-   [ ] RITE con un solo cuatrimestre evaluable utiliza ese
    cuatrimestre.
-   [ ] RITE sin cuatrimestres evaluables produce TED.
-   [ ] Los resultados no cambian por redondeos de presentación.

## 8.2. Matrículas y alumnos

-   [ ] Un alumno puede estar matriculado en dos asignaciones del mismo
    docente sin duplicar su registro personal.
-   [ ] Dos asignaciones del mismo curso pueden tener matrículas
    diferentes.
-   [ ] Copiar una matrícula no copia resultados ni notas.
-   [ ] La copia permite desmarcar alumnos.
-   [ ] Un homónimo requiere confirmación antes de reutilizar un
    registro.
-   [ ] Modificar el nombre actualiza todas las asignaciones del
    docente.
-   [ ] Dar de baja una matrícula no afecta otras asignaciones.
-   [ ] La baja conserva resultados e historial.
-   [ ] La reactivación queda registrada.
-   [ ] Un ingresante posterior a un examen queda ausente en ese examen.
-   [ ] Un alumno cargado tarde que estaba desde el inicio queda
    pendiente en exámenes anteriores.
-   [ ] El dashboard excluye matrículas dadas de baja y personas
    anonimizadas.
-   [ ] Dos asignaciones del mismo curso con 10 y 11 matrículas activas
    muestran 10 y 11.

## 8.3. Exámenes y cierres

-   [ ] Crear un examen genera resultados pendientes para las matrículas
    que corresponden.
-   [ ] Un examen no admite dos resultados para la misma matrícula.
-   [ ] Un campo vacío no se guarda como nota 1.
-   [ ] Una nota fuera del rango 1--10 se rechaza.
-   [ ] Un recuperatorio no habilitado no bloquea el cierre.
-   [ ] Un recuperatorio habilitado y pendiente bloquea el cierre.
-   [ ] Un examen pendiente bloquea el cierre.
-   [ ] El cierre de un cuatrimestre sin exámenes exige confirmación.
-   [ ] Un cuatrimestre cerrado no permite modificar resultados.
-   [ ] La reapertura exige motivo y genera auditoría.
-   [ ] El nuevo cierre vuelve a validar los pendientes.
-   [ ] Cambiar la fecha de un examen no modifica estados
    silenciosamente.

## 8.4. Aislamiento y seguridad

-   [ ] Un docente no puede consultar asignaciones de otro.
-   [ ] Un docente no puede modificar notas de otro.
-   [ ] Un docente no puede consultar alumnos, matrículas ni historial
    de otro.
-   [ ] Un docente no puede descargar exportaciones ajenas.
-   [ ] Manipular IDs en URL, cuerpo o parámetros no permite acceder a
    recursos ajenos.
-   [ ] Las operaciones de escritura validan CSRF cuando corresponde.
-   [ ] Los tokens de recuperación vencen y no pueden reutilizarse.
-   [ ] Cambiar contraseña revoca sesiones anteriores.
-   [ ] El segundo factor no puede omitirse mediante recuperación
    ordinaria de contraseña.

## 8.5. Anonimización y backups

-   [ ] Anonimizar elimina los datos identificatorios.
-   [ ] La persona anonimizada no aparece en búsquedas, listados ni
    exportaciones nominales.
-   [ ] No participa en contadores, cálculos ni gráficos actuales.
-   [ ] El historial conserva identificadores internos y no nombres.
-   [ ] La aplicación no permite revertir la anonimización.
-   [ ] Restaurar un backup anterior reaplica las anonimaciones
    posteriores antes de habilitar el sistema.

## 8.6. Importación y exportación

-   [ ] Un CSV inválido produce errores por fila comprensibles.
-   [ ] Los duplicados se detectan y no se crean silenciosamente.
-   [ ] La exportación neutraliza fórmulas potenciales.
-   [ ] Los archivos exportados respetan bajas y anonimización.
-   [ ] Un docente no puede acceder a archivos generados para otro.

# 9. Stack tecnológico propuesto

El stack definitivo se confirmará antes de iniciar la implementación.

  Capa             Propuesta
  ---------------- --------------------------------------------------------
  Frontend         React + TypeScript
  Backend          NestJS
  Base de datos    PostgreSQL
  Acceso a datos   Prisma o TypeORM
  Autenticación    Sesiones de servidor + cookies seguras
  Gráficos         Recharts
  Excel            ExcelJS
  PDF              PDFKit o solución equivalente
  Pruebas          Jest + Supertest
  Entorno local    Docker Compose
  Producción       VPS o plataforma administrada, con backups y monitoreo

**Recomendación:** PostgreSQL por sus restricciones, transacciones,
índices parciales y soporte de collations. La elección entre Prisma y
TypeORM debe hacerse con una prueba pequeña de migraciones, relaciones
compuestas y consultas de reportes. No conviene elegir el ORM únicamente
por familiaridad.

# 10. Organización sugerida del backend

``` text
src/
├── auth/
├── docentes/
├── escuelas/
├── materias/
├── alumnos/
├── asignaciones/
├── matriculas/
├── cuatrimestres/
├── examenes/
├── resultados/
├── calculos-academicos/
├── reportes/
├── exportaciones/
├── auditoria/
├── sesiones/
└── common/
```

Los cálculos académicos deben ser funciones o servicios de dominio
independientes de los controladores HTTP y de React. Las mismas
funciones deben utilizarse en el detalle del alumno, dashboard, reportes
y exportaciones.

# 11. Plan de desarrollo: dónde arrancar

No comenzar por el dashboard, los gráficos ni la autenticación avanzada.
El primer objetivo es validar el modelo académico y el aislamiento de
datos.

## Fase 0 --- Cerrar decisiones

Antes de programar, confirmar:

-   [ ] La fórmula RITE con igual peso para ambos promedios
    cuatrimestrales.
-   [ ] La regla de recuperatorio: solo se habilita para un examen
    calificado.
-   [ ] La política de fecha de alta y de modificación posterior de
    fechas de exámenes.
-   [ ] La política de reactivación de matrículas.
-   [ ] Qué ocurre con resultados históricos al anular un examen.
-   [ ] La regla de cierre cuando no existen exámenes.
-   [ ] Los formatos de CSV, Excel y PDF.
-   [ ] La responsabilidad legal y contractual del servicio.

## Fase 1 --- Modelo de datos y dominio

1.  Crear el repositorio y configurar el entorno local.
2.  Crear el esquema PostgreSQL y las migraciones.
3.  Implementar docentes y aislamiento multiusuario desde el principio.
4.  Implementar escuelas, materias y asignaciones.
5.  Implementar alumnos y matrículas.
6.  Implementar cuatrimestres, exámenes y resultados.
7.  Agregar restricciones e índices en la base.
8.  Escribir pruebas unitarias de las reglas académicas antes de
    construir la interfaz.

**Resultado esperado:** una API capaz de crear una asignación,
matricular alumnos, crear exámenes y registrar resultados sin interfaz
gráfica.

## Fase 2 --- Núcleo académico

1.  Carga y modificación de notas.
2.  Recuperatorios.
3.  Cálculo de nota efectiva.
4.  Cálculo TEA/TEP/TED.
5.  Cálculo RITE.
6.  Cierre y reapertura.
7.  Historial de modificaciones.

**Resultado esperado:** el sistema calcula correctamente todas las
condiciones y puede demostrar cómo llegó a cada resultado.

## Fase 3 --- Interfaz de uso diario

1.  Registro, login y configuración inicial.
2.  Dashboard de asignaciones.
3.  Detalle de asignación y listado de alumnos.
4.  Alta, edición, baja y copia de matrículas.
5.  Pantalla de exámenes y carga de notas.
6.  Vista de condiciones por alumno y cuatrimestre.
7.  Cierre y reapertura.

**Resultado esperado:** un docente puede gestionar un ciclo lectivo
completo.

## Fase 4 --- Reportes y operaciones

1.  Gráficos.
2.  Importación CSV.
3.  Exportación Excel.
4.  Exportación PDF.
5.  Filtros por ciclo, asignación y cuatrimestre.

## Fase 5 --- Seguridad y salida a producción

1.  TOTP obligatorio y recuperación segura.
2.  Revisión de autorización y pruebas de acceso cruzado.
3.  Auditoría reforzada.
4.  Anonimización y registro externo de operaciones.
5.  Backups cifrados y restauración probada.
6.  Revisión ASVS 5.0 nivel 2.
7.  Revisión legal y política de privacidad.
8.  Despliegue, monitoreo y alertas.

Los controles básicos de seguridad, validación, autorización, CSRF,
gestión de secretos y aislamiento no se postergan: se implementan desde
las primeras fases. Esta etapa concentra los controles reforzados y la
verificación integral.

# 12. Primera tarea concreta

La primera entrega de desarrollo debería ser un **prototipo del dominio
académico**, sin dashboard ni diseño visual complejo.

Debe permitir, mediante pruebas automatizadas o una API mínima:

1.  Crear dos docentes.
2.  Crear una asignación para cada docente.
3.  Crear alumnos y matrículas.
4.  Crear un examen.
5.  Cargar notas, ausencias y recuperatorios.
6.  Calcular la nota efectiva y las condiciones.
7.  Verificar que un docente no pueda acceder a los datos del otro.

No avanzar a la interfaz completa hasta que estas pruebas pasen.

------------------------------------------------------------------------

# 13. Decisiones pendientes para una versión 1.1

Estas decisiones no deben quedar implícitas en el código:

1.  ¿El RITE institucional utiliza efectivamente el mismo peso para
    ambos promedios cuatrimestrales?
2.  ¿Se permite recuperar una ausencia o solamente una evaluación
    calificada?
3.  ¿Un examen anulado se excluye de todos los cálculos o conserva su
    efecto histórico?
4.  ¿La reactivación de una matrícula recupera los resultados anteriores
    o inicia una nueva trayectoria?
5.  ¿Qué política se aplica a los datos de una cuenta docente eliminada?
6.  ¿Qué período de retención tendrán los backups?
7.  ¿Qué volumen máximo de alumnos, exámenes y docentes se espera
    inicialmente?

**Criterio de finalización de la especificación:** las decisiones que
afectan cálculos, integridad, privacidad o ciclo de vida de los datos
deben estar confirmadas antes de publicar el sistema. Las decisiones
visuales pueden evolucionar durante el desarrollo.
