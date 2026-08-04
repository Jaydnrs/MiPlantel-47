# Diario de progreso — MiPlantel 47

Acá iré escribiendo entradas por cada día de trabajo para ir documentando mis avances de manera ordenada.


## Entrada: 04/08/2026

### Objetivo
Configurar Spring Boot para servir el prototipo frontend (HTML/CSS/JS + SVG de prueba) ya construido, lograr que el proyecto arranque sin errores, y validar que el mapa de prueba (3 salones, 3 laboratorios, 3 oficinas) funcione de punta a punta: carga, selección con click, resaltado visual y panel de información lateral.

---

### Qué hice

**1. Configuración inicial de Spring Boot**
- Definí la configuración del proyecto en Spring Initializr: Maven, Java 21, Group `mx.cbtis47`, Artifact `miplantel-47`, Package `mx.cbtis47.miplantel`, Packaging Jar.
- Seleccioné las dependencias: Spring Web, Spring Data JPA, PostgreSQL Driver, Spring Boot DevTools, Validation.
- Consulta realizada a Claude: *"que dependencias de springboot necesito? Dame todas las necesarias para de una vez instalar todo lo necesario + como deberia de nombrar el artifact y eso"*.
- Descargué el proyecto generado y lo abrí en NetBeans (*File → Open Project*, detectó el `pom.xml` automáticamente).

**2. Ubicación de archivos existentes**
- Coloqué `index.html` en `src/main/resources/static/index.html`.
- Coloqué el SVG de prueba (9 espacios: 3 salones, 3 laboratorios, 3 oficinas) en `src/main/resources/static/assets/mapa-plantel.svg`.
- Consulta realizada: *"ahora dime todo lo que tengo que agregar al proyecto(controllers, archivos y eso), tambien dime en donde deberia de colocar todos los archivos que ya tenemos(html, svg de prueba)"*.
- Confirmé que **no se necesita ningún `@RestController`** en esta etapa: Spring Boot sirve automáticamente cualquier archivo dentro de `static/`, sin código adicional. Los controllers/modelos/repositorios quedan pendientes para cuando se migren los datos de `ESPACIOS` (hardcodeados en JS) a PostgreSQL.

**3. Primer intento de ejecución**
- Consulta realizada: *"como lo ejecuto de manera correcta? yo recordaba que se requeria un controller o un @rest"*.
- Corrí el proyecto desde NetBeans (botón Run).

---

### Qué problema encontré

**Error 1 — Fallo de arranque por `DataSource`**

Al correr el proyecto, la consola mostró:
```
Failed to configure a DataSource: 'url' attribute is not specified and no embedded datasource could be configured.
Reason: Failed to determine a suitable driver class
```
Causa: al incluir `spring-boot-starter-data-jpa` y el driver de PostgreSQL, Spring Boot intenta configurar una conexión a base de datos automáticamente al arrancar, aunque todavía no exista ninguna entidad ni repositorio (el log confirmó: *"Found 0 JPA repository interfaces"*). Como no había una URL de base de datos configurada, el arranque fallaba antes de levantar el servidor — por eso el navegador mostraba "conexión rechazada".

**Error 2 — IDE no reconoce las clases de exclusión**

Primer intento de solución: agregar `spring.autoconfigure.exclude=...` en `application.properties` con los nombres de paquete "clásicos" (`org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration` y `org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration`).

NetBeans marcó la línea con:
```
Cannot parse 'org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration' as java.lang.Class
Cannot parse 'org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration' as java.lang.Class
```
Investigué (búsqueda web: *"spring boot org.springframework.boot.jdbc.autoconfigure.DataSourceAutoConfiguration exclude package moved"*) y confirmé que el proyecto usa **Spring Boot 4.0**, versión donde se reorganizaron los paquetes de auto-configuración (de `autoconfigure.<módulo>` a `<módulo>.autoconfigure`). Corregí los nombres a `org.springframework.boot.jdbc.autoconfigure.DataSourceAutoConfiguration` y `org.springframework.boot.hibernate.autoconfigure.HibernateJpaAutoConfiguration`.

El IDE **siguió marcando el mismo error** con los nombres corregidos — el índice de clases de NetBeans no las reconocía de todas formas.

**Error 3 — 404 Whitelabel Error Page**

Tras resolver el problema de base de datos (ver solución abajo), el servidor arrancó correctamente pero al entrar a `http://localhost:8080` apareció:
```
Whitelabel Error Page
No static resource .
NoResourceFoundException: No static resource for request '/'
```
Esto indicaba que Spring Boot no encontraba ningún `index.html` en la ruta esperada.

**Error 4 — Cuadrados del SVG sin color**

Una vez cargado el mapa, los cuadrados se veían vacíos (relleno blanco, borde negro) en vez de tomar los colores definidos por tipo (`tipo-salon`, `tipo-laboratorio`, `tipo-oficina`) en el CSS.

---

### Cómo lo resolví

**Error 1 y 2 — Descartar la exclusión, usar H2 en memoria**

En vez de seguir depurando el nombre exacto del paquete de exclusión (dependiente de versión y poco confiable), cambié de estrategia: agregué la dependencia de H2 (base de datos en memoria) al `pom.xml`:
```xml
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
</dependency>
```
Quité la línea `spring.autoconfigure.exclude=...` de `application.properties`. Con H2 presente, Spring Boot configura automáticamente una base de datos en memoria sin necesidad de URL, usuario ni contraseña — el servidor arrancó sin el error de `DataSource`.

**Error 3 — Verificar ubicación de archivos y forzar rebuild**

Verifiqué que `index.html` estuviera exactamente en `src/main/resources/static/index.html` (no en una subcarpeta ni con nombre distinto), y ejecuté *Clean and Build* en NetBeans antes de volver a correr el proyecto, para forzar que Maven recopiara los recursos estáticos a `target/classes/static/`. Tras esto, `http://localhost:8080` cargó correctamente el prototipo.

**Error 4 — Eliminar el `style` inline generado por Inkscape**

Diagnostiqué que Inkscape agrega un atributo `style="fill:#ffffff;stroke:#000000;..."` directamente en cada forma del SVG, el cual tiene prioridad sobre las reglas de la hoja de estilos externa (aunque las formas ya tuvieran las clases `casilla tipo-salon`, etc.). Solución: eliminar el atributo `style` de cada forma desde el Editor XML de Inkscape (o mediante *Objeto → Relleno y borde* → "Sin pintura" en relleno y borde, seleccionando todas las formas con `Ctrl+A`, y volviendo a exportar como Plain SVG). Con el `style` inline eliminado, las clases CSS volvieron a tener control total del color.

---

### Qué aprendí
- Spring Boot sirve archivos estáticos (`static/`) sin necesidad de ningún `@RestController` — ese código solo es necesario para exponer datos (JSON) o renderizado dinámico del lado del servidor.
- Agregar `spring-boot-starter-data-jpa` + un driver de base de datos activa auto-configuración de conexión **desde el arranque**, aunque no haya entidades ni repositorios definidos todavía.
- Los nombres de paquete de las clases de auto-configuración de Spring Boot **cambian entre versiones mayores** (en este caso, Spring Boot 4.0 reorganizó `autoconfigure.<módulo>` a `<módulo>.autoconfigure`) — no se debe asumir que un nombre de clase de documentación antigua sigue siendo válido.
- En CSS, un atributo `style` inline en el propio elemento tiene más prioridad que una regla de clase en una hoja de estilos externa, sin importar el orden en que se escriban los archivos.
- Inkscape agrega estilos inline por defecto al exportar, lo cual puede interferir con un sistema de estilos basado en clases — hay que limpiarlo explícitamente antes de integrar el SVG al proyecto.

### Qué cambiaría
- Antes de intentar la exclusión de auto-configuración (que consumió tiempo en depurar nombres de paquete), debí probar directamente con H2 — es la solución más simple y menos dependiente de la versión exacta de Spring Boot.
- Revisar el `style` generado por Inkscape justo al exportar el SVG de prueba, no hasta después de haber armado toda la conexión con el proyecto — hubiera ahorrado un ciclo de depuración.

### Próximo paso
- Separar el JavaScript inline del `index.html` en archivos externos (`mapa.js`, `busqueda.js`) según la estructura de carpetas planeada.
- Probar la interfaz desde el celular (misma red WiFi, usando la IP local de la laptop) para validar el requisito de uso responsive del MVP.
- Comenzar la investigación con usuarios reales (entrevistas cortas con compañeros/profesores) usando el prototipo ya funcional como apoyo visual.
