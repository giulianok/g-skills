# Full example

A complete module that follows every rule in `SKILL.md`.

```ts
import fs from 'node:fs';
import path from 'node:path';
import semver from 'semver';
import { z } from 'zod';

const pkgSchema = z.object({
  name: z.string(),
  version: z.string(),
  engines: z.record(z.string(), z.string()).optional(),
});
const releasesSchema = z.record(z.string(), z.string());

type Pkg = z.infer<typeof pkgSchema>;
type Releases = z.infer<typeof releasesSchema>;
type Env = 'dev' | 'prod';

/**
 * A release that matched the package's engine constraint.
 */
type MatchedRelease = {
  version: string;
  tag: string;
};

const RELEASES_ENDPOINT: Record<Env, string> = {
  dev: 'https://releases.dev.example.com/api/releases',
  prod: 'https://releases.example.com/api/releases',
} as const;

let CWD = '.';
let RELEASES: Releases | null = null;

/**
 * Returns the package.json as an object.
 * @param cwd where to find the package.json file
 * @returns the parsed package.json
 */
const getPkg = (cwd = CWD): Pkg =>
  pkgSchema.parse(JSON.parse(fs.readFileSync(path.join(cwd, 'package.json'), 'utf-8')));

/**
 * Fetches all releases once and caches them for later calls.
 * @param env target environment
 * @returns map of version -> release tag
 */
const getReleases = async (env: Env): Promise<Releases> => {
  RELEASES =
    RELEASES ??
    releasesSchema.parse(await fetch(RELEASES_ENDPOINT[env]).then((r) => r.json()));

  return RELEASES;
};

/**
 * Checks whether a version string is a stable (non-prerelease) semver.
 * @param version version to check
 * @returns true when the version is valid and stable
 */
const isStable = (version: string): boolean =>
  !!semver.valid(version) && !semver.prerelease(version);

/**
 * Finds the releases that satisfy the package's engine range, newest first.
 * @param env target environment
 * @param pkg package to match against
 * @returns matching releases sorted from newest to oldest
 */
export const findMatchingReleases = async (
  env: Env,
  pkg: Pkg = getPkg()
): Promise<MatchedRelease[]> => {
  const range = pkg.engines?.app;
  const releases = await getReleases(env);

  if (!range) {
    throw new Error(`Package "${pkg.name}" does not declare an "app" engine range`);
  }

  const matches = Object.entries(releases)
    .filter(([version]) => isStable(version))
    .filter(([version]) => semver.satisfies(version, range))
    .map(([version, tag]) => ({ version, tag }));

  return [...matches].sort((a, b) => semver.rcompare(a.version, b.version));
};

/**
 * Sets the working directory used to resolve package.json.
 * @param cwd new working directory
 */
export const setCwd = (cwd: string): void => {
  CWD = cwd;
};
```

## What it shows

- **Layout:** imports → schemas → types → constants → module `let` caches → functions.
- **JSDoc** on every function, and on the non-trivial `MatchedRelease` type.
- **Arrow functions:** expression bodies for `getPkg` and `isStable`, block bodies only where statements are needed.
- **Immutability:** a chained `filter`/`filter`/`map`, and the array is copied before `sort` (`sort` mutates).
- **Allowed mutation:** the `RELEASES` lazy cache and `CWD`, both module state.
- **Guard clause:** throws early with the bad value in the message.
- **Spacing:** declarations grouped, the `if` block surrounded by blank lines, a blank line before `return`.
