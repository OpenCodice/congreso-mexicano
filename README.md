# Congreso mexicano

Quién legisla en México y cómo vota, enlazado con el texto de las leyes que
reforma. Diputados y senadores con su partido, entidad y distrito; cada
iniciativa y proposición que presentan, con su autor y en qué quedó (también
las que nunca se dictaminaron); cada expediente legislativo con sus
dictámenes; cada votación nominal
con el voto de cada legislador; y cada decreto del Diario Oficial con el
commit que trajo la reforma a
[constitucion-mexicana](https://github.com/OpenCodice/constitucion-mexicana),
[leyes-federales](https://github.com/OpenCodice/leyes-federales) o
[fiscal-mexicano](https://github.com/OpenCodice/fiscal-mexicano).

> Repo de **datos**, generado por
> [`congreso-extractor`](https://github.com/OpenCodice/congreso-extractor) a
> partir del Sistema de Información Legislativa de Gobernación, el DOF, el
> sitio del Senado y la Gaceta Parlamentaria de la Cámara de Diputados. Cada
> dato cita la página oficial de la que salió, con su fecha de consulta y su
> sha256; las páginas oficiales se guardan en `fuentes/`.

## Un ejemplo

La reforma a los artículos 82, 116 y 122 de la Constitución (nacionalidad
única), publicada en el DOF el 7 de octubre de 2026:

- `decretos/2026/dof-5800617.json` — el decreto, el commit `260df83` de
  `constitucion-mexicana` que trajo el cambio, y los 22 congresos locales que
  la aprobaron.
- `expedientes/sil-5128211.json` — las iniciativas (del Ejecutivo y de una
  diputada), los dictámenes de cada cámara, la minuta y las declaratorias.
- `votaciones/lxvi/diputados/sil-5135887-1.json` — 344 a favor, 116 en contra,
  1 abstención y 38 ausentes, con el voto de cada diputado.
- `votaciones/lxvi/senado/sil-5146374-1.json` — 86 a favor y 42 en contra,
  con el voto de cada senador.

## Estructura

Sigue [Popolo](https://www.popoloproject.com/specs/), el estándar abierto para
datos legislativos, más extensiones propias (asuntos, expedientes y decretos). El detalle
está en [AGENTS.md](AGENTS.md).

```
organizaciones/   cámaras, partidos y los 32 congresos locales
personas/         una persona por archivo
mandatos/         lxvi/diputados/…, lxvi/senado/… (cargo, partido, distrito, licencias)
iniciativas/      lxvi/sil-{clave}.json: autor, comisiones, estatus y desenlace
proposiciones/    lxvi/… (puntos de acuerdo), con los mismos campos
dictamenes-negativos/  lxvi/… (los que desechan iniciativas o proposiciones)
expedientes/      el proceso de cada asunto en el SIL
votaciones/       lxvi/{diputados,senado,permanente}/… con el voto de cada legislador
                  (sil-…: del SIL; senado-… y gaceta-…: de cada cámara)
decretos/         {año}/dof-{código}.json
fuentes/          copias de las páginas oficiales, por su sha256
```

## Límites

- El voto individual de las votaciones en lo particular viene del sitio del
  Senado y de la Gaceta; el SIL solo desglosa la de lo general. Cuando dos
  fuentes difieren, se conservan ambas y la diferencia queda anotada.
- De los congresos locales solo se sabe cuáles aprobaron una reforma
  constitucional, no el voto de cada diputado local.
- Las votaciones económicas no registran nombres; cada expediente las lista
  en `votaciones_economicas`.
- Lo que una fuente publica se conserva aunque no cuadre consigo misma
  (`inconsistencias`) o aunque después lo retire (`retirada_de_la_fuente`).
- Por ahora cubre la LXVI Legislatura (2024–2027).

## Licencia

Datos públicos, dedicados al dominio público con [CC0 1.0](LICENSE).
