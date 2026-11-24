# Atenea Klass — Sistema de Gestión Académica para FP (V1)

> **Versión:** 0.1.0  
> **Fecha de inicio del proyecto:** 24 de noviembre de 2026

**Atenea Klass** es una aplicación web desarrollada en **Symfony** pensada inicialmente para uso personal como docente de Formación Profesional (FP) y diseñada para evolucionar de forma incremental hacia un sistema multiusuario más completo.

Este proyecto forma parte de mi portafolio profesional y documenta tanto la arquitectura técnica como el proceso iterativo de diseño y refinamiento de la aplicación.

---

## 🎯 Enfoque y filosofía del proyecto

* **V1 acotada y pragmática:** Empezar cubriendo el flujo real del profesor en el aula (grupos, módulos, alumnado, horarios, asistencia, evaluaciones y calificaciones) sin añadir todavía complejidad de RR. HH. (`employees`), partes disciplinarios ni justificantes con adjuntos.
* **Lógica explícita ("sin magia"):** Separar claramente el modelo de datos de las reglas de negocio. Las entidades de Doctrine representan el estado y las relaciones; los servicios de dominio concentran los cálculos, validaciones y estrategias.
* **Convención de idioma:**
  * **Código, base de datos, entidades, campos y enums:** en **inglés** (`item_marks`, `term_marks`, `course_year`, `recovery_strategy`, etc.).
  * **Interfaz de usuario (UX), vistas y mensajes al usuario:** en **español**.

---

## 🧱 Stack tecnológico

* **Backend / Framework:** Symfony (`symfony/security-bundle`, Doctrine ORM).
* **Base de datos:** Modelo relacional documentado en DBML (`item_marks` / `term_marks` con precisión decimal y claves únicas).
* **Interfaz:** En español (Twig / UX a definir en la fase de implementación).

---

## 🗺️ Evolución del diseño (Historial de decisiones)

### 1. Definición del núcleo académico y de asistencia
* Se estableció el modelo base para grados (`degrees`), grupos (`classrooms`), módulos/asignaturas (`subjects`), asignación por grupo (`classroom_subjects`), alumnado (`students` con soporte NEAE y teléfonos de contacto `student_telephones`), horarios (`schedules`) y asistencias (`attendances`).
* Se incluyó `total_hours` en `subjects` pensando en el control del umbral del 20% de faltas y `incident` dentro de `attendance_status` para futuras conexiones disciplinarias.

### 2. Modelo de evaluación en dos niveles y recuperaciones
* Se estructuró la evaluación en `evaluation_categories` (porcentajes de la programación didáctica) y `evaluation_items` (pruebas/ítems, con soporte para reparto equitativo cuando `weight` es `null`).
* Se incorporaron `terms` (con `starts_at`, `ends_at`, `grading_deadline_at` y `grace_until_at`) y estrategias de recuperación (`replace`, `highest`, `average`, `highest_capped`) vinculadas opcionalmente a `recovers_item_id`.

### 3. Clarificación de nomenclatura: `marks` frente a `grade`
* Se detectó ambigüedad al usar `grade` tanto para el **curso** (`1`, `2`) como para las **calificaciones**.
* Se adoptó `marks` para las tablas de calificaciones (`item_marks` y `term_marks`) y se renombró el curso en `subjects` a `course_year` para mantener un vocabulario inequívoco.

### 4. Escalabilidad histórica y configuración de notas en lógica de negocio
* Se decidió mantener reglas como la coherencia entre `term` y `academic_year`, el comportamiento de los ítems de recuperación y los rangos de calificación en la **lógica de negocio**, favoreciendo consultas SQL más limpias y rápidas en el curso activo.
* Se contempló separar el **historial** de calificaciones y la trazabilidad de cambios de grupo/curso de cara a evitar que las tablas operativas del año en curso se sobrecarguen con el paso de los cursos.
* Se acordó dejar un tipo decimal amplio para las notas y controlar el máximo permitido desde la configuración/lógica de la aplicación.

### 5. Simplificación de autenticación para la V1 (`users`) y elección de Symfony
* Para la V1 personal se descartó crear una tabla `employees` compleja desde el inicio, optando por una tabla **`users`** compatible con Symfony Security (`email`, `roles` en JSON, `password`, `is_active`) y vinculando `user_id` en `classroom_subjects`.
* Se confirmó **Symfony** por encajar con una arquitectura estricta, servicios desacoplados, Doctrine y `security.yaml` con `role_hierarchy` (`ROLE_ADMIN` heredando de `ROLE_TEACHER`).

### 6. Estructura de carpetas y organización de `src/`
* Se definió una organización limpia en `src/`:
  * `src/Controller/`: controladores delgados y rutas de acceso.
  * `src/Service/`: servicios de negocio (cálculo de calificaciones, pesos y recuperaciones).
  * `src/Exception/`: excepciones personalizadas para errores de dominio y diagnóstico claro.
  * Uso nativo de `LoggerInterface` (Monolog) de Symfony para el registro de eventos y errores.

---

## 📄 Licencia

Copyright (c) 2026 Alexander Álvarez Marques. Todos los derechos reservados.

Este proyecto es de carácter propietario. Queda prohibida la copia,
modificación, distribución o reutilización total o parcial de este código
sin autorización expresa y por escrito del autor. Consulta el archivo
[LICENSE](LICENSE.txt) para más detalles.
## 🗄️ Modelo de datos (DBML - V1)

---

```dbml
enum relationship {
  mother
  father
  student
  legal_guardian
  other
}

enum attendance_status {
  present
  unjustified_absence
  justified_absence
  unjustified_late
  justified_late
  incident
}

enum term_name {
  first_term
  second_term
  third_term
  final_evaluation
}

enum recovery_strategy {
  replace
  highest
  average
  highest_capped
}

table users {
  id integer [pk, increment]
  name varchar(100) [not null]
  surname varchar(150) [not null]
  email varchar(180) [not null, unique]
  roles json [not null]
  password varchar(255) [not null]
  is_active boolean [default: true, not null]
  created_at timestamp [default: `now()`, not null]
  updated_at timestamp
}

table degrees {
  id integer [pk, increment]
  code varchar(8) [not null, unique]
  name varchar(150) [not null]
  enable boolean [default: true, not null]
  created_at timestamp [default: `now()`, not null]
  updated_at timestamp
  deleted_at timestamp
}

table classrooms {
  id integer [pk, increment]
  degree_id integer [not null]
  name varchar(100) [not null]
  acronym varchar(12) [not null]
  academic_year varchar(20) [not null]
  created_at timestamp [default: `now()`, not null]
  updated_at timestamp

  indexes {
    degree_id
    (degree_id, name, academic_year) [unique]
  }
}

table subjects {
  id integer [pk, increment]
  degree_id integer [not null]
  code varchar(12) [not null, unique]
  name varchar(150) [not null]
  acronym varchar(12) [not null]
  course_year smallint [not null]
  total_hours integer [not null]
  created_at timestamp [default: `now()`, not null]
  updated_at timestamp

  indexes {
    degree_id
    (degree_id, acronym) [unique]
  }
}

table classroom_subjects {
  id integer [pk, increment]
  classroom_id integer [not null]
  subject_id integer [not null]
  user_id integer [not null]
  room varchar(50)
  created_at timestamp [default: `now()`, not null]
  updated_at timestamp

  indexes {
    classroom_id
    subject_id
    user_id
    (classroom_id, subject_id) [unique]
  }
}

table students {
  id integer [pk, increment]
  classroom_id integer [not null]
  name varchar(100) [not null]
  surname varchar(150) [not null]
  ssesn boolean [default: false, not null]
  specific_educational_support_needs text
  created_at timestamp [default: `now()`, not null]
  updated_at timestamp

  indexes {
    classroom_id
  }
}

table student_telephones {
  id integer [pk, increment]
  student_id integer [not null]
  telephone varchar(20) [not null]
  relationship relationship [not null]
  created_at timestamp [default: `now()`, not null]
  updated_at timestamp

  indexes {
    student_id
  }
}

table schedules {
  id integer [pk, increment]
  classroom_subject_id integer [not null]
  weekday smallint [not null]
  start_time time [not null]
  end_time time [not null]
  created_at timestamp [default: `now()`, not null]
  updated_at timestamp

  indexes {
    classroom_subject_id
    (classroom_subject_id, weekday, start_time)
  }
}

table attendances {
  id integer [pk, increment]
  schedule_id integer [not null]
  student_id integer [not null]
  session_date date [not null]
  status attendance_status [not null]
  created_at timestamp [default: `now()`, not null]
  updated_at timestamp

  indexes {
    student_id
    (schedule_id, session_date)
    (schedule_id, student_id, session_date) [unique]
  }
}

table terms {
  id integer [pk, increment]
  academic_year varchar(20) [not null]
  name term_name [not null]
  starts_at date [not null]
  ends_at date [not null]
  grading_deadline_at date [not null]
  grace_until_at date
  created_at timestamp [default: `now()`, not null]
  updated_at timestamp

  indexes {
    (academic_year, name) [unique]
  }
}

table evaluation_categories {
  id integer [pk, increment]
  classroom_subject_id integer [not null]
  name varchar(100) [not null]
  weight decimal(5,2) [not null]
  created_at timestamp [default: `now()`, not null]
  updated_at timestamp

  indexes {
    classroom_subject_id
    (classroom_subject_id, name) [unique]
  }
}

table evaluation_items {
  id integer [pk, increment]
  evaluation_category_id integer [not null]
  term_id integer [not null]
  recovers_item_id integer
  title varchar(150) [not null]
  weight decimal(5,2)
  is_recovery boolean [default: false, not null]
  recovery_strategy recovery_strategy
  due_date date
  allows_grace boolean [default: false, not null]
  created_at timestamp [default: `now()`, not null]
  updated_at timestamp

  indexes {
    evaluation_category_id
    term_id
    recovers_item_id
    (evaluation_category_id, term_id)
  }
}

table item_marks {
  id integer [pk, increment]
  evaluation_item_id integer [not null]
  student_id integer [not null]
  mark decimal(4,2)
  submitted_at timestamp
  comments text
  created_at timestamp [default: `now()`, not null]
  updated_at timestamp

  indexes {
    evaluation_item_id
    student_id
    (evaluation_item_id, student_id) [unique]
  }
}

table term_marks {
  id integer [pk, increment]
  classroom_subject_id integer [not null]
  term_id integer [not null]
  student_id integer [not null]
  calculated_mark decimal(4,2)
  final_mark decimal(4,2)
  is_locked boolean [default: false, not null]
  created_at timestamp [default: `now()`, not null]
  updated_at timestamp

  indexes {
    classroom_subject_id
    term_id
    student_id
    (classroom_subject_id, term_id, student_id) [unique]
  }
}

Ref: classrooms.degree_id > degrees.id
Ref: subjects.degree_id > degrees.id
Ref: classroom_subjects.classroom_id > classrooms.id
Ref: classroom_subjects.subject_id > subjects.id
Ref: classroom_subjects.user_id > users.id
Ref: students.classroom_id > classrooms.id
Ref: student_telephones.student_id > students.id
Ref: schedules.classroom_subject_id > classroom_subjects.id
Ref: attendances.schedule_id > schedules.id
Ref: attendances.student_id > students.id
Ref: evaluation_categories.classroom_subject_id > classroom_subjects.id
Ref: evaluation_items.evaluation_category_id > evaluation_categories.id
Ref: evaluation_items.term_id > terms.id
Ref: evaluation_items.recovers_item_id > evaluation_items.id
Ref: item_marks.evaluation_item_id > evaluation_items.id
Ref: item_marks.student_id > students.id
Ref: term_marks.classroom_subject_id > classroom_subjects.id
Ref: term_marks.term_id > terms.id
Ref: term_marks.student_id > students.id
