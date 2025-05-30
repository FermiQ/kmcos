## Overview
The `kmcos/io.py` module is a cornerstone of the `kmcos` (Kinetic Monte Carlo Open-source Software) package. It handles crucial input/output operations, focusing on the translation of KMC project definitions between different representations. Its main functionalities include:

1.  **Importing**: Reading KMC model definitions from XML files.
2.  **Exporting**:
    *   Generating Fortran 90 source code from the KMC model. This code is then compiled to create the simulation engine.
    *   Creating a `kmc_settings.py` file, which contains user-adjustable parameters, rate constants, species information, and an embedded copy of the model's XML definition.
3.  **Code Generation**: Supporting multiple Fortran code generation backends/strategies (e.g., `local_smart`, `lat_int`, `otf`) to optimize for different model types or simulation needs. It can also incorporate advanced features like temporal acceleration and autocorrelation function (ACF) calculations into the generated code.

## Key Components

*   **`ProcListWriter(data, dir)` (Class)**:
    This is the central class responsible for generating all the dynamic Fortran 90 source files and the `kmc_settings.py` file.
    *   `__init__(self, data, dir)`: Constructor that takes the project data (an instance of `kmcos.types.Project`) and the target output directory.
    *   `write_template(self, filename, target=None, options=None)`: A utility method to load a Fortran template file (with a `.mpy` extension, located in `kmcos/fortran_src/`) and fill it with project-specific data to produce a `.f90` source file.
    *   `write_proclist(self, smart=True, code_generator='local_smart', accelerated=False)`: The main method for generating `proclist.f90`. This Fortran module defines the KMC processes, their conditions, actions, and rates. The output varies significantly based on the chosen `code_generator` and whether `accelerated` mode is enabled.
    *   `write_proclist_acf(...)`: Generates `proclist_acf.f90`, containing routines for calculating autocorrelation functions or mean squared displacements if requested.
    *   `write_proclist_constants(...)`, `write_proclist_generic_subroutines(...)`: These methods generate common Fortran code, including parameter definitions and utility subroutines used by the specific backend.
    *   Specialized `write_proclist_*` methods for different backends (e.g., `write_proclist_run_proc_nr_smart`, `write_proclist_lat_int`, `write_proclist_otf`): These generate the core logic for process execution and updates for each backend.
    *   `_write_optimal_iftree(...)`, `_write_optimal_iftree_otf(...)`: Private helper methods that construct optimized Fortran `SELECT CASE` or `IF-THEN-ELSEIF-ELSE` structures for efficiently checking process conditions.
    *   `write_settings(self, code_generator='lat_int', accelerated=False)`: Generates the `kmc_settings.py` file. This Python file stores model parameters, initial rate constants, species representations (for visualization), TOF (Turnover Frequency) counting configurations, and the full XML definition of the model.
    *   `_otf_get_auxilirary_params(...)`, `_parse_otf_rate(...)`: Helper methods for the 'otf' (on-the-fly) backend, used to parse and translate rate expressions that might depend on runtime conditions.

*   **`export_source(project_tree, export_dir=None, code_generator=None, options=None, accelerated=False)` (Function)**:
    The primary top-level function for exporting a `kmcos.types.Project` instance into a full set of compilable Fortran 90 source files and the associated `kmc_settings.py`. It:
    1.  Creates the specified `export_dir`.
    2.  Copies static Fortran files (like `kind_values.f90`, `main.f90`, and backend-specific base modules) from `kmcos/fortran_src/` to the `export_dir`.
    3.  Instantiates `ProcListWriter` and calls its methods to generate all dynamic Fortran files (`proclist.f90`, `lattice.f90`, etc.) and `kmc_settings.py`.

*   **`import_xml_file(filename)` (Function)**:
    Parses an XML file specified by `filename` and returns a `kmcos.types.Project` object representing the KMC model.

*   **`import_xml(xml_string)` (Function)**:
    Parses an XML string and returns a `kmcos.types.Project` object.

*   **`export_xml(project_tree, filename=None)` (Function)**:
    Writes the XML representation of a `kmcos.types.Project` object to the specified `filename`. If no filename is given, it defaults to `model_name.xml`.

*   **`clear_model(model_name, backend="local_smart")` (Function)**:
    A utility to delete files and directories associated with a previously exported model. This is often used to ensure a clean export. *Note: It uses direct `os.system("del ...")` and `os.system("rm ...")` calls, making its success platform-dependent.*

## Important Variables/Constants
*   **`APP_ABS_PATH`**: Imported from `kmcos.config`, this variable provides the absolute path to the `kmcos` installation directory, which is used to locate the `fortran_src` template directory.
*   **Template Files (`*.mpy`)**: Located in `kmcos/fortran_src/`, these files are crucial. They are not Python modules but text templates that `ProcListWriter` uses to generate the Fortran code. They contain placeholders that are filled in with model-specific details.
*   **`project_tree` / `self.data`**: An instance of `kmcos.types.Project`, this object holds the entire definition of the KMC model (species, sites, processes, parameters, etc.) and is the primary input for the export process.
*   **`code_generator`**: A string (e.g., `'local_smart'`, `'lat_int'`, `'otf'`) that dictates the strategy and type of Fortran code to be generated. This choice affects the structure and potential performance characteristics of the resulting simulation.

## Usage Examples

**1. Importing a KMC model from an XML file:**
```python
from kmcos import io
from kmcos import types

# Assuming "my_model.xml" contains a valid kmcos model definition
try:
    project = io.import_xml_file("my_model.xml")
    print(f"Successfully imported model: {project.meta.model_name}")
except FileNotFoundError:
    print("Error: my_model.xml not found.")
except Exception as e:
    print(f"Error importing XML: {e}")
```

**2. Exporting a KMC model to Fortran source code:**
```python
from kmcos import io
from kmcos import types # Assuming 'project' is an instance of types.Project

# Create a dummy project for demonstration if you don't have one
# project = types.Project()
# project.meta.model_name = "test_model"
# ... (further populate the project object)

# Assuming 'project' is a populated kmcos.types.Project object
export_directory = "exported_model_files"
try:
    io.export_source(project,
                     export_dir=export_directory,
                     code_generator="local_smart", # or "lat_int", "otf"
                     accelerated=False)
    print(f"Model exported to {export_directory}")
    # The 'exported_model_files' directory will now contain:
    # - Fortran source files (*.f90)
    # - kmc_settings.py
    # - assert.ppc
except Exception as e:
    print(f"Error exporting source: {e}")
```
(Note: A minimal `project` object needs more setup to be fully exportable.)

**3. Writing `kmc_settings.py` (typically done via `export_source`):**
The `ProcListWriter.write_settings()` method is responsible for this. When `io.export_source(...)` is called, it internally calls this method. The `kmc_settings.py` file will be created in the specified export directory.

## Dependencies and Interactions

*   **Internal `kmcos` Modules**:
    *   `kmcos.types`: Essential for the data structures (`Project`, `Species`, `Process`, `ConditionAction`, `Coord`, etc.) that define the KMC model.
    *   `kmcos.config`: Provides `APP_ABS_PATH` for locating resource files (Fortran templates).
    *   `kmcos.utils`: Uses `evaluate_template` for processing the `.mpy` template files and `ProgressBar` for visual feedback during export.
    *   `kmcos` (root package): For `evaluate_rate_expression`, `rate_aliases`, `units` used in parsing and evaluating rate constant expressions, especially for the 'otf' backend.

*   **Standard Python Libraries**:
    *   `os`: Extensive use for path manipulation, file system checks (existence, creation of directories), and `os.system` calls in `clear_model`.
    *   `shutil`: For copying static files (e.g., `kind_values.f90`) during the export process.
    *   `sys`: Used for system-related information, though less prominently in direct public interfaces.
    *   `copy`: For duplicating objects, e.g., `ConditionAction` lists.
    *   `itertools`, `operator`, `functools`: Used for list manipulation, sorting, and functional programming helpers in code generation logic (e.g., `_most_common`, `cmp_to_key`).
    *   `pprint`: For pretty-printing data structures, likely during debugging.
    *   `collections`: For `Counter` (used in `write_settings` for accelerated mode process pairing) and other collection utilities.
    *   `tempfile`: Used in `import_xml` to temporarily store an XML string as a file.
    *   `io`, `tokenize`, `re`: Used for parsing rate expressions, particularly for the 'otf' backend.

*   **Third-Party Libraries**:
    *   `numpy`: While not directly imported in `io.py` for its own operations, the generated Fortran code often relies on NumPy-compatible array indexing and the overall f2py compilation process is tied to NumPy. The logic within `ProcListWriter` generates code that interacts with data structures managed by the Fortran side, which are then exposed to Python via f2py as NumPy arrays.

*   **File System**:
    *   Reads `.xml` files for model import.
    *   Reads `.mpy` template files from its `fortran_src` directory.
    *   Writes multiple `.f90` Fortran source files to the specified export directory.
    *   Writes `kmc_settings.py` to the export directory.
    *   Creates directories if they don't exist.

*   **Generated Fortran Code**: The primary "interaction" is the production of Fortran code. The `io.py` module translates the Python-based KMC model definition into a set of Fortran modules that can be compiled into a runnable simulation.

*   **XML Format**: Relies on a specific XML schema for defining KMC models.

This module acts as a bridge between the abstract model definition and the low-level, performance-oriented Fortran code required for efficient KMC simulations. The complexity arises from the need to support different code generation strategies and features while maintaining a consistent interface.
