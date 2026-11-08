rebar_mix_example
=================

Example of using an Elixir dependency from hex ([jason](https://hex.pm/packages/jason))
in an Erlang rebar3 release, via the [rebar_mix](https://hex.pm/packages/rebar_mix) plugin.

On startup, `example_sup` encodes an Erlang map to JSON with `Jason.encode!/1`,
decodes it again with `Jason.decode!/1`, and prints the result.

Requirements
------------

Tested with Erlang/OTP 29.1.2, Elixir 1.20.5 (built for OTP 29), rebar3 3.28.0,
rebar_mix 0.5.1 and jason 1.4.5.

- Erlang/OTP and rebar3
- Elixir, with `elixir` and `mix` on the `PATH`. rebar_mix shells out to mix to
  build Elixir deps, and to `elixir` to consolidate protocols.
- Hex for mix: `mix local.hex --force`. (Version managers such as mise or asdf
  may already install it for you.)

Build
-----

    $ rebar3 compile

This fetches jason from hex, builds it with mix, and runs
`rebar3 mix consolidate_protocols` (a post-compile hook), which writes
consolidated protocol beams to `_build/default/consolidated`.

Run
---

Build and start the release:

    $ rebar3 do compile, release
    $ _build/default/rel/example/bin/example foreground
    ...
    Decoded_Jason=#{<<"age">> => 44,<<"name">> => <<"Steve Irwin">>,
                    <<"nationality">> => <<"Australian">>}

Stop it with Ctrl+C. Other ways to run it:

    $ _build/default/rel/example/bin/example console      # interactive shell
    $ _build/default/rel/example/bin/example daemon       # in the background
    $ _build/default/rel/example/bin/example stop

For a self-contained release that includes ERTS and Elixir's runtime apps (it
runs without Erlang or Elixir installed on the target):

    $ rebar3 as prod do compile, release
    $ _build/prod/rel/example/bin/example foreground

Why `do compile, release` and not plain `rebar3 release`? On a fresh checkout
(an empty `_build`), plain `rebar3 release` / `rebar3 as prod release` crashes
in relx while copying jason into the release:

    ===> Uncaught error in rebar_core. ...
    Error: function_clause
    [{filename,join1,[<<"1.4.5">>,[], ...]}, ..., {rlx_assemble,copy_app,4,...}, ...]

jason's version reaches relx as a binary when rebar_mix builds it in the same
run. Running `compile` first avoids this, either with `do` as above or as a
separate `rebar3 compile`. A second `rebar3 release` also succeeds.

Protocol consolidation in the release
-------------------------------------

The relx `overlay` in `rebar.config` copies `_build/<profile>/consolidated` into
`releases/<vsn>/consolidated`, and `config/vm.args.src` puts that directory at
the front of the code path:

    -pa ${ROOTDIR}/releases/${RELEASE_VSN}/consolidated

The release start script substitutes `${ROOTDIR}` and `${RELEASE_VSN}` when it
renders `vm.args` from `vm.args.src`. (rebar_mix's README shows
`${REL_VSN}`, which current relx start scripts no longer export.) To check that
the consolidated protocol is the one loaded:

    $ _build/default/rel/example/bin/example daemon
    $ _build/default/rel/example/bin/example eval "'Elixir.Protocol':'consolidated?'('Elixir.Jason.Encoder')."
    true
    $ _build/default/rel/example/bin/example stop

`rebar3 shell` also works, but it doesn't add the consolidated directory to the
code path, so it uses the unconsolidated protocol. That behaves the same, just
dispatches more slowly.

Expected warnings
-----------------

These are harmless:

- `warning: found quoted keyword ...` while compiling jason. Elixir 1.20 warns
  about the quoted keys in jason's own `mix.lock`.
- `*WARNING* jason: Source code not found: 'Elixir.Jason....'.erl` during
  `rebar3 release`. relx checks for Erlang sources, and Elixir apps ship `.ex`
  files instead.
