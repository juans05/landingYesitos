# Medición de Meta Ads — Yesitos Divertidos (TendyPeru)

Todo vive en `index.html`, en el bloque `MEDICIÓN META PIXEL + CONSENTIMIENTO`, justo antes del script principal.

## 1. Activar el píxel

1. En **Meta Business → Administrador de eventos → Conectar orígenes de datos → Web**, crea (o elige) el píxel y copia su **ID** (15–16 dígitos).
2. En `index.html` cambia:
   ```js
   const META_PIXEL_ID = "PENDIENTE_PIXEL_ID";   // → "123456789012345"
   ```
3. Publica la página.

Mientras el ID sea `PENDIENTE_PIXEL_ID` (o no tenga formato válido) **no se carga nada de Meta**, no aparece el aviso de cookies y el pie dice "Esta página no usa cookies de analítica ni de publicidad".

No hay tokens en el frontend: el Pixel ID es público por diseño. El token de la Conversions API **nunca** va en esta página (ver §5).

## 2. Consentimiento

- Con ID válido, la primera visita muestra el aviso: **Aceptar · Rechazar · Configurar** (la casilla de medición viene desmarcada).
- Hasta que la persona acepte: no se descarga `fbevents.js`, no se envía ningún evento y no se crean las cookies `_fbp` / `_fbc`.
- La elección se guarda en `localStorage` (`tp_consent_v1`, con fecha) y se cambia luego desde **Preferencias de cookies** en el pie de página. Al retirar el permiso se llama a `fbq('consent','revoke')` y se borran `_fbp` / `_fbc`.
- Pendiente legal: la política de privacidad (Ley N.° 29733) sigue marcada como *Pendiente* en el pie; debe mencionar Meta Pixel.
- **No** copiar este bloque a las páginas de historias del QR (`tendyperu.com/k/...`) ni a álbumes de niños: el seguimiento publicitario es solo para esta landing de compradores adultos.

## 3. Eventos

| Evento | Cuándo se envía | Parámetros |
|---|---|---|
| `PageView` | Una vez al cargar el píxel (tras el consentimiento) | — |
| `ViewContent` | Al abrir el formulario con un kit y al cambiar de kit. Una vez por kit por visita | `content_name`, `content_type: product`, `content_ids`, `value`, `currency: PEN` |
| `Contact` | Al confirmar un pedido **válido** y abrirse WhatsApp. Una vez por pedido (código `TP-…`), aunque también toquen el botón de respaldo | lo anterior + `contact_method: whatsapp`, `num_items`, `value` = subtotal sin delivery |

| Kit | `content_ids` | Precio |
|---|---|---|
| Kit Mini (1 figura) | `yesitos-kit-1` | 5.90 |
| Kit Dúo (2) | `yesitos-kit-2` | 9.90 |
| Kit Aventura (3) | `yesitos-kit-3` | 12.00 |
| Kit Familiar (4) | `yesitos-kit-4` | 15.00 |

Lo que **no** envía `Contact`: abrir la página, los botones "Pedir / Elegir kit / Comprar aquí / Hacer mi pedido" (solo abren el formulario), elegir un kit, escribir en el formulario, un formulario con errores ni un envío frenado por el límite de pedidos.

No hay `InitiateCheckout` (no existe un paso de pago en la web) ni `Purchase` en el navegador.

`autoConfig` está desactivado para que Meta no invente eventos automáticos por clics ni lea campos del formulario.

## 4. Verificar en el Administrador de eventos

1. **Administrador de eventos → tu píxel → Probar eventos**, abre la URL publicada desde ahí (o con la extensión *Meta Pixel Helper*).
2. Acepta las cookies en el aviso.
3. **PageView**: debe aparecer uno al cargar.
4. **ViewContent**: pulsa "Elegir Kit Dúo" → `ViewContent` con `yesitos-kit-2` y `value 9.9`. Cambia a otro kit → un evento nuevo con ese kit. Volver a un kit ya visto no repite.
5. **Contact**: completa el formulario y confirma → un solo `Contact` con el kit, la cantidad y el subtotal. Pulsar además "Enviar pedido por WhatsApp" no crea otro.
6. **Duplicados**: un clic = un evento.
7. **Purchase**: no debe aparecer nunca al abrir WhatsApp.
8. Pulsa **Rechazar** (o retira el permiso en *Preferencias de cookies*) y recarga: no debe llegar ningún evento.

## 5. Compras reales (`Purchase`) — siguiente paso

El clic a WhatsApp es un interesado, no una venta. Para optimizar a compradores reales, la compra se registra **cuando el pago está confirmado**, desde el servidor (Conversions API):

1. Cada pedido llega por WhatsApp con su **código** (`• Código de pedido: TP-…`).
2. Al confirmar el pago (Yape/Plin/transferencia), alguien marca el pedido como pagado en un panel o planilla.
3. Un backend (por ejemplo un Cloudflare Worker o una función serverless) envía a la Conversions API:
   ```json
   {
     "data": [{
       "event_name": "Purchase",
       "event_time": 1761000000,
       "event_id": "purchase-TP-XXXX",
       "action_source": "website",
       "event_source_url": "https://<tu-dominio>/",
       "user_data": { "ph": ["<sha256 de 519XXXXXXXX>"] },
       "custom_data": { "currency": "PEN", "value": 34.00, "order_id": "TP-XXXX",
                         "content_ids": ["yesitos-kit-3"], "num_items": 2 }
     }]
   }
   ```
   `POST https://graph.facebook.com/<versión>/<PIXEL_ID>/events?access_token=<TOKEN>` — el **token** se guarda como secreto del backend, nunca en `index.html`.
4. Duplicados: `event_id = "purchase-" + código` y el backend guarda qué códigos ya envió; un pedido se envía una sola vez.
5. Solo compras pagadas de verdad: nada de eventos de prueba en producción (usa el `test_event_code` de *Probar eventos* mientras pruebas).

Revisa en la documentación vigente de la Conversions API qué `action_source` corresponde a pedidos cerrados por WhatsApp y qué datos de cliente convienen para la coincidencia: el teléfono (con consentimiento informado en la política de privacidad) suele ser el más útil aquí.

## 6. Usar `Contact` como conversión en Meta Ads

1. Crea la campaña con objetivo **Ventas** (o *Clientes potenciales*) y destino **Sitio web**.
2. En el **conjunto de anuncios → Evento de conversión**, elige el píxel y el evento **Contacto** (`Contact`).
3. Antes de lanzar, confirma en el Administrador de eventos que `Contact` aparece como **activo**.
4. Cuando `Purchase` llegue por la Conversions API con volumen suficiente (Meta suele pedir unas 50 conversiones por semana por conjunto para salir del aprendizaje), crea conjuntos nuevos optimizados a **Compra** para buscar compradores reales y no solo clics.
