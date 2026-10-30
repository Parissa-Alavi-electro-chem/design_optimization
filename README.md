# Hydrogel–Electrode Design Studies

Python notebooks for exploring cathode pin geometry and fixed-current transport through two gel compartments. The studies support early design comparisons: one calculates exposed geometric area, while the other screens transport and voltage requirements.

**The notebooks are independent.** Pin geometry is not automatically coupled to the transport solver. Increasing exposed electrode area in the geometry study does not automatically change the transport cross-section, electrical resistance, or delivered inventory in the 1D model.

## Repository contents

| File | Purpose |
| --- | --- |
| [`Geometric_comparison.ipynb`](Geometric_comparison.ipynb) | Compare pin shapes, taper, width, spacing, count, and height on a finite cathode face. |
| [`Gel_design_1d_code.ipynb`](Gel_design_1d_code.ipynb) | Compare diffusivity, conductivity, and transport length at fixed total current, with voltage screening and local sensitivity analysis. |

Both notebooks contain their model definitions. No COMSOL installation or external experimental dataset is required to run these studies. The transport notebook includes embedded default screening and contour results.

## Quick start on Windows

Install Python 3 if needed. Python 3.10 or newer is recommended as a starting environment; dependency versions are not pinned in the supplied notebooks.

Place this README and both notebooks in the same project folder. Open PowerShell in that folder and run:

```powershell
py -m venv .venv
.\.venv\Scripts\python.exe -m pip install --upgrade pip
.\.venv\Scripts\python.exe -m pip install jupyterlab ipykernel numpy scipy pandas matplotlib plotly
.\.venv\Scripts\python.exe -m jupyter lab
```

These commands use the virtual environment directly and do not require PowerShell activation scripts. If `py` is unavailable, use `python` for the first command, provided it points to Python 3.

In JupyterLab:

1. Open either notebook and select the environment's Python kernel.
2. Review its editable settings before running.
3. Use **Restart Kernel and Run All Cells** to execute in order.
4. Review the figures, tables, and any numerical checks.
5. Save the notebook to retain edited code and displayed results.

To reopen the project later:

```powershell
.\.venv\Scripts\python.exe -m jupyter lab
```

For macOS/Linux, create the environment with `python3 -m venv .venv` and replace the Windows executable path with `.venv/bin/python`.

### Dependencies

| Package | Used for |
| --- | --- |
| NumPy | Arrays and numerical calculations |
| SciPy | Banded linear solver in the transport model |
| pandas | Transport result tables and exports |
| Matplotlib | Static plots |
| Plotly | Optional interactive geometry viewers and surfaces |
| JupyterLab / IPython | Notebook execution and rich displays |

Plotly is optional for the geometry notebook's static calculations. Install it to generate the interactive views.

## 1. Cathode geometry comparison

### Question

At a controlled penetration height, how do pin shape and array layout change the total exposed cathode area?

The notebook compares cylinders, cones, truncated cones, square prisms, square pyramids, and truncated pyramids. Pin material volume is calculated and allowed to vary.

### Geometry model

For a cathode face of area $A_0=LW$ and $N$ identical pins:

$$A=A_0+N(S+T-B),\qquad \Delta A=A-A_0.$$

Here $S$ is one pin's sidewall area, $T$ its exposed top area, and $B$ its base footprint. Subtracting the footprint avoids counting the buried attachment face as exposed area.

The top-to-base linear size ratio $t$ describes taper: $t=0$ gives a pointed shape, $t=1$ gives a straight pin, and intermediate values give frusta. A top ratio of 0.5 gives a top area one-quarter of the base area.

For a complete rectangular grid with clear base gap $g$ and edge margin $m$:

$$n_x=\left\lfloor\frac{L-2m+g}{w+g}\right\rfloor,\quad n_y=\left\lfloor\frac{W-2m+g}{w+g}\right\rfloor,\quad N=n_xn_y.$$

Width $w$ means base diameter for round pins and base side length for square pins. All geometry lengths are in **mm**, areas in **mm²**, and volumes in **mm³**.

### Current settings in the supplied code

| Setting | Value |
| --- | --- |
| `SETUP` | `'mini'`: 20 × 1 mm face; `'large'` selects 20 × 5 mm |
| `MARGIN` | 0.05 mm |
| `H_MAIN` | 0.08 mm |
| `HEIGHTS` | `[0.05, 0.08, 1.0, 0.3]` mm |
| Width bounds | 0.030–0.20 mm |
| Gap bounds | 0.030–0.20 mm |
| Width/gap samples | 100 / 80 |
| `MAX_ASPECT_RATIO` | `None` — no aspect-ratio cap imposed |
| `T_TRUNCATED` | 0.50 |
| Reference array | 30 × 3 pins; width 0.05 mm; gap 0.05 mm |
| `COMPARTMENT_VOLUME_MM3` | `None` — actual receiving compartment volume is unspecified |

**Use the settings cell as the source of truth.** The explanatory text mentions an illustrative 0.5 mm height, but the supplied code currently uses `H_MAIN = 0.08` mm. Heights and fabrication bounds are assumed inputs, not validated manufacturing limits.

### Study guide

| Section | Comparison |
| --- | --- |
| A / A1 | Shape and continuous taper at equal height, bounding width, gap, and count |
| A2 | Shape at equal individual footprint area rather than equal bounding width |
| Physical geometry viewer | Rotatable single pins and complete reference arrays |
| B | Width–gap sweep at fixed height, using integer packing counts |
| C | Gap and count effects at fixed pin width and height |
| D | Height and width effects at fixed array and gap |
| E | Repeat packed comparisons at several heights |
| F | More pins versus wider pins for a selected shape at fixed height and gap |

Equal bounding width and equal footprint area are different comparisons. Report which is used. Likewise, distinguish added area from total area when reporting percentage advantages.

Steps in the packing plots occur when a whole row or column stops fitting. The best sampled design is the best point on the chosen grid within its assumptions, rather than a continuous or experimentally validated optimum.

### Outputs

Results are written relative to the kernel working directory:

```text
fixed_height_cathode_results/mini/
```

Selecting the large setup changes the final folder to `large`. Outputs include PNG plots, CSV comparison tables, optional self-contained Plotly HTML viewers, and `parameters.json`. Repeated runs with the same setup can overwrite files with the same names; archive results before changing settings if you need to retain them.

### Scope

The model assumes ideal rigid shapes, full gel contact on the remaining base and exposed pins, and complete rectangular grids. It does not predict adhesion, gel deformation or damage, wetting, bubbles, current distribution, electrochemical edge effects, or transport. Pin volume displaces gel, but the actual compartment volume must be supplied before interpreting displacement fractions.

## 2. Fixed-current gel transport

### Question

How do reservoir/receiver diffusivity, conductivity, and total length affect transfer, passive storage, reversal, and required voltage at fixed current?

The 1D domain contains two gel regions with constant cross-section:

- **R:** initially loaded reservoir.
- **G:** initially empty receiver.

A positive monovalent drug proxy moves by diffusion and electromigration. The outer boundaries are closed to tracer flux. There is no convection, tissue sink, preferential interface partitioning, or electrode reaction model. Concentration is represented as mass concentration in kg/m³.

### Governing relations

Tracer conservation and flux are:

$$\frac{\partial c}{\partial t}=-\frac{\partial J}{\partial x},\qquad J=-D\frac{\partial c}{\partial x}+\frac{zFD}{RT}Ec.$$

At imposed current $I$ and transport cross-section $A$:

$$j=\frac{I}{A},\qquad E_R=\frac{j}{\sigma_R},\qquad E_G=\frac{j}{\sigma_G}.$$

The gel resistance and screening voltage are:

$$R_{gel}=\frac{L_R/\sigma_R+L_G/\sigma_G}{A},\qquad V_{gel}=IR_{gel},\qquad V_{screen}=V_{gel}+V_{extra}.$$

$D$ is diffusivity (m²/s), $\sigma$ conductivity (S/m), $z$ charge number, $F$ the Faraday constant, $R$ the gas constant, and $T$ absolute temperature. The solver uses SI units internally. The area here is transport cross-section, not the geometry notebook's total exposed pin area.

The implementation uses conservative fitted intercell fluxes, cosine-spaced cells aligned with the gel interface, and implicit time stepping through a banded linear system. Conductivity is prescribed independently of tracer concentration and diffusivity.

### Current reference settings

| Input | Value |
| --- | --- |
| Current | 0.1 mA |
| Transport cross-section | 20 mm² |
| Reference total length | 2 mm |
| Reservoir fraction of length | 0.5 |
| Reference `D_R`, `D_G` | Both $10^{-10}$ m²/s |
| Reference `sigma_R`, `sigma_G` | Both 0.1 S/m |
| Initial reservoir loading | 5.9 kg/m³ |
| Temperature | 293.15 K |
| Tracer charge number | +1 |
| Forward / reversal duration | 600 s / 600 s |
| Independent passive storage | 72 h |
| Cells per gel | 200 |
| Active / storage time step | 2 s / 120 s |
| `VOLTAGE_LIMIT_V` | 1.23 V |
| `EXTRA_VOLTAGE_V` | 0 V; additional losses omitted |

The default screen contains **882 cases**: six lengths × seven reservoir diffusivities × seven receiver diffusivities × three conductivity ratios. Receiver conductivity is 0.1 S/m in this screen; reservoir conductivity is obtained from the selected ratio.

### Simulation protocols

| Protocol | Initial state | Drive |
| --- | --- | --- |
| Forward transfer | Original reservoir loading | +I for 10 min |
| Reversal | Forward endpoint | −I for 10 min |
| Zero-current hold | Same forward endpoint | Zero current for 10 min |
| Passive storage | Original reservoir loading | Zero current for 72 h |

Passive storage is a separate simulation, not a 72 h continuation of the forward run. With equal compartment volumes, passive equilibrium places 50% of the initial inventory in G.

### Reading the results

| Metric | Meaning |
| --- | --- |
| `forward_fraction` / `forward_pct` | Receiver inventory after forward drive, divided by initial total inventory; fraction / percent |
| `forward_mass_ug` | Absolute receiver mass after forward drive, in µg |
| `storage_leakage_fraction` | Receiver inventory after independent passive storage, divided by initial inventory |
| `net_return_fraction_of_forward` | `(forward receiver inventory − reversed receiver inventory) / forward receiver inventory` |
| `reversal_benefit_vs_hold` | `(held receiver inventory − reversed receiver inventory) / forward receiver inventory` |
| `V_screen_V` | Gel voltage plus the optional extra voltage |
| `voltage_pass` | Whether screening voltage is at or below the selected limit |

Initial concentration is held fixed, so increasing length also increases initial mass. Compare both fraction and absolute mass when assessing a design.

**Voltage passing is only a screen.** The 1.23 V threshold does not demonstrate electrochemical stability, prevent electrolysis, or establish a suitable dose. The model does not simulate breakdown. Cases above the limit are flagged rather than rerun with a reduced current. Extra electrode/contact voltage changes screening only; it does not alter the prescribed gel fields.

The diffusivity–conductivity contour panels use identical gels (`D_R = D_G`, `sigma_R = sigma_G`) at lengths of 2, 5, and 10 mm. They represent a slice through the full parameter space.

### Sensitivity analysis

The final one-at-a-time plots use input multipliers of 0.1, 0.3, 1, 3, and 10. Local sensitivities use equal ±10% changes near the reference:

$$S_{y,p}\approx\frac{\ln y[p(1+\epsilon)]-\ln y[p(1-\epsilon)]}{\ln(1+\epsilon)-\ln(1-\epsilon)}.$$

The sign indicates direction; magnitude compares local influence. A sensitivity of −0.5 means a small 1% input increase produces approximately a 0.5% output decrease near that reference. Only pairs passing the voltage screen are included in the displayed ranking.

In the supplied code, the final `PARAMETERS` list enables **receiver diffusivity, receiver conductivity, and total length**. The reservoir diffusivity and conductivity entries are commented out. Uncomment them to include all five inputs described in the notebook text.

### Caching and numerical checks

Embedded default screening and contour data allow the default analysis to reuse precomputed results. Matching checks control cache reuse. Set `FORCE_RECOMPUTE = True` to request fresh calculations, especially after changing solver logic. Fresh full sweeps can take substantially longer than cached runs.

The notebook contains checks for mass conservation, concentration non-negativity, passive diffusion against an analytical solution, zero-current uniform concentration, stationary drift/diffusion equilibrium, and fixed-current voltage/field consistency. It also defines selected-case refinement checks. A voltage-passing row is not evidence that every case has been mesh/time-step converged; review refinement status and rerun relevant checks for new settings.

### Outputs

With `SAVE_OUTPUTS = True`, each run creates a timestamped folder under:

```text
gel_transport_outputs/
```

Exports include:

- Figures as PNG and PDF; displayed tables as CSV and HTML.
- `all_cases_with_voltage_criterion.csv` and `voltage_passing_candidates.csv`.
- `length_comparison.csv` and `diffusivity_comparison.csv`.
- `reservoir_conductivity_comparison.csv` and `common_conductivity_comparison.csv`.
- `one_at_a_time_comparison.csv` and `local_parameter_influence.csv`.
- `design_contour_data.npz` and `settings.json`.

Run through the final export cell to write the named datasets. Disabling saving retains in-memory and notebook results without these automatic exports.

## Recommended workflow

1. Confirm measured dimensions, wetted overlap, transport cross-section, and electrical operating conditions.
2. Use geometry comparisons to select feasible pin candidates under explicit manufacturing bounds.
3. Use the 1D study to screen material properties and transport lengths at the chosen current.
4. Apply the voltage screen, then assess delivery, storage, and reversal against independently specified targets.
5. Check numerical convergence for shortlisted transport cases.
6. Validate candidates experimentally or with a coupled spatial model before drawing conclusions about a pin-equipped device.

## Troubleshooting

| Symptom | Action |
| --- | --- |
| `ModuleNotFoundError` | Install the missing package in the selected kernel; `%pip install numpy scipy pandas matplotlib plotly` can be run in a notebook cell. |
| Changes do not appear in later plots | Restart the kernel and run all cells in order. |
| Results still use embedded values after solver edits | Set `FORCE_RECOMPUTE = True` and rerun. |
| Interactive geometry is missing | Install Plotly, rerun its cells, or open the exported HTML viewer in a browser. |
| Geometry assertion says an array does not fit | Check width, gap, margin, height constraints, and row/column counts. |
| Output folder seems missing | Inspect the printed resolved output path; paths are relative to the kernel working directory. |
| Full screen is slow | Check whether changed settings or forced recomputation disabled cache reuse. |

## Reproducibility and sharing

Retain the input notebooks, exported settings, and result folder for each comparison. For an environment record, run:

```powershell
.\.venv\Scripts\python.exe -m pip freeze > requirements-lock.txt
```

This records the environment you installed; it is not an existing tested lockfile supplied with the notebooks. Review saved notebook outputs and experimental/project details before publishing the repository. No license file was supplied with these two notebooks; add the appropriate license and attribution before redistribution.
