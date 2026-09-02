---
name: step-parts
description: 从 step.parts 查找、评估和下载常见的可购买 CAD 零件，包括指定的现成执行器、伺服系统、电机、电子板、连接器、螺钉、螺栓、螺母、垫圈、轴承、支座和其他目录组件。当 Codex 需要在创建简化的占位符几何体之前搜索托管的 step.parts 目录、解析模糊零件名称、标准、别名或尺寸、选择匹配零件、获取规范 .step 文件、验证校验和或使用 step.parts API/OpenAPI/catalog 端点进行标准零件发现时使用。
---
# CAD Parts

Provenance: maintained in [earthtojake/text-to-cad](https://github.com/earthtojake/text-to-cad).
Use the installed local skill files as the runtime source of truth; the
repository link is only for provenance and release review.

## Overview

Use the hosted step.parts machine endpoints instead of scraping HTML or relying on local repository files. Treat `https://api.step.parts` as the canonical API origin and `https://www.step.parts` as the site/static-asset origin unless the user provides a different hosted mirror. Network/DNS failures are inconclusive: if `api.step.parts` cannot be reached from the sandbox, retry once with network permission before reporting a miss or using placeholder geometry. Do not describe a part as unavailable unless the API was reachable and returned no relevant candidates.

When a CAD assembly includes named off-the-shelf actuators, servos, motors, electronics boards, connectors, or other purchasable components, search step.parts before creating simplified placeholder geometry. For named servos, motors, and actuators, search both exact model strings and common aliases/vendor spellings before giving up. For example, `STS3215` may also appear as `ST3215`, `3215`, `Waveshare Feetech ST3215`, or under `family=feetech`. If the API was reachable and no exact or near-exact match is available, record the search miss and then use a documented envelope or simplified stand-in.

## Quick Workflow

1. Interpret the requested part into search terms and optional facets:
   - `q` for fuzzy tokens, standards, aliases, dimensions, source/product URLs, and attribute names/values.
   - `category`, `family`, `standard`, or `tag` when the user gives an exact facet.
2. Search `/v1/parts` and inspect `items`, `total`, and `facets`. For actuator model numbers, retry likely aliases, dropped letters, vendor names, and relevant family facets before treating an empty result as a miss.
3. If results are ambiguous, present the best few options with `id`, `name`, `standard`, and key attributes before choosing. If one result clearly matches, return the selected record details without downloading unless the user asked for a local STEP file.
4. When an exact or near-exact off-the-shelf actuator model is found, prefer downloading and using its STEP file unless there is a clear assembly-time reason to use a simplified envelope. Record that choice explicitly.
5. When the user asks to download or save a STEP file, download its `stepUrl`, then verify the file with the record's `sha256` when present.
6. Return the local path when downloaded, plus the selected part id and page/API URLs so the user can trace provenance.

## CAD Viewer Handoff

After completing step.parts work that creates or updates a local `.step` or `.stp` file, you must ALWAYS hand the explicit file path to `$cad-viewer` when that skill is installed. `$cad-viewer` must start CAD Viewer if it is not already running and return link(s) to the relevant created or updated file(s); if `$cad-viewer` is unavailable or startup fails, report that instead of silently omitting the handoff.

## Bundled Downloader

Use `scripts/download_step_part.py` for deterministic search, download, and checksum verification:

```bash
python scripts/download_step_part.py "M3 socket head 12" --download
python scripts/download_step_part.py --id iso4762_socket_head_cap_screw_m3x12 --download
python scripts/download_step_part.py "bearing 608zz" --limit 5
```

Useful options:

- `--origin`: override `https://api.step.parts` only when the user provides another hosted API origin.
- `--tag`, `--category`, `--family`, `--standard`: repeatable facet filters.
- `--out-dir`: override the download directory when the user asks for a specific destination.
- `--all`: with `--download`, download every result on the returned page as individual STEP downloads.
- `--overwrite`: replace an existing output file.

The script prints JSON to stdout. For searches, it prints matched records. For downloads, it prints saved file paths, checksums, and source URLs.

## API Reference

Read `references/step-parts-api.md` when you need endpoint details, field meanings, or query semantics. Prefer:

- `/v1/parts` for filtered search with absolute asset URLs.
- `/v1/parts/{id}` for one enriched record.
- Returned `stepUrl` for STEP downloads.
- `/v1/catalog/parts.index.json` for a compact discovery index.
- `/v1/catalog/schema` for field and family attribute meanings.
- `/v1/openapi.json` when generating a client or tool.

## Search Guidance

- Query tokens are ANDed by the API, so start specific but not overconstrained. For example, use `M3 SHCS 12` before adding exact family and standard filters.
- Values within one facet are ORed together, and selected `tag`, `category`, `family`, and `standard` fields are ANDed together. Use exact facets to narrow within known categories, then rank manually by name and attributes.
- Standards can be queried as `ISO 4762`, `ISO4762`, or the exact `standard.designation`.
- The `attributes` object contains family-specific facts such as `thread`, `lengthMm`, `bore1Mm`, `material`, `profileSeries`, `slotSizeMm`, and dimensions in millimeters.
- Part, GLB, and PNG URL patterns are predictable on `https://www.step.parts`; STEP URLs are environment-aware and may resolve to GitHub LFS media in production. Use catalog/API `stepUrl` for downloads.
