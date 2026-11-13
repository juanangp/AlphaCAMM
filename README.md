# AlphaCAMM

Simulation and analysis of AlphaCAMM, the Micromegas TPC measuring the alpha emission of material samples,
with [rest-for-physics-toolkit](https://github.com/rest-for-physics-toolkit/rest-framework), the new version
of REST-for-Physics. It replaces the rml files of [TREX-DM/AlphaCAMM](https://github.com/TREX-DM/AlphaCAMM)
(legacy REST-for-Physics), with the same layout without submodules: `geometry/`, `simulations/` and
`analysis/` (the readout is in rest-common-readouts).

| File | Content |
|---|---|
| `geometry/setup.gdml` | Geometry with the sample (`geometry/currentSample*.xml`). |
| `geometry/setup_noSample.gdml` | Geometry without sample (calibrations). |
| `geometry/dimensions.xml` | Dimensions of the detector, common to both. |
| `geometry/geometry.gdml` | Vessel, gas, cathode and readout. |
| `geometry/samples/` | The samples, two files each (shape and material, placement), e.g. `viton.xml` and `viton_placement.xml`; `none.xml` without sample. |
| `geometry/currentSample.xml`, `currentSample_placement.xml` | Symbolic links to the files of the sample simulated (see Samples). |
| `simulations/Simulation.yaml` | restG4 simulation: single decay of `--set isotope=...` (e.g. Po210) at `--set source=...` (where it decays). |
| `simulations/Geant4ToRaw.yaml` | From the Geant4 events to raw signals, as the data (diffusion, shaping, electronics, noise). |
| `analysis/Analysis.yaml` | Alpha track analysis (track linearization and line analysis) of raw signals: data or simulation. |
| `analysis/DAQViewer.yaml` | Event display of the data files of femclient (restDAQ). |
| `analysis/plots/Plots.yaml` | Plots of the track analysis: origin, end, spectra, length, angle and rate (simulation or data). |
| `analysis/plots/CompareSim.yaml` | Reconstructed tracks of a simulation compared with the Geant4 truth. |

The materials, the transport tables of the gas (`ArIso.root`, medium `Ar99_iC4H10_1`) and the readout
(`AlphaCAMM.root`, readout `AlphaCAMM_Readout`: 256 + 256 strips, 0.96 mm pitch) are those of
[rest-common-materials](https://github.com/rest-for-physics-toolkit/rest-common-materials), [rest-common-transport](https://github.com/rest-for-physics-toolkit/rest-common-transport)
and [rest-common-readouts](https://github.com/rest-for-physics-toolkit/rest-common-readouts) v1.0, read by URL (downloaded once
and cached in `$REST_HOME/.rest/download`).

The geometry is in the detector frame: the readout plane at z = 0 and the cathode at z = 90 mm, the
electrons drifting towards -z. In the laboratory the Micromegas is on top, so +z is downwards (`Downwards`
and `Angle` of the line analysis refer to the drift axis).

## Chain

The commands below are launched from the directory of the outputs, e.g. the parent of this repository (the
paths of the geometry are relative to the YAML, the outputs to the launch directory). The `.root` and `.log`
files are not committed (`.gitignore`).

Simulation (raw signals, as the data):

```bash
restG4 AlphaCAMM/simulations/Simulation.yaml 2>&1 | tee sim.log          # Run00001_Po210_sample_AlphaCAMM.root
restManager --c AlphaCAMM/simulations/Geant4ToRaw.yaml --i Run00001_Po210_sample_AlphaCAMM.root \
    2>&1 | tee raw.log                                                    # R00001_RAW_Po210_sample_AlphaCAMM.root
```

Analysis and plots, the same for the simulation and the data (the files of femclient):

```bash
restManager --c AlphaCAMM/analysis/Analysis.yaml --i R00001_RAW_Po210_sample_AlphaCAMM.root \
    2>&1 | tee ana.log                                                    # R00001_ANA_Po210_sample_AlphaCAMM.root
restManager --c AlphaCAMM/analysis/Analysis.yaml --i "Run00012_<tag>_AlphaCAMM_background_*.root" \
    2>&1 | tee ana_data.log                                               # data: all the subruns of a run
restManager --c AlphaCAMM/analysis/plots/Plots.yaml --i R00001_ANA_Po210_sample_AlphaCAMM.root
restManager --c AlphaCAMM/analysis/plots/CompareSim.yaml --i R00001_ANA_Po210_sample_AlphaCAMM.root   # simulation only
```

The analysis copies the observables of its input to its analysis tree: those of the Geant4 truth of a
simulation (`g4Ana_*`), compared with the reconstruction by `CompareSim.yaml`.

Tuned parameters, to be tuned together (a change of one needs the others again):

| Parameter | Value | Where | Tuned with |
|---|---|---|---|
| `shapingTime` (shaperSin) | 200 ns | `signalShaping` of `simulations/Geant4ToRaw.yaml` | the pulses of the data |
| `recoDriftVelocity` | 16.5 mm/us | parameter of `analysis/Analysis.yaml` (`signalToHits`) | `CompareSim.yaml`: the narrowest `AngleDiff` centred at 0 |
| `intWindow` | 80 ns | parameter of `analysis/Analysis.yaml` (`signalToHits`) | Z step of the hits (`intWindow` * `recoDriftVelocity` = 1.3 mm) between the strip pitch (0.96 mm) and `clusterDistance` (3 mm) |

The reconstruction velocity is lower than that of the gas (21.7 mm/us at 140 V/cm, used by the simulation):
each strip gives hits along its whole shaped pulse, a false extent in Z that tilts the horizontal tracks.
Both pipelines store every event of the chain (`outputEventStorage` and `inputEventStorage`), for the event
display; with `false` the files keep only the last event and are much smaller.

Event display of the data, while it is taken or afterwards: `restRoot` and
`REST_DAQViewer("AlphaCAMM/analysis/DAQViewer.yaml", "Run00012_..._000.root")`.

## Sources

`simulations/Simulation.yaml` has two independent parameters, in its `parameters` section (the commands of
this section are written as launched from `simulations/`; from the parent of the repository the file is
`AlphaCAMM/simulations/Simulation.yaml`):

```yaml
parameters:
  isotope: "Po210"   # the isotope that decays (Geant4 name of the ion)
  source: "sample"   # where it decays, and the geometry
```

`--set name=value` changes one of them for that execution only (the file is not modified). It can be given
several times, one per parameter; a parameter not given keeps its value of the file. The name must be one of
`parameters` and `source` one of the table below, otherwise restG4 stops with an error. The run tag, and so
the output file name, is `<isotope>_<source>`:

| Command | isotope | source | Output |
|---|---|---|---|
| `restG4 Simulation.yaml` | Po210 | sample | `Run00001_Po210_sample_AlphaCAMM.root` |
| `restG4 Simulation.yaml --set isotope=Po218` | Po218 | sample | `Run00001_Po218_sample_AlphaCAMM.root` |
| `restG4 Simulation.yaml --set source=gas` | Po210 | gas | `Run00001_Po210_gas_AlphaCAMM.root` |
| `restG4 Simulation.yaml --set isotope=Rn222 --set source=gas` | Rn222 | gas | `Run00001_Rn222_gas_AlphaCAMM.root` |
| `restG4 Simulation.yaml --set isotope=Am241 --set source=calibration` | Am241 | calibration | `Run00001_Am241_calibration_AlphaCAMM.root` |
| `restG4 Simulation.yaml --set isotope=Po210 --set source=calibration` | Po210 | calibration | `Run00001_Po210_calibration_AlphaCAMM.root` |
| `restG4 Simulation.yaml --set source=cathode` | Po210 | cathode | `Run00001_Po210_cathode_AlphaCAMM.root` |

`-n` changes the number of events (`nEvents`, 10000 in the file), e.g.
`restG4 Simulation.yaml --set isotope=Am241 --set source=calibration -n 100000 2>&1 | tee sim_Am241.log`. The
same `--set` works in every executable (restManager, restGarfield...) for the parameters of its own YAML;
`Geant4ToRaw.yaml` and `Analysis.yaml` read the file given with `--i`, so they need no `--set`:

```bash
restManager --c Geant4ToRaw.yaml --i Run00001_Am241_calibration_AlphaCAMM.root   # R00001_RAW_Am241_calibration_AlphaCAMM.root
```

Inside `Simulation.yaml`, `${PARAM:isotope}` and `${PARAM:source}` are replaced by the values, and each
`select` block keeps the case named by the value (`default` if none matches).

Each event is a single decay of the isotope, at rest (`fullChain: false`: its daughter does not decay):
Geant4 gives its products (the alpha, the recoil nucleus, gammas and conversion electrons) with their
energies and branching ratios. `isotope` is the Geant4 name of the ion (element and mass number), so any
isotope can be simulated, e.g.:

| `isotope` | Main alpha |
|---|---|
| `Po210` (default) | 5.3 MeV |
| `Rn222` | 5.5 MeV |
| `Am241` | 5.5 MeV (and a gamma of 60 keV) |
| `Po218` | 6.0 MeV |
| `Po214` | 7.7 MeV |

| `source` | Where the isotope decays (uniformly) | Geometry |
|---|---|---|
| `sample` (default) | in the emitting layer of the sample (volume `source`) | `setup.gdml` |
| `gas` | in the volume of the gas | `setup_noSample.gdml` |
| `calibration` | in the calibration source (e.g. Am241): a disk of 5 mm diameter at the centre of the cathode, 1 um below it (to change it, edit the case `calibration`) | `setup_noSample.gdml` |
| `cathode` | on the surface of the cathode facing the gas (e.g. its contamination): the whole cathode, a disk of 376 mm diameter (copper, 5 mm thick, z = 90 to 95 mm), 1 um below it | `setup_noSample.gdml` |

The legacy simulations: `241AmFromSource.rml` is `--set isotope=Am241 --set source=calibration`, `222RnFromGas.rml`
is `--set isotope=Rn222 --set source=gas` and the Po210 from the sample is the default.

Only `sample` includes the sample in the geometry. The legacy Po218 and Po214 from the mylar have no place:
the geometry has no mylar (they can be simulated from the gas or the calibration source).

## Samples

Each sample has two files in `geometry/samples/`: `<name>.xml` (its solids and volumes, with the solid of
the gas) and `<name>_placement.xml` (where it is placed). The sample simulated is chosen with two symbolic
links of `geometry/`, read by `setup.gdml`:

```
geometry/
├── setup.gdml                    reads currentSample.xml and currentSample_placement.xml
├── currentSample.xml             -> samples/viton.xml               (symbolic link)
├── currentSample_placement.xml   -> samples/viton_placement.xml     (symbolic link)
└── samples/
    ├── viton.xml
    ├── viton_placement.xml
    ├── none.xml                  (no sample: used by setup_noSample.gdml)
    └── none_placement.xml
```

To simulate another sample, e.g. `teflon`, point both links to its files and check them:

```bash
cd geometry
ln -sf samples/teflon.xml currentSample.xml
ln -sf samples/teflon_placement.xml currentSample_placement.xml
ls -l currentSample*                   # both -> samples/teflon...
```

`Simulation.yaml` does not change, and the output file keeps the geometry simulated, with the sample. The
links are committed: commit their change only to change the default sample of the repository. On Windows,
git must be configured to create symbolic links (`core.symlinks`).

A new sample, copying `samples/viton.xml` and `samples/viton_placement.xml`:

- `<name>.xml` defines the solid of the gas, `gasSolid` (with the hole of the sample: the sample is not
  nested in the gas; `none.xml` defines only the whole gas), and the solids, volumes and constants of the
  sample. `viton.xml` (a Viton ring of radii 30 to 100 mm, 5 mm thick) has `sample_thickness` and
  `source_thickness`, the body `sampleVolume` and the emitting layer `sourceVolume`, whose physical volume
  must be named `source` (the generator of the source `sample`).
- `<name>_placement.xml` has the `physvol`s of the sample, placed inside the World (`none_placement.xml` is
  empty). They are two files because GDML needs the volumes defined before the World and their placement
  inside it (in `geometry.gdml`).

The constants of the GDML (e.g. `sample_thickness`) can also be changed with `--set`, but only after adding
a parameter of the same name to `Simulation.yaml`, whose value replaces the one of the GDML:

```yaml
parameters:
  isotope: "Po210"
  source: "sample"
  sample_thickness: 5   # mm, constant of samples/viton.xml
```

```bash
restG4 Simulation.yaml --set sample_thickness=3
```

Such a parameter only works with `source=sample`: the geometry without sample has no `sample_thickness`, and
a parameter used nowhere is an error.

## Legacy

| Legacy | Here |
|---|---|
| alphacamm-geometry | `geometry/` (the sample in `geometry/samples/`) |
| alphacamm-simulations/*.rml | `simulations/Simulation.yaml` (`isotope`, `source`) |
| alphacamm-readouts/microbulkReadout.rml, .dec | `AlphaCAMM/` of rest-common-readouts (`AlphaCAMM.root`) |
| alphacamm-analysis/g4ToSignal.rml, g4Ana.rml | `simulations/Geant4ToRaw.yaml` |
| alphacamm-analysis/alphaAna.rml | `analysis/Analysis.yaml` |
| alphacamm-analysis/plots/summary.rml | `analysis/plots/Plots.yaml` |
| alphacamm-analysis/plots/trackVis.rml | the event viewers of rest-framework |
| alphacamm-analysis/macros/CompareSim.C | `analysis/plots/CompareSim.yaml` |
| alphacamm-analysis/rawToSignal.rml | not needed: femclient writes `TRestRawSignalEvent` files |
| Gas from the gas server (`TRestDetectorGas`) | `ArIso.root` of rest-common-transport (restGarfield), medium `Ar99_iC4H10_1` |
