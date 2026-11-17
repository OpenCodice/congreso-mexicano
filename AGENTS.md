# congreso-mexicano — guía para agentes

Quién legisla y cómo vota: legisladores, sus mandatos, los expedientes del
proceso legislativo, las votaciones con el voto de cada quien y los decretos
del DOF que enlazan todo eso con los repos de leyes. Es un repo de datos: aquí
no hay código. Todo lo genera
[`congreso-extractor`](https://github.com/OpenCodice/congreso-extractor) a
partir del SIL de Gobernación (sil.gobernacion.gob.mx), del DOF (SIDOF), del
sitio de votaciones del Senado y de la Gaceta Parlamentaria de la Cámara de
Diputados.

**No edites a mano nada de este repo.** Si un dato está mal, el arreglo va en
el extractor.

## Modelo

[Popolo](https://www.popoloproject.com/specs/) para personas, organizaciones,
mandatos y votaciones, con dos extensiones propias: el expediente y el decreto.
Un registro por archivo JSON.

| Carpeta | Qué es | Popolo |
|---|---|---|
| `organizaciones/` | Cámaras, Comisión Permanente, partidos y los 32 congresos locales (`congreso-<estado>`, con la clave de `leyes-estatales`). | Organization |
| `personas/{id}.json` | Una persona, con las `Referencia` del SIL de cada mandato en `identifiers`. | Person |
| `mandatos/{legislatura}/{camara}/{referencia}.json` | Un mandato: cámara, legislatura, partido, calidad (propietario o suplente), principio de elección, entidad, distrito o circunscripción, licencias y comisiones. | Membership |
| `expedientes/sil-{seguimiento}.json` | Un expediente del SIL: sus asuntos (iniciativas, dictámenes, minuta, declaratorias), su cronología y sus votaciones. | extensión |
| `votaciones/{legislatura}/{camara}/sil-{asunto}-{n}.json` | Una votación nominal según el SIL: sus conteos y, si el SIL la desglosa, el voto de cada legislador. | VoteEvent + Vote |
| `votaciones/{legislatura}/senado/senado-{clave}.json` | Una votación nominal según el sitio del Senado, con el voto de cada senador (también las de lo particular). | VoteEvent + Vote |
| `votaciones/{legislatura}/diputados/gaceta-{tabla}.json` | Una votación nominal según la Gaceta Parlamentaria, con el voto de cada diputado (también las de lo particular). | VoteEvent + Vote |
| `decretos/{año}/dof-{codigo}.json` | Un decreto del DOF: qué documentos reforma (con el commit que lo trajo a cada repo de leyes), su expediente y, en reformas constitucionales, los congresos locales que la aprobaron. | extensión |
| `fuentes/` | Copias de las páginas oficiales en que se basan las votaciones y los expedientes, nombradas por su sha256. | — |

Opciones de voto: `yes`, `no`, `abstain`, `absent` y `not voting` (Popolo; la
Gaceta lo llama "quórum": pasó lista y no votó).

Una misma votación puede estar en dos fuentes (el SIL y el Senado o la
Gaceta). Cada una queda en su propio registro; el de la cámara apunta al del
SIL con `votacion_sil`, y lo que difiere entre las dos (conteos o el voto de
alguien) queda en `discrepancias`. Ninguna corrige a la otra.

## Cómo se enlaza

```
persona ─ mandato ─ voto ─ votación ─ expediente ─ decreto ─ (Documento, Publicacion) ─ commit del repo de leyes
```

- **Voto → mandato**: por nombre, dentro de la cámara y la legislatura (las
  filas de votación no traen la `Referencia` del SIL). En el SIL, si un nombre
  no aparece o aparece dos veces, la votación no se publica. El Senado y la
  Gaceta escriben los nombres de otra forma (orden, abreviaturas, variantes
  ortográficas); cada voto ligado así dice cómo en `enlace`, y un nombre que no
  da un único mandato no se atribuye: queda en `votos_faltantes` con
  `nombres_fuente`.
- **Votación de la cámara → votación del SIL**: misma fecha, la denominación
  del SIL contenida en el título de la fuente y el mismo número de votaciones
  del asunto en ambas, emparejadas en orden. Si no es inequívoco, solo se liga
  al expediente, o a nada.
- **Persona**: el SIL da una `Referencia` por mandato, no por persona. La
  persona es el nombre completo normalizado; dos mandatos de la misma cámara y
  legislatura con el mismo nombre detienen el padrón.
- **Partido**: el del mandato es el que el SIL muestra hoy; el de cada voto
  (`grupo_fuente`) es el que la lista de votación muestra para esa votación.
- **Decreto → documento**: el título del decreto en el sumario del DOF del día
  de la `Publicacion` nombra al documento.
- **Decreto → expediente**: el SIL dice "Publicado en DOF" en esa fecha y la
  denominación nombra al mismo documento. Cada enlace lleva su `criterio`; si
  nada distingue entre varios, quedan en `candidatos_expediente` sin elegir.
  Lo que el código no liga lo puede resolver un agente: propone el expediente
  con citas textuales del decreto y del expediente, un programa comprueba que
  cada cita esté en su fuente y otro agente confirma. Esos enlaces llevan
  `resolucion` (el lote del registro en `congreso-extractor/resoluciones/`)
  y `evidencia` (las citas); ante la duda, el decreto se queda sin expediente.

## Lo que no está (y por qué)

- **El voto individual de la segunda votación nominal, en el SIL**: el SIL
  solo desglosa una votación por asunto; las demás quedan con conteos y
  `votos_publicados: false`. El voto de cada legislador está en el registro
  `senado-…` o `gaceta-…` que apunta a ella.
- **Legisladores que el resumen cuenta y la lista no trae**: en algunas
  votaciones del Senado el resumen por grupo del SIL cuenta un senador más que
  su lista nominal (el sitio del Senado sí lo nombra). Los votos listados se publican y lo que falta queda en
  `votos_faltantes` (grupo, sentido y cuántos), sin adivinar quién es. Si la
  lista trae de más o el resumen no suma, la votación no se publica.
- **El sentido del voto de cada congreso local**: el SIL solo lista los que
  remitieron voto aprobatorio.
- **Votaciones económicas**: no registran nombres; están como trámites en el
  expediente.
- **Cambios de bancada**: el SIL solo muestra el partido actual del mandato.
