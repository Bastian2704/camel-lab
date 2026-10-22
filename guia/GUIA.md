# Guía paso a paso – Taller 1: File Transfer con Apache Camel (Linux / fish)

Adaptación del taller (escrito para Windows) a CachyOS con shell **fish**.
Todo se hace dentro de:

```
~/Documents/Universidad/IntegracionSistemas/camel-lab/
```

> 📸 = toma una captura de pantalla en ese punto (la usaremos en el informe).
> Guarda las capturas en `camel-lab/capturas/` con el nombre indicado.

---

## Paso 0: Instalación de herramientas

### 0.1 Java

```fish
java -version
```

Debe salir 11 o superior. Tú tienes **Java 17**, que además es el mínimo que exige Camel 4. ✅

📸 `paso0-java.png`

### 0.2 Apache Camel

⚠️ **Diferencia con el documento:** desde Camel 4 la página oficial ya **no publica un `.zip` binario con carpeta `lib/`** (en https://camel.apache.org/download/ sólo está el código fuente). Por eso usaremos **Maven** solamente para descargar los `.jar` de Camel a una carpeta `lib/`. Después compilamos y ejecutamos exactamente igual que dice el taller (`javac` / `java`).

1. Instala Maven:

   ```fish
   sudo pacman -S maven
   mvn -version
   ```

2. Entra a la carpeta del laboratorio:

   ```fish
   cd ~/Documents/Universidad/IntegracionSistemas/camel-lab
   ```

3. Crea un archivo `pom.xml` (con VS Code: `code pom.xml`) con este contenido:

   ```xml
   <project xmlns="http://maven.apache.org/POM/4.0.0">
     <modelVersion>4.0.0</modelVersion>
     <groupId>tiendasol</groupId>
     <artifactId>camel-lab</artifactId>
     <version>1.0</version>

     <dependencies>
       <dependency>
         <groupId>org.apache.camel</groupId>
         <artifactId>camel-core</artifactId>
         <version>4.23.0</version>
       </dependency>
       <dependency>
         <groupId>org.apache.camel</groupId>
         <artifactId>camel-main</artifactId>
         <version>4.23.0</version>
       </dependency>
       <dependency>
         <groupId>org.slf4j</groupId>
         <artifactId>slf4j-simple</artifactId>
         <version>2.0.20</version>
       </dependency>
     </dependencies>
   </project>
   ```

   - `camel-core`: el motor de Camel + componentes básicos (`file`, `log`, lenguaje `simple`…).
   - `camel-main`: la clase `Main` que usa el taller para arrancar.
   - `slf4j-simple`: para que los mensajes `INFO` se vean en la consola.

4. Descarga los jars a `lib/`:

   ```fish
   mvn dependency:copy-dependencies -DoutputDirectory=lib
   ```

5. Verifica:

   ```fish
   ls
   ls lib
   ```

   Debes ver la carpeta `lib/` llena de archivos `camel-*.jar` (equivale al `dir` del taller).

📸 `paso0-lib.png`

### 0.3 Editor

VS Code → Extensiones → instala **"Extension Pack for Apache Camel by Red Hat"** (o "Language Support for Apache Camel").

---

## Paso 1: Estructura de carpetas

```fish
mkdir -p input output logs archived capturas
ls
```

| Carpeta    | Simula                                             |
|------------|----------------------------------------------------|
| `input`    | donde el sistema de **ventas** deja el archivo     |
| `output`   | donde el sistema de **inventario** lo recibe       |
| `logs`     | registro de la actividad                           |
| `archived` | archivos ya procesados (se usa en el Paso 6)       |

📸 `paso1-carpetas.png`

---

## Paso 2: CSV de ejemplo

fish **no** tiene heredoc (`<<EOF`), así que usamos `printf`:

```fish
printf '%s\n' 'id,producto,cantidad,precio' '1,Monitor,2,150' '2,Teclado,5,25' '3,Mouse,3,15' > input/ventas.csv
cat input/ventas.csv
```

📸 `paso2-csv.png`

---

## Paso 3: Ruta básica

Crea `FileTransferRoute.java` en `camel-lab/` (`code FileTransferRoute.java`):

```java
import org.apache.camel.builder.RouteBuilder;
import org.apache.camel.main.Main;

public class FileTransferRoute extends RouteBuilder {

    public static void main(String[] args) throws Exception {
        Main main = new Main();
        main.configure().addRoutesBuilder(new FileTransferRoute());
        main.run();
    }

    @Override
    public void configure() throws Exception {
        from("file:input?noop=true")
            .log("Procesando archivo: ${file:name}")
            .to("file:output");
    }
}
```

**Cambio respecto al documento:** en vez de `C:/Cursos/...` uso rutas **relativas** (`input`, `output`). Funcionan porque ejecutamos el programa desde `camel-lab/`.

- `from("file:input?noop=true")` → lee archivos de `input` **sin borrarlos** (`noop`).
- `.log(...)` → muestra el nombre del archivo procesado.
- `.to("file:output")` → escribe el archivo en `output`.

---

## Paso 4: Compilar y ejecutar

```fish
javac -cp "lib/*" FileTransferRoute.java
java -cp ".:lib/*" FileTransferRoute
```

> En Linux el separador del classpath es **`:`**; el `;` del documento es sólo para Windows.

> ⚠️ **No crees una carpeta llamada `camel/` dentro de `camel-lab/`.** Camel Main carga automáticamente como rutas todo lo que haya en `classpath:camel/*`, y como `.` está en el classpath, intentará cargar esos archivos como rutas. Con un `.md` adentro falla así: `Cannot find RoutesBuilderLoader in classpath supporting file extension: md`.

En la consola debe aparecer algo como:

```
[main] INFO route1 - Procesando archivo: ventas.csv
```

El programa **se queda corriendo** (vigila la carpeta `input`). Para detenerlo: **Ctrl+C**.

📸 `paso4-consola.png`

En otra terminal (o después de Ctrl+C):

```fish
cat output/ventas.csv
```

📸 `paso4-output.png`

---

## Paso 5: Transformación (mayúsculas)

Cambia el `configure()`:

```java
from("file:input?noop=true")
    .log("Procesando archivo: ${file:name}")
    .convertBodyTo(String.class)
    .transform().simple("${body.toUpperCase()}")
    .to("file:output");
```

> ⚠️ **Diferencia con el documento:** en Camel 4 el cuerpo que entrega `file:` es un objeto `GenericFile` (una referencia al archivo), no texto. Sin `.convertBodyTo(String.class)` falla con `Method with name: toUpperCase() not found on bean: GenericFile[ventas.csv]`, y como el intercambio hace rollback, Camel reintenta el archivo en cada sondeo (el error se repite en bucle).

Borra la salida anterior y vuelve a compilar y ejecutar:

```fish
rm -rf output; mkdir output
javac -cp "lib/*" FileTransferRoute.java
java -cp ".:lib/*" FileTransferRoute
```

Ctrl+C y revisa:

```fish
cat output/ventas.csv
```

Resultado esperado:

```
ID,PRODUCTO,CANTIDAD,PRECIO
1,MONITOR,2,150
2,TECLADO,5,25
3,MOUSE,3,15
```

> Nota para el informe: `toUpperCase()` se aplica a **todo** el contenido, por eso la cabecera también queda en mayúsculas.

📸 `paso5-consola.png`, `paso5-output.png`

---

## Paso 6: Ruta extra y filtro

Reemplaza el `configure()` completo:

```java
@Override
public void configure() throws Exception {
    // Ruta 1: ventas -> inventario (solo .csv, transformado)
    from("file:input?noop=true")
        .filter(header("CamelFileName").endsWith(".csv"))
        .log("Procesando archivo: ${file:name}")
        .convertBodyTo(String.class)
        .transform().simple("${body.toUpperCase()}")
        .to("file:output");

    // Ruta 2: copia lo procesado a archived con fecha y hora
    from("file:output?noop=true")
        .log("Archivando ${file:name} el ${date:now:yyyy-MM-dd HH:mm:ss}")
        .to("file:archived");
}
```

- `.filter(...)` → sólo deja pasar archivos que terminan en `.csv`.
- La 2ª ruta lee `output` y copia a `archived` registrando **fecha y hora**. Uso `noop=true` para que `output` conserve el archivo transformado (es un entregable).

Prueba el filtro creando un archivo que **no** es CSV:

```fish
echo "no soy csv" > input/notas.txt
rm -rf output archived; mkdir output archived
javac -cp "lib/*" FileTransferRoute.java
java -cp ".:lib/*" FileTransferRoute 2>&1 | tee logs/camel.log
```

`tee` muestra los logs en pantalla **y** los guarda en `logs/camel.log`. El `2>&1` es necesario porque `slf4j-simple` escribe los logs por **stderr**; sin él, `camel.log` queda vacío.

Debes ver dos líneas de log (procesando + archivando) y **ninguna** para `notas.txt`. **Espera unos 3 segundos** hasta que salga `Archivando ventas.csv el ...` (la Ruta 2 sólo actúa cuando revisa `output`, después de que la Ruta 1 escribió ahí). Luego Ctrl+C y revisa:

```fish
ls input output archived
cat logs/camel.log
```

`notas.txt` debe estar sólo en `input`.

📸 `paso6-consola.png`, `paso6-carpetas.png`

---

## Paso 7: Validar

```fish
cat input/ventas.csv
cat output/ventas.csv
diff input/ventas.csv output/ventas.csv
```

`diff` muestra exactamente qué cambió (las mayúsculas).

📸 `paso7-diff.png`

Preguntas de reflexión (se responden en el informe):
1. ¿Qué representa el patrón "File Transfer"?
2. ¿Qué limitaciones tiene este enfoque?
3. ¿En qué escenarios reales sería útil?

---

## Entregable: repositorio

```fish
printf '%s\n' '*.class' 'lib/' 'target/' > .gitignore
git init
git add .
git commit -m "Taller 1: File Transfer con Apache Camel"
gh repo create camel-lab --public --source=. --push
```

(Si no tienes `gh`: crea el repo vacío en github.com y sigue sus instrucciones de `git remote add` / `git push`.)
`lib/` no se sube: se regenera con `mvn dependency:copy-dependencies -DoutputDirectory=lib`.

---

## Problemas encontrados y soluciones

| # | Paso | Síntoma | Causa | Solución |
|---|------|---------|-------|----------|
| 1 | 0.2 | No existe `.zip` binario con `lib/` | Camel 4 sólo publica el código fuente | `pom.xml` mínimo + `mvn dependency:copy-dependencies -DoutputDirectory=lib` |
| 2 | 4 | `Cannot find RoutesBuilderLoader in classpath supporting file extension: md` | Había una carpeta `camel/` con `GUIA.md`; Camel Main carga `classpath:camel/*` como rutas y `.` está en el classpath | Mover la guía a otra carpeta (`guia/`) |
| 3 | 5 | `Method with name: toUpperCase() not found on bean: GenericFile[ventas.csv]`, repetido en bucle | En Camel 4 el body de `file:` es un `GenericFile`, no texto; al fallar hace rollback y lo reintenta en cada sondeo | `.convertBodyTo(String.class)` antes del `.transform()` |
| 4 | 5–6 | `fish: No matches for wildcard 'archived/*'` y no se borra nada | En fish, un comodín sin coincidencias cancela todo el comando | `rm -rf output archived; mkdir output archived` |
| 5 | 6 | `logs/camel.log` vacío | Los logs salen por stderr y `\|` sólo pasa stdout | `java ... 2>&1 \| tee logs/camel.log` |
| 6 | 6 | `archived/` vacío | Se detuvo con Ctrl+C antes de que la Ruta 2 sondeara `output` | Esperar al log `Archivando ...` antes de Ctrl+C |

---

## Qué guardar para el informe

- Capturas en `capturas/`: `paso0-java`, `paso0-lib`, `paso1-carpetas`, `paso2-csv`, `paso4-consola`, `paso4-output`, `paso5-consola`, `paso5-output`, `paso6-consola`, `paso6-carpetas`, `paso7-diff`.
- `logs/camel.log`.
- Cualquier error que te salga (cópialo tal cual) para documentarlo.
- El link del repositorio.

Cuando termines, avísame y armo el `INFORME.md` con todo esto.
