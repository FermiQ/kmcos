## Overview
The `kmcos/run/steady_state.py` module is designed to automatically detect the onset of a steady state in Kinetic Monte Carlo (KMC) simulations performed using `kmcos`. The primary purpose is to identify and exclude the initial transient (or "warm-up") phase of a simulation, ensuring that subsequent data analysis and averaging are based on data that accurately represents the system's long-term, stable behavior. This is achieved by implementing a statistical method based on Exponentially Weighted Moving Average (EWMA) control charts, as described in literature by M. Rossetti.

## Key Components

The module consists of several functions that work together to perform the steady-state analysis:

*   **`ewma_alpha(y, alpha, prev_ewma=None, adjust=True)`**:
    Calculates the Exponentially Weighted Moving Average for a given time series `y` using a smoothing factor `alpha`. This function is fundamental for smoothing out fluctuations in the simulation data.

*   **`lcl_ucl(y, cutoff, L, lambda_factor)`**:
    Computes the Lower Control Limit (LCL) and Upper Control Limit (UCL) for an EWMA chart. These limits define a statistically determined band around the estimated mean of the data.
    *   `y`: The input time series.
    *   `cutoff`: An index indicating the point from which data is used to estimate the mean and standard deviation (this part is assumed to be in steady state for the calculation of limits).
    *   `L`: A parameter that sets the width of the control limits (e.g., `L=3` for 3-sigma limits, though `L=4` is used by default in `sample_steady_state` for robustness).
    *   `lambda_factor`: The smoothing factor, typically the same as `alpha`.

*   **`p2d(y, cutoff, L, alpha)`**:
    Calculates the proportion of data points (after the `cutoff` index) for which the EWMA value falls within the LCL and UCL. This metric helps quantify how much of the data series appears to be "under control" or stable.

*   **`get_scrap_fraction(y, L, alpha, warm_up)`**:
    Determines the fraction of the initial data that should be considered part of the warm-up period and thus "scrapped." It iterates through different potential `cutoff` points, calculates `p2d` for each, and identifies the cutoff that maximizes the fraction of data under control after an initial `warm_up` number of data points.

*   **`sample_steady_state(model, batch_size=1000000, L=4, alpha=0.05, bias_threshold=0.15, tof_method='integ', warm_up=20, check_frequency=10, show_progress=True, make_plots=False, output='str', seed='EWMA')`**:
    This is the main user-facing function. It automates the process of running a KMC simulation until a steady state is detected.
    *   `model`: An instance of `kmcos.run.KMC_Model`.
    *   `batch_size`: The number of KMC steps to run in each data collection batch.
    *   It iteratively runs simulation batches, collects time-averaged data (e.g., coverages, turnover frequencies) for each batch.
    *   It applies the EWMA analysis (`get_scrap_fraction`) to these batch-averaged quantities.
    *   The simulation continues until the `max_scrap` fraction (the highest scrap fraction among all monitored quantities) is below the specified `bias_threshold`.
    *   Once converged, it averages the data from the identified steady-state region and returns it.
    *   Optional features include progress bars (`show_progress`) and diagnostic plots (`make_plots`).

*   **`plot_normal(y, n=-1, *args, **kwargs)`** and **`make_ewma_plots(data, L, alpha, bias_threshold, seed)`**:
    Helper functions for visualizing the data and the EWMA control charts, primarily for debugging and validation. These require `matplotlib`.

## Important Variables/Constants

Within `sample_steady_state`:
*   **`L`** (default: `4`): Multiplier for standard deviation to define LCL/UCL width.
*   **`alpha`** (default: `0.05`): Smoothing factor for EWMA.
*   **`bias_threshold`** (default: `0.15`): The target maximum fraction of data to be considered "warm-up" for the simulation to be deemed in steady state.
*   **`warm_up`** (default: `20`): The number of initial batches to simulate before starting to apply the steady-state detection logic.
*   **`check_frequency`** (default: `10`): How often (in terms of batches) the steady-state analysis is performed.

## Usage Examples

The primary way to use this module is by calling the `sample_steady_state` function. This is typically done after initializing a `KMC_Model` object.

```python
import kmcos.run
from kmcos.run import steady_state # Explicit import

# This assumes you have a compiled KMC model in the current directory
# or that KMC_Model can find its necessary files.
try:
    with kmcos.run.KMC_Model(banner=False, print_rates=False) as model:
        print("Starting steady-state sampling...")
        # Run simulation until steady state is detected, then get averaged data
        results_string = steady_state.sample_steady_state(
            model,
            batch_size=50000,       # KMC steps per batch
            L=4,                    # Control limit width factor
            alpha=0.05,             # EWMA smoothing factor
            bias_threshold=0.10,    # Stricter threshold for steady state
            tof_method='integ',     # Method for TOF calculation
            warm_up=15,             # Batches before first check
            check_frequency=5,      # Check every 5 batches
            show_progress=True,     # Display progress bar
            make_plots=False,       # Set to True to generate diagnostic plots (requires matplotlib)
            output='dict'           # Get results as a dictionary
        )
        
        print("\nSteady-state analysis complete.")
        print("Steady-state averaged data:")
        for key, value in results_string.items():
            print(f"  {key}: {value:.5e}")

except Exception as e:
    print(f"An error occurred: {e}")
    print("Ensure a compiled KMC model is available for KMC_Model() to load.")

```

## Dependencies and Interactions

*   **`numpy`**: Essential for all numerical computations, including array manipulations, mean, standard deviation, and EWMA calculations.
*   **`kmcos.run.KMC_Model`**: The `sample_steady_state` function critically depends on an instance of `KMC_Model`. It uses this object to:
    *   Run KMC simulation steps.
    *   Obtain time-averaged data per batch (via `model.get_std_sampled_data()`).
    *   Get data headers (via `model.get_std_header()`).
*   **`kmcos.utils.progressbar`** (optional): If `show_progress=True`, this utility is used to display an ASCII progress bar during the simulation.
*   **`matplotlib.pyplot`** (optional): If `make_plots=True`, Matplotlib is used by `make_ewma_plots` to generate diagnostic plots of the EWMA control charts. If Matplotlib is not installed and plots are requested, an `ImportError` will occur.
*   **Standard Python Libraries**:
    *   `itertools` (for `itertools.count` in `sample_steady_state`).
    *   `pprint` (mentioned in module comments, but not directly used in the provided code snippet).

This module provides a valuable tool for robust KMC simulation by automating the often tricky process of determining when a simulation has reached a reliable steady state, thereby improving the quality and reproducibility of simulation results.
