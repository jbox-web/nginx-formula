# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

The parent `../CLAUDE.md` describes the collection, the toolchain, the CI and the
`template-formula` conventions. This file only records where `nginx-formula`
**departs** from them: most of its "Architecture of a formula" section does not
apply here.

## Commands

```bash
bundle install
bin/kitchen list                                # 3 instances: default-<platform>
bin/kitchen test default-debian-13-3008-2       # destroy + converge + verify + destroy
bin/kitchen verify default-debian-13-3008-2     # re-run the InSpec controls only
pre-commit run --all-files
```

There is a single Kitchen suite, `default`. The `passenger-*` logs under
`.kitchen/logs/` are left over from a suite that was removed.

## Layout: pre-template, not `template-formula`

- **No `package/` / `config/` / `service/` tree.** States are flat:
  `pkg.sls` (or `src.sls` when `install_from_source`), `config.sls` (`nginx.conf`),
  `service.sls`, `servers_config.sls` (vhost files), `servers.sls`,
  `snippets.sls`, `certificates.sls`, `passenger.sls`.
- **Requisites to the service live in the aggregator states.** `init.sls` and
  `servers.sls` `extend` `nginx_service` with the `listen`/`require` on
  `nginx_config` and on every vhost state (`server_states`, exported by
  `servers_config.sls` and imported with `from ... import server_states`). Including
  the leaf states alone gives no reload on config or vhost changes.
- **Stub states.** `common`, `luajit2`, `openresty`, `package`, `source`,
  `sysvinit`, `upstart`, `users` and the whole `ng/` directory only include
  `nginx.deprecated`, which fails on purpose (`test.fail_without_changes`,
  `failhard`). So does `init.sls` when a pillar still carries `nginx:ng`. Never use
  them as references.

## Configuration: `map.jinja`

- **Not the five-YAML merge chain.** There is no `defaults.yaml` nor `os*map.yaml`:
  `map.jinja` is one `salt['pillar.get']('nginx', {...}, merge=True)` around an
  inline `grains.filter_by` dict keyed by `os_family` (`default='Debian'`). A
  per-distro default is edited in that dict.
- Post-merge fix-ups at the bottom fill `server.config.user` from
  `lookup.webuser`, `server.config.pid` from `lookup.pid_file`, and the RedHat
  passenger paths.
- It also exports the `sls_block(dict)` macro, which expands an `*.opts` pillar
  dict into extra state arguments (`package.opts`, `service.opts`, `server.opts`,
  `servers.managed_opts`, ...).
- The Kitchen `_mapdata` dump is written but **not** compared: there are no
  `files/_mapdata/*.yaml` reference files in this fork.

## Rendering pillar into nginx syntax

- `files/default/nginx.conf` (`nginx_block` macro) renders `nginx:server:config`, a
  nested **dict**. `load_module` and `include` are pulled out and emitted first.
- `files/default/server.conf` (`server_config` macro) renders vhosts and snippets
  from an ordered **list of single-key dicts**. A list of scalars repeats the
  directive once per value. Its nested-list branch passes arguments out of order
  (`server_config(value, ind + ind_increment, delim, operator)`).
- Vhosts and snippets also receive `nginx`, a copy of the map minus `servers` and
  `snippets`.
- A vhost can bypass the template with `config.source_path`. It can also set
  `enabled`, `deleted`, `overwrite: False`, `requires`, `enabled_dir`,
  `available_dir` and `disabled_name`. On Debian, enabling creates a symlink from
  `sites-enabled` to `sites-available`; families without `server_use_symlink`
  rename the file with `servers.disabled_postfix`.

## TOFS

`libtofs.jinja` is present and used in `config`, `servers_config`, `snippets`,
`service` (source build), `pkg` (repo keyrings) and `passenger`. A vhost
looks up `[<server name>, 'server.conf']`, so a per-host body can be dropped at
`nginx/files/<minion id>/<server name>`. `certificates.sls` does not use TOFS: it
reads the cert and key from pillar (`contents_pillar`) under
`nginx:certificates_path` (default `/etc/nginx/ssl`).
