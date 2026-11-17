# congreso-mexicano — guía para agentes

Quién legisla y cómo vota: legisladores, sus mandatos, todo lo que presentan
(iniciativas, proposiciones) y en qué quedó, los expedientes del proceso
legislativo, las votaciones con el voto de cada quien y los decretos
del DOF que enlazan todo eso con los repos de leyes. Es un repo de datos: aquí
no hay código. Todo lo genera
[`congreso-extractor`](https://github.com/OpenCodice/congreso-extractor) a
partir del SIL de Gobernación (sil.gobernacion.gob.mx), del DOF (SIDOF), del
sitio de votaciones del Senado y de la Gaceta Parlamentaria de la Cámara de
Diputados.

**No edites a mano nada de este repo.** Si un dato está mal, el arreglo va en
el extractor.

Este repo no tiene workflows: solo datos. A `main` se llega solo por PR (con
historial lineal, también para los administradores). Los PR de datos los
abre la actualización semanal del extractor y los valida y fusiona su
`revisar-pr.yml`, que corre `congreso validar` y revisa que cada commit lleve
un `Tipo` y solo toque sus carpetas.

## Modelo

[Popolo](https://www.popoloproject.com/specs/) para personas, organizaciones,
mandatos y votaciones, con extensiones propias: el asunto (iniciativa,
proposición, dictamen en sentido negativo), el expediente y el decreto.
Un registro por archivo JSON.

| Carpeta | Qué es | Popolo |
|---|---|---|
| `organizaciones/` | Cámaras, Comisión Permanente, partidos y los 32 congresos locales (`congreso-<estado>`, con la clave de `leyes-estatales`). | Organization |
| `personas/{id}.json` | Una persona, con las `Referencia` del SIL de cada mandato en `identifiers`. | Person |
| `mandatos/{legislatura}/{camara}/{referencia}.json` | Un mandato: cámara, legislatura, partido, calidad (propietario o suplente), principio de elección, entidad, distrito o circunscripción, licencias y comisiones. | Membership |
| `iniciativas/{legislatura}/sil-{clave}.json` | Una iniciativa: quién la presentó (ligado a su mandato por la `Referencia` del SIL, o el grupo, congreso local o Ejecutivo), a qué comisiones se turnó, su estatus con fecha (`estatus.resultado`: pendiente, aprobado, desechado, retirado) y su expediente si llegó a votarse. | extensión |
| `proposiciones/{legislatura}/sil-{clave}.json` | Una proposición con punto de acuerdo, con los mismos campos. | extensión |
| `dictamenes-negativos/{legislatura}/sil-{clave}.json` | Un dictamen en sentido negativo (desecha iniciativas o proposiciones), con las comisiones que lo presentan. | extensión |
| `expedientes/sil-{seguimiento}.json` | Un expediente del SIL: sus asuntos (iniciativas, dictámenes, minuta, declaratorias), su cronología y sus votaciones. | extensión |
| `votaciones/{legislatura}/{camara}/sil-{asunto}-{n}.json` | Una votación nominal según el SIL: sus conteos y, si el SIL la desglosa, el voto de cada legislador. | VoteEvent + Vote |
| `votaciones/{legislatura}/senado/senado-{clave}.json` | Una votación nominal según el sitio del Senado, con el voto de cada senador (también las de lo particular). | VoteEvent + Vote |
| `votaciones/{legislatura}/diputados/gaceta-{tabla}.json` | Una votación nominal según la Gaceta Parlamentaria, con el voto de cada diputado (también las de lo particular). | VoteEvent + Vote |
| `decretos/{año}/dof-{codigo}.json` | Un decreto del DOF: qué documentos reforma (con el commit que lo trajo a cada repo de leyes), su expediente y, en reformas constitucionales, los congresos locales que la aprobaron. | extensión |
| `fuentes/` | Copias de las páginas oficiales en que se basan los datos, nombradas por el sha256 de lo descargado (`.html.gz`: comprimida; la huella es del original). | — |

Opciones de voto: `yes`, `no`, `abstain`, `absent` y `not voting` (Popolo; la
Gaceta lo llama "quórum": pasó lista y no votó).

Una misma votación puede estar en dos fuentes (el SIL y el Senado o la
Gaceta). Cada una queda en su propio registro; el de la cámara apunta al del
SIL con `votacion_sil`, y lo que difiere entre las dos (conteos o el voto de
alguien) queda en `discrepancias`. Ninguna corrige a la otra.

Lo que una fuente publica se registra aunque no cuadre consigo misma (una lista
nominal con votos de más, grupos que no suman el conteo general): la votación
se publica tal como viene, con lo que no cuadra explicado en
`inconsistencias`, y los conteos no se ajustan. La diferencia exacta entre los
votos (listados más `votos_faltantes`) y los conteos queda declarada por
opción en `diferencia_conteos` y por grupo en `diferencia_grupos` (votos menos
conteo); el validador la recalcula, así que una votación que no cuadra nunca
pasa sin declararlo.

Las páginas del Senado y de la Gaceta las lee un programa. Si una página
trae un formato que el programa todavía no entiende, la lee un agente, y su
lectura solo se publica si pasa una verificación determinista que el agente no
controla: cada nombre y cada cifra están en la página, las listas suman los
totales que declara la fuente y las etiquetas son las del catálogo. Esas
votaciones lo dicen en `lecturas` (qué página, el registro de la lectura en
`congreso-extractor/lecturas/` y lo que se verificó). Mientras tanto se corrige
el programa; cuando ya lee la página, su resultado reemplaza al del agente. Solo se omite lo que no se pudo leer. Nada se borra: un
expediente o una votación que la fuente deja de publicar se conserva con
`retirada_de_la_fuente` (la fecha en que se notó) y su última versión.

## Cómo se enlaza

```
persona ─ mandato ─ voto ─ votación ─ expediente ─ decreto ─ (Documento, Publicacion) ─ commit del repo de leyes
```

- **Voto → mandato**: por nombre, dentro de la cámara y la legislatura (las
  filas de votación no traen la `Referencia` del SIL). El Senado y la Gaceta
  escriben los nombres de otra forma (orden, abreviaturas, variantes
  ortográficas); cada voto ligado así dice cómo en `enlace`. En cualquier
  fuente, un nombre que no da un único mandato, o dos filas que dan con la
  misma persona, no se atribuyen a nadie: quedan en `votos_faltantes` con el
  nombre tal como lo escribe la fuente (`nombres_fuente`).
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
- **Asunto → autor**: por la `Referencia` del SIL que trae el enlace del
  autor, la misma del mandato; no se compara el nombre. Si el texto del autor
  no se puede emparejar uno a uno con sus enlaces, se conserva tal cual sin
  ligarlo (`tipo: legisladores`).
- **Asunto → expediente**: por su `Seguimiento` del SIL, cuando ese
  expediente está publicado aquí (los que llegaron a votación nominal).
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
  su lista nominal (el sitio del Senado sí lo nombra). Los votos listados se
  publican y lo que falta queda en `votos_faltantes` (grupo, sentido y
  cuántos), sin adivinar quién es.
- **El sentido del voto de cada congreso local**: el SIL solo lista los que
  remitieron voto aprobatorio.
- **Quién votó en las votaciones económicas**: se votan sin registro nominal.
  Cada expediente las lista en `votaciones_economicas` (dictamen, cámara,
  fecha, texto y resultado), para que se vea qué se decidió así; por ejemplo,
  las reservas desechadas sin votación nominal.
- **Cómo se desechó un asunto sin dictamen**: el SIL da la fecha en que una
  iniciativa o proposición quedó desechada, pero no siempre el acto (un
  dictamen negativo, un acuerdo de la Mesa Directiva por vencimiento de
  plazo); si no lo da, el registro solo dice "desechado" y la fecha.
- **Firmantes adicionales**: el SIL da el autor o autores que registra; una
  iniciativa suscrita por más legisladores puede listar solo a quien la
  presentó.
- **Cambios de bancada**: el SIL solo muestra el partido actual del mandato.
