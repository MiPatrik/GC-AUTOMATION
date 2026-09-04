# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this project is

`GC-AUTOMATION` is a jBPM **kjar** (Kie archive) — a Maven project whose deliverable is BPMN2 process definitions, not Java code. It is authored in jBPM Business Central (the Process Modeler) and deployed to a Kie Server / jBPM runtime. The `src/main/java` and `src/test/java` trees are placeholders (`.gitkeep` only); all behavior lives in `.bpmn` files under `src/main/resources/com/gc/gc_automation/`.

## Build

```bash
mvn clean install   # produces target/GC-AUTOMATION-1.0.0-SNAPSHOT.jar (kjar)
```

`kie-maven-plugin` validates all BPMN2/DMN/DRL resources at package time — a broken process fails the build. There is no separate lint step and no test command (the `test` scope deps exist but no tests are written).

The `cz.sykora:sdc-jbpm:1.0.0-SNAPSHOT` dependency is not on a public repository; the build only succeeds if that artifact is already installed in the local `~/.m2`. If it's missing, install it from its source repo before running this build.

To run a single jBPM test you would add it under `src/test/java/com/gc/gc_automation/` and invoke `mvn test -Dtest=ClassName#method`; the `kie-api`/`junit`/`xstream` deps already in `pom.xml` are the intended harness.

## Architecture

### Everything is an `SdcApiCall`

The processes never call REST endpoints directly. Every service task has `drools:taskName="SdcApiCall"` and is dispatched to the custom work item handler `cz.sykora.SdcApiWorkItemHandler` (from the `cz.sykora:sdc-jbpm` dependency), wired in `src/main/resources/META-INF/kie-deployment-descriptor.xml`.

The handler is a generic API-call router. Each task passes two conceptual parameter families:

- `routeMap.*` — tells the handler *where* to go: `domain` (e.g. `TMF_STORE`, `REST_API_CALL`), `operation` (`GET` / `POST`), `transformation` (e.g. a JSONata template name), `source`. In practice only `TMF_STORE: get data` sets `operation` (`GET`); `REST_API_CALL` tasks route purely by `source` (the server infers the verb) — do **not** add `routeMap.operation` / `dataMap.method` to them.
- `dataMap.*` — the payload. Keys beginning `dataMap.jsonata.*` feed into the transformation template.

The endpoint itself comes from the process-scoped global `_apiCallUrl`, which is injected from the deployment descriptor via an MVEL resolver (see the `<global name="_apiCallUrl">` block in `kie-deployment-descriptor.xml` for the current value — it points at an environment-specific host and changes between dev/demo/prod). Every SdcApiCall task reads `_apiCallUrl` into its `apiUrl` input. When changing environments, edit that `<global>`; do **not** hardcode URLs in the BPMN.

### Work item definitions

`global/WorkDefinitions.wid` and `global/SdcApiCall.wid` (at the **repo root**, not under `src/main/resources/`) register work items so Business Central shows them in the palette, along with their PNG icons. These are metadata only — the runtime binding is the `<work-item-handlers>` block in `META-INF/kie-deployment-descriptor.xml`. Keep the two in sync when adding a new work item.

### How processes are started (Automaton entry pattern)

These processes are not invoked directly by ODIN. The upstream **Automaton** service (Camel + Groovy, in a sibling repo) accepts the HTTP request, stores the body in `automaton.tmf_store` keyed by an id, then calls `wfService.startWfSynchronousProcess("<processId>", { _order: <tmf_store.id> })`. As a result:

- Every process expects `_order` (`String`) as a start parameter — it is the `tmf_store.id` of the original request.
- The first SdcApiCall task in every process is `TMF_STORE: get data`, which uses `_order` to fetch that stored body and extract the request fields into process variables via JSONata.
- Synchronous processes return a `response` Map (assembled in a final script task) that Automaton sends back as the HTTP body. Async processes (`*Async.bpmn`) follow the same shape but Automaton returns immediately.

`pniDesign.md` (repo root) documents this end-to-end for the `pniDesign` flow (sequence diagram, request/response schemas, per-task variables) and is the best reference when working on any of the design processes — read it first. Flow specs themselves are maintained **outside this repo**; newer flows are handed over as PlantUML sequence diagrams annotated with `/'jBPM`, `/'SdcApiCall` and `/'Script` comment blocks that spell out each task's `routeMap`/`dataMap` inputs, outputs, gateway and on-entry/exit scripts (this is how `logicalDesign` was built).

### Processes

- `resourceAvailabilityCheck.bpmn` / `resourceAvailabilityCheckAsync.bpmn` — pre-design check. Chains SdcApiCall tasks: `TMF_STORE: get data` → `REST_API_CALL: get address list by masterId` → validates uniqueness (embedded Java `conditionExpression` on outgoing flows) → `REST_API_CALL: check equipment availability`. Errors set `errorMessage` via `kcontext.setVariable(...)` inside gateway conditions, then route to a `Failed` throw-link → catch-link → `FAILED` script task that puts `errorMessage` into `response.error`.
- `pniDesign.bpmn` — the main PNI design flow (creates CROSS project, resolves address/locality/building, allocates a cage or transceiver on the chosen edge equipment, creates an Optical Circuit, creates a service from a service template, then attaches everything). See `pniDesign.md` for the full step list and process variables.
- `pniDesign_Service_Access.bpmn` — service-access subset of the PNI design flow.
- `pniDesign_createPlace.bpmn` — sub-process called by `pniDesign` when no PLACE exists for the address: SMW token → SMW `create_object` (building) → CROSS PLACE node with the SMW id as `PNI_DE` external id. Returns `placeCrossId`, `placeSmwId`, `errorMessage`. See `pniDesign_createPlace.md`.
- `lniDesign.bpmn` — LNI design counterpart.
- `logicalDesign.bpmn` — logical design flow. From `serviceCrossId` (resolved via `TMF_STORE: get data`) it finds the service, project, `SC_PHYSICAL_ACCESS` placement and optical circuit, then creates a CPE, customer- and network-side optical patch cords and an ETH link, and finally sets the ETH link providers (network-side patch cord + optical circuit + customer-side patch cord). All FIND tasks run before the CREATE_OR_UPDATE tasks except `findCustomerSideCage`, which must follow `place CPE` (it needs the created `cpeCrossId`).

All processes share the same error-routing shape (any failure throws to a `Failed` link, the catch-link assembles a `FAILED` response). Script tasks and gateway conditions use inline Java (`language="http://www.java.com/java"`). Types allowed without FQN come from `project.imports` (`String`, `Integer`, `Long`, `Double`, `List`, `ArrayList`, `Map`, `LinkedHashMap`, `Collection`, `Number`, `Boolean`).

### Runtime assumptions

`kie-deployment-descriptor.xml` declares `runtime-strategy=SINGLETON` and JPA persistence against `org.jbpm.domain` / `java:jboss/datasources/ExampleDS` — the kjar is intended for a jBPM on JBoss/WildFly environment, not Kogito/Quarkus.

## Editing BPMN files

Each process is a **pair**: `foo.bpmn` (the model) and `GC-AUTOMATION.foo-svg.svg` (the rendered diagram Business Central displays). Business Central regenerates the SVG when you save through the modeler. If you edit `.bpmn` XML by hand, the SVG will go stale — preferred workflow is to round-trip through Business Central so the diagram stays in sync, and commit both files together (recent history shows this pattern: BPMN + matching SVG in each commit).

BPMN element IDs like `_8CB7A114-8CEA-4461-8718-36C5581E42D1` are used as prefixes across dozens of `<itemDefinition>` / `<dataInput>` / `<dataInputAssociation>` entries for a single task. When renaming or duplicating a task by hand, you must rewrite every occurrence of the UUID consistently or the kjar build will fail.

Validate any hand-authored or hand-edited process with `mvn -o clean install` — `kie-maven-plugin` compiles the embedded Java, so a bad script fails the build. Two traps:
- A script or gateway that references a **declared process variable** must not redeclare it (e.g. `String errorMessage = ...`) — the variable is already injected as a typed local, so this fails with *Duplicate local variable*. (`resourceAvailabilityCheck` does declare `String errorMessage` only because `errorMessage` is *not* a process variable there.)
- Ignore the pre-existing non-fatal parser noise from the other files (`shape_null` shapes in pniDesign/lniDesign/resourceAvailabilityCheck, a duplicate `signal` id in pniDesign); it does not fail the build.
