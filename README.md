# Proyecto Módulo 4: Análisis de Desastres Naturales (1970-2021)

Este proyecto se centra en el análisis de la Base de Datos Internacional sobre Desastres (EM-DAT), explorando eventos ocurridos entre 1970 y 2021. El objetivo es limpiar, transformar y preparar los datos para su posterior análisis y visualización, con el fin de identificar patrones y tendencias en los desastres naturales a nivel mundial.

## 📄 Descripción del Dataset Original

El conjunto de datos inicial, `1970-2021_natural_disasters-emdat_data.csv`, contiene información detallada sobre más de 27,000 desastres, con 47 columnas que describen cada evento, incluyendo:

- **Identificadores:** `Dis No`, `Year`, `Seq`, `Glide`.
- **Clasificación del Desastre:** `Disaster Group`, `Disaster Subgroup`, `Disaster Type`, `Disaster Subtype`.
- **Ubicación:** `Country`, `ISO`, `Region`, `Continent`, `Location`, `Latitude`, `Longitude`.
- **Fechas:** `Start Year/Month/Day`, `End Year/Month/Day`.
- **Impacto Humano:** `Total Deaths`, `No Injured`, `No Affected`, `No Homeless`, `Total Affected`.
- **Impacto Económico:** `Reconstruction Costs`, `Insured Damages`, `Total Damages` (en miles de USD).

## 🔄 Proceso de Limpieza y Transformación

Se realizó un exhaustivo proceso de limpieza y transformación de datos para preparar el dataset para el análisis. Las principales acciones llevadas a cabo fueron:

### 1. Eliminación de Columnas

Se eliminaron más de 25 columnas por diversas razones:
- **Alto porcentaje de nulos:** Columnas como `Glide` (89%), `Disaster Subsubtype` (93%), `Aid Contribution` (95%), `Reconstruction Costs` (100%) y `Insured Damages` (95%) fueron descartadas por la falta de datos.
- **Redundancia o bajo valor informativo:** Se eliminaron `Seq` (información contenida en `Dis No`), `Disaster Group` (todos los valores eran "Natural"), `ISO` y códigos administrativos (`Admin1 Code`, `Admin2 Code`).
- **Datos no relevantes para el análisis:** `Local Time`, `River Basin` y `CPI`.

### 2. Gestión de Valores Nulos

Para las columnas que se conservaron, se aplicaron las siguientes estrategias:
- **Relleno con información relacionada:**
  - `Disaster Subtype`: Los valores nulos se rellenaron con la información de la columna `Disaster Type`.
  - `Location`: Los nulos se completaron con el valor de la columna `Region`.
- **Conservación de nulos:** En columnas clave como `Event Name`, `Dis Mag Value`, `Latitude` y `Longitude`, se mantuvieron los nulos, ya que la información existente sigue siendo valiosa para análisis parciales.

### 3. Normalización y Estandarización

- **Países (`Country`):** Se normalizaron los nombres de los países para unificar registros, como agrupar "Germany Fed Rep" y "Germany Dem Rep" en "Germany", y estandarizar los nombres de Congo y Yemen.

### 4. Creación de Nuevas Columnas (Feature Engineering)

Se crearon columnas para facilitar el análisis temporal y de duración:
- **`Start Date` y `End Date`:** Se combinaron las columnas de año, mes y día (`Start Year`, `Start Month`, `Start Day`, etc.) en dos nuevas columnas de tipo fecha. Se manejaron los casos donde faltaba el día o el mes, resultando en valores `NaT` (Not a Time).
- **`Duration`:** Se calculó la duración de cada evento en días, restando `Start Date` de `End Date`.

### 5. Consolidación y Ajuste Final

- **Impacto Humano:** Se decidió conservar únicamente la columna `Total Affected`, ya que teóricamente representa la suma de `No Injured`, `No Affected` y `No Homeless`, simplificando el análisis del impacto total.
- **Tipos de Datos:** Se ajustaron los tipos de datos de las columnas de impacto a `Int64` (`Total Deaths`, `Total Affected`, `Duration`) para un manejo numérico correcto, admitiendo a la vez los valores nulos.

## ✅ Resultado Final

El proceso culminó con la creación de un nuevo archivo, `datos_revisados.csv`, que contiene un conjunto de datos limpio, estructurado y listo para ser utilizado en herramientas de visualización como Tableau. El DataFrame final está compuesto por las columnas más relevantes y transformadas para un análisis efectivo.

## 🧪 Pruebas de Mejora de Coordenadas con Geopy

Se identificaron errores e inconsistencias en las columnas de coordenadas (`Latitude`, `Longitude`), como valores textuales (ej. '30.37 N') o datos fuera de los rangos válidos. Se realizaron varios intentos para corregir y enriquecer estos datos.

### Intento 1: Corrección de formato y limpieza de títulos

El primer paso consistió en crear un script para:
1.  **Corregir el formato de las coordenadas**: Convertir valores textuales (ej. '30.37 N', '78.30 W') a un formato numérico decimal estándar.
2.  **Limpiar los nombres de las columnas**: Estandarizar los títulos a un formato `snake_case` para mejorar la compatibilidad con herramientas de análisis y bases de datos.

```python
import pandas as pd
import re
import unicodedata

def corregir_coordenadas(coord):
    """
    Convierte una coordenada en formato de texto (ej. '30.37 N', '78.30 W')
    a un formato numérico decimal.
    - 'N' y 'E' son positivos.
    - 'S' y 'W' son negativos.
    """
    if pd.isna(coord) or isinstance(coord, (int, float)):
        return coord

    coord_str = str(coord).strip().upper()
    numero_match = re.search(r'[-+]?\d*\.\d+|\d+', coord_str)
    if not numero_match:
        return None

    numero = float(numero_match.group(0))

    if 'S' in coord_str or 'W' in coord_str:
        return -abs(numero)
    
    return abs(numero)

def limpiar_titulos(df):
    """
    Limpia y estandariza los títulos de las columnas de un DataFrame.
    """
    nuevas_columnas = []
    for col in df.columns:
        col_normalizada = unicodedata.normalize('NFKD', col)
        col_ascii = col_normalizada.encode('ascii', 'ignore').decode('utf-8')
        col_limpia = re.sub(r'[^a-zA-Z0-9]+', '_', col_ascii).lower()
        col_limpia = col_limpia.strip('_')
        nuevas_columnas.append(col_limpia)
    
    df.columns = nuevas_columnas
    return df

try:
    # --- Carga y corrección inicial ---
    ruta_archivo = r'c:\Users\krigu\OneDrive\Desktop\ADALAB\Modulo-4\Proyecto_desastres_naturales_modulo_4\files\datos_revisados.csv'
    df = pd.read_csv(ruta_archivo)

    df['Latitud_corregida'] = df['Latitude'].apply(corregir_coordenadas)
    df['Longitud_corregida'] = df['Longitude'].apply(corregir_coordenadas)

    # --- Relleno de nulos y limpieza de títulos ---
    df['Latitud_corregida'].fillna(df['Latitude'], inplace=True)
    df['Longitud_corregida'].fillna(df['Longitude'], inplace=True)
    
    if 'Unnamed: 0' in df.columns:
        df = df.drop(columns=['Unnamed: 0'])
        
    df = limpiar_titulos(df)

    # --- Ajuste final de tipos de datos ---
    df["total_deaths"] = df["total_deaths"].astype("Int64")
    df["total_affected"] = df["total_affected"].astype("Int64")
    df["duration"] = df["duration"].astype("Int64")

    # --- Guardado ---
    ruta_salida = r'c:\Users\krigu\OneDrive\Desktop\ADALAB\Modulo-4\Proyecto_desastres_naturales_modulo_4\files\datos_finales_limpios.csv'
    df.to_csv(ruta_salida, index=False)
    
    print(f"¡Proceso completado! El archivo final ha sido guardado en: {ruta_salida}")

except Exception as e:
    print(f"Ha ocurrido un error: {e}")
```

### Intento 2: Enriquecimiento de coordenadas con Geopy

Posteriormente, se intentó utilizar la librería `geopy` para obtener coordenadas más precisas a partir de las columnas `location` y `country`, especialmente para las filas que presentaban datos inválidos.

### Intento 3: Optimización del proceso de Geocodificación

Para acelerar el proceso, se optimizó el script para que solo realizara una petición por cada localización única, en lugar de una por cada fila, y se añadió una barra de progreso.

```python
import pandas as pd
from geopy.geocoders import Nominatim
import time
from tqdm import tqdm

ruta_archivo_entrada = r'c:\Users\krigu\OneDrive\Desktop\ADALAB\Modulo-4\Proyecto_desastres_naturales_modulo_4\datos_revisados.csv'
ruta_archivo_salida = r'c:\Users\krigu\OneDrive\Desktop\ADALAB\Modulo-4\Proyecto_desastres_naturales_modulo_4\datos_con_coordenadas_reales.csv'

df = pd.read_csv(ruta_archivo_entrada, index_col=0)
geolocator = Nominatim(user_agent="proyecto_desastres_karen_adalab_v2", timeout=15)

df['Full Location'] = df['Location'].fillna('') + ', ' + df['Country']
localizaciones_unicas = df['Full Location'].unique()

coordenadas_cache = {}

for direccion in tqdm(localizaciones_unicas, desc="Geocodificando"):
    # ... (lógica de la función obtener_coordenadas_reales) ...
    time.sleep(1)

# Mapear resultados y guardar
coordenadas_series = df['Full Location'].map(coordenadas_cache)
df['Latitud Real'] = coordenadas_series.str[0]
df['Longitud Real'] = coordenadas_series.str[1]
df.to_csv(ruta_archivo_salida)
```

### Conclusión y Próximos Pasos

A pesar de los esfuerzos, el proceso de geocodificación con `geopy` resultó ser extremadamente lento (varias horas) debido a la necesidad de hacer pausas para no saturar el servicio gratuito de Nominatim. Además, se observaron algunas inconsistencias en los resultados que modificaban otras partes del dataset.

Debido a la falta de tiempo, se ha decidido posponer la corrección completa de las coordenadas y dejarla como un punto a mejorar en futuros pasos del proyecto. Al cargar los datos en Tableau, se observó que algunas columnas no se interpretaban correctamente, lo que refuerza la necesidad de una limpieza más profunda en esta área. Para futuras iteraciones, se proponen los siguientes pasos:

- **Mejora de Coordenadas con IA:** Usar la IA para mejorar los datos de coordenadas y así poder mostrar el punto exacto del desastre en el mapa, en lugar de solo el país.
- **Enriquecimiento de Datos de Origen:** Obtener más datos sobre el `Origen` del desastre para poder trabajar con esta variable y descubrir nuevas correlaciones.
- **Ampliación del Proyecto:** Considerar la necesidad de alargar la duración del proyecto para poder seguir revisando los datos. Esto permitiría sacar conclusiones más profundas sobre la duración de los desastres, obtener más información sobre los nombres de los eventos y refinar aún más el dataset.

## 📊 Visualización de los datos en Tableau

Utilizando el conjunto de datos limpio, se creó un dashboard interactivo en Tableau para visualizar las tendencias y patrones de los desastres naturales. Las principales conclusiones obtenidas del análisis visual son:

- **Frecuencia y Efecto Multiplicador:** Los desastres naturales son cada vez más frecuentes. Se observa un aumento constante en el número de eventos anuales, con un predominio claro de los desastres hidrológicos y meteorológicos.
- **Impacto Geográfico Desigual:** Aunque los desastres afectan a todo el mundo, su magnitud varía significativamente por región. Asia es el continente más afectado en términos de número de eventos y personas damnificadas, mientras que Oceanía es el menos afectado.
- **Consecuencias en la Salud Pública y Cooperación:** El aumento de las temperaturas y los desastres asociados, como olas de calor e inundaciones, generan graves problemas de salud pública. Esto incluye desde golpes de calor hasta la propagación de enfermedades transmitidas por vectores. Se destaca la necesidad de una cooperación internacional robusta para enfrentar estas consecuencias y mitigar los riesgos de futuros desastres.

Se puede ver el documento de Tableau, Desastres_naturales_1970-2021.twbx, en https://public.tableau.com/app/profile/laura.fern.ndez.rodr.guez/vizzes