# Create a new directory for our project
mkdir weather
cd weather

# Initialize a new npm project
npm init -y

# Install dependencies
npm install @modelcontextprotocol/sdk zod
npm install -D @types/node typescript

# Create our files
mkdir src
touch src/index.ts

SosaJuanPedro: Puedes explicar el código y para qué sirve

GitHub Copilot: Claro, el código que compartiste es una implementación de un servidor que utiliza el protocolo **Model Context Protocol (MCP)** para proporcionar herramientas relacionadas con el clima. Este servidor interactúa con la API del Servicio Meteorológico Nacional de EE. UU. (**NWS API**) para obtener datos meteorológicos como alertas y pronósticos. A continuación, te explico las partes principales del código y su propósito:

---

### **1. Configuración del servidor MCP**
El servidor se crea utilizando la clase `McpServer` del SDK de MCP. Este servidor tiene un nombre (`weather`), una versión (`1.0.0`) y capacidades definidas (aunque en este caso están vacías).

```typescript
const server = new McpServer({
    name: "weather",
    version: "1.0.0",
    capabilities: {
        resources: {},
        tools: {},
    },
});
```

---

### **2. Función para realizar solicitudes a la API de NWS**
La función `makeNWSRequest` realiza solicitudes HTTP a la API de NWS, manejando encabezados específicos y errores. Devuelve los datos en formato JSON o `null` si ocurre un error.

```typescript
async function makeNWSRequest<T>(url: string): Promise<T | null> {
    const headers = {
        "User-Agent": USER_AGENT,
        Accept: "application/geo+json",
    };

    try {
        const response = await fetch(url, { headers });
        if (!response.ok) {
            throw new Error(`HTTP error! status: ${response.status}`);
        }
        return (await response.json()) as T;
    } catch (error) {
        console.error("Error making NWS request:", error);
        return null;
    }
}
```

---

### **3. Herramienta `get_alerts`: Obtener alertas meteorológicas**
Esta herramienta permite al usuario obtener alertas meteorológicas activas para un estado específico de EE. UU. (por ejemplo, "CA" para California). 

1. Construye la URL de la API para las alertas.
2. Realiza la solicitud a la API.
3. Si hay alertas, las formatea utilizando la función `formatAlert` y las devuelve como texto.

```typescript
server.tool(
    "get_alerts",
    "Get weather alerts for a state",
    {
        state: z.string().length(2).describe("Two-letter state code (e.g. CA, NY)"),
    },
    async ({ state }) => {
        const stateCode = state.toUpperCase();
        const alertsUrl = `${NWS_API_BASE}/alerts?area=${stateCode}`;
        const alertsData = await makeNWSRequest<AlertsResponse>(alertsUrl);

        if (!alertsData) {
            return {
                content: [
                    {
                        type: "text",
                        text: "Failed to retrieve alerts data",
                    },
                ],
            };
        }

        const features = alertsData.features || [];
        if (features.length === 0) {
            return {
                content: [
                    {
                        type: "text",
                        text: `No active alerts for ${stateCode}`,
                    },
                ],
            };
        }

        const formattedAlerts = features.map(formatAlert);
        const alertsText = `Active alerts for ${stateCode}:\n\n${formattedAlerts.join("\n")}`;

        return {
            content: [
                {
                    type: "text",
                    text: alertsText,
                },
            ],
        };
    },
);
```

---

### **4. Herramienta `get_forecast`: Obtener pronóstico del clima**
Esta herramienta permite al usuario obtener el pronóstico del clima para una ubicación específica (latitud y longitud).

1. Realiza una solicitud a la API de NWS para obtener datos de puntos de cuadrícula (grid points).
2. Usa esos datos para obtener la URL del pronóstico.
3. Recupera los datos del pronóstico y los formatea en un texto legible.

```typescript
server.tool(
    "get_forecast",
    "Get weather forecast for a location",
    {
        latitude: z.number().min(-90).max(90).describe("Latitude of the location"),
        longitude: z.number().min(-180).max(180).describe("Longitude of the location"),
    },
    async ({ latitude, longitude }) => {
        const pointsUrl = `${NWS_API_BASE}/points/${latitude.toFixed(4)},${longitude.toFixed(4)}`;
        const pointsData = await makeNWSRequest<PointsResponse>(pointsUrl);

        if (!pointsData) {
            return {
                content: [
                    {
                        type: "text",
                        text: `Failed to retrieve grid point data for coordinates: ${latitude}, ${longitude}. This location may not be supported by the NWS API (only US locations are supported).`,
                    },
                ],
            };
        }

        const forecastUrl = pointsData.properties?.forecast;
        if (!forecastUrl) {
            return {
                content: [
                    {
                        type: "text",
                        text: "Failed to get forecast URL from grid point data",
                    },
                ],
            };
        }

        const forecastData = await makeNWSRequest<ForecastResponse>(forecastUrl);
        if (!forecastData) {
            return {
                content: [
                    {
                        type: "text",
                        text: "Failed to retrieve forecast data",
                    },
                ],
            };
        }

        const periods = forecastData.properties?.periods || [];
        if (periods.length === 0) {
            return {
                content: [
                    {
                        type: "text",
                        text: "No forecast periods available",
                    },
                ],
            };
        }

        const formattedForecast = periods.map((period: ForecastPeriod) =>
            [
                `${period.name || "Unknown"}:`,
                `Temperature: ${period.temperature || "Unknown"}°${period.temperatureUnit || "F"}`,
                `Wind: ${period.windSpeed || "Unknown"} ${period.windDirection || ""}`,
                `${period.shortForecast || "No forecast available"}`,
                "---",
            ].join("\n"),
        );

        const forecastText = `Forecast for ${latitude}, ${longitude}:\n\n${formattedForecast.join("\n")}`;

        return {
            content: [
                {
                    type: "text",
                    text: forecastText,
                },
            ],
        };
    },
);
```

---

### **5. Ejecución del servidor**
El servidor utiliza `StdioServerTransport` para comunicarse a través de la entrada/salida estándar. Esto permite que el servidor reciba solicitudes y envíe respuestas.

```typescript
async function main() {
    const transport = new StdioServerTransport();
    await server.connect(transport);
    console.error("Weather MCP Server running on stdio");
}

main().catch((error) => {
    console.error("Fatal error in main():", error);
    process.exit(1);
});
```

---

### **Propósito general**
Este servidor MCP permite a los usuarios interactuar con herramientas para obtener:

1. **Alertas meteorológicas** para un estado específico.
2. **Pronósticos del clima** para una ubicación específica (latitud y longitud).

El servidor utiliza la API de NWS para obtener los datos y los formatea en un formato legible para el usuario. Es útil para aplicaciones que necesitan integrar datos meteorológicos en tiempo real.

SosaJuanPedro: Puedes darme la explicación en formato mark down

GitHub Copilot: ### Explicación del Código

El código proporcionado es una implementación de un servidor MCP (Model Context Protocol) que ofrece herramientas para obtener datos meteorológicos, como alertas y pronósticos, utilizando la API del Servicio Meteorológico Nacional de EE. UU. (**NWS API**). A continuación, se detalla la funcionalidad del fragmento de código destacado y su contexto:

---

#### **Fragmento Destacado**
```typescript
const formattedForecast = periods.map((period: ForecastPeriod) =>
    [
        `${period.name || "Unknown"}:`,
        `Temperature: ${period.temperature || "Unknown"}°${period.temperatureUnit || "F"}`,
        `Wind: ${period.windSpeed || "Unknown"} ${period.windDirection || ""}`,
        `${period.shortForecast || "No forecast available"}`,
        "---",
    ].join("\n"),
);
```

---

#### **¿Qué hace este fragmento?**

1. **Propósito**: 
   Este código formatea los datos de los períodos del pronóstico meteorológico obtenidos de la API de NWS para que sean legibles y fáciles de interpretar.

2. **Entrada**: 
   - `periods`: Una lista de objetos que representan los períodos del pronóstico meteorológico. Cada objeto tiene propiedades como:
     - `name`: Nombre del período (por ejemplo, "Hoy", "Esta noche").
     - `temperature`: Temperatura esperada.
     - `temperatureUnit`: Unidad de temperatura (por ejemplo, "F" para Fahrenheit).
     - `windSpeed`: Velocidad del viento.
     - `windDirection`: Dirección del viento.
     - `shortForecast`: Resumen breve del pronóstico (por ejemplo, "Soleado").

3. **Salida**: 
   - Una lista de cadenas de texto formateadas, donde cada cadena representa un período del pronóstico con información como:
     - Nombre del período.
     - Temperatura y unidad.
     - Velocidad y dirección del viento.
     - Resumen breve del pronóstico.

4. **Manejo de Valores Faltantes**:
   - Si alguna propiedad no está disponible, se utiliza un valor predeterminado como `"Unknown"` o `"No forecast available"`.

---

#### **Ejemplo de Salida**

Si `periods` contiene los siguientes datos:

```typescript
[
    {
        name: "Today",
        temperature: 75,
        temperatureUnit: "F",
        windSpeed: "10 mph",
        windDirection: "NW",
        shortForecast: "Sunny",
    },
    {
        name: "Tonight",
        temperature: 60,
        temperatureUnit: "F",
        windSpeed: "5 mph",
        windDirection: "N",
        shortForecast: "Clear",
    },
]
```

El resultado de `formattedForecast` será:

```
Today:
Temperature: 75°F
Wind: 10 mph NW
Sunny
---
Tonight:
Temperature: 60°F
Wind: 5 mph N
Clear
---
```

---

#### **Contexto en el Código**

Este fragmento forma parte de la herramienta `get_forecast`, que:

1. **Obtiene el pronóstico meteorológico** para una ubicación específica (latitud y longitud) utilizando la API de NWS.
2. **Formatea los datos del pronóstico** (como se muestra en este fragmento) para devolverlos al usuario en un formato legible.
3. **Devuelve el pronóstico** como una respuesta estructurada que puede ser utilizada por el cliente MCP.

---

#### **Propósito General**

El fragmento es clave para transformar los datos crudos de la API en un formato amigable para el usuario, asegurando que la información meteorológica sea clara y comprensible. Esto es útil para aplicaciones que necesitan mostrar pronósticos detallados de manera estructurada.