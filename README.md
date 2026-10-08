# Model catalogue

`models.json` lists every image provider, image model and video model the app offers,
ranked for each thing a model does. The app ships this copy. It also fetches the
published one from `dimitritholen/letmecook-models` (`models.json` on `main`) at start
and then at most once a day, and keeps the last good one under its config dir
(`catalogue/models.json`). The code that reads it is `src/generate/catalogue.rs`.

## Top level

| Field | Meaning |
| --- | --- |
| `schema` | `1`. A file with a higher schema is refused, and the app keeps the catalogue it has. |
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
| `id`, `name`, `family` | Its catalogue id (kept in Settings and in a plan), what the plan card calls it, and its model line (`seedance`, `wan`). |
| `provider` | `fal`. It is the only provider with a video client. |
| `image_to_video`, `text_to_video`, `start_end` | The fal endpoint for each mode it can do, or `null`. |
| `rank` | One rank per mode it has an endpoint for, and none for the other modes. 1 is the best. |
| `seconds` | `[shortest, longest]`, between 1 and 60. |
| `auto_seconds` | Whether it can choose its own length. |
| `resolutions` | Some of `480p`, `720p`, `1080p`. |
| `aspects` | `auto` first, then some of `21:9`, `16:9`, `4:3`, `1:1`, `3:4`, `9:16`. |
| `inputs` | What the model calls each input field: `image`, `end_image` (needed for `start_end`), `duration`, `resolution`, `aspect`, `audio` (or `null` without sound), and `duration_as` (`text` sends `"5"`, `number` sends `5`). `auto_aspect` is how the model spells "take the shape from the image". Field names are `[a-z_]`, at most 40 characters. |
| `billing` | Used for the estimate when fal's pricing call gives nothing usable. `{"tokens": {"usd_per_1k": n}}`, or `{"per_second": [["480p", n], ...]}` with one price for each resolution. |
| `price`, `wait` | What the plan card says about cost and time. |
| `status` | `active` or `retired`. |

## Adding or changing a model

- Use only providers and request shapes the app already knows. An image model must take
  the request its provider's client already sends. A video model must be a fal queue
  endpoint whose inputs can be named in `inputs`.
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
- the schema isn't 1
- `updated` isn't a date
- after the drops, there is no active model for a provider or video mode that the
  bundled file has

## Publishing a change

Edit `models.json` here, run `cargo test --lib generate::catalogue`, and copy the same file to
`main` of the public repo `dimitritholen/letmecook-models`. Raise `updated` with every change: the app
ignores a published file older than the copy it ships.
