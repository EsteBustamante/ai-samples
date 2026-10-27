# AEM Local Instances Sync

A [Claude Code](https://claude.com/claude-code) slash command that copies content and `/conf`
branches between two local AEM instances, directly on disk, with both instances shut down.

It's for the moment you drop a fresh AEM SDK QuickStart and need your existing local content in
it — no content packages, no Package Manager. You give it a source and a target; it builds and
runs a Jackrabbit [`oak-upgrade`](https://jackrabbit.apache.org/oak/docs/migration.html)
sidegrade, asks before anything destructive, and verifies the result afterwards.

## Scope

- Filesystem **TarMK only** — not MongoMK or RDB.
- **Not for AEM as a Cloud Service environments.** There the tool is the Content Transfer Tool.
- Local developer instances, not production.

## Install

Copy `oak-migrate.md` into `.claude/commands/` in your project, or into `~/.claude/commands/`
to have it available everywhere. Then:

```
/oak-migrate <source-instance> <target-instance>
```

Optional flags: `--merge`, `--with-versions`, `--verify`.

You'll also need an `oak-upgrade` jar, and an `oak-run` jar if you want the verification step:

```bash
mvn dependency:get -Dartifact=org.apache.jackrabbit:oak-upgrade:2.6.0:jar
mvn dependency:get -Dartifact=org.apache.jackrabbit:oak-run:1.22.9:jar
```

## Before first use

Run the discovery step (step 0 in the command) to list the real children of `/content`,
`/content/dam` and `/conf` in your source repository, then edit the path list in the command
file. **The list that ships with it is an example, not a default** — `oak-upgrade` silently
ignores an include path that doesn't exist in the source and still exits 0, so an unedited list
can produce a "successful" run that copies nothing.

## Workflow

1. Shut down both AEM instances.
2. Run the command with a source and a target.
3. Answer the datastore question — copy binaries, or share a datastore.
4. Approve the printed command.
5. Check the verification output: every path should match between source and target.
6. Don't start the target until the content is final; it reindexes on first boot.

## Notes

- Recent `oak-upgrade` builds (2.x) are compiled for Java 17+ and fail on older JVMs with
  `UnsupportedClassVersionError`. `oak-run` 1.22.x runs on Java 11.
- Pass the `repository` directories as arguments, not `.../segmentstore` — the tool appends that
  itself.
- Exit code 0 means the jar ran, not that your content arrived. Verify.
- The shell snippets cover macOS and Linux. Windows is not covered.
- The command body is plain Markdown, so the procedure ports to any agent that reads
  instructions from your repo; `.claude/commands/` and `/slash` invocation are Claude Code
  specifics.

## License

MIT.
