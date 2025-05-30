## Overview
The `kmcos/runfile_init.py` module is designed to manage the initialization and state restoration of Kinetic Monte Carlo (KMC) simulations within the `kmcos` framework. It is particularly relevant when using advanced simulation features like "snapshots" (saving simulation states) and "throttling" (adjusting simulation parameters dynamically or between runs). The module provides functions to set up parameters for new simulation runs or to load parameters and KMC model states from previously saved files, enabling continuation or replication of simulations.

## Key Components

*   **`set_runfile_params(load_saved_parameters_on)` (Function)**:
    *   Purpose: Sets or resets specific simulation parameters. It's called during the initialization process.
    *   Functionality:
        *   Modifies global variables within imported modules `tg` (assumed to be `throttling_globals`) and `sg` (assumed to be `snapshots_globals`).
        *   Sets the simulation cutoff time (e.g., `tg.cutoff_time = 100.0`).
        *   Sets model parameters like temperature (e.g., `sg.model.parameters.T = 600`).
        *   Calls `set_PRNG_state` to initialize or restore the state of the Pseudo-Random Number Generator.
    *   Note: The parameter `load_saved_parameters_on` influences how the PRNG state is set.

*   **`load_saved_data()` (Function)**:
    *   Purpose: Loads previously saved simulation state and parameters from files.
    *   Functionality:
        *   Defines hardcoded filenames for parameter files related to "throttling" (e.g., `test_reaction_throttling_parameters.txt`) and "snapshots" (e.g., `test_reaction_snapshots_parameters.txt`).
        *   Uses an external library `export_import_library` (aliased as `eil`) to handle the actual loading of parameters into the `tg` and `sg` modules.
        *   Restores the KMC model's state within `sg.model` by setting its time (`sg.model.base.set_kmc_time`), step count (`sg.model.base.set_kmc_step`), and lattice configuration (`sg.model._set_configuration`).
        *   Updates a snapshot counter (`tg.current_snapshot`).

*   **`init(load_saved_parameters_on=False)` (Function)**:
    *   Purpose: The main initialization entry point for a simulation run.
    *   Functionality:
        *   If `load_saved_parameters_on` is `True`, it calls `load_saved_data()` to restore the simulation state from files.
        *   It then calls `set_runfile_params()` to apply any further parameter configurations or overrides.
        *   *Potential Typo Noted: The original code calls `set_runfile_params(load_saved_data_on)`, where `load_saved_data_on` is not defined; it likely intends to use `load_saved_parameters_on`.*

*   **`set_PRNG_state(load_saved_parameters_on, random_seed=None)` (Function)**:
    *   Purpose: Manages the initialization of the Pseudo-Random Number Generator (PRNG).
    *   Functionality:
        *   If `load_saved_parameters_on` is `True`, it seeds the PRNG using a state loaded from `sg.PRNG_state` via `snapshots.seed_PRNG(restart=True, ...)`. This ensures reproducibility when continuing a saved simulation.
        *   Otherwise (for a new simulation), it seeds the PRNG using `snapshots.seed_PRNG(restart=False, ...)`, potentially using the `random_seed` if provided.

## Important Variables/Constants

*   **Module-level globals `sg` and `tg`**: These are imported (conditionally from `kmcos.snapshots_globals`, `kmcos.throttling_globals` or directly as `snapshots_globals`, `throttling_globals`). They are expected to hold the live simulation state that this module reads from and writes to. Examples:
    *   `sg.model`: An instance of the KMC model (likely `kmcos.run.KMC_Model`).
    *   `tg.cutoff_time`: The simulation end time.
    *   `sg.kmc_time`, `sg.steps_so_far`, `sg.config`, `sg.PRNG_state`: Variables holding the state loaded from snapshot files.
*   **Hardcoded Filenames**: Strings like `'test_reaction_throttling_parameters.txt'` and `'test_reaction_snapshots_parameters.txt'` are used for saving and loading parameters.

## Usage Examples

This module is intended to be part of a larger simulation script or framework.

**Scenario 1: Starting a new simulation with defined parameters**
```python
# In your main simulation script:
# import kmcos.runfile_init as runfile_init
# from kmcos.snapshots_globals import sg # Assuming sg is properly initialized elsewhere
# from kmcos.throttling_globals import tg # Assuming tg is properly initialized elsewhere
# from kmcos.run import KMC_Model # Or your KMC model class

# sg.model = KMC_Model() # Initialize your KMC model

# Initialize for a new run (not loading saved data)
# runfile_init.init(load_saved_parameters_on=False)

# After this call:
# - tg.cutoff_time would be set (e.g., to 100.0).
# - sg.model.parameters.T would be set (e.g., to 600).
# - The PRNG would be seeded for a new run.
# print(f"Simulation will run until t = {tg.cutoff_time}")
# print(f"Model temperature set to: {sg.model.parameters.T}")
```

**Scenario 2: Resuming a simulation from saved data**
```python
# In your main simulation script:
# import kmcos.runfile_init as runfile_init
# from kmcos.snapshots_globals import sg
# from kmcos.throttling_globals import tg
# from kmcos.run import KMC_Model

# sg.model = KMC_Model() # Initialize KMC model (state will be overwritten by loaded data)

# Initialize by loading saved data
# runfile_init.init(load_saved_parameters_on=True)

# After this call (assuming save files exist and are valid):
# - Parameters from 'test_reaction_throttling_parameters.txt' and 
#   'test_reaction_snapshots_parameters.txt' would be loaded into tg and sg.
# - sg.model's KMC time, step, and lattice configuration would be restored.
# - The PRNG state would be restored.
# - tg.cutoff_time and sg.model.parameters.T would then be (re-)set by set_runfile_params.
# print(f"Resumed simulation. KMC time is now: {sg.model.base.get_kmc_time()}")
# print(f"Current snapshot number: {tg.current_snapshot}")
```

## Dependencies and Interactions

*   **`export_import_library` (aliased as `eil`)**: This external library is fundamental for the loading/saving functionality. It provides the `module_export_import` class used to read parameters from files into the `sg` and `tg` modules.
*   **`kmcos.snapshots_globals` (imported as `sg`)** and **`kmcos.throttling_globals` (imported as `tg`)**: (Or their non-namespaced equivalents). This `runfile_init.py` module directly reads from and writes to variables within these global state modules. It relies on these modules to store the live simulation context (like the KMC model instance `sg.model`, PRNG state `sg.PRNG_state`, etc.).
*   **`kmcos.snapshots` (imported as `snapshots`)** and **`kmcos.throttling` (imported as `throttling`)**:
    *   The `snapshots.seed_PRNG()` function is explicitly called to manage the state of the random number generator.
*   **`numpy`**: Used implicitly when restoring the lattice configuration: `sg.model._set_configuration(np.array(sg.config))`.
*   **`kmcos.run.KMC_Model` (or similar, via `sg.model`)**: The module heavily interacts with the `sg.model` object, expecting it to have:
    *   A `base` attribute with methods like `set_kmc_time()` and `set_kmc_step()`.
    *   A `parameters` attribute (e.g., `sg.model.parameters.T`).
    *   Methods like `_set_configuration()` and `_adjust_database()` for restoring the lattice state.
*   **File System**: Reads data from specified text files (e.g., `test_reaction_throttling_parameters.txt`) when the `load_saved_parameters_on` option is enabled during initialization. The paths to these files are hardcoded.

This module acts as a setup and state management utility, orchestrating the initialization of a `kmcos` simulation by either setting default/new parameters or by restoring a complex state from disk, ensuring consistency especially for restartable and long-running simulations involving snapshots.
