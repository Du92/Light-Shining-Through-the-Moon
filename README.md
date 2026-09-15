# Light Shining Through the Moon

Numerical code and derived data associated with the manuscript:

**Light Shining Through the Moon: A Lunar-Occultation Helioscope for Solar Axions**

D. Suárez-Fontanella

This repository reproduces the orbital geometry, geomagnetic axion-photon conversion, B16-AGSS09 Primakoff region-of-interest fraction, and detector-level sensitivity benchmark used in the paper.

> The detector response and background used in the current sensitivity estimate are benchmark inputs. A mission-specific analysis must replace them with the actual instrument response and post-occultation background.

---

## 1. Reference setup

Baseline calculation:

- eclipse: 12 August 2026
- reference epoch: 17:45:54 UTC
- circular polar orbit
- nominal altitude: 600 km
- energy range: 3–15 keV
- solar ROI radius: 4 arcmin
- fiducial geomagnetic activity: Kp = 2
- magnetic field: IGRF + Tsyganenko T89
- line-of-sight cutoff: 15 Earth radii
- solar model: B16-AGSS09
- reference coupling: $g_{a\gamma}=10^{-10}$ GeV$^{-1}$

Reference values:

| Quantity | Value |
|---|---:|
| valid occultation interval | 470 s |
| $I_{B,min}$ | 69.147 T m |
| $I_{B,max}$ | 96.420 T m |
| $I_{B,rms}$ | 76.888 T m |
| $E_B$ | $2.778552\times10^6$ (T m)$^2$ s |
| 3–15 keV flux fraction inside 4 arcmin | 0.973734 |
| reference signal | 0.8717 counts |
| benchmark background | 0.9400 counts |
| low-mass 95% C.L. benchmark | $1.314\times10^{-10}$ GeV$^{-1}$ |

---

## 2. Installation

Create a Python environment and install:

```bash
python -m venv .venv
source .venv/bin/activate

pip install numpy scipy matplotlib pandas astropy skyfield sgp4 geopack requests jupyter
```

For reproducibility, save the final environment:

```bash
pip freeze > requirements-lock.txt
```

---

## 3. JPL ephemeris

The calculation uses JPL DE421 through Skyfield.

Documentation:

https://rhodesmill.org/skyfield/planets.html

Download once:

```bash
mkdir -p skyfield_data

python - <<'PY'
from skyfield.api import Loader
load = Loader("skyfield_data")
load("de421.bsp")
PY
```

The file must be available as:

```text
skyfield_data/de421.bsp
```

The orbit is anchored at 17:45:54 UTC, corresponding to the predicted instant of greatest eclipse rounded to the nearest second.

NASA eclipse data:

https://eclipse.gsfc.nasa.gov/SEbeselm/SEbeselm2001/SE2026Aug12Tbeselm.html

---

## 4. Orbit and occultation

The reference orbit is a circular two-body orbit used as a realizability test, not as a mission design.

$$r(t)=r_0\left[\hat r_0\cos(nt)+\hat v_0\sin(nt)\right]$$

$$n=\sqrt{\frac{GM_\oplus}{r_0^3}}$$

At every orbital sample, the code recomputes the apparent Sun and Moon positions. The solar direction is therefore not held fixed during propagation.

The adopted ROI is accepted as fully occulted when:

$$\delta_{\odot M}(t)+\theta_{ROI}<\alpha_M(t)$$

The Sun-directed ray must also remain clear of the Earth.

For the reference orbit, the valid interval is approximately:

```text
-250 s <= t - t_occ <= +220 s
```

---

## 5. Geomagnetic field

The conversion path follows the solar-axion direction:

$$\hat\ell=\frac{\mathbf r_\odot-\mathbf r_{sat}}{|\mathbf r_\odot-\mathbf r_{sat}|}$$

The magnetic field is evaluated as:

$$\mathbf B=\mathbf B_{IGRF}+\mathbf B_{T89}$$

The current implementation uses the Python `geopack` package:

https://github.com/tsssss/geopack

T89 reference:

https://doi.org/10.1016/0032-0633(89)90066-4

IGRF information:

https://www.ncei.noaa.gov/products/international-geomagnetic-reference-field

Before the final paper release, record the exact `geopack` version and confirm the IGRF coefficient generation used by that version.

---

## 6. Signed transverse components

The calculation keeps the direction and sign of the transverse magnetic field.

For a fixed basis perpendicular to the line of sight:

$$B_1(s)=\mathbf B(s)\cdot\mathbf e_1$$

$$B_2(s)=\mathbf B(s)\cdot\mathbf e_2$$

The coherent low-mass magnetic lever arm is:

$$I_B=\sqrt{\left(\int B_1\,ds\right)^2+\left(\int B_2\,ds\right)^2}$$

The physical amplitude therefore does not use:

```text
integral |B_perp| ds
```

because that would remove cancellations caused by changes in field direction.

---

## 7. Plasma model

The plasma contribution enters through:

$$\omega_{pl}=3.69\times10^{-11}\sqrt{n_e}\;{\rm eV}$$

with $n_e$ in cm$^{-3}$.

The inner plasmaspheric profile follows the Carpenter-Anderson parameterization:

$$n_{pl}(r)=10^{3.9043-0.3145r}\;{\rm cm}^{-3}$$

with the approximate plasmapause position:

$$L_{pp}=5.6-0.46K_p$$

Outside the plasmapause, the feasibility calculation uses a smooth $r^{-2}$ continuation. The transition is smoothed numerically.

Reference:

D. L. Carpenter and R. R. Anderson, J. Geophys. Res. 97, 1097–1108 (1992).

https://doi.org/10.1029/91JA01548

This is a semi-empirical feasibility model, not an event-specific plasma reconstruction.

---

## 8. Axion-photon conversion

The phase mismatch is:

$$q(s)=\frac{m_a^2-\omega_{pl}^2(s)}{2E}$$

The accumulated phase is:

$$\phi(s)=\int_0^s q(s')\,ds'$$

For the two transverse components:

$$I_i(E,m_a)=\int B_i(s)e^{i\phi(s)}\,ds$$

and the conversion probability is:

$$P_{a\to\gamma}=\frac{g_{a\gamma}^2}{4}\left(|I_1|^2+|I_2|^2\right)$$

The final run should use:

```python
FAST = False
PHASE_STEP = 0.05
```

and the result should be checked for convergence near $m_a=10^{-4}$ eV.

---

## 9. Solar Primakoff flux

The full-Sun spectrum used in the detector calculation is:

$$\frac{d\Phi_a}{dE}=6.02\times10^{10}g_{10}^2E^{2.481}e^{-E/1.205}$$

with $E$ in keV and:

$$g_{10}=\frac{g_{a\gamma}}{10^{-10}\;{\rm GeV}^{-1}}$$

The flux inside the adopted ROI is:

$$\frac{d\Phi_a^{ROI}}{dE}=f_{ROI}(E)\frac{d\Phi_a}{dE}$$

---

## 10. B16-AGSS09 and the ROI fraction

The projected Primakoff image is generated with:

https://github.com/sebhoof/SolarAxionFlux

Relevant references:

- https://arxiv.org/abs/2306.00077
- https://arxiv.org/abs/1611.09867
- https://arxiv.org/abs/2101.08789

The 4 arcmin ROI corresponds to:

$$\rho_{ROI}=\frac{4.0}{15.7779}=0.253519$$

The ROI fraction is:

$$f_{ROI}(E)=\frac{\int_0^{\rho_{ROI}}\frac{d^2\Phi_a}{dE\,d\rho}\,d\rho}{\int_0^1\frac{d^2\Phi_a}{dE\,d\rho}\,d\rho}$$

Do not use the geometrical area fraction as the signal fraction.

The surface-flux file can be stored as:

```text
solar_flux/primakoff_surface_B16_AGSS09.csv
```

with columns:

```text
E_keV,rho,d2Phi_dEdrho
```

The derived ROI file is:

```text
solar_flux/primakoff_roi_fraction_B16_AGSS09.csv
```

with columns:

```text
E_keV,f_roi
```

Reference values:

```text
f_ROI(3 keV)  = 0.91955462
f_ROI(5 keV)  = 0.98279225
f_ROI(8 keV)  = 0.99827838
f_ROI(10 keV) = 0.99962093
f_ROI(15 keV) = 0.99999223
```

The 3–15 keV spectrum-weighted value is:

$$\bar f_{ROI}=\frac{\int_3^{15}E^{2.481}e^{-E/1.205}f_{ROI}(E)\,dE}{\int_3^{15}E^{2.481}e^{-E/1.205}\,dE}=0.973734$$

---

## 11. Magnetic exposure

Along the valid occultation interval:

$$E_B=\int_{occ}I_B^2(t)\,dt$$

and:

$$I_{B,rms}=\sqrt{\frac{E_B}{\Delta t_{occ}}}$$

For the reference event:

```text
I_B,min = 69.147 T m
I_B,max = 96.420 T m
I_B,rms = 76.888 T m
```

The value 96.4 T m is the maximum instantaneous value reached in the propagated valid window. It is not the RMS value.

---

## 12. Detector benchmark

The current sensitivity calculation uses a representative focusing X-ray detector.

The reference residual background is:

```text
2.0e-3 count/s
```

inside a 4 arcmin radius aperture over 3–15 keV.

For the 470 s occultation:

```text
B = 0.9400 counts
```

These are benchmark values only. For a specific mission, use calibrated response files and the actual post-occultation background.

---

## 13. Signal and sensitivity

The expected signal at the reference coupling is:

$$S_{ref}(m_a)=\int_{occ}dt\int dE\,A_{eff}(E)\frac{d\Phi_a^{ROI}}{dE}P_{a\to\gamma}(E,m_a,t)$$

Because solar production and geomagnetic conversion each scale as $g_{a\gamma}^2$:

$$S(g)=S_{ref}\left(\frac{g}{g_{ref}}\right)^4$$

The current expected-limit benchmark uses:

$$q_A=2\left[S-B\ln\left(1+\frac{S}{B}\right)\right]$$

and solves:

$$q_A=2.71$$

The low-mass result is approximately:

$$g_{a\gamma}^{95}=1.314\times10^{-10}\;{\rm GeV}^{-1}$$

Because the expected counts are of order unity, a mission-level analysis should use an exact low-count statistical treatment.

---

## 14. Run sequence

Open:

```text
axion_moon_PRL_working_v3_consolidated.ipynb
```

Run the notebook from top to bottom.

For the final paper numbers:

```python
FAST = False
```

The notebook should reproduce the orbit, magnetic exposure, Primakoff ROI fraction, conversion probability, signal counts, and sensitivity curve.

---

## 15. Main outputs

```text
PRL_propagated_orbit_IB.csv
PRL_primakoff_roi_fraction.csv
PRL_sensitivity_data.csv
figures_PRL/
```

Before tagging the paper release, verify:

- 470 s valid occultation window
- $I_{B,rms}\approx76.888$ T m
- $I_{B,max}\approx96.420$ T m
- weighted ROI fraction = 0.973734
- high-mass numerical convergence
- exact software versions
- exact IGRF coefficients used
- detector and background assumptions clearly labelled as benchmarks

---

## 16. Code and data availability

The numerical code and derived data used to reproduce the results in the paper are available in this repository:

https://github.com/Du92/Light-Shining-Through-the-Moon

For the publication version, a tagged GitHub release should be archived with Zenodo and cited through its DOI.

---

## 17. Citation

If you use this repository, please cite the accompanying paper and the external software/model references listed above.

