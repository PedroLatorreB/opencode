# Design: Anular Signos Vitales

## Technical Approach

Full componentization: extract the 3x duplicated inline SV vuetables into a reusable `sv-table.vue` component, adding soft-delete (anulación) capability with reason selection, strikethrough styling, and a new API endpoint. The component encapsulates vuetable, detail row, anular modal, and agregar/borrar controls — one prop (`readonly`) controls mode.

## Architecture Decisions

| Decision | Alternative | Rationale |
|----------|-------------|-----------|
| Component path: `src/components/signos-vitales/` dir | Keep in `components/` root | Groups all SV-related components (sv-table + table-row-sv) for clean module boundary |
| Anular modal inside component via `b-modal` | Separate modal component | Self-contained; avoids prop-drilling modal state across views; matches existing `modal-anulacion-evolucion.vue` pattern |
| Motivos loaded inside component via `Global.ObtenerMotivosAnulacion(10)` | Passed as prop from parent | Self-contained, reduces boilerplate in 4 views; matches how existing components like `anamnesis.vue` load motivosAnulacion internally |
| `AnularSignosVitales(signosVitales, login)` takes list + login | Single-item endpoint | Atomicity per registry (NFR-01); controller handles iteration; same pattern as `EliminarSignosVitales` |
| Reuse existing `Prestador` property (already present in DTO) | Add new prestador field | DTO already has `PrestadorDTO Prestador` initialized in constructor — no change needed for that |
| Strikethrough via CSS class `text-danger` + inline style | Dynamic slot template | Simplest, matches Bootstrap utility pattern used elsewhere |

## Data Flow

```
┌──────────┐   spu_fcu_anular_signos_vitales    ┌──────────────┐
│   DB     │ ◄──────────────────────────────     │  API Layer   │
│ REG_     │   @REGISTRO, @INGRESO,              │              │
│ SIGNOS_  │   @RSV_CORREL, @MR_CODIGO,          │ POST /anular │
│ VITALES  │   @USU_LOGIN                        │              │
│          │   SET RSV_ANULADO=1,                │  ┌──────────┐│
│ +4 cols  │   MR_CODIGO,                        │  │Controller ││
│          │   USU_LOGIN_ANULACION,              │  │  ┌───────┐││
│          │   RSV_FECHA_ANULACION=GETDATE()      │  │  │Repo   │││
│          │                                     │  │  │SP call │││
│          │   spu_fcu_obtener_signos_vitales     │  │  └───────┘││
│          │ ────────────────────────────────►    │  └──────────┘│
│          │   +5 new columns in SELECT           └──────┬───────┘
└──────────┘                                           │
                                                        │ JSON
                                                        ▼
┌──────────────────────────────────────────────────────────┐
│  WEB Layer                                                │
│  ┌──────────────────────────────────────────────────────┐ │
│  │  sv-table.vue (NEW)                                   │ │
│  │  ┌──────────┐  ┌──────────────┐  ┌───────────────┐  │ │
│  │  │Vuetable  │  │table-row-sv  │  │b-modal anular │  │ │
│  │  │(columns) │  │(detail row)  │  │(v-select      │  │ │
│  │  │checkbox  │  │observaciones │  │ motivo +      │  │ │
│  │  │strike-   │  │              │  │ confirmar)    │  │ │
│  │  │through)  │  │              │  │               │  │ │
│  │  └──────────┘  └──────────────┘  └───────────────┘  │ │
│  └──────────────────────────────────────────────────────┘ │
│         ▲              ▲              ▲                    │
│         │              │              │                    │
│  ┌──────┴──────┐ ┌─────┴──────┐ ┌────┴──────┐             │
│  │ atencion   │ │ atencion-  │ │ atencion- │ categoriz.  │
│  │ .vue       │ │ enfermeria │ │ espec.    │ .vue        │
│  │            │ │ .vue       │ │ .vue      │             │
│  │ <SvTable   │ │ <SvTable   │ │ <SvTable  │ <SvTable    │
│  │  :signos-  │ │  :signos-  │ │  :signos- │  :signos-   │
│  │  vitales=  │ │  vitales=  │ │  vitales= │  vitales=   │
│  │  "..." />  │ │  "..." />  │ │  "..." /> │  "..." />   │
│  └────────────┘ └────────────┘ └────────────┘ └───────────┘
│  fichaClinica.js: Atencion.AnularSignosVitales(list)
└──────────────────────────────────────────────────────────┘
```

## File Changes

| File | Action | Description |
|------|--------|-------------|
| `DB/REG_SIGNOS_VITALES.sql` | **New** | ALTER TABLE: +4 columns (MR_CODIGO, RSV_ANULADO, USU_LOGIN_ANULACION, RSV_FECHA_ANULACION) |
| `DB/spu_fcu_anular_signos_vitales.sql` | **New** | SP: SET RSV_ANULADO=1, MR_CODIGO, USU_LOGIN_ANULACION, RSV_FECHA_ANULACION=GETDATE() |
| `DB/spu_fcu_obtener_signos_vitales.sql` | **Modified** | SELECT +5 columns (MR_CODIGO, RSV_ANULADO, USU_LOGIN_ANULACION, RSV_FECHA_ANULACION, USU_LOGIN) |
| `Common.Models/SignosVitalesDTO.cs` | **Modified** | +4 properties: MrCodigo (int?), RsvAnulado (bool), UsuLoginAnulacion (string?), RsvFechaAnulacion (string?). Prestador already exists. |
| `Urgencia.Data/Repository/AtencionRepository.cs` | **Modified** | Update `ObtenerSignosVitales()` mapping for 5 new columns + new `AnularSignosVitales(List<SignosVitalesDTO>, LoginDTO)` method |
| `Urgencia.Api/Controllers/AtencionController.cs` | **Modified** | New `POST api/v{version}/atencion/signosVitales/anular` endpoint |
| `WEB/src/components/signos-vitales/sv-table.vue` | **New** | Reusable SV component: vuetable + detail row + anular modal + agregar/borrar |
| `WEB/src/components/signos-vitales/table-row-sv.vue` | **Moved** | From `components/table-row-sv.vue` to `components/signos-vitales/table-row-sv.vue` |
| `WEB/src/services/fichaClinica.js` | **Modified** | New static `Atencion.AnularSignosVitales(signosVitales)` method |
| `WEB/.../views/fichaClinica/atencion.vue` | **Modified** | Replace inline vuetable with `<SvTable>` |
| `WEB/.../views/fichaClinica/atencion-enfermeria.vue` | **Modified** | Add `<SvTable>` (currently missing), clean dead SV code |
| `WEB/.../views/fichaClinica/atencion-especialista.vue` | **Modified** | Replace inline vuetable with `<SvTable>` |
| `WEB/.../views/fichaClinica/categorizacion.vue` | **Modified** | Replace inline vuetable with `<SvTable>` |

## Interfaces / Contracts

### POST /api/v{version}/atencion/signosVitales/anular

**Request:**
```json
{
  "signosVitales": [
    { "registro": 123, "ingreso": 456, "rsvCorrel": 1, "mrCodigo": 10 }
  ],
  "login": { "usuLogin": "admin" }
}
```

**Response 200:**
```json
{ "status": "ok", "message": "Signos vitales anulados correctamente" }
```

**Response 404:**
```json
{ "status": "error", "message": "Registro no encontrado" }
```

### sv-table.vue Props

| Prop | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `signosVitales` | Array | Yes | — | Array of SignosVitalesDTO objects |
| `readonly` | Boolean | No | `false` | Hides agregar/borrar when true |

### SignosVitalesDTO (added properties)

| Property | Type | Notes |
|----------|------|-------|
| `MrCodigo` | `int?` | Motivo rechazo code |
| `RsvAnulado` | `bool` | Default `false` |
| `UsuLoginAnulacion` | `string?` | Who anulated |
| `RsvFechaAnulacion` | `string?` | ISO date string |

## Testing Strategy

Manual testing in all 4 views (no unit tests exist in project):

1. **Happy path**: Open a ficha with SV records → click anular on a row → select motivo → confirm → verify strikethrough + red styling
2. **Readonly mode**: Open categorizacion view → verify agregar/borrar hidden → anular button still visible
3. **Empty state**: Open ficha without SV records → verify "No hay registros" message
4. **API 404**: Call anular endpoint with non-existent registro → verify 404 + error notification
5. **Already anulado**: Anular a row twice → verify second attempt shows appropriate state (button disabled)
6. **atencion-enfermeria**: Verify the new SV table renders correctly in this view for the first time

## Migration / Rollout

1. **Branch**: `feature/anular-signos-vitales` from `release`
2. **DB**: Run migration script on target DB before code deploy
3. **API**: Build + deploy after DB migration
4. **WEB**: Build + deploy after API
5. **Rollback**: `git revert` or `git checkout release && git branch -D feature/anular-signos-vitales`
6. **Rollback SQL**: `ALTER TABLE REG_SIGNOS_VITALES DROP COLUMN MR_CODIGO, RSV_ANULADO, USU_LOGIN_ANULACION, RSV_FECHA_ANULACION;` and restore SPs from backup

## Open Questions

- Should the anular button be disabled for already-anulated rows (prevent re-anulacion) or allow idempotent re-anulacion? Spec leaves this TBD (scenario 3.3).
- Does `atencion-enfermeria.vue` currently load SV data from `ficha.atencion.signosVitales` or a different path? The component imports from `ficha.atencionEspecialista.signosVitales` but also references `ficha.atencion.signosVitales` — needs verification during implementation.
- What specific `tmpCodigo` value does `Global.ObtenerMotivosAnulacion(tmpCodigo)` need for SV anulación? Spec uses `10` (same as other anulaciones) — verify this matches the MR_TMPCODIGO values in the DB.
