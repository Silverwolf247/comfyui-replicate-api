# ComfyUI Text-to-Image API (via Replicate)

Generate images by running a ComfyUI workflow through Replicate's hosted
`fofr/any-comfyui-workflow` model. This is a plain HTTPS JSON API: submit a
workflow, poll for the result, receive image URLs.

- **Base URL**: `https://api.replicate.com/v1`
- **Host**: `api.replicate.com`
- **Auth**: HTTP Bearer token in the `Authorization` header. Token name: `REPLICATE_API_TOKEN`.
- **Usage pattern**: async two-phase — `create-prediction` returns immediately with an id;
  poll `get-prediction` every few seconds until `status` is `succeeded` or `failed`.
  Cold start can take 30-60s; warm runs finish in ~6s.

## Endpoints

### 1. Create prediction (submit a ComfyUI workflow)

`POST /predictions`

Starts an image generation run. **This endpoint spends account credit (~$0.02 per run), so treat it as a mutating operation.**

Body parameters (JSON):

| name | in | type | required | description |
|---|---|---|---|---|
| `version` | body | string | yes | Model version id. Always use `16d0a881fbfc066f0471a3519a347db456fe8cbcbd53abb435a50a74efaeb427`. |
| `input` | body | JSON object (pass as an object, not a string) | yes | `{"workflow_json": "<ComfyUI workflow serialized as a JSON string>", "randomise_seeds": true, "output_format": "webp", "output_quality": 90}` |

Response `201`: `{"id": "<prediction-id>", "status": "starting", ...}` — keep the `id` for polling.

Example:

```bash
curl -X POST https://api.replicate.com/v1/predictions \
  -H "Authorization: Bearer $REPLICATE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"version":"16d0a881fbfc066f0471a3519a347db456fe8cbcbd53abb435a50a74efaeb427","input":{"workflow_json":"<stringified workflow>","randomise_seeds":true,"output_format":"webp","output_quality":90}}'
```

### 2. Get prediction (poll for the result)

`GET /predictions/{id}`

Read-only. Poll every 3-5 seconds. Response field `status` is one of
`starting | processing | succeeded | failed | canceled`. When `succeeded`,
`output` is an array of image URLs (e.g. on `replicate.delivery`). Return
those URLs to the user directly — do not download the image bytes. Note:
output URLs expire after ~1 hour.

| name | in | type | required | description |
|---|---|---|---|---|
| `id` | path | string | yes | Prediction id from create-prediction. |

### 3. List predictions

`GET /predictions`

Read-only. Returns `{"results": [...]}` — the account's recent predictions
(possibly empty). Useful as a cheap connectivity/auth check.

## The workflow template

`workflow_json` must be a ComfyUI **API-format** workflow, serialized to a
string. Use this known-good SDXL text-to-image template (the checkpoint
`SDXL-Flash.safetensors` is preinstalled on the Replicate model). Only edit
the three documented knobs; keep every other node untouched:

```json
{
  "3": {"class_type": "KSampler", "inputs": {"seed": 156680208700286, "steps": 10, "cfg": 2.5, "sampler_name": "dpmpp_2m_sde", "scheduler": "karras", "denoise": 1, "model": ["4", 0], "positive": ["6", 0], "negative": ["7", 0], "latent_image": ["5", 0]}},
  "4": {"class_type": "CheckpointLoaderSimple", "inputs": {"ckpt_name": "SDXL-Flash.safetensors"}},
  "5": {"class_type": "EmptyLatentImage", "inputs": {"width": 1024, "height": 1024, "batch_size": 1}},
  "6": {"class_type": "CLIPTextEncode", "inputs": {"text": "beautiful scenery nature glass bottle landscape, purple galaxy bottle,", "clip": ["4", 1]}},
  "7": {"class_type": "CLIPTextEncode", "inputs": {"text": "text, watermark", "clip": ["4", 1]}},
  "8": {"class_type": "VAEDecode", "inputs": {"samples": ["3", 0], "vae": ["4", 2]}},
  "9": {"class_type": "SaveImage", "inputs": {"filename_prefix": "ComfyUI", "images": ["8", 0]}}
}
```

Editable knobs:

- **Prompt**: node `"6"` → `inputs.text` (positive prompt). Node `"7"` is the negative prompt.
- **Resolution**: node `"5"` → `inputs.width` / `inputs.height` (multiples of 8; e.g. 1024x1024, 768x1280).
- **Seed**: node `"3"` → `inputs.seed`, and set `"randomise_seeds": false` in `input` to make it take effect. Leave `randomise_seeds: true` for random results.

## Notes

- Poll responses echo the submitted `workflow_json` back in the `input`
  field; keep workflows small.
- A `402` response means the Replicate account is out of credit.
- Community-model predictions MUST be created via `POST /predictions` with a
  `version` field (the `POST /models/{owner}/{name}/predictions` shortcut is
  only for official models and returns 404 here).
