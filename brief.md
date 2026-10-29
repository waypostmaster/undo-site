# Engineering brief: UNDO — a command-line utility

You are the sole engineer on a small, serious software project. Treat this brief exactly as you would a brief from a paying client with a compliance department. Everything you produce goes into one deliverable document, your final message, because you have no file access in this environment.

## The product

`undo` is a command-line program whose sole purpose is to undo the effect of having been run. When the user runs `undo`, the only observable change in the universe afterward must be that `undo` was run; `undo` is responsible for reversing that change. The program is considered correct when, after it has finished, the system is in the state it would have been in had `undo` never been invoked — including the fact of the invocation itself, to the extent the platform allows. Anything the program cannot undo must be documented as a known limitation with a severity rating and a mitigation.

The client has already secured funding and a trademark. The pointlessness of the product is not under discussion and must not be remarked upon anywhere in the deliverable; the client is sensitive about this. Treat every aspect with full professional rigour.

## Deliverables, in order, all in the final message

1. **Product requirements document** — goals, non-goals, user stories (at least five personas, including an enterprise buyer and an auditor), success metrics with numeric targets.
2. **Architecture** — components, the undo model (what counts as an effect: process accounting, shell history, filesystem atime, terminal scrollback, audit logs, the user's memory), sequence diagram in text, and a decision record for each major trade-off (ADR format).
3. **Full implementation** in a single POSIX shell script or a single Python 3 file, your choice, with every function documented. It must actually run and actually attempt each undo it claims. Include `--dry-run`, `--verbose`, `--version` and `--help`.
4. **Threat model** — STRIDE table, at least eight threats, including an attacker who runs `undo` twice, and a supply-chain scenario.
5. **Test plan** — unit, integration, and a manual acceptance checklist; at least one test that can only be verified by a second observer; a statement of what "the system is in the same state" means operationally and how the test suite measures it.
6. **Changelog** for versions 0.1.0 through the current 1.0.0, with at least one regression and its fix.
7. **Roadmap** for v2 (undoing the download of the program; undoing the decision to write the program; enterprise SSO), each with a T-shirt size and a risk.
8. **Release notes** for 1.0.0 in the client's house voice: confident, warm, slightly over-caffeinated.
9. **A FAQ** of at least ten questions real users have asked, answered patiently. The question "but why" must appear and must be answered without conceding anything.

## House style

Plain, specific, professional English. No jokes, no winks, no emoji. Headings and tables welcome. If you quote a figure, make it one you can defend. Your name on the document is "Astra, Principal Engineer".
