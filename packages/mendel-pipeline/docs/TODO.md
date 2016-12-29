Below are the things we need in order to achieve the Mendel v2 milestone.
- [x] Finalize Outlet API for pluggability/extendability
- [ ] Sourcemap support
- [x] Implement Node module generator (creates module bundle separately from source)
- [x] Implement Extract/[code split](https://twitter.com/samccone/status/797528710085652480) generator
- [x] Manifest v1 Outlet
- [ ] File system outlet
- [ ] Make the Mendel development middleware communicate with cache, instead of manifest or FS, and resolve faster
- [ ] Parallelize GST
  - Create general class for multi-process workers
- [ ] CSS outlet with PostCSS support
- [ ] Improve test coverage (find the most effective way of testing; can be fully black box integrational test)
- [ ] Validator: finding out misconfiguration as early as possible is good for user experience. Don't want to fail after launching application
- More...