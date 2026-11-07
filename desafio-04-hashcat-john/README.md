# Desafío 4: John the Ripper y Hashcat

**Escenario:** dos retos de cracking de hashes en Kali Linux (VM). Reto 1:
extraer y romper el hash de un `.zip` protegido con John the Ripper. Reto 2:
romper un hash SHA-512 crypt con un ataque de máscara en Hashcat, siguiendo
una política de contraseñas conocida, sin usar diccionario.

## Proceso

### Preparación
1. Descargué `secreto.zip` y `h2.txt` directamente desde el navegador dentro
   de la VM de Kali (no hizo falta traspasarlos desde el host).
   **Error:** `cd ~/Descargas` → `no such file or directory`.
   **Solución:** la carpeta en Kali está en inglés, `cd ~/Downloads`.

### Reto 1: John the Ripper
1. Extraje el hash del zip:
   ```
   zip2john secreto.zip > hash.txt
   ```
2. **Error:** corrí `john hash.txt` sin indicar diccionario. John usó por
   defecto la lista pequeña `password.lst` y luego pasó a modo
   `incremental:ASCII` (fuerza bruta total), lo que lo hacía inviable.
   **Solución:** apuntar explícitamente al diccionario grande:
   ```
   john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
   ```
3. **Error:** `sudo gunzip /usr/share/wordlists/rockyou.txt.gz` dio
   `No such file or directory`.
   **Causa:** el archivo ya estaba descomprimido de antes.
   **Verificación:** `ls -lh /usr/share/wordlists/ | grep -i rockyou`
   confirmó que `rockyou.txt` ya existía.
4. Con el diccionario correcto, John encontró la contraseña en menos de un
   segundo.

### Reto 2: Hashcat (ataque de máscara)
1. Revisé el hash de `h2.txt`: empieza con `$6$`, que corresponde a
   SHA-512 crypt (modo `-m 1800` en Hashcat).
2. Traduje la política de contraseñas (1 mayúscula + 5 minúsculas +
   2 dígitos) a la máscara `?u?l?l?l?l?l?d?d`.
3. Primer intento:
   ```
   hashcat -m 1800 -a 3 h2.txt ?u?l?l?l?l?l?d?d
   ```
   Velocidad ~915 H/s, tiempo estimado: **1 año, 2 días**. Inviable.
4. Agregué kernel optimizado y más workload:
   ```
   hashcat -m 1800 -a 3 -O -w 3 h2.txt ?u?l?l?l?l?l?d?d
   ```
   Subió apenas a ~924 H/s (prácticamente sin cambio: el cuello de botella
   no era el kernel).
5. Subí los núcleos asignados a la VM de 1 a 4 (VirtualBox → Configuración →
   Sistema → Procesador) y volví a correr el mismo comando. Velocidad subió
   a ~1334 H/s, tiempo estimado: **268 días**. Sigue siendo inviable para
   este entorno.
6. Decidí no continuar el ataque: documentar el intento tal como quedó, en
   vez de forzarlo o cambiar de sistema operativo, ya que la actividad pide
   trabajar en Kali Linux.

## Resultados

| Reto | Herramienta | Resultado |
|---|---|---|
| Reto 1 — secreto.zip | John the Ripper (`zip2john` + `--wordlist=rockyou.txt`) | **Panda123** (encontrada en <1 s) |
| Reto 2 — h2.txt | Hashcat, ataque de máscara (`-m 1800 -a 3 ?u?l?l?l?l?l?d?d`) | No completado en este entorno. Con 4 núcleos de CPU virtual: ~1334 H/s, tiempo estimado ~268 días para el espacio total (~30.900 millones de combinaciones) |

## Evidencias
- Extracción del hash con `zip2john`.
- Captura con `Panda123` y "Session completed" de John.
- Estado de Hashcat con velocidad ~915 H/s y estimado de 1 año (1 núcleo).
- Estado de Hashcat tras `-O -w 3`, ~924 H/s.
- Estado de Hashcat con 4 núcleos, ~1334 H/s, estimado 268 días.
- Configuración de CPUs de la VM en VirtualBox.

## Apoyo de IA
Usé Claude (Anthropic) con estas consultas:
- Cómo pasar archivos del host a una VM de Kali en VirtualBox (opción que
  finalmente no usé, al descargar los archivos directo desde el navegador
  de Kali)
- Por qué John tardaba tanto corriendo `john hash.txt` sin diccionario
- Cómo identificar el modo correcto de Hashcat (`-m 1800`) a partir del
  prefijo del hash (`$6$`)
- Cómo mejorar el rendimiento de Hashcat dentro de una VM (kernel
  optimizado, más núcleos de CPU)

## Conclusión
El contraste entre los dos retos muestra por qué el algoritmo de hash importa
tanto como la contraseña misma. El hash del zip (PKZIP) se rompió en menos de
un segundo con un diccionario, porque es un algoritmo rápido y sin
iteraciones de por medio. El hash SHA-512 crypt, en cambio, aplica entre 1000
y 1500 iteraciones por intento a propósito, para que cada prueba sea costosa;
sumado a que una VM sin GPU dedicada solo ofrece CPU virtualizada, el ataque
de máscara —aunque bien planteado y mucho más acotado que una fuerza bruta
genérica— resultó inviable de completar en un tiempo razonable. Esto confirma
el consejo de la pista: conocer la política reduce el espacio de búsqueda,
pero un algoritmo de hash lento sigue siendo una defensa real incluso frente
a un atacante que conoce esa política. También aprendí a indicar explícitamente
el diccionario a John (sin la bandera `--wordlist`, cae en modo de fuerza
bruta) y a identificar el tipo de hash por su prefijo (`$6$` → SHA-512 crypt).
