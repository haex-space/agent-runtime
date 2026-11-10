# Harness Package Contract

## V1 package contents

A package is declarative content with:

- Stable package ID and semantic version.
- Runtime compatibility range and declared package dependencies.
- Immutable source revision and content digest.
- Optional priority/order where explicitly relevant.
- Contributions limited to prompt sections, routing hints, model defaults, non-authority task defaults, and references to host-registered tool IDs.

V1 does not load executable code or run install/activation hooks.

## Composition

- The consumer selects packages and pins each version and immutable source identity.
- The resolver validates package identity, digest, compatibility, dependencies, and contributions before activation.
- Packages are composed in explicit order. Contributions with no order-sensitive semantics are stable under reorder.
- A duplicate key with differing values is a conflict unless the consumer names the overriding package/value.
- Task defaults may make a task more restrictive; effective limits are the stricter of the host's ceiling and the package/user value.
- The resolved configuration reports the source of every contribution and the final effective value.
- Unknown contribution kinds are rejected or preserved as unsupported metadata; they are never executed.

## Trust and authority

Package loading and capability authorization are separate. A package can reference a tool only if the host registered that tool and granted it to the task. No package may define executable hooks, credentials, permission escalation, approval bypass, or arbitrary process access.

## Import/export

An exported configuration contains package IDs, versions, immutable source revisions, content digests, ordering, and explicit overrides. Import validates all pins before activation and reports missing or changed sources without silently substituting a newer version.
