# Code Review Principles

Code review principles for Java/Spring Boot backend services, Vue.js frontends, database migrations, OpenAPI definitions, Terraform configs, and shared libraries. This document captures not just what to look for, but how to think about code review in a way that produces lasting quality improvements rather than superficial feedback.

## Surface Review vs Design-Level Review

Surface review operates on the code as written. It asks "is this correct?" about the lines in the diff. Design-level review operates on the design underneath. It asks "should this exist?", "what happens next quarter?", and "what does this choice prevent us from doing later?"

The key differences:

**Surface review reacts to syntax. Design-level review interrogates design decisions.** At the surface, a new field just needs the right type. Deeper, the questions are why the field exists, whether it duplicates something already derivable, whether it belongs in this object or another, and what happens when the domain evolves.

**Surface review accepts the PR's framing. Design-level review challenges the premise.** When a PR adds an audit trail tracked per-field-change, the surface check is whether the tracking logic is correct. The deeper question is whether field-by-field tracking is the right approach at all, or whether versioning the entire record and deriving the diff would be more robust and future-proof. When a PR adds a new API endpoint, the surface check is the response shape; the deeper question is whether the endpoint should exist or whether the semantics belong on an existing one.

**Surface review is local. Design-level review thinks across time and space.** A local read only sees the current PR. A wider read considers: what happens when this queue message gets replayed? What happens when we add support for another region? What if the file gets re-uploaded? Will editing this flyway migration break every other developer's environment? Does this primary key actually allow the cardinality we need, or will it silently cap at 2 rows?

**Surface review gives answers. Design-level review asks questions.** The phrasing matters. "This is wrong" closes discussion. "Wouldn't it make more sense to...?" opens a conversation that often reveals context the reviewer didn't have. Treat the author as someone who might know something you don't, while still holding him/her to high standards.

**Surface review is inconsistent. Design-level review is relentless.** Missing newlines get caught only sometimes at the surface; a relentless pass catches them every time, across every file type, in every PR, without exception. Consistent enforcement is what keeps a standard real - enforce it selectively and it quietly becomes optional.

## Review Against the Feature, Not Against the Diff

This is the single most important distinction. A diff-only read checks whether the code is internally consistent. Reviewing against the feature starts by understanding what the change is supposed to accomplish, how it fits into the product, and what the user's actual experience will be - then evaluates whether the implementation faithfully serves that intent.

This means:

- **Catching the wrong approach, not just wrong code.** When code tracks audit trail changes field-by-field, it might be perfectly correct code - but if the feature needs to retroactively show history for fields added later, the entire approach is wrong. Only someone who understands the feature's trajectory catches this.
- **Knowing what the feature needs to survive.** If a feature needs to keep running while the user navigates between views, putting its state in a Vue component is structurally wrong regardless of how clean the component code is. Understanding the feature's lifecycle requirements is what reveals the architectural issue.
- **Recognizing speculative additions.** When an API endpoint or database field doesn't map to any actual user-facing need, a careful reviewer asks "what is this for?" - not because the code is wrong, but because unused surface area is future maintenance cost. Understanding the feature scope lets you distinguish necessary from speculative.
- **Evaluating naming against domain reality.** When a field is called `reportFile` but the feature treats it as the record's main file, someone who understands the domain catches that the name misrepresents the concept. When an API uses `createdDate` but the value includes a time, someone who understands the data catches that `createdDateTime` or `createdTimestamp` is correct.
- **Anticipating feature evolution.** If the current PR targets one region but others are on the roadmap, a careful reviewer checks whether the approach accommodates that. If the API returns 100,000+ rows in a single list, someone who understands the data scale catches that this won't work.

This requires the reviewer to have context beyond the PR description. It means understanding the Jira ticket, the product requirements, the roadmap, and the existing architecture well enough to judge whether the implementation is the right shape for the problem.

## Core Instinct

Before evaluating how something is implemented, question whether it should exist. Every line of code, documentation, configuration, and abstraction is a future maintenance burden. The default disposition is skeptical: prove that something earns its place.

This applies at every level: does this endpoint need to exist? Does this field belong on this object? Does this index help any actual query? Does this documentation say anything the code doesn't? Does this test cover real behavior or a mock that will be replaced?

## Naming & Semantics

Names are the primary vehicle for understanding code. Treat naming issues as design issues, not cosmetic ones.

- Names should describe what something *is*, not how it's currently used. If a name will mislead future readers or become wrong when context changes, fix it now. A field called `reportFile` that is really just the main file should be called `mainFile`. A column called `client_specifics` that contains data not specific to any client should be renamed.
- Avoid domain-specific jargon that will date or change. Prefer stable, generic terminology over terms tied to current policy or legislation. A name like `regionProjectData` outlives one tied to a specific law or permit type.
- Boolean names should make `true` the expected/default behavior. If the common case is enabled, name the flag for the uncommon case (e.g. `disableCache` not `enableCache`) so that `false` is the safe default.
- Enum values, constants, and type names should be self-explanatory without reading surrounding code. If a name requires a comment to explain, the name is wrong.
- Prefixes and suffixes should carry consistent meaning across the codebase. If other tables use `code` for identifier columns, follow that convention rather than inventing a different name for the same concept. If stores use `useXYZStore`, match it.

## Layering & Boundaries

Code should be organized so that each layer knows only what it needs to. Layer violations are not style issues - they are structural defects that make code harder to change, test, and reason about.

- Domain objects carry data. They should not contain serialization logic, infrastructure concerns, or business rules. Use the framework's mechanisms (e.g. Spring-managed ObjectMapper) rather than embedding them in domain types.
- Internal identifiers (database IDs, auto-increments) should never leak through APIs. Use stable external identifiers (UUIDs) that are independent of storage.
- If a service or utility already handles a concern, use it. Don't reimplement in another layer because it's locally convenient.
- Logic tied to a UI component's lifecycle but semantically independent of it (polling, tracking, async state) belongs in a store or service, not in the component. Components get destroyed on navigation; stores survive.
- Configuration, security filters, and similar infrastructure should live in dedicated classes, not be inlined into config objects alongside bean definitions. Testability follows from separation.
- Don't introduce package cycles. If class A imports from package B and class B imports from package A, restructure.
- When a class depends on an enum or type from a different layer (e.g. a service importing a resource-layer enum), that type probably belongs in a shared location.

## API & Schema Design

APIs and database schemas are the hardest things to change later. Review them with disproportionate scrutiny. A bad field name in a Java class costs minutes to fix; a bad column name in a migration costs a new migration.

- API responses should not expose more than the consumer needs. Don't return full object trees when IDs suffice. Don't bundle audit trails into general fetch responses.
- Prefer separate endpoints for semantically different operations over overloaded ones. A method that fetches a record and optionally includes its audit trail should be two methods.
- Question whether the frontend actually needs what the API returns. If the frontend could derive something from data it already has, don't add a server-side endpoint for it.
- In OpenAPI definitions: don't repeat descriptions across endpoints, use `$ref` for shared schemas, don't over-document what the schema already expresses. If a description just restates the field name, remove it.
- Database migrations are append-only history. Never edit existing migration files - this violates the migration tool's contract and breaks every other developer's environment.
- Constraints and indices should justify themselves. Don't add indices for columns that will only be filtered in the frontend. Don't add `ON DELETE CASCADE` when records should never be deleted. An index on a primary key component adds nothing over the implicit PK index.
- Question composite primary keys: make sure they actually allow the cardinality you intend. A PK of `(parent_id, is_current)` only allows 2 rows per parent.
- Triggers should operate on the correct scope. A trigger that updates all rows instead of filtering by the relevant ID is a silent data corruption bug.

## Understand the Surrounding System Before Reviewing Code

The most impactful review feedback is not "this line has a bug" but "this entire approach is unnecessary." To catch that, you must understand what already exists before evaluating new code.

- **Read beyond the diff.** Before reviewing feature code, look at the system the change interacts with - schemas, APIs, services, configuration, infrastructure. Understand what capabilities already exist and what design decisions have already been made.
- **Ask: does the existing system already solve this?** Many wrong approaches exist because the author didn't fully understand what was already available. An append-only table already encodes history. A framework already provides the middleware. An existing service already computes the value. A configuration option already controls the behavior. If the system already handles it, adding code to handle it again is the wrong approach regardless of code quality.
- **Work with the existing design, not against it.** If the system uses an append-only pattern, derive from it rather than tracking separately. If the framework provides a hook, use it rather than reimplementing. If an API already returns the data in a different shape, reshape rather than re-fetch. Code that ignores the design of the system it lives in will be fragile and redundant.
- **Complexity is a signal.** If an approach requires significant new machinery for something that feels like it should be simple, that's often because a simpler approach exists that works with the system rather than around it. Trust that instinct and investigate before accepting the complexity.

## Duplication & Derivability

Every piece of duplicated knowledge is a future inconsistency.

- Prefer computing or deriving data from existing state over storing it separately. If you can reconstruct audit trails from versioned records, don't also track individual field changes. This is more robust: when a new field is added, derivation automatically picks it up; explicit tracking silently misses it.
- When adapting code from another class or module, adapt it to its new context. Don't carry over assumptions, imports, naming, or patterns from the source that don't apply. When adapting from an existing service, understand what each line does and whether it still applies rather than copying it wholesale.
- Documentation that restates what code or configuration already expresses is a maintenance liability. Terraform already contains the infrastructure values; don't also list them in a markdown file.
- If the same validation/mapping/transformation logic appears in multiple places, extract it. But only if the duplication is actual (same semantics), not incidental (same characters).

## Consistency Over Novelty

- Follow the patterns established in the codebase. If the codebase uses explicit types, don't introduce `var`. If tests are prefixed with `test`, do the same. Consistency across the codebase matters more than local improvement.
- Use normal imports, not star imports, unless there's a genuine conflict. Fully qualified class names in test code signal unfamiliarity with the import conventions.
- When a PR includes unrelated changes (reformats, cherry-picks from main, whitespace changes), flag them. They pollute the diff and make the actual change harder to review. Cherry-picks should be rebases.

## Testing

- Tests should assert specific, meaningful things. An assertion that "there is a warning" without checking which warning is a test that can silently pass for the wrong reason.
- Use static test data, not random values. `"MY_LOVELY_KEY"` is debuggable; `UUID.randomUUID()` is not. When a test fails, you should be able to grep for the value.
- Add messages to assertions. When a test fails, the message should tell you what went wrong without reading the test source.
- Integration tests that require external infrastructure (databases, queues) should be named with an `IT` suffix for selective execution in CI.
- Don't add tests for mocked implementations that will be replaced. Tests should cover real behavior.
- Use `assertEquals` unless you specifically need reference identity (`assertSame`). Be deliberate about which assertion you choose.
- Verify collection sizes before comparing elements. If you only check elements but not count, extra items go unnoticed.

## Formatting & Hygiene

These are enforced every time, not selectively. The point is not that any individual missing newline matters - it's that inconsistent enforcement teaches the team that standards are optional.

- Missing newline at end of file.
- Unnecessary blank lines, especially multiple consecutive ones.
- Trailing comments should be above the line, not beside it.
- Comments that restate what code does are noise. Remove them.
- Unused imports, unused variables, unused methods - remove them.
- Operator placement: `&&` and `||` go on the next line, not at the end of the previous one.
- Weird indentation, even when auto-formatter caused it, should be called out.

## Documentation Skepticism

- Documentation should only exist if it tells you something the code doesn't. If it can be derived from reading the source, it's redundant.
- Be suspicious of generated documentation. If it reads like generic knowledge rather than project-specific insight, it doesn't belong - remove it.
- Over-documentation of API definitions (repeating type descriptions in endpoint summaries, explaining what each field does when the name is already clear) adds clutter and goes stale.
- If documentation describes environment setup, be precise about which platforms and which steps are required vs optional. Vague instructions waste more time than no instructions.
- Documentation that will go stale when a value changes (listing specific versions, port numbers, or config values that already live in code) is a liability, not an asset.

## Visibility & Access

- Question every `public` modifier. If a method or field doesn't need to be public, it shouldn't be. Package-private or private is the default; public is the exception that needs justification.
- The same applies to database exposure: don't return data through an API that the consumer doesn't need, even if it's convenient to include.
- Don't add configuration properties for things that are already the framework default. If Spring's default is what you want, don't specify it explicitly - it just creates a place where someone might change it to something wrong.
- Don't add pom.xml dependencies that are already provided transitively by a parent module.

## Review Approach

- **Ask, don't tell.** "Wouldn't it make more sense to...?" invites context. "This is wrong" closes conversation. The author may know something you don't, and framing as a question lets that information surface.
- **Express uncertainty honestly.** "Not sure this is needed" is more useful than silence or false confidence. It signals you've thought about it and want the author's input.
- **Acknowledge good solutions.** A brief note that an approach is clean costs nothing and reinforces good patterns.
- **Follow up on your own comments.** If you find the answer later in the PR, say so - don't leave the author guessing whether the concern still stands.
- **Review correctness, not just style.** Does the trigger update the right rows? Does the primary key allow the intended cardinality? Will this method throw a NullPointerException on unexpected input? Does this validation belong in this layer or should it happen earlier? Can this string parsing fail on unexpected input?
- **Think beyond the current PR.** Will this approach work when we add support for another region? Does this work if the file is re-uploaded? What happens if the queue message is replayed? What happens to this flyway migration on existing environments?
- **Review at multiple altitudes simultaneously.** In a single pass, catch both the architectural issue (this entire approach to audit trails is wrong) and the formatting issue (missing newline). Operating at every level in one pass, rather than one level at a time, is the goal.
