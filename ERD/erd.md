
erDiagram
    DOCENTES {
        uuid id PK
        varchar nombre
        varchar apellido
        varchar email UK
        varchar password_hash
        timestamptz created_at
        timestamptz updated_at
    }

    REFRESH_TOKENS {
        uuid id PK
        uuid docente_id FK
        varchar token_hash UK
        timestamptz expires_at
        timestamptz revoked_at
        timestamptz created_at
    }

    ESCUELAS {
        uuid id PK
        uuid docente_id FK
        varchar nombre
        varchar direccion
        varchar localidad
        timestamptz created_at
    }

    CURSOS {
        uuid id PK
        uuid escuela_id FK
        smallint anio
        varchar division
        varchar turno
    }

    MATERIAS {
        uuid id PK
        uuid docente_id FK
        varchar nombre
        text descripcion
        timestamptz created_at
    }

    ASIGNACIONES {
        uuid id PK
        uuid docente_id FK
        uuid curso_id FK
        uuid materia_id FK
        smallint ciclo_lectivo
        timestamptz created_at
    }

    ALUMNOS {
        uuid id PK
        uuid docente_id FK
        varchar nombre
        varchar apellido
        varchar documento
        date fecha_nacimiento
        varchar email
        timestamptz created_at
    }

    MATRICULAS {
        uuid id PK
        uuid docente_id FK
        uuid alumno_id FK
        uuid asignacion_id FK
        date fecha_alta
        date fecha_baja
        varchar estado
        text motivo_baja
        timestamptz created_at
    }

    EXAMENES {
        uuid id PK
        uuid asignacion_id FK
        uuid examen_original_id FK
        varchar titulo
        text descripcion
        date fecha
        smallint cuatrimestre
        numeric nota_maxima
        varchar tipo
        varchar estado
        text motivo_anulacion
        timestamptz created_at
    }

    RESULTADOS_EXAMEN {
        uuid id PK
        uuid examen_id FK
        uuid matricula_id FK
        uuid asignacion_id FK
        varchar estado
        numeric nota
        text observacion
        timestamptz updated_at
    }

    CIERRES_CUATRIMESTRALES {
        uuid id PK
        uuid matricula_id FK
        smallint cuatrimestre
        numeric promedio
        varchar estado_automatico
        varchar estado_manual
        text motivo_estado_manual
        varchar estado_cierre
        timestamptz cerrado_at
        timestamptz created_at
        timestamptz updated_at
    }

    REGISTROS_RITE {
        uuid id PK
        uuid matricula_id FK
        numeric promedio_cuatrimestre_1
        numeric promedio_cuatrimestre_2
        numeric promedio_anual
        varchar estado_cierre
        timestamptz cerrado_at
        timestamptz created_at
        timestamptz updated_at
    }

    AUDITORIA {
        uuid id PK
        uuid docente_id FK
        varchar entidad
        uuid entidad_id
        varchar accion
        jsonb datos_anteriores
        jsonb datos_nuevos
        timestamptz created_at
    }

    DOCENTES ||--o{ REFRESH_TOKENS : posee
    DOCENTES ||--o{ ESCUELAS : crea
    DOCENTES ||--o{ MATERIAS : crea
    DOCENTES ||--o{ ASIGNACIONES : administra
    DOCENTES ||--o{ ALUMNOS : registra
    DOCENTES ||--o{ MATRICULAS : administra
    DOCENTES ||--o{ AUDITORIA : genera

    ESCUELAS ||--o{ CURSOS : contiene
    CURSOS ||--o{ ASIGNACIONES : tiene
    MATERIAS ||--o{ ASIGNACIONES : corresponde

    ALUMNOS ||--o{ MATRICULAS : tiene
    ASIGNACIONES ||--o{ MATRICULAS : registra

    ASIGNACIONES ||--o{ EXAMENES : contiene
    EXAMENES o|--o{ EXAMENES : recuperatorio_de

    EXAMENES ||--o{ RESULTADOS_EXAMEN : genera
    MATRICULAS ||--o{ RESULTADOS_EXAMEN : obtiene

    MATRICULAS ||--o{ CIERRES_CUATRIMESTRALES : tiene
    MATRICULAS ||--o| REGISTROS_RITE : tiene
