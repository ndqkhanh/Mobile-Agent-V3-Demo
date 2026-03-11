# Mobile-Agent-v3 Demo

A demo built on top of [**Mobile-Agent-v3**](https://github.com/X-PLUG/MobileAgent) by Tongyi Lab, Alibaba Group (X-PLUG). The original framework provides the multi-agent architecture for autonomous GUI automation; this repository adds a standalone demo runner with Excel test reporting and structured JSON logging.

Uses the **Manager → Executor → Reflector** loop to plan and execute actions on a real Android phone over ADB, with support for **GUI-Owl** (self-hosted vLLM) and **GPT-4o** (OpenAI API) as the underlying vision-language model.

> **Credit:** All core agent designs, prompt templates, and the GUI-Owl model originate from the [X-PLUG/MobileAgent](https://github.com/X-PLUG/MobileAgent) project. This repo is a personal demo and is not affiliated with or endorsed by the original authors.

## Project Structure

```
Mobile-Agent-v3/
├── LICENSE
├── Readme.md
└── mobile_v3/
    ├── run_demo.py                    # Main entry point & Excel report generation
    └── utils/
        ├── controller.py              # Abstract Controller base class (ABC)
        ├── android_controller.py      # ADB-based Android controller implementation
        ├── mobile_agent_e.py          # Agent definitions & prompt templates
        │                              #   InfoPool, Manager, Executor, Notetaker, ActionReflector
        ├── call_mobile_agent_e.py     # LLM wrappers (GUIOwlWrapper, GPT4Wrapper)
        └── new_json_action.py         # Action type constants (click, swipe, type, …)
```

## Prerequisites

### Python Dependencies

```bash
pip install Pillow openpyxl openai numpy qwen-vl-utils
```

### ADB Setup

1. Download the [Android Debug Bridge (ADB)](https://developer.android.com/tools/releases/platform-tools?hl=en).
2. Enable **USB Debugging** in your Android phone's Developer Options. On HyperOS, also enable **USB Debugging (Security Settings)**.
3. Connect your phone via USB and select "Transfer files".
4. Verify the connection:
   ```bash
   /path/to/adb devices
   ```
5. On macOS / Linux, ensure ADB is executable:
   ```bash
   sudo chmod +x /path/to/adb
   ```

### ADB Keyboard

The framework types text character-by-character via ADB input events and falls back to `ADBKeyBoard` broadcasts for special characters. You must install the ADB Keyboard IME:

1. Download the [ADB Keyboard APK](https://github.com/senzhk/ADBKeyBoard/blob/master/ADBKeyboard.apk).
2. Install it on your device.
3. Set the default input method to **ADB Keyboard** in system settings.

When ADB Keyboard is active, you will see `ADB Keyboard {on}` at the bottom of the screen — the Executor agent uses this as a signal that a text field is ready for input.

## Usage

```bash
cd Mobile-Agent-v3/mobile_v3

python run_demo.py \
    --adb_path "/path/to/adb" \
    --api_key "your-api-key" \
    --base_url "https://your-vllm-endpoint/v1" \
    --model "mPLUG/GUI-Owl-7B" \
    --instruction "Open Settings and turn on Wi-Fi" \
    --add_info ""
```

### Example: YouTube Video Search with GUI-Owl

```bash
python -u run_demo.py \
  --adb_path /opt/homebrew/bin/adb \
  --api_key "your-api-key" \
  --base_url "https://your-ngrok-endpoint.ngrok-free.dev/v1" \
  --model "GUI-Owl-7B" \
  --instruction "Add a new contact with name John Doe and phone 1234567890" \
  --add_info "If the Contacts app is not on the home screen, open the app drawer and search for 'Contacts'."
```

### Arguments

| Argument | Required | Default | Description |
|---|---|---|---|
| `--adb_path` | Yes | — | Path to the `adb` binary |
| `--api_key` | No | `$OPENAI_API_KEY` | API key for the LLM service |
| `--base_url` | No | Auto-detected | Base URL of the LLM API endpoint |
| `--model` | No | `mPLUG/GUI-Owl-7B` | Model name. Names containing `gui-owl` / `gui_owl` use `GUIOwlWrapper`; all others use `GPT4Wrapper` |
| `--instruction` | Yes | — | Natural language task for the agent to complete |
| `--add_info` | No | `""` | Supplementary knowledge to guide planning |
| `--max_step` | No | `25` | Maximum number of agent steps |
| `--coor_type` | No | `abs` | `abs` for absolute pixel coordinates, `qwen-vl` for 0–1000 range |
| `--excel_path` | No | Auto-generated | Custom path for the Excel test report |

### Model Selection

- **GUI-Owl** (e.g. `mPLUG/GUI-Owl-7B`): Uses `GUIOwlWrapper` with `smart_resize` preprocessing (via `qwen-vl-utils`). Requires a vLLM-compatible endpoint.
- **GPT-4o** (e.g. `gpt-4o`): Uses `GPT4Wrapper` with standard base64 encoding (no resize). Uses the OpenAI API.

If `--base_url` is not set, it is auto-detected: GUI-Owl models default to a Cloudflare tunnel URL, while all others default to `https://api.openai.com/v1`.

## Output

Each run creates a timestamped directory under `./logs/` containing:

- `images/` — screenshots captured before and after each action
- `step_N/` — JSON logs for each step:
  - `manager.json` — Manager prompt, raw response, and parsed plan
  - `operator.json` — Executor prompt, raw response, and chosen action
  - `reflector.json` — Reflector prompt, raw response, and outcome assessment
- `task_result.json` — final task status with completion timestamp
- `test_report.xlsx` — Excel report with:
  - Embedded before/after screenshot thumbnails
  - Color-coded outcome cells (green = success, red = failure, yellow = pending)
  - Summary row with final status and total step count

## Architecture

The agent loop runs up to four roles per step:

1. **Manager** — Observes the current screenshot and creates or updates a high-level plan with subgoals. Tracks completed subgoals across steps. Receives error escalation when the Executor fails repeatedly (configurable via `err_to_manager_thresh`, default 2 consecutive failures).

2. **Executor** — Selects the next atomic action based on the current plan, screenshot, and action history. Available actions:
   - `click(coordinate)` — tap a point on screen
   - `long_press(coordinate)` — long-press a point
   - `swipe(coordinate, coordinate2)` — scroll/swipe between two points
   - `type(text)` — type into the active input field
   - `system_button(button)` — press Back or Home
   - `answer(text)` — return a textual answer to the user

3. **Action Reflector** — Compares before/after screenshots to evaluate whether the action:
   - **A** — Succeeded (or partially succeeded)
   - **B** — Failed, navigated to a wrong page
   - **C** — Failed, produced no visible change

4. **Notetaker** *(defined but not invoked in the current demo loop)* — Records important on-screen information relevant to the task across steps.

The loop continues until the Manager marks the plan as "Finished", the Executor issues an `answer` action, or `--max_step` is reached.

### Coordinate Systems

- **`abs`** (default): The model outputs absolute pixel coordinates matching the phone's screen resolution.
- **`qwen-vl`**: The model outputs coordinates in the 0–1000 range, which are scaled to screen resolution at execution time.

### Error Recovery

When the last `err_to_manager_thresh` actions all receive outcome B or C, the Manager is flagged as "Potentially Stuck" and receives the recent failure log. This lets it revise the plan rather than repeating failed actions. The Executor is also prompted to avoid repeating previously failed actions.

## Acknowledgements

This demo is based on the [Mobile-Agent](https://github.com/X-PLUG/MobileAgent) project by Tongyi Lab, Alibaba Group. The original project is licensed under [MIT](https://github.com/X-PLUG/MobileAgent/blob/main/LICENSE).

## Citation

```bibtex
@article{ye2025mobile,
  title={Mobile-Agent-v3: Foundamental Agents for GUI Automation},
  author={Ye, Jiabo and Zhang, Xi and Xu, Haiyang and Liu, Haowei and Wang, Junyang and Zhu, Zhaoqing and Zheng, Ziwei and Gao, Feiyu and Cao, Junjie and Lu, Zhengxi and others},
  journal={arXiv preprint arXiv:2508.15144},
  year={2025}
}
```
