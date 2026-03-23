# AgencyLettings

**Sistema completo de agentes AI + motor operativo + base de datos local**
Stack: OpenClaw · Gemini 3.1 Flash Lite · Python · SQLite · wacli · Telegram

---

## Arquitectura

```
                     FUENTES DE LEADS
        ┌──────────┬──────────┬──────────┬──────────┐
        │ Facebook │ Gumtree  │  Directo │ WhatsApp │
        │  (CTWA)  │ listings │  web     │ inbound  │
        └────┬─────┴────┬─────┴────┬─────┴────┬─────┘
             │          │          │          │
             ▼          ▼          ▼          ▼
        ┌─────────────────────────────────────────────┐
        │         OpenClaw Gateway :3000              │
        │      openclaw/claw.config.json              │
        └────────────────┬────────────────────────────┘
                         │  routing determinista (bindings)
           ┌─────────────┼─────────────┐
           ▼             ▼             ▼
        [Rose]        [Ivy]          [Salo]
     FB/IG leads   WhatsApp        Gumtree/
                    directo       Rightmove/
                                   Zoopla
           │             │             │
           └──────┬───────┴──────┬──────┘
                  │  sessions_send (escalación)
                  ▼
            [Jeanette]          [Alex]
          Qualificación +    Manager Telegram
          Booking office     (reportes + alertas)
                  │
                  ▼
             [Script]
          Reactivación interna
          (aprobación humana)
                  │
                  ▼
        ┌─────────────────────┐
        │   engine/run.py     │  ← loop principal (sin cron)
        │   booking_engine.py │
        │   lead_engine.py    │
        └──────────┬──────────┘
                   │
                   ▼
        ┌─────────────────────┐
        │  database/agency.db │  ← SQLite local
        └──────────┬──────────┘
                   │
                   ▼
        ┌─────────────────────┐
        │  notifications/     │
        │  notify_manager.py  │  ← Telegram + WhatsApp
        └─────────────────────┘
```

**Reglas de oro:**
- Cero cron jobs — todo lo llama `engine/run.py` en loop de 60s
- Base de datos SQLite local — sin Supabase, sin dependencias cloud
- Modelo IA: `google/gemini-3.1-flash-lite` para todos los agentes
- Skills de Jeanette cargadas bajo demanda (no inyectadas completas en contexto)

---

## Estructura de directorios

```
~/AgencyLettings/
├── .env                          ← credenciales reales (NUNCA en git)
├── .env.example                  ← template seguro
├── .gitignore
├── Makefile                      ← todos los comandos del proyecto
├── requirements.txt
│
├── openclaw/
│   └── claw.config.json          ← gateway config (bindings + agentes)
│
├── agents/
│   ├── jeanette/                 ← workspace Jeanette (bootstrap auto-inyectado)
│   │   ├── SOUL.md               ← identidad y tono
│   │   ├── AGENTS.md             ← router multi-skill
│   │   ├── USER.md               ← perfil operador
│   │   ├── TOOLS.md              ← channels, address gate, slots
│   │   ├── MEMORY.md             ← memoria durable de leads
│   │   ├── memory/               ← logs diarios YYYY-MM-DD.md
│   │   └── skills/
│   │       ├── qualifier/SKILL.md
│   │       ├── scheduler/SKILL.md
│   │       ├── inventory/SKILL.md
│   │       ├── objections/SKILL.md
│   │       ├── nurture/SKILL.md
│   │       └── disqualify/SKILL.md
│   ├── ivy/IDENTITY.md
│   ├── rose/IDENTITY.md
│   ├── salo/IDENTITY.md
│   ├── alex/IDENTITY.md
│   └── script/IDENTITY.md
│
├── engine/
│   ├── run.py                    ← entry point (loop 60s)
│   ├── booking_engine.py
│   ├── lead_engine.py
│   │   └── sheets_sync.py
│
├── notifications/
│   └── notify_manager.py         ← Telegram + WhatsApp dual channel
│
├── database/
│   ├── agency.db                 ← SQLite runtime (gitignored)
│   └── schema/
│       ├── 001_leads.sql
│       ├── 002_properties.sql
│       ├── 003_bookings.sql
│       ├── 004_interactions.sql
│       ├── 005_contracts.sql
│       └── 006_listings_history.sql
│
├── scripts/
│   ├── init_db.py
│   ├── setup_server.sh
│   └── deploy.sh
│
└── tests/
    └── test_engine.py
```

---

## Instalación paso a paso

### Paso 1 — Clonar y configurar servidor

```bash
git clone https://github.com/tuuser/agencylettings.git ~/AgencyLettings
cd ~/AgencyLettings
bash scripts/setup_server.sh
```

### Paso 2 — Completar credenciales

```bash
nano .env
# Completar: TELEGRAM_BOT_TOKEN, TELEGRAM_CHAT_ID, MANAGER_WHATSAPP_NUMBER,
#            GOOGLE_SHEET_ID, HMAC_SECRET, números de WhatsApp por agente
```

### Paso 3 — Instalar servicio systemd

```bash
make install-service
```

### Paso 4 — Conectar WhatsApp (una cuenta por agente)

```bash
openclaw channels login --channel whatsapp --account lettings-jeanette
openclaw channels login --channel whatsapp --account lettings-ivy
openclaw channels login --channel whatsapp --account lettings-rose
openclaw channels login --channel whatsapp --account lettings-salo
# Escanear QR con cada número de WhatsApp correspondiente
```

### Paso 5 — Iniciar todo

```bash
make start
make status   # verificar que todo está activo
```

---

## Comandos rápidos

```bash
make start          # Iniciar engine + gateway
make stop           # Detener todo
make restart        # Reiniciar
make logs           # Ver logs del engine en tiempo real
make logs-notify    # Ver notificaciones enviadas
make status         # Estado completo del sistema
make db-shell       # SQLite shell interactivo
make test           # Ejecutar tests
make deploy         # Pull + restart
make backup         # Backup de la base de datos
make sync-sheets    # Sincronizar a Google Sheets ahora
```

---

## Flujo completo de un lead

```
1. Lead llega por Facebook/WhatsApp/Gumtree
2. OpenClaw Gateway → routing por accountId
3. Rose/Ivy/Salo recibe → colecta nombre + fuente
4. sessions_send escalación a Jeanette
5. Jeanette AGENTS.md router → carga skill correcta
6. qualifier: Fase 1→2→3 (nombre, budget, área, income, visa)
7. scheduler: ofrece slot → confirma → revela dirección
8. BookingEngine.schedule() → notificación manager
9. Engine loop: reminders 2h → no_shows → followups
10. record_outcome() → leads.status = 'won'
11. 08:00 London → daily_report() → Telegram + WhatsApp
```

---

## Queries SQLite útiles

```bash
make db-shell

-- Leads activos por score
SELECT name, status, score, agent FROM leads
WHERE status NOT IN ('won','lost','suppressed')
ORDER BY score DESC;

-- Bookings de hoy
SELECT l.name, b.booking_time, b.agent, b.status, b.outcome
FROM bookings b JOIN leads l ON b.lead_id=l.id
WHERE b.booking_date=date('now')
ORDER BY b.booking_time;

-- Pipeline resumen
SELECT status, COUNT(*) total FROM leads
GROUP BY status ORDER BY total DESC;

-- Leads sin contactar en 24h
SELECT name, agent,
  CAST((julianday('now')-julianday(last_contact))*24 AS INTEGER) hours_silent
FROM leads
WHERE status NOT IN ('won','lost','suppressed')
  AND last_contact IS NOT NULL
  AND hours_silent >= 24
ORDER BY hours_silent DESC;
```

---

## Modelo IA

Todos los agentes usan `google/gemini-3.1-flash-lite` configurado en
`openclaw/claw.config.json` bajo `agents.defaults.model`.

Las skills de Jeanette se cargan **bajo demanda** (el modelo hace `read`
del SKILL.md correspondiente cuando AGENTS.md lo indica), lo que mantiene
el contexto compacto y reduce tokens por mensaje.

---

## Mantenimiento

```bash
# Rotar logs (añadir a /etc/logrotate.d/agencylettings)
/home/ubuntu/AgencyLettings/data/logs/*.log {
    weekly
    rotate 4
    compress
    missingok
    notifempty
}

# Backup automático (añadir a crontab del usuario)
0 3 * * * cd ~/AgencyLettings && make backup
```