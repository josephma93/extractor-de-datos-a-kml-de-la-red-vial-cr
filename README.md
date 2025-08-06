# Extractor de datos de la red de carreteras de CR a kml
Un poco de codigo que GTP se genero para extraer datos de mapas a formato KML.

Estos son los mapas en cuestion:
- https://conavi.go.cr/red-vial-nacional-conavi
- https://www.arcgis.com/apps/dashboards/ff59885ec24940fcb1876b2669d78eed

GPT logro encontrar usando el modo agente que eso por debajo tiene una API que esta publica.
Les dejo un resumen del "conocimiento" que GPT recolecto (igual escrito por GPT).

---

# ArcGIS REST para la Red Vial Nacional y Cantonal de Costa Rica

**Dominio base de datos públicos:** `https://services8.arcgis.com/gEjMfQjIBmRHMGao/arcgis/rest/services`

## 1. Servicios principales

| Servicio                          | Propósito                                                                  | Capa (layer 0)              | CRS                       | Campos clave                                                                            |
| --------------------------------- | -------------------------------------------------------------------------- | --------------------------- | ------------------------- | --------------------------------------------------------------------------------------- |
| **Red\_Vial\_Nacional\_Proyecto** | Carreteras nacionales (MOPT‑Lanamme). Útil para rutas numéricas (Ruta 319) | `DBO.RVN`                   | CRTM05 (EPSG 102305/5367) | `RUTA`, `SECCIÓN`, `LONG__KM_`, `ZONA_DE_CO`, `TIPO_DE_SU`, etc.                        |
| **RVC\_Version\_final**           | Red Vial Cantonal validada (Lanamme). Cubre caminos cantonales             | `DBO.RVC_Validada_Merge`    | CRTM05                    | `Canton_1`, `Cod_camino_1`, `Ruta`, `Seccion`, `Validado`, `INICIO`, `FIN`, `LongSIG_m` |
| **RVN\_porcanton**                | Variante cantonal de rutas nacionales, más campos duplicados               | `DBO.RVN_porCanton`         | CRTM05                    | `canton`, `canton_1`, `Ruta`, `Seccion`, `Cod_camino` ...                               |
| **Rutas\_nacionales\_canton**     | Versión simplificada de rutas nacionales por cantón                        | `rvn_sirgas_cantones_final` | WKID 8908 (SIRGAS/CRTM05) | `Ruta`, `canton`, `Seccion`, `Sentido`, etc.                                            |

## 2. Pautas de consulta (endpoint `/query`)

* Estructura básica:

  ```text
  <service>/FeatureServer/0/query
    ?where=<cláusula SQL URL‑encoded>
    &outFields=*
    &returnGeometry=true|false
    &f=json
  ```
* **Cláusulas comunes**

  * Ruta específica: `where=RUTA=319`
  * Por cantón (cantonal): `where=Canton_1='Orotina'`
  * Añadir validación: `AND Validado='Si'`
  * Listar valores únicos: `returnDistinctValues=true&outFields=canton`
* **Encoding**: codificar la cláusula completa con `encodeURIComponent` si se construye en el frontend.
* **Formatos**: `f=json` (mejor para parseo); `pjson` añade pretty‑print; `geojson` para geometría directa.
* **Limitar tamaño**: `resultRecordCount` y `resultOffset` funcionan, pero las capas estudiadas no superan el límite por defecto (2000 features).

## 3. Estructura JSON devuelto

```jsonc
{
  "objectIdFieldName": "FID",
  "geometryType": "esriGeometryPolyline",
  "spatialReference": { "wkid": 102305 },
  "features": [
    {
      "attributes": {
        "RUTA": 319,
        "SECCIÓN": 10813,
        "Canton_1": "Orotina",
        "LongSIG_m": 9260.4,
        ...
      },
      "geometry": {
        "paths": [ [ [x1,y1], [x2,y2], ... ] ]
      }
    }
  ]
}
```

* `paths` es un array de segmentos; cada segmento es una polilínea de pares `(x,y)` en metros CRTM05.

## 4. Proyección

* **Origen**: CRTM05 (Transverse Mercator, lon₀ = –84°, lat₀ = 0°, k = 0.9999, GRS80/WGS84).
* **Destino habitual**: WGS 84 (EPSG 4326) para KML / web.
* **Proj4js** string: `+proj=tmerc +lat_0=0 +lon_0=-84 +k=0.9999 +x_0=500000 +y_0=0 +ellps=WGS84 +units=m +no_defs`.
* Necesario reproyectar antes de generar KML para que las rutas aparezcan en Costa Rica y no en el Golfo de Guinea.

## 5. Problemas y soluciones comunes

| Síntoma                            | Causa                                                  | Solución                                                                     |
| ---------------------------------- | ------------------------------------------------------ | ---------------------------------------------------------------------------- |
| “Invalid URL”                      | path mal cased (`arcgis` vs `ArcGIS`) o falta `/query` | Verificar mayúsculas y endpoint completo.                                    |
| “No where clause specified”        | parámetro `where` no llega codificado o malformado     | Usar `encodeURIComponent(where)` o probar en formulario interactivo primero. |
| Segmento colapsa y coloca (0,0)    | Polilínea con <2 puntos únicos tras redondeo           | Colapsar vértices cercanos o descartar segmentos < 0.01 m.                   |
| `proj4 is not defined` en frontend | librería no cargada o fuera de scope                   | Asegurar `<script src=…proj4.js>` antes de usar y asignar `window.proj4`.    |

## 6. Flujos de trabajo recomendados

1. **Listar cantones**

   ```
   …/RVC_Version_final/FeatureServer/0/query
     ?where=1=1&outFields=Canton_1&returnDistinctValues=true&returnGeometry=false&f=json
   ```
2. **Descargar rutas de un cantón**

   ```
   where = "Canton_1='Garabito'" // opcional + " AND Validado='Si'"
   url = <endpoint>/query?where=${encodeURIComponent(where)}&outFields=*&returnGeometry=true&f=json
   ```
3. **Reproyectar** y colapsar puntos cercanos (<0.01 m).
4. **Generar KML** con nombre descriptivo `Ruta <canton>` y `<name>Sección {Seccion}</name>`.

## 7. Buenas prácticas de logging (frontend)

* Registrar eventos como JSON por línea: `{ts, event, canton, url, features, segmentsBefore, segmentsAfter, discarded}`.
* Incluir error handling para `fetch` y reproyección.

---

Este documento resume la organización de datos, servicios y técnicas clave para consultar, reproyectar y generar KML de la red vial costarricense usando ArcGIS REST.

## Sistema de Referencia de Coordenadas (CRS)

### CRTM05 / EPSG 102305 / WKID 8908

| Parámetro         | Valor                                                                                    |
| ----------------- | ---------------------------------------------------------------------------------------- |
| Proyección        | Transverse Mercator                                                                      |
| Latitud de origen | 9 ° 00′ 00″ N (para Proj4 se expresa como `+lat_0=0` porque se mide respecto al ecuador) |
| Meridiano central |  −84 ° 00′ 00″ W (`+lon_0=-84`)                                                          |
| Factor de escala  |  0.9999 (`+k=0.9999`)                                                                    |
| Falso Este        |  500 000 m (`+x_0=500000`)                                                               |
| Falso Norte       |  0 m (`+y_0=0`)                                                                          |
| Elipsoide         |  WGS 84 / GRS80 (`+ellps=WGS84`)                                                         |

Este CRS se usa en todos los servicios de LanammeUCR para coordenadas en metros. Para visualizar en Google Earth o en visores web se necesita convertir a WGS 84 (EPSG 4326, grados decimales).

### Ejemplo de reproyección con Proj4js

```js
// Definir CRTM05 y WGS 84
proj4.defs('EPSG:5367', '+proj=tmerc +lat_0=0 +lon_0=-84 +k=0.9999 +x_0=500000 +y_0=0 +ellps=WGS84 +units=m +no_defs');
proj4.defs('EPSG:4326', '+proj=longlat +datum=WGS84 +no_defs');

// Convertir un par (x, y) de CRTM05 a lon/lat WGS84
auto const [x, y] = [450001.22, 1097040.33];
const [lon, lat] = proj4('EPSG:5367', 'EPSG:4326', [x, y]);
// lon ≈ -84.524 , lat ≈ 9.734
```

### Ejemplo de reproyección con pyproj (Python)

```python
from pyproj import Transformer
transform = Transformer.from_crs(102305, 4326, always_xy=True)
lon, lat = transform.transform(x, y)
```

Usar esta transformación antes de generar el KML evita que las rutas aparezcan en 0,0 o frente al golfo de Guinea.
