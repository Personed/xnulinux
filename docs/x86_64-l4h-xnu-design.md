# x86_64 L4H/FKI/XNU initial design

## Status and scope

Validation-only initial design. This document turns the owner-accepted L4H/seL4 direction into a bounded first implementation architecture, but it does not authorize or implement executable code. It is not a production-safety claim.

The upstream evidence pin is the same as `docs/x86_64-l4h-enabler-assessment.md`: `losfair/l4h` commit `69c0a8d24d2c0e6d830ff515b79f71c28baac9a9`, consulted on 2026-11-28. This document does not duplicate that assessment; it designs around its open obligations.

## Non-negotiable upstream constraint

The pinned L4H design states at [`DESIGN.md` line 833](https://github.com/losfair/l4h/blob/69c0a8d24d2c0e6d830ff515b79f71c28baac9a9/DESIGN.md#L833): **“No exception tables. A guest-kernel fault stops the guest”**. The memory-management section repeats at [line 2211](https://github.com/losfair/l4h/blob/69c0a8d24d2c0e6d830ff515b79f71c28baac9a9/DESIGN.md#L2211) that kernel mappings cannot be left to fault again because a guest-kernel fault stops the guest.

Therefore any design that executes XNU kernel code inside the Linux guest kernel's fault domain inherits a fatal-stop boundary unless a separate recoverable mediation mechanism is proven.

## Fault-mediation alternatives

| Alternative | Assessment | Consequence |
| --- | --- | --- |
| **A. XNU as a module inside the Linux guest kernel** | Blocked under current upstream evidence. XNU text/data would execute in the same guest-kernel fault domain as Linux. A bad XNU memory access, unintended privileged instruction, or corruption would be a guest-kernel fault and stop the whole guest. No evidence shows recoverable per-module fault isolation. | Not acceptable as the main architecture. At most a fatal-crash development shortcut, not even the first validation target. |
| **B. XNU as a separate seL4 execution domain supervised by Linux/FKI** | Best candidate. A fault in the XNU domain can be delivered as seL4 fault IPC to a supervisor instead of stopping the Linux guest. This matches the microkernel boundary that already mediates user tasks. However, current upstream L4H provider evidence demonstrates Linux user-task contexts, not a general foreign-kernel domain ABI. | Recommended first experimental direction, subject to validation of existing module-owned-context primitives and to the explicit alternative-E decision gate below. |
| **C. Modify `arch/l4h` for recoverable guest-kernel exceptions/exception tables** | Technically conceivable but not minimal. It changes the core Linux guest fault model, must handle recursive faults, corrupted kernel state, locking/preemption, SMP rollback, exception-table semantics, and Linux reliability assumptions. It also does not by itself distinguish XNU faults from Linux kernel faults. | Reject as first implementation path. Reconsider only if isolated domains prove impossible. |
| **D. Treat faults as fatal and rollback/restart the guest** | Useful for early debugging because it is simple and honest, but every XNU fault is a full guest failure. It cannot provide production reliability or meaningful fault recovery. | Development-only fallback, not production-safe. |
| **E. XNU as a hardware-assisted VM through KVM inside the L4H Linux guest** | Upstream documents `/dev/kvm` inside x86_64 guests: Linux KVM common code runs over seL4 VCPU objects on VT-x/AMD-V. A nested guest can execute ring-0 code with VM exits mediated through the KVM/seL4 boundary, so XNU faults/control operations need not become Linux guest-kernel faults. However, this depends on usable VT-x/SVM and upstream lists significant feature/performance limits. | Must be evaluated explicitly before freezing a final architecture; not the default first proof of the hosted-FKI mediation model. |

## Alternative E evidence and tradeoffs

Pinned upstream [`DESIGN.md` §5.10](https://github.com/losfair/l4h/blob/69c0a8d24d2c0e6d830ff515b79f71c28baac9a9/DESIGN.md#L991) states that, where the CPU has VT-x or AMD-V, the x86_64 L4H guest provides `/dev/kvm` for unmodified QEMU, Firecracker, and cloud-hypervisor. The guest cannot execute VMX/SVM instructions in ring 3; instead, `arch/l4h/kvm` implements `kvm_x86_ops` over seL4 VCPU objects. `KVM_RUN` enters through seL4, VM exits return register/control evidence to the Linux guest, notifications become external-interrupt exits, nested guest memory is mirrored into EPT/NPT leaves, and XSAVE/XCR0 state is mediated.

That evidence matters because alternative E is fundamentally different from alternative B:

| Dimension | B: isolated ring-3 XNU domain | E: hardware-assisted XNU VM |
| --- | --- | --- |
| Fault isolation | Requires proving the foreign-domain fault/resume contract; promising but unvalidated. | VM exits are an established isolation mechanism; XNU kernel faults need not enter the Linux guest kernel fault domain. |
| Ring-0 fidelity | Low: XNU must be deeply patched so privileged operations become explicit hostcalls. | High: XNU can execute ring-0 instructions inside the VM, subject to KVM/seL4 feature limits. |
| XNU patching | Very large and central to the architecture. | Potentially much smaller for core privileged execution, though device/ABI integration still requires work. |
| Hardware requirement | No VT-x/SVM requirement beyond what L4H itself needs. | Requires usable VT-x or AMD-V. This weakens the broad “any PC” objective, especially in nested/cloud environments where virtualization is unavailable. |
| Upstream feature limits | Ring-3 mediation depends on L4H user/fault primitives, not KVM feature parity. | Upstream says APICv/AVIC/posted interrupts, PML, nested VMX/SVM, vNMI, PMU, PKU, XSAVES, AVX-512, RDTSCP/RDPID, guest hardware breakpoints, speculation-control MSRs, SMM, and 5-level paging are unavailable or hidden; every VM exit is an seL4 call/return with additional VMCS access cost. |
| Linux/FKI integration | Direct: Linux/FKI can mediate individual user tasks and kernel hostcalls. | Indirect: Linux acts mostly as a VMM/VM supervisor. User processes inside XNU are not independently mediated as host user contexts. |
| SMP/timer/IPI | Must be designed over L4H notifications/scheduling contexts. | KVM supplies a vCPU model, but interrupt-controller and timer fidelity inherit the upstream feature limits. |
| Memory/rollback | Requires a new host-mediated pmap/copy/ownership design. | EPT/NPT memslot mirroring is existing upstream evidence, though rollback and XNU integration still need design. |
| Relation to project goal | Distinctive native-hosted/deep-paravirtualization experiment and does not duplicate the existing QEMU/KVM path. | Closer to the repository's complementary QEMU/KVM virtualization path, even though the outer host is L4H rather than stock Linux. |

## Bounded recommendation

Use **alternative B for the first experimental mediation proof**, not as a frozen final architecture: XNU must not run as ordinary module code inside the Linux guest kernel. In that experiment it should run as a separate logically privileged but architecturally unprivileged x86 execution domain, with Linux/FKI as supervisor and mediator.

The reasons B remains the first proof are bounded: it does not require VT-x/SVM, it directly exercises the host-mediated user-task/kernel-hostcall model that distinguishes this project from ordinary VM launching, and pinned evidence already shows module-owned execution contexts in the L4H architecture. Those are reasons to test the unique mediation model first, not evidence that deep paravirtualization is production-viable.

Alternative E must remain an explicit architecture candidate. Before stage 6 XNU integration or any decision freezing the final architecture, a separate small validation lane must assess KVM-inside-guest feasibility on the intended host profile: usable VT-x/SVM, required CPU-feature masking, vCPU/timer/IPI behavior, EPT/NPT memory semantics, rollback, performance cost, and whether the resulting architecture is materially different from the repository's complementary QEMU/KVM route. Until that comparison exists, this design does not choose a production architecture.

“Logically privileged” means XNU may implement kernel semantics and manage user tasks. “Architecturally unprivileged” means its instructions execute in seL4 user mode; privileged x86 operations are either patched into explicit hostcalls or rejected. This is a deep-paravirtualization design, not execution of unmodified ring-0 XNU.

The minimum upstream/L4H requirement before XNU is a **foreign-kernel domain primitive**: a way for Linux/FKI to create, enter, interrupt, inspect, resume, and destroy an isolated execution context whose faults are routed to a supervisor endpoint, with controlled sharing of memory and capabilities.

Pinned upstream evidence narrows but does not eliminate that requirement. `arch/l4h/kernel/uctx.c` describes `l4h_uctx_alloc_badge()` as being “for a module's own user contexts”; `arch/l4h/kernel/ksyms.c` exports object allocation, CNode mint/copy/delete, TCB configure/set-space/sched/flags/suspend/resume, user-run/register/FPU operations, VSpace mapping, timer, signal, and IPI primitives. The repository's existing AArch64 `fki_l4h.c` already uses those primitives to allocate a module-owned TCB/reply object, mint a badged fault endpoint, enable syscall trapping, and run the context through `l4h_user_run_ex`.

Therefore the first synthetic x86 domain probe may not require an upstream L4H feature addition if the same primitives behave correctly on x86_64. What remains unproven is the foreign-kernel-domain contract around them: independent VSpace composition, controlled memory sharing, interruption priority/scheduling, full x86 register/FPU control, repairable fault resume, cleanup, and the semantic distinction between user tasks and the XNU domain. Stage 1 must first validate or reject reuse of these existing primitives before requesting any upstream modification.

## Target architecture for the MVP

1. **Linux/FKI supervisor** runs inside the paravirtualized Linux guest. It owns policy, XNU loading/orchestration, user address-space mediation, timers/IPIs, rollback, and diagnostics.
2. **XNU kernel domain** is a separate seL4 execution context/VSpace. It contains XNU kernel text/data and only explicitly shared memory. It does not share the Linux guest kernel fault domain.
3. **Foreign user tasks** remain separate user execution contexts/address spaces, conceptually similar to upstream L4H uTCBs but governed by Darwin/XNU semantics rather than Linux syscall semantics.
4. **Supervisor endpoints** receive XNU-domain faults, user-task syscalls/faults, timer notifications, and explicit hostcalls with distinct badges so the mediator never guesses their origin.
5. **Hostcalls are explicit ABI**, not accidental privileged instructions. An unmediated privileged or invalid instruction in the XNU domain is a stop event for development diagnosis and a fatal XNU-domain error in production; it must never stop Linux.
6. **Rollback is exact-once**: every context, mapping, frame-cap copy, endpoint, timer, and shared-memory registration has an owner, generation, and destruction order.

## Minimum x86 ABI for the first implementation

This is an initial experimental ABI contract, not a stable public ABI.

### User-task register record

Adopt the x86_64 Linux `struct user_regs_struct` word order used by upstream `include/l4h/provider-registers.h` for the general-purpose subset:

`r15,r14,r13,r12,rbp,rbx,r11,r10,r9,r8,rax,rcx,rdx,rsi,rdi,orig_rax,rip,cs,rflags,rsp,ss,fs_base,gs_base`.

MVP requirements:

- preserve all GPRs, `rip`, `rsp`, `rflags`, `fs_base`, and `gs_base` exactly across stop/resume;
- `cs` and `ss` are expected fixed user selectors but must be reported/reserved rather than assumed away;
- segment selector fields `ds/es/fs/gs` are out of MVP except as reserved zero/ABI-extension fields;
- the mediator must choose one canonical syscall restart representation: the address of the `syscall` instruction, the following RIP, `rcx`, `r11`, and `rflags` must not be interpreted differently by producer and consumer;
- a fast resume may use syscall-reply semantics only when `rcx` and `r11` reconstruct the architecturally required `sysretq` state; otherwise it must use full-register restoration.

### XSAVE/FPU state

- MVP treats FPU/XSAVE as an opaque variable-size host blob accessed only through the supervisor.
- Initial supported subset: x87/SSE plus AVX state when exposed by the host XCR0 policy. No AVX-512, MPX, PKU, AMX, or supervisor-state claims in MVP.
- The record must carry size, alignment, feature mask, and generation. XSAVE state is synchronized at user-context stop/resume boundaries and when XNU explicitly reads or writes thread state.
- Production requires dynamic XCR0/XSAVE sizing, artifact-specific CPU-feature policy, and exact signal/thread-state serialization. QEMU TCG profiles are not evidence of physical CPU support.

### Syscall classification and stop taxonomy

Pinned XNU evidence for the x86_64 userspace syscall contract is tag `xnu-12377.121.6` (checkout commit `ac9718fb1af618d5ce8678d0dc6e8a58f252216f`), especially [`osfmk/mach/i386/syscall_sw.h`](https://github.com/apple-oss-distributions/xnu/blob/xnu-12377.121.6/osfmk/mach/i386/syscall_sw.h) and [`libsyscall/custom/SYS.h`](https://github.com/apple-oss-distributions/xnu/blob/xnu-12377.121.6/libsyscall/custom/SYS.h):

- the 64-bit syscall number uses the low 32 bits; class occupies bits 24–31, and the pinned header says the high-order 32 bits are unused;
- classes are Mach=1, Unix/BSD=2, machine-dependent=3, diagnostics=4, and Mach IPC=5;
- x86_64 BSD wrappers move `rcx` to `r10` before `syscall` because the architectural instruction clobbers `rcx`;
- x86_64 BSD wrappers test the carry flag after return: carry clear is success, carry set treats `rax` as the error value passed to the userspace error path;
- Mach call wrappers list negative trap numbers, but the pinned x86_64 wrapper macro negates that input when constructing the class-1 value. The decoder must therefore treat the low syscall-number bits as the positive trap magnitude, not as a negative encoded trap number;
- pinned [`bsd/dev/i386/systemcalls.c`](https://github.com/apple-oss-distributions/xnu/blob/xnu-12377.121.6/bsd/dev/i386/systemcalls.c) writes BSD return outputs into the saved user registers according to the syscall return type: integer/pair returns use the two return slots, while single 64-bit-style return types zero `rdx` on success.

For the MVP register contract, `rax` low 32 bits carry the encoded syscall number, arguments follow the wrapper-visible order `rdi,rsi,rdx,r10,r8,r9`, and `rcx/r11` are architecturally clobbered by `syscall`. BSD success/error uses the pinned carry-flag rule above. The mediator must write the syscall's output register values rather than restore input argument registers: `rax` receives the return or error value, and `rdx` receives the pinned second return output for the syscall/return type. In particular, successful single 64-bit-style returns such as read/write-size results clear `rdx` rather than preserving the original third argument.

The stop record must carry scope, kind, register generation, fault vector/code, fault address, fault IP, and a timestamp/sequence number. It must never collapse Darwin and Linux namespaces.

Minimum user-task stop kinds:

1. `BSD_SYSCALL`: `syscall` with Darwin BSD class encoding. The MVP recognizes class 2 in bits 24–31 of `rax`; BSD write is therefore encoded as `0x02000004`. Other BSD numbers remain data for the mediator, not Linux syscalls.
2. `FOREIGN_SYSCALL`: Mach, machine-dependent, diagnostic, unknown-class, or malformed Darwin syscall encoding. MVP may fail closed; production needs full pinned-XNU class decoding.
3. `PAGEFAULT`: address, access type, present/protection information when available, fault IP, and raw architecture fault code.
4. `USER_EXCEPTION`: vector/error-code plus full register record.
5. `INTERRUPT`: host event/preemption stop with user registers either synchronized explicitly or marked invalid.
6. `HALT`: controlled task/domain termination.

Minimum XNU-domain stop kinds:

1. `HOSTCALL`: explicit, validated ABI request.
2. `KERNEL_PAGEFAULT`: fault inside the XNU domain, isolated from Linux.
3. `KERNEL_EXCEPTION`: vector/error-code inside the XNU domain.
4. `UNMEDIATED_PRIVILEGED_OP`: development-diagnostic stop for an instruction that should have been patched or hostcalled.
5. `TIMER`, `IPI`, `SHUTDOWN`, and `PANIC` as separate stops, never overloaded onto user syscall kinds.

Production requires exact Darwin BSD/Mach/machine-dependent decoding and restart rules from pinned XNU sources. The MVP may implement only synthetic probes plus BSD write/read/exit-like behavior needed for validation.

### Address spaces and user memory

MVP operations:

- `as_create`, `as_destroy`, `as_map`, `as_unmap`, `as_protect`, `as_flush`;
- 4 KiB pages only, no huge pages, no nested/shared page tables, no execute-only claim;
- mappings are explicit host-mediated frame-cap copies or equivalents, with generation/ownership tracking;
- XNU kernel memory and user memory are in separate VSpaces; no implicit dual mapping;
- user-memory copyin/copyout is performed by the supervisor against validated VMA/pmap state, using pinned frames or equivalent revocation protection. No speculative dereference of user pointers by the XNU domain.

Production additionally requires shared/nested memory objects, large pages where appropriate, exact protection semantics, eviction under memory pressure, copy atomicity, fault retry behavior, wired memory, and cross-CPU TLB invalidation.

### Timer, IPI, SMP, and cleanup

- MVP: one XNU virtual CPU/domain, one-shot host timer event, no SMP and no cross-vCPU IPI.
- Production: per-vCPU contexts, interrupt/preempt routing, IPI semantics, CPU start/stop, idle behavior, priority inversion/budget analysis, and rollback across partial SMP startup.
- Every cleanup path must be idempotent or generation-checked. Destroying the XNU domain must not leak or revoke memory still referenced by user contexts; destruction order is user contexts, user address spaces, XNU domain, supervisor registrations.

## MVP versus production boundary

The MVP is valid only if it proves mediation mechanics. It may use synthetic code and synthetic Darwin-encoded syscall probes. It must not load a real XNU kernel or claim compatibility.

Production requires all of the following before support claims:

- recoverable XNU-domain fault mediation demonstrated for page faults and synchronous exceptions;
- complete pinned-XNU syscall decoding/restart semantics;
- real XNU machine-layer integration and shutdown/rollback;
- memory-pressure and eviction correctness;
- SMP/timer/IPI behavior;
- hardware acceptance separately on Intel and AMD;
- upstream L4H/seL4 security/maintenance review and pinned-version update process.

## Implementation plan in stages

1. **Provider/supervisor capability artifact**
   - Implement or verify the minimal foreign-kernel-domain primitive in L4H/provider APIs.
   - Exit criterion: a compile-time artifact exposes create/enter/stop/resume/inspect/destroy semantics for an isolated x86 execution domain, without XNU.

2. **QEMU/L4H boot harness**
   - Boot a pinned L4H/seL4 x86_64 Linux guest in QEMU with the supervisor module present.
   - Exit criterion: reproducible boot, version/hash reporting, module load/unload, and clean host rollback. This proves harness mechanics only, not XNU.

3. **Synthetic kernel-domain fault probe**
   - Run controlled x86 code in the isolated domain; trigger explicit hostcall, controlled page fault, and controlled synchronous exception.
   - Include one repairable page fault: initially leave a data page unmapped, take the fault, map the page through the supervisor, resume the interrupted instruction/stream, and require the probe to execute a post-fault completion marker. Merely terminating the domain proves isolation but not recovery.
   - Exit criterion: supervisor receives correctly badged stops with full registers, completes the repairable-fault resume path, can still terminate the domain after an unrecoverable exception, and Linux continues running.

4. **Synthetic user-task mediation probe**
   - Execute a Darwin-encoded safe synthetic syscall such as the already established `0x02000004` encoding, plus controlled page fault and exception fixtures.
   - Exercise the MVP memory operations without XNU: map, unmap, protection change, flush/invalidation, copyin/copyout success, copyin/copyout fault, and revocation/cleanup races that are representable in the harness.
   - Exercise at least one one-shot host timer stop before ABI freeze: arm it, observe correct preemption/stop ordering, resume or clean up the interrupted context, and verify that the timer event cannot be confused with a syscall or fault.
   - Exit criterion: correct Darwin-vs-Linux classification, full register/XSAVE preservation, restart/resume behavior, memory/timer semantics, and cleanup.

5. **XNU machine ABI adapter design freeze**
   - Only after probes pass, freeze an x86-specific replacement/parallel contract for the current AArch64 LINUX64 ABI concepts.
   - Exit criterion: reviewed header/design with version handshake, complete ownership rules, and fail-closed behavior.

6. **Initial XNU integration**
   - Load the smallest XNU build artifact into the isolated domain with patched hostcalls and no physical-device claims.
   - Exit criterion: deterministic entry, explicit panic/hostcall path, controlled shutdown, and no Linux-guest stop on synthetic XNU faults.

7. **Iterative XNU runtime validation**
   - Add memory objects, user tasks, timers, interrupts, SMP, and real artifact boot stages one at a time.
   - Exit criterion per stage: reproducible tests, rollback, diagnostics, and updated conservative support claims.

## Decision requested by this design

The first implementation lane should be stage 1 plus a non-XNU synthetic design artifact only if it can remain small: define the foreign-kernel-domain primitive and its exact owner/rollback contract, explicitly starting from the existing module-owned-context primitives rather than assuming a new upstream feature. If validation shows they cannot express the required isolated domain without major redesign, the correct next output is a revised architecture decision rather than XNU code.
