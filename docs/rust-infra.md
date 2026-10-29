---
type: markdown
---

Rust Calibration and Pathfinder Integration Meeting Minutes
Meeting details
Date: October 21, 2026
Recorded start time: 2:03 PM
Duration: 57 minutes, 4 seconds
Participants speaking in the transcript: Santiago NÃºÃ±ez Corrales, Amir Shehata, Claire Marvinney, and Michael Sandoval
Main topics: CCAM runtime and Rust implementation; IQM/Pathfinder integration; Slurm and QPMD resource management; fault-tolerant execution and pulse-level timing; end-to-end stack description
Meeting purpose and central takeaway
The discussion reviewed Santiagoâ€™s early Rust-based CCAM runtime and IQM backend, then worked through what would be required to run the software against the ORNL Pathfinder testbed. The group also reviewed a separate fault-tolerant execution stack, including lower-level pulse and timing requirements.

The central takeaway was that integration work needs a shared, end-to-end view of the software stack before the group can make sound decisions about individual interfaces. The group needs to understand how the whole stack fits together; where the boundaries between layers sit; what data moves across each boundary; which component owns each responsibility; and where timing requirements begin and become strict. A high-level flow and sequence diagrams should establish this common picture, including the actors and artifacts involved, before deeper implementation discussions.

Executive summary
Santiago demonstrated a prototype that runs a classicalâ€“quantum program through a runtime that identifies classical and quantum regions, elaborates quantum instructions into circuits, and invokes a QPU backend.
The current approach includes provider-specific backend code and can use device-native gates. The transcript distinguishes this existing prototype from a possible future design using common QRI APIs and an exchange format, with hardware-specific translation handled by a service-side component.
Pathfinder adds facility-level requirements that are not captured by a direct cloud-provider workflow. Slurm and the Quantum Platform Manager Daemon (QPMD) need to coordinate reservations, admission control, scheduling, and access to both classical and quantum resources. The group needs to define how runtime submissions, payload formats, and dynamic shot counts fit into that model.
Adaptive shot selection can make a jobâ€™s quantum resource use difficult to predict at admission time. The group discussed feeding updated estimates back to admission control. Disabling adaptation in an ORNL-hosted fork was suggested as a simple initial path, while retaining adaptation as a later integration question.
Santiago also presented an early fault-tolerant path with successive runtime and library layers, syndrome/decoder streams, and hardware-facing operations. Pulse scheduling and the placement of data and ancilla qubits on Pathfinder were identified as integration concerns.
The group agreed to use end-to-end diagrams to create a shared vocabulary and invited Santiago to the Monday integration meeting. Santiago also agreed to apply for the OLCF project identified in the discussion as CSC 713.
Detailed discussion
1. CCAM programming model and runtime flow
Santiago introduced CCAM as a textual classicalâ€“quantum assembly language intended initially for systems programmers. A user may run a text program or a compiled binary through the same runtime path. A higher-level language is also being explored; it would let users describe desired properties of measurement distributions without directly specifying qubits, gates, or CCAM instructions.

The runtime divides a mixed program into classical and quantum regions. It symbolically processes the program, constructs an internal representation, and uses microcode or circuit templates to elaborate quantum instructions. Templates can adapt to the number of qubits used. Classical regions execute on the classical side, while quantum regions become circuits that are sent to a backend. When execution returns from a quantum region, the runtime incorporates the result and continues the classical program until it reaches another quantum region.

The discussion clarified the current conceptual path:

A user supplies a CCAM program, either as text or as a compiled binary.
The runtime identifies and separates classical and quantum regions and builds an internal representation.
Quantum instructions are elaborated into circuit blocks using templates; the runtime manages classical/quantum orchestration.
A QPU backend provides device-related behavior, including access to relevant device attributes and calibration information. The prototype can use native gates and includes qubit placement based on topology and calibration data.
The backend produces a device-compatible representation and submits the quantum work through the applicable provider or service interface.
Results return to the runtime, which makes them available to the remaining classical computation and continues execution.
This is a conceptual reconstruction of the flow discussed in the meeting, not a settled interface specification. In particular, the group has not yet decided exactly which compilation, translation, and submission responsibilities belong on the client, in the backend, or in QPMD and its service-side components.

Santiago noted that the current prototype has Rust crates for providers such as IBM and IonQ and a rough IQM backend. The provider/backend selection, credentials, device configuration, and simulator choice are currently controlled through runtime configuration. The design can use a state-vector or density-matrix simulator when a hardware backend is not selected.

2. Pathfinder deployment and the Slurm/QPMD boundary
The group contrasted a cloud-oriented workflow with the Pathfinder testbed. A direct provider backend may work for cloud access, but Pathfinder is a shared, Slurm-managed resource. The runtime is expected to be installed on the classical side of the testbed and made available to users through the facility environment. Users would submit a Slurm job that requests the needed classical resources and QPU access.

Amir described the current reservation concept. Before a program runs, the user provides metadata about expected quantum work, such as circuit count and maximum depth. A Slurm plugin passes that information to QPMD, which uses a resource estimator to determine whether the requested quantum capacity is available. If the request is admitted, Slurm reserves the quantum capacity and classical nodes together. If capacity is unavailable, the request can be rejected or retried later through the scheduler lifecycle. The intent is to avoid holding one resource while waiting indefinitely for the other and to avoid a separate queue at each side of the hybrid job.

The runtimeâ€™s execution model introduces a further boundary question. Santiago explained that when the program reaches a quantum region, classical execution pauses while the region is submitted with a shot count; execution resumes when results return. This may happen multiple times in a program. With adaptive shots, the runtime can submit an initial batch, evaluate whether the measurement distribution meets a quality criterion, and request more shots until the criterion is met. That behavior can change the resource requirement after the job has started.

The group identified a need to connect this runtime behavior to facility admission control. The system needs a way to estimate or communicate changing shot requirements so QPMD can track quantum capacity and make informed decisions about subsequent jobs. The group discussed a possible feedback mechanism in which the runtime reports updated estimates to admission control. A simple initial experiment could use an ORNL-hosted fork with adaptive shots disabled, but this was discussed as a starting option rather than a final design decision.

Michael raised the possibility of Slurm honoring an exclusive-access request by booking a Pathfinder time slot. Amir said exclusive access is a desired capability and is not yet in place. The conversation also raised dynamic resource estimation as an area worth exploring alongside exclusive scheduling.

3. Client-side translation, common interfaces, and payload flow
The discussion focused on how Santiagoâ€™s current backend would work with QPMD. In the prototype, the backend can generate a device-specific payload, such as an IQM-compatible representation. The current Pathfinder path, however, is being designed around a service that manages reservations and job admission, and a common interface for submitting circuits.

The group identified several possible architectural approaches that need to be made explicit in the diagrams and then evaluated:

Current prototype path: Provider-specific backend code prepares device-native or provider-specific work and submits it through that providerâ€™s interface.
Pathfinder integration path: The client-side runtime or backend calls the APIs exposed by QPMD or its service, and the service handles facility-specific execution and resource management.
Possible common-interface direction: A generic backend could retrieve device properties and calibration through QRI APIs, translate the program to a common exchange format, and submit that format to a service-side component. The server side could then translate to the format required by QRMI/QDMI or the device.
Amir described the need to support a payload that can pass through the submission path without being re-transpiled if it has already been prepared for the target device. He also raised the question of how this work fits with compiler infrastructure intended to support multiple SDKs, including MQT Core. Santiago was receptive to a common interface because maintaining a separate crate for every vendor could become difficult. The group did not settle the exchange format, the division of compilation and translation across client and server, or the relationship between CCAM and the broader compiler infrastructure.

The minutes therefore treat a generic backend, QRI-based discovery, a common payload, and server-side translation as candidate design directions, not approved implementation decisions.

4. Simulation, debugging, and user workflow
Santiago showed simulation capabilities intended to let users explore noise profiles and collect statistics on a classical system before using a QPU allocation. He also demonstrated a debugger concept that steps through the mixed program, exposing register and quantum-language state. He clarified that this is a program-level debugger using simulation, rather than a debugger that steps through the physical deviceâ€™s execution. A hardware-backed execution mode could be considered for tightly integrated systems, but it was not part of the current demonstration.

The discussion distinguished these development tools from a pilot job or runtime estimate. The debugger is intended to inspect program execution step by step; it is not primarily a way to predict how long the corresponding hardware run will take.

5. Fault-tolerant stack and timing requirements
Santiago presented a second, early stack for fault-tolerant execution. It begins with an OpenQASM source program that is compiled into annotated error-corrected code. Successive runtime and library layers handle logical operations, stabilizers, encoding, transversal gates, logical measurements, code and frame management, and error-correction rounds. A lower layer represents syndromes and decoders as streams with timing requirements, followed by hardware-facing instructions for operations such as selecting qubits, applying gates, and performing readout.

Santiago described pulse-level scheduling as a way to express the low-level schedules and timing boundaries associated with hardware operations. The current work runs in simulation. Santiago said it could be ready for hardware testing within the next couple of weeks, subject to the remaining work. He also said Chris Zimmer would help investigate which parts could run on an FPGA. The example raised a layout question for Pathfinder: how to place a codeâ€™s data and ancilla qubits on the available device topology.

Amir noted that the ORNL side still needs to support pulse-level operation and that work would begin on that support. The discussion reinforced the need to locate timing requirements in the stack: application-level timing and iteration, runtime submission and response, error-correction rounds, decoder processing, pulse schedules, and device operations have different constraints and may belong to different execution layers. The exact interfaces and timing guarantees between these layers remain to be mapped.

6. End-to-end stack understanding and sequence diagrams
Amir described a broader integration gap: teams can work deeply on their own components while still lacking a shared understanding of how the entire stack operates. He proposed starting from the top of the stack, identifying the actors and the information passed among them, and creating sequence diagrams that show which layer produces each artifact, whether work occurs at compile time or runtime, and where timing constraints appear.

Santiago agreed that this would help establish a common language and offered to prepare a short presentation or diagrams to explain the stack. Amir said he was already working on the diagrams and invited Santiago to the Monday meeting, held from 3:00 to 4:00 PM. Santiago accepted and said he would attend as often as possible.

The diagrams should make the following questions answerable:

What are the actors and layers from application source through runtime, compiler/backend, Slurm, QPMD, QRMI/QDMI, control stack, and device?
What data or artifact crosses each boundary (source, IR, circuit, native-gate payload, reservation metadata, credentials, job identifier, results, calibration/device metadata, or timing/syndrome data)?
Which component is responsible for generating, transforming, validating, scheduling, submitting, and returning each artifact?
Which work happens at compile time, job admission time, runtime, or within real-time control and error-correction loops?
Where does each timing requirement begin, what deadline or latency applies, and which layer is responsible for meeting it?
How do adaptive shots and repeated quantum regions affect reservations, resource estimates, and admission control?
Architecture boundary summary
Application and programming model	Express classicalâ€“quantum work in CCAM or a higher-level language; consume measurement results	Program source or binary; desired outputs and program parameters	Application-level iterations; waiting for quantum regions to return
CCAM runtime and IR/microcode	Identify classical and quantum regions, elaborate instructions, orchestrate execution, resume classical work	IR, circuit blocks/templates, shot requests, results	Runtime pauses at quantum regions; adaptive shots may add iterations
Compiler/backend and device description	Map logical/virtual qubits; use topology, calibration, and gate capabilities; prepare circuit or payload	Device attributes, calibration, circuit/native-gate representation, payload	Compilation and optimization may occur before submission; boundary is not yet settled
Slurm and QPMD	Reserve and coordinate classical/QPU resources; perform admission control and scheduling	Reservation metadata, capacity estimates, job state, service requests	Admission occurs before execution; retries and shared-resource scheduling affect wait time
QRMI/QDMI and device service	Provide a resource-facing interface and deliver or translate submitted work	Common or device-specific payload; submission request; status and results	Submission latency and hardware execution; exact service responsibilities remain to be defined
Fault-tolerant runtime, decoder, and pulse layers	Manage logical operations, error-correction rounds, syndrome/decoder processing, and hardware operations	Logical instructions, syndromes, decoder outputs, pulse schedules, hardware commands	Tighter timing constraints emerge in lower layers; specific guarantees and ownership need mapping
This table summarizes the discussion and highlights areas for design work. It is not a finalized allocation of responsibilities.

Agreements and working directions
Build a shared, end-to-end view of the stack before deep-diving further into individual components.
Use sequence diagrams to show actors, boundaries, data movement, responsibilities, compile-time versus runtime work, and timing constraints.
Add Santiago to the Monday 3:00â€“4:00 PM integration meeting; he agreed to participate when possible.
Santiago will apply for the OLCF project discussed as CSC 713 to obtain access needed for running on the system.
For an initial Pathfinder integration experiment, disabling adaptive shots in an ORNL-hosted fork was raised as a simple option. The longer-term interface between adaptive resource use and QPMD admission control remains open.
ORNL-side pulse support is needed for the fault-tolerant path; Amir said the team would begin looking into it.
Open questions
What is the authoritative end-to-end execution path for CCAM on Pathfinder, including the exact boundary between runtime, backend, QPMD, QRMI/QDMI, and the device service?
Which component owns circuit generation, native-gate mapping, transpilation/optimization, payload validation, and final device-specific translation?
What payload format should cross the client/service boundary, and how can already device-ready payloads pass through without being recompiled?
How should CCAM and its backend participate in the compiler infrastructure for multiple SDKs, including MQT Core?
What QRI APIs will provide device topology, native-gate information, calibration data, and other metadata, and which layers consume that information?
How should the runtime communicate adaptive shot counts or changing resource estimates to QPMD after admission?
How will Slurm express QPU exclusivity or time-slot reservations, and how will those interact with shared use and dynamic resource estimation?
Which timing constraints apply at each layer, particularly for pulse schedules, syndrome streams, decoder work, and hardware operations? Which layer guarantees each deadline?
How should the fault-tolerant path map code layouts, including data and ancilla qubits, to Pathfinderâ€™s device topology?
Which components of the fault-tolerant stack are candidates for FPGA execution, and what interfaces would they require?
Action items
1	Continue the end-to-end stack and sequence diagrams, identifying actors, layers, artifacts, data movement, responsibilities, and timing boundaries.	Amir	In progress; bring to the Monday integration meeting
2	Add Santiago to the Monday 3:00â€“4:00 PM integration meeting.	Amir	Follow-up agreed during the meeting
3	Apply for access to the OLCF project identified in the discussion as CSC 713; ask Michael if the project link or project number causes problems.	Santiago	To do
4	Contribute a short presentation or diagrams that explain the CCAM and fault-tolerant execution flows, including what happens at compile time and runtime.	Santiago	Proposed; coordinate with Amirâ€™s diagrams
5	Define a minimal Pathfinder integration path for CCAM, including how the runtime reaches QPMD and how a device-ready payload is submitted without unintended re-transpilation.	Santiago and ORNL integration team	Design work needed; exact interface not assigned
6	Clarify the client/service payload format and the division of work among the CCAM backend, QRI APIs, QPMD, and QRMI/QDMI. Include how this relates to MQT Core and multi-SDK compiler infrastructure.	Amir, Santiago, and compiler/QPMD contributors	Open architecture task
7	Determine how adaptive shot selection can inform QPMDâ€™s resource estimates and admission control; consider a disabled-adaptation prototype as an initial baseline.	Santiago and QPMD/admission-control team	Open; baseline is a suggested starting option
8	Investigate support for Slurm exclusive access or time-slot reservations for Pathfinder, and how it interacts with shared scheduling.	ORNL Slurm/QPMD team	Open; capability described as not yet implemented
9	Begin evaluating pulse-level support needed on the ORNL side for Santiagoâ€™s fault-tolerant work.	Amir / ORNL team	Follow-up stated during the meeting
10	Work with Santiago to identify which fault-tolerant components could run on an FPGA.	Chris Zimmer and Santiago	Collaboration mentioned; timing not specified
11	Prepare the fault-tolerant implementation for hardware testing and determine the device-layout mapping for data and ancilla qubits.	Santiago, with ORNL integration support	Santiago estimated hardware testing could be possible within the next couple of weeks; dependent on pulse and device integration