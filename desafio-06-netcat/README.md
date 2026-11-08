# Desafío 6: El registro perdido (Netcat)

**Escenario:** un servidor TCP local (`servidor_ctf.py`) expone un servicio
de texto simple. El objetivo es encontrar el puerto en el que escucha
(dentro de un rango permitido) y, usando solo Netcat, conversar con el
servicio para localizar un "registro" y obtener la FLAG.

## Proceso
1. En Kali, instalé lo necesario (ya tenía Python 3; agregué netcat):
   ```
   sudo apt update
   sudo apt install python3 netcat-openbsd
   ```
2. Descargué `servidor_ctf.py` directo desde el navegador de Kali.
3. **Terminal A:** levanté el servidor en modo reto:
   ```
   cd ~/Downloads
   python3 servidor_ctf.py --modo reto
   ```
   Quedó escuchando (el enunciado no revela el puerto exacto en el mensaje,
   solo confirma que el rango permitido es 4644–4648).
4. **Terminal B:** escaneé el rango de puertos permitido con Netcat, sin
   revisar el código del servidor:
   ```
   nc -nvz -w 1 127.0.0.1 4644-4648
   ```
   Solo el puerto **4646** respondió `open`; el resto quedó cerrado/sin
   respuesta.
5. Me conecté al puerto encontrado:
   ```
   nc -n -w 15 127.0.0.1 4646
   ```
   El servicio saludó con: *"Archivo CTF local. Escribe AYUDA para
   comenzar."*
6. Escribí `AYUDA` y until Enter. Respondió con la lista de comandos:
   `AYUDA, INFO, LEER <nombre>, SALIR`.
7. Escribí `INFO`. Respondió: *"Archivo logico disponible: registro.txt"*.
8. Escribí `LEER registro.txt`. El servicio devolvió la FLAG directamente.
9. Cerré la sesión con `SALIR` (el propio servicio la terminó al recibir
   ese comando) y detuve el servidor en la Terminal A con `Ctrl+C`.

## Resultados

| Misión | Resultado |
|---|---|
| Puerto encontrado | `4646` (dentro del rango permitido 4644–4648) |
| Comando para listar recursos | `INFO` → reveló `registro.txt` |
| Comando para obtener la flag | `LEER registro.txt` |
| **Flag** | `FLAG{consulta_el_servicio_2026}` |

## Evidencias
- Escaneo de puertos con `nc -nvz -w 1`, mostrando `4646` como `open`.
- Conexión al servicio y mensaje de bienvenida con `AYUDA`.
- Respuesta de `INFO` mostrando `registro.txt`.
- Secuencia completa: `LEER registro.txt` → FLAG → `SALIR` → "Sesion
  terminada."

## Apoyo de IA
Usé Claude (Anthropic) con estas consultas:
- Confirmación de los pasos con Netcat (escaneo de rango y conexión)

## Conclusión
Netcat sirvió aquí como cliente TCP genérico: primero para descubrir cuál
de varios puertos candidatos estaba realmente escuchando (`-nvz`, un
escaneo rápido sin enviar datos), y después para mantener una conexión
interactiva de texto plano con el servicio (sin `-z`, enviando línea por
línea). El reto en sí fue sencillo una vez conectado, porque el propio
servicio documentaba sus comandos con `AYUDA`; lo importante fue no asumir
el puerto de memoria y comprobarlo explícitamente, como pedía el enunciado.
