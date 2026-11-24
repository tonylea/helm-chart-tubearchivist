# Changelog

All notable changes to this project will be documented in this file. See [commit-and-tag-version](https://github.com/absolute-version/commit-and-tag-version) for commit guidelines.

## [2.1.0](https://github.com/tonylea/helm-chart-tubearchivist/compare/v2.0.0...v2.1.0) (2026-11-24)

### Features

* serve Redis's metrics through redis_exporter ([e7ddd76](https://github.com/tonylea/helm-chart-tubearchivist/commit/e7ddd7696bfe38711f17f3bbb57bd497b3ffe3c2))

## [2.0.0](https://github.com/tonylea/helm-chart-tubearchivist/compare/v1.0.0...v2.0.0) (2026-11-24)

### ⚠ BREAKING CHANGES

* the chart no longer creates the download claim. Set
  `persistence.download.existingClaim`; `persistence.download.storageClass`
  is gone. The cache claim's size is now `persistence.cache.size`.
* the chart no longer sets requests. Set `resources` and
  `redis.resources`; the README suggests starting figures.

### Features

* take requests from values instead of the templates ([3bf54d0](https://github.com/tonylea/helm-chart-tubearchivist/commit/3bf54d0642a2082534118a460be259bbabb2b716))
* take the download claim and cache size from values ([ad3c4e8](https://github.com/tonylea/helm-chart-tubearchivist/commit/ad3c4e8520ab72f0c78e6d82a91e0674d630dfd7))

## 1.0.0 (2026-11-24)

### Features

* deploy Tube Archivist with its own Redis ([d39c9e7](https://github.com/tonylea/helm-chart-tubearchivist/commit/d39c9e735df932a491062113123c5ac1b730b7b2))
* require parentRefs when the HTTPRoute is enabled ([d7b0daf](https://github.com/tonylea/helm-chart-tubearchivist/commit/d7b0daffc90fec00eab6b6b120ef06ec5c859588))
