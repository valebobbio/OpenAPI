# Contratos OpenAPI — PPC Crianza

Este repositorio contiene la definición de la API en formato OpenAPI 3.0,
organizada como contrato-first: se define primero la forma de la
comunicación entre los clientes (app móvil, CMS web) y el backend, y a
partir de ahí se implementa.

## Estructura del repositorio

```
├── common.yaml           # Definiciones compartidas entre dominios
├── contenido.yaml        # Módulos, secciones, recursos multimedia
├── encuestas.yaml        # Encuestas, preguntas, respuestas
├── usuarios.yaml         # Administradores, referentes parentales, niños a cargo
└── notificaciones.yaml   # Notificaciones
```

Cada archivo de dominio corresponde a un bounded context del modelo de
dominio del producto. `common.yaml` no es un contrato en sí mismo — no se
sirve solo — sino el lugar donde viven las definiciones que **se repiten en
más de un archivo de dominio**: los enums (`Genero`, `Relacion`,
`EstadoPublicacion`, `TipoRecurso`, `TipoAccesibilidad`), el formato
estándar de `Error`, la paginación, las respuestas HTTP reutilizables
(`NoEncontrado`, `SolicitudInvalida`, `NoAutorizado`, `Prohibido`) y el
esquema de autenticación. Cada archivo de dominio lo referencia con `$ref:
'common.yaml#/components/...'` en vez de redefinir esos tipos.

## Versionado del contrato

Cada archivo de dominio lleva su propia versión semántica en `info.version`
(`"1.0.0"`). La regla para actualizarla:

- **Cambio menor** (agregar un endpoint nuevo, un campo opcional, un valor
  nuevo a un enum): sube el segundo número (`1.0.0` → `1.1.0`).
- **Cambio que rompe compatibilidad** (quitar/renombrar un campo, cambiar
  un tipo de dato, quitar un endpoint): sube el primer número (`1.0.0` →
  `2.0.0`).
- El historial de qué cambió en cada versión queda en el mensaje del commit
  o pull request correspondiente; si el volumen de cambios crece, se puede
  sumar un `CHANGELOG.md` por archivo.

## Ejemplos de respuestas y errores

Todo endpoint debe documentar al menos un ejemplo de respuesta exitosa y un
ejemplo por cada código de error que declare — no alcanza con el `schema`
solo. Esto se agrega con la clave `examples` (no confundir con `example`
en singular, que es solo para un campo suelto):

```yaml
responses:
  '200':
    description: Módulo encontrado.
    content:
      application/json:
        schema:
          $ref: '#/components/schemas/Modulo'
        examples:
          publicado:
            summary: Módulo ya publicado
            value:
              id: "3fa85f64-5717-4562-b3fc-2c963f66afa6"
              titulo: "Lactancia materna: primeros pasos"
              estadoPublicacion: PUBLICADO
              fechaCreacion: "2026-03-10"
  '404':
    description: No existe un módulo con ese id.
    content:
      application/json:
        schema:
          $ref: 'common.yaml#/components/schemas/Error'
        examples:
          noEncontrado:
            value:
              codigo: MODULO_NO_ENCONTRADO
              mensaje: No existe un módulo con el id solicitado.
```

Cada archivo de dominio debe tener al menos un ejemplo de error usando el
`Error` de `common.yaml`, con un `codigo` específico de ese dominio (ej.
`MODULO_NO_ENCONTRADO`, `ENCUESTA_NO_ENCONTRADA`) — así queda claro y
consistente para quien consuma la API desde ambos lados.

## Tags de audiencia (móvil vs. CMS, sin separar archivos)

Los contratos no se separan en archivos distintos para móvil y CMS —
conviven en el mismo archivo de dominio — pero cada operación indica a qué
cliente corresponde mediante un segundo tag, además del tag de dominio:

```yaml
get:
  operationId: listarModulos
  tags: [Modulos, cms]      # el CMS ve/gestiona todos los estados

get:
  operationId: listarModulosPublicados
  tags: [Modulos, mobile]   # la app móvil solo ve los publicados
```

Los tags disponibles son `cms` y `mobile`. Una operación puede tener ambos
si la usan los dos clientes (ej. `GET /modulos/{id}` para ver el detalle).
Herramientas como Redoc o Swagger UI permiten filtrar la documentación por
tag, así que cada equipo puede mirar solo "su" subconjunto de endpoints sin
que exista ningún archivo aparte que mantener.

## Cómo visualizar los contratos (offline, sin subir nada a la nube)

Como los archivos se referencian entre sí, herramientas basadas solo en el
navegador (como el editor online de Swagger) no pueden resolver las
referencias a `common.yaml` por sí solas. Para ver la documentación
navegable de forma local, sin depender de ninguna herramienta externa:

```bash
npx @redocly/cli build-docs contenido.yaml -o contenido-docs.html
```

Esto genera un `.html` autocontenido (con las referencias a `common.yaml`
ya resueltas) que se abre con doble clic en cualquier navegador, sin
servidor ni conexión a internet. Repetir cambiando `contenido.yaml` por el
archivo de dominio que se quiera ver.

## Cómo subir un contrato a SwaggerHub

SwaggerHub no tiene acceso al resto de los archivos del repositorio, así
que no puede resolver un `$ref` a `common.yaml` directamente — hay que
subir una versión "empaquetada" que ya tenga ese contenido incluido.

**Recomendación práctica para el equipo:** sigan editando los archivos
multi-archivo (`contenido.yaml` + `common.yaml`) como fuente de verdad en
su repo de Git — es más mantenible y evita duplicar los enums en cada
archivo. Antes de subir a SwaggerHub, corren el comando de bundle
(`npx @redocly/cli bundle contenido.yaml -o contenido.bundled.yaml`) por
cada archivo de dominio. Es un paso de un segundo que pueden automatizar
con un script si van a hacerlo seguido.

El archivo `*.bundled.yaml` resultante **no se edita nunca a mano** — se
regenera cada vez que cambie el archivo de dominio o `common.yaml`, y es
el único que se sube a SwaggerHub.

## Convenciones generales

- IDs como `string` en formato `uuid`, nunca enteros autoincrementales
  expuestos.
- Campos de dominio en español, `camelCase`, igual que en el modelo de
  dominio.
- Toda relación con una entidad de otro bounded context se representa como
  un campo `xId` (el id, como string), nunca como el objeto completo
  embebido.
- Todo listado pagina con `pagina` / `tamañoPagina` y devuelve el wrapper
  de `common.yaml#/components/schemas/PaginacionMeta`.
- Todo error usa `common.yaml#/components/schemas/Error`.