# Práctica de Peritaje Informático
Auditoría Informática – LSI – FCEQyN – UNaM · Sesión práctica de informática forense | peritaje

Trabajo individual - Datos 100 % ficticios

---

## El caso

**Distribuidora Litoral S.A.** llamó a licitación privada (N.° 14/2026) para comprar insumos agrícolas. Cerró el 24/10/2026 y ganó **Agro Norte SRL**, que cotizó apenas por debajo de los precios tope que el Directorio había aprobado en reserva.

Auditoría Interna sospecha que alguien filtró esos precios. El 24/10 a las 08:40, con dos testigos presentes, se secuestró un pendrive del escritorio de **Juan Pérez** (área Compras), se le sacó una imagen forense y se registró su hash en el acta.

Ustedes son el equipo pericial. La pregunta que tienen que responder es:

> **¿Hay evidencia en el pendrive de que los precios tope de la Licitación 14/2026 salieron de la empresa? Si la hay, ¿cuándo, cómo y hacia quién?**

Además, RRHH les entregó por su cuenta una "copia" del pendrive. Tienen que decidir si se puede usar.

---

## Antes de empezar (5 min)

**Antes del día de la clase** (requiere conexión, se hace una sola vez):
```bash
git clone https://github.com/tinxo/peritaje-lsi.git
cd peritaje-lsi
docker compose pull
```

**En la clase:**
```bash
cd peritaje-lsi
docker compose run --rm perito
```

Van a ver el prompt `perito@lab:/casos$`. Desde ahí:

- `/evidencia` es la evidencia original, en **solo lectura**. Así funciona un bloqueador de escritura.
- `/casos` es su carpeta de trabajo. Lo que guarden ahí queda en su computadora, en la carpeta `casos/`.
- **Todo comando que ejecuten queda registrado** en `/casos/.historial_comandos`, con fecha y hora. Eso forma parte de la documentación pericial.

Para salir: `exit`. Para volver a entrar, el mismo comando; su trabajo en `/casos` se conserva.

---

## Etapa 1 – Preservación y adquisición

**1.1 Leer el acta.**
```bash
cat /evidencia/ACTA_DE_SECUESTRO.txt
```

**1.2 Comprobar que la evidencia no es escribible.** Intenten crear un archivo en `/evidencia` y anoten el error.
```bash
touch /evidencia/prueba
```

**1.3 Verificar la integridad del original contra el acta.**
```bash
sha256sum /evidencia/USB_JPerez_Kingston_16GB.dd
```
¿Coincide con el hash del acta?

**1.4 Evaluar la copia de RRHH.**
```bash
sha256sum /evidencia/COPIA_B_entregada_por_RRHH.dd
cmp -l /evidencia/USB_JPerez_Kingston_16GB.dd /evidencia/COPIA_B_entregada_por_RRHH.dd
```
- ¿Cuántos bytes difieren y en qué posición?
- Pista: `cmp` informa la posición contando desde 1 y los valores en octal. Con `xxd -s <posición-1> -l 32 <archivo>` pueden ver el contexto. ¿Qué cambió?
- **¿Se puede usar esa copia como prueba? Fundamenten.**

**1.5 Adquirir su copia de trabajo** con cálculo y verificación de hash:
```bash
dc3dd if=/evidencia/USB_JPerez_Kingston_16GB.dd hof=/casos/usb.dd \
      hash=sha256 log=/casos/adquisicion.log
cat /casos/adquisicion.log
```
`hof` (*hashed output file*) hace que dc3dd vuelva a leer la copia y compruebe que su hash coincida con el del origen.

**1.6 Completar** el acta de adquisición y la cadena de custodia. Copien antes las plantillas a su carpeta de trabajo (desde una terminal fuera del contenedor, en la carpeta del repo): `cp plantillas/*.md casos/`

> Desde acá trabajen **siempre sobre `/casos/usb.dd`**, nunca sobre el original.

---

## Etapa 2 – Análisis 

### 2.1 Estructura del dispositivo 
```bash
mmls usb.dd                       # tabla de particiones
fsstat -o 2048 usb.dd | head -20  # sistema de archivos de la partición
```
- ¿Por qué se usa `-o 2048`?
- ¿Qué sistema de archivos tiene? ¿Cuál es la etiqueta y cuál el Volume ID?

### 2.2 Listado de archivos, incluidos los borrados 
```bash
fls -r -p -o 2048 usb.dd
```
Las entradas con `*` están **borradas**. Para ver los metadatos de una entrada:
```bash
istat -o 2048 usb.dd <número_de_entrada>
```
- ¿Qué archivos se borraron?
- En `istat`, el nombre aparece cambiado (por ejemplo `_RECIO~1.CSV`). ¿Por qué? Pista: qué hace FAT al borrar una entrada.

### 2.3 Recuperación 
```bash
icat -o 2048 usb.dd <número_de_entrada>
mkdir -p recuperados
tsk_recover -e -o 2048 usb.dd recuperados   # exporta todo: existentes y borrados
tree recuperados
```

### 2.4 Firmas de archivo: la extensión no alcanza 
```bash
file recuperados/Imagenes/*
```
- ¿Hay algún archivo cuya extensión no corresponda a su contenido real? Ábranlo con la herramienta que corresponda (`unzip -l`, `unzip -p`).

### 2.5 Metadatos EXIF 
```bash
exiftool recuperados/Imagenes/IMG_20261019_183245.jpg
exiftool -Make -Model -DateTimeOriginal -GPSPosition -c "%.5f" recuperados/Imagenes/*.jpg
```
- ¿Con qué dispositivo y cuándo se sacó la foto? ¿Dónde?
- Peguen las coordenadas en un mapa (fuera del contenedor). ¿Qué tan fuerte es esta evidencia?

### 2.6 Historial del navegador portable 
Chrome guarda las fechas como **microsegundos desde el 01/01/1601 (UTC)**.
```bash
H=recuperados/PortableApps/ChromePortable/Data/profile/Default/History
sqlite3 $H ".tables"
sqlite3 -header -column $H "
  SELECT datetime(v.visit_time/1000000 - 11644473600, 'unixepoch', '-3 hours') AS hora_AR,
         u.title, u.url
  FROM visits v JOIN urls u ON u.id = v.url
  ORDER BY v.visit_time;"
sqlite3 -header -column $H "
  SELECT datetime(start_time/1000000 - 11644473600, 'unixepoch', '-3 hours') AS hora_AR,
         target_path, tab_url
  FROM downloads;"
```
- ¿Qué servicio se usó para enviar archivos? ¿Cuándo?
- ¿De dónde salió originalmente el archivo de precios?
- ¿Qué búsquedas se hicieron el 23/10?

### 2.7 Línea de tiempo
```bash
fls -r -m / -o 2048 usb.dd > body.txt
mactime -b body.txt -d 2026-10-15..2026-10-31 2>/dev/null > timeline.csv
column -s, -t timeline.csv | less -S
```
`m`, `a`, `c` y `b` indican modificación, acceso, cambio de metadatos y creación. Armen una **cronología unificada** que combine el sistema de archivos, el historial y el EXIF.

> En FAT las fechas se guardan en hora local y el acceso solo registra el día (por eso aparece 00:00:00). FAT **no registra cuándo se borró un archivo**. ¿Cómo podrían acotar el momento del borrado con otras fuentes?

---

## Etapa 3 – Informe 

Completen `casos/informe_pericial.md` (la copia que hicieron en 1.6). Debe contener:

1. **Objeto y puntos de pericia**: qué se les pidió.
2. **Elementos recibidos e integridad**: hashes y el dictamen sobre la copia de RRHH.
3. **Metodología y herramientas**, con sus versiones (`fls -V`, `dc3dd --version`).
4. **Hallazgos**: cada uno vinculado a su fuente (número de entrada, archivo, consulta).
5. **Cronología**.
6. **Conclusiones**: respondan la pregunta del caso. Separen **lo que muestra la evidencia** de **lo que se infiere**. Por ejemplo: ¿la evidencia prueba que Pérez lo envió, o que se envió desde un navegador presente en ese pendrive?
7. **Relación con la auditoría**: ¿qué controles fallaron y qué recomendarían (DLP, bloqueo de USB, clasificación de la información, segregación de funciones)?

**Entrega**: la carpeta `casos/` comprimida **sin los archivos `.dd`** (pesan 64 MB cada uno), con `adquisicion.log`, `.historial_comandos`, `timeline.csv` y las tres plantillas completas.

---

## Para cerrar

- Usamos una imagen **ya adquirida**. En un procedimiento real, ¿qué cambia si el equipo está encendido? ¿Qué se pierde si se lo apaga? (RAM, conexiones abiertas, cifrado).
- ¿Qué parte de lo que hicieron hoy sería **reproducible** por un perito de la otra parte? ¿Qué necesitaría?
