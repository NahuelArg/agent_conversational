# 🏗️ Arquitectura del Sistema - Análisis de Reputación MMI Analytics

---

## 1. VISIÓN GENERAL DEL SISTEMA

### Problema de Negocio

**Contexto ICC:**
Los hoteles canarios enfrentan un desafío crítico: **la gestión reactiva de la reputación online**. Actualmente, los gerentes hoteleros dedican 2-3 horas diarias a:
- Revisar manualmente cientos de comentarios en TripAdvisor, Google Reviews, Booking
- Identificar patrones de quejas recurrentes
- Preparar reportes ejecutivos para la dirección
- Responder a crisis de reputación cuando ya es tarde

**Impacto económico:**
- 1 estrella perdida en TripAdvisor = -15% en reservas (estudio Cornell University)
- Tiempo medio de detección de problemas: 7-14 días
- Coste de gestión manual: 4,800€/mes por hotel (2h/día × 30 días × 80€/h)

**Solución propuesta:**
Sistema conversacional inteligente que automatiza el 95% de este proceso, reduciendo:
- Tiempo de análisis: de 2 horas → 2 minutos
- Tiempo de detección de problemas: de 7 días → tiempo real
- Coste operativo: de 4,800€ → 299€/mes

---

## 2. ARQUITECTURA TÉCNICA DETALLADA

### Stack Tecnológico
┌─────────────────────────────────────────────────────────────┐
│                      CAPA DE PRESENTACIÓN                   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  Frontend SPA (HTML5 + Vanilla JavaScript)         │  │
│  │  - SessionStorage para persistencia cliente        │  │
│  │  - Fetch API para comunicación async               │  │
│  │  - CSS3 Grid/Flexbox para responsive design       │  │
│  └─────────────────────────────────────────────────────┘  │
│                           ↓ HTTPS/JSON                     │
└─────────────────────────────────────────────────────────────┘
↓
┌─────────────────────────────────────────────────────────────┐
│                    CAPA DE ORQUESTACIÓN                     │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  N8N Workflow Engine (Node.js)                     │  │
│  │                                                     │  │
│  │  • Webhook Server (Express.js interno)            │  │
│  │  • JavaScript Code Nodes (V8 engine)              │  │
│  │  • HTTP Request Client (Axios)                    │  │
│  │  • Error Handling & Retry Logic                   │  │
│  └─────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
↓
┌─────────────────────────────────────────────────────────────┐
│                    CAPA DE INTELIGENCIA                     │
│                                                             │
│  ┌──────────────────┐        ┌─────────────────────────┐  │
│  │  Google       │        │  Lógica Determinista   │  │
│  │  (Gemini)     │        │  (Regex + Keywords)    │  │
│  │                  │        │                         │  │
│  │  • GPT-4 level   │        │  • Extracción params   │  │
│  │  • 100k context  │        │  • Validación estado   │  │
│  │  • Streaming     │        │  • Filtrado avanzado   │  │
│  └──────────────────┘        └─────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
↓
┌─────────────────────────────────────────────────────────────┐
│                    CAPA DE DATOS                            │
│                                                             │
│  ┌──────────────────┐        ┌─────────────────────────┐  │
│  │  Google Sheets   │        │  TripAdvisor Data      │  │
│  │  (Temporal)      │        │  (via Apps Script)     │  │
│  │                  │        │                         │  │
│  │  • Sesiones      │        │  • 3,369 reviews       │  │
│  │  • Parámetros    │        │  • Multi-destino       │  │
│  │  • Historial     │        │  • Sentiment analyzed  │  │
│  └──────────────────┘        └─────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
### Flujo de Datos End-to-End
FASE 1: CAPTURA DE INTENCIÓN
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Usuario escribe: "hotel botanico"
↓
[Frontend] Genera session_id único (si no existe)
↓
POST /webhook {
session_id: "session_1234...",
mensaje: "hotel botanico"
}
↓
[N8N Webhook] Recibe y parsea JSON
↓
Output: {body: {session_id, mensaje}}
FASE 2: RECUPERACIÓN DE ESTADO
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[Google Sheets Get Row]
WHERE session_id = "session_1234..."
↓
Output (si existe): {
session_id: "session_1234...",
tipo: "hotel",
nombre: null,
periodo: null,
aspecto: null
}
↓
Output (si NO existe): {}
FASE 3: EXTRACCIÓN INTELIGENTE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[Code: Extraer Parámetros]
Input combinado:
• Webhook: {mensaje: "hotel botanico"}
• Sheets: {tipo: "hotel"}
Proceso:

Normalizar mensaje → "hotel botanico"
Detectar keywords:
✓ "botanico" → nombre: "Hotel Botánico"
Combinar con estado previo:
tipo: "hotel" (de Sheets)
nombre: "Hotel Botánico" (nuevo)
Validar completitud:
Faltan: periodo, aspecto
Generar mensaje apropiado

Output: {
session_id: "session_1234...",
status: "incompleto",
tipo: "hotel",
nombre: "Hotel Botánico",
periodo: null,
aspecto: null,
parametros_faltantes: ["periodo", "aspecto"],
mensaje: "Perfecto, Hotel Botánico. ¿Qué período?",
timestamp: "2025-10-08T10:30:00Z"
}
FASE 4: PERSISTENCIA
━━━━━━━━━━━━━━━━━━━━
[Google Sheets Update]
UPSERT WHERE session_id = "session_1234..."
↓
Sheets ahora contiene:
session_1234... | hotel | Hotel Botánico | null | null | ...
FASE 5: ENRUTAMIENTO INTELIGENTE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[Switch: status]
IF status = "completo":
→ Ruta ANÁLISIS
ELSE IF status = "incompleto":
→ Ruta CONVERSACIÓN
ELSE:
→ Ruta ERROR
En este caso: status = "incompleto"
↓
RUTA CONVERSACIÓN
FASE 6A: RESPUESTA CONVERSACIONAL
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[AI Agent: Conversacional]
Input: {
mensaje: "Perfecto, Hotel Botánico. ¿Qué período?",
contexto: {tipo: "hotel", nombre: "Hotel Botánico"}
}
Google Gemini procesa con memory window:
• Recuerda mensajes previos (últimos 10)
• Genera respuesta amigable
• Mantiene tono profesional
Output: {
output: "¡Excelente elección! El Hotel Botánico
es muy conocido. 😊
      ¿Qué período te interesa analizar?
      📅 Último mes
      📅 Últimos 3 meses
      📅 Último año"
}
↓
[Respond to Webhook]
JSON Response → Frontend
↓
Usuario ve el mensaje en el chat
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
CICLO SE REPITE hasta status = "completo"
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
FASE 6B: ANÁLISIS COMPLETO (cuando status = "completo")
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[HTTP Request: TripAdvisor]
GET https://script.google.com/.../exec?action=getData&limit=50
↓
Response: {
status: 200,
data: [
{Title: "...", Snippet: "...", Sentiment: "positive", ...},
{Title: "...", Snippet: "...", Sentiment: "negative", ...},
...50 comentarios
]
}
↓
[Code: Filtrar y Limpiar]
Proceso:

Filtrar por aspecto (ej: "comida"):
Keywords: ["comida", "restaurant", "buffet", "food"]
→ 23 comentarios coinciden
Filtrar por nombre (ej: "Hotel Botánico"):
Buscar en: Title, Snippet, Full Text, URL
→ 8 comentarios coinciden
Validar mínimo (necesita >= 3):
✓ Tenemos 8 comentarios
Seleccionar 3-5 aleatorios:
→ 5 comentarios seleccionados
Formatear estructura:
{
id: "comment_...",
texto: "El buffet es excelente...",
sentimiento: "positivo",
puntuacion: 5,
fecha: "2025-09-15",
aspecto_principal: "comida"
}
Calcular métricas:
Total: 5
Positivos: 3 (60%)
Negativos: 1 (20%)
Neutros: 1 (20%)

Output: {
parametros: {tipo, nombre, periodo, aspecto},
resumen: {total, positivos, negativos, neutros, %},
comentarios: [...]
}
↓
[Code: Preparar Prompt para AI]
Construye prompt estructurado:
Analiza la reputación de: Hotel Botánico
Aspecto: comida
Período: último mes

📊 MÉTRICAS:
- Total: 5 comentarios
- Positivos: 3 (60%)
- Negativos: 1 (20%)
- Neutros: 1 (20%)

📝 COMENTARIOS REALES:
Comentario 1:
- Sentimiento: positivo
- Puntuación: 5/5
- Texto: "El buffet es excelente, ingredientes frescos..."

[...más comentarios...]

Genera análisis profesional.
↓
[AI Agent: Analista]
Gemini analiza con context window:
• Lee métricas
• Identifica patrones
• Encuentra comentario negativo crítico
• Genera recomendaciones
Output: {
output: "📊 Análisis del Hotel Botánico - Comida
Se analizaron 5 comentarios reales. El 60%
      son positivos, indicando buena percepción general.
      
      ⚠️ Comentario negativo destacado:
      'El buffet tenía poca variedad para celíacos'
      
      ✅ Aspectos positivos:
      Los huéspedes elogian la calidad de ingredientes
      frescos y la presentación de platos.
      
      💡 Recomendaciones:
      1. Ampliar opciones sin gluten en buffet
      2. Capacitar personal sobre alergias
      3. Destacar opciones veganas/vegetarianas"
      }
↓
[Respond to Webhook]
JSON Response → Frontend
↓
Usuario ve análisis completo en el chat
---

## 3. DECISIONES ARQUITECTÓNICAS CLAVE

### Por qué N8N (vs otras opciones)

| Criterio | N8N | Zapier | Make | Desarrollo Custom |
|----------|-----|--------|------|-------------------|
| **Coste** | €0-20/mes | €30-100/mes | €10-50/mes | €5,000+ setup |
| **Flexibilidad** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Code custom** | ✅ JavaScript | ❌ No | ⚠️ Limitado | ✅ Total |
| **Self-hosted** | ✅ Sí | ❌ No | ❌ No | ✅ Sí |
| **AI integración** | ✅ Nativa | ⚠️ Parcial | ⚠️ Parcial | ✅ Total |
| **Learning curve** | Media | Baja | Media | Alta |
| **Time to market** | 1 semana | 3 días | 1 semana | 4-6 semanas |

**Decisión:** N8N por balance perfecto entre flexibilidad, coste y velocidad.

### Por qué Google Sheets (temporal)

**Ventajas:**
- ✅ Setup inmediato (0 configuración)
- ✅ Visual (cliente puede ver datos en tiempo real)
- ✅ Familiar para no-técnicos
- ✅ Backup automático (Google Drive)
- ✅ Colaborativo (múltiples usuarios)

**Limitaciones conocidas:**
- ⚠️ Escalabilidad: máx 10,000 filas
- ⚠️ Latencia: 500-1000ms por query
- ⚠️ Sin índices (búsqueda lineal)

**Plan de migración (Fase 2):**
```sql
-- PostgreSQL schema (futuro)
CREATE TABLE sesiones (
  session_id VARCHAR(50) PRIMARY KEY,
  tipo VARCHAR(20),
  nombre VARCHAR(100),
  periodo VARCHAR(20),
  aspecto VARCHAR(30),
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_session_updated ON sesiones(updated_at DESC);
CREATE INDEX idx_nombre ON sesiones(nombre);