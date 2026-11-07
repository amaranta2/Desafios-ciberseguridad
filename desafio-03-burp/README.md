# Desafío 3: La FLAG perdida (Burp Suite + Inyección SQL)

**Escenario:** un portal de login en `http://desafiouls.free.nf` oculta una FLAG
detrás de un panel de administrador. El objetivo es interceptar el login con
Burp Suite y acceder sin credenciales válidas.

## Proceso
1. Abrí Burp Suite Community Edition por primera vez. En la ventana inicial
   dejé marcada **Temporary project in memory** → **Next** → **Start Burp**
   (configuración por defecto).
2. Abrí el navegador integrado de Burp e intenté entrar a
   `http://desafiouls.free.nf`.
   **Error:** la página no cargaba, solo mostraba la promoción de PortSwigger.
   **Causa:** el interceptor (**Intercept is on**) estaba deteniendo la
   petición antes de que llegara al sitio.
   **Solución:** pasar a **Intercept is off** para que la página cargara
   normalmente, y recién activarlo (**Intercept is on**) antes de enviar el
   login.
3. Cargó el "Portal de Acceso - CTF" con campos Usuario y Contraseña
   (el aviso "No es seguro" del navegador es esperable: el sitio es HTTP sin
   certificado).
4. Con el interceptor activado, ingresé datos cualquiera (`test` / `test`) y
   pulsé Entrar. La petición quedó detenida en Burp: `POST /index.php` con
   cuerpo `username=test&password=test`.
5. Edité el cuerpo de la petición, reemplazando el valor de `username` por el
   payload de la pista 3: `' or 1=1-- -`, dejando
   `username=' or 1=1-- -&password=test`.
6. Pulsé **Forward** para reenviar la petición modificada.

## Resultados

| Misión | Resultado |
|---|---|
| Endpoint del login | `POST /index.php` (sitio `desafiouls.free.nf`) |
| Payload usado | `' or 1=1-- -` en el campo `username` |
| Acceso | "Acceso Concedido" |
| **Flag** | `CTF{PHP_SQLi_Byp4ss_M4st3r}` |

## Evidencias
- Petición interceptada original (`username=test&password=test`).
- Petición editada con el payload antes de Forward.
- Pantalla final "Acceso Concedido" con la flag.

## Apoyo de IA
Usé Claude (Anthropic) con estas consultas:
- Cómo usar Burp Suite desde cero (primera vez abriendo el programa)
- Por qué no cargaba el sitio con el interceptor activado
- Cómo editar el campo `username` en la petición interceptada y reenviarla

## Conclusión
El servidor arma la consulta SQL concatenando directamente el valor de
`username`, sin validar ni escapar lo que se ingresa. Al mandar
`' or 1=1-- -`, la comilla cierra el valor esperado, `or 1=1` convierte la
condición en siempre verdadera y `-- -` comenta el resto de la consulta
(incluida la verificación de la contraseña), por lo que el sistema concede
acceso sin conocer credenciales reales. La defensa correcta es usar consultas
parametrizadas (prepared statements) en vez de concatenar texto del usuario
directamente en el SQL. También aprendí el flujo básico de Burp: interceptar,
editar la petición en el panel Pretty/Raw, y Forward para reenviarla.
