# End-to-end QEC workflows

Read the numbered files in order. Each sequence begins with its scope and assumptions and ends with questions about ownership, interfaces, and data. These are architecture proposals informed by the local knowledge base and vendor documentation reviewed on October 21, 2026. Meeting statements are evidence to evaluate, rather than requirements to inherit automatically.

## Narrative

| File | Contents |
| --- | --- |
| [00. End-to-end flow](00.end-to-end-flow.md) | Coarse allocation, scripts, compilation, execution, results, and release. |
| [01. Allocation and admission](01.allocation-and-admission.md) | Seven focused sequences for submission, evaluation, reservation, application handoff, waiting, retry, and rejection. |
| [02. Quantum task submission](02.quantum-task-submission.md) | Application runtime, hybrid programs, reservation propagation, QPM authorization, and dispatch. |
| [03. Compilation and QRI](03.compilation-and-qri.md) | Pass configuration, quantum and classical capabilities, physical mapping, and execution artifacts. |
| [04. Pilot runs and estimation](04.pilot-runs-and-estimation.md) | Initial characterization, noise-model refinement, decoder choice, and resource bounds. |
| [05. QEC configuration](05.qec-configuration.md) | Device runtime binds a Qblox control path to Riverlane Deltaflow or another compatible decoder. |
| [06. QEC execution loop](06.qec-execution-loop.md) | Measurement interpretation, decoding, frame ownership, feedback deadlines, and final observables. |
| [07. Hierarchical decoding](07.hierarchical-decoding.md) | Configure a supported multi-stage decoder, partition ownership, routes, windows, and reconciliation. |
| [08. Learning and syndrome export](08.learning-and-syndrome-export.md) | Configurable asynchronous capture, provenance, learning, and controlled model deployment. |
| [09. Calibration lifecycle](09.calibration-lifecycle.md) | Discovery, pass-specific reads, pre-launch validation, drift, and selective rebuilds. |
| [10. JIT and adaptive execution](10.jit-and-adaptive-execution.md) | Distinguish precompiled branches, parameter updates, host iteration, and genuine JIT compilation. |
| [11. Completion and recovery](11.completion-and-recovery.md) | Stop, drain, results, accounting, allocation release, and ambiguous failures. |
| [12. Demonstration workflow](12.demonstration-workflow.md) | A bounded memory experiment, simulator/hardware gates, reference decoding, and scientific versus integration claims. |
| [13. Component relationships](13.component-relationships.md) | Component diagram connecting the same actors across management and execution paths. |

## Shared actors

Actors denote responsibilities. They do not require separate processes or new services. Planner, model builder, exporter, and decoder management can be modules of existing runtimes. A participant named QRI represents the resource interface, not an additional scheduler or mandatory daemon.

| Diagram actor | Responsibility and knowledge-base relationship |
| --- | --- |
| Workflow Engine (`WE`) | Plans stages, submits Slurm allocations, tracks completion, and stages artifacts. |
| Slurm (`SL`) | Owns classical placement, job lifecycle, step launch, and retry. |
| qfw-slurm (`QS`) | Groups allocator SPANK, job-submit policy, burst-buffer callbacks, helper, and native driver at coarse scale. File 01 expands these. Remote SPANK injects reservation context. |
| Gateway (`GW`) | Authenticates Slurm requests, resolves QPM bindings, journals reservations, and coordinates release. |
| DEFw (`DF`) | Resolves exact service identity and runtime generation. Directory resolution does not create reservations. |
| QPM (`PM`) | QPMd service/controller; validates reservation-bound submissions, owns task state, and invokes admission, scheduling, and providers. |
| Admission (`AD`) | qhw-admission inside QPM; authoritative quantum capacity, reservation allowance, and usage. |
| QPU Scheduler (`SC`) | qhw-scheduler inside QPM; orders authorized tasks for dispatch. It is distinct from Slurm and the hardware pulse scheduler. |
| Application / Scripts (`APP`) | Starts on allocated classical resources; supplies source, circuits, parameters, and scientific goals. |
| Application Runtime (`AR`) | Drives classical/quantum regions, compiler passes, optional pilots, submission, and application-level iteration. QFw backend/resolver functions fit here. |
| QEC Planner (`PL`) | Chooses code, logical layout, decoder policy, and experiment constraints. May be application configuration or an application-runtime module. |
| Compiler (`CP`) | Logical lowering, code expansion, physical mapping, scheduling, and artifact generation. Passes may read QRI. |
| QRI (`RI`) | Capability and submission interface with QRMI/QDMI provider adaptation. In QFw the QPM telemetry APIs expose much of the existing discovery surface. |
| Device Runtime (`DR`) | Resource-side setup, binding, readiness, start, stop, and drain. Backline is one candidate implementation; transport plumbing such as SpaceRT may sit beneath it. |
| QPU Control (`CT`) | Sequencers, pulse/readout scheduling, measurement transport, and controller feedback. Qblox Cluster is the worked example. |
| QPU (`QP`) | Physical qubits or a simulator implementing the required execution behavior. |
| Decoder (`DE`) | An algorithm instance with a code/model contract, placement, throughput, latency, and output semantics. Riverlane Deltaflow is one worked implementation. |
| Model Builder (`MB`) | Estimates a circuit-specific noise model or decoder model from calibration and pilot data. |
| Telemetry Export (`TX`) | Buffers/copies selected data without blocking QEC execution. |
| Artifact Store (`ST`) | Existing workflow storage for programs, mappings, models, captures, and results. |
| Learning / Analysis (`LA`) | Evaluates results, trains models, and proposes versioned updates. |

## Evidence and proposed extensions

The local Slurm documents support allocation-scoped reservations, `QFW_RESERVATIONS`, and reservation-bound QPM execution. Existing discovery includes `get_backend_info`, `get_device_info`, `get_coupling_graph`, `get_calibration_snapshot`, and `capability_map`. The reviewed API inventory does not establish generic QEC-session configuration, raw syndrome subscriptions, hierarchical decoder setup, or a complete real-time controller ABI. Messages for those operations in these diagrams describe proposed contracts, not callable API names.

Some high-level notes place capacity holds in preliminary evaluation or let the compiler submit work directly. This set instead follows the more detailed qfw-slurm lifecycle: evaluate without a hold, select classical nodes, reserve during pre-run, then launch. Application execution goes through the application runtime and reservation-bound QPM path. The compiler produces artifacts and requirements.

Physical data/ancilla mapping belongs to compilation or a final mapping/lowering stage before launch. The device runtime applies and validates it. Decoder endpoints and physical link selection belong to resource-side binding, unless a target requires relocation into its final executable. Compiler requirements describe logical streams and timing rather than embedding arbitrary host IP addresses.

The previous [QEC analysis](../qec-wl-analysis.md) remains background. This set refines its actor table: physical-qubit mapping and runtime stream routing have different owners. QEC code and decoder selection may be automated planning/optimization passes under an explicit user policy; they need not require separate services or manual decisions.

## Answers that guide the diagrams

- Decoder selection has two dimensions: algorithm and execution placement. CPU, GPU, FPGA, and Riverlane are not interchangeable algorithm names. Choose from implementations compatible with the code, noise model, output semantics, capacity, and deadline. A vendor appliance need not accept an arbitrary decoder implementation.
- Expose application intent, such as supported code, target logical error rate, feedback mode, or an optional decoder preference. Keep concrete device binding with the resource. Recompile only if a choice changes circuit structure, timing, measurement interpretation, or the feedback ABI; changing only compatible weights may require a decoder reconfiguration.
- The compiler needs classical QEC capabilities when they constrain the quantum program. Ordinary host CPU allocation comes from Slurm. Resource-internal decoder/transport capabilities may be advertised through an extended QRI profile. Neither existing QRI coverage nor Slurm QPU bounds alone establish joint admission for every decoder resource.
- JIT is required when new information demands new executable instructions that were not prepared in advance. A measurement-dependent branch, a Pauli-frame update, and a parameter change do not inherently require compilation. Host-side JIT cannot be assumed to satisfy a live-qubit feedback deadline.
- Any arbitrary circuit is not automatically a useful FTQC demo. It must fit a supported logical gate set, encoding, topology, reset/readout capabilities, decoder contract, and resource budget. A memory experiment with known logical observables provides a smaller, interpretable first workflow. Compiling it demonstrates integration; improved logical performance requires measured evidence.

## Source map

The relative links below point to the local knowledge base outside this project. `repos` and `repos-main-sync` are snapshots, not guaranteed to be identical. Detailed lifecycle documents were preferred over conceptual sketches when they conflicted. No repository implementation was executed to establish hardware support.

| Input | Use |
| --- | --- |
| [qfw-slurm workflow](../../../../openQSE/QFw-release-v0.1/repos/qfw-slurm/docs/workflow.md), [integration guide](../../../../openQSE/QFw-release-v0.1/repos-main-sync/qfw-slurm/docs/workflow-manager-integration.md) | Allocation, bounds, launch, and ownership. |
| [Reservation sequence](../../../../openQSE/QFw-release-v0.1/repos-main-sync/reservation-workflow-sequence.md), [running sequence](../../../../openQSE/QFw-release-v0.1/repos-main-sync/running-job-workflow-sequence.md), [release sequence](../../../../openQSE/QFw-release-v0.1/repos-main-sync/release-workflow-sequence.md), [actor glossary](../../../../openQSE/QFw-release-v0.1/repos-main-sync/workflow-actors.md) | Detailed actors, identity, evaluation, reservation, execution, and release. |
| [Staged workflow](../../../../openQSE/QFw-release-v0.1/repos-main-sync/01.staged-workflow-reservation.md), [hybrid sketch](../../../../openQSE/QFw-release-v0.1/repos-main-sync/02.hybrid-coscheduled-reservation.md) | Application-level staging and co-scheduling intent; refined by the detailed lifecycle. |
| [QPM API inventory](../../../../openQSE/QFw-release-v0.1/repos-main-sync/QFw/docs/qpmd-supported-apis.md), [front-end contract](../../../../openQSE/QFw-release-v0.1/repos-main-sync/QFw/docs/design/qpu-frontend-contract.md), [shim guide](../../../../openQSE/QFw-release-v0.1/repos-main-sync/QFw/docs/qfw-shim-provider-guide.md) | Discovery, capability negotiation, submission, and reservation/provider binding. |
| [Admission standard](../../../../openQSE/QFw-release-v0.1/repos-main-sync/qhw-admission/docs/qhw-admission-standard.md), [scheduler](../../../../openQSE/QFw-release-v0.1/repos-main-sync/qhw-scheduler/README.md) | Capacity and task dispatch are distinct responsibilities. |
| [Backline workflow](../../../../openQSE/QFw-release-v0.1/repos-main-sync/backline-workflow.md) | Candidate resource-side controller/decoder/transport setup. |
| [Hardware data schemas](../../../../openQSE/QFw-release-v0.1/repos-main-sync/qhw-data/README.md), [characterization workflows](../../../../openQSE/QFw-release-v0.1/repos-main-sync/qhw-characterization/README.md) | Versioned metadata, calibration, pilots, and the existing memory-workflow limitations. |
| [October 19 minutes](../../10-19-2026.md), [CCAM/Rust minutes](../rust-infra.md) | Questions about modular QEC, hybrid programs, and timing domains. |

Vendor evidence is linked alongside the claims in files 05 and 06. Exact board connectors, QECi fields, management SDK calls, supported telemetry, and deployed firmware must be established with the actual system. The reviewed public QECi page describes the interface and offers specification access through a form; the full protocol specification was not inspected.
