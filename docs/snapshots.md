## Overview
The `kmcos/snapshots.py` module provides a framework for running Kinetic Monte Carlo (KMC) simulations in segments called "snapshots." During each snapshot, specified KMC steps (or simulation time) are executed, and relevant data suchs as Turnover Frequencies (TOFs), species coverages (occupations), and selected model parameters are recorded. This data is typically written to CSV files for subsequent analysis. The module also supports logging simulation metadata, saving the final system configuration, and managing the Pseudo-Random Number Generator (PRNG) state, which is crucial for restarting or reproducing simulations.

It relies heavily on a companion module, `snapshots_globals` (typically imported as `sg`), which holds the live state of the simulation (e.g., the KMC model instance, simulation counters, configuration data).

## Key Components

*   **`create_headers(print_all_parameters=True)` (Function)**:
    *   Purpose: Initializes output files at the beginning of a simulation run.
    *   Functionality:
        *   Records the wall clock start time (stored in `sg.start`).
        *   Creates a primary CSV data file (e.g., `your_simulation_name_TOFs_and_Coverages.csv`).
        *   Writes headers to this CSV file for: simulation name, snapshot number, KMC step, KMC time, steps per second (sps), time per snapshot (tps), user-defined "parameters of interest" (from `sg.parameters_of_interest`), TOF data columns (from `sg.TOF_header_array`), integrated TOF columns, and occupation columns (from `sg.occ_header_array`).
        *   Writes the initial (time=0, step=0) values for these quantities to the CSV.
        *   Optionally, if `print_all_parameters` is `True`, creates a separate text file (e.g., `your_simulation_name_initial_parameters.txt`) listing all parameters of the KMC model and their initial values.

*   **`save_snapshot_output_as_list()` (Function)**:
    *   Purpose: Collects all relevant data for the current snapshot into an in-memory list (`sg.last_snapshot_outputs`).
    *   Functionality: Appends simulation name, current snapshot/step/time, calculated sps/tps, values of "parameters of interest", TOF data, integrated TOF data, and occupation data to `sg.last_snapshot_outputs`.
    *   Note: The module comment suggests this list might not be actively used by other parts of the module.

*   **`write_snapshot_data()` (Function)**:
    *   Purpose: Appends the data for the most recently completed snapshot to the main CSV file.
    *   Functionality: Writes a new row to the CSV file with the current values of simulation name, snapshot/step/time, sps, tps, parameters of interest, TOFs, and occupations.

*   **`do_snapshots(n_snapshots, sps, tps=None, acc=False)` (Function)**:
    *   Purpose: The main workhorse function that executes the KMC simulation over a series of snapshots.
    *   Arguments:
        *   `n_snapshots`: The total number of snapshots to perform.
        *   `sps`: The number of KMC steps to execute per snapshot.
        *   `tps` (optional): The target amount of KMC simulation time to elapse per snapshot. If provided, the simulation runs for at most `sps` steps or until `tps` time has passed.
        *   `acc` (bool, optional): If `True`, uses an accelerated KMC step function (`sg.model.do_acc_steps()`).
    *   Functionality:
        *   Loops `n_snapshots` times.
        *   In each iteration:
            *   Executes KMC steps using `sg.model.do_steps()`, `sg.model.do_acc_steps()`, or `sg.model.do_steps_time()`.
            *   Updates global counters in `sg` (e.g., `sg.steps_so_far`, `sg.snapshots_so_far`, `sg.kmc_time`).
            *   Retrieves updated atomic information using `sg.model.get_atoms()`.
            *   Calculates actual steps per snapshot (`sg.sps_actual`) and time per snapshot (`sg.tps_actual`).
            *   Calls `save_snapshot_output_as_list()`.
            *   If `sg.write_output` is `True`, calls `write_snapshot_data()` (data might be written less frequently if `sg.snapshots_sampling` is set to a value > 1).
        *   After the loop, updates global lists in `sg` (e.g., `sg.TOF_data_list`, `sg.occ_data_list`) with the final data.
        *   Saves the final lattice configuration to `sg.config` and the PRNG state to `sg.PRNG_state`.

*   **`create_log(dump_configuration=True, print_all_parameters=True)` (Function)**:
    *   Purpose: Called after all snapshots are completed to finalize the simulation logging.
    *   Functionality:
        *   Optionally, if `dump_configuration` is `True`, saves the final KMC model configuration (e.g., to `your_simulation_name_final_config`).
        *   Optionally, if `print_all_parameters` is `True`, writes the final values of all model parameters to a text file (e.g., `your_simulation_name_final_parameters.txt`).
        *   Calculates the total wall clock runtime of the simulation.
        *   Writes a log file (e.g., `your_simulation_name_log.txt`) containing a summary: total snapshots, total KMC steps, total KMC time, wall clock runtime, and final values of "parameters of interest."

*   **`seed_PRNG(restart=False, state=None)` (Function)**:
    *   Purpose: Initializes or restores the state of the Pseudo-Random Number Generator used by the KMC model.
    *   Functionality:
        *   If `restart` is `True` and `state` (a previously saved PRNG state) is provided, it attempts to restore the PRNG in `sg.model.proclist` to this state.
        *   If `restart` is `False` (for a new simulation), it initializes the PRNG, using `state` as a seed if provided (typically an integer).
        *   Includes error handling in case the KMC model's compiled `proclist` module does not support PRNG manipulation.

*   **`reload(sg_module_state)` (Function)**:
    *   Purpose: Restores the entire state of a simulation, presumably to continue a previous run.
    *   `sg_module_state`: An object (likely from the `export_import_library`) that can load data into the `sg` (snapshots_globals) module.
    *   Functionality:
        *   Loads data into `sg` using `sg_module_state.load_params()`. This would repopulate variables like `sg.steps_so_far`, `sg.kmc_time`, `sg.config`, `sg.PRNG_state`, etc.
        *   Resets the KMC step and time in the `sg.model` instance.
        *   Restores the lattice configuration of `sg.model` using `sg.config`.
        *   Calls `seed_PRNG(restart=True, state=sg.PRNG_state)` to restore the PRNG state.
        *   Attempts to update rate constants in the model (comment notes a potential bug here).

## Important Variables/Constants
This module is highly dependent on global variables defined within an imported module, typically `kmcos.snapshots_globals` (referred to as `sg`). Key variables managed in `sg` and used by this module include:

*   `sg.model`: The instance of the `kmcos` KMC model being simulated.
*   `sg.simulation_name` (str): A user-defined name for the simulation, used as a prefix for output files.
*   `sg.parameters_of_interest` (list of str): A list of KMC model parameter names whose values should be recorded at each snapshot.
*   `sg.TOF_header_array` (list of str): Headers for different TOF values.
*   `sg.occ_header_array` (list of str): Headers for different species/site occupations.
*   `sg.steps_so_far` (int): Cumulative number of KMC steps performed.
*   `sg.snapshots_so_far` (int): Number of snapshots already completed.
*   `sg.kmc_time` (float): Current cumulative KMC simulation time.
*   `sg.atoms`: An ASE Atoms-like object representing the current state of the system, obtained from `sg.model`.
*   `sg.write_output` (bool): Flag to enable/disable writing snapshot data to disk.
*   `sg.snapshots_sampling` (int, optional): If set, data is written to disk only every `sg.snapshots_sampling` snapshots.
*   `sg.config` (list): Stores the lattice configuration, used for saving/reloading.
*   `sg.PRNG_state` (list/int): Stores the PRNG state for saving/reloading.
*   `sg.start` (float): Wall clock time when the simulation started.

## Usage Examples

This module is intended to be imported and driven by a main simulation script.

**1. Setting up and running a new simulation with snapshots:**
```python
# In a main Python script:
# import kmcos.snapshots as snapshots
# import kmcos.snapshots_globals as sg
# from kmcos.run import KMC_Model # Or your KMC model loading mechanism

# --- Configure snapshots_globals (sg) ---
# sg.model = KMC_Model() # Load your compiled KMC model
# sg.simulation_name = "my_co_oxidation_run"
# sg.parameters_of_interest = ["T", "p_CO", "p_O2"] # Parameters to log
# sg.TOF_header_array = ["CO2_formation_rate"] # Example, get from model
# sg.occ_header_array = ["CO_on_Pt", "O_on_Pt"] # Example, get from model
# sg.write_output = True
# sg.snapshots_sampling = 1 # Log every snapshot

# --- Initialize PRNG for a new run ---
# snapshots.seed_PRNG(restart=False, state=123456)

# --- Create output file headers ---
# snapshots.create_headers(print_all_parameters=True)

# --- Run the simulation with snapshots ---
# num_snapshots_to_run = 200
# steps_per_snapshot = 50000
# snapshots.do_snapshots(n_snapshots=num_snapshots_to_run, sps=steps_per_snapshot)

# --- Finalize logging and save configuration ---
# snapshots.create_log(dump_configuration=True, print_all_parameters=True)
# print(f"Simulation {sg.simulation_name} complete.")
```

**2. Reloading and continuing a simulation (conceptual):**
```python
# import kmcos.snapshots as snapshots
# import kmcos.snapshots_globals as sg
# from kmcos.run import KMC_Model
# import export_import_library as eil # For loading sg's state

# --- Load sg module's saved state ---
# sg_loader = eil.module_export_import("sg_savefile.txt", "sg_loadfile.txt", sg)
# snapshots.reload(sg_loader) # This populates sg.* and sets up sg.model

# --- Continue simulation ---
# snapshots.do_snapshots(n_snapshots=100, sps=50000) # Run 100 more snapshots
# snapshots.create_log()
# print(f"Continued simulation {sg.simulation_name} complete.")
```

## Dependencies and Interactions

*   **`time` (Python standard library)**: Used to get the current wall clock time for calculating total runtime.
*   **`kmcos.snapshots_globals` (imported as `sg`)**: This is a **critical, hard dependency**. The `snapshots.py` module is stateful through `sg`. It reads its configuration (like `sg.simulation_name`, `sg.model`) from `sg` and writes its results and ongoing state (like `sg.steps_so_far`, `sg.config`) back to `sg`.
*   **`kmcos.run.KMC_Model` (via `sg.model`)**: All core KMC operations (advancing steps, getting atomic configurations, accessing parameters, PRNG functions) are performed through the `sg.model` object. This object is expected to be a fully initialized and compiled `kmcos` KMC model instance.
*   **`numpy`**: Used implicitly when `sg.model._get_configuration()` is called (returns a NumPy array) and when `sg.model._set_configuration(np.array(sg.config))` is used in `reload`.
*   **File System**:
    *   Creates and appends to a main CSV data file (`<simulation_name>_TOFs_and_Coverages.csv`).
    *   Creates text files for initial parameters (`<simulation_name>_initial_parameters.txt`), final parameters (`<simulation_name>_final_parameters.txt`), and a summary log (`<simulation_name>_log.txt`).
    *   Saves the final KMC configuration (e.g., `<simulation_name>_final_config`), format depends on `sg.model.dump_config`.
*   **`export_import_library` (aliased as `eil` in `kmcos.runfile_init`)**: While not directly used by most functions in `snapshots.py`, the `reload` function signifies that the state of the `sg` module itself is expected to be loadable, presumably using this library.

This module provides a comprehensive system for running KMC simulations in managed segments, facilitating data logging, analysis of transient behavior, and the ability to save and restart complex simulation states. Its tight coupling with `snapshots_globals` is a key architectural feature.
