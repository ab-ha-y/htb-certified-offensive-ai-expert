# Environment Setup

> HTB Certified Offensive AI Expert -- Study Guide
> Module: Applications of AI in InfoSec | Section: Environment Setup

---

## Table of Contents

1. [Why Environment Setup Matters](#1-why-environment-setup-matters)
2. [What is Miniconda?](#2-what-is-miniconda)
3. [Installing Miniconda](#3-installing-miniconda)
4. [Conda Environments Explained](#4-conda-environments-explained)
5. [What is JupyterLab?](#5-what-is-jupyterlab)
6. [Installing and Launching JupyterLab](#6-installing-and-launching-jupyterlab)
7. [Managing Python Dependencies](#7-managing-python-dependencies)
8. [A Full COAE Lab Setup Walkthrough](#8-a-full-coae-lab-setup-walkthrough)
9. [Common Pitfalls](#9-common-pitfalls)
10. [Key Takeaways](#10-key-takeaways)

---

## 1. Why Environment Setup Matters

Before you can classify a single spam email or detect a single malicious packet, you need a working **Python environment** -- a self-contained space on your machine where a specific version of Python and a specific set of libraries live together, isolated from everything else on the system.

### The Analogy

Think of your laptop as a shared kitchen. If everyone who wants to cook throws their ingredients into the same pantry, you eventually end up with expired milk mixed in with fresh milk, someone's fish sauce ruining someone else's dessert, and nobody can tell which jar of flour belongs to which recipe. That is what happens when every Python project on your machine shares one global set of installed packages: Project A needs `scikit-learn 1.0`, Project B needs `scikit-learn 1.4`, and suddenly nothing works for either of them.

A **virtual environment** is your own private kitchen for each recipe (project). You bring in exactly the ingredients (packages) that recipe needs, at exactly the versions it needs, and nothing you do in this kitchen affects any other kitchen in the building.

### Why This Matters Specifically for Offensive AI Work

- Security research often means juggling multiple projects at once: a spam classifier here, a malware-image CNN there, an adversarial-example generator somewhere else -- each with different, sometimes conflicting, library version requirements (e.g., one project pinned to an older PyTorch build for a specific research paper's reproducibility).
- Reproducibility matters for the exam and for real research: if you cannot recreate the exact environment that produced a result, you cannot debug it or defend it.
- Malware analysis environments in particular benefit from isolation -- you do not want a dependency mix-up in your analysis tooling to silently break in the middle of a lab exercise.

---

## 2. What is Miniconda?

**Miniconda** is a small, free installer for two things bundled together:

1. **conda** -- a package manager and environment manager (like a librarian who fetches books and also manages separate reading rooms).
2. **Python** itself -- a minimal Python installation to start from.

It is the lightweight sibling of **Anaconda**, which is the same idea but ships with hundreds of data-science packages pre-installed (multiple gigabytes). Miniconda gives you just the manager and a bare Python, and you install only what you actually need.

### conda vs. pip -- What is the Difference?

| Tool | What It Manages | Handles Non-Python Dependencies? | Environment Isolation? |
|------|-----------------|-----------------------------------|-------------------------|
| **pip** | Python packages only | No (assumes system libraries like CUDA, BLAS already exist) | No, on its own (needs `venv` alongside it) |
| **conda** | Python packages AND system-level dependencies (compilers, CUDA toolkits, C libraries) | Yes | Yes, built in |

**Plain English**: `pip` is great at installing pure-Python libraries, but many ML libraries (like PyTorch with GPU support) depend on lower-level, non-Python components (like NVIDIA's CUDA drivers). `conda` can install and manage those too, which is why it is the standard starting point for ML environments. In practice, most people use `conda` to create the environment and Python version, then use `pip` *inside* that conda environment to install the specific Python packages -- the two tools work together, not against each other.

```
                     YOUR MACHINE
    +-----------------------------------------------------+
    |                                                      |
    |   +-------------------+     +-------------------+    |
    |   |  conda env:       |     |  conda env:       |    |
    |   |  "spam-project"   |     |  "malware-project"|    |
    |   |                   |     |                   |    |
    |   |  Python 3.10      |     |  Python 3.11      |    |
    |   |  scikit-learn 1.4 |     |  torch 2.3        |    |
    |   |  pandas 2.2       |     |  torchvision 0.18 |    |
    |   +-------------------+     +-------------------+    |
    |                                                      |
    |   Completely isolated from each other and from       |
    |   whatever Python your operating system uses.        |
    +-----------------------------------------------------+
```

---

## 3. Installing Miniconda

### macOS / Linux

```bash
# Download the installer (macOS example, Apple Silicon)
curl -O https://repo.anaconda.com/miniconda/Miniconda3-latest-MacOSX-arm64.sh

# Run the installer
bash Miniconda3-latest-MacOSX-arm64.sh

# Reload your shell configuration so the "conda" command is available
source ~/.zshrc      # or ~/.bashrc, depending on your shell
```

### Windows

Download the `.exe` installer from the official Miniconda page and run it through the graphical installer. It adds an "Anaconda Prompt" to your Start menu, which is a terminal pre-configured to use `conda`.

### Verifying the Install

```bash
conda --version
# conda 24.x.x

conda info
# Shows base environment path, active environment, channels, platform, etc.
```

---

## 4. Conda Environments Explained

An **environment** is a named, isolated folder containing a specific Python interpreter and a specific set of installed packages. You can have as many as you like, and you switch between them with `activate`/`deactivate`.

### Core Commands

| Command | What It Does |
|---------|--------------|
| `conda create -n <name> python=3.11` | Create a new environment named `<name>` with Python 3.11 |
| `conda activate <name>` | Switch into that environment (your shell prompt will show `(name)`) |
| `conda deactivate` | Leave the current environment, returning to `base` |
| `conda env list` | List all environments on this machine |
| `conda list` | List all packages installed in the *active* environment |
| `conda install <package>` | Install a package into the active environment |
| `conda remove -n <name> --all` | Delete an environment entirely |
| `conda env export > environment.yml` | Save the exact environment definition to a file |
| `conda env create -f environment.yml` | Recreate an environment from that file (reproducibility!) |

### Worked Example: Creating a COAE Environment

```bash
# Create an environment dedicated to this course, with Python 3.11
conda create -n coae python=3.11 -y

# Activate it
conda activate coae

# Your prompt now looks like:
# (coae) user@machine ~ %

# Install the core data science + ML stack
conda install -y numpy pandas matplotlib scikit-learn jupyterlab

# Install PyTorch (CPU-only example; GPU builds use a different index)
pip install torch torchvision torchaudio

# Confirm what is installed
conda list | grep -E "pandas|scikit-learn|torch"
```

```
    ENVIRONMENT LIFECYCLE
    ======================

    conda create -n coae python=3.11
              |
              v
    +-------------------+
    |   coae (empty)    |
    +-------------------+
              |
              v   conda install / pip install
    +-------------------+
    |   coae            |
    |   + numpy         |
    |   + pandas        |
    |   + scikit-learn  |
    |   + jupyterlab    |
    |   + torch         |
    +-------------------+
              |
              v   conda activate coae
    +-------------------+
    |   ACTIVE: coae    |  <-- all installs/commands now happen here
    +-------------------+
```

---

## 5. What is JupyterLab?

**JupyterLab** is a web-based interactive development environment for writing and running code in small, ordered chunks called **cells**, alongside rendered text, images, and plots -- all in one document called a **notebook** (file extension `.ipynb`).

### The Analogy

A regular Python script is like writing a letter: you write the whole thing, then run it top to bottom, and you only see the final result. A Jupyter notebook is like a lab notebook a scientist keeps at the bench: you run one small experiment (a cell), look at the result immediately (a printed table, a chart, an error message), then decide what to try next -- without re-running everything from scratch each time.

This matters enormously for ML work, where you constantly want to peek at intermediate results: "What does this dataframe look like after I dropped those columns? What does this histogram look like before vs. after I fixed the skew?"

### Why It Is the Standard Tool for This Kind of Work

- **Immediate feedback**: run one cell, inspect the output, adjust, re-run -- much faster than a script/print/rerun cycle.
- **Inline visualizations**: charts render directly below the code that produced them.
- **Mixed content**: markdown text cells let you document your reasoning right next to the code, which is exactly the "explain as you go" workflow used throughout security research and in this course.
- **Shareable artifacts**: a `.ipynb` file preserves code, output, and commentary together -- useful for writeups and for grading/lab submissions.

```
    ANATOMY OF A JUPYTER NOTEBOOK
    ==============================

    +--------------------------------------------------+
    | [Markdown Cell]                                  |
    | # Step 1: Load the dataset                       |
    +--------------------------------------------------+
    | [Code Cell]                     [1]:              |
    | import pandas as pd                               |
    | df = pd.read_csv("emails.csv")                     |
    | df.head()                                          |
    +--------------------------------------------------+
    | [Output]                                           |
    |    text            label                           |
    | 0  "free money"    spam                             |
    | 1  "meeting at 3"  ham                              |
    +--------------------------------------------------+
    | [Markdown Cell]                                    |
    | # Step 2: Check for missing values                 |
    +--------------------------------------------------+
    | [Code Cell]                     [2]:               |
    | df.isnull().sum()                                  |
    +--------------------------------------------------+
```

---

## 6. Installing and Launching JupyterLab

If you followed the environment creation steps above, JupyterLab is already installed inside your `coae` environment. To launch it:

```bash
conda activate coae
jupyter lab
```

This starts a local web server (usually at `http://localhost:8888`) and opens your default browser to the JupyterLab interface. From there you can:

1. Click **File > New > Notebook** and pick your `coae` kernel (the kernel is just "which Python environment this notebook is talking to").
2. Start typing code into the first cell and run it with `Shift+Enter`.

### Kernels: The Bridge Between Notebook and Environment

A **kernel** is the running Python process behind a notebook. Each conda environment can register itself as a selectable kernel, which is how a single JupyterLab install can run notebooks against many different environments.

```bash
# Register the "coae" environment as a Jupyter kernel
python -m ipykernel install --user --name coae --display-name "Python (coae)"
```

After this, "Python (coae)" appears as a kernel option in JupyterLab's notebook creation menu and in the kernel-switcher in the top right of an open notebook.

---

## 7. Managing Python Dependencies

Beyond conda's own environment files, Python projects commonly track dependencies with `pip` and a **requirements file**.

### requirements.txt

A plain text file listing packages (and optionally exact versions) that a project needs.

```text
# requirements.txt
numpy==1.26.4
pandas==2.2.1
scikit-learn==1.4.1.post1
matplotlib==3.8.3
torch==2.3.0
torchvision==0.18.0
jupyterlab==4.1.5
```

```bash
# Install everything listed
pip install -r requirements.txt

# Generate a requirements file from your current environment
pip freeze > requirements.txt
```

### Why Pin Exact Versions?

| Approach | Example | Risk |
|----------|---------|------|
| **Unpinned** | `pandas` | A future `pandas` release could change behavior and silently break your preprocessing code |
| **Loosely pinned** | `pandas>=2.0` | Safer, but still allows breaking changes within major-version updates |
| **Exactly pinned** | `pandas==2.2.1` | Reproducible -- everyone who installs this gets byte-identical behavior |

For security research specifically, exact pinning matters because a model's behavior (including its vulnerabilities to specific adversarial techniques) can shift subtly between library versions. If you cannot reproduce the exact library stack, you cannot reproduce the exact attack or defense result.

### conda environment.yml vs. requirements.txt

| File | Managed By | Captures OS-Level Dependencies? | Typical Use |
|------|-----------|----------------------------------|--------------|
| `environment.yml` | conda | Yes (e.g., CUDA toolkit version) | Full reproducible environment, cross-platform ML setups |
| `requirements.txt` | pip | No | Python-only dependency lists, often used *inside* an already-created conda environment |

A common, practical pattern: use conda to pin Python's version and any heavy system-level dependencies, then use `pip install -r requirements.txt` inside that environment for the rest.

---

## 8. A Full COAE Lab Setup Walkthrough

Here is a complete, start-to-finish setup you would realistically run before starting any exercise in this module.

```bash
# 1. Create and activate a dedicated environment
conda create -n coae python=3.11 -y
conda activate coae

# 2. Install the core stack
pip install numpy pandas matplotlib seaborn scikit-learn jupyterlab \
            torch torchvision ipykernel

# 3. Register this environment as a Jupyter kernel
python -m ipykernel install --user --name coae --display-name "Python (coae)"

# 4. Freeze exact versions for reproducibility
pip freeze > requirements.txt

# 5. Launch JupyterLab and select the "Python (coae)" kernel
jupyter lab
```

Inside your first notebook cell, a quick sanity check that everything is wired up correctly:

```python
import sys
import numpy as np
import pandas as pd
import sklearn
import torch

print("Python:", sys.version)
print("NumPy:", np.__version__)
print("pandas:", pd.__version__)
print("scikit-learn:", sklearn.__version__)
print("PyTorch:", torch.__version__)
print("CUDA available:", torch.cuda.is_available())
```

Expected-style output:

```
Python: 3.11.8 (main, ...)
NumPy: 1.26.4
pandas: 2.2.1
scikit-learn: 1.4.1.post1
PyTorch: 2.3.0
CUDA available: False   # (True if you have a supported NVIDIA GPU configured)
```

If every line prints cleanly with no `ImportError`, your environment is ready for the rest of this module.

---

## 9. Common Pitfalls

| Pitfall | Symptom | Fix |
|---------|---------|-----|
| Forgetting to activate the environment | `ModuleNotFoundError` even though you "installed" the package | Run `conda activate coae` before installing or running anything |
| Mixing `conda install` and `pip install` carelessly for the same package | Version conflicts, broken environments | Prefer one tool per package family; if using both, install with conda first, then pip for anything conda does not have |
| Installing packages into the `base` environment | Your `base` environment becomes bloated and fragile over time | Always create a dedicated named environment per project |
| Jupyter opens but shows the wrong kernel | Code runs against a different Python than you expect, "it works on my machine" bugs | Explicitly select "Python (coae)" from the kernel menu, and register kernels for every environment you use |
| Not pinning versions | A working notebook stops working weeks later after a library update | Use `pip freeze > requirements.txt` right after your setup works |
| GPU code failing silently on a CPU-only machine | `torch.cuda.is_available()` returns `False`, training is very slow | Confirm hardware first; write code that checks `torch.cuda.is_available()` and falls back to CPU gracefully |

---

## 10. Key Takeaways

- **Miniconda** gives you `conda` (a package + environment manager) plus a minimal Python install -- the foundation for every exercise in this module.
- **Environments isolate dependencies per project**, so a spam-classification project and a malware-classification project can use completely different library versions without conflicting.
- **conda** handles both Python packages and system-level dependencies (like CUDA); **pip** handles Python packages only and is typically used *inside* a conda environment for the rest of the install.
- **JupyterLab** is a browser-based notebook environment built for the iterative, inspect-as-you-go workflow that ML work demands -- code, output, charts, and notes live together in one `.ipynb` file.
- **Kernels** connect a notebook to a specific conda environment, so make sure you always know (and select) the right kernel.
- **Pin your dependencies** with `requirements.txt` or `environment.yml` so your work -- and any security findings built on top of it -- is reproducible.

*Next up: Python Libraries for ML -- a tour of scikit-learn and PyTorch, the two toolkits you will use throughout every hands-on project in this module.*
