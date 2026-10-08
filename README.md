# Model catalogue

`models.json` lists every image provider, image model and video model the app offers,
ranked for each thing a model does. The app ships this copy. It also fetches the
published one from `dimitritholen/letmecook-models` (`models.json` on `main`) at start
and then at most once a day, and keeps the last good one under its config dir
(`catalogue/models.json`). The code that reads it is `src/generate/catalogue.rs`.

## Top level

| Field | Meaning |
| --- | --- |
| `schema` | `2`. The app reads this schema and every older one; a file with a higher schema is refused, and the app keeps the catalogue it has. Raise it when an entry needs a field older app versions don't know (2 added `resolution_values` and `aspect_modes`), so those versions keep their own list instead of sending requests they can't build. OpenRouter video models needed no new field, so the schema stayed 2: an older app drops an entry whose video provider it has no client for, and keeps the rest. |
| `updated` | The day the file last changed, `YYYY-MM-DD`. A fetched file older than the bundled one is ignored. |
| `image.providers` | The image providers. The first one is the default. |
| `video.models` | The video models. |

## Image provider

| Field | Meaning |
| --- | --- |
| `id` | `openrouter` or `fal`. These are the only providers the app has a client for. |
| `name` | The name Settings shows. |
| `key_page` | Where the user makes a key. It must be on the provider's own site (`https://openrouter.ai/`, `https://fal.ai/`). |
| `models` | The models that make an image from text. Each one has `rank.image`. |
| `edit_models` | The models that change an existing image. Each one has `rank.edit`. The best active one is used. |
| `edit_result` | What an edit gives back, as New variant says it. |

## Image model

| Field | Meaning |
| --- | --- |
| `id` | What the provider calls it. It goes into the request as is. |
| `label` | What Settings calls it. |
| `family` | The model line (`flux`, `seedream`, `recraft`). |
| `price` | The price as the provider publishes it. |
| `seed` | Whether the provider takes a seed for it. |
| `rank` | `{"image": n}` in `models`, `{"edit": n}` in `edit_models`. 1 is the best. |
| `status` | `active` or `retired`. |

## Video model

| Field | Meaning |
| --- | --- |
| `id`, `name`, `family` | Its catalogue id (kept in Settings and in a plan), what the plan card calls it, and its model line (`seedance`, `wan`, `minimax`). |
| `provider` | `fal` or `openrouter`, the providers with a video client. The provider picks the client and the request shape: fal's queue, or OpenRouter's video API (`POST /api/v1/videos`). A model offered at both gets one entry each, with distinct ids and names, and the same `family`. |
| `image_to_video`, `text_to_video`, `start_end` | The fal endpoint for each mode it can do, or `null`. At OpenRouter it is the model id sent as `model`, the same for every mode the model does (`bytedance/seedance-2.0`); the entry's own `id` is `openrouter/<that id>`, so it never meets fal's. |
| `rank` | One rank per mode it has an endpoint for, and none for the other modes. 1 is the best. |
| `seconds` | `[shortest, longest]`, between 1 and 60. |
| `auto_seconds` | Whether it can choose its own length. |
| `resolutions` | Some of `480p`, `720p`, `1080p`. |
| `aspects` | `auto` first, then some of `21:9`, `16:9`, `4:3`, `1:1`, `3:4`, `9:16`. Modes left out of `inputs.aspect_modes` offer `auto` only. |
| `inputs` | What the model calls each input field: `image`, `end_image` (needed for `start_end`), `duration`, `resolution`, `aspect`, `audio` (or `null` without sound), and `duration_as` (`text` sends `"5"`, `number` sends `5`). An OpenRouter model has `auto_seconds` false and `duration_as` `number` (it takes whole seconds), `image` and `end_image` both `frame_images` (the one list holding a `first_frame` and a `last_frame`; `end_image` is `null` for a model that takes a first frame only), `auto_aspect` `null` (no `aspect_ratio` is sent for "auto", so the image, or the provider, decides) and `audio` `generate_audio` (sent every time, because OpenRouter turns sound on by default). `auto_aspect` is how the model spells "take the shape from the image"; `null` or left out sends no aspect for "auto". Field names are `[a-z_]`, at most 40 characters. Two optional fields cover models that spell things their own way. `resolution_values` maps each of the model's `resolutions` to what the model calls it (`{"480p": "480P"}`); it needs an entry for every resolution, each a word of letters and digits up to 16 characters, and without it the resolution is sent as is. `aspect_modes` lists the modes (`image_to_video`, `text_to_video`, `start_end`) whose request has the `aspect` field; in the other modes the image sets the shape, no aspect is sent, and only `auto` can be asked for. Left out, every mode has it. |
| `billing` | Used for the estimate when fal's pricing call gives nothing usable, and always at OpenRouter, which has no pricing call. A token-priced model with a dearer top resolution is a `per_second` table worked out from its tokens (width × height × 24 / 1024 a second) at each resolution. `{"tokens": {"usd_per_1k": n}}`, or `{"per_second": [["480p", n], ...]}` with one price for each resolution. |
| `estimate_factor` | Optional, 1 to 5 (1 when left out). The estimate from `billing` is multiplied by it, for a provider that bills more than its price list says. The OpenRouter video models use 2.2: a 2 s 480p Wan priced at $0.10 by the list was billed $0.2125 on 2026-10-08. Older apps ignore it and show the list price. |
| `price`, `wait` | What the plan card says about cost and time. |
| `status` | `active` or `retired`. |

## Adding or changing a model

- Use only providers and request shapes the app already knows. An image model must take
  the request its provider's client already sends. A video model must be a fal queue
  endpoint, or a model of OpenRouter's video API, whose inputs can be named in `inputs`.
- Give a model a rank for each mode it serves. Lower is better. Ranks within a mode
  should not tie.
- Retire a model by setting `status` to `retired`; don't delete it. A retired model is
  never offered or picked, but old settings and assets still find its name. Settings
  that name it fall back to the best active model of that provider or mode.
- Raise `updated` with every change.
- Ids and endpoints contain only letters, digits, `.`, `_`, `-` and `/`, with nothing
  empty between slashes.

## How the app reads a file

The app ignores unknown fields. It drops an invalid single entry, such as a provider or
model, and logs a warning. It refuses the whole file, and keeps the catalogue it has,
in these cases:

- the file isn't JSON
- the file is larger than 256 KB
- the schema is higher than the app reads
- `updated` isn't a date
- after the drops, there is no active model for a provider or video mode that the
  bundled file has

## Publishing a change

Edit `models.json` here, run `cargo test --lib generate::catalogue`, and copy the same file to
`main` of the public repo `dimitritholen/letmecook-models`. Raise `updated` with every change: the app
ignores a published file older than the copy it ships.
