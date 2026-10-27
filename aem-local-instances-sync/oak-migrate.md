---
description: Copy content and /conf branches between two local AEM SDK Oak repositories (sidegrade)
argument-hint: "<source-path> <target-path> [--merge] [--with-versions] [--verify]"
---

<!--
oak-migrate — a Claude Code slash command for refreshing a local AEM SDK
without repackaging your content.

WHAT IT DOES
  Builds and runs a Jackrabbit `oak-upgrade` sidegrade that copies selected
  content and /conf branches from one shut-down local Oak repository into
  another. Use it when you drop a new AEM SDK QuickStart and want your
  existing local content in it, with no Package Manager round trip.

WHAT IT IS NOT FOR
  - AEM as a Cloud Service environments. There the tool is the
    Content Transfer Tool, not this.
  - MongoMK or RDB node stores. Filesystem TarMK only.
  - Production repositories. This is a local developer convenience.

INSTALL
  Drop this file in `.claude/commands/oak-migrate.md` in your project
  (or `~/.claude/commands/` to have it everywhere), then run
  `/oak-migrate <source> <target>`. The `description` and `argument-hint`
  keys above are Claude Code specific; the body is plain Markdown, so the
  procedure ports to any agent that reads instructions from your repo.

REQUIREMENTS
  - `oak-upgrade` jar. Recent builds (2.x) are compiled for Java 17+.
      mvn dependency:get -Dartifact=org.apache.jackrabbit:oak-upgrade:2.6.0:jar
  - Optional, for verification: an `oak-run` jar. 1.22.x works and reads
    newer segment stores; it runs on Java 11.
      mvn dependency:get -Dartifact=org.apache.jackrabbit:oak-run:1.22.9:jar
  - Both AEM instances fully shut down.

  Tested on macOS with AEM SDK QuickStarts. The shell snippets below handle
  macOS and Linux; Windows is not covered.

BEFORE YOU USE IT
  Step 0 is not optional. The path list in this file is an EXAMPLE from a
  multi-brand project. Your repository has different roots, and an include
  path that does not exist in the source is silently ignored.

License: MIT. Attribution welcome but not required.
-->

Build an `oak-upgrade` command to copy content from one local Oak repository to another.
Arguments received: `$ARGUMENTS`

Filesystem TarMK only. If the paths look like anything other than a local SDK instance — a
cloud environment, a MongoDB or JDBC URI — stop and say so.

## Paths to copy

> **EDIT THIS SECTION before first use.** The entries below are an example from a project with
> four brands. Replace them with your own roots, discovered in step 0, and keep the three groups
> separate so the DAM list cannot be forgotten.

Content:
`/content/brand-a,/content/brand-b,/content/experience-fragments,/content/campaigns,/content/cq:tags`

DAM — **one branch per brand.** `/content/dam/brand-a` alone is not enough if your assets are
split per brand:
`/content/dam/brand-a,/content/dam/brand-b`

Configuration:
`/conf/global,/conf/brand-a,/conf/brand-b`

Hard rules, regardless of project:

- Never a bare `/conf`. It overwrites Adobe's `/conf/global` and any other tenant's config.
- Never a bare `/content/dam`. It drags in `appdata`, `assetinsights`, `trainingmodels` and
  other Adobe-owned branches.
- Never `/apps`. That ships through your `ui.apps` build, not through content migration.

## Steps

**0. Discover the real content roots (first run, and whenever the project changes).**

Do this before trusting the list above. `oak-run` reads a shut-down segment store offline, so it
costs nothing and needs no running instance. Save as `roots.groovy`:

```groovy
['/content','/content/dam','/conf'].each { p -> def n = session.getRoot(); p.substring(1).split('/').each { s -> n = n.getChildNode(s) }; println "CHILDREN ${p} :: " + (n.exists() ? n.getChildNodeNames().collect{it}.sort().join(', ') : "MISSING") }
```

```bash
printf ":load roots.groovy\n:quit\n" \
  | java -jar /path/to/oak-run-1.22.9.jar console <source>/crx-quickstart/repository/segmentstore 2>&1 \
  | grep "^CHILDREN"
```

Show the user the three listings and confirm which roots belong to the project before building
any command. Adobe-owned siblings (`launches`, `usergenerated`, `projects`, `appdata`,
`assetinsights`, `trainingmodels`, `versionhistory`) are normally **not** migrated.

**1. Validate arguments.** `$1` = source, `$2` = target. If either is missing, ask and stop.
If a path ends in `crx-quickstart`, append `/repository`. Reject `mongodb://` and `jdbc:` URIs.
`test -d` the source; stop if absent. The target may not exist yet on a clean instance — warn,
then continue.

Pass the **`repository` directories** as the positional arguments, never `.../segmentstore`.
The tool appends `segmentstore` itself; passing it explicitly fails with
`.../segmentstore/segmentstore does not exist or is not a directory`.

State the direction out loud before running: `$1` is written onto `$2`. `du -sh` on both
repositories, plus each instance's `version.txt`, makes it obvious which one holds the content.

**2. Confirm both instances are shut down.** A sidegrade needs both repositories closed.

```bash
lock() { lsof "$1/segmentstore/repo.lock" 2>/dev/null || fuser "$1/segmentstore/repo.lock" 2>/dev/null; }
```

If neither `lsof` nor `fuser` is installed, fall back to `pgrep -fl 'crx-quickstart|aem-sdk'`.
If anything holds a lock, name the process and stop.

**3. Locate the jar** — first hit wins:

```bash
ls ./oak-upgrade-*.jar ~/oak-upgrade-*.jar 2>/dev/null
ls ~/.m2/repository/org/apache/jackrabbit/oak-upgrade/*/oak-upgrade-*.jar 2>/dev/null
```

If there is none, do not invent a path. Give the user the `mvn dependency:get` line from the
header and stop.

**3b. Match the Java runtime to that jar.** `oak-upgrade` 2.x is compiled for Java 17+ and dies
on older JVMs with `UnsupportedClassVersionError`; older 1.x builds used with AEM 6.5 run on
Java 11. Resolve a JDK explicitly instead of trusting `PATH`:

```bash
case "$(uname -s)" in
  Darwin) JAVA17="$(/usr/libexec/java_home -v 17+ 2>/dev/null)/bin/java" ;;
  *)      JAVA17="$(ls -d /usr/lib/jvm/*17*/bin/java /usr/lib/jvm/*21*/bin/java 2>/dev/null | head -1)" ;;
esac
"$JAVA17" -version
```

Run it with `-Xmx4g` and these flags, which recent JDKs need for Oak's reflection:

```
--add-opens java.base/java.lang=ALL-UNNAMED
--add-opens java.base/java.nio=ALL-UNNAMED
--add-opens java.base/java.util=ALL-UNNAMED
--add-opens java.base/sun.nio.ch=ALL-UNNAMED
```

**4. Decide on binaries — ask the user, do not guess.** Compare the two datastores:

```bash
inode() { case "$(uname -s)" in Darwin) stat -f %i "$1" ;; *) stat -c %i "$1" ;; esac; }
```

Report whether each `datastore/` exists and whether they share an inode, then ask:

- *Copy binaries* → `--copy-binaries --src-datastore=<source>/datastore --datastore=<target>/datastore`
- *Shared datastore* → add nothing; only blob references are copied, and the target must keep
  using the source's datastore

If the source has no `datastore/` but has an S3DataStore config in `crx-quickstart/install`,
say that `--src-s3config` is required and ask for the path rather than guessing.

**5. Optional flags from `$ARGUMENTS`:**

- `--merge` → add `--merge-paths=` with the same list as `--include-paths`, merging into existing
  subtrees instead of replacing them. Warn that a merge can fail with `OakConstraint0030` when a
  node was renamed on either side while keeping its UUID.
- `--with-versions` → omit `--copy-versions=false --copy-orphaned-versions=false`. Versions are
  skipped by default, which is much faster.
- `--verify` → run step 7 afterwards. Recommended whenever the path list changed.
- Always `--skip-checkpoints`. It suppresses the warning that checkpoints are not copied;
  checkpoints are skipped anyway once custom include paths are set, so the target reindexes on
  first startup either way. Tell the user not to start the target until the content is final.

**6. Print, confirm, run.** Show the whole command in a fenced `bash` block with `\`
continuations, then state how many paths, whether binaries are copied, and whether versions are.
Ask before running.

Run it in the background — a real sidegrade can take a long time — and redirect with
`> oak-migrate.log 2>&1`. Do **not** pipe through `tee`: that masks the jar's exit code behind
the pipeline's.

Then read the log. **Exit 0 means the jar ran, not that your content arrived.** Look for
`RepositorySidegrade - Copying node #` progress lines and for exceptions. A run that finishes in
seconds copied almost nothing.
`WARN ... Missing provider for nrt/sync index` is benign.

**7. Verify.** Same offline trick as step 0, now path by path. Save as `paths.groovy` with your
full include list:

```groovy
['/content/brand-a','/content/dam/brand-a','/conf/global'].each { p -> def n = session.getRoot(); p.substring(1).split('/').each { s -> n = n.getChildNode(s) }; println "PATH ${p} :: " + (n.exists() ? "EXISTS children=" + n.getChildNodeCount(1000) : "MISSING") }
```

```bash
for repo in <source> <target>; do
  printf ":load paths.groovy\n:quit\n" \
    | java -jar /path/to/oak-run-1.22.9.jar console "$repo/segmentstore" 2>&1 \
    | grep "^PATH" > "verify-$(basename "$repo").txt"
done
diff verify-*.txt && echo "IDENTICAL — every path matches the source"
```

Note the 1.22.x console is read-only unless given `--read-write`; there is no `--read-only` flag.
Keep each Groovy statement on one line — closures defined across `:load`ed lines are not visible
to later lines in `groovysh`.

Every path present in the source must be present in the target with the same child count. A
`MISSING` row in the target means that include path was wrong or absent in the source — the one
failure mode the exit code will never show you.

**8. After startup.** The target reindexes on first boot. Before trusting it, check the async
indexing lanes via the `IndexStats` MBeans
(`/system/console/jmx/org.apache.jackrabbit.oak%3Aname%3Dasync%2Ctype%3DIndexStats`) and let the
indexer finish rather than restarting into a half-indexed repository.
