# Desafío 7: SQL Injection — El catálogo y El acceso (2 niveles)

**Escenario:** laboratorio web (PHP/Apache + MariaDB, en Docker) con dos
ejercicios de SQL Injection. Cada uno tiene una versión vulnerable (concatena
el texto del usuario directo en el SQL) y una protegida (usa consultas
preparadas con PDO).

**Nota de integridad:** junto a la ficha de alumnos se entregó también la
guía técnica del docente, con los payloads ya resueltos. No usé esos
payloads directamente del documento; los construí razonando sobre la
estructura del SQL mostrado en "Inspeccionar el SQL utilizado" de cada
ejercicio.

## Proceso

### Preparación
1. Abrí Docker Desktop (ya instalado de un desafío anterior).
2. Descomprimí el kit, entré a `Proyecto_Docker` (contiene `compose.yaml`).
3. Abrí PowerShell desde esa carpeta y ejecuté:
   ```
   docker compose up --build -d
   docker compose ps
   ```
4. Entré a `http://localhost:8080`: cargó el sitio "SQ Injection Studies"
   con los capítulos **01 El catálogo** y **02 El acceso**.

### Nivel 1 — El catálogo
1. En **Consulta vulnerable**, busqué `Python`: devolvió un resultado normal
   (curso de Python), confirmando el funcionamiento base.
2. Abrí **Inspeccionar el SQL utilizado** para ver la consulta real:
   ```sql
   SELECT titulo, autor, categoria, descripcion, codigo, publico
   FROM recursos
   WHERE titulo LIKE '%TEXTO%' AND publico = 1 ORDER BY id
   ```
3. Razoné la entrada necesaria a partir de esa consulta:
   - Una comilla simple (`'`) para cerrar el `%'` de la búsqueda.
   - `OR 1=1` para forzar una condición siempre verdadera.
   - `-- -` para comentar el resto de la línea, incluido `AND publico = 1`.
4. Probé en **Consulta vulnerable**:
   ```
   %' OR 1=1 -- -
   ```
   Resultado: aparecieron los 6 recursos, incluido el **"Memorando interno
   del aula"** (marcado como RESERVADO) y la bandera
   **`AULA{BUSQUEDA_ALTERADA}`**.
5. Repetí exactamente la misma entrada en **Consulta protegida**: la
   respuesta usa una sentencia con parámetro (`LIKE :termino`, vinculado por
   PDO) y **no mostró** el recurso reservado — el texto se trató como dato
   literal, no como SQL.

### Nivel 2 — El acceso
6. Probé el comportamiento normal: usuario `coordinador` con una contraseña
   incorrecta en **Consulta vulnerable** → acceso denegado.
7. Inspeccioné el SQL: la condición exige usuario correcto **AND** clave
   correcta (comparada vía `SHA2(...)`).
8. Apliqué el mismo razonamiento que en el Nivel 1 — cerrar el usuario con
   una comilla y comentar el resto de la condición (incluida la verificación
   de clave) — y probé:
   - Usuario: `%' OR 1=1 -- -`
   - Contraseña: cualquier valor
   Resultado: **"Acceso simulado concedido"** como Coordinación del
   Laboratorio, con la bandera **`AULA{LOGIN_SIN_CLAVE}`**. El SQL mostrado
   confirma que el comentario (`-- -`) anuló la comparación de contraseña.
9. Repetí la misma entrada en **Consulta protegida**: "Acceso denegado. Las
   credenciales no coincidieron" — la consulta protegida usa `usuario =
   :usuario` parametrizado, así que la entrada no altera el SQL.
10. Verifiqué que la versión protegida sí funciona con una cuenta legítima:
    usuario `visitante`, contraseña `Practica2026!` → acceso simulado
    concedido correctamente.

## Resultados

| Nivel | Entrada usada | Resultado |
|---|---|---|
| 1 — Catálogo (vulnerable) | `%' OR 1=1 -- -` | Memorando interno expuesto — `AULA{BUSQUEDA_ALTERADA}` |
| 1 — Catálogo (protegida) | misma entrada | No expone el recurso reservado |
| 2 — Acceso (vulnerable) | Usuario `%' OR 1=1 -- -` / contraseña cualquiera | Acceso simulado como coordinador — `AULA{LOGIN_SIN_CLAVE}` |
| 2 — Acceso (protegida) | misma entrada | Acceso denegado |
| 2 — Acceso (protegida, cuenta real) | `visitante` / `Practica2026!` | Acceso simulado concedido |

## Evidencias
- Nivel 1: resultados con el memorando y la bandera visibles (modo vulnerable).
- Nivel 1: SQL inspeccionado del modo protegido, sin el recurso reservado.
- Nivel 2: SQL inspeccionado del modo vulnerable, con la bandera obtenida.
- Nivel 2: acceso denegado en modo protegido con la misma entrada.
- Nivel 2: acceso concedido en modo protegido con `visitante` / `Practica2026!`.

## Apoyo de IA
Usé Claude (Anthropic) con estas consultas:
- Guía para razonar la estructura del payload a partir del SQL mostrado.

## Conclusión
Ambos niveles muestran la misma falla de fondo: concatenar texto del
usuario directo dentro de una instrucción SQL permite que ese texto cambie
la lógica de la consulta, no solo su contenido. Una comilla cierra el
campo esperado, `OR 1=1` fuerza una condición siempre verdadera, y `-- -`
comenta cualquier filtro o verificación que venga después (el `AND publico
= 1` en el catálogo, la comparación de contraseña en el acceso). La defensa
en ambos casos fue la misma: consultas preparadas (PDO), donde la entrada
se envía como parámetro y nunca se interpreta como código SQL — por eso la
misma entrada maliciosa, repetida sin cambios, no tuvo efecto en el modo
protegido, mientras que una cuenta legítima siguió funcionando con
normalidad. También confirma que la protección contra SQLi y la
verificación de contraseñas (hash, `password_verify`) son controles
distintos y complementarios: uno protege la estructura de la consulta, el
otro protege el secreto en sí.
