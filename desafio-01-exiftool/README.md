# Desafío 1: La Huella Digital del Pato (ExifTool)

**Escenario:** analizar los metadatos de `20261006_143910.jpg` para verificar su
timestamp, identificar el dispositivo, extraer su ubicación GPS y detectar
señales de edición.

## Proceso
1. Descargué ExifTool portable para Windows y renombré `exiftool(-k).exe` a
   `exiftool.exe`.
2. Coloqué la imagen en la misma carpeta y corrí `exiftool.exe archivo.jpg`.
3. **Error:** PowerShell no reconoció el comando (`CommandNotFoundException`),
   ya que por seguridad no ejecuta binarios de la carpeta actual sin indicarlo.
   **Solución:** usar `.\exiftool.exe archivo.jpg`.
4. Exporté la salida completa con `.\exiftool.exe archivo.jpg > resultado.txt`
   (adjunto en esta carpeta).

## Resultados

| Misión | Resultado |
|---|---|
| Timestamp | Coincide: nombre de archivo, `DateTimeOriginal` y offset (-03:00) son idénticos |
| Hardware | Samsung Galaxy S23 Ultra (SM-S918B), lente principal trasera, f/1.7, ISO 64, 1/500s, 23mm equiv. |
| GPS | No stripped. Lat/Lon: `29°54'9.72"S, 71°15'7.20"W` → decimal: **-29.9027, -71.2520** |
| Integridad | Sin MakerNotes ni miniaturas. `Software: Adobe Photoshop 25.0` → la imagen fue editada tras la captura |

**Flag:** `FLAG{SM-S918B_-29.9027_-71.2520_AdobePhotoshop25.0}`

## Apoyo de IA
Usé Claude (Anthropic) para resolver el error de PowerShell y verificar la
conversión de coordenadas GMS a decimal.

## Conclusión
Los metadatos EXIF revelan mucho más de lo que parece a simple vista: modelo
exacto del equipo, configuración de cámara, ubicación precisa e incluso el
software usado para editar la imagen después de la captura. También quedó claro
que PowerShell tiene restricciones de seguridad para ejecutar binarios locales,
algo básico pero necesario de manejar.
