# SketchBot

SketchBot is a drawing game where the player receives a prompt, tries to draw it, and AI players guess what it is.

SketchBot combines pictionary with the game of telephone: one player draws the original prompt, the next player guesses what it is, the next player (who hasn't seen the original prompt or drawing) takes the guess as a new prompt and draws it, and so on. The game can be scored based on how similar the final guess is to the original prompt.

## AI SketchBot

This is a game of SketchBot played with AIs!

### AI Participants

The following AIs are involved:
1. An **announcer LLM** that narrates the gameplay.
2. A **Kokoro text-to-speech model** that speaks the narration in real time.
3. **N AI players**, each playing exactly one role per turn. Each AI player is either:
    - **Guessing player**: A vision LLM that looks at the most recent image and guesses what it is (with the shortest phrase possible), or
    - **Drawing player**: Consists of two models working together:
        - A **prompt enhancement LLM** (PromptBridge) that transforms the guesser's phrase (e.g., "big bird") into a suitable Stable Diffusion prompt with tags for making a crude hand drawing.
        - A **Stable Diffusion model** that creates the image from that prompt.
4. A **reranking model** that compares each guess to the original prompt and produces a similarity score.

### Player Counts

Players come in a specific structure: 1 initial guesser, then additional drawer-guesser pairs.

- **1 AI player**: Human draws → VLM1 guesses. (1 guesser)
- **3 AI players**: Human draws → VLM1 guesses → SD1 draws → VLM2 guesses. (2 guessers, 1 drawer)
- **5 AI players**: Human draws → VLM1 guesses → SD1 draws → VLM2 guesses → SD2 draws → VLM3 guesses. (3 guessers, 2 drawers)

Each AI player is a unique entity — guessers are distinct VLMs, drawers are distinct SD models. No model is reused across player roles.

### Models

Each AI participant uses a unique model. They should all be relatively small so that the game flows well. Models are assigned in the order listed below — no user selection needed.

**Player models (assigned in order based on player count):**
1. Guessing VLMs: Qwen3-4B-VL-FLM, Gemma-3-4b-it-GGUF, Qwen2.5-VL-3B-Instruct-GGUF
2. Drawing SDs: SD-Turbo, SDXL-Turbo, SD-1.5

**Utility models (always active):**
1. Announcer LLM: LFM2.5-1.2B-Instruct-GGUF
2. Prompt enhancement LLM: PromptBridge-0.6b-Alpha-GGUF (shared across all drawing turns)
3. Reranking: bge-reranker-v2-m3-GGUF
4. Speech: kokoro-v1

**FLM fallback:** The app checks the `/api/v1/system-info` endpoint to see if the FLM recipe is supported. If FLM is not available (no NPU), Qwen3-4B-VL-FLM is automatically replaced with Qwen3-VL-4B-Instruct-GGUF.

### Gameplay

The gameplay should flow like this:

1. The announcer LLM picks a prompt that would be funny to hand draw, like "big bird"
2. The human player draws the prompt on a canvas
3. Then the AI players take turns in sequence — each guesser sees only the most recent image (not the original prompt), and each drawer receives only the most recent guess
4. Each guess is ranked against the original prompt using the reranking model
5. The final score gets announced with a flashy banner that celebrates a strong final guess or gently roasts a weak final guess
6. The narrator + speech model make short, quippy commentary about the game at each turn

## Implementation

### Tech Stack

The game itself is a webui HTML/CSS/JS project in `examples/sketchbot/`. It relies on Lemonade Server for all AI capabilities.

### Setup UI

There should be a setup screen that:
- Detects if Lemonade Server is running and helps the user install and start it if not.
- Checks `/api/v1/system-info` to determine FLM availability (for VLM fallback).
- Checks `/api/v1/health` to verify the server has enough model slots (`max_models`) for the required models. If not, provides clear advice on how to restart the server with sufficient slots (e.g., `lemonade-server serve --max-loaded-models N`).
- Ensures all required models are downloaded, showing nice download progress indicators via the streaming `/api/v1/pull` endpoint.
- Loads all required models via `/api/v1/load` so they are ready before gameplay begins.

### Main UI

It is essential that the UI fits nicely on a mobile phone screen, but expands ok to a desktop browser.

The UI also needs to indicate which AI model(s) are active at any given time, in a clean and minimal way.

There should be a prompt decision screen where the user is offered a prompt, but can "re-roll" the prompt to something else. This screen will also show the connectivity between all the AI models in the app. This screen also lets the player choose between 1, 3, and 5 AI players (1 AI guesser is requested, and then they come in drawer-guesser pairs).

The final score screen should also show the connectivity of the models.
