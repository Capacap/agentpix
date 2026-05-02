---
name: agentpix
description: Generate or edit images using Gemini's native image generation. TRIGGER when the user wants to create, edit, or transform images, or references agentpix, image generation, or Nano Banana. DO NOT TRIGGER for image compression, optimization, or format conversion.
---

# Agentpix Image Generation

Agentpix is a CLI that wraps Google's Gemini image models (also known as Nano Banana) for image generation and editing. For this work, function as a creative collaborator. Image generation is exploratory; stochastic model variance means most tasks require multiple attempts to converge on what the user wants. Each rejected image informs direction by revealing what works and what does not. The generation loop (assess, compose, execute, diagnose) is the central operating mode.

## CLI Reference

The agentpix binary lives in this skill's directory, not on PATH. Construct the binary path from the skill location provided by the system when this skill was invoked.

```
agentpix -p <prompt> -o <output> [-i <input>...] [-s <session>] [-m model] [-r <ratio>] [-z 1K|2K|4K] [-t min|high] [-f]
```

| Flag | Required | Description |
|------|----------|-------------|
| `-p` | yes | Text prompt |
| `-o` | yes | Output PNG file path (must end in `.png`) |
| `-i` | no | Input image for editing/reference (repeatable; supports png, jpg/jpeg, webp, heic, heif) |
| `-s` | no | Session file to continue from |
| `-m` | no | Model: `flash` (default), `pro`, `flash-2.5`, `flash-3.1`, `pro-3.0` |
| `-r` | no | Aspect ratio (default `1:1`): `1:1`, `2:3`, `3:2`, `3:4`, `4:3`, `9:16`, `16:9`, `21:9` |
| `-z` | no | Output size: `1K`, `2K`, or `4K` (`flash-3.1`, `pro-3.0` only) |
| `-t` | no | Thinking level: `min` (default), `high` (`flash-3.1` only) |
| `-f` | no | Overwrite output and session files if they already exist |

Pass `-i` multiple times for multiple reference images. Each file must be under 7 MB. Model-specific limits: Flash 2.5 supports up to 3 input images, Flash 3.1 and Pro 3.0 support up to 14. The CLI validates file existence, MIME type, and size before making the API call. If the input count exceeds the model limit, the error message suggests switching to a model with a higher cap.

Output is always PNG. The Gemini API returns PNG data; other output formats are not supported.

### Sessions

Every generation produces a session file alongside the image output (e.g., `cat.png` produces `cat.session.json`). It records the history for multi-turn interactions. Continuing a session gives the model the full context of every prior prompt and output in the chain, so it can make targeted changes without redescription. This makes sessions the preferred tool for iterative refinement where multiple targeted changes are spread across turns. Sessions can also be used to create alternative camera angles of the same motif.

A session cannot switch models. The chosen model must match the one recorded in the file. Bare aliases are resolved before comparison, so `-m flash` and `-m flash-3.1` are equivalent when `flash` aliases to `flash-3.1`. If you must pivot to a different model, start a new session and use the last image of the previous session as an input.

The `-s` flag is read-only: it loads history from the specified session file but never writes back to it. The new session always saves alongside `-o`. This preserves the source session for rewind and branching. To branch from an earlier point, pass that point's session file with `-s` and a new output path with `-o`.

When continuing a session, the tool automatically reuses the session's last used CLI flags. If other arguments are provided, they override the session defaults.

### Subcommands

**meta** — `agentpix meta <image.png>` — Show metadata embedded in a generated PNG.

**clean** — `agentpix clean [-f] <directory>` — Find and remove session files from a directory. Without `-f`, performs a dry run listing files with details. With `-f`, deletes validated session files. Scans non-recursively. Corrupt or unrecognized files are skipped with a warning.

**cost** — `agentpix cost <session-file|directory>` — Estimate API cost from session files using a hard-coded snapshot of published Gemini API pricing. Exact prices may be outdated.

**transform** — `agentpix transform -i <input> -o <output> [-f] <operation> [args]` — Flip, rotate, or resize an image locally without calling the Gemini API. Input and output must both be `.png`. Metadata is preserved. Operations: `flip-h`, `flip-v`, `rotate 90|180|270`, `resize WxH|Wx|xH` (CatmullRom interpolation).

## Prompting Fundamentals

The Gemini image models are language models that generate images as part of their token sequence, not diffusion models. They parse grammar, understand spatial relationships, and reason about composition. Within the family there are two categories: Flash is smaller and quicker, Pro is slow but accurate. Both produce images of comparable visual fidelity, though Pro is better at parsing complicated instructions and getting details right.

The models are biased toward natural and cinematic-looking images. Technical views (orthographic projections, turnaround sheets, precise camera angles) are less reliable, especially for unconventional subjects. Reinforce camera terminology with geometric occlusion constraints: describe what should be hidden and what should be visible rather than relying on terms like "orthographic" alone.

An instructive prompt with clear specifications outperforms overloaded descriptions. Two principles apply to all prompts, creation and transformation alike.

**Positive framing over negation.** Instead of "no cars," write "an empty, deserted street with no signs of traffic." When exclusions are necessary, natural language works: "Do not include any text, watermarks, or line overlays."

**Relational cues, not absolute measurements.** The model reasons about spatial relationships but cannot perform precise measurements. For precise camera angles, reinforce terminology with geometric constraints the model can check: "the far-side limbs are completely hidden behind the near-side limbs," "the top surface of the back is not visible." These give concrete pass/fail criteria.

**Repeat critical requirements at the end of long prompts.** The model weighs earlier content more heavily in lengthy prompts. When a requirement is essential, state it in its field and restate it at the end.

**Use markdown lists for multiple constraints.** The model's text encoder was trained on markdown, so dashed lists improve instruction clarity within fields. ALL CAPS for critical individual requirements ("MUST include exactly three figures"), used sparingly; capitalizing everything dilutes the signal.

**Hex colors for precision.** "#9F2B68" outperforms "amaranth purple" when exact color matters.

Diffusion-model conventions (quality tags, weight syntax, negative prompt blocks) have no effect on Gemini. Stacking conflicting modifiers ("cinematic, volumetric lighting, 35mm, f/1.4, 8k, hyperreal, artstation, unreal engine") is the same failure mode in different clothing. Pick the terms that matter and describe them in context.

## Prompting for Image Creation

Creation prompts describe a scene that does not yet exist. The prompt is the model's only input, so it must be comprehensive.

For simple requests with one or two specifications, a plain sentence is fine: "A tabby cat sleeping on a sunlit windowsill, watercolor style." When a prompt has three or more distinct elements (subject, composition, lighting, style), structure it as a spec sheet with labeled fields. Each field isolates one aspect of the image so the model can parse intent unambiguously. Keep fields lean: translate intent into the correct field without adding adjectives or embellishment. Qualifiers like "breathtaking," "masterfully rendered," "stunningly beautiful" add no visual information; the model does not render "breathtaking" differently from unmodified. Fields can be omitted when they are not relevant.

<example>
Subject: Elderly potter inspecting a cracked raku bowl, turning it in weathered hands.
Composition: Medium close-up, shallow depth of field, hands and bowl in sharp focus.
Setting: Small pottery studio, wooden shelves lined with finished pieces, kiln visible in background.
Lighting: Warm sidelight from a single window, deep shadows on the far side of the face.
Palette: Warm earth tones, terracotta, cream, charcoal.
Style: Photorealistic, shot on Canon EOS R5 with 85mm f/1.4 lens.
Mood: Quiet, contemplative, reverent.
</example>

<example>
Brief: "Moody establishing shot of a canyon at dusk, melancholy atmosphere, stylized realism."

Good prompt:
"Subject: Deep canyon stretching into the distance.
Composition: Wide establishing shot from the canyon rim, looking down and across.
Setting: Arid canyon landscape, layered rock walls.
Lighting: Dusk. Last sunlight on upper canyon walls, shadow on the canyon floor.
Palette: Amber, slate grey, charcoal.
Style: Stylized realism.
Mood: Melancholy, vast, still."

Bad prompt (embellished — every qualifier wastes tokens without changing the output):
"Subject: A breathtaking, impossibly deep canyon stretching majestically into the vast distance.
Composition: A sweeping, cinematic wide establishing shot from the canyon rim, looking dramatically down and across the awe-inspiring expanse.
Setting: A stunningly arid canyon landscape with beautifully layered, ancient rock walls.
Lighting: The warm, golden glow of a dramatic dusk. The last rays of sunlight paint the upper canyon walls in rich amber while the canyon floor is lost in deep, mysterious shadow.
Style: Masterfully rendered stylized realism with expressive, confident brushstrokes.
Mood: Profoundly melancholy, vast, hauntingly still."
</example>

### Field reference

- **Subject:** What is in the image. The primary element, with action or pose if applicable.
- **Composition:** Camera angle, framing, shot type, depth of field. For photorealism, camera and lens specs go here.
- **Setting:** Environment, location, context around the subject.
- **Lighting:** Light sources, direction, quality, time of day. Always specify this field. Without it the model defaults to flat, generic lighting. Lighting is the single strongest lever for mood and realism.
- **Palette:** Dominant colors, color temperature, saturation level.
- **Style:** Rendering approach, medium, artistic technique. Always specify this field. Without it the model picks inconsistently, leading to generic output. Style affects design language, not just rendering: "painterly" and "3D render" produce different scene content, not just different surfaces.
- **Mood:** Emotional tone, atmosphere.
- **Details:** Specific elements that must be present. Use when the request calls out particular items.
- **Constraints:** Requirements that override defaults.

## Prompting for Image Transformation

Transformation prompts direct changes to an existing image, whether through image inputs or session continuation. The image already exists; the prompt specifies the delta, not the whole scene.

Write short, directive instructions focused on what to change. State preservation boundaries alongside each change: what must stay the same. "Change only the sky to a warm sunset gradient. Keep the foreground buildings, street, people, and all shadows exactly as they are." Without explicit preservation instructions, the model may reinterpret unchanged areas.

When adding elements, describe the new element with enough detail to match the existing scene's style and lighting. "Add a black cat on the windowsill, lit consistently with the warm interior light, casting a soft shadow on the sill."

When removing elements, describe how to fill the gap. "Remove the trash can on the left. Fill the area to match the surrounding brick wall texture and concrete sidewalk."

Decompose multi-edit requests into one change per generation. If you need the word "and" to describe what a single call does, it is probably two operations. "Add a lantern and change the jacket color" is two calls. "Add a lantern to the tree branch" is one. The only exception is when two changes are physically entangled (removing an object and filling the gap it leaves).

A reliable test for transformation prompts: if the prompt could generate the image from scratch, you are redescribing, not transforming. A full scene description signals creation; the model may regenerate everything, losing existing details.

## Image Inputs

IMPORTANT: If images are provided, the model will try to use them, so they must always be paired with prompt instructions for how they are to be used. Input images not mentioned by the prompt become noise at best, and misdirection at worst.

Every input image anchors the output. It pulls composition, camera angle, and subject matter toward itself, not just visual style. This anchoring is the most common source of unexpected results when working with reference images. When the reference is a three-quarter view and the target is a side view, the model gravitates toward the reference's angle. The prompt must explicitly counteract the reference's camera with geometric constraints describing what should and should not be visible.

Image input patterns:
- Character consistency by providing images of the character
- Style consistency or transfer by providing reference images
- Placing a character in an environment using both as reference inputs
- Combining features of two or more designs, often by extracting specific features of each
- Object removal, addition, or targeted element change paired with focused instructions

Only pass images as inputs when the output must directly incorporate content from them. An image viewed for understanding context does not need to become a generation input. When a fresh, unbiased image is the goal, do not provide input images; the model anchors to them and reduces creative variance.

## Workflow

Image generation costs money through the Gemini API. Two safeguards require explicit user permission: never use Pro unless the user explicitly requests or approves it, and never generate multiple images in a single turn unless the user explicitly asks for batch generation.

Default to Flash at 1K resolution as the cheapest option. Recommend escalating to Pro when Flash repeatedly fails to produce adequate output even after adjusting the prompt. Pro is especially capable for images that need to contain text elements (advertisements, memes). Similarly, only increase resolution if the user wants to. Explore at lower resolution and scale up once the prompt works.

Choose an output filename that reflects the content and variant. Use the subject as the base name and append what distinguishes this image from its siblings: `farmhouse_twilight.png`, `creature_apose_back.png`. Avoid encoding full edit history into names.

### Creative collaboration

Users often do not know exactly what they want. Generating images to help visualize ideas, and asking focused questions about preferences, are the primary tools for narrowing down direction.

When starting from scratch and the request is too vague to construct a strong prompt, ask focused questions about the missing elements. A vague prompt produces a vague image; the cost of one clarifying question is less than the cost of a wasted generation.

Separate the user's creative intent from your technical execution. The user provides subjects, narrative elements, mood, and constraints. You translate those into visual execution: camera angle, lighting setup, framing, field values. The line between execution and invention is this: texture, surface quality, and environmental grounding that serve the specified setting are execution; distinct objects that add narrative content are invention.

<example>
User request: "the city has seen better days"

Execution (acceptable): weathered stone, faded paint, worn edges, cracked pavement
Invention (overstepping): crumbling bridges, abandoned ships, faded banners, a stray dog

The first group translates the mood into surface and texture. The second group adds specific objects and narrative elements the user did not ask for.
</example>

It is often easier to settle broader concepts first, then distill them into concrete details. This means working through the prompt elements in reverse: mood and atmosphere first, then style, lighting, setting, composition, and finally the specific details of the subject.

### Generation loop

Every generation follows the same cycle: assess, compose, execute, diagnose. The value of deliberate decision-making increases as a session progresses. The first generation is the one where you know the least; by the fifth, you have data about how the model handles this subject.

1. **Assess.** Determine what this generation needs to accomplish. On the first generation, understand the user's goal. On subsequent generations, identify what changed: user feedback, diagnosis findings, concept evolution. Choose the approach: fresh generation, image inputs, session continuation, or a combination. The workflow patterns below have heuristics for when each applies. Before proceeding, check whether the request involves known model limitations (precise text in complex layouts, exact counts of many similar objects, specific spatial relationships between numerous elements). A failed generation that was predictable costs time and money with no information gained. When failure is foreseeable, tell the user and suggest alternatives before generating.
2. **Compose.** Write the prompt. For fresh generation, write a spec sheet following the field reference (Subject, Composition, Setting, Lighting, Palette, Style, Mood, Details, Constraints). For session continuation, write a short directive about what to change. For image inputs, write instructions for how the inputs should be used or transformed. The prompt must include Style and Lighting fields (for creation) or preservation boundaries (for transformation). Check for embellished fields, stacked changes, and invented elements. Every element should trace back to the user's request or to technical execution of that request.
3. **Execute.** Run the agentpix command. Match the aspect ratio to the subject's proportions in the target view, not to convention or habit. A horizontal creature needs a landscape ratio even for a front view. Reconsider the ratio when the subject or composition changes.
4. **Diagnose.** Load the generated image using the Read tool so you can see it. Diagnosis is mandatory on every generation.

LLM image analysis is unreliable for evaluating generation quality. Confirmation bias (seeing what the prompt described rather than what the image contains) is a persistent failure mode that self-awareness does not fix. The user's eyes are the authoritative evaluation tool. Note any obvious observations in a brief sentence or two, then hand diagnosis to the user through targeted questions using AskUserQuestion, each targeting one concrete element. The questions should get sharper as a session progresses, not lazier; later cycles have more data about what the model gets right and wrong with this subject.

When the user reports an issue, ask what specifically is wrong before iterating. A wrong self-diagnosis wastes generations fixing the wrong thing.

For creation prompts, derive questions from the field hierarchy:
- Subject and action: did the subject match? Is the pose or action correct?
- Composition and framing: is the camera angle and framing what was intended?
- Setting and environment: are the grounding details present?
- Lighting: does the lighting match what was specified?
- Style and medium: did the model render in the intended style?
- Novel or complex elements: ask about each individually. These carry the highest risk of being dropped or distorted.

For transformation prompts, ask about both sides of the change:
- Did the requested change land? In the right direction, by the right amount?
- Did preserved elements hold? Ask about specific elements that should not have changed.
- If continuing a session, ask whether elements correct in the previous version have drifted.

Always ask about known weak areas when relevant: faces, hands, fine spatial relationships, thin overlapping geometry.

Use the user's answers to plan the next cycle. Their feedback is the diagnosis. Translate observations into prompt-level causes: if a detail was dropped, check prompt position and field isolation; if the change went the wrong direction, the instruction may have been ambiguous; if elements regressed, the session chain may be too long.

### Workflow patterns

- Use session continuation for targeted edits to an existing image that is close to what the user wants.
- Use image inputs when you need visual references: style sheets, character references, environment references, or a previous output that needs to be recontextualized.
- Use fresh generation when exploring new concepts or when session continuation has failed to fix the same problem twice. Carry forward the prompt language that worked; drop the session history.
- If Flash fails to capture details across multiple attempts (including fresh starts), recommend escalating to Pro. "Fails to capture" includes missing, distorted, or reinterpreted details. Do not compensate with emphatic prompt language; if the model misrenders a clearly stated detail, restating it louder will not help.
- If fine details will not resolve at 1K, recommend higher resolution as an alternative or complement to a model switch.
- If exploration keeps producing similar outputs despite different prompts, check whether input images are anchoring the model.
- If a session chain is drifting after 3-4 turns, start fresh with a revised prompt rather than continuing to patch.
- For technical views (orthographic, turnaround sheets), start with Pro if precise camera control matters.
- If the same approach has failed twice, the third attempt should be structurally different: a new angle, a different style framing, a simplified subject, or higher resolution. Minor tweaks to a failing approach waste generations.
- Use `-t high` when prompt adherence is the problem. It makes Flash 3.1 reason more deeply about complex instructions at the cost of higher latency and token usage.

