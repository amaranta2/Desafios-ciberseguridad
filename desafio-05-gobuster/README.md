# Desafío 5: ORION-7 (Gobuster)

**Escenario:** el portal público de la instalación ficticia ORION-7 esconde
3 fragmentos de una FLAG, dejados por un operador (K. Vidal) antes de irse.
Herramienta principal: Gobuster, atacando desde fuera del contenedor
(`http://localhost:8080`), sin inspeccionar la imagen Docker.

## Proceso

### Preparación del entorno (Windows)
1. Instalé Gobuster descargando el binario `gobuster_Windows_x86_64.zip`
   desde la página de releases oficial en GitHub (Chocolatey no tiene el
   paquete en su repositorio comunitario — lo intenté primero y falló con
   "package was not found").
2. Descargué `common.txt` de SecLists (`Discovery/Web-Content/common.txt`).
3. Levanté el desafío con `levantar.bat`. Antes de ejecutarlo revisé el
   script: corre un contenedor Docker con `--read-only`, `--cap-drop ALL` y
   publica el puerto solo en `127.0.0.1`, es decir, accesible únicamente
   desde el propio equipo.

### Enumeración inicial
4. Escaneo base:
   ```
   gobuster dir -u http://localhost:8080 -w C:\wordlists\common.txt
   ```
   Encontró: `/archive/` (301), `/css/` (301), `index.html` (200),
   `/personal/` (301), `robots.txt` (200).
5. `robots.txt` reveló `Disallow: /personal/` — señal de "no mirar aquí",
   justo lo contrario de lo que haría un atacante.
6. `/archive/` devolvió **403**, pero la propia página de error ya mostraba
   el **fragmento 1/3: `n4d4`**, junto a la pista "si conoces el nombre
   exacto de un archivo, es otra historia".
7. `/personal/` cargó (200): un "registro de turno" del operador K. Vidal,
   que menciona haber escondido archivos viejos en una carpeta no enlazada
   y **"no alcancé a borrar uno de los respaldos"**.
8. En la página principal encontré texto oculto bajo barras negras de estilo
   CSS (clase `censura`). Seleccionando el texto a ciegas (o viendo el
   código fuente con `Ctrl+U`) descubrí que la versión del "agente de
   monitoreo interno" era **2.1**, y que "los sistemas internos solo
   responden a sus solicitudes" — pista de control por User-Agent.

### Encontrar el fragmento 2/3
9. Repetí el escaneo en `/archive/` agregando extensiones de respaldo:
   ```
   gobuster dir -u http://localhost:8080/archive -w C:\wordlists\common.txt -x txt,bak,old,zip,conf,config
   ```
   Encontró `config.bak` (200), sin necesitar aún el User-Agent especial.
10. `config.bak` es un archivo de configuración del portal, con un campo
    `nota_operador` codificado en **Base64**. Al decodificarlo:
    ```
    curl.exe http://localhost:8080/archive/config.bak
    # decodificado:
    "Nota para quien herede esto: el sistema de control quedo en /core/ y
    solo responde al agente de monitoreo, que se identifica como
    SENTINEL/2.1. Desde afuera parece que no existe.
    [fragmento 2/3: 3st4] -K.V."
    ```
    Esto dio el **fragmento 2/3: `3st4`**, el User-Agent correcto
    (**SENTINEL/2.1**, no "MonitorAgent/2.1" como probé al inicio por el
    texto oculto de la versión) y la ruta correcta (**/core/**).

### Encontrar el fragmento 3/3
11. Escaneé `/core/` con el User-Agent correcto:
    ```
    gobuster dir -u http://localhost:8080/core -w C:\wordlists\common.txt -a "SENTINEL/2.1"
    ```
    Encontró `/core/console/` (301).
12. Accedí con el mismo User-Agent:
    ```
    curl.exe -A "SENTINEL/2.1" http://localhost:8080/core/console/
    ```
    La página mostró "Bienvenido, SENTINEL" y el **fragmento 3/3: `0cult0`**.

## Resultados

| Fragmento | Dónde se encontró | Valor |
|---|---|---|
| 1/3 | Página de error 403 de `/archive/` | `n4d4` |
| 2/3 | `nota_operador` (Base64) dentro de `/archive/config.bak` | `3st4` |
| 3/3 | `/core/console/`, accesible solo con User-Agent `SENTINEL/2.1` | `0cult0` |

**FLAG:** `FLAG{n4d4_3st4_0cult0}`

## Evidencias
- Página 403 de `/archive/` con el fragmento 1.
- `robots.txt` con `Disallow: /personal/`.
- Registro de turno en `/personal/`.
- Código fuente de la página principal mostrando el texto oculto
  (versión `2.1` del agente de monitoreo).
- Escaneo de gobuster con extensiones encontrando `config.bak`.
- Contenido de `config.bak` y la nota decodificada.
- Escaneo de `/core/` con el User-Agent `SENTINEL/2.1` mostrando
  `/core/console/`.
- Página final de `/core/console/` con el fragmento 3 y la FLAG completa.

## Apoyo de IA
Usé Claude (Anthropic) con estas consultas:
- Instalación de Gobuster en Windows (Chocolatey no tenía el paquete)
- Interpretación de los resultados de gobuster (códigos 301/403/200)
- Cómo revisar texto oculto por CSS en una página web
- Decodificación de la nota en Base64 encontrada en `config.bak`
- Cómo cambiar el User-Agent en gobuster (`-a`) y en curl (`-A`)
- Si era seguro ejecutar `levantar.bat` sin revisarlo a ciegas

## Conclusión
El reto mostró capas típicas de "seguridad por oscuridad": ocultar una
carpeta de `robots.txt`, tapar texto con CSS, o filtrar por User-Agent no
son controles de acceso reales, solo dificultan un poco la exploración si
no se sabe dónde mirar. Gobuster por sí solo encontró rutas y, con
extensiones, el archivo de respaldo; pero el contexto (el diario del
operador, el texto oculto, el Base64) fue necesario para entender qué
nombre y qué cabecera probar. También aprendí a usar `curl.exe -A` para
suplantar un User-Agent específico y a decodificar Base64 desde la
terminal, además de a revisar un script `.bat` antes de ejecutarlo para
confirmar que era seguro.
