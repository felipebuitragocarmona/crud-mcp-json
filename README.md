# Tutorial Completo: Crear un Servidor MCP para Gestión de Estudiantes con FastMCP

## 📋 Introducción a MCP (Model Context Protocol)

MCP (Model Context Protocol) es un estándar abierto que permite a modelos de IA como Claude acceder a herramientas y datos externos. FastMCP es una implementación Python que facilita la creación de servidores MCP.

### ¿Qué vamos a construir?
Un servidor MCP para gestión de estudiantes que expone herramientas CRUD (Crear, Leer, Actualizar, Eliminar) a través de una API que Claude puede usar directamente.

---

## 🏗️ Arquitectura del Proyecto

```
mi-servidor-estudiantes/
├── students_server.py      # Código principal
├── students.json          # Base de datos (se crea automáticamente)
├── students_mcp.log      # Archivo de logs
└── requirements.txt      # Dependencias
```

---

## 🔧 Paso 1: Configuración del Entorno

### 1.1 Instalar dependencias
Para este proyecto se requiere python>=3.10
```bash
# Crear entorno virtual
python -m venv venv

# Activar entorno virtual
# En Windows:
venv\Scripts\activate
# En Linux/Mac:
source venv/bin/activate

# Instalar FastMCP
pip install fastmcp
```

### 1.2 Crear requirements.txt
```txt
fastmcp==2.14.4
mcp-proxy==0.11.0
```

---

## 📝 Paso 2: Explicación del Código

### 2.1 Importaciones y Configuración Inicial

```python
from typing import Any
import logging
import sys
import json
from pathlib import Path
from datetime import datetime
from fastmcp import FastMCP
```

**¿Qué hace cada import?**
- `typing.Any`: Para tipado dinámico
- `logging`: Sistema de registro de eventos
- `json`: Manejo de archivos JSON
- `Path`: Manejo de rutas multiplataforma
- `datetime`: Manejo de fechas/horas
- `FastMCP`: Clase principal del servidor

### 2.2 Configuración de Logging

```python
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('students_mcp.log'),
        logging.StreamHandler(sys.stderr)
    ]
)
logger = logging.getLogger(__name__)
```

**Configuración:**
- `level=logging.INFO`: Registra desde nivel INFO en adelante
- Dos handlers: archivo y consola
- Formato con timestamp, nombre, nivel y mensaje

### 2.3 Inicialización del Servidor MCP

```python
mcp = FastMCP("students", port=8000)
```
Crea un servidor llamado "students" en el puerto 8000 usando transporte SSE (Server-Sent Events).

---

## 🗄️ Paso 3: Funciones de Persistencia

### 3.1 Estructura del Archivo JSON
El archivo se debe nombrar `students.json`
```json
[
  {
    "id": 1,
    "name": "Juan Pérez",
    "email": "juan@example.com",
    "age": 21,
    "career": "Ingeniería de Sistemas",
    "semester": 5,
    "created_at": "2024-01-20T10:30:00"
  }
]
```

### 3.2 Función `load_students()`
```python
# Constants
STUDENTS_FILE = Path("students.json")

def load_students() -> list[dict[str, Any]]:
    """Carga estudiantes desde archivo JSON."""
    try:
        if STUDENTS_FILE.exists():
            with open(STUDENTS_FILE, 'r', encoding='utf-8') as f:
                data = json.load(f)
                logger.info(f"Loaded {len(data)} students from file")
                return data
        else:
            logger.info("Students file doesn't exist, creating new one")
            return []
    except Exception as e:
        logger.error(f"Error loading students: {str(e)}")
        return []
def save_students(students: list[dict[str, Any]]) -> bool:
    """Save students to JSON file."""
    try:
        with open(STUDENTS_FILE, 'w', encoding='utf-8') as f:
            json.dump(students, f, indent=2, ensure_ascii=False)
        logger.info(f"Saved {len(students)} students to file")
        return True
    except Exception as e:
        logger.error(f"Error saving students: {str(e)}")
        return False
```

**Características:**
- Verifica si el archivo existe
- Usa encoding UTF-8 para caracteres especiales
- Maneja errores con try-except
- Retorna lista vacía si hay errores

---

## 🛠️ Paso 4: Definición de Herramientas (Tools)

### 4.1 Anatomía de un Tool MCP

```python
@mcp.tool()
async def nombre_tool(param1: tipo, param2: tipo = valor_default) -> str:
    """Descripción de la herramienta.
    
    Args:
        param1: Descripción del parámetro 1
        param2: Descripción del parámetro 2
    """
    # Lógica de la herramienta
    return "Resultado"
```

### 4.2 Tool: `add_student`

```python
@mcp.tool()
async def add_student(
        name: str,
        email: str,
        age: int,
        career: str,
        semester: int
) -> str:
    """Add a new student to the database.

    Args:
        name: Full name of the student
        email: Email address of the student
        age: Age of the student
        career: Career/program the student is enrolled in
        semester: Current semester number
    """
    logger.info(f"add_student called for: {name}")

    students = load_students()

    if any(s['email'] == email for s in students):
        logger.warning(f"Student with email {email} already exists")
        return f"Error: A student with email {email} already exists."

    new_student = {
        "id": len(students) + 1,
        "name": name,
        "email": email,
        "age": age,
        "career": career,
        "semester": semester,
        "created_at": datetime.now().isoformat()
    }

    students.append(new_student)

    if save_students(students):
        logger.info(f"Successfully added student: {name}")
        return f"✅ Student added successfully!\n\nID: {new_student['id']}\nName: {name}\nEmail: {email}\nCareer: {career}\nSemester: {semester}"
    else:
        logger.error(f"Failed to save student: {name}")
        return "Error: Could not save student to file."
```

### 4.3 Tool: `list_students` con Filtro

```python
@mcp.tool()
async def list_students(filter_career: str = None) -> str:
    """Lista estudiantes, opcionalmente filtrados por carrera."""
    
    students = load_students()
    
    # Aplicar filtro si se proporciona
    if filter_career:
        students = [s for s in students 
                   if s['career'].lower() == filter_career.lower()]
    
    # Formatear resultado
    result = f"📚 **Total: {len(students)}**\n\n"
    for student in students:
        result += f"ID: {student['id']}\nName: {student['name']}\n..."
    
    return result
```
### 4.4 Tool: `get_student`

```python
@mcp.tool()
async def get_student(student_id: int) -> str:
    """Get details of a specific student by ID.

    Args:
        student_id: The ID of the student to retrieve
    """
    logger.info(f"get_student called for ID: {student_id}")

    students = load_students()
    student = next((s for s in students if s['id'] == student_id), None)

    if not student:
        logger.warning(f"Student with ID {student_id} not found")
        return f"Student with ID {student_id} not found."

    result = f"""
        📋 **Student Details**
        
        ID: {student['id']}
        Name: {student['name']}
        Email: {student['email']}
        Age: {student['age']}
        Career: {student['career']}
        Semester: {student['semester']}
        Added: {student.get('created_at', 'N/A')}
"""

    logger.info(f"Retrieved student: {student['name']}")
    return result
```
### 4.5 Tool: `update_student` (Actualización Parcial)

```python
@mcp.tool()
async def update_student(
        student_id: int,
        name: str = None,
        email: str = None,
        age: int = None,
        career: str = None,
        semester: int = None
) -> str:
    """Update student information. Only provided fields will be updated.

    Args:
        student_id: The ID of the student to update
        name: New name (optional)
        email: New email (optional)
        age: New age (optional)
        career: New career (optional)
        semester: New semester (optional)
    """
    logger.info(f"update_student called for ID: {student_id}")

    students = load_students()
    student = next((s for s in students if s['id'] == student_id), None)

    if not student:
        logger.warning(f"Student with ID {student_id} not found")
        return f"Student with ID {student_id} not found."

    updated_fields = []
    if name is not None:
        student['name'] = name
        updated_fields.append("name")
    if email is not None:
        student['email'] = email
        updated_fields.append("email")
    if age is not None:
        student['age'] = age
        updated_fields.append("age")
    if career is not None:
        student['career'] = career
        updated_fields.append("career")
    if semester is not None:
        student['semester'] = semester
        updated_fields.append("semester")

    student['updated_at'] = datetime.now().isoformat()

    if save_students(students):
        logger.info(f"Successfully updated student ID {student_id}: {updated_fields}")
        return f"✅ Student updated successfully!\n\nUpdated fields: {', '.join(updated_fields)}\n\nCurrent data:\n{student}"
    else:
        logger.error(f"Failed to update student ID: {student_id}")
        return "Error: Could not save changes to file."

```
### 4.6 Tool: `delete_student` 

```python
@mcp.tool()
async def delete_student(student_id: int) -> str:
    """Delete a student from the database by ID.

    Args:
        student_id: The ID of the student to delete
    """
    logger.info(f"delete_student called for ID: {student_id}")

    students = load_students()
    original_count = len(students)
    students = [s for s in students if s['id'] != student_id]

    if len(students) == original_count:
        logger.warning(f"Student with ID {student_id} not found")
        return f"Student with ID {student_id} not found."

    if save_students(students):
        logger.info(f"Successfully deleted student ID: {student_id}")
        return f"✅ Student with ID {student_id} has been deleted successfully."
    else:
        logger.error(f"Failed to delete student ID: {student_id}")
        return "Error: Could not save changes to file."
```
### 4.7 Tool: `get_stats` 
```python
@mcp.tool()
async def get_stats() -> str:
    """Get statistics about the student database."""
    logger.info("get_stats called")

    students = load_students()

    if not students:
        return "No students in database to generate statistics."

    total = len(students)
    careers = {}
    total_age = 0

    for student in students:
        career = student['career']
        careers[career] = careers.get(career, 0) + 1
        total_age += student['age']

    avg_age = total_age / total

    result = f"""
📊 **Student Database Statistics**

Total Students: {total}
Average Age: {avg_age:.1f}

Students by Career:
"""

    for career, count in sorted(careers.items(), key=lambda x: x[1], reverse=True):
        percentage = (count / total) * 100
        result += f"  • {career}: {count} ({percentage:.1f}%)\n"

    logger.info(f"Generated statistics for {total} students")
    return result
```

---

## 🚀 Paso 5: Ejecutar el Servidor

### 5.1 Bloque Principal

```python
if __name__ == "__main__":
    logger.info("=" * 50)
    logger.info("Starting Students MCP Server (FastMCP SSE)")
    logger.info("=" * 50)
    logger.info("Available tools: add_student, list_students, get_student, delete_student, update_student, get_stats")
    logger.info(f"Data file location: {STUDENTS_FILE.absolute()}")
    logger.info("Server running on SSE transport")
    logger.info("Server will be available at: http://localhost:8000/sse")
    logger.info("Waiting for connections...")

    try:
        # Run with SSE transport
        mcp.run(transport="sse")
    except KeyboardInterrupt:
        logger.info("Server stopped by user")
    except Exception as e:
        logger.error(f"Server error: {str(e)}", exc_info=True)
```

### 5.2 Iniciar el Servidor

```bash
python students_server.py
```

**Salida esperada:**
```
2024-01-20 10:30:00 - __main__ - INFO - ==================================================
2024-01-20 10:30:00 - __main__ - INFO - Starting Students MCP Server (FastMCP SSE)
2024-01-20 10:30:00 - __main__ - INFO - ==================================================
2024-01-20 10:30:00 - __main__ - INFO - Available tools: add_student, list_students, get_student, delete_student, update_student, get_stats
2024-01-20 10:30:00 - __main__ - INFO - Data file location: /ruta/completa/students.json
2024-01-20 10:30:00 - __main__ - INFO - Server running on SSE transport
2024-01-20 10:30:00 - __main__ - INFO - Server will be available at: http://localhost:8000/sse
2024-01-20 10:30:00 - __main__ - INFO - Waiting for connections...
```

---

## 💡 Paso 6: Usar con Claude Desktop

### 6.1 Configurar Claude Desktop

1. **En Mac:** `~/Library/Application Support/Claude/claude_desktop_config.json`
2. **En Windows:** `%APPDATA%\Claude\claude_desktop_config.json`

### 6.2 Agregar Configuración

```json
{
  "mcpServers": {
    "students": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "http://localhost:8000/sse"
      ]
    }
  }
}
```

### 6.3 Reiniciar Claude Desktop
Para aplicar los cambios, reinicia la aplicación Claude Desktop.

---

## 🎯 Paso 7: Ejemplos de Uso con Claude

### 7.1 Agregar un Estudiante
**Tú:**
```
Agrega un estudiante llamado María García, email maria@example.com, 
edad 22, carrera Ingeniería de Sistemas, semestre 6
```

**Claude (usando el tool):**
```
He usado la herramienta add_student para agregar a María:

✅ Student added successfully!

ID: 1
Name: María García
Email: maria@example.com
Career: Ingeniería de Sistemas
Semester: 6
```

### 7.2 Listar Estudiantes
**Tú:**
```
Muéstrame todos los estudiantes de Ingeniería de Sistemas
```

**Claude:**
```
📚 **Total Students: 3**

==================================================
ID: 1
Name: María García
Email: maria@example.com
Age: 22
Career: Ingeniería de Sistemas
Semester: 6
Added: 2024-01-20T10:30:00
...
```

### 7.3 Actualizar Información
**Tú:**
```
Actualiza el estudiante con ID 1 para que esté en semestre 7
```

**Claude:**
```
✅ Student updated successfully!

Updated fields: semester

Current data:
{
  "id": 1,
  "name": "María García",
  "email": "maria@example.com",
  "age": 22,
  "career": "Ingeniería de Sistemas",
  "semester": 7,
  "created_at": "2024-01-20T10:30:00",
  "updated_at": "2024-01-20T11:00:00"
}
```

### 7.4 Ver Estadísticas
**Tú:**
```
¿Cuántos estudiantes tengo y cuál es el promedio de edad?
```

**Claude:**
```
📊 **Student Database Statistics**

Total Students: 15
Average Age: 21.3

Students by Career:
  • Ingeniería de Sistemas: 8 (53.3%)
  • Medicina: 4 (26.7%)
  • Derecho: 3 (20.0%)
```

---

## 🔍 Paso 8: Monitoreo y Debugging

### 8.1 Ver Logs en Tiempo Real
```bash
# Seguir el archivo de log
tail -f students_mcp.log
```

**Ejemplo de logs:**
```
2024-01-20 10:30:15 - __main__ - INFO - add_student called for: María García
2024-01-20 10:30:15 - __main__ - INFO - Loaded 0 students from file
2024-01-20 10:30:15 - __main__ - INFO - Saved 1 students to file
2024-01-20 10:30:15 - __main__ - INFO - Successfully added student: María García
```

### 8.2 Verificar Archivo JSON
```bash
cat students.json | python -m json.tool
```

---

## 🛡️ Paso 9: Mejoras de Seguridad y Robustez

### 9.1 Validación de Datos (Ejemplo)
```python
def validate_student_data(name: str, email: str, age: int) -> list[str]:
    """Valida datos del estudiante y retorna lista de errores."""
    errors = []
    
    if not name or len(name.strip()) < 2:
        errors.append("Nombre debe tener al menos 2 caracteres")
    
    if "@" not in email or "." not in email:
        errors.append("Email inválido")
    
    if age < 16 or age > 80:
        errors.append("Edad debe estar entre 16 y 80")
    
    return errors
```

### 9.2 Backup Automático
```python
def create_backup():
    """Crea backup del archivo de estudiantes."""
    backup_file = STUDENTS_FILE.with_suffix(f".backup.{datetime.now().strftime('%Y%m%d_%H%M%S')}.json")
    if STUDENTS_FILE.exists():
        shutil.copy2(STUDENTS_FILE, backup_file)
        logger.info(f"Backup created: {backup_file}")
```

---

## 📚 Paso 10: Recursos y Referencias

### Documentación Oficial:
1. **FastMCP GitHub**: https://github.com/punkpeye/fastmcp
2. **MCP Protocol**: https://spec.modelcontextprotocol.io
3. **Claude MCP Docs**: https://docs.anthropic.com/en/docs/about-claude/mcp

### Herramientas Adicionales:
```python
# Herramienta para buscar estudiantes
@mcp.tool()
async def search_students(query: str) -> str:
    """Busca estudiantes por nombre o email."""
    students = load_students()
    results = [
        s for s in students 
        if query.lower() in s['name'].lower() 
        or query.lower() in s['email'].lower()
    ]
    # ... retornar resultados
```

---
