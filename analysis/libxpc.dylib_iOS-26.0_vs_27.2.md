# libxpc.dylib: iOS 26.0 (23A340) → iOS 27.2 (24B115)

A direct, end-to-end disassembly diff of `/usr/lib/system/libxpc.dylib` between the first
public iOS 26 release and the iOS 27.2 RC.

Unlike the per-build summaries under [`iOS/`](../iOS) (which chain ~17 consecutive deltas and
have gaps where a build shipped no libxpc diff), this compares the two end binaries themselves.
Both were pulled from the **iPhone17,1** restore images with [blacktop's ipsw](https://github.com/blacktop/ipsw),
so it is a like-for-like, device-matched comparison. Method is in [How this was produced](#how-this-was-produced);
raw diff lists are in [`data/`](data).

## Identity

| | iOS 26.0 (23A340) | iOS 27.2 (24B115) |
| :-- | :-- | :-- |
| `LC_ID_DYLIB` current version | **3089** | **3298.42.1** |
| `LC_SOURCE_VERSION` | 3089.0.11.0.0 | 3298.42.1.0.0 |
| `LC_BUILD_VERSION` | iOS, MinOS 26, SDK 26, ld 1220.3 | iOS, MinOS 27.2, SDK 27.2, ld 27106 |
| Cache UUID (arm64e DSC) | C6340F50-… | (27.2 cache) |
| `__TEXT.__text` | 0x3c034 (245,812 B) | **0x53074 (340,084 B, +38%)** |
| `__TEXT.__cstring` | 0x7696 | 0x7d30 |
| `__TEXT.__oslogstring` | 0x1668 | 0x3109 (≈ doubled) |
| Function starts (`LC_FUNCTION_STARTS`) | **1791** | **1545** |
| Disassembled instructions | 61,453 | 85,021 (+38%) |
| Exported symbols (export trie) | 803 | 801 |

The headline is counter-intuitive: **fewer functions but 38% more code.** 27.2 has 246 fewer
function-starts yet ~24k more instructions. The cause is visible throughout the diff —
aggressive inlining of cold paths and helper routines into their callers (see
[Inlining](#inlining-fewer-functions-more-code)).

## Reliable signals vs. noise

Two things in a DSC symbol diff are trustworthy, and one is not:

- **Trustworthy — export trie** (803 → 801) and **cstrings/oslog strings**. These live in the
  binary itself.
- **Trustworthy — `LC_FUNCTION_STARTS`** (1791 → 1545): a name-independent count of real functions.
- **Not trustworthy — the private/local symbol *names*.** iOS 27's local symbol table (the DSC
  `.symbols` file) carries far fewer private names than iOS 26's. A raw name diff reports ~197
  internal symbols "removed," but most still exist as code — e.g. `__xpc_retain`, `__xpc_release`,
  `_do_mach_notify_send_once`, `__xpc_dictionary_set_value` all show as "removed" yet are plainly
  still present. **The lists in [`data/symbols_removed_or_stripped.txt`](data/symbols_removed_or_stripped.txt)
  mix genuine removals with mere name-stripping and must not be read as "deleted."** Everything
  below is drawn from the export trie, the strings, and the function-body diff, which are not
  affected by stripping.

## Public API changes (export trie — authoritative)

**Added (7):**

| Symbol | What it is |
| :-- | :-- |
| `_xpc_connection_set_bs_checkin_info` | Set BoardServices check-in info on a connection |
| `_xpc_connection_consume_bs_checkin_info` | Consume that check-in info |
| `_xpc_service_instance_set_sanitizer_flags` | Run an XPC service instance under sanitizer flags |
| `_launch_get_user_context` | launchd user-context lookup |
| `_launch_load_jetsam_properties_path` | Load jetsam properties from a path |
| `_amfi_interface_authorize_local_signing_with_length` | AMFI local-signing authorization (re-export) |
| `__xpc_spawnattr_pack_xpc_data` | Pack spawn attributes as xpc_data |

**Removed (9):**

| Symbol | Note |
| :-- | :-- |
| `_xpc_event_stream_check_in`, `_xpc_event_stream_check_in2` | Legacy event-stream check-in, retired |
| `_xpc_bundle_copy_resource_path` | Bundle resource-path helper, gone |
| `_xpc_dictionary_get_connection` | Removed from the export trie (the `.symbols` file no longer lists it either) |
| `__system_version_copy_string_plist` / `_sysctl` / `_fallback` / `_parse_string` / `_plist_path` | The whole system-version-string family moved out of libxpc |

## Strings added (verified in `__cstring`/`__oslogstring`)

169 new strings, 112 removed. Full lists: [`data/strings_added.txt`](data/strings_added.txt),
[`data/strings_removed.txt`](data/strings_removed.txt). The substantive ones:

**BoardServices check-in info (new SPI) — strict state machine:**
```
"Attempt to set the bs_checkin_info on a {listener|live|peer|non-bs_type} connection."
"Attempt to fetch the bs_checkin_info on a {listener|live|non-peer} connection."
"Cannot set NONE when specifying checkin info."
"Check-in is too big : %zu"              "[%p] invalid checkin_info length : %zu"
"[%p] bs_checkin_info failed to encode : %s"   "[%p] bs_checkin_info failed to decode"
"[%p] bs_checkin_info is too big : %s"
```
The old property was also renamed: `"Attempt to change the bs_managed propertyon a live
connection."` (26.0, note the typo) → `"Attempt to change the bs_type property on a live
connection."` (27.2).

**Connection setup/teardown — per-site diagnostics.** 26.0 assembled one message from fragments
(`"failed at %s%s with error %d - %s"` + `"lookup"`, `"send right creation"`, `" (reconnect)"`, …).
27.2 has a distinct string per failure site:
```
"Connection init failed at lookup with error %d - %s"
"Connection init failed at {listener activation|no-senders registration|receive right creation|send right creation} with error %d - %s"
"Connection init failed with readying error with error %d - %s"
"Reconnect failed at …"   "Finalizing BS connection"   "Peer image changed"   "Client is gone"
```

**Sanitizers / hardening:**
```
"sanitizers"
"Reply messages must not be sent as notifications; use xpc_dictionary_send_reply()"
"Given object not of required type. Given: %s, required: %s"
"Attempted to serialize xpc_data that is too large for what the wire protocol currently supports. Length was %zu and max supported is %u"
"Tried to create an XPC data with NULL bytes and %zu length"
"Kernel bug: Unexpected error from pipe mach_port_construct()"
"Kernel bug: Unexpected error from service attach port mach_port_construct()"
```

**Assertion messages (+112).** iOS 26.0 already had expression-carrying assertions; 27.2 adds 112
more `assertion failure: "<expr>" -> %llu` strings, matching the doubled `__oslogstring`.

**Other:** `"The executable did not ship in the bundle"`, `"An extension with the same bundle ID
already exists with a different path"`, `"bundle %s missing from disk and dyld shared cache"`,
transaction logs `"[%p] '%s': Transaction created/released"`, and keys `"pid domain"`,
`"loginwindow"`, `"token.pid"`, `"bssendp"`.

**Removed:** the fragment-assembly strings above, `"Peer rejected"`, `"Underlying connection
invalidated"`, `"Attempt to send non-reply message as reply"`, `"Bug in libxpc: Unexpected
disposition"`, and a batch of ObjC runtime selector/type strings (`"respondsToSelector:"`,
`"v16@0:8"`, `"OS_os_transaction"`, …) that are no longer emitted as plain cstrings.

## New functions (body diff — 61 real, verified present in 27.2)

Full list (incl. compiler helpers): [`data/symbols_added.txt`](data/symbols_added.txt). Grouped:

**Connection init split into single-purpose functions.** This is the central refactor and it is
provable from the bodies: `__xpc_connection_init` collapsed from **486 → 88 instructions**, with
the work moved into new functions:
```
__xpc_connection_init_send      __xpc_connection_init_recv       __xpc_connection_init_term
__xpc_connection_init_send_named  __xpc_connection_init_recv_named
__xpc_connection_init_send_anon   __xpc_connection_init_recv_anon
__xpc_connection_init_recv_port   __xpc_connection_init_no_more_senders
__xpc_connection_init_cleanup_from_failed_readying
__xpc_connection_derive_connection_port   __xpc_connection_pipe_check_in
__xpc_connection_handle_cancel_event      __xpc_connection_handle_term_event
__xpc_connection_remove_peer_async        __xpc_connection_unwind_peers
__xpc_connection_cancelation_get_description
```

**Pipes:** `__xpc_pipe_derive_port`, `__xpc_pipe_init_send_port`, `__xpc_pipe_init_recv_port`,
`__xpc_pipe_message_deserialize`, `__xpc_pipe_simpleroutine_impl`.

**Sessions:** `__xpc_session_activate_new`, `__xpc_session_cancel_with_code`,
`__xpc_session_handle_incoming_error_event`, `__xpc_session_fault`.

**Services & bundles:** `__xpc_init_service`, `__xpc_init_proxy`, `__add_dlopened_services`
(discover services in dlopen'd images), `__xpc_should_add_image`, `__xpc_service_check_subname`,
`__xpc_bundle_resolve_bundle_variant`, `__xpc_bundle_taint_with_reason`, `__xpc_realpath_cryptex`,
`__xpc_resolve_real_path`, `__trim_trailing_slashes`, `__copy_path_in_shared_cache`,
`__copy_key_from_plist`.

**Serialization / misc:** `__xpc_data_serialize_ool`, `__xpc_string_cache_insert`,
`__xpc_plist_parse_data`, `__xpc_plist_parse_real`, `__xpc_log_cycle_path`,
`__xpc_event_publisher_check_and_update_inflight_count`, `__os_transaction_log_snapshot`,
`__xpc_activity_is_unmanaged`, `__objc_getTaggedPointerTag`.

## Inlining: fewer functions, more code

Of 1,478 functions present under the same name in both: **425 are byte-identical, 1,053 changed.**
The changed ones overwhelmingly grew. Most-changed (full table:
[`data/changed_functions.tsv`](data/changed_functions.tsv)):

| Δ insns | function | 26.0 → 27.2 |
| --: | :-- | :-- |
| **+1274** | `_xpc_create_from_plist_with_string_cache` | 709 → 1983 |
| **−398** | `__xpc_connection_init` | 486 → 88 (split out, above) |
| +346 | `__xpc_init_pid_domain` | 91 → 437 |
| +339 | `__xpc_dictionary_insert` | 164 → 503 |
| +263 | `__xpc_interface_routine` | 133 → 396 |
| +262 | `__translate_attrs` | 84 → 346 |
| +249 | `__xpc_bundle_resolve_executable` | 176 → 425 |
| +226 | `__xpc_user_sessions_info_routine` | 84 → 310 |
| +215 | `__xpc_event_publisher_set_subscriptions` | 296 → 511 |
| +162 | `-[OS_xpc_session dealloc]` | 18 → 180 |

The pattern — callers ballooning while helper/`.cold` routines vanish from the function table —
is inlining, and it accounts for the "1545 functions but +38% text" result.

## Themes (what actually changed, 26.0 → 27.2)

1. **Connection lifecycle rewrite.** `_xpc_connection_init` was decomposed into explicit
   send/recv × named/anon paths plus dedicated no-more-senders / term / cancel handlers and peer
   teardown. Provable from both the new symbols and the 486→88 shrink.
2. **BoardServices check-in SPI.** New public `xpc_connection_{set,consume}_bs_checkin_info`,
   `bs_managed` renamed to `bs_type`, guarded by a strict connection-state machine.
3. **Sanitizer / correctness hardening.** New `xpc_service_instance_set_sanitizer_flags` + a
   `"sanitizers"` key; new misuse faults (reply-as-notification, type mismatch, oversized/NULL
   xpc_data); +112 expression-carrying assertions.
4. **Per-site diagnostics.** Connection failures went from fragment assembly to one explicit
   string per failure site; transactions now log create/release.
5. **Service discovery & cryptex-aware bundles.** Services can be registered from dlopen'd images;
   bundle resolution gained cryptex real-path normalization and variant/taint handling.
6. **Legacy removal.** `xpc_event_stream_check_in{,2}`, `xpc_bundle_copy_resource_path`, and the
   in-libxpc system-version-string family were dropped from the exports.

## How this was produced

```
# per build (iPhone17,1 restore image):
ipsw fw aea <SystemOS .dmg.aea>            # decrypt (keys bundled in ipsw)
7z x <dmg> 'System/Library/Caches/com.apple.dyld/*'   # extract dyld_shared_cache (no mount)
#   + LZBITMAP resource-fork files decompressed with a small lzfse helper
ipsw dyld extract <dsc> libxpc.dylib       # slide+stubs+objc
ipsw dyld macho   <dsc> libxpc.dylib --loads --symbols --strings --starts
ipsw dyld disass  <dsc> --image libxpc.dylib
```
Disassembly was normalized (addresses, `loc_`/`sub_`, absolute immediates masked) before diffing
so that only real instruction changes register. Scripts and intermediates live under the session
scratchpad; the reduced diff lists are in [`data/`](data).

### Caveats

- **Private symbol names are stripped in iOS 27**, so a raw local-symbol name diff overstates
  removals (see [Reliable signals](#reliable-signals-vs-noise)). Export trie, strings, and
  function bodies are used instead and are not affected.
- The 27.2 binary is libxpc **3298.42.1**, marginally newer than the last build that carried a
  libxpc diff in the repo's chain (3298.40.20, 27.2 b1).
- Device is iPhone17,1 for both ends (the repo's per-build chain switches devices mid-stream);
  this avoids the spurious size deltas that device switches introduce.
