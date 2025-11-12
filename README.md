# Automatización de Scraping y Análisis de Tendencias — ***n8n Project***

Este proyecto orquesta una serie de flujos en **n8n** para recopilar, analizar y generar recomendaciones de tendencias de **Instagram**, **TikTok** y **Google Trends**, enfocadas en **comida** y **moda** en las ciudades de **Quito** y **Guayaquil (Ecuador)**.

El sistema también incluye un envío automático de notificaciones y actualiza dashboards de Metabase.

---

## Estructura General

El proyecto está compuesto por **4 flujos principales**:

- **FLUJO ORQUESTADOR** — Controla y ejecuta la secuencia completa del proceso automáticamente.  
- **SCRAPPER PARA TIKTOK** — Obtiene, limpia y analiza tendencias desde TikTok.  
- **SCRAPPER PARA INSTAGRAM** — Obtiene, limpia y analiza tendencias desde Instagram.  
- **SCRAPPER PARA GOOGLE TRENDS** — Extrae y analiza tendencias de búsqueda de Google Trends.

---

## FLUJO ORQUESTADOR

**Objetivo:** Ejecutar los *scrapers* de TikTok, Instagram y Google Trends en secuencia cada día a las **6:00 AM**.

### Flujo de Ejecución

1. **Iniciar Flujo (Diario 6 AM):** Usa un nodo `Schedule Trigger` para ejecutar el flujo automáticamente cada mañana.  
2. **Ejecutar Sub-Flujo: TikTok:** Llama al flujo `SCRAPPER PARA TIKTOK` y espera su finalización.  
3. **Ejecutar Sub-Flujo: Instagram:** Una vez terminado TikTok, ejecuta `SCRAPPER PARA INSTAGRAM`.  
4. **Ejecutar Sub-Flujo: Google Trends:** Posteriormente, lanza el flujo encargado de extraer tendencias desde Google.  
5. **Notificar Fin del Proceso (Telegram):** Envía un mensaje al canal de Telegram configurado (`BotTendencias`) informando que los datos ya fueron actualizados en Metabase.

---

## SCRAPPER PARA TIKTOK

**Objetivo:** Obtener publicaciones de TikTok relacionadas con comida y moda en Quito y Guayaquil, clasificarlas y generar oportunidades de negocio mediante IA.

### Pasos del flujo

1. **Limpiar Recomendaciones Antiguas:** Elimina la colección `resumen_ia_tiktok` en MongoDB para evitar duplicados.  
2. **Generar Búsquedas (Queries):** Crea 4 búsquedas automáticas:
   "comida quito", "comida guayaquil", "ropa quito", "ropa guayaquil"
4. **Scrapear Videos de TikTok (Apify):** Usa el actor de Apify configurado para obtener los datos de las publicaciones.  
5. **Separar y Limpiar Datos Crudos:** Extrae información relevante (ID, descripción, hashtags, autor, vistas, likes, comentarios, etc.).  
6. **Asignar Ciudad (Quito / Guayaquil):** Detecta la ciudad mediante palabras clave en el texto, ubicación o hashtags.  
7. **Asignar Categoría (Comida / Ropa):** Clasifica el contenido con un conjunto ampliado de palabras clave para ambos temas (incluye ropa de segunda mano).  
8. **Guardar Videos en MongoDB (Upsert):** Inserta o actualiza los registros en la colección `tendencias_tiktok`.  
9. **Preparar Datos para IA:** Filtra y resume los campos más relevantes (descripción, hashtags, ciudad y categoría) de cada video.  
10. **Agrupar Videos para IA (Batch):** Une todos los posts en un solo bloque de texto numerado para enviarlo a la IA en una sola petición.  
11. **Generar Recomendaciones (Gemini IA):** Usa el modelo **`gemini-2.0-flash-lite`** para analizar las publicaciones y detectar oportunidades comerciales.  
12. **Extraer Texto de la IA y Particionar Recomendaciones:** Divide el resultado de la IA por grupos (`🍤 Comida Quito / Guayaquil`, `👗 Ropa Quito / Guayaquil`) y guarda cada segmento en la colección `resumen_ia_tiktok`.

---

## SCRAPPER PARA INSTAGRAM

**Objetivo:** Extraer *Reels* desde Instagram, clasificarlos por ciudad y categoría, y generar recomendaciones con IA.

### Pasos del flujo

1. **Limpiar Recomendaciones Antiguas:** Borra la colección `resumen_ia` en MongoDB.  
2. **Generar Búsquedas (Queries):** Genera automáticamente 4 consultas:  
"comida quito", "comida guayaquil", "ropa quito", "ropa guayaquil"

3. **Scrapear Reels de Instagram (ScrapeCreators API):** Recupera datos de *Reels* mediante la API configurada.  
4. **Separar Listas de Videos y Filtrar Datos Crudos:** Extrae campos clave como ID, URL, likes, comentarios, *caption*, hashtags, ubicación, etc.  
5. **Asignar Ciudad (Quito / Guayaquil):** Detecta la ciudad con coincidencias en *captions*, hashtags o *location*.  
6. **Asignar Categoría (Comida / Ropa):** Clasifica según palabras clave amplias (incluye ropa de segunda mano).  
7. **Guardar Videos en MongoDB (Upsert):** Inserta o actualiza en `tendencias_instagram`.  
8. **Filtrar Datos para IA y Agrupar Videos:** Prepara un resumen en texto para la IA, seleccionando campos clave y uniéndolos en un solo bloque de datos.  
9. **Generar Recomendaciones (Gemini IA):** El modelo analiza los posts y propone ideas de negocio específicas.  
10. **Extraer Texto y Particionar Recomendaciones:** Separa el texto de salida en los cuatro grupos y los guarda en la colección `resumen_ia`.

---

## SCRAPPER PARA GOOGLE TRENDS

**Objetivo:** Obtener, procesar y analizar las tendencias de búsqueda de Google Trends para comida y moda en Ecuador, y generar recomendaciones de negocio.

### Pasos del flujo

1. **Limpiar Recomendaciones Antiguas:** Elimina la colección de resúmenes generados por IA para evitar duplicados.  
2. **Generar Items de Trends:** Genera 8 ítems de datos, cada uno con el ID de la hoja de cálculo (`docId`) y el nombre de la hoja (`sheetName`) correspondientes a las tendencias almacenadas en Google Sheets. Estos incluyen:
- Series de tiempo (`MultiTimeline`)
- Mapas geográficos (`geoMap`)
- Consultas relacionadas (`consultas_relacionadas`)
3. **Bucle de Ejecución:** Itera sobre los 8 documentos de tendencia.  
4. **Leer y Formatear Datos:** Obtiene los datos del documento de Google Sheets para la iteración actual y los formatea automáticamente para la base de datos.  
5. **Guardar Datos en MongoDB (Upsert):** Inserta o actualiza los registros en las colecciones de MongoDB, clasificadas por tipo y categoría (ej: `google_timeline_comida`, `google_consultas_ropa`).  
6. **Filtrar y Unificar Consultas Relacionadas:** Aísla las consultas relacionadas de **Comida** y **Ropa**, y las combina en un único *string* de texto para el análisis de IA.  
7. **Generar Recomendaciones con IA:** El modelo **Gemini** analiza los términos combinados para generar una **Observación** clave y **dos Oportunidades** de negocio específicas para cada categoría (`Comida Ecuador` y `Ropa Ecuador`).  
8. **Guardar Resumen en MongoDB:** Almacena la recomendación final de la IA en una colección de MongoDB.

---

## Resultados Finales

- **Bases de datos actualizadas:**
- `tendencias_tiktok`
- `tendencias_instagram`
- `resumen_ia_tiktok`
- `resumen_ia` (Instagram)
- Colecciones de Google Trends (ej: `google_timeline_comida`, `google_consultas_ropa`)
- **Recomendaciones generadas por IA** agrupadas por tipo de contenido y ciudad.  
- **Notificación automática en Telegram** al finalizar el proceso.  
- Los resultados se visualizan en **Metabase** para análisis de tendencias y oportunidades.

---

