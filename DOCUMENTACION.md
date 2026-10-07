# Documentación técnica

## 1. Objetivo

Este proyecto modifica el thumbnailer `Comic Books` de KDE `kio-extras` para ampliar los tipos de archivos que pueden generar miniaturas en Dolphin.

La modificación permite utilizar archivos de imagen contenidos dentro de:

* ZIP
* RAR
* 7z
* TAR

además de los formatos de cómic reconocidos originalmente por KDE.

El objetivo principal es que Dolphin pueda mostrar una miniatura de archivos `.zip`, `.rar` y `.7z` que contienen imágenes, incluso cuando dichos archivos no utilizan una extensión específica de cómic como `.cbz`, `.cbr` o `.cb7`.

---

# 2. Arquitectura

La arquitectura resultante es:

```text
                         Dolphin
                            │
                            ▼
                    KDE Thumbnail Service
                            │
                            ▼
                 comicbookthumbnail.so
                            │
                            ▼
                      ComicCreator
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
          ▼                 ▼                 ▼
       ZIP / 7z            RAR              TAR
          │                 │                 │
          ▼                 ▼                 ▼
         7z            unrar / rar          KTar
          │                 │                 │
          └─────────────────┼─────────────────┘
                            ▼
                          QImage
                            │
                            ▼
                      ThumbnailResult
                            │
                            ▼
                          Dolphin
```

El plugin modificado sigue siendo un thumbnailer de KDE. No se reemplaza el servicio general de miniaturas.

---

# 3. Archivos modificados

El patch modifica exclusivamente:

```text
thumbnail/comiccreator.cpp
thumbnail/comiccreator.h
thumbnail/comicbookthumbnail.json
```

No se modifica:

```text
thumbnail/thumbnail.cpp
```

ni:

```text
kio/
```

ni el servicio global:

```text
thumbnail.so
```

La modificación queda aislada dentro del plugin:

```text
comicbookthumbnail.so
```

---

# 4. Registro de tipos MIME

El archivo:

```text
thumbnail/comicbookthumbnail.json
```

originalmente contiene los tipos MIME relacionados con archivos de cómic.

El patch añade:

```text
application/zip
application/vnd.rar
application/x-7z-compressed
```

Por lo tanto, el plugin puede recibir solicitudes de thumbnail para archivos ZIP, RAR y 7z normales.

Los tipos de cómic existentes se mantienen.

La modificación no cambia el sistema MIME de Linux. Solamente amplía los tipos MIME declarados por el plugin.

---

# 5. Flujo de `ComicCreator::create()`

`ComicCreator::create()` es el punto de entrada principal del thumbnailer.

El flujo general es:

```text
ThumbnailRequest
       │
       ▼
Obtener ruta local
       │
       ▼
¿Ruta válida?
       │
       ▼
Determinar MIME
       │
       ├── ZIP / CBZ
       │
       ├── TAR / CBT
       │
       ├── 7z / CB7
       │
       └── RAR / CBR
       │
       ▼
Extraer imagen
       │
       ▼
QImage
       │
       ▼
ThumbnailResult
```

Si la ruta local está vacía, el thumbnailer devuelve:

```cpp
KIO::ThumbnailResult::fail();
```

Si el formato no está soportado, tampoco se intenta procesar el archivo.

Si la extracción no produce una imagen válida, la solicitud falla.

---

# 6. ZIP y CBZ

ZIP y CBZ se procesan mediante el ejecutable externo:

```text
7z
```

La decisión de utilizar `7z` en lugar de `KZip` es intencional.

El flujo es:

```text
ZIP / CBZ
    │
    ▼
7z l -slt archive
    │
    ▼
Obtener lista de archivos
    │
    ▼
filterImages()
    │
    ▼
Seleccionar imagen
    │
    ▼
7z e -so archive image
    │
    ▼
QImage::loadFromData()
```

Esto permite utilizar el mismo mecanismo de extracción para ZIP y 7z.

---

# 7. 7z y CB7

Los archivos:

```text
.7z
.cb7
```

utilizan el mismo backend externo:

```text
7z
```

Primero se ejecuta:

```text
7z l -slt archive
```

La opción `-slt` proporciona información técnica estructurada sobre los elementos del archivo.

La implementación busca las entradas:

```text
Path = ...
```

y construye la lista de archivos contenidos en el archivo.

Posteriormente se filtran las entradas para obtener únicamente imágenes compatibles.

---

# 8. Extracción mediante `7z`

La función:

```cpp
QImage ComicCreator::extractSevenZipImage(
    const QString &path,
    const QString &imagePath)
```

es responsable de obtener una imagen concreta.

El comando utilizado conceptualmente es:

```text
7z e -so archive image
```

La opción:

```text
-so
```

envía el contenido extraído a `stdout`.

Esto permite evitar la creación permanente de un archivo temporal.

El flujo es:

```text
7z
 │
 ├── stdout
 │
 ▼
QByteArray
 │
 ▼
QImage::loadFromData()
 │
 ▼
QImage
```

Si `7z` termina correctamente y los datos corresponden a una imagen válida, se devuelve el `QImage`.

---

# 9. Por qué no se utiliza escape manual de rutas

Las rutas de archivos pueden contener:

* espacios
* paréntesis
* caracteres especiales

No se realiza una conversión manual como:

```text
\40
```

ni:

```text
%20
```

La razón es que `QProcess` recibe los argumentos como elementos separados.

Por ejemplo:

```cpp
QStringList args;
args << QStringLiteral("e");
args << QStringLiteral("-so");
args << path;
args << imagePath;
```

El espacio contenido en `imagePath` no debe convertirse manualmente.

`QProcess` entrega el argumento al proceso hijo como una unidad.

---

# 10. Archivos RAR

El soporte RAR utiliza el backend externo existente.

La búsqueda del ejecutable se realiza mediante:

```text
unrar
unrar-nonfree
rar
```

La implementación intenta encontrar un ejecutable disponible y posteriormente valida que corresponda a una herramienta RAR/UNRAR válida.

La lista de archivos se obtiene mediante:

```text
unrar vb archive
```

Posteriormente:

```text
filterImages()
```

selecciona las entradas que corresponden a imágenes.

La imagen seleccionada se extrae a un directorio temporal.

---

# 11. Selección de imágenes

La función:

```cpp
filterImages(QStringList &entries)
```

se utiliza para filtrar las entradas del archivo.

Actualmente se consideran extensiones como:

```text
avif
bmp
gif
heif
jpg
jpeg
jxl
png
webp
```

También se excluyen entradas que pertenecen a:

```text
__MACOSX
.DS_Store
```

El objetivo es evitar seleccionar metadatos o archivos auxiliares generados por macOS.

---

# 12. Preservación del nombre de entrada

El nombre original de la entrada se conserva.

Por ejemplo:

```text
Cthulhu@vse_pro_3D/Cthulhu.jpg
```

se mantiene como ruta de entrada al proceso `7z`.

No se realiza una sustitución arbitraria de:

```text
/
```

ni de:

```text
@
```

ni de espacios.

Esto es importante porque la ruta utilizada por `7z` debe coincidir con la ruta existente dentro del archivo.

---

# 13. Archivos TAR

Los archivos TAR continúan utilizando las clases de KDE.

El flujo es:

```text
TAR
 │
 ▼
KTar
 │
 ▼
KArchiveDirectory
 │
 ▼
getArchiveFileList()
 │
 ▼
filterImages()
 │
 ▼
QImage
```

No se utiliza el ejecutable externo `7z` para TAR.

Esto permite conservar el mecanismo original de KDE para este tipo de archivo.

---

# 14. `getArchiveFileList()`

La función recorre recursivamente la estructura de directorios del archivo.

Por ejemplo:

```text
folder/
    subfolder/
        cover.jpg
```

se convierte en una entrada equivalente a:

```text
folder/subfolder/cover.jpg
```

La ruta completa se construye durante la recursión.

También se realizan comprobaciones para evitar acceder a directorios o entradas nulas.

---

# 15. Archivos solid

Los archivos 7z pueden utilizar compresión solid.

Un archivo solid puede contener múltiples archivos dentro de un mismo bloque de compresión.

Por ejemplo:

```text
Archive.7z
   │
   └── Solid block
       ├── image01.jpg
       ├── image02.jpg
       ├── image03.jpg
       └── ...
```

En este caso, solicitar una imagen situada posteriormente dentro del bloque puede requerir que el descompresor procese una parte importante del bloque antes de poder entregar la imagen.

Por esta razón:

```text
7z l -slt
```

puede ser muy rápido, mientras que:

```text
7z e -so ...
```

puede tardar considerablemente más.

Esto no significa necesariamente que exista un problema en Dolphin o en el plugin.

Es una característica del formato de compresión solid.

---

# 16. Tiempo de espera de `7z`

La extracción de imágenes mediante `QProcess` utiliza espera sin límite artificial:

```cpp
waitForFinished(-1)
```

Esto evita que una extracción válida sea cancelada simplemente porque un archivo grande necesita más tiempo de procesamiento.

Esto es especialmente relevante para archivos 7z solid.

La contrapartida es que un archivo extremadamente grande o problemático puede tardar bastante tiempo en procesarse.

---

# 17. Manejo de errores

El thumbnailer comprueba:

* existencia de una ruta local válida;
* inicio correcto del proceso;
* finalización normal del proceso;
* código de salida del proceso;
* existencia de datos de salida;
* posibilidad de cargar los datos como `QImage`.

Si cualquiera de estas etapas falla, se devuelve:

```cpp
KIO::ThumbnailResult::fail();
```

Esto permite que Dolphin continúe funcionando aunque un archivo concreto no pueda producir una miniatura.

---

# 18. Herramientas externas

## 7z

El backend ZIP/7z requiere que el ejecutable `7z` esté disponible.

La implementación utiliza:

```text
/usr/bin/7z
```

Por tanto, el sistema debe proporcionar el ejecutable en esa ubicación.

Puede verificarse con:

```bash
which 7z
```

y:

```bash
7z
```

---

## RAR

El backend RAR busca:

```text
unrar
unrar-nonfree
rar
```

La disponibilidad puede comprobarse mediante:

```bash
which unrar
```

o:

```bash
which unrar-nonfree
```

o:

```bash
which rar
```

---

# 19. Diagnóstico

Si Dolphin no muestra miniaturas, primero comprobar:

### 19.1 Plugin instalado

```bash
ls -l \
/usr/lib/x86_64-linux-gnu/qt6/plugins/kf6/thumbcreator/comicbookthumbnail.so
```

### 19.2 Ejecutable 7z

```bash
which 7z
7z
```

### 19.3 MIME

Para un archivo:

```bash
file archivo.7z
```

o:

```bash
xdg-mime query filetype archivo.7z
```

### 19.4 Extracción directa

Antes de investigar KDE, comprobar que `7z` puede extraer la imagen:

```bash
7z l -slt archivo.7z
```

Después:

```bash
7z e -so \
    archivo.7z \
    "ruta/interna/imagen.jpg" \
    > /tmp/test-image.jpg
```

Y comprobar:

```bash
file /tmp/test-image.jpg
```

Si la extracción directa funciona pero Dolphin no genera la miniatura, el problema probablemente se encuentra en el flujo del thumbnailer y no en el archivo 7z.

---

# 20. Archivos de prueba

Durante el desarrollo se utilizó un archivo 7z de prueba:

```text
Origin.7z
```

El archivo era un archivo 7z solid con un único bloque de compresión.

Su tamaño físico era aproximadamente:

```text
840 MB
```

y contenía aproximadamente:

```text
1.73 GB
```

de datos sin comprimir.

La extracción directa de una imagen interna demostró que `7z e -so` podía recuperar correctamente la imagen.

Este tipo de archivo es útil para comprobar el comportamiento con archivos solid grandes.

---

# 21. Consideraciones sobre rendimiento

El thumbnailer no extrae deliberadamente todo el archivo a disco para generar una miniatura.

El objetivo es:

```text
listar
  ↓
identificar imagen
  ↓
extraer una imagen
  ↓
cargar QImage
```

Sin embargo, esto no garantiza que el descompresor procese únicamente los bytes correspondientes a esa imagen.

En archivos solid, el algoritmo de descompresión puede necesitar procesar datos anteriores dentro del mismo bloque.

Por tanto:

```text
"extraer solamente una imagen"
```

no significa necesariamente:

```text
"descomprimir solamente una pequeña fracción del archivo".
```

---

# 22. Diseño actual frente a mejoras futuras

El código actual procesa cada solicitud de thumbnail de forma independiente.

Una posible mejora futura sería incorporar una cola de trabajos:

```text
Dolphin
   │
   ├── archive A
   ├── archive A
   ├── archive B
   └── archive A
          │
          ▼
    ArchiveJobQueue
          │
          ├── A → un único trabajo
          └── B → un trabajo
```

Esto permitiría deduplicar solicitudes simultáneas para el mismo archivo.

Otra posible mejora sería mantener una caché temporal de resultados de extracción.

Estas mejoras no forman parte de la versión actual.

---

# 23. Instalación recomendada

La instalación debe limitarse al plugin:

```text
comicbookthumbnail.so
```

No es necesario reemplazar:

```text
thumbnail.so
```

Esto reduce el impacto de la modificación y facilita la restauración.

La estrategia recomendada es:

```text
compilar
   │
   ▼
hacer backup
   │
   ▼
reemplazar comicbookthumbnail.so
   │
   ▼
reiniciar Dolphin
   │
   ▼
probar
```

---

# 24. Desinstalación

La desinstalación consiste en restaurar la copia original:

```bash
sudo cp \
    /usr/lib/x86_64-linux-gnu/qt6/plugins/kf6/thumbcreator/comicbookthumbnail.so.backup \
    /usr/lib/x86_64-linux-gnu/qt6/plugins/kf6/thumbcreator/comicbookthumbnail.so
```

Después se debe reiniciar Dolphin.

El código fuente del sistema no necesita permanecer modificado después de instalar el plugin compilado.

---

# 25. Compatibilidad con futuras versiones de KDE

El patch está asociado específicamente a:

```text
kio-extras 26.04.0
```

No debe asumirse que puede aplicarse directamente a una versión posterior.

KDE puede cambiar:

* `ComicCreator`
* APIs de KIO
* APIs de KArchive
* estructura de `comicbookthumbnail.json`
* sistema de plugins
* rutas de instalación
* dependencias de Qt/KDE

Por ello, una actualización de `kio-extras` debe considerarse una nueva versión del patch.

---

# 26. Principio de mantenimiento

Cuando se modifique el patch, se debe mantener la siguiente regla:

> Modificar solamente lo necesario para el soporte de archivos de archivo genéricos y conservar el comportamiento existente de KDE siempre que sea posible.

En particular, no se debe modificar:

```text
thumbnail.so
```

ni introducir cambios globales en el sistema de thumbnails para resolver problemas específicos del plugin.

---

# 27. Resumen técnico

La modificación implementa cuatro rutas principales:

```text
ZIP / CBZ
    ↓
7z
    ↓
imagen
    ↓
QImage
```

```text
7z / CB7
    ↓
7z
    ↓
imagen
    ↓
QImage
```

```text
RAR / CBR
    ↓
unrar / rar
    ↓
imagen
    ↓
QImage
```

```text
TAR / CBT
    ↓
KTar
    ↓
imagen
    ↓
QImage
```

La integración se realiza dentro de `comicbookthumbnail.so`, manteniendo intacto el servicio general de thumbnails de KDE.
