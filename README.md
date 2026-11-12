# AI Support Engine

Prototipo funcional para clasificar solicitudes de soporte con dos enfoques: reglas tradicionales y un modelo generativo real mediante la API de **Groq**.

## Requisitos

- Node.js 22 o superior.
- Conexión a internet para el motor generativo.
- Una API Key de Groq. Se obtiene en https://console.groq.com/keys.

## Instalación

1. Descomprime el proyecto y abre la carpeta `AI-Support-Engine` en Visual Studio Code.
2. Abre una terminal dentro de esa carpeta.
3. Instala dependencias:

   ```bash
   npm install
   ```

4. Copia `.env.example` y crea un archivo llamado `.env`.
   En Windows PowerShell:

   ```powershell
   Copy-Item .env.example .env
   ```

5. Abre `.env` y reemplaza `PEGA_AQUI_TU_CLAVE_DE_GROQ` por tu clave privada. No compartas ni subas ese archivo a GitHub.

## Ejecutar el servidor

```bash
npm start
```

El servidor estará disponible en `http://localhost:3000`.

Comprobar funcionamiento:

```bash
Invoke-RestMethod http://localhost:3000/health
```

## Endpoint

`POST /api/analyze`

### Motor tradicional

```powershell
$body = @{ message = "No puedo iniciar sesión"; mode = "traditional" } | ConvertTo-Json
Invoke-RestMethod -Uri http://localhost:3000/api/analyze -Method Post -ContentType "application/json" -Body $body
```

### Motor generativo (Groq)

```powershell
$body = @{ message = "No puedo iniciar sesión"; mode = "generative" } | ConvertTo-Json
Invoke-RestMethod -Uri http://localhost:3000/api/analyze -Method Post -ContentType "application/json" -Body $body
```

Los dos modos devuelven el mismo contrato:

```json
{
  "category": "ACCESS",
  "priority": "MEDIUM",
  "response": "Respuesta profesional...",
  "requiresHumanReview": false,
  "mode": "generative"
}
```

Categorías: `ACCESS`, `BUG`, `FEATURE`, `SECURITY`. Prioridades: `LOW`, `MEDIUM`, `HIGH`, `CRITICAL`.

## Validaciones y seguridad

- Rechaza mensajes vacíos, tipos incorrectos, modos no permitidos y mensajes mayores a 2000 caracteres.
- Limita el tamaño del cuerpo JSON a 10 KB.
- La clave de Groq se lee desde variables de entorno; no se incluye en el repositorio.
- El motor generativo usa timeout de 20 segundos y maneja errores HTTP, de red y JSON.
- Se valida categoría, prioridad, respuesta y booleano antes de devolver el resultado.
- Todo resultado `SECURITY` fuerza `requiresHumanReview: true`.
- El prompt indica que el mensaje es contenido no confiable y que no debe obedecer instrucciones incrustadas.

## Pruebas comparativas (10 casos)

Con el servidor detenido o en otra terminal, ejecuta:

```bash
npm run compare
```

Este comando llama ambos motores en los mismos 10 casos (incluye mensajes ambiguos y dos intentos de prompt injection). Groq se consulta realmente; no hay respuestas generativas escritas manualmente.

Genera:
- `evidence/comparison-results.json`: resultados completos por caso.
- `evidence/comparison-results.md`: tabla, porcentajes y conclusión técnica de esa ejecución.

La precisión se calcula como:

`(casos con categoría y prioridad correctas / total de casos) * 100`

Si Groq falla, el caso se registra como error y no se inventa una respuesta. Configura la clave y repite el comando para obtener evidencia válida.

Prueba adicional del motor tradicional:

```bash
npm run test:traditional
```

## Estructura

```text
AI-Support-Engine/
├── src/
│   ├── server.js
│   ├── traditional.js
│   └── generative.js
├── tests/
│   ├── cases.js
│   └── traditional.test.js
├── scripts/
│   └── compare.js
├── evidence/
├── .env.example
├── .gitignore
├── package.json
└── README.md
```

## Decisiones técnicas

Se eligió Node.js con Express para crear una API sencilla. El motor tradicional utiliza palabras clave y reglas deterministas; el generativo consume la API compatible con OpenAI de Groq usando `fetch` nativo de Node.js. Se usa `llama-3.3-70b-versatile` como modelo predeterminado, configurable mediante `GROQ_MODEL`.

## Limitaciones

Las reglas tradicionales dependen de palabras clave y pueden fallar ante sinónimos o contexto. El modelo generativo puede producir resultados variables; por ello se valida estrictamente su salida y los incidentes de seguridad siempre requieren revisión humana. La precisión reportada depende de la ejecución real y de las etiquetas de referencia definidas en `tests/cases.js`.
