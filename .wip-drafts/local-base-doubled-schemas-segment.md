# Draft texts, not filed. Build only, per instruction.

## Draft issue

Title: Local base bundling doubles a path segment after #66, breaking schema local base resolution against a real ucp checkout

Body:

## Observed

`ucp-schema validate <payload> --schema <base>/discovery/profile_schema.json --def signing_key --op read --schema-local-base <base>` where base is a checkout of the ucp v2026-04-08 source tree:

| Commit | Result |
| --- | --- |
| 9b5c3206 (before #66) | exit 0, Valid |
| main b52518f5 (after #66) | exit 2, error below |

```
Error: bundling refs: failed to bundle schema: failed to bundle schema: Resource 'https://ucp.dev/schemas/schemas/shopping/types/reverse_domain_name.json' is not present in a registry and retrieving it failed: Retrieving external resources is not supported once the registry is populated
```

Bisected the range between those two commits by building and running each step. 9f91c3d (#66) is the first commit where this fails. The two docs only commits after it, d165c4a and b52518f, do not change the result.

The doubled schemas segment traces to discovery/profile_schema.json, which declares $id https://ucp.dev/schemas/discovery/profile.json while discovery sits on disk as a sibling of schemas, not nested inside it. The relative ref ../schemas/ucp.json inside it is written to be correct on disk, but resolved by URL join against the declared $id it lands on https://ucp.dev/schemas/schemas/ucp.json instead. The retriever still finds the real file through the learned $id directory anchor, and crawl_external_refs registers it correctly under the declared $id too, so this first hop does not fail on its own. Upstream dereference then resolves the relative refs inside that fetched document against the URI used to retrieve it, not against its declared $id, so the doubled base carries forward to every further hop, and only the correct, undoubled key had been registered for those hops.

A second, related site: with both schema local base and schema remote base set, the same mismatch makes the explicit URL mapping in the retriever join onto a path that does not exist on disk, and that step fails immediately instead of falling through to the same learned anchor recovery that already covers this for the local base only case.

## Consequence

Release PR #67 (2.0.0) carries #66 unreleased. Cutting it before this is fixed ships the regression into the release, breaking schema local base validation against a real ucp source checkout, which is the primary way to validate examples against a pinned or unreleased spec version outside this repo.

On how this relates to #70: the ask there for the 2.0.0 cut stands. #66 closed #43, #45 and #46 and holds up everywhere else we exercised it; this is one layout shape the rework did not cover. The ask here is sequencing only: land the fix below first, so the cut ships #66 without this regression.

## Fix

PR [link once opened] fixes both sites and adds coverage for both bundling entry points, bundle_refs and bundle_refs_with_url_mapping, the second of which had no direct test before this. The command above exits 0 on the fix branch. The full suite, the #45 and #46 regression fixtures, and the four Draft 2020 12 conformance cases #66 itself added all still pass, and the released v2026-08-25 tree behaves identically before and after the fix.

---

## Draft PR

Title: fix: stop crawl external refs and the local base retriever from doubling a path segment

Body:

## Root cause

Introduced by #66, commit 9f91c3d497d8f9d7f4051fbfb49b2262536acfd7. Bisected the range between 9b5c3206 (good) and b52518f5 (bad) commit by commit; 9f91c3d is the first bad one, and the two docs only commits after it do not change the result. #66 is otherwise a sound rework that closed #43, #45 and #46; this is one layout shape it did not cover.

crawl_external_refs in src/loader.rs resolves the relative refs of a fetched document against its declared $id (fetched_base), which is correct on its own, but upstream jsonschema::dereference resolves the relative refs of a fetched resource against the URI it was retrieved under, not its declared $id. When a relative ref is written correctly for the referrer disk location, but the referrer $id nests it one level deeper than a sibling directory (discovery beside schemas, both under https://ucp.dev/schemas/...), the URL join off the retrieval position doubles the shared segment. Only the correct, undoubled key from our own crawl had ever been registered, so upstream dereference fails with not present in a registry the moment it independently rederives the doubled key for a further hop.

A related site: the UcpRetriever explicit schema local base plus schema remote base mapping (step 1) hits the same doubled join and returns a hard file not found error instead of falling through to the learned anchor relocation in step 3, which already recovers from this mismatch for the no remote base case.

## Fix

Both changes are additive.

- crawl_external_refs: when the declared $id of a fetched document differs from the URI it was retrieved under, crawl the refs of that document a second time using the retrieval URI as base, in addition to the existing pass using its declared $id, so resources reachable through either base end up registered. The second pass is guarded on that difference, so it is a no op when the two agree.
- UcpRetriever::retrieve: only return the explicit mapping candidate when the mapped path exists on disk, falling through to the remaining steps otherwise, exactly as an unmapped URI already does. One behavior delta to name: a mapped URI whose local file is missing used to fail hard at the mapping step and now takes the same fallthrough as every unmapped URI, which with the remote feature can end in a network fetch.

## Tests

Added to tests/conformance_test.rs, matching the bundle and oracle_is_valid idiom already used in the file (accept and reject verdicts checked through the jsonschema crate, never our own emitted schema text):

- sibling_directory_nested_under_id_resolves_without_doubling reproduces the shape through bundle_refs, the path exercised by validate --schema --schema-local-base with no remote base.
- sibling_directory_nested_under_id_resolves_without_doubling_with_url_mapping covers the same shape through bundle_refs_with_url_mapping (schema local base plus schema remote base, also used internally by compose), which had no direct test before this change; its only prior coverage came through CLI fixtures whose declared $id and disk layout agree.

Each fails before the fix with the failure shape of its own site: the first with the same not present in a registry error the real repro produces, the second with a doubled path file not found error at the explicit mapping step. Both pass after. Reverting src/loader.rs alone with the tests kept turns both red again with those same shapes, confirming they test the fix rather than passing on their own.

## Verification

- The exact reported command now passes against a real ucp v2026-04-08 source checkout: exit 0, Valid (was exit 2 on main).
- cargo test --all-targets: 355 passed, 0 failed, 0 ignored.
- The #45 and #46 regression fixtures, the four Draft 2020 12 conformance cases #66 itself added, and every other recursion, self ref, and def selection test in the suite still pass, confirming this does not resurrect either stack overflow.
- The released v2026-08-25 tree is unaffected: lint over its source directory passes identically (124 files), and def selection on common/types/payment_instrument.json returns identical verdicts and messages on main and on this branch, in both the accept and the reject direction.
- cargo fmt --check and cargo clippy --all-targets -- -D warnings are both clean.

Fixes #[issue number once opened].
