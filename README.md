# MCP Recipe Research Server

A Model Context Protocol (MCP) server that connects to [TheMealDB](https://www.themealdb.com) API to search, retrieve, and organize recipe data. Built with [FastMCP](https://github.com/jlowin/fastmcp) and designed to integrate seamlessly with MCP-compatible AI clients such as Claude Desktop.

---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Usage](#usage)
- [Tools Reference](#tools-reference)
- [Resources Reference](#resources-reference)
- [Prompts Reference](#prompts-reference)
- [Data Storage](#data-storage)
- [API Reference](#api-reference)

---

## Overview

This MCP server acts as a bridge between an AI assistant and TheMealDB recipe database. When connected to an MCP-compatible client, it exposes a set of **tools**, **resources**, and **prompt templates** that the assistant can use to search for recipes, retrieve detailed cooking instructions, build meal plans, and explore cuisine data — all fetched live from the TheMealDB API and cached locally as JSON files.

---

## Features

- **Recipe search** by dish name or by starting letter
- **Detailed recipe lookup** including ingredients, instructions, and media links
- **Meal plan creation** from a curated selection of recipe IDs
- **Random recipe discovery**
- **Local JSON caching** of all fetched recipes organized by dish/cuisine
- **Collection statistics** and resource browsing via MCP resources
- **Pre-built prompt templates** for cuisine exploration, meal planning, cooking lessons, and ingredient deep-dives
- Cross-platform filename sanitization (Windows, macOS, Linux)

---

## Project Structure

```
mcp-recipe-final/
├── recipe_server.py       # Main MCP server — all tools, resources, and prompts
├── pyproject.toml         # Project metadata and dependencies
├── runtime.txt            # Python version pin
├── .python-version        # Python version for uv
├── .venv/                 # Virtual environment (created locally, not committed)
└── recipes/               # Auto-created at runtime; stores cached recipe JSON
    ├── <dish_name>/
    │   └── recipes_info.json
    ├── by_letter/
    │   └── letter_<x>_search.json
    └── meal_plans/
        └── <plan_name>.json
```

> **Note:** The `recipes/` directory is generated automatically the first time the server runs. It should be added to `.gitignore`.

---

## Prerequisites

- **Python** ≥ 3.14 (see `runtime.txt`)
- **[uv](https://github.com/astral-sh/uv)** — fast Python package and project manager

---

## Installation

These steps mirror exactly how the project was set up originally.

**1. Clone the repository**

```bash
git clone https://github.com/<your-username>/mcp-recipe-final.git
cd mcp-recipe-final
```

**2. Initialize the virtual environment**

```bash
uv venv
```

This creates a `.venv/` folder in the project root.

**3. Activate the virtual environment**

On macOS / Linux:
```bash
source .venv/bin/activate
```

On Windows (PowerShell):
```powershell
.venv\Scripts\Activate.ps1
```

**4. Install dependencies**

```bash
uv add "mcp[cli]"
uv add requests
```

These are the two core packages the server depends on:

| Package | Purpose |
|---|---|
| `mcp[cli]` | FastMCP server framework and CLI runner |
| `requests` | HTTP calls to TheMealDB API |

---

## Configuration

No API key is required. TheMealDB's free public API (v1) is used throughout:

```
https://www.themealdb.com/api/json/v1/1/
```

The server listens on port `8000` by default. This can be overridden with the `PORT` environment variable:

```bash
PORT=9000 python recipe_server.py
```

For deployment with HTTP transport, uncomment the relevant lines at the bottom of `recipe_server.py`:

```python
# mcp = FastMCP("recipe_research", host="0.0.0.0", port=port)
# mcp.run(transport="streamable-http")
```

---

## Usage

**Run the server locally (stdio transport — default for Claude Desktop):**

```bash
python recipe_server.py
```

**Run via the MCP CLI:**

```bash
mcp run recipe_server.py
```

**Connect to Claude Desktop** by adding the following to your `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "recipe_research": {
      "command": "uv",
      "args": [
        "--directory",
        "/absolute/path/to/mcp-recipe-final",
        "run",
        "recipe_server.py"
      ]
    }
  }
}
```

Once connected, the tools, resources, and prompts described below will be available inside the Claude Desktop conversation.

---

## Tools Reference

### `search_recipes(dish_name, max_results=5)`

Searches TheMealDB for recipes matching a dish name. Fetches full recipe details (ingredients, instructions, media links) and caches the results locally under `recipes/<dish_name>/recipes_info.json`.

| Parameter | Type | Description |
|---|---|---|
| `dish_name` | `str` | The dish or food name to search for (e.g. `"pasta"`, `"chicken curry"`) |
| `max_results` | `int` | Maximum number of results to return (default: 5) |

**Returns:** A list of recipe ID strings.

---

### `get_recipe_details(recipe_id)`

Looks up detailed information for a specific recipe ID from the local cache (populated by a prior `search_recipes` call).

| Parameter | Type | Description |
|---|---|---|
| `recipe_id` | `str` | The numeric recipe ID (e.g. `"52771"`) |

**Returns:** A JSON string containing the recipe's name, cuisine, category, ingredients, instructions, image URL, YouTube link, and tags.

---

### `create_meal_plan(recipe_ids, plan_name="My Meal Plan")`

Assembles a meal plan from a list of previously searched recipe IDs and saves it as a JSON file under `recipes/meal_plans/`.

| Parameter | Type | Description |
|---|---|---|
| `recipe_ids` | `List[str]` | List of recipe ID strings to include in the plan |
| `plan_name` | `str` | Display name for the meal plan (default: `"My Meal Plan"`) |

**Returns:** A success message with the save path, or an error message.

---

### `search_by_first_letter(letter, max_results=5)`

Retrieves recipes whose names begin with a specific letter, using TheMealDB's letter-search endpoint. Results are cached under `recipes/by_letter/`.

| Parameter | Type | Description |
|---|---|---|
| `letter` | `str` | A single letter A–Z |
| `max_results` | `int` | Maximum number of results to return (default: 5) |

**Returns:** A list of recipe ID strings.

---

### `get_random_recipe()`

Fetches a single random recipe from TheMealDB. Does not cache the result locally.

**Returns:** A JSON string with the recipe's full details.

---

### `test_filesystem()`

Diagnostic tool that verifies the server can read from and write to the `recipes/` directory. Useful for troubleshooting permission issues.

**Returns:** A plain-text status report.

---

### `get_system_info()`

Returns runtime environment information including the Python version, operating system, current working directory, and the resolved path to the `recipes/` directory.

**Returns:** A plain-text system summary.

---

## Resources Reference

MCP resources are read-only data endpoints that an AI client can browse without invoking a tool call.

### `recipes://collections`

Lists all recipe collections currently cached locally (one collection per dish search term).

### `recipes://{cuisine}`

Returns detailed markdown-formatted content for a specific collection folder. Replace `{cuisine}` with the folder name shown in the collections list (e.g. `recipes://pasta`, `recipes://chicken_curry`).

### `recipes://meal-plans`

Lists all saved meal plans with their names, recipe counts, and creation dates.

### `recipes://stats`

Provides aggregate statistics across the entire local recipe cache: total recipe count, number of collections, top cuisines, recipe categories, and meal plan count.

---

## Prompts Reference

Pre-built prompt templates that guide an AI assistant through structured recipe-related tasks.

### `generate_recipe_search_prompt(cuisine_type, num_recipes=5)`

Instructs the assistant to search for recipes from a given cuisine, extract structured data for each result, and provide a cultural and culinary overview.

### `generate_meal_planning_prompt(meal_type, people_count=4, dietary_restrictions="none")`

Instructs the assistant to build a complete meal plan including a shopping list, prep timeline, nutritional balance notes, and cost estimates.

### `generate_cooking_lesson_prompt(skill_level, technique_focus, cuisine_style="any")`

Instructs the assistant to design a structured cooking lesson covering technique theory, step-by-step demonstration recipes, common mistakes, and skill-building exercises.

### `generate_ingredient_exploration_prompt(main_ingredient, cooking_styles="diverse", num_recipes=6)`

Instructs the assistant to explore a single ingredient across multiple recipes, covering its culinary profile, nutritional value, preparation techniques, and global uses.

### `generate_cultural_cuisine_prompt(cuisine_name, cultural_context="traditional", num_recipes=5)`

Instructs the assistant to research the cultural history, traditions, and culinary philosophy of a specific cuisine through representative recipes.

---

## Data Storage

All fetched data is saved automatically as UTF-8 encoded JSON files within the `recipes/` directory created at the project root. The directory layout is:

```
recipes/
├── arrabiata/
│   └── recipes_info.json        # Keyed by recipe ID
├── chicken_curry/
│   └── recipes_info.json
├── by_letter/
│   ├── letter_a_search.json
│   └── letter_b_search.json
└── meal_plans/
    └── italian_week.json
```

Folder and file names are automatically sanitized to be valid across Windows, macOS, and Linux (spaces replaced with underscores, special characters removed, names capped at 200 characters).

> Add `recipes/` to your `.gitignore` to avoid committing cached data to the repository.

---

## API Reference

This project uses the **TheMealDB free public API (v1)**. No authentication is required.

| Endpoint | Used by |
|---|---|
| `/search.php?s={name}` | `search_recipes` |
| `/search.php?f={letter}` | `search_by_first_letter` |
| `/lookup.php?i={id}` | (direct lookup fallback) |
| `/random.php` | `get_random_recipe` |

Full API documentation: [https://www.themealdb.com/api.php](https://www.themealdb.com/api.php)

---

## License

This project is open source. See `LICENSE` for details.
