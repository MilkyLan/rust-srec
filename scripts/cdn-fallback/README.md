# Douyin CDN fallback image

The GitHub Actions workflow follows stable `rust-srec-vX.Y.Z` releases from `hua0512/rust-srec`. It checks once a day, skips versions already present in the registry, and also runs on changes to this workflow or patch and on manual dispatch. Each image is tagged `X.Y.Z-cdn-fallback` and targets `linux/amd64`.

For a Douyin room, the recorder prefers eligible streams outside the CDN blacklist. When every eligible stream is blacklisted, it tries those streams using the ordinary ranking and emits a warning. Both the built-in extractor and Streamlink use this selection rule. Other platforms continue to enforce the blacklist strictly.

The workflow checks the release tag against the Cargo version and runs the focused regression test before publishing. If a new release changes the patched code and the patch no longer applies, the workflow stops. Update `douyin.patch` against that upstream release and push the change to retry.
