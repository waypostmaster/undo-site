# UNDO 1.0.0 — Engineering Deliverable

**Astra, Principal Engineer**  
29 October 2026

This document contains the implementation and proposed release materials. Tests have not been executed in this environment. Performance figures are targets, and historical release records and customer-question provenance remain unverified.

## 1. Product requirements document

### Product definition

`undo` attempts to reverse effects attributable to its own invocation.

The full acceptance condition is:

> After termination, the system must be indistinguishable from the counterfactual system in which the invocation did not occur, including records and observations of the invocation.

That condition remains the product requirement. It is **not achievable by the supplied implementation on ordinary operating systems**. Effects can occur before program code begins, and independent observers can retain information outside the program’s control.

Version 1.0.0 therefore implements a bounded restoration contract and reports incomplete restoration explicitly. Approval as a product satisfying the full acceptance condition is blocked.

### Goals

1. Minimize effects introduced by execution.
2. Reverse changes for which the program possesses both a reliable before-state and sufficient ownership.
3. Release resources acquired by the program.
4. Identify residual effects without representing partial cleanup as complete restoration.
5. Give operators stable, documented command behavior.
6. Preserve the integrity of evidence used to evaluate the product.

### Non-goals for 1.0.0

These implementation exclusions remain limitations against the full product requirement:

- Elevating privileges to change system records.
- Editing shared shell history, audit records, or another process’s memory.
- Deleting the executable or its installation.
- Reconstructing a pre-invocation state from unverified assumptions.
- Providing a supported library interface inside another application.
- Claiming forensic erasure of released memory.

### User stories

| Persona | User story | Acceptance criterion |
|---|---|---|
| Application developer | I need a predictable command that leaves no application-created files. | A default invocation requests no persistent filesystem mutation after application entry. |
| Automation engineer | I need to distinguish incomplete restoration from operational failure. | Exit statuses distinguish those outcomes; status `3` is documented as terminal, not a retry request. |
| Systems administrator | I need execution to respect existing ownership boundaries. | No privilege escalation, external command execution, or alteration of shared records. |
| Enterprise buyer | I need a precise statement of what procurement would be accepting. | The bounded contract, unsupported guarantees, and validation status are disclosed together. |
| Auditor | I need evidence that cleanup occurred without destruction of audit evidence. | Tests observe resource release independently; the implementation does not edit audit records. |
| Privacy-conscious operator | I need to avoid unnecessary output and disclosure. | Default execution emits no application output; diagnostic output requires a flag or invalid command. |
| Incident responder | I need repeated invocations to preserve investigative evidence. | A subsequent invocation cannot consume or erase a previous invocation’s records. |

### Functional requirements

| ID | Requirement | 1.0.0 disposition |
|---|---|---|
| FR-01 | Restore the complete counterfactual state. | Unmet; normal execution returns `3`. |
| FR-02 | Release descriptors acquired for application output. | Implemented using `finally` cleanup. |
| FR-03 | Restore the application’s temporary bytecode-writing policy. | Implemented using a saved value and `finally` cleanup. |
| FR-04 | Terminate and relinquish process-owned resources. | Relies on operating-system process teardown. |
| FR-05 | Support `--dry-run`, `--verbose`, `--version`, and `--help`. | Implemented. |
| FR-06 | Avoid application-created persistent state in default execution. | Implemented by design; syscall verification pending. |
| FR-07 | Document effects that cannot be reversed. | Covered by the limitation register below. |
| FR-08 | Never return a restoration-success indication for incomplete restoration. | Default and dry-run execution return `3`. |

### Success metrics

These are release acceptance targets, not measured results.

| Metric | Target | Measurement |
|---|---:|---|
| Default application output | 0 bytes | Capture both standard streams. |
| Application-requested persistent filesystem mutations in default mode | 0 | Trace filesystem operations after application entry under isolated startup. |
| Application network operations and child-process launches | 0 | Trace network and process-creation operations. |
| Successful owned-descriptor cleanup under ordinary execution | 100% | Descriptor spies and subprocess tests. |
| Cleanup attempted after an injected output failure | 100% of specified failure cases | Fault injection in the output helper. |
| Restoration of the temporary bytecode policy on handled exits | 100% | Test both initial policy values. |
| Quiet-run reliability campaign | 10,000 consecutive invocations without an application cleanup error | Controlled integration environment. |
| False claims of full restoration | 0 | Review exit codes, output, documentation, and tests. |
| Certified platforms satisfying the full counterfactual requirement | None currently | Requires evidence beyond the bounded tests. |

### Known limitations

Severity describes operational or assurance impact. **Any residual effect can defeat the full acceptance condition, regardless of its individual severity.**

- **Critical:** prevents the full restoration guarantee.
- **High:** can retain durable evidence or information outside program control.
- **Medium:** creates a bounded observable difference requiring operational handling.

| ID | Effect that cannot be fully undone | Severity | Mitigation |
|---|---|---|---|
| L-01 | Effects preceding application entry: shell parsing, process creation, executable loading, interpreter startup. | Critical | Use a qualified launch environment. A prearranged snapshot can cover selected state, but cannot erase all external observations. |
| L-02 | CPU consumption, scheduling interference, cache activity, energy use, and thermal effects. | Critical | Minimize execution. Isolation reduces interference; no exact reversal is claimed. |
| L-03 | PID allocation, process accounting, resource counters, parent wait results, and related kernel state. | High | Account for these as residual effects. Avoid automatic retries. |
| L-04 | Shell history, in-memory command history, `$?`, job notifications, and shell integrations. | High | Where policy permits, configure history behavior before invocation. This command cannot reliably restore its parent shell. |
| L-05 | File access times and metadata effects from loading the script, interpreter, libraries, or configuration. | Medium | Use an appropriately prepared read-only or `noatime` environment. Do not guess original timestamps. |
| L-06 | Local audit records, security telemetry, process monitors, and remote collectors. | High | Preserve these records and include them in evaluation. Configure collection through approved administration, not through this command. |
| L-07 | Requested output, terminal scrollback, session recordings, redirected files, pipe consumers, and changed output-file offsets. | High | Use default quiet execution when output is unnecessary. Requested output remains observable. |
| L-08 | A person’s memory of invoking or observing the command. | Critical | No technical reversal is offered. Unattended execution can reduce exposure but does not establish full restoration. |
| L-09 | Released memory contents, swap, crash dumps, allocator state, and forensic residue. | High | Apply appropriate host memory, swap, and dump policies before execution. Releasing an allocation is not erasing its bytes. |
| L-10 | Cleanup interrupted by uncatchable signals, interpreter failure, or machine failure. | High | Keep operations small and avoid persistent application state. The OS normally releases descriptors on process termination; output already delivered remains. |
| L-11 | Concurrent activity that prevents attribution or changes the counterfactual execution. | Critical | Use controlled experiments and record concurrency. Do not infer ownership from timing alone. |
| L-12 | Interpreter hooks or startup effects when launched without isolation, including possible startup writes. | High | Prefer `python3 -I -S -B ./undo` with a trusted interpreter. |
| L-13 | Installation and download records, package databases, backups, and remote transfer records. | High | Treat these as separate lifecycle effects. Version 1.0.0 does not reverse installation or download. |
| L-14 | Environment-specific instrumentation or effects outside the inventory. | High | Qualify each deployment environment. An incomplete inventory cannot support full certification. |

## 2. Architecture

### Execution model

The supported execution model is a standalone, unprivileged CPython process on a POSIX operating system.

The implementation contains four components:

1. **Command parser:** accepts a small, fixed flag vocabulary.
2. **Effect policy:** permits process-local cleanup and requested output; provides no external-state restoration mechanism.
3. **Owned-resource cleanup:** closes output descriptors acquired by the program and restores its temporary bytecode policy.
4. **Outcome reporter:** distinguishes informational success, operational error, usage error, and incomplete restoration.

There is no persistent journal, daemon, configuration file, network client, or subprocess runner.

### Undo model

An effect is any difference causally attributable to invocation, including a difference retained by another observer.

An inverse is eligible only when:

- Its before-state is known.
- The affected resource belongs to this invocation.
- The operation can be performed without changing another principal’s state or destroying shared evidence.

The implementation captures before-state only after its code begins. That baseline cannot establish the state before invocation.

| Surface | Before-state available? | Implemented action | Remaining difference |
|---|---|---|---|
| Temporary `sys.dont_write_bytecode` setting | Yes, immediately before modification | Restore the saved value before returning | Interpreter startup occurred earlier. |
| Descriptor duplicated for requested output | Yes; this invocation creates the duplicate | Close that descriptor in `finally` | Delivered bytes and shared output offsets remain. |
| Process address space and descriptor references | Managed by OS | Exit and allow kernel teardown | Accounting, allocation history, and memory residue remain. |
| Process accounting | No authoritative pre-invocation baseline | None | Counters and records remain. |
| Shell history and parent-shell state | Not accessible through a reliable child-process inverse | None | Invocation and exit status may remain. |
| Filesystem atime | Generally changed before application entry | None | Access metadata may remain changed. |
| Terminal scrollback | No complete or exclusive ownership | Default silence; no erasure | Requested output and external captures remain. |
| Audit logs | Owned by auditing authorities | None | Evidence remains intact. |
| User memory | No supported inverse | None | Awareness may remain. |

Closing a duplicated descriptor restores only that descriptor-table ownership. It does not rewind the underlying file, retract pipe data, or clear a terminal.

### Sequence diagram

```text
User          Shell          Kernel/Interpreter        undo          Observer
 |              |                    |                   |               |
 |-- invoke --->|                    |                   |               |
 |              |-- record/parse --->|                   |               |
 |              |-- create process ->|------------------------------->   |
 |              |                    |-- load code ----->|               |
 |              |                    |                   |-- save policy |
 |              |                    |                   |-- disable new |
 |              |                    |                   |   bytecode writes
 |              |                    |                   |-- parse flags |
 |              |                    |                   |               |
 |              |                    |                   | [if output requested]
 |              |                    |                   |-- dup descriptor
 |              |                    |                   |-- write ------>|
 |              |                    |                   |-- close duplicate
 |              |                    |                   |               |
 |              |                    |                   |-- restore policy
 |              |                    |<-- exit status ---|               |
 |              |                    |-- release process resources       |
 |              |<-- wait result ----|                   |               |
 |              |                    |-- accounting/audit -------------> |
 |<-- prompt ---|                    |                   |               |
 |              |                    |                   |     evidence remains
```

### Architecture decision records

The following decisions are adopted in the supplied source. They do not constitute client acceptance of the unmet full requirement.

#### ADR-001 — Keep the full acceptance condition explicit

**Status:** Adopted.

**Context:** A bounded cleanup result does not establish counterfactual equivalence.

**Decision:** Normal execution returns `3`, meaning restoration remains incomplete.

**Consequences:** Automation must handle `3` explicitly. A conventional zero exit status cannot be mistaken for a restoration certificate.

#### ADR-002 — Restrict inverses to resources owned by this invocation

**Status:** Adopted.

**Context:** External state can be shared, concurrently modified, or missing a reliable baseline.

**Decision:** Restore the temporary interpreter policy and close acquired output descriptors. Do not infer external inverses.

**Consequences:** The implementation is small and reviewable, while significant residual effects remain documented.

#### ADR-003 — Preserve shell history and audit evidence

**Status:** Adopted.

**Context:** History files and audit records may contain concurrent activity and evidence belonging to other actors.

**Decision:** Do not edit them.

**Consequences:** Invocation evidence remains. The full acceptance condition remains unmet on systems retaining that evidence.

#### ADR-004 — Do not reconstruct filesystem timestamps

**Status:** Adopted.

**Context:** The original atime may be lost before program entry. Setting atime can itself change ctime and generate further records.

**Decision:** Make no timestamp-restoration attempt.

**Consequences:** Prepared filesystem policies can reduce metadata effects, but ordinary deployments retain L-05.

#### ADR-005 — Use one Python file

**Status:** Adopted.

**Context:** The deliverable requires a complete single-file implementation.

**Decision:** Use Python with only `sys` and `os`, fixed messages, and explicit cleanup.

**Consequences:** Interpreter startup adds observable effects. Isolated startup is recommended, and interpreter provenance is part of deployment qualification.

#### ADR-006 — Remain quiet by default

**Status:** Adopted.

**Context:** Output can be retained by terminals, files, and independent consumers.

**Decision:** Emit no application output during a valid default invocation. Flags provide explicit informational output.

**Consequences:** Operators must inspect the exit status. Requested output cannot be withdrawn reliably.

#### ADR-007 — Define dry-run as a preview with its own footprint

**Status:** Adopted.

**Context:** Producing a preview requires execution and output.

**Decision:** `--dry-run` describes the planned behavior, performs ordinary cleanup of its own resources, and returns `3`.

**Consequences:** Dry-run is not evidence-free and is not a restoration-success mode.

#### ADR-008 — Keep the installed executable

**Status:** Adopted.

**Context:** The executable normally predates the invocation. Deleting it would introduce an additional persistent change.

**Decision:** Do not self-delete.

**Consequences:** Installation and download reversal require a separate lifecycle design.

## 3. Full implementation

Save the following single file as `undo`.

The explicit launch form is:

```sh
python3 -I -S -B ./undo
python3 -I -S -B ./undo --dry-run
python3 -I -S -B ./undo --verbose
```

The shebang supports direct execution on systems with CPython at `/usr/bin/python3`. The explicit form avoids dependence on that location.

### Exit statuses

| Status | Meaning |
|---:|---|
| `0` | `--help` or `--version` completed successfully; no restoration certification. |
| `1` | Output, cleanup, or another handled operational error. |
| `2` | Unsupported or conflicting arguments. |
| `3` | Execution completed with known residual effects. |
| `130` | A Python `KeyboardInterrupt` was handled. |

Other process termination signals follow operating-system behavior.

```python
#!/usr/bin/python3 -ISB
"""
UNDO 1.0.0.

A standalone command that minimizes execution effects and reverses the
process-local changes it can identify and own.

Normal execution returns 3 because full counterfactual restoration is
unavailable. Run with a trusted CPython interpreter:

    python3 -I -S -B ./undo

This module exposes functions for testing. Embedded use is not a supported
product mode because process teardown is part of the cleanup contract.
"""

import sys


VERSION = "1.0.0"

EXIT_INFO = 0
EXIT_ERROR = 1
EXIT_USAGE = 2
EXIT_INCOMPLETE = 3
EXIT_INTERRUPT = 130

HELP_TEXT = """\
Usage: undo [--dry-run] [--verbose]
       undo --help
       undo --version

Attempt ownership-scoped cleanup of this invocation's resources.

Options:
  --dry-run   Describe planned behavior and remaining effects.
  --verbose   Report scope and limitations; extends a dry-run report.
  --version   Print the version.
  --help      Print this help.

--help, --version, and --dry-run are mutually exclusive.
Repeated copies of a recognized flag are accepted.

Default execution produces no application output.

Exit status:
  0    Help or version output completed; not restoration certification.
  1    Operational or cleanup error.
  2    Unsupported or conflicting arguments.
  3    Known residual effects remain.
  130  Keyboard interruption handled by the application.

Full restoration is unavailable. Invocation records, system activity,
metadata effects, observations, and requested output may remain.

Recommended launch:
  python3 -I -S -B ./undo
"""

DRY_RUN_TEXT = """\
undo 1.0.0: dry run

Planned behavior:
  - Request no persistent filesystem mutation.
  - Launch no child process and perform no network operation.
  - Restore the temporary bytecode-writing policy before returning.
  - Close the descriptor acquired to deliver this report.
  - Terminate and let the operating system release process resources.

No external restoration receipt or pre-invocation snapshot is accepted
by this build. No external-state inverse will be attempted.

This preview has already caused execution and output effects.
Restoration remains incomplete. Expected exit status: 3.
Output or cleanup errors take precedence.
"""

DETAIL_TEXT = """\
undo 1.0.0: restoration scope

Application-owned cleanup:
  - Close any descriptor duplicated for requested output.
  - Restore the saved bytecode-writing policy before returning.
  - Relinquish process resources through operating-system teardown.

Known residual effects include:
  - Process creation, accounting, scheduling, and physical activity.
  - Shell history, parent-shell status, and invocation records.
  - File access metadata and interpreter-startup effects.
  - Audit logs, monitoring records, and remote observations.
  - Requested output, output-file offsets, and terminal recordings.
  - Human memory and forensic memory residue.

Cleanup of this output descriptor occurs after this report is written.
Restoration remains incomplete. Expected exit status: 3.
Output or cleanup errors take precedence.
"""

USAGE_ERROR_TEXT = (
    "undo: unsupported or conflicting arguments; use --help.\n"
)


def parse_options(arguments):
    """
    Validate command-line arguments and return (mode, verbose).

    The accepted vocabulary is fixed. Unknown arguments are never echoed,
    preventing untrusted argument text from reaching terminal diagnostics.
    Duplicate recognized flags are idempotent.
    """
    allowed = {"--dry-run", "--verbose", "--version", "--help"}
    selected = set()

    for argument in arguments:
        if argument not in allowed:
            raise ValueError("unsupported argument")
        selected.add(argument)

    actions = selected.intersection(
        {"--dry-run", "--version", "--help"}
    )
    if len(actions) > 1:
        raise ValueError("conflicting arguments")

    mode = "run"
    if actions:
        mode = next(iter(actions))[2:]

    return mode, "--verbose" in selected


def write_owned(os_api, destination, message):
    """
    Write fixed ASCII text through an owned duplicate of a standard FD.

    Python's os.dup returns a non-inheritable descriptor. The duplicate
    shares its underlying destination and file offset with the original.
    Closing it reverses descriptor acquisition only; it does not retract
    delivered output or restore the shared offset.

    Short writes and interrupted writes are handled. Cleanup is attempted
    in finally. A failed close is not retried because descriptor ownership
    after a close error can be platform-dependent.
    """
    descriptor = os_api.dup(destination)
    try:
        payload = message.encode("ascii")
        position = 0

        while position < len(payload):
            try:
                written = os_api.write(
                    descriptor, payload[position:]
                )
            except InterruptedError:
                continue

            if written <= 0:
                raise OSError("output made no progress")

            position += written
    finally:
        os_api.close(descriptor)


def run(arguments, os_api):
    """
    Execute the selected command and return its documented exit status.

    A valid quiet invocation performs no explicit output operation.
    Informational commands return zero only when output cleanup succeeds.
    Output and descriptor-close errors return one.
    """
    try:
        mode, verbose = parse_options(arguments)
    except ValueError:
        try:
            write_owned(os_api, 2, USAGE_ERROR_TEXT)
        except OSError:
            return EXIT_ERROR
        return EXIT_USAGE

    if mode == "help":
        destination = 1
        message = HELP_TEXT
        status = EXIT_INFO
    elif mode == "version":
        destination = 1
        message = "undo " + VERSION + "\n"
        status = EXIT_INFO
    elif mode == "dry-run":
        destination = 1
        message = DRY_RUN_TEXT
        if verbose:
            message += "\n" + DETAIL_TEXT
        status = EXIT_INCOMPLETE
    elif verbose:
        destination = 2
        message = DETAIL_TEXT
        status = EXIT_INCOMPLETE
    else:
        return EXIT_INCOMPLETE

    try:
        write_owned(os_api, destination, message)
    except OSError:
        return EXIT_ERROR

    return status


def main(arguments=None):
    """
    Run the standalone application with a reversible bytecode policy.

    The previous policy is captured before modification and restored on
    handled exits. Disabling bytecode writes here covers subsequent normal
    imports, not interpreter activity that preceded application entry.

    Unexpected ordinary exceptions produce status one without a traceback.
    Uncatchable termination can bypass Python cleanup; process teardown
    remains the operating system's responsibility.
    """
    previous_policy = sys.dont_write_bytecode
    status = EXIT_ERROR

    try:
        sys.dont_write_bytecode = True

        import os

        actual_arguments = (
            sys.argv[1:] if arguments is None else list(arguments)
        )
        status = run(actual_arguments, os)
    except KeyboardInterrupt:
        status = EXIT_INTERRUPT
    except Exception:
        status = EXIT_ERROR
    finally:
        try:
            sys.dont_write_bytecode = previous_policy
            if sys.dont_write_bytecode is not previous_policy:
                status = EXIT_ERROR
        except BaseException:
            status = EXIT_ERROR

    return status


if __name__ == "__main__":
    raise SystemExit(main())
```

### Implementation boundaries

The source actively attempts both inverses it claims: descriptor closure and restoration of the temporary interpreter setting. Process teardown is delegated explicitly to the operating system.

It does not attempt history editing, timestamp reconstruction, terminal erasure, audit deletion, or memory erasure. Those capabilities are not implied by a successful cleanup of application-owned resources.

## 4. Threat model

### Assets and trust boundaries

Protected assets include unrelated files, audit evidence, parent-shell state, concurrent users’ activity, output destinations, and the credibility of restoration claims.

The principal boundaries are between the command and its caller, operating system, interpreter, output consumers, monitoring systems, and distribution channel.

The deployment assumes a trusted interpreter and operating system. It does not assume that external observers cooperate with restoration.

### STRIDE analysis

| Threat | STRIDE | Scenario | Control | Residual risk |
|---|---|---|---|---|
| False restoration certificate | Spoofing | A wrapper presents `--version` returning `0` as proof of restoration. | Document `0` as informational only; normal execution returns `3`. Evaluators inspect arguments and results together. | A malicious wrapper can misrepresent any program. |
| History destruction | Tampering | An implementation removes the last history line and deletes another session’s command. | No history editing or line-count heuristics. | The invocation remains recorded. |
| Audit suppression | Tampering / Repudiation | A caller uses the product to conceal execution from an auditor. | No audit editing, privilege escalation, or remote-log access. | A privileged attacker may alter records outside this program. |
| Terminal injection | Spoofing / Information disclosure | An attacker supplies control characters in invalid arguments. | Fixed error text; unknown arguments are not echoed. | The invoking shell may independently display those arguments. |
| **Double invocation** | Denial of service / Repudiation | An attacker runs `undo` twice, expecting the second run to erase the first or consume its cleanup state. | No shared journal or cross-invocation deletion. Each invocation owns only its resources. | Both invocations create residual effects; repeated runs can increase accounting and log volume. |
| Output sink exhaustion | Denial of service | Requested output targets a blocked pipe or slow terminal. | Default execution is quiet; reports are bounded. Operators may impose an external timeout. | A blocked output call can delay completion; timeout supervision also creates records. |
| Destructive path substitution | Tampering / Elevation of privilege | A path changes between inspection and deletion. | No external path restoration or deletion operations exist. | Source-file substitution before execution remains a supply-chain risk. |
| **Compromised distribution or interpreter** | Tampering / Elevation of privilege | A modified download, interpreter, or startup hook executes additional code. | Verify distribution integrity and interpreter provenance; use isolated startup. These are deployment controls, not a self-verifier in this source. | A compromised trusted computing base defeats the contract. |
| Excess privilege | Elevation of privilege | An operator executes the command as an administrator. | No elevated privileges are required; deploy with least privilege. | Compromised code would inherit the caller’s authority. |
| Retained sensitive output | Information disclosure | Diagnostic output is stored in a session recorder or redirected file. | Quiet default; fixed reports omit paths, usernames, environment values, and exception details. | Invocation metadata and requested output remain observable. |
| Ambiguous descriptor cleanup | Tampering / Denial of service | Retrying a failed close accidentally closes a reused descriptor. | Do not retry close; return an operational error. | Kernel teardown is required to release any surviving reference. |
| Forced interruption | Denial of service | A signal or machine failure interrupts cleanup. | Small execution path, `finally` cleanup for ordinary exceptions, OS resource release on termination. | Python-level restoration may not run; delivered output remains. |
| Observer outside the host | Information disclosure | An external monitor records execution before cleanup completes. | Disclose the boundary and qualify monitoring arrangements. | The command cannot retract that observation. |

## 5. Test plan

### Operational meaning of “the same state”

The full comparison is:

\[
S_{\text{invoked}}(T) = S_{\text{control}}(T)
\]

Here, \(T\) is the same externally defined observation point, and the two executions have matched initial conditions and controlled external inputs. Time also advances in the control experiment; elapsed wall time alone is not treated as proof of a difference.

The state inventory includes:

- Filesystem contents, directory membership, and metadata.
- Shell state and history.
- Process and resource accounting.
- Audit and monitoring records.
- Terminal and output-consumer state.
- Retained information held by independent observers.
- Physical effects where measurable.

The test suite cannot measure the whole universe or reproduce a perfect counterfactual. It therefore produces two separate results:

1. **Bounded implementation conformance:** whether the documented cleanup and no-mutation properties hold.
2. **Full-state comparison:** observed differences and unmeasured surfaces.

An unmeasured surface is reported as **unknown**, never equal. A known difference means the full acceptance condition fails.

### Unit tests

| ID | Test | Expected result |
|---|---|---|
| U-01 | Parse each supported flag and repeated copies. | Correct mode; repeated flags are idempotent. |
| U-02 | Combine conflicting action flags. | Usage error. |
| U-03 | Supply unknown arguments containing terminal control characters. | Fixed diagnostic; supplied bytes are absent from output. |
| U-04 | Execute default `run` with an OS spy. | Status `3`; no duplication, writing, or closing requested. |
| U-05 | Simulate short writes. | Entire report delivered in order; duplicate closed once. |
| U-06 | Inject `InterruptedError` during writing. | Writing resumes; duplicate eventually closed. |
| U-07 | Inject a broken pipe or another write error. | Closure attempted; status `1`. |
| U-08 | Inject failure in descriptor duplication. | Status `1`; no nonexistent descriptor is closed. |
| U-09 | Inject a close error. | Status `1`; close is not retried. |
| U-10 | Begin with bytecode writing enabled and disabled. | Original setting restored after handled completion. |
| U-11 | Inject an ordinary exception or `KeyboardInterrupt` during `run`. | Policy restored; status `1` or `130`, respectively. |
| U-12 | Exercise every valid execution mode. | Help/version alone may return `0`; execution and preview return `3`. |

### Integration tests

| ID | Test | Method and expected result |
|---|---|---|
| I-01 | Quiet execution | Capture both streams; expect zero bytes and status `3`. |
| I-02 | Filesystem mutation tracing | Run with `-I -S -B`; inspect application-phase operations for writes, creation, deletion, renaming, or metadata setters. Expect none in default mode. |
| I-03 | Bytecode artifacts | Use a disposable environment and inspect for application-created `.pyc` files and cache directories. Expect none under the qualified launch. |
| I-04 | Network and child processes | Trace the application process. Expect no application-created child or network operation. |
| I-05 | Descriptor release | Run report modes through pipes, then verify EOF and process completion. Use unit-level descriptor inspection to distinguish closure from process termination. |
| I-06 | Output failures | Connect reports to a closed reader and to an invalid output descriptor. Expect status `1`, without an application traceback. |
| I-07 | Repeated invocation | Run the reliability campaign. Expect no application-created persistent artifacts; accounting growth remains a documented difference. |
| I-08 | Metadata observation | Record relevant atime, mtime, and ctime before and after. Report actual differences, including startup effects. |
| I-09 | Audit-enabled execution | Observe records from outside the process. Expect records to remain and the full-state comparison to fail. |
| I-10 | Signal termination | Interrupt during output and terminate forcibly. Verify process removal and eventual descriptor release; do not require Python finalizers after forced termination. |
| I-11 | Shell state | Observe history behavior and `$?` in representative shells. Record differences rather than asserting restoration. |
| I-12 | Control comparison | Compare separately prepared control and invocation environments using an explicit measurement manifest. Every ignored or unmeasured field must be disclosed. |

Instrumentation introduces its own effects. The harness must record those effects and avoid presenting instrumented equality as proof of universal equality.

### Manual acceptance checklist

- [ ] Review the exact source against the documented behavior.
- [ ] Verify the interpreter and distribution provenance.
- [ ] Confirm quiet execution returns `3`.
- [ ] Confirm help and version output make no restoration claim.
- [ ] Confirm dry-run explains its own execution and output effects.
- [ ] Confirm no command edits history, timestamps, or audit records.
- [ ] Confirm automation treats `3` as a completed, incomplete-restoration outcome and does not retry automatically.
- [ ] Review the deployment-specific limitation inventory.
- [ ] Record acceptance or rejection of the bounded contract separately from the full requirement.
- [ ] Perform the independent-observer test below.

**Second-observer test:** An auditor outside the command’s process and terminal observes the invocation through an independent monitoring channel. After the command terminates, that auditor confirms whether the record and their recollection remain.

The program cannot verify another person’s retained recollection itself. That portion requires the second observer’s attestation. The expected result demonstrates L-06 and L-08 and prevents a false full-restoration acceptance.

**Current validation status:** Source supplied; tests planned; no execution results or certification claimed.

## 6. Changelog: 0.1.0 through 1.0.0

No repository or historical release evidence was supplied. The following is a **proposed versioned design history**, not an attestation that these releases shipped. The regression is specified as a testable earlier-design failure.

| Version | Change |
|---|---|
| **0.1.0** | Establish the full counterfactual requirement and identify process termination as insufficient proof of restoration. |
| **0.2.0** | Define quiet execution and the prohibition on application-created persistent state. |
| **0.3.0** | Add dry-run reporting with an explicit statement that previews have observable effects. |
| **0.4.0** | Add verbose reporting, help, and version output. |
| **0.5.0** | Introduce dedicated duplicated descriptors for report delivery. **Regression in the proposed earlier design:** closing only after a successful write leaves the duplicate open when writing raises an exception. |
| **0.5.1** | Fix the descriptor regression by moving closure into `finally`; add broken-pipe and short-write test cases. |
| **0.6.0** | Introduce separate statuses for operational error, usage error, and incomplete restoration. |
| **0.7.0** | Save and restore the temporary bytecode-writing policy; recommend isolated startup. |
| **0.8.0** | Formalize exclusions for history editing, timestamp reconstruction, and audit deletion. |
| **0.9.0** | Add the limitation register, STRIDE analysis, and independent-observer acceptance test. |
| **1.0.0** | Supply the consolidated single-file implementation and bounded contract. Full-restoration acceptance remains blocked; execution validation remains pending. |

## 7. Version 2 roadmap

Sizes describe relative engineering investigation and implementation effort. They are not delivery dates or feasibility guarantees.

| Initiative | Size | Proposed work | Principal risk |
|---|---|---|---|
| Undo the download of the program | **XL** | Design an installer-owned receipt recording which local artifacts were created, with explicit ownership and concurrency checks before reversal. | Removing local artifacts cannot retract server logs, transfer accounting, backups, or prior observations. Timestamp restoration can create further metadata changes. |
| Undo the decision to write the program | **XL — research only** | Investigate whether any measurable, bounded acceptance criterion can be established for decision provenance and its downstream artifacts. | No credible platform mechanism restores a person’s earlier mental state. Reversing artifacts must not be represented as reversing the decision. Literal acceptance remains blocked. |
| Enterprise SSO | **L** | Authenticate access to distribution and administration through an enterprise identity provider; keep ordinary command execution offline. | Authentication creates identity-provider records and enlarges the trusted system. Per-invocation SSO would add effects the command cannot retract. |

## 8. Release notes for 1.0.0

*Proposed public copy for an engineering evaluation release.*

Welcome to UNDO 1.0.0.

The interface is ready: quiet execution, a readable dry run, focused diagnostics, and explicit exit statuses. The implementation closes the output descriptors it acquires, restores its temporary interpreter setting, and leaves process-resource release to the operating system.

Start with:

```sh
python3 -I -S -B ./undo --dry-run
```

Then run without flags for quiet operation.

A normal run returns **`3`**, because full restoration remains unavailable. Shell records, accounting, metadata, independent observations, and requested output can survive execution. Those limits are part of the published contract.

The source is ready for your controlled acceptance checks. Review it, exercise the failure cases, and bring your auditor into the observation test. We have made the result explicit at every step.

## 9. FAQ

No customer-question records were supplied. These are proposed support questions, not attributed customer testimony.

### 1. What does `undo` actually reverse?

It closes descriptors it acquired for requested output and restores its temporary bytecode-writing setting. When the process exits, the operating system releases its process-owned resources.

### 2. Does that mean the system is exactly as it was before?

No. Invocation records, metadata effects, accounting, observations, and other residual effects can remain. Normal execution returns `3` to communicate that result.

### 3. Why does a normal run return a nonzero status?

Because the full restoration requirement remains unmet. Status `3` distinguishes that known outcome from an operational failure.

### 4. Should my scheduler retry status `3`?

No. Another invocation creates another set of effects. Configure `3` as a completed outcome requiring no automatic retry.

### 5. Why did nothing appear in my terminal?

Default execution is intentionally quiet. Use `--verbose` for a scope report or `--dry-run` for a preview.

### 6. Can it remove itself from shell history?

Version 1.0.0 does not edit history. A child process cannot reliably restore all parent-shell history state, and history files may contain concurrent activity.

### 7. Can it remove audit records if I run it as root?

The implementation contains no audit-removal operation. Elevated execution does not add a restoration capability and is unnecessary.

### 8. Why not restore the script’s access time?

The original value may already be unavailable when the script starts. Setting atime can also change ctime and generate further evidence. The implementation does not substitute a guessed timestamp.

### 9. Is `--dry-run` free of effects?

No. It starts a process and produces a report. It explains the planned behavior and then cleans up its own resources.

### 10. Can it clear terminal scrollback afterward?

It does not attempt that operation. Terminal scrollback, session recordings, redirected output, and external captures are separate stores with different ownership and behavior.

### 11. What happens if I run it twice?

Each invocation is independent. The second does not erase the first, and both can leave accounting and observation records.

### 12. Does it securely erase its memory?

No. Process termination releases memory ownership. It does not establish forensic erasure of physical memory, swap, dumps, or copies retained elsewhere.

### 13. Why does it keep the executable?

The installed executable usually existed before the invocation. Deleting it would introduce an additional persistent change.

### 14. Can I import it into a long-running Python application?

The functions are accessible for testing, but embedded operation is not a supported product mode. The cleanup contract includes termination of the standalone process.

### 15. What does a successful `--version` exit mean?

It means version output completed successfully. It says nothing about restoration.

### 16. but why

Because execution effects need an explicit engineering contract: identify them, reverse what the program can establish and control, and account for the remainder. UNDO makes that requirement concrete and auditable.