## Overview
The `kmcos/run/acf.py` module provides a Python application programming interface (API) for utilizing the Autocorrelation Function (ACF) and Mean Squared Displacement (MSD) functionalities within a compiled `kmcos` Kinetic Monte Carlo (KMC) model. It serves as a wrapper around lower-level Fortran routines, which are responsible for the actual computation and data management. This module allows users to configure, initialize, run, and retrieve results from ACF and MSD simulations programmatically using Python.

## Key Components
The module is composed of several Python functions that interact with a `KMC_Model` object (which contains the compiled Fortran backend). These functions can be categorized as follows:

**1. Allocation Functions:**
*   `allocate_acf(kmc_model, nr_of_types, t_bin, t_f, safety_factor=None, extending_factor=None)`: Allocates the necessary arrays and internal structures in the Fortran backend for ACF calculations. Parameters include the number of distinct property types to track, the time bin size (`t_bin`), the final time for ACF (`t_f`), and optional factors for buffer management.
*   `allocate_trajectory(kmc_model, nr_of_steps)`: Allocates arrays in Fortran for recording the trajectories of tracked particles over a specified number of KMC steps.

**2. Configuration and Setup Functions:**
*   `set_types_acf(kmc_model, site_property)`: Defines a numerical property value and associates it with an internal "type index" in the Fortran backend.
*   `calc_product_property(kmc_model)`: Calculates a matrix of all possible products g(0)g(t) based on the defined property types and stores it in the Fortran backend. This is used for ACF computation.
*   `set_property_acf(kmc_model, layer_site_name, property_type)`: Assigns a previously defined property type (via its index) to specific sites on the KMC lattice. The `layer_site_name` refers to a site identifier like `'default_a_1'`.

**3. Initialization Functions:**
*   `initialize_acf(kmc_model, species)`: Prepares the Fortran backend to start an ACF calculation for particles of the specified `species` (e.g., `'ion'`). This typically involves identifying initial particles of that species and their properties.
*   `initialize_msd(kmc_model, species)`: Similar to `initialize_acf`, but prepares for an MSD calculation for the given `species`.

**4. KMC Step Execution Functions:**
*   `do_kmc_steps_acf(kmc_model, n, traj_on=False)`: Executes `n` KMC simulation steps while actively collecting data required for the ACF. Trajectory recording can be optionally enabled.
*   `do_kmc_steps_displacement(kmc_model, n, traj_on=False)`: Executes `n` KMC simulation steps while collecting data for MSD calculations (i.e., particle displacements). Trajectory recording is also optional.

**5. Data Retrieval (Getter) Functions:**
These functions retrieve various arrays and values from the Fortran backend, typically returning them as NumPy arrays.
*   `get_id_arr(kmc_model)`: Returns an array of IDs for tracked particles.
*   `get_site_arr(kmc_model)`: Returns an array of site indices for tracked particles.
*   `get_property_o(kmc_model)`: Returns an array of initial properties g(0) for each tracked particle.
*   `get_property_acf(kmc_model)`: Returns an array mapping lattice sites to their assigned property type indices.
*   `get_buffer_acf(kmc_model)`: Returns an array of the g(0)g(t) products for tracked particles.
*   `get_config_bin_acf(kmc_model)`: Returns the raw sums collected in each ACF time bin.
*   `get_counter_write_in_bin_acf(kmc_model)`: Returns the count of contributions to each time bin.
*   `get_acf(kmc_model, normalization=False)`: Calculates and returns the ACF. Normalization (dividing by ACF[0]) is optional.
*   `get_types_acf(kmc_model)`: Returns an array of the property values associated with each type index.
*   `get_product_property(kmc_model)`: Returns the full g(0)g(t) matrix.
*   `get_trajectory(kmc_model)`: Returns the recorded particle trajectories.
*   `get_displacement(kmc_model)`: Returns the recorded net displacements for tracked particles.

**6. Calculation Functions:**
*   `calc_msd(kmc_model)`: Calculates and returns the Mean Squared Displacement based on the collected displacement data.

**7. Reset Functions:**
*   `set_acf_to_zero(kmc_model)`: Resets all ACF-related arrays and parameters in the Fortran backend to their initial states, allowing for a new ACF calculation to begin.
*   `set_displacement_to_zero(kmc_model)`: Resets displacement data in the Fortran backend.

## Important Variables/Constants
The module itself does not define significant variables or constants. Its operation relies on:
*   **`kmc_model` object**: An instance of `kmcos.run.KMC_Model`. This object acts as the bridge to the compiled Fortran code and must have the ACF-specific modules (e.g., `base_acf`, `proclist_acf`) available as attributes.
*   **Parameters passed to functions**: Values like `nr_of_types`, `t_bin`, `t_f`, `species` name, etc., control the behavior of the underlying Fortran routines.

## Usage Examples
The typical workflow for using this module to calculate an ACF is as follows (refer to the module's docstring for a detailed code example):

1.  **Obtain `KMC_Model`**: Load or create a `kmcos.run.KMC_Model` instance. This model must have been compiled with ACF support (usually via the `--acf` flag during `kmcos export`).
    ```python
    from kmcos.run import KMC_Model
    model = KMC_Model() # Assumes model files are in the current directory
    ```

2.  **Define ACF Parameters**: Specify parameters like the number of property types, time bin size, and total ACF duration.
    ```python
    nr_of_types = 2  # e.g., for two distinct properties
    t_bin = 0.001    # Time step for ACF bins
    t_f = 0.1        # Total time for ACF
    ```

3.  **Allocate Resources**: Call `allocate_acf` to prepare the Fortran backend.
    ```python
    from kmcos.run import acf as kmc_acf # Convention
    kmc_acf.allocate_acf(model, nr_of_types, t_bin, t_f)
    ```

4.  **Define Properties**:
    *   Use `set_types_acf` to associate numerical property values with type indices.
    *   Call `calc_product_property` to pre-calculate g(0)g(t) products.
    *   Use `set_property_acf` to assign these property types to specific sites on the lattice.
    ```python
    kmc_acf.set_types_acf(model, 0.5) # Type 1 will have property 0.5
    kmc_acf.set_types_acf(model, 1.0) # Type 2 will have property 1.0
    kmc_acf.calc_product_property(model)
    kmc_acf.set_property_acf(model, 'lattice_site_a', 1) # Assign Type 1 to 'lattice_site_a'
    kmc_acf.set_property_acf(model, 'lattice_site_b', 2) # Assign Type 2 to 'lattice_site_b'
    ```

5.  **Initialize ACF Calculation**: Specify the particle species to track.
    ```python
    kmc_acf.initialize_acf(model, species='tracked_species_name')
    ```

6.  **Run KMC Simulation**: Execute KMC steps with ACF data collection.
    ```python
    nr_of_simulation_steps = 1000000
    kmc_acf.do_kmc_steps_acf(model, nr_of_simulation_steps)
    ```

7.  **Retrieve ACF**: Get the calculated ACF.
    ```python
    ACF_data = kmc_acf.get_acf(model, normalization=True)
    # ACF_data can now be plotted or further analyzed
    ```

A similar workflow applies for MSD calculations using `initialize_msd`, `do_kmc_steps_displacement`, and `calc_msd`.

## Dependencies and Interactions

*   **`numpy`**: This module relies heavily on NumPy. All array data retrieved from the Fortran backend (e.g., via `get_acf`, `get_trajectory`) is returned as NumPy arrays.
*   **`kmcos.run.KMC_Model`**: A `KMC_Model` instance is a mandatory argument for all functions in this module. This object provides access to the compiled Fortran routines.
*   **Compiled Fortran Backend**:
    *   The core logic for ACF/MSD calculations, data storage, and KMC step execution resides in Fortran modules (e.g., `base_acf.f90`, `proclist_acf.f90`). These are compiled and linked into the `KMC_Model` object using f2py.
    *   This `acf.py` module essentially acts as a Python interface, making calls to functions within `model.base_acf` and `model.proclist_acf`.
*   **Prerequisites**: For this module to function correctly, the KMC model must have been exported and compiled with the ACF/MSD features enabled. This is typically controlled by a flag (e.g., `--acf`) during the `kmcos export` or `kmcos build` process, which ensures the necessary Fortran files (`base_acf.f90`, `proclist_acf.f90`) are generated by `kmcos.io` and included in the compilation.

In summary, `kmcos/run/acf.py` bridges the gap between high-level Python scripting and computationally intensive Fortran-based ACF/MSD simulations, offering users a convenient way to perform these advanced analyses.
