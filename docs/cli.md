## Overview
This module, `kmcos/cli.py`, serves as the primary entry point for the command-line interface (CLI) of the `kmcos` (Kinetic Monte Carlo Open-source Software) application. It allows users to interact with various `kmcos` functionalities by issuing commands in a terminal. These commands facilitate tasks such as building KMC models, exporting source code, running simulations, benchmarking performance, and viewing simulation results.

## Key Components
*   **`usage` (dictionary)**: A dictionary that stores detailed help messages and syntax for each available CLI command (e.g., `benchmark`, `build`, `export`).
*   **`get_options(args=None, get_parser=False)` (function)**: Responsible for parsing command-line arguments and options provided by the user. It utilizes the `optparse` module to define and read various flags (e.g., `-d` for debug, `-s` for source-only).
*   **`match_keys(arg, usage, parser)` (function)**: Enables users to use shortened versions of commands. This function attempts to match an ambiguous, user-provided command string to one of the full command keys defined in the `usage` dictionary.
*   **`main(args=None)` (function)**: This is the core function that orchestrates the CLI's operation. It first parses the options and arguments, then matches the primary command. Based on the command, it executes the corresponding logic, which often involves calling other modules within the `kmcos` package (e.g., `kmcos.io` for file operations, `kmcos.run` for simulations).
*   **`sh(banner)` (function)**: A utility function that acts as a wrapper to launch an interactive IPython shell. It handles potential differences across various IPython versions to provide a consistent experience.
*   **Command Handling Logic (within `main()`)**: A series of `if/elif` blocks within the `main` function, where each block is dedicated to handling a specific CLI command (e.g., `benchmark`, `build`, `export`, `help`, `run`, `view`, `xml`). These blocks contain the specific instructions and function calls to perform the requested operation.

## Important Variables/Constants
*   **`usage` (dictionary)**: This is a critical data structure that maps command names (strings) to their respective help texts and usage instructions. It's used for generating help messages and validating commands.
*   **`options` (optparse.Values instance)**: An object returned by `get_options`. It holds the values of command-line options (flags) passed by the user, such as `options.debug` or `options.backend`. These values control the behavior of the executed commands.
*   **`args` (list)**: A list containing the positional command-line arguments. `args[0]` typically holds the main command (e.g., 'export'), and subsequent elements hold parameters for that command (e.g., input file paths).

## Usage Examples
The CLI supports a variety of commands. Here are a few examples:

*   **Display help for the 'export' command:**
    ```bash
    kmcos help export
    ```

*   **Export KMC model source code from an XML file:**
    ```bash
    kmcos export model_definition.xml /path/to/export_directory
    ```

*   **Build a KMC model in the current directory (assuming *.f90 files exist):**
    ```bash
    kmcos build --debug
    ```

*   **Run a benchmark test on the model in the current directory:**
    ```bash
    kmcos benchmark
    ```

*   **Open an interactive shell with the KMC model loaded:**
    ```bash
    kmcos run
    ```
    or
    ```bash
    kmcos shell
    ```

*   **View a simulation visually (requires kmc_model and kmc_settings.py):**
    ```bash
    kmcos view --steps-per-frame 10000
    ```

## Dependencies and Interactions
The `cli.py` module has several dependencies and interactions:

*   **Internal `kmcos` Modules**:
    *   `kmcos.types`: For project data structures.
    *   `kmcos.io`: For input/output operations, including reading XML files and exporting source code.
    *   `kmcos.utils`: For utility functions like building the Fortran model.
    *   `kmcos.run`: For creating and running KMC models (`KMC_Model` class).
    *   `kmcos.gui`: For the (deprecated) GUI editor.
    *   `kmcos.view`: For the visualization module.
    *   `kmcos.VERSION`: To access the version of the `kmcos` package.

*   **Standard Python Libraries**:
    *   `os`: For operating system interactions (paths, file system operations).
    *   `shutil`: For high-level file operations (moving files).
    *   `sys`: For system-specific parameters and functions (accessing command-line arguments, path manipulation).
    *   `time`: For time-related functions (used in benchmarking).
    *   `glob`: For finding files matching a pattern.
    *   `optparse`: For parsing command-line options.
    *   `tempfile`: For creating temporary files.

*   **Third-Party Libraries**:
    *   `IPython`: Used by the `sh()` function to provide an interactive shell environment.
    *   `numpy`: Used implicitly by other `kmcos` modules and for Fortran integration (`numpy.distutils`). Required for model compilation and execution.
    *   `matplotlib` (optional): Can be used by the `run`/`shell` command for plotting within the interactive session.
    *   `catmap` (optional): Can be used by the `run`/`shell` command for integrating with CatMAP models.

*   **System Interactions**:
    *   **File System**: Reads XML model definition files, writes Fortran source code, Python settings files, and compiled binaries. Creates and manages directories for exported projects.
    *   **Fortran Compiler**: Interacts with a Fortran compiler (typically via `f2py` provided by `numpy.distutils`) to build the computationally intensive parts of the KMC model from generated Fortran code.
    *   **IPython Shell**: Can launch an IPython session for interactive model exploration and simulation control.
