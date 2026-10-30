# libxpc.dylib: iOS 26.0 (23A340) → iOS 27.2 RC (24B115)

`/usr/lib/system/libxpc.dylib`, traced across every consecutive diff in [`iOS/`](../iOS)
from the first public iOS 26 release to the latest iOS 27 build in this repo.

| | iOS 26.0 (23A340) | iOS 27.2 RC (24B115) |
| :-- | :-- | :-- |
| Project version | `3089.0.11.0.0` | `3298.40.20.0.0` (last recorded, 24B5084k; no later libxpc diffs) |
| UUID | `FD0E3621-E113-3F18-9F29-529D3B568825` | not recorded after 27.0 b1 |
| `__TEXT.__text` | `0x3c034` (245,812 B) | `0x51b60` (334,688 B) on iPhone19,2 (+36%) |
| `__TEXT.__cstring` | `0x7696` | `0x7ce6` |
| `__TEXT.__oslogstring` | `0x1668` | `0x3109` (more than doubled) |
| Functions / Symbols / CStrings | 1791 / 4488 / 1242 | 1545 / 2427 / 1296 (see caveats) |

## Build chain and versions

Path followed: 26.0 → 26.1 → 26.2 → 26.3 → 26.4 → 26.5 → 27.0 → 27.2. The 26.6/26.7 branch and
27.0.1 split off this path and are not on the line to 27.2.

| Step | libxpc version | Change |
| :-- | :-- | :-- |
| 26.0 23A340 / 23A341 / 23A345 | 3089.0.11 | none recorded |
| 26.1 b1 23B5044l | **3089.40.13.0.1** | big: assertion messages, os_log, reply-as-notification check |
| 26.1 b2 23B5059e | 3089.40.15 | connection-failure diagnostics rework |
| 26.1 b3 … 26.2 b1 | → 3089.60.3 | *version bumped, no libxpc diff in repo* |
| 26.2 b2 23C5033h | 3089.60.4 | `-[OS_xpc_object debugDescription]` tweak |
| 26.2 … 26.2 23C55 | → 3089.60.7 | *no diff in repo* |
| 26.3 b1 23D5089e | 3089.80.8 | `launch_load_jetsam_properties_path` |
| 26.3 b2 23D5103d | → 3089.80.9 | *no diff in repo* |
| 26.3 b3 23D5114d | 3089.80.10.502.3 | bundle executable-missing error |
| 26.3 23D125 | 3089.82.3 | rebuild only |
| 26.4 b1 23E5207q | **3102.100.97.502.1** | big: port derivation, system-version code moved out, transaction logging |
| 26.4 b2 23E5218e | 3102.100.101.502.1 | cryptex path resolution |
| 26.4 … 26.4 23E246 | → 3102.102.1 | *no diff in repo* |
| 26.5 b1 23F5043g | 3102.120.11 | attrs/entitlement lookup by pid |
| 26.5 … 26.5 23F77 | → 3102.120.13 | *no diff in repo* |
| 27.0 b1 24A5355q | **3295.0.0.502.1** | major: connection init refactor, BS check-in info, sanitizer flags, … |
| 27.0 b2 24A5370h | 3298.0.4.502.1 | recompile: 140 functions change size by ±4–48 B, the rest hash-only |
| 27.0 b3 24A5380h | 3298.0.10 | hash functions shrink by 8 B |
| 27.0 b4 24A5390f | 3298.0.21 | content-only change |
| 27.0 b5 24A5408d | 3298.0.26.502.1 | port init and duplicate-extension check |
| 27.0 b6 … RC | → 3298.2.1 | *no diff in repo* |
| 27.0 24A435 | 3298.2.1 | event-publisher subscriptions, bundle variant subpath |
| 27.2 b1 24B5084k | 3298.40.20 | `_libxpc_initializer` +8 B |
| 27.2 b2 24B5089g | — | `__DATA_DIRTY.__bss` 0x138 → 0x140 |
| 27.2 b3 … RC 24B115 | — | no libxpc diff recorded |

## Changes by release

### iOS 26.1: hardening and diagnostics (3089.0.11 → 3089.40.15)

**26.1 b1 (23B5044l)**: `__text` grew 0x3c034 → 0x42244 (+25 KB, +119 functions), `__oslogstring` doubled.

- **Assertions now carry a message.** About 120 new `assertion failure: "<expr>" -> %llu` strings.
  Internal invariants that used to crash without context now report the failing expression. They cover
  connections, listeners, sessions, the serializer/deserializer (`xgd->_depth`, `_ool_callback`, string caches),
  activities (`_xpc_activity_is_unmanaged()`, eligibility handlers), file transfers, event publishers, spawnattr
  binprefs, and mach-port guarding (`change_fdguard_np`, `guarded_close_np`). The new `__os_crash_msg`,
  `__os_log_send_and_compose_impl` and `__os_log_default` symbols match this.
- **New misuse checks**:
  - `"Reply messages must not be sent as notifications; use xpc_dictionary_send_reply()"`
  - `"Given object not of required type. Given: %s, required: %s"`
- Transactions: `__os_transaction_log_snapshot` is new. `"on activation of exit_when_clean"` was added.
- New `__xpc_allocate_buffer` and `_xpc_data_create.cold.1` (an xpc_data allocation path).
- **Removed: `_xpc_bundle_copy_resource_path`, `_xpc_event_stream_check_in`, `_xpc_event_stream_check_in2`**
  (legacy event-stream check-in), and the `__xpc_is_being_debugged` dispatch-once machinery.

**26.1 b2 (23B5059e)**: connection cancellation was reworked.

- New `__xpc_connection_cancelation_description` and `__xpc_connection_cancelation_get_description`.
  `xpc_connection_copy_invalidation_reason` grew 32 → 84 B.
- Error strings used to be built from fragments (`"failed at %s%s with error %d - %s"` + `"lookup"`,
  `" (reconnect)"` …). Each failure site now has its own string, for example
  `"Connection init failed at lookup with error %d - %s"` and `"Reconnect failed at send right creation …"`.
  The phrases were also capitalized: `"Client is gone"`, `"Peer image changed"`.
- Imports `fmtcheck`, which validates format strings.

### iOS 26.2 (→ 3089.60.7)

- `-[OS_xpc_object debugDescription]` 260 → 280 B. Nothing else was recorded.

### iOS 26.3 (→ 3089.82.3)

- New `launch_load_jetsam_properties_path` (b1).
- `__xpc_bundle_resolve_executable` gained `"The executable did not ship in the bundle"` (b3).

### iOS 26.4: new 3102 train (3089.82.3 → 3102.102.1)

**26.4 b1 (23E5207q)**:

- **Port derivation**: new `__xpc_connection_derive_connection_port`, `__xpc_connection_init_recv_port`,
  `__xpc_pipe_derive_port`. Two new crash strings:
  `"Kernel bug: Unexpected error from pipe mach_port_construct()"` and
  `"... service attach port mach_port_construct()"`.
- **System-version code moved out of libxpc.** These were removed: `__system_version_*`,
  `_os_system_version_get_current_version.*`, `_populate_current_*version`, `_parsed_*version`.
  Their replacements are `__os_system_version_initialize`, `__current_version`, `__current_dyld_version.*` and
  `__sim_current_host_version`. `__availability_version_check.cold.1` is gone.
- **Size limits on xpc_data**: `"Attempted to serialize xpc_data that is too large for what the wire protocol
  currently supports…"` and `"Tried to create an XPC data with NULL bytes and %zu length"`.
- Transactions: `"[%p] '%s': Transaction created/released"` debug logs. `_os_transaction_create` and
  `__os_transaction_xref_dispose` gained cold paths.
- New `_launch_get_user_context`, `__xpc_session_fault`, `__copy_key_from_plist`, `__copy_path_in_shared_cache`.
  New strings `"pid domain"`, `"loginwindow"`, `"token.pid"`, `"bssendp"`.
- `"Could not get attrs using %s for connection %s, pid %d…"` (`__xpc_connection_copy_attrs`).
- Removed: `__xpc_assert_dumping_ground`, `__xpc_plist_parse_date`, `_der_vm_CEType_from_ccder_tag`,
  `__os_once`, and the strings `"Peer rejected"` and `"Underlying connection invalidated"`.

**26.4 b2 (23E5218e)**: cryptex-aware path resolution.

- New `__cryptex_prefix_paths`, `__trim_trailing_slashes`, `__xpc_resolve_real_path`, `__normalize_cryptex_path` cold path.
  `__copy_path_in_shared_cache` was dropped.
- The bundle errors were simplified to `"bundle %s missing from disk and dyld shared cache"`, without the errno.

### iOS 26.5 (→ 3102.120.13)

- `__xpc_connection_copy_attrs` grew 472 → 564 B and `xpc_copy_entitlement_for_token` 140 → 212 B. The new
  `"Could not get attrs using %s for pid %d…"` shows that lookups now go by pid without needing a connection.

### iOS 27.0: major release (3102.120.13 → 3298.2.1)

**27.0 b1 (24A5355q)**: version 3295.0.0. `__text` grew 0x41924 → 0x530c8 (+70 KB, about +27%), the largest
jump in this range.

- **Connection setup split into single-purpose functions**: `__xpc_connection_init_send{,_named,_anon}`,
  `__xpc_connection_init_recv{,_named,_anon}`, `__xpc_connection_init_no_more_senders`,
  `__xpc_connection_init_term`, `__xpc_connection_init_cleanup_from_failed_readying`.
  There are also new event handlers `__xpc_connection_handle_cancel_event` and `__xpc_connection_handle_term_event`.
  Peer teardown gained `__xpc_connection_remove_peer_async`, `__xpc_connection_unwind_peers` and
  `__xpc_connection_pipe_check_in`. The `__xpc_connection_init.cold.*` / `__xpc_connection_mach_event.cold.*`
  paths and `__xpc_connection_bs_cancel` are gone.
- **BS check-in info, new SPI**: `xpc_connection_set_bs_checkin_info` and `xpc_connection_consume_bs_checkin_info`,
  checked by these strings:
  - `"Attempt to set the bs_checkin_info on a {listener|live|peer|non-bs_type} connection."`
  - `"Attempt to fetch the bs_checkin_info on a {listener|live|non-peer} connection."`
  - `"Check-in is too big : %zu"`, `"[%p] invalid checkin_info length : %zu"`
  - `"[%p] bs_checkin_info failed to encode/decode"`
  - `"Cannot set NONE when specifying checkin info."`

  The old property was renamed: `"Attempt to change the bs_managed propertyon a live connection."` became
  `"Attempt to change the bs_type property on a live connection."`. (In 27.0 b2 the symbol table shows
  `xpc_connection_{set,get}_bs_type`, `xpc_connection_bs_seal_listener` and `xpc_endpoint_create_bs_{from_port,service}`.)
- **Sanitizer support for services**: `xpc_service_instance_set_sanitizer_flags` and a `"sanitizers"` key. This
  goes with the `"checked-allocations"` key added to `xpc_service_instance_run` in the 26.0 RC.
- **Service discovery from dlopen'd images**: `__add_dlopened_services`, `__xpc_should_add_image`.
  There are also new `__xpc_init_service`, `__xpc_init_proxy` and `__xpc_service_check_subname`.
- **Bundles**: `__xpc_bundle_resolve_bundle_variant`, `__xpc_bundle_taint_with_reason`, `__xpc_realpath_cryptex`.
  `__copy_path_in_shared_cache` returned. `__xpc_bundle_resolve_executable_{name,variant}` and
  `__xpc_bundle_is_valid_variant` were removed.
- **Pipes and sessions**: `__xpc_pipe_simpleroutine_impl`, `__xpc_pipe_message_deserialize`,
  `__xpc_pipe_init_{send,recv}_port`, `__xpc_session_activate_new` (replacing `_activate_if_new`),
  `__xpc_session_cancel_with_code`, `__xpc_session_handle_incoming_error_event`.
- **Serialization and plist**: `__xpc_data_serialize_ool`, `__xpc_spawnattr_pack_xpc_data`,
  `__xpc_string_cache_insert`, `__xpc_plist_parse_data`, `__xpc_plist_parse_real`, and `__xpc_log_cycle_path`
  (looks like a reference-cycle diagnostic).
- **Event publisher**: `__xpc_event_publisher_check_and_update_inflight_count` is now a standalone function.
- **New imports**: `amfi_interface_authorize_local_signing_with_length`, `dispatch_io_set_low_water`.
- **Removed/renamed** (where it can be told apart from tooling noise, see caveats): `CEXPCRuntime`,
  `__xpc_enum2label`, `__copy_secure_plist`/`__xpc_copy_secure_plist`/`__copy_plist_from_fd`,
  `__waiting4memory`, `__transaction_snapshot_destroy`, `__xpc_activate_endpoint`,
  `__xpc_connection_validate_check_in_is_for_service`, `__xpc_connection_set_creds`, `__xpc_bootstrap_routine*`,
  `__xpc_domain_routine*`, `__xpc_event_routine*`, `__xpc_control_routine`, `__xpc_vproc_routine`.
- Strings removed: `"Attempt to send non-reply message as reply"`, `"Bug in libxpc: Unexpected disposition"`,
  `"Invalid requested disposition in deserializer"`.

**27.0 b2 → GM**:

- b2 (3298.0.4): a full recompile. Of 1241 functions with changes, 140 changed size, nearly all by ±4–8 B, and
  the rest differ only in hash. Larger changes: `__xpc_event_publisher_set_subscriptions` +48,
  `xpc_create_from_plist_with_string_cache` +40, `__xpc_xml_replace_entities` +20, `CESerializeWithOptions` +20,
  `xpc_dictionary_create` +16, `xpc_bundle_copy_normalized_cryptex_path` −20.
- b3 (3298.0.10): the `__xpc_{date,double,uuid,pointer}_hash` functions shrank 48 → 40 B and the int64/uint64
  hash functions 248 → 240 B, which suggests the hash routine was simplified.
- b5 (3298.0.26.502.1): `__xpc_connection_init_{recv,send}_{named,anon,port}` and
  `__xpc_connection_derive_connection_port` changed size. New string:
  `"An extension with the same bundle ID already exists with a different path"`.
- 24A435 (3298.2.1): `__xpc_event_publisher_set_subscriptions` 2076 → 2044 B,
  `__xpc_bundle_variant_create_subpath` 428 → 480 B, `CESizeSerialization`/`CESerializeWithOptions` +4.

### iOS 27.2 (→ 3298.40.20)

- b1: only `_libxpc_initializer` 1524 → 1532 B.
- b2: `__DATA_DIRTY.__bss` +8 B (one new global).
- b3 → RC (24B115): no libxpc changes recorded.

## Themes

1. **Better diagnostics** (26.1, 26.4). Assertions report their expression, every connection-failure site has its
   own message, transactions are logged, and `__oslogstring` doubled in size.
2. **Connection lifecycle rewrite** (26.4 → 27.0). Port derivation moved into its own functions, then
   `_xpc_connection_init` was split into send/recv × named/anon parts with explicit teardown and cancel/term handlers.
3. **BoardServices (`bs_*`) integration grows**. `bs_managed` became `bs_type`, and check-in info can now be
   passed through the connection with size and state checks.
4. **Cryptex and shared-cache bundle resolution** (26.4 → 27.0). Real-path normalization covers cryptex prefixes,
   and bundles can be "tainted" with a reason.
5. **Tooling for services**: sanitizer flags, checked allocations, discovery of services in dlopen'd images.
6. **Legacy removal**: `xpc_event_stream_check_in{,2}` and `xpc_bundle_copy_resource_path` (26.1), the system-version
   code (26.4), and several old `*_routine` MIG-era helpers (27.0).

## Caveats

- **Gaps in the data.** The libxpc version changed in several steps that have no libxpc diff in the repo (listed
  above). Any changes in those builds are missing here. There are no libxpc diffs after 27.2 b2.
- **Device changes.** The source IPSWs move from iPhone17,1 (26.0) to iPhone18,1 (26.1–27.0) and then iPhone19,2/3/7
  (27.0 24A437 onward). The jump in `__text` from 0x5306c (iPhone18,1, 24A435) to 0x51b58 (iPhone19,2, 24A437)
  comes from the device switch. It is not a code change.
- **The 27.0 b1 diff format changed.** It adds sha256 per section, reports `__auth_stubs` and the objc name sections
  as 0x0, and drops every `.cold.N` / `OUTLINED_FUNCTION_N` symbol. That explains most of the drop from 1936 to
  1545 functions and from 4633 to 3479 symbols. The objc type/selector strings (`OS_xpc_*`, `v16@0:8`, …) also
  disappear from `__cstring` while `__objc_classlist` keeps the same size. These are almost certainly
  shared-cache/tooling artifacts. Removals from that diff are listed only where they match a real refactor. Public
  API such as `xpc_dictionary_get_connection` or `xpc_rich_error_create`, which appear in its removed list, should
  be checked against a live 27.x cache before concluding they are gone.
- Symbol counts dropped again (3479 → 2427) somewhere in 27.0 b3/b4 while the function count stayed at 1545. This
  is also a symbol-table effect.
