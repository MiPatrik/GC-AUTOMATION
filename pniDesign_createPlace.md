# pniDesign_createPlace

Sub-process of `pniDesign`, called from the "Create place" call activity that replaced the
`REST_API_CALL: cross/createPlace` stub. It runs only on the *place not found* branch after
`cross/findPlaceForAddress`; on success the parent loops back to that find task, which now
returns the freshly created PLACE.

Spec: `api_de/sequence_diagrams/7000_0210-cross_smw_synchro.puml` (group "PNI Design") and
`api_de/prm_api/prm_api_testing_summary_24072026.md` (PRM API calls).

```
Odin -> CROSS: PNIDesign
CROSS -> SMW : get Token                      smw/getToken
CROSS -> SMW : Create place                   smw/createBuilding   (create_object, object_type=building)
SMW  --> CROSS: PLACE (building:12345)
CROSS -> CROSS: Create building               cross/createPlace    (POST /v1/node/, PLACE, ext id PNI_DE)
```

## Flow

```
start -> Initialize -> smw/getToken -> ◇ -> smw/createBuilding -> ◇ -> cross/createPlace -> ◇ -> Completed
                                       └ Failed                   └ Failed                  └ Failed
Completed -> "Create result: COMPLETED" -> end
Failed    -> "Create result: FAILED"    -> end
```

Every `◇` routes to `Failed` when `errorMessage != null`; each task's on-exit script validates its
outputs and sets `errorMessage`. HTTP failures from the Automaton handler are exceptions and are
caught by the boundary error event on the call activity in `pniDesign`.

## Variables

| Variable | Dir | Set by | Meaning |
|---|---|---|---|
| `addressCrossId` | in | parent | CROSS id of the address (logging / validation only) |
| `globalReferenceId` | in | parent | Address master id = SMW `address_id` = value of the PLACE `ADDRESS` reference (`gc_standard_configuration/data/ADDRESS/*.xml`: `ex:id` == `GLOBALREFERENCEID`) |
| `crossProjectName` | in | parent | CROSS project the node is created in |
| `addressLatitude`, `addressLongitude` | in | parent (`findAddress` on-exit) | WGS84 point used as node geometry (root node, `inheritedGeometry=false`) |
| `smwSystemId` | – | Initialize | `PNI_DE` (DE only, same rule as `updateCustomerRoom`) |
| `smwProjectName` | – | Initialize | `= crossProjectName` (see Gaps) |
| `smwAccessToken` | – | `smw/getToken` | `$.access_token` |
| `smwBuildingId`, `smwBuildingName`, `smwRequestStatus` | – | `smw/createBuilding` | `$.inserted.id`, `$.inserted.prm_name`, `$.request_status` |
| `placeCrossId` | out | `cross/createPlace` | crossId of the new PLACE |
| `placeSmwId` | out | `smw/createBuilding` on-exit | `building:<inserted.id>` — the `<object_type>:<id>` form CROSS already stores for SMW ids |
| `errorMessage` | out | on-exit scripts | `null` on success |

`pniDesign` gained `addressLatitude`, `addressLongitude`, `placeSmwId`, `errorMessage` as declared
variables (the FAILED script no longer declares its own `errorMessage` local) and puts `placeSmwId`
into the response.

## Automaton sources (`210_automaton/init/runtime-config(gc*).js`)

| source | call | notes |
|---|---|---|
| `smw/getToken` | `POST {gvarSmwTokenApi.baseUrl}/oauth/token?grant_type=client_credentials`, basic auth `client` | maps `accessToken` |
| `smw/createBuilding` | `POST {gvarSmwApi.baseUrl}/glb_nig_prm_api_create_object?json=true&access_token=…` body `object_type=building, use_existing_project=true, project_name, address_id, create_ap=true` | maps `smwBuildingId`, `smwBuildingName`, `smwRequestStatus` |
| `cross/createPlace` | `POST {gvarCrossApi.baseUrl}/v1/node/?projectName=…` — `nodeTypeDiscriminators=[PLACE]`, `statusDiscriminator=DESIGNED`, `geometry=POINT(lon lat)`, `externalIds=[{PNI_DE, building:<id>}]`, `customAttributes=[ADDRESS -> {externalId: address master id, systemId: ADDRESS_MASTER}]` | maps `crossId` |

No `processes[]` entry is needed: only the parent is started by Automaton.

### Why the SMW URL is not a BPMN global

`SdcApiWorkItemHandler` posts every task to `_apiCallUrl` (Automaton `/api/jbpm/apiCall`) and
Automaton resolves the target URL from the source definition. A `_smwApiUrl` global would never be
read by anything, so the SMW hosts live in `gVars` (`gvarSmwTokenApi`, `gvarSmwApi`) next to
`gvarOdinApi`.

## Mocking SMW on smartmock.io

Both gVars point at the existing smartmock project host (`green-ladybug-12.app.smartmock.io`). Add
two routes there:

1. `POST /uaa/oauth/token` → 200
   ```json
   {"access_token": "mock-token", "token_type": "bearer", "expires_in": 43199, "scope": "gss.user", "jti": "mock"}
   ```
2. `POST /prmapi/glb_nig_prm_api_create_object` → 200
   ```json
   {
     "inserted": {
       "id": "132384385",
       "object_type": "building",
       "user!fullname": "DE NI HANN A0087",
       "address_id": "353e9b6c-f46e-e13f-49b7-74fe4e61c2de",
       "construction_status": "In Service",
       "address": "Dreikreuzenstr. 2, 30449 Hannover",
       "prm_name": "DE NI HANN A0087"
     },
     "design_id": 9059,
     "project_name": "00012-26-240726",
     "design_name": "26-240726",
     "status": "Planning Internal",
     "job_type": "Delivery project",
     "request_status": "ok"
   }
   ```
   A smartmock template can echo the request (`{{body.address_id}}`, `{{query.access_token}}`) to make
   the mock stateful enough for repeated runs; a static body is enough for the happy path.

The mock accepts any body, so the JSON body Automaton sends is fine. Failure path: return
`{"request_status": "error", "message": "..."}` and the flow ends with `errorMessage` set.

## Gaps before the real PRM API can be called

1. **Content type** — PRM expects `application/x-www-form-urlencoded`; `RestApiCallService.postOrPut`
   always serialises the payload as JSON. Needs a per-source `contentType` (or form encoding) in Automaton.
2. **Bearer header** — `headerAuth.headerValue` and `headers[].headerValue` are not SpEL-resolved
   (see the `TODO` in `RestApiCallService.setHeaders`), so the token from `smw/getToken` cannot be
   placed in `Authorization`. It is passed as `access_token` query parameter (RFC 6750 §2.3), which
   Spring-based UAA-protected APIs usually accept; the PRM deployment has to be checked.
3. **SMW project** — `create_object` requires an existing SMW project (`use_existing_project=true`).
   The puml's PNI Design group has no "create project in SMW" step and SMW names projects
   `<index>-<yy>-<order_number>`, so `smwProjectName = crossProjectName` is a placeholder. Either add a
   `glb_nig_prm_api_create_project` task (idempotency unclear — Globema's index generation creates a
   new project per call) or agree with Globema to address projects by `design_id`.
4. **Country** — `PNI_DE` is hard-coded; `pniDesign` has no `country` input (unlike
   `updateCustomerRoom`).
5. **SVG** — `GC-AUTOMATION.pniDesign_createPlace-svg.svg` is generated from the BPMNDI by a script and
   `GC-AUTOMATION.pniDesign-svg.svg` is stale. Open both in Business Central and save to regenerate.
