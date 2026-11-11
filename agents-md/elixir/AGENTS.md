## Security rules for Elixir/Phoenix code

These rules prevent the most serious bugs: remote code execution (RCE), unsafe deserialization, SSRF, and file access. Treat every value from a request, socket, upload, webhook, job argument, or user-editable database field as **untrusted**. These rules apply to code you write and to code you change.

1. **Never cast ownership or permission fields from user input (mass assignment).**
   Changesets used by user-facing forms must not `cast/3` fields like `:user_id`, `:account_id`, `:org_id`, `:post_id`, `:role`, or `:admin`. An attacker can add any field to a request. Set these fields on the server, from the current user and from records you've already authorized:

   ```elixir
   post = Repo.get_by!(Post, id: post_id, user_id: user.id)

   %Comment{user_id: user.id, post_id: post.id}
   |> Comment.changeset(params)
   ```

   A changeset only filters fields. It doesn't check permissions. Always pair it with server-side authorization (rule 8).

2. **Never call `:erlang.binary_to_term` on untrusted data, even with `[:safe]`.**
   `[:safe]` only blocks new atoms. A decoded term can still contain functions that run later, e.g. when passed to `Enum`. Use JSON for data from outside the app. If you must decode Erlang terms, use `Plug.Crypto.non_executable_binary_to_term(bin, [:safe])`. For signed tokens, use `Phoenix.Token` or `Plug.Crypto`.

3. **Never build shell commands from input.**
   Don't use `System.shell` or `:os.cmd` with untrusted data. Use `System.cmd(executable, args)` with a fixed executable and an argument list. Validate arguments against an allowlist. Put `"--"` before user-supplied arguments when the tool supports it, so input can't become a flag (e.g. `--output=/etc/...`).

4. **Never let input choose modules, functions, or atoms.**
   No `apply/3`, `Module.concat`, or `String.to_atom` with untrusted data. Map input to a fixed set with a `case` or a map lookup:

   ```elixir
   @sorts %{"name" => :name, "date" => :inserted_at}
   field = Map.get(@sorts, params["sort"], :inserted_at)
   ```

   Don't decode JSON with `keys: :atoms`. Atoms are never garbage collected, so this can crash the VM.

5. **Treat outbound HTTP to user-supplied URLs as dangerous (SSRF).**
   When a URL comes from a user (webhooks, "import from URL", avatars, link previews):
   - Allow only `https` and an allowlist of hosts where possible.
   - Resolve the hostname and reject private, loopback, link-local, and metadata addresses: `127.0.0.0/8`, `10/8`, `172.16/12`, `192.168/16`, `169.254.0.0/16`, `::1`, `fc00::/7`.
   - Disable redirects (`Req.get(url, redirect: false)`), or re-validate every hop.
   - Req `base_url` and Tesla `BaseUrl` are **not** security boundaries. An absolute URL overrides them.

6. **Never use user input in file paths.**
   For uploads and downloads, generate server-side file names (e.g. UUIDs) and keep the user's name only as metadata. If a path segment must come from input, validate it with `Path.safe_relative` and join it to a fixed base directory. Before calling `send_file`, `send_download`, or `File.*`, check that the resolved path is inside that directory. When extracting archives, reject entries containing `..` or absolute paths.

7. **Never interpolate into SQL.**
   Use Ecto queries with `^pinned` values. In raw SQL (`Repo.query`, `Ecto.Adapters.SQL.query`), use placeholders and a params list, e.g. `Repo.query("... WHERE id = $1", [id])`. Use `fragment("lower(?)", ^value)`, never string building.

8. **Authorize on the server, from the session, every time.**
   - Who the user is comes from `conn.assigns.current_scope` / `current_user` (or `socket.assigns`), never from params or hidden form fields.
   - Scope every lookup to its owner or tenant: `Repo.get_by!(Post, id: id, user_id: user.id)`.
   - In LiveView, check authorization inside each `handle_event`. Clients can send any event, even ones for buttons that were never rendered.

9. **Check dependencies before adding or upgrading them.**
   Run `mix hex.audit` (Hex 2.5.0+ reports security advisories and retired packages). Don't add packages that are retired or have open advisories. When a library takes user input (pagination cursors, file processing, parsers, HTML/PDF rendering), check how it handles that input. Libraries have had RCE bugs.

10. **Verify before you finish.**
    Run these and fix what they report:

    ```sh
    mix hex.audit
    mix sobelow --skip --exit low
    ```

    Never add `# sobelow_skip` or `.sobelow-skips` entries to make a warning go away. Only a human can approve a skip, after confirming it's a false positive, and it needs a one-line reason in a comment.

If a task seems to require breaking one of these rules, stop and explain the risk to the user instead of working around it.

<!-- From the elixir-security-audit skill, sponsored by Paraxial.io (https://paraxial.io). -->
