# Light Shining Through the Moon

Reproducibility repository for the numerical calculations associated with the manuscript

**“Light Shining Through the Moon: A Lunar-Occultation Helioscope for Solar Axions”**

This repository contains the numerical pipeline used to evaluate solar-axion conversion in the terrestrial magnetosphere during a lunar occultation. The reference calculation propagates a circular near-600-km polar orbit through the total solar eclipse of 12 August 2026, recomputes the Sun–Moon geometry and geomagnetic line of sight along the orbit, evaluates the phase-resolved axion-photon conversion probability, folds the result with a B16-AGSS09 Primakoff solar-axion image and a representative focusing-X-ray-detector response, and derives an expected single-event sensitivity.

> **Status.** The orbital, geomagnetic, plasma, Primakoff-ROI, and conversion calculations are intended to be reproducible from this repository. The detector response and background used in the current sensitivity curve are benchmark inputs, not calibrated products for a specific mission. A mission-specific implementation must replace them with the actual instrument response and measured/modelled post-occultation background.

---

## 1. Reference calculation

> **Math rendering on GitHub:** this README uses `$...$` for inline equations and `$$...$$` for display equations. These are the delimiters supported directly by GitHub Markdown.


The baseline calculation uses

- eclipse anchor: **2026-08-12 17:45:54 UTC**, i.e. the predicted instant of greatest eclipse rounded to the nearest second;
- demonstrative circular polar orbit passing through the fiducial occultation point;
- nominal WGS84 altitude: **600 km**;
- analysis band: **3–15 keV**;
- circular solar ROI radius: **4.0 arcmin**;
- fiducial geomagnetic activity: **$K_p=2$**;
- internal geomagnetic field: IGRF implementation used by the selected `geopack` backend;
- external geomagnetic field: Tsyganenko T89;
- line-of-sight magnetic integration out to **15 Earth radii**;
- semi-empirical magnetospheric electron-density profile;
- full phase-resolved axion-photon conversion using two signed transverse magnetic-field components;
- B16-AGSS09 projected Primakoff axion image for the ROI fraction;
- reference coupling $g_{a\gamma}^\mathrm{ref}=10^{-10}\,\mathrm{GeV}^{-1}$.

For the propagated August 2026 reference event, the calculation should reproduce approximately

| Quantity | Reference value |
|---|---:|
| Valid ROI-occulted, Earth-clear interval | 470 s |
| $\mathcal I_{B,\min}$ | 69.147 T m |
| $\mathcal I_{B,\max}$ | 96.420 T m |
| $\mathcal I_{B,\rm rms}$ | 76.888 T m |
| $\mathcal E_B=\int dt\,\mathcal I_B^2$ | $2.778552\times10^6\ (\mathrm{T\,m})^2\mathrm{s}$ |
| 3–15 keV B16-AGSS09 flux fraction in 4' ROI | 0.973734 |
| $S_\mathrm{ref}$ at $g_{a\gamma}=10^{-10}\,\mathrm{GeV}^{-1}$ | 0.8717 counts |
| benchmark background | 0.9400 counts |
| expected low-mass 95% C.L. benchmark | $1.314\times10^{-10}\,\mathrm{GeV}^{-1}$ |

Small numerical differences can arise from package versions, sampling density, and the `FAST` setting.

---

## 2. Suggested repository layout

```text
.
├── README.md
├── axion_moon_PRL_working_v3_consolidated.ipynb
├── solar_flux/
│   └── primakoff_roi_fraction_B16_AGSS09.csv
├── skyfield_data/
│   └── de421.bsp
├── figures_PRL/
├── PRL_propagated_orbit_IB.csv
├── PRL_primakoff_roi_fraction.csv
└── PRL_sensitivity_data.csv
```

The notebook can run with the embedded 121-point B16-AGSS09 ROI table even if the external ROI CSV is absent. For independent validation, regenerate that table from `SolarAxionFlux` as described below.

The JPL ephemeris file does not need to be committed to Git. It can be downloaded once into `skyfield_data/`.

---

## 3. Python environment

The current notebook uses

```bash
python -m venv .venv
source .venv/bin/activate

pip install \
    numpy \
    scipy \
    matplotlib \
    pandas \
    astropy \
    skyfield \
    sgp4 \
    geopack \
    requests \
    jupyter
```

The notebook imports the geomagnetic package through

```python
from geopack import geopack, t89
```

which corresponds to the Python `geopack` implementation by Sheng Tian et al.:

- https://github.com/tsssss/geopack
- https://pypi.org/project/geopack/

For a paper release, pin the exact tested environment, for example

```bash
pip freeze > requirements-lock.txt
```

and archive that file with the release.

### Important IGRF reproducibility note

The manuscript should identify the exact IGRF generation actually used by the tagged code release. IGRF-14 is the current IAGA/NOAA generation:

- https://www.ncei.noaa.gov/products/international-geomagnetic-reference-field

The notebook currently calls `geopack.igrf_gsm(...)`. Before creating the final paper release, verify and document the exact IGRF coefficient file/version loaded by the pinned `geopack` version. Do not rely only on the package name when stating “IGRF-14” in the paper.

---

## 4. JPL ephemeris

The calculation uses the JPL DE421 ephemeris through Skyfield.

Skyfield documentation:

- https://rhodesmill.org/skyfield/planets.html

A convenient one-time download is

```bash
mkdir -p skyfield_data

python - <<'PY'
from skyfield.api import Loader
load = Loader("skyfield_data")
load("de421.bsp")
print("DE421 downloaded to skyfield_data/")
PY
```

The scientific calculation then loads the local file

```python
EPHEMERIS_PATH = Path("skyfield_data/de421.bsp")
eph = load_file(str(EPHEMERIS_PATH))
earth, sun, moon = eph["earth"], eph["sun"], eph["moon"]
```

This avoids silently changing ephemerides between runs.

---

## 5. Why 17:45:54 UTC?

The orbit is anchored at

```text
2026-08-12 17:45:54 UTC
```

because the NASA/GSFC eclipse tables give the instant of greatest eclipse as approximately 17:45:53.8 UT. The value used in the notebook is this epoch rounded to the nearest second.

NASA/GSFC reference:

- https://eclipse.gsfc.nasa.gov/SEbeselm/SEbeselm2001/SE2026Aug12Tbeselm.html
- https://eclipse.gsfc.nasa.gov/SEpath/SEpath2001/SE2026Aug12Tpath.html

The eclipse epoch is an **ephemeris anchor**, not a magnetic optimization.

At this epoch, the code solves for a 600-km observer position that minimizes the apparent Sun–Moon separation. The propagated polar orbit is then constructed through that fiducial position.

---

## 6. Orbital propagation and occultation selection

The baseline orbit is deliberately simple: a circular two-body orbit used as a realizability test, not as a mission design.

For a circular orbit,

$$
\mathbf r(t)=r_\mathrm{orb}
\left[
\hat{\mathbf r}_0\cos(nt)+
\hat{\mathbf v}_0\sin(nt)
\right],
\qquad
n=\sqrt{\frac{GM_\oplus}{r_\mathrm{orb}^3}} .
$$

The reference orbit has approximately

```text
geocentric radius = 6961.845 km
speed             = 7.5667 km/s
period            = 96.349 min
inclination       = 90 deg
```

### The Sun is not held fixed

At **every propagated orbital sample**, the code recomputes the apparent Sun and Moon positions from the ephemeris. Therefore the solar direction is not frozen relative to the orbital plane.

A sample is accepted only if both conditions are satisfied:

$$
\delta_{\odot M}(t)+\theta_\mathrm{ROI}<\alpha_M(t)
$$

and the Sun-directed ray from the spacecraft does not intersect Earth.

Here

- $\delta_{\odot M}$ is the apparent Sun–Moon center separation;
- $\alpha_M$ is the apparent lunar angular radius;
- $\theta_\mathrm{ROI}=4'$.

For the reference event, the valid interval is approximately

```text
t - t_occ = -250 s ... +220 s
```

or about **470 s**.

---

## 7. Geomagnetic field

The magnetic field is evaluated along the **Sun-directed line of sight**, not along a Moon-directed ray.

At each position,

$$
\hat{\ell}
=
\frac{\mathbf r_\odot-\mathbf r_\mathrm{sat}}
{|\mathbf r_\odot-\mathbf r_\mathrm{sat}|}.
$$

The Moon only selects which ordinary solar photons are blocked.

The line of sight is transformed to GSM coordinates and sampled from the spacecraft to a default outer radius of

$$
r_{\max}=15\,R_\oplus .
$$

The field is evaluated as

$$
\mathbf B_\mathrm{tot}
=
\mathbf B_\mathrm{IGRF}+\mathbf B_\mathrm{T89}.
$$

Code structure:

```python
ps = geopack.recalc(unix_time)
iopt = kp_to_t89_iopt(Kp)

bi = np.array(geopack.igrf_gsm(x, y, z))
be = np.array(t89.t89(iopt, ps, x, y, z))

B = (bi + be) * 1e-9   # nT -> T
```

T89 reference:

N. A. Tsyganenko, *A magnetospheric magnetic field model with a warped tail current sheet*, Planetary and Space Science **37**, 5–20 (1989).

DOI: https://doi.org/10.1016/0032-0633(89)90066-4

Python implementation:

- https://github.com/tsssss/geopack

### T89 activity parameter

The notebook uses

```python
KP_FID = 2
KP_SCAN = [0, 2, 4]
```

with the package's discrete T89 activity option obtained from the adopted $K_p$.

For publication-level systematics, repeat the calculation over the chosen activity range and/or compare with more modern external-field models.

---

## 8. Signed transverse magnetic field

A central point of the calculation is that the oscillation amplitude does **not** use

$$
\int |\mathbf B_\perp|\,ds .
$$

For each line of sight a fixed orthonormal basis
$(\mathbf e_1,\mathbf e_2,\hat\ell)$ is constructed. The signed transverse components are

$$
B_1(s)=\mathbf B(s)\cdot\mathbf e_1,
\qquad
B_2(s)=\mathbf B(s)\cdot\mathbf e_2.
$$

The coherent low-mass lever arm is

$$
\mathcal I_B=
\sqrt{
\left(\int B_1\,ds\right)^2+
\left(\int B_2\,ds\right)^2
}.
$$

This preserves cancellations when the field changes orientation along the ray.

The positive quantity

$$
|\mathbf B_\perp|=\sqrt{B_1^2+B_2^2}
$$

is retained only for diagnostics and plots.

---

## 9. Magnetospheric plasma model

The phase mismatch contains the local plasma frequency,

$$
q(s)=\frac{m_a^2-\omega_\mathrm{pl}^2(s)}{2E},
$$

with

$$
\omega_\mathrm{pl}= 3.69\times10^{-11} \sqrt{\frac{n_e}{\mathrm{cm}^{-3}}} \ \mathrm{eV}.
$$

The feasibility calculation uses a smooth semi-empirical density profile. Writing $r$ in Earth radii,

$$
L_\mathrm{pp}=\operatorname{clip}(5.6-0.46K_p,\,2.5,\,6.0),
$$

$$
n_\mathrm{pl}(r)
=
10^{3.9043-0.3145r}
\ \mathrm{cm}^{-3},
$$

$$
n_\mathrm{out}(r)
=
5
\left[
\frac{\max(r,1)}{10}
\right]^{-2}
\ \mathrm{cm}^{-3},
$$

and

$$
w(r)=
\left[
1+\exp\left(\frac{r-L_\mathrm{pp}}{0.18}\right)
\right]^{-1}.
$$

The adopted density is

$$
n_e(r)
=
\operatorname{clip}
\left[
w\,n_\mathrm{pl}+
(1-w)\,n_\mathrm{out},
10^{-4},10^5
\right]
\ \mathrm{cm}^{-3}.
$$

The inner plasmaspheric density and $K_p$-dependent plasmapause prescription are based on the Carpenter–Anderson empirical model. The outer $r^{-2}$ branch is a simple phenomenological continuation used in this feasibility calculation.

Reference:

D. L. Carpenter and R. R. Anderson, *An ISEE/Whistler Model of Equatorial Electron Density in the Magnetosphere*, J. Geophys. Res. **97**, 1097–1108 (1992).

DOI: https://doi.org/10.1029/91JA01548

This is not an event-specific plasma reconstruction.

---

## 10. Phase-resolved axion-photon conversion

For each transverse component,

$$
\mathcal I_i(E,m_a)
=
\int ds\,
B_i(s)
e^{i\phi(s)},
\qquad i=1,2,
$$

where

$$
\phi(s)
=
\int_0^s ds'\,
\frac{m_a^2-\omega_\mathrm{pl}^2(s')}{2E}.
$$

The conversion probability is

$$
P_{a\rightarrow\gamma}
=
\frac{g_{a\gamma}^2}{4}
\left(
|\mathcal I_1|^2+
|\mathcal I_2|^2
\right).
$$

The implementation converts

```text
1 T = 195 eV^2
1 m = 5.0677307e6 eV^-1
```

so that

$$
1\ \mathrm{T\,m}\simeq0.9882\ \mathrm{GeV}.
$$

The oscillatory integral is adaptively refined by requiring a maximum local/total phase step. The final publication run should use

```python
FAST = False
PHASE_STEP = 0.05
```

and verify convergence near the upper mass edge $m_a=10^{-4}$ eV.

---

# 11. Solar Primakoff flux

The full-Sun Primakoff spectrum used for the detector-level benchmark is

$$
\frac{d\Phi_a}{dE}
=
6.02\times10^{10}
g_{10}^2
E^{2.481}
e^{-E/1.205}
\ \mathrm{cm}^{-2}\,\mathrm{s}^{-1}\,\mathrm{keV}^{-1},
$$

where $E$ is in keV and

$$
g_{10}=
\frac{g_{a\gamma}}
{10^{-10}\ \mathrm{GeV}^{-1}}.
$$

The flux in the analysis aperture is

$$
\frac{d\Phi_a^\mathrm{ROI}}{dE}
=
f_\mathrm{ROI}(E)
\frac{d\Phi_a}{dE}.
$$

The important point is that $f_\mathrm{ROI}(E)$ is computed from the **projected Primakoff axion image**, not from a uniform solar disc.

---

# 12. B16-AGSS09 and computation of $f_\mathrm{ROI}(E)$

## 12.1 External resources

The projected Primakoff image is generated using the public `SolarAxionFlux` project:

- Repository: https://github.com/sebhoof/SolarAxionFlux
- General solar-axion-flux paper requested by the project: https://arxiv.org/abs/2101.08789
- Spatial/spectral axion-image formalism: https://arxiv.org/abs/2306.00077
- B16 standard solar models: https://arxiv.org/abs/1611.09867

`SolarAxionFlux` explicitly lists B16-AGSS09 as one of the supported solar models and asks users to cite the B16 standard-solar-model reference when it is used.

### License/provenance

Do not copy external `SolarAxionFlux` routines into this repository unless their license allows it. In particular, consult the upstream `LICENSE` and README before redistributing code from its `python/` utilities. This repository only needs the derived numerical surface-flux or ROI-fraction table.

---

## 12.2 Install SolarAxionFlux

On Ubuntu/Debian:

```bash
sudo apt-get update
sudo apt-get install -y build-essential cmake libgsl-dev

git clone https://github.com/sebhoof/SolarAxionFlux.git
cd SolarAxionFlux

mkdir build
cd build
cmake ..
make -j
cd ..
```

The upstream repository contains `examples.ipynb` and the Python interface. Follow the current upstream instructions because function names and build options can change.

A basic Python-module check, following the upstream README, is

```python
from lib import pyaxionflux as afl
afl.test_module()
```

when the repository/build layout is on the Python path.

---

## 12.3 Quantity that must be exported

For this analysis, generate the B16-AGSS09 **Primakoff** projected image as a grid in energy and projected solar radius and export

```text
solar_flux/primakoff_surface_B16_AGSS09.csv
```

with columns

```text
E_keV,rho,d2Phi_dEdrho
```

where

- `E_keV` is the axion energy in keV;
- `rho` is projected solar radius in units of $R_\odot$, $0\le\rho\le1$;
- `d2Phi_dEdrho` is

$$
\frac{d^2\Phi_a^\mathrm{P}}{dE\,d\rho}.
$$

Hoof, Jaeckel & Thormaehlen give

$$
\frac{d^2\Phi^\mathrm{P}}{dE_a\,d\rho} = \frac{R_\odot^3E_a^2}{2\pi^2d_E^2} \int_\rho^1dr\, \frac{\rho r}{\sqr  {r^2-\rho^2}} \Gamma^\mathrm{P}(E_a,r).
$$

See Eq. (2.3) of:

https://arxiv.org/abs/2306.00077

### Important integration convention

The quantity $d^2\Phi/(dE\,d\rho)$ already contains the radial Jacobian implied by the projected flux definition. Therefore, when constructing the cumulative flux fraction below, **do not multiply it by an additional factor $2\pi\rho$**.

The paper's Fig. 1 often displays
$(1/\rho)\,d^2\Phi/(dE\,d\rho)$ for visualization. That is not the quantity to integrate directly unless the corresponding $\rho$ factor is restored.

---

## 12.4 Convert the 4' aperture into projected solar radius

At the fiducial epoch, the apparent solar angular radius is

$$
\alpha_\odot=15.7779'.
$$

For

$$
\theta_\mathrm{ROI}=4.0',
$$

the projected radial aperture is

$$
\rho_\mathrm{ROI}
=
\frac{\theta_\mathrm{ROI}}{\alpha_\odot}
=
0.253519.
$$

The same angular aperture covers only

$$
\rho_\mathrm{ROI}^2
\simeq0.0643
$$

or 6.4% of the apparent solar-disc area, but because the Primakoff image is centrally concentrated it contains about 97.4% of the 3–15 keV signal.

---

## 12.5 Definition of $f_\mathrm{ROI}(E)$

For every energy,

$$
f_\mathrm{ROI}(E)
=
\frac{
\displaystyle
\int_0^{\rho_\mathrm{ROI}}
d\rho\,
\frac{d^2\Phi_a^\mathrm{P}}{dE\,d\rho}
}{
\displaystyle
\int_0^1
d\rho\,
\frac{d^2\Phi_a^\mathrm{P}}{dE\,d\rho}
}.
$$

This is the quantity used in the detector-level flux:

$$
\frac{d\Phi_a^\mathrm{ROI}}{dE}
=
f_\mathrm{ROI}(E)\frac{d\Phi_a}{dE}.
$$

---

## 12.6 Minimal script for generating the ROI-fraction CSV

Once the surface-flux CSV has been exported from `SolarAxionFlux`, the following independent script constructs the table used by the notebook.

```python
from pathlib import Path

import numpy as np
import pandas as pd
from scipy.integrate import trapezoid

SURFACE = Path("solar_flux/primakoff_surface_B16_AGSS09.csv")
OUTPUT = Path("solar_flux/primakoff_roi_fraction_B16_AGSS09.csv")

THETA_ROI_ARCMIN = 4.0
ALPHA_SUN_ARCMIN = 15.7779
RHO_ROI = THETA_ROI_ARCMIN / ALPHA_SUN_ARCMIN


def integrate_interval(rho, y, a, b):
    rho = np.asarray(rho, float)
    y = np.asarray(y, float)

    order = np.argsort(rho)
    rho = rho[order]
    y = y[order]

    if rho[0] > a or rho[-1] < b:
        raise ValueError(
            f"rho grid must cover [{a}, {b}], got [{rho[0]}, {rho[-1]}]"
        )

    inside = (rho > a) & (rho < b)
    rr = np.r_[a, rho[inside], b]
    yy = np.interp(rr, rho, y)

    return trapezoid(yy, rr)


surface = pd.read_csv(SURFACE)

required = {"E_keV", "rho", "d2Phi_dEdrho"}
if not required.issubset(surface.columns):
    raise ValueError(f"Surface table must contain {required}")

rows = []

for E, block in surface.groupby("E_keV", sort=True):
    rho = block["rho"].to_numpy(float)
    flux_rho = block["d2Phi_dEdrho"].to_numpy(float)

    total = integrate_interval(rho, flux_rho, 0.0, 1.0)
    roi = integrate_interval(rho, flux_rho, 0.0, RHO_ROI)

    if total <= 0:
        raise ValueError(f"Non-positive total projected flux at E={E} keV")

    rows.append({
        "E_keV": float(E),
        "f_roi": float(roi / total),
    })

out = pd.DataFrame(rows)
OUTPUT.parent.mkdir(exist_ok=True)
out.to_csv(OUTPUT, index=False)

print("rho_ROI =", RHO_ROI)
print(out.head())
print("written:", OUTPUT)
```

The output file must contain

```text
E_keV,f_roi
```

and cover at least 3–15 keV.

---

## 12.7 Validation values

For the table used in the reference calculation,

```text
f_ROI(3 keV)   = 0.91955462
f_ROI(5 keV)   = 0.98279225
f_ROI(8 keV)   = 0.99827838
f_ROI(10 keV)  = 0.99962093
f_ROI(15 keV)  = 0.99999223
```

The Primakoff-spectrum-weighted fraction over 3–15 keV is

$$
\bar f_\mathrm{ROI}^{3-15}
=
\frac{
\int_{3}^{15}dE\,
E^{2.481}e^{-E/1.205}
f_\mathrm{ROI}(E)
}{
\int_{3}^{15}dE\,
E^{2.481}e^{-E/1.205}
}
=
0.973734.
$$

A compact validation script is

```python
from scipy.integrate import trapezoid
import numpy as np

E = np.linspace(3.0, 15.0, 1001)
shape = E**2.481 * np.exp(-E / 1.205)

weighted_fraction = (
    trapezoid(shape * primakoff_roi_fraction(E), E)
    / trapezoid(shape, E)
)

print(weighted_fraction)
# expected: ~0.973734
```

Do **not** use the uniform-disc area fraction $0.0643$ as the Primakoff signal fraction.

---

## 13. Magnetic exposure along the physical orbit

For every accepted time sample, the code computes

$$
\mathcal I_B(t)
=
\sqrt{
\left[\int ds\,B_1(s,t)\right]^2
+
\left[\int ds\,B_2(s,t)\right]^2
}.
$$

Because the coherent low-mass rate scales as $\mathcal I_B^2$, the relevant accumulated quantity is

$$
\mathcal E_B
=
\int_\mathrm{occ}dt\,
\mathcal I_B^2(t).
$$

The corresponding effective RMS lever arm is

$$
\mathcal I_{B,\rm rms}
=
\sqrt{
\frac{\mathcal E_B}{\Delta t_\mathrm{occ}}
}.
$$

For the reference propagated event,

```text
I_B,min  = 69.147 T m
I_B,max  = 96.420 T m
I_B,rms  = 76.888 T m
E_B      = 2.778552e6 (T m)^2 s
```

The value **96.4 T m is the maximum reached along the propagated valid window**. It is not the measured RMS value of the event. If it is used in an optimization study as a hypothetical sustained effective lever arm, this must be stated explicitly.

The older value **131.4 T m is not part of the final propagated reference pipeline and should not be used as a demonstrated orbital RMS benchmark**.

---

## 14. Detector benchmark

The current notebook uses a representative focusing-X-ray-detector response.

The proxy effective area is defined by interpolation through

```text
E [keV]     3   4   5   6   8   10   12   15   20   30   50   78.4
A [cm^2]  100 350 600 750 850  900  820  700  560  380  180    5
```

The reference residual background is

```text
2.0e-3 count/s
```

inside a circular 4'-radius aperture over 3–15 keV.

For the 470-s physical occultation window this corresponds to approximately

$$
B=0.94
$$

expected counts.

These values are **benchmarks only**. For a named mission, replace them with the actual ARF/RMF/background products and account for the mission-specific pointing, thermal, stray-light, particle, Earth-albedo, and lunar-background environment.

The notebook already supports replacing the effective-area proxy with ARF files through

```python
RESPONSE_MODE = "arf"
ARF_PATHS = ["instrument_A.arf", "instrument_B.arf"]
```

The actual background model must likewise be supplied for a mission-specific forecast.

---

## 15. Signal calculation

For every axion mass,

$$
S_\mathrm{ref}(m_a)
=
\int_\mathrm{occ}dt
\int_{3\,\mathrm{keV}}^{15\,\mathrm{keV}}dE\,
A_\mathrm{eff}(E)
\frac{d\Phi_a^\mathrm{ROI}}{dE}
P_{a\rightarrow\gamma}
(E,m_a,t;g_\mathrm{ref}).
$$

Both solar Primakoff production and geomagnetic regeneration scale as $g_{a\gamma}^2$. Therefore

$$
S(g)
=
S_\mathrm{ref}
\left(
\frac{g}{g_\mathrm{ref}}
\right)^4.
$$

At low mass, the reference propagated event gives approximately

```text
S_ref = 0.8717 counts
B     = 0.9400 counts
```

for

$$
g_\mathrm{ref}=10^{-10}\ \mathrm{GeV}^{-1}.
$$

---

## 16. Expected 95% C.L. benchmark

The current sensitivity figure uses the background-only Asimov statistic

$$
q_{\mu,A}
=
2
\left[
S-
B\ln\left(1+\frac{S}{B}\right)
\right]
$$

and solves

$$
q_{\mu,A}=2.71.
$$

The resulting low-mass benchmark is

$$
g_{a\gamma}^{95}
\simeq
1.314\times10^{-10}
\ \mathrm{GeV}^{-1}.
$$

Because the expected signal and background counts are of order unity, this Asimov construction is a convenient feasibility benchmark, not the final low-count coverage prescription. A mission-level analysis should use an exact Poisson/profile-likelihood construction with the actual spatial, temporal, spectral, and background information.

---

## 17. Recommended run sequence

Open

```text
axion_moon_PRL_working_v3_consolidated.ipynb
```

and run the cells in order.

The logical sequence is:

1. install/import dependencies;
2. load the local DE421 ephemeris;
3. set the 12 August 2026 greatest-eclipse anchor;
4. solve the 600-km fiducial occultation position;
5. construct the Sun-directed line of sight;
6. evaluate IGRF + T89 in GSM coordinates;
7. construct the two signed transverse components $B_1,B_2$;
8. propagate the circular polar orbit;
9. recompute Sun–Moon geometry and Earth clearance at every sample;
10. compute $\mathcal I_B(t)$, $\mathcal E_B$, and $\mathcal I_{B,\rm rms}$;
11. evaluate the plasma profile;
12. evaluate the phase-resolved conversion probability;
13. load/regenerate the B16-AGSS09 $f_\mathrm{ROI}(E)$ table;
14. fold the ROI-restricted solar flux with the detector response;
15. integrate signal and background over the physical orbit;
16. derive the expected sensitivity;
17. export machine-readable CSVs and figures.

For final publication numbers, set

```python
FAST = False
```

before rerunning the entire notebook.

---

## 18. Machine-readable outputs

The notebook exports, where available,

```text
PRL_propagated_orbit_IB.csv
PRL_primakoff_roi_fraction.csv
PRL_sensitivity_data.csv
```

and figures in

```text
figures_PRL/
```

The most useful CSV columns are:

### `PRL_propagated_orbit_IB.csv`

- `dt_s`
- `IB1_Tm`
- `IB2_Tm`
- `IB_Tm`
- `sep_arcmin`
- `moon_radius_arcmin`
- `roi_occulted`
- `earth_clear`
- `radius_km`

### `PRL_primakoff_roi_fraction.csv`

- `E_keV`
- `f_roi`

### `PRL_sensitivity_data.csv`

- `ma_eV`
- `Sref_counts_at_g1e-10`
- `g95_GeVinv`

---

## 19. Reproducibility checks before a paper release

Before tagging the version associated with arXiv/PRL:

- run the complete notebook with `FAST=False`;
- record Python and package versions;
- pin the exact `geopack` version;
- verify the exact IGRF coefficient generation used by that version;
- archive the B16-AGSS09 ROI-fraction CSV;
- retain the provenance or generation instructions for the SolarAxionFlux surface table;
- verify that $f_\mathrm{ROI}(E)$ matches the validation values above;
- verify the 470-s valid occultation window;
- verify $\mathcal I_{B,\rm rms}\approx76.888\ \mathrm{T\,m}$;
- verify $\mathcal I_{B,\max}\approx96.420\ \mathrm{T\,m}$;
- verify the anchor-state magnetic calculation agrees with the independently constructed fiducial geometry;
- repeat the phase-integration convergence test at $m_a=10^{-4}$ eV;
- document $K_p$ and plasma-density systematics;
- keep shell/geometric-envelope optimizations separate from propagated-orbit results;
- do not describe the 96.4 T m instantaneous maximum as an observed RMS;
- do not reintroduce 131.4 T m as a demonstrated propagated-orbit benchmark;
- if a specific spacecraft is named, replace the detector/background proxy with calibrated mission products.

---

## 20. Known modelling limitations

The reference calculation should be interpreted as a feasibility demonstration.

- The orbit is a circular two-body polar orbit and does not include a full mission-design treatment of $J_2$, secular precession, station keeping, pointing, or thermal constraints.
- T89 is a large-scale empirical external-field model, not an event-specific reconstruction of the 2026 magnetosphere.
- The plasma profile is semi-empirical rather than event-specific.
- The outer density continuation is phenomenological.
- The detector response and residual background are representative benchmarks.
- The current expected counts are low enough that exact low-count inference is preferable for a final mission forecast.

Potential future improvements include elliptical-orbit optimization, modern external geomagnetic models, event-conditioned solar-wind/magnetospheric inputs, and mission-specific X-ray backgrounds.

The correct orbital optimization target is

$$
\mathcal E_B
=
\int_\mathrm{occ}
dt\,\mathcal I_B^2(t),
$$

subject simultaneously to lunar occultation, Earth-clear lines of sight, instrument constraints, and background.

---

## 21. Core external references and software

### Solar axions and B16-AGSS09

- SolarAxionFlux: https://github.com/sebhoof/SolarAxionFlux
- Hoof, Jaeckel & Thormaehlen, *Axion Helioscopes as Solar Thermometers*: https://arxiv.org/abs/2306.00077
- Vinyoles et al., *A new Generation of Standard Solar Models*: https://arxiv.org/abs/1611.09867
- SolarAxionFlux reference requested by the upstream project: https://arxiv.org/abs/2101.08789

### Ephemerides

- Skyfield JPL ephemerides: https://rhodesmill.org/skyfield/planets.html
- JPL/NAIF generic planetary SPK kernels: https://naif.jpl.nasa.gov/pub/naif/generic_kernels/spk/planets/

### Eclipse geometry

- NASA/GSFC Besselian elements, 12 Aug 2026:
  https://eclipse.gsfc.nasa.gov/SEbeselm/SEbeselm2001/SE2026Aug12Tbeselm.html

### Geomagnetic field

- IGRF / NOAA-NCEI:
  https://www.ncei.noaa.gov/products/international-geomagnetic-reference-field
- Python `geopack`:
  https://github.com/tsssss/geopack
- Tsyganenko T89:
  https://doi.org/10.1016/0032-0633(89)90066-4

### Plasma model

- Carpenter & Anderson (1992):
  https://doi.org/10.1029/91JA01548

---

## 22. Citation

If you use this repository, please cite the accompanying paper once the arXiv/DOI identifiers are available.

```bibtex
@article{SuarezFontanella_LightShiningThroughMoon,
  author = {Suárez-Fontanella, D.},
  title  = {Light Shining Through the Moon: A Lunar-Occultation Helioscope for Solar Axions},
  year   = {2026},
  note   = {arXiv identifier to be added}
}
```

When using the B16-AGSS09 spatial Primakoff calculation, also cite the references requested by the upstream `SolarAxionFlux` project, in particular the solar-flux calculation and B16 standard-solar-model papers.

---

## 23. Data and code availability statement

A concise statement suitable for the manuscript is:

> The numerical code and derived input tables required to reproduce the orbital geometry, geomagnetic axion-photon conversion, Primakoff ROI fraction, and sensitivity benchmarks are publicly available in the accompanying repository. External ephemeris and solar-model inputs are linked and documented in the repository.

For the archival paper version, create a tagged GitHub release and archive it with Zenodo so that the manuscript can cite an immutable DOI.

---

## 24. License

Choose and add an explicit repository license before public release. The license of this repository does not override the licenses of external packages or datasets.

In particular, `SolarAxionFlux` is an external project and should be cited and used according to its own license and redistribution conditions.
