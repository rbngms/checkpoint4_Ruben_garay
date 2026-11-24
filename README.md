# Checkpoint 4 · Integraciones Avanzadas — Ruben Garay

Agente de **Calificación de Leads** construido en n8n. Este checkpoint extiende el workflow del Módulo 3 (multi-agente + memoria persistente) y lo conecta con tres herramientas reales del negocio mediante **OAuth2**.

## Credenciales conectadas (OAuth2)

| Rol | Herramienta | Uso en el flujo |
|---|---|---|
| CRM | **HubSpot** | Buscar contacto (lectura) y crear/actualizar contacto (escritura) |
| Casilla de correo | **Gmail** | Leer correos entrantes (trigger) y crear borradores (nunca envía) |
| Canal del equipo | **Slack** | Avisar al equipo de cada correo procesado y de cada escalamiento |

![Credenciales de n8n](credentials.png)

## Archivo

- `checkpoint4_ruben_garay.json` — workflow exportado desde n8n (Workflow > Download).

## Circuito de correo entrante

```
[Gmail Trigger: correo entrante]
        │
① [IF ¿auto-reply / no-reply?] ── Sí ─▶ Ignorar (corta el bucle infinito)
        │ No
[Set: Limpiar correo]  → From, Subject, BodyText (truncado)
        │
[Memoria (Sheets) por remitente] → [AI Agent: clasifica y redacta]
        │
④ [Set: Validar payload CRM] → [IF email válido] ── No ─▶ Descartar (evita 400)
        │ Sí
② [HubSpot: Look up por email]
   ┌────┴─────┐
 Existe     No existe
   ▼            ▼
[Update]    [Create]          (el Look up evita el 409)
        │
③ [Gmail: Create Draft]  → Human-in-the-loop: un humano revisa y envía
        │
④ [Set: aviso limpio] → [Slack: notificar al equipo] → [Guardar memoria]
```

## Controles implementados

1. **Anti bucle de auto-respuestas:** el IF revisa el asunto (*Auto-reply, Out of office, Undeliverable, Respuesta automática…*), el remitente (*no-reply@, mailer-daemon, postmaster*) y los encabezados `Auto-Submitted` / `Precedence`.
2. **Sin duplicados (409):** antes de escribir en HubSpot se busca el contacto por email; según el resultado se actualiza o se crea.
3. **Human-in-the-loop:** el único nodo de correo saliente usa la operación **Create Draft**. Ningún correo se envía sin aprobación humana.
4. **Payload limpio (400):** nodos Set dejan solo los campos necesarios (texto truncado, sin HTML ni binarios) y un IF valida el email antes del CRM.
5. **Mínimo privilegio:** el flujo solo lee y escribe contactos en HubSpot, solo lee correos y crea borradores en Gmail, y solo publica mensajes en un canal de Slack.
