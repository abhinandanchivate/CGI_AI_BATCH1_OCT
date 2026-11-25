
---

## Part 1: How to run it

### Step 1: Unzip and open a terminal in the folder
```bash
cd adk_demo
```
You should see `check_setup.py`, `README.md` and a `hello_agent/` folder.

### Step 2: Create a virtual environment
```bash
python -m venv .venv
# macOS / Linux
source .venv/bin/activate
# Windows (PowerShell)
.venv\Scripts\Activate.ps1
```
Your prompt should now start with `(.venv)`.

### Step 3: Install ADK
```bash
pip install google-adk
adk --version
```

### Step 4: Add your API key
1. Get a free key at **aistudio.google.com/apikey**.
2. Copy the sample settings file:
   ```bash
   cp hello_agent/.env.example hello_agent/.env      # Windows: copy hello_agent\.env.example hello_agent\.env
   ```
3. Open `hello_agent/.env` and replace `paste-your-key-here` with your key.

### Step 5: Run the check
```bash
python check_setup.py
```
When everything works, you'll see:
```
[PASS] Python 3.10+
[PASS] google-adk installed
[PASS] Credentials (Gemini API key)
[PASS] Agent definition loads
[PASS] Tools run locally
[PASS] Model reachable and responded
[PASS] Agent called its tools
ALL GOOD: ADK is set up correctly.
```

### Step 6: Chat with the agent in the terminal
```bash
adk run hello_agent
```
Ask: *What's the weather in London and what time is it there?* Type `exit` to quit.

### Step 7: Open the browser test page
```bash
adk web
```
Open **http://localhost:8000**, pick **hello_agent** from the dropdown, and chat. Click any reply to see which tool it called and what came back. Press `Ctrl+C` to stop.

### If something fails

| What you see | How to fix it |
|---|---|
| `google-adk installed` FAIL | Activate the venv, then run `pip install google-adk` |
| `Credentials` FAIL | Check that `hello_agent/.env` exists and the key is pasted in |
| `API key rejected` | Make a new key in AI Studio |
| `model not found` | Add `ADK_MODEL=gemini-2.5-flash` (or a model your account has) to `.env` |
| `quota exceeded` | Wait a minute, or check billing |
| `adk web` shows no agents | Run it from the `adk_demo` folder, not from inside `hello_agent` |
| `'adk' is not recognized` | The venv isn't active; repeat Step 2 |

---

## Part 2: Activity — add a currency-converter tool (15–20 minutes)

**Goal:** teach the agent a new skill by writing a Python function with three inputs, and see the model choose the right tool by itself.

### Task 1: Add the tool
Open `hello_agent/agent.py` and paste this **above** `root_agent = Agent(...)`:

```python
_RATES_TO_INR = {"INR": 1.0, "USD": 88.0, "EUR": 102.0, "GBP": 117.0}  # demo rates only


def convert_currency(amount: float, from_currency: str, to_currency: str) -> dict:
    """Converts money between INR, USD, EUR and GBP using fixed demo rates.

    Args:
        amount: The amount to convert, e.g. 100.
        from_currency: Three-letter code to convert from, e.g. "USD".
        to_currency: Three-letter code to convert to, e.g. "INR".
    """
    src, dst = from_currency.upper(), to_currency.upper()
    if src not in _RATES_TO_INR or dst not in _RATES_TO_INR:
        return {"status": "error", "error_message": f"Supported currencies: {', '.join(_RATES_TO_INR)}."}
    result = amount * _RATES_TO_INR[src] / _RATES_TO_INR[dst]
    return {"status": "success", "report": f"{amount:g} {src} = {result:,.2f} {dst} (demo rate)."}
```

### Task 2: Give the tool to the agent
In the same file, update `root_agent`:

```python
root_agent = Agent(
    name="hello_agent",
    model=MODEL,
    description="Answers weather, time and currency questions for a few cities.",
    instruction=(
        "You help users with the weather, local time and currency conversion. "
        "Always use the get_weather, get_current_time and convert_currency tools; never guess. "
        "If a tool returns an error, tell the user politely."
    ),
    tools=[get_weather, get_current_time, convert_currency],
)
```

### Task 3: Check it, then test it
```bash
python check_setup.py --offline     # "Agent definition loads" should now list 3 tools
adk web
```
Try these prompts in the browser page:

| Prompt | What should happen |
|---|---|
| Convert 100 USD to INR | Calls `convert_currency` once → 8,800.00 INR |
| I'm going to London with 5000 rupees. How many pounds is that, and what's the weather? | Calls **two** tools in one answer |
| Convert 50 JPY to INR | The tool returns an error and the agent politely lists the supported currencies |
| What's the weather in Mumbai? | Mumbai isn't set up yet, so the agent says it has no data (it should not guess) |

### Task 4: Stretch goals (optional)
1. **Add Mumbai:** add `"mumbai": ("Asia/Kolkata", "humid, 31 °C")` to `_CITIES` and ask the Mumbai question again.
2. **Make your own tool:** for example, `get_packing_tip(city)`, which suggests an umbrella when the weather says rain. Remember the docstring: it's what the model reads to decide when to use the tool.
3. **Look at the steps:** in `adk web`, open the steps view for the London prompt and note the order of the tool calls and what each returned.

### Done when
- `check_setup.py` still ends with **ALL GOOD**
- The London prompt uses both `convert_currency` and `get_weather`
- The JPY prompt gets a polite error, not a made-up number

**Takeaway:** in ADK, a tool is just a Python function. The model learns from the function's name, its typed inputs and its docstring when to call it and what to pass in, so clear docstrings lead to correct tool calls.
