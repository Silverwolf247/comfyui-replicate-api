Knobs (edit ONLY these fields in the template graph):
- Prompt: node "6" → inputs.text (positive). Node "7" → inputs.text is the negative prompt (keep short, e.g. "text, watermark").
- Resolution: node "5" → inputs.width / inputs.height (multiples of 8: 1024x1024 square, 768x1280 portrait, 1280x768 landscape). Batch: node "5" → inputs.batch_size (1-4).
- Seed: node "3" → inputs.seed — only takes effect when randomise_seeds is false.

How to submit: serialize the edited graph to a JSON STRING and pass it as input.workflow_json on the create-prediction endpoint (MUTATING, spends ~$0.02 per run):
params = { "version": "16d0a881fbfc066f0471a3519a347db456fe8cbcbd53abb435a50a74efaeb427", "input": { "workflow_json": "<the edited graph serialized as a string>", "randomise_seeds": true, "output_format": "webp", "output_quality": 90 } }
Always use exactly that version id. Then poll get-prediction every 3-5s until status is succeeded or failed.

Delivering the result: when the prediction returns an image URL, persist it with library.save({ url: "<the image url>", name: "<short-name>.webp" }). This downloads and stores the image so it displays inline in chat and never expires — do NOT just paste the raw link (a link renders as text, and the generation URL expires in about 1 hour). After saving, describe what you made in one sentence; the saved image appears inline on its own.

Iterating on a previous image ("same image but ..."): reuse the previous workflow, set node "3" inputs.seed to the seed from that run, set "randomise_seeds": false, and change only the knob the user asked about. For a fresh variation keep randomise_seeds true.
