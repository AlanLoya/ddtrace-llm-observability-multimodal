# ddtrace-llm-observability-multimodal

Skill reutilizable para implementar, migrar y diagnosticar **Datadog LLM
Observability** con `ddtrace` en aplicaciones Python que consumen modelos de
lenguaje, especialmente cuando trabajan con imágenes, audio o documentos
multimodales.

La skill está diseñada para evitar los problemas más comunes observados en
evaluaciones custom de Datadog:

- `DATA_ISSUE` por contenido visual incompleto.
- Payloads base64 truncados dentro del texto de los spans.
- Imágenes visibles solo como `mime_type` o referencias de adjuntos.
- URLs con SAS tokens, query strings o credenciales expuestas.
- Respuestas estructuradas que omiten campos obligatorios.
- Nombres, scores o conclusiones inferidas sin evidencia visual verificable.
- Inputs multimodales demasiado comprimidos para ser legibles.

## Objetivos

Esta skill ayuda a implementar un patrón consistente para:

1. Crear spans LLM con `LLMObs.llm`.
2. Anotar inputs y outputs con `LLMObs.annotate`.
3. Representar imágenes mediante `image_parts`.
4. Mantener el texto de los spans pequeño y libre de base64.
5. Reducir imágenes grandes sin destruir su legibilidad.
6. Sanitizar prompts, URLs, rutas y secretos.
7. Validar respuestas estructuradas antes de anotarlas o usarlas.
8. Rechazar resultados no verificables o con confianza insuficiente.
9. Registrar fallos explícitamente.
10. Probar el contrato de observabilidad sin depender de Datadog en tests.

## Cuándo usarla

Usa esta skill cuando necesites:

- agregar trazabilidad a una llamada de OpenAI, Azure OpenAI, Asimov u otro
  proveedor compatible;
- instrumentar extracción de texto desde imágenes;
- observar comparación de firmas, documentos, identificaciones o formularios;
- registrar imágenes o audio dentro de spans LLM;
- migrar el patrón de observabilidad a otro proyecto Python;
- investigar evaluaciones custom con `DATA_ISSUE`;
- corregir mensajes como:
  - `inline base64 content is explicitly truncated`;
  - `complete image content is unavailable`;
  - `the source is not fully available for assessment`;
  - `the output omits a required field`;
- evitar alucinaciones en extracción visual;
- hacer que la salida observada coincida con el contrato declarado por el prompt.

## Estructura de la skill

```text
ddtrace-llm-observability-multimodal/
├── SKILL.md
└── README.md
```

- [`SKILL.md`](./SKILL.md): instrucciones operativas que el agente carga bajo
  demanda.
- `README.md`: documentación humana, ejemplos y guía de adopción.

## Requisitos

La aplicación debe utilizar Python y tener instalado `ddtrace`:

```bash
pip install "ddtrace>=3.18.0"
```

Para una aplicación administrada por `uv`:

```bash
uv add "ddtrace>=3.18.0"
```

Para habilitar la instrumentación automática mediante `ddtrace-run`:

```bash
ddtrace-run python src/main.py
```

La configuración exacta de Datadog depende del entorno de despliegue. Como
mínimo, revisa la configuración de LLM Observability del servicio y asegúrate
de que el agente reciba los spans de `ddtrace`.

## Patrón recomendado

El flujo tiene cuatro capas separadas:

```text
Fuente de imagen/audio
        │
        ├── Payload para el proveedor LLM
        │      └── data URL u objeto multimodal requerido por el proveedor
        │
        ├── Payload para Datadog LLM Observability
        │      └── image_parts/audio_parts, sin duplicar base64 en content
        │
        └── Parser y validador de salida
               └── resultado usable o null/error explícito
```

El payload para el proveedor y el payload para Datadog no tienen que ser
idénticos. El proveedor puede requerir una data URL completa; Datadog debe
recibir el contenido multimodal en su campo estructurado correspondiente.

## Instrumentación básica

```python
import contextlib
import json
import os

from ddtrace.llmobs import LLMObs


def call_model(client, request_body):
    llm_context = (
        LLMObs.llm(
            model_name=os.getenv("MODEL_NAME", "unknown"),
            name="documents.extract_name",
            model_provider="provider",
        )
        if LLMObs.enabled
        else contextlib.nullcontext()
    )

    with llm_context as llm_span:
        response = client.post("/chat/completions", json=request_body)
        result = parse_and_validate_response(response)

        if LLMObs.enabled:
            LLMObs.annotate(
                span=llm_span,
                input_data={
                    "role": "user",
                    "content": "Extrae el nombre visible del documento.",
                },
                output_data={
                    "role": "assistant",
                    "content": json.dumps(result, ensure_ascii=False),
                },
            )

        return result
```

### Reglas del span

- Usa nombres estables y descriptivos:
  - `documents.extract_name`;
  - `signatures.compare`;
  - `invoices.classify`;
  - `contracts.validate`.
- Usa el nombre del modelo configurado, sin incluir secretos.
- Usa el proveedor correcto en `model_provider`.
- Anota el span después de obtener la respuesta.
- Mantén el mismo span para input, output, métricas y error.
- No dependas de `LLMObs.enabled` para que la lógica principal funcione.

## Manejo correcto de imágenes

### Formato de `image_parts`

Cada imagen debe anotarse como una parte estructurada:

```python
{
    "mime_type": "image/png",
    "content": "BASE64_SIN_PREFIJO_DATA_URL",
}
```

Formatos recomendados:

- `image/png`;
- `image/jpeg`;
- `image/gif`;
- `image/webp`.

El campo `content` debe contener únicamente el payload base64. No debe incluir:

```text
data:image/png;base64,
```

### Ejemplo completo

```python
LLMObs.annotate(
    span=llm_span,
    input_data={
        "role": "user",
        "content": sanitized_prompt,
        "image_parts": [
            {
                "mime_type": "image/png",
                "content": image_base64,
            }
        ],
    },
    output_data={
        "role": "assistant",
        "content": json.dumps(
            {
                "full_name": "MARIA LOPEZ",
                "confidence": 1.0,
            },
            ensure_ascii=False,
        ),
    },
)
```

### Regla más importante: no duplicar base64

No hagas esto:

```python
input_data = {
    "role": "user",
    "content": f"Imagen: data:image/png;base64,{image_base64}",
    "image_parts": [
        {"mime_type": "image/png", "content": image_base64}
    ],
}
```

La imagen queda duplicada. El campo textual puede ser truncado por Datadog y
las evaluaciones empiezan a recibir:

```text
[Truncated for length...]
```

El patrón correcto es:

```python
input_data = {
    "role": "user",
    "content": sanitized_prompt,
    "image_parts": [
        {"mime_type": "image/png", "content": image_base64}
    ],
}
```

## Presupuesto y compresión de imágenes

Las imágenes grandes pueden superar los límites del evento de observabilidad.
Antes de anotarlas:

1. valida que sean imágenes reales;
2. decodifica el base64 estrictamente;
3. conserva el MIME correcto;
4. limita el tamaño;
5. reduce dimensiones si es necesario;
6. recomprime conservando texto legible;
7. no uses thumbnails extremos para documentos.

Valores iniciales razonables:

```python
MAX_IMAGE_BYTES = 1_500_000
MAX_IMAGE_DIMENSION = 1600
```

Estos límites son una guía, no una garantía universal. Deben ajustarse al
tamaño de evento y la configuración de la versión de `ddtrace` utilizada.

### Error común: compresión excesiva

Una imagen de 320 px o de pocos kilobytes puede caber en el span, pero el texto
del documento puede dejar de ser legible. Esto produce evaluaciones que indican
que el nombre no puede verificarse, aunque el span sí tenga una imagen.

Para documentos:

- prioriza legibilidad sobre tamaño mínimo;
- conserva resolución suficiente para caracteres pequeños;
- comprime solo cuando el payload supera el presupuesto;
- valida visualmente una muestra del resultado.

## Sanitización de información sensible

La URL usada por el proveedor puede incluir tokens:

```text
https://storage.example/doc.png?sv=...&sig=...
```

La copia anotada en Datadog debe eliminar query strings y fragmentos:

```text
https://storage.example/doc.png
```

Para rutas locales, usa únicamente el nombre del archivo:

```text
/srv/app/storage/images/ine_123.png
```

se convierte en:

```text
ine_123.png
```

### Qué nunca debe llegar al prompt anotado

- API keys;
- SAS tokens;
- cookies;
- tokens OAuth;
- encabezados `Authorization`;
- rutas absolutas del servidor;
- secretos de configuración;
- identificadores internos innecesarios.

La sanitización debe aplicarse al texto de observabilidad, no necesariamente al
payload que se envía al proveedor. Sanitizar el request equivocado puede
romper la llamada al modelo.

## Validación de salidas estructuradas

Un prompt no es un validador. La respuesta debe validarse en código antes de
ser utilizada o anotada.

### Contrato recomendado para extracción visual

```json
{
  "full_name": "NOMBRE COMPLETO",
  "confidence": 1.0
}
```

Si la fuente no es verificable:

```json
{
  "full_name": null,
  "confidence": 0.0
}
```

### Reglas recomendadas

Rechaza el nombre cuando:

- la respuesta no es JSON;
- falta `full_name`;
- falta `confidence`;
- `confidence` es booleano;
- `confidence` no es numérico;
- `confidence` está fuera de `0.0..1.0`;
- `confidence < 0.95`;
- el nombre está vacío;
- el valor es `null`, `none`, `n/a` o `desconocido`;
- el modelo reconoce incertidumbre en cualquier carácter.

### Ejemplo de validación

```python
import json

MIN_CONFIDENCE = 0.95


def parse_name_response(content: str) -> dict:
    try:
        payload = json.loads(content)
    except (TypeError, json.JSONDecodeError):
        return {"full_name": None, "confidence": 0.0}

    confidence = payload.get("confidence") if isinstance(payload, dict) else None
    if (
        isinstance(confidence, bool)
        or not isinstance(confidence, (int, float))
        or not 0.0 <= confidence <= 1.0
        or confidence < MIN_CONFIDENCE
    ):
        return {"full_name": None, "confidence": 0.0}

    name = payload.get("full_name")
    if not isinstance(name, str) or not name.strip():
        return {"full_name": None, "confidence": float(confidence)}

    return {
        "full_name": " ".join(name.split()),
        "confidence": float(confidence),
    }
```

### Evitar alucinaciones visuales

El prompt debe exigir:

- copiar solo caracteres visibles;
- no completar palabras;
- no corregir ortografía;
- no usar el nombre de archivo;
- no usar la firma como fuente del nombre;
- no inferir desde contexto;
- devolver `null` si un carácter es incierto;
- distinguir el campo del titular de otros nombres del documento.

La validación en código debe reforzar esas reglas. Nunca conviertas una
respuesta inválida en un nombre plausible.

## Manejo de errores

El error debe ser visible para desarrolladores y para observabilidad:

```python
try:
    response = client.post(...)
    result = parse_response(response)
except (httpx.HTTPError, ValueError, KeyError) as exc:
    logger.exception("Error en documents.extract_name: %s", exc)
    result = {
        "full_name": None,
        "confidence": 0.0,
        "error": type(exc).__name__,
    }
    if LLMObs.enabled:
        LLMObs.annotate(
            span=llm_span,
            output_data={
                "role": "assistant",
                "content": json.dumps(result, ensure_ascii=False),
            },
        )
    return result
```

No uses:

- `except Exception: return {"full_name": "..."}`;
- fallbacks silenciosos con forma de éxito;
- scores inventados;
- nombres predeterminados;
- campos omitidos para ocultar errores.

## Diagnóstico de `DATA_ISSUE`

### Síntoma: base64 truncado

Reasoning típico:

```text
The embedded base64/image content is truncated.
```

Diagnóstico:

1. inspecciona el span;
2. busca `data:image` dentro de `input.content`;
3. elimina el base64 textual;
4. confirma `input.image_parts`;
5. confirma que cada parte tenga MIME y base64 válidos;
6. revisa que no exista una segunda copia en metadata o prompt;
7. emite un span nuevo.

### Síntoma: solo aparece metadata

Reasoning típico:

```text
Only attachment metadata is visible; image content is unavailable.
```

Diagnóstico:

1. confirma que `image_parts` incluya `content` o un mecanismo de adjunto
   compatible con la configuración de evaluación;
2. no dependas exclusivamente de un `attachment_key` si la evaluación custom
   no puede resolverlo;
3. evita guardar la imagen únicamente como una URL;
4. valida que el writer de la versión instalada de `ddtrace` soporte la forma
   elegida;
5. mantén una sola representación multimodal, sin copiar el base64 a texto.

### Síntoma: falta un campo de salida

Reasoning típico:

```text
The output provides a name but omits the required confidence field.
```

Diagnóstico:

1. valida la salida antes de anotarla;
2. conserva todos los campos declarados por el contrato;
3. si falta un campo crítico, anota una salida nula o de error;
4. actualiza las pruebas para cubrir la omisión.

### Síntoma: imagen presente pero ilegible

La fuente existe, pero fue reducida demasiado. Revisa:

- dimensiones;
- calidad JPEG;
- tamaño de caracteres;
- orientación;
- recorte;
- contraste;
- rotación del documento.

## Pruebas unitarias

Las pruebas deben verificar el contrato enviado a `LLMObs.annotate`, no solo el
resultado del proveedor.

### Caso básico

```python
def test_annotation_uses_image_parts(monkeypatch):
    annotations = []
    monkeypatch.setattr(LLMObs, "enabled", True)
    monkeypatch.setattr(LLMObs, "annotate", lambda **kwargs: annotations.append(kwargs))

    result = call_model(...)

    input_data = annotations[0]["input_data"]
    assert "base64" not in input_data["content"]
    assert input_data["image_parts"][0]["mime_type"] == "image/png"
    assert input_data["image_parts"][0]["content"] == "..."
```

### Casos mínimos

Prueba al menos:

1. imagen PNG válida;
2. imagen JPEG válida;
3. imagen demasiado grande;
4. imagen corrupta;
5. URL con token;
6. ruta local;
7. nombre válido con confianza `1.0`;
8. nombre sin confianza;
9. confianza menor a `0.95`;
10. confianza booleana;
11. JSON inválido;
12. excepción del proveedor;
13. output con `null`;
14. prompt sin base64;
15. ausencia de duplicación del contenido visual.

Los tests deben usar mocks. No deben requerir:

- API keys reales;
- tokens de almacenamiento;
- conectividad a Datadog;
- una cuenta del proveedor;
- imágenes sensibles del negocio.

## Checklist de implementación

### Instrumentación

- [ ] Existe un span `LLMObs.llm` por operación LLM.
- [ ] El nombre del span es estable y descriptivo.
- [ ] El modelo y proveedor no exponen secretos.
- [ ] La aplicación funciona si LLM Observability está deshabilitado.

### Input multimodal

- [ ] `content` contiene solo texto.
- [ ] No hay base64 dentro de `content`.
- [ ] Las imágenes están en `image_parts`.
- [ ] El MIME es correcto.
- [ ] El base64 es válido y no incluye el prefijo `data:`.
- [ ] No existe duplicación del mismo payload.
- [ ] Las imágenes conservan legibilidad.

### Seguridad

- [ ] Se eliminan query strings y fragmentos de URLs anotadas.
- [ ] Se eliminan SAS tokens y credenciales.
- [ ] No se anotan rutas absolutas innecesarias.
- [ ] Los logs no imprimen secretos.

### Output

- [ ] La respuesta se parsea antes de usarse.
- [ ] Se validan todos los campos requeridos.
- [ ] Se conserva `confidence` cuando forma parte del contrato.
- [ ] La incertidumbre devuelve `null`.
- [ ] Los fallos no se convierten en resultados exitosos.

### Validación

- [ ] Existen tests para el shape de `input_data`.
- [ ] Existen tests para el shape de `output_data`.
- [ ] Existen tests para imágenes grandes y corruptas.
- [ ] Existen tests para respuestas inválidas.
- [ ] Se inspeccionó al menos un span real en Datadog.
- [ ] Las evaluaciones ya no muestran truncamiento por base64 textual.

## Uso con agentes

La skill está disponible para el agente de VS Code porque se encuentra en
`.github/skills/`. Puede invocarse explícitamente con:

```text
/ddtrace-llm-observability-multimodal
```

También puede cargarse automáticamente cuando la solicitud menciona conceptos
como:

- `ddtrace`;
- `LLMObs`;
- `LLM Observability`;
- Datadog LLM;
- `DATA_ISSUE`;
- spans multimodales;
- `image_parts`;
- imágenes base64;
- payload truncado;
- validación de confianza;
- alucinaciones en OCR o extracción visual.

## Adaptación a otro proyecto

Para migrar el patrón:

1. copia el directorio de la skill al `.github/skills/` del nuevo repositorio;
2. identifica el boundary real de la llamada LLM;
3. adapta la creación del payload requerido por el proveedor;
4. reutiliza el contrato de `LLMObs.annotate`;
5. implementa el helper de sanitización de URLs y rutas;
6. implementa el helper de preparación de imágenes;
7. adapta el parser al esquema de salida del proyecto;
8. agrega tests del payload multimodal;
9. ejecuta el lint y las pruebas enfocadas;
10. inspecciona un span real antes de activar evaluaciones custom.

No copies ciegamente nombres de variables, límites o excepciones. Verifica la
versión instalada de `ddtrace`, el proveedor y la configuración de Datadog del
proyecto destino.

## Resumen

El patrón correcto es:

```text
LLM request
  -> LLMObs.llm
  -> input_data.content = texto corto y sanitizado
  -> input_data.image_parts = imagen completa y validada
  -> parsear y validar output
  -> output_data = JSON completo y verificable
  -> tests del contrato
```

La regla central es simple:

> El contenido visual debe viajar como contenido multimodal estructurado, nunca
> como base64 incrustado dentro del texto del span.
