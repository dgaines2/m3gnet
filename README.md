[![GitHub license](https://img.shields.io/github/license/materialsvirtuallab/m3gnet)](https://github.com/materialsvirtuallab/m3gnet/blob/main/LICENSE)[![Linting](https://github.com/materialsvirtuallab/m3gnet/workflows/Linting/badge.svg)](https://github.com/materialsvirtuallab/m3gnet/workflows/Linting/badge.svg)[![Testing](https://github.com/materialsvirtuallab/m3gnet/workflows/Testing%20-%20main/badge.svg)](https://github.com/materialsvirtuallab/m3gnet/workflows/Testing/badge.svg)[![Downloads](https://pepy.tech/badge/m3gnet)](https://pepy.tech/project/m3gnet)

# M3GNet

M3GNet is a new materials graph neural network architecture that incorporates 3-body interactions. A key difference 
with prior materials graph implementations such as [MEGNet](https://github.com/materialsvirtuallab/megnet) is the
addition of the coordinates for atoms and the 3×3 lattice matrix in crystals, which are necessary for obtaining
tensorial quantities such as forces and stresses via auto-differentiation. 

As a framework, M3GNet has diverse applications, including:
- Interatomic potential development. With the same training data, M3GNet performs similarly to state-of-the-art machine
  learning interatomic potentials (ML-IAPs). However, a key feature of a graph-based potential is its flexibility to
  scale to diverse chemical spaces. One of the key accomplishments of M3GNet is the development of a *universal IAP*
  that can work across the entire periodic table of the elements by training on relaxations performed in the Materials
  Project.
- Surrogate models for property predictions. Like the previous MEGNet architecture, M3GNet can be used to develop
  surrogate models for property predictions, achieving in many cases accuracies that better or similar to other state
  of the art ML models.

For detailed performance benchmarks, please refer to the publication in the [References](#references) section. The 
Sphinx-generated API documentation is available via the [Github Page](http://materialsvirtuallab.github.io/m3gnet).

# Table of Contents
* [System requirements](#system-requirements)
* [Installation](#installation)
* [Usage](#usage)
* [Model training](#model-training)
* [Datasets](#datasets)
* [References](#references)

# System requirements

Inferences using the pre-trained models can be ran on any standard computer. For model training, the GPU memory needs 
to be > 18 Gb for a batch size of 32 using the crystal training data. In our work, we used a single RTX 3090 
GPU for model training. 

# Installation

M3GNet can be installed via pip:

```
pip install m3gnet
```

You can also directly download the source from Github and install from source.

## Apple Silicon Installation

Special care may be needed for Apple Silicon (M1, M1 Pro, M1 Max, M1 Ultra) machines. Apple Silicon has extremely
powerful ML capabilities. Here are the recommended steps to get m3gnet working on Apple Silicon devices.

1. Ensure that you already have XCode and CLI installed.
2. Install Miniconda or Anaconda.
3. Create a Python 3.9 environment,
```bash
conda create --name m3gnet python=3.9
conda activate m3gnet
```
4. First install tensorflow and its dependencies for Apple Silicon.
```bash
conda install -c apple tensorflow-deps
pip install tensorflow-macos
```
5. If you wish, you can install tensorflow-metal, which helps speed up training. If you encounter weird tensorflow 
   errors, you should uninstall tensorflow-metal and see if it fixes the errors first.
```
pip install tensorflow-metal 
```
6. Install m3gnet but ignore dependencies (otherwise, pip will look for tensorflow).
```bash
pip install --no-deps m3gnet
```
7. Install other dependencies like pymatgen, etc. manually.
```bash
pip install protobuf==3.20.0 pymatgen ase cython
```
8. Once you are done, you can try running `pytest m3gnet` to see if all tests pass.

# Usage

## Structure relaxation

A M3Gnet universal potential for the periodic table has been developed using data from Materials Project relaxations
since 2012. This universal potential can be used to perform structural relaxation of any arbitrary crystal as follows.

```python
from pymatgen.core import Structure, Lattice
from m3gnet.models import Relaxer

# Init a Mo structure with stretched lattice (DFT lattice constant ~ 3.168)
mo = Structure(Lattice.cubic(3.3), 
               ["Mo", "Mo"], [[0., 0., 0.], [0.5, 0.5, 0.5]])

relaxer = Relaxer()  # This loads the default pre-trained model

relax_results = relaxer.relax(mo)

final_structure = relax_results['final_structure']
final_energy = relax_results['trajectory'].energies[-1] / 2

print(f"Relaxed lattice parameter is {final_structure.lattice.abc[0]: .3f} Å")
print(f"Final energy is {final_energy.item(): .3f} eV/atom")
```

We will see the following output: 
```
Relaxed lattice parameter is  3.169 Å
Final energy is -10.859 eV/atom
```
The initial lattice parameter of 3.3 Å was successfully relaxed to 3.169 Å, close to the DFT value of 3.168 Å. 

The final energy -10.859 eV/atom is also close to Materials Project DFT value of
[-10.8456 eV/atom](http://materialsproject.org/materials/mp-129/).

The relaxation takes less than 20 seconds on a single laptop.

Here are more comprehensive benchmarks for cubic crystals based on 
[exp data on Wikipedia](http://en.wikipedia.org/wiki/Lattice_constant) and MP DFT data. The Jupyter notebook is in the 
[examples](/examples/Cubic%20Crystal%20Test.ipynb) folder. We only limit to cubic crystals here since there is only
one lattice parameter for comparison. However, M3GNet is not limited to cubic systems of course.

| Material    | Crystal structure   |       a |    mp_a |   predicted_a | % error vs Expt   | % error vs MP |
|:------------|:--------------------|--------:|--------:|--------------:|:------------------|:--------------|
| C (diamond) | Diamond (FCC)       | 3.567   | 3.57371 |       3.5718  | 0.13%             | -0.05%        |
| Si          | Diamond (FCC)       | 5.43102 | 5.46873 |       5.45002 | 0.35%             | -0.34%        |
| Ge          | Diamond (FCC)       | 5.658   | 5.76286 |       5.7698  | 1.98%             | 0.12%         |
| AlAs        | Zinc blende (FCC)   | 5.6605  | 5.73376 |       5.73027 | 1.23%             | -0.06%        |
| AlP         | Zinc blende (FCC)   | 5.451   | 5.50711 |       5.50346 | 0.96%             | -0.07%        |
| AlSb        | Zinc blende (FCC)   | 6.1355  | 6.23376 |       6.22817 | 1.51%             | -0.09%        |
| GaP         | Zinc blende (FCC)   | 5.4505  |  5.5063 |       5.5054  | 1.01%             | -0.02%        |
| GaAs        | Zinc blende (FCC)   | 5.653   | 5.75018 |       5.75055 | 1.73%             | 0.01%         |
| GaSb        | Zinc blende (FCC)   | 6.0959  | 6.21906 |       6.21939 | 2.03%             | 0.01%         |
| InP         | Zinc blende (FCC)   | 5.869   | 5.95673 |       5.9679  | 1.69%             | 0.19%         |
| InAs        | Zinc blende (FCC)   | 6.0583  | 6.18148 |       6.25374 | 3.23%             | 1.17%         |
| InSb        | Zinc blende (FCC)   | 6.479   | 6.63322 |       6.63863 | 2.46%             | 0.08%         |
| MgO         | Halite (FCC)        | 4.212   | 4.25648 |       4.2567  | 1.06%             | 0.01%         |
| CdS         | Zinc blende (FCC)   | 5.832   | 5.94083 |       5.9419  | 1.88%             | 0.02%         |
| CdSe        | Zinc blende (FCC)   | 6.05    | 6.21283 |       6.20987 | 2.64%             | -0.05%        |
| CdTe        | Zinc blende (FCC)   | 6.482   | 6.62905 |       6.62619 | 2.22%             | -0.04%        |
| ZnO         | Halite (FCC)        | 4.58    | 4.33888 |       4.33424 | -5.37%            | -0.11%        |
| ZnS         | Zinc blende (FCC)   | 5.42    | 5.45027 |       5.45297 | 0.61%             | 0.05%         |
| PbS         | Halite (FCC)        | 5.9362  | 6.00645 |       6.01752 | 1.37%             | 0.18%         |
| PbTe        | Halite (FCC)        | 6.462   | 6.56567 |       6.56111 | 1.53%             | -0.07%        |
| BN          | Zinc blende (FCC)   | 3.615   |   3.626 |       3.62485 | 0.27%             | -0.03%        |
| BP          | Zinc blende (FCC)   | 4.538   | 4.54682 |       4.54711 | 0.20%             | 0.01%         |
| LiF         | Halite              | 4.03    | 4.08343 |       4.08531 | 1.37%             | 0.05%         |
| LiCl        | Halite              | 5.14    | 5.15275 |       5.14745 | 0.15%             | -0.10%        |
| LiBr        | Halite              | 5.5     | 5.51343 |       5.51076 | 0.20%             | -0.05%        |
| LiI         | Halite              | 6.01    |  6.0257 |       6.02709 | 0.28%             | 0.02%         |
| NaF         | Halite              | 4.63    | 4.69625 |       4.69553 | 1.42%             | -0.02%        |
| NaCl        | Halite              | 5.64    | 5.69169 |       5.69497 | 0.97%             | 0.06%         |
| NaBr        | Halite              | 5.97    |  6.0276 |       6.01922 | 0.82%             | -0.14%        |
| NaI         | Halite              | 6.47    |   6.532 |       6.52739 | 0.89%             | -0.07%        |
| KF          | Halite              | 5.34    | 5.42398 |       5.41971 | 1.49%             | -0.08%        |
| KCl         | Halite              | 6.29    | 6.38359 |       6.39634 | 1.69%             | 0.20%         |
| KBr         | Halite              | 6.6     | 6.70308 |       6.70797 | 1.64%             | 0.07%         |
| KI          | Halite              | 7.07    | 7.18534 |       7.18309 | 1.60%             | -0.03%        |
| RbF         | Halite              | 5.65    | 5.73892 |       5.76843 | 2.10%             | 0.51%         |
| RbCl        | Halite              | 6.59    | 6.69873 |       6.67994 | 1.36%             | -0.28%        |
| RbBr        | Halite              | 6.89    | 7.02793 |       6.98219 | 1.34%             | -0.65%        |
| RbI         | Halite              | 7.35    | 7.48785 |       7.61756 | 3.64%             | 1.73%         |
| CsF         | Halite              | 6.02    | 6.11801 |       6.1265  | 1.77%             | 0.14%         |
| CsCl        | Caesium chloride    | 4.123   | 4.20906 |       4.20308 | 1.94%             | -0.14%        |
| CsI         | Caesium chloride    | 4.567   | 4.66521 |       4.90767 | 7.46%             | 5.20%         |
| Al          | FCC                 | 4.046   | 4.03893 |       4.04108 | -0.12%            | 0.05%         |
| Fe          | BCC                 | 2.856   | 2.84005 |       2.85237 | -0.13%            | 0.43%         |
| Ni          | FCC                 | 3.499   |  3.5058 |       3.5086  | 0.27%             | 0.08%         |
| Cu          | FCC                 | 3.597   | 3.62126 |       3.61199 | 0.42%             | -0.26%        |
| Mo          | BCC                 | 3.142   | 3.16762 |       3.16937 | 0.87%             | 0.06%         |
| Pd          | FCC                 | 3.859   | 3.95707 |       3.95466 | 2.48%             | -0.06%        |
| Ag          | FCC                 | 4.079   | 4.16055 |       4.16702 | 2.16%             | 0.16%         |
| W           | BCC                 | 3.155   | 3.18741 |       3.18826 | 1.05%             | 0.03%         |
| Pt          | FCC                 | 3.912   | 3.97677 |       3.97714 | 1.67%             | 0.01%         |
| Au          | FCC                 | 4.065   | 4.17129 |       4.17431 | 2.69%             | 0.07%         |
| Pb          | FCC                 | 4.92    | 5.05053 |       5.02849 | 2.21%             | -0.44%        |
| V           | BCC                 | 3.0399  | 2.99254 |       2.99346 | -1.53%            | 0.03%         |
| Nb          | BCC                 | 3.3008  | 3.32052 |       3.32221 | 0.65%             | 0.05%         |
| Ta          | BCC                 | 3.3058  | 3.32229 |       3.31741 | 0.35%             | -0.15%        |
| TiN         | Halite              | 4.249   | 4.25353 |       4.25254 | 0.08%             | -0.02%        |
| ZrN         | Halite              | 4.577   | 4.61762 |       4.61602 | 0.85%             | -0.03%        |
| HfN         | Halite              | 4.392   | 4.53774 |       4.53838 | 3.33%             | 0.01%         |
| VN          | Halite              | 4.136   | 4.12493 |       4.1281  | -0.19%            | 0.08%         |
| CrN         | Halite              | 4.149   |       - |       4.16068 | 0.28%             | -             |
| NbN         | Halite              | 4.392   | 4.45247 |       4.45474 | 1.43%             | 0.05%         |
| TiC         | Halite              | 4.328   | 4.33565 |       4.33493 | 0.16%             | -0.02%        |
| ZrC0.97     | Halite              | 4.698   | 4.72434 |       4.72451 | 0.56%             | 0.00%         |
| HfC0.99     | Halite              | 4.64    | 4.65131 |       4.65023 | 0.22%             | -0.02%        |
| VC0.97      | Halite              | 4.166   | 4.16195 |       4.16476 | -0.03%            | 0.07%         |
| TaC0.99     | Halite              | 4.456   | 4.48208 |       4.48225 | 0.59%             | 0.00%         |
| ScN         | Halite              | 4.52    | 4.51831 |       4.51797 | -0.04%            | -0.01%        |
| KTaO3       | Cubic perovskite    | 3.9885  | 4.03084 |       4.03265 | 1.11%             | 0.05%         |
| SrTiO3      | Cubic perovskite    | 3.98805 | 3.94513 |       3.94481 | -1.08%            | -0.01%        |
| EuTiO3      | Cubic perovskite    | 7.81    | 3.96119 |       3.92943 | -49.69%           | -0.80%        |
| SrVO3       | Cubic perovskite    | 3.838   | 3.90089 |       3.90604 | 1.77%             | 0.13%         |
| CaVO3       | Cubic perovskite    | 3.767   | 3.83041 |       3.83451 | 1.79%             | 0.11%         |

As you can see, almost all cubic lattice constants are within 1% of the DFT values. The only major errors are with
EuTiO3 wrt to the Wikipedia value. It is quite likely the Wikipedia value is wrong. All relaxations take less than 1s
on a M1 Max Mac.

## Molecular dynamics

Similarly, the universal IAP can be used to perform molecular dynamics (MD) simulations as well.

```python
from pymatgen.core import Structure, Lattice
from m3gnet.models import MolecularDynamics

# Init a Mo structure with stretched lattice (DFT lattice constant ~ 3.168)
mo = Structure(Lattice.cubic(3.3), 
               ["Mo", "Mo"], [[0., 0., 0.], [0.5, 0.5, 0.5]])

md = MolecularDynamics(
    atoms=mo,
    temperature=1000,  # 1000 K
    ensemble='nvt',  # NVT ensemble
    timestep=1, # 1fs,
    trajectory="mo.traj",  # save trajectory to mo.traj
    logfile="mo.log",  # log file for MD
    loginterval=100,  # interval for record the log
)

md.run(steps=1000)
```

After the run, `mo.log` contains thermodynamic information similar to the following:

```bash
Time[ps]      Etot[eV]     Epot[eV]     Ekin[eV]    T[K]
0.0000         -21.3307     -21.3307       0.0000     0.0
0.1000         -21.3307     -21.3307       0.0000     0.0
0.2000         -21.2441     -21.3087       0.0645   249.7
0.3000         -21.0466     -21.2358       0.1891   731.6
0.4000         -20.9702     -21.1149       0.1447   559.6
0.5000         -20.9380     -21.1093       0.1713   662.6
0.6000         -20.9176     -21.1376       0.2200   850.9
0.7000         -20.9016     -21.1789       0.2773  1072.8
0.8000         -20.8804     -21.1638       0.2835  1096.4
0.9000         -20.8770     -21.0695       0.1925   744.5
1.0000         -20.8908     -21.0772       0.1864   721.2
```

The MD run takes less than 1 minute. 

# Model training

You can also train your own IAP using the `PotentialTrainer` in `m3gnet.trainers`. The training dataset can include:
- structures, a list of pymatgen Structures
- energies, a list of energy floats with unit eV.
- forces, a list of nx3 force matrix with unit eV/Å, where n is the number of atom in 
  each structure. n does not need to be the same for all structures. 
- stresses, a list of 3x3 stress matrices with unit GPa (optional)

For stresses, we use the convention that compressive stress gives negative values. Stresses obtained from
VASP calculations (default unit is kBar) should be multiplied by -0.1 to work directly with the model.

We use validation dataset to select the stopping epoch number. The dataset has similar format as the training dataset. 

A minimal example of model training is shown below.

```python
from m3gnet.models import M3GNet, Potential
from m3gnet.trainers import PotentialTrainer

import tensorflow as tf

m3gnet = M3GNet(is_intensive=False)
potential = Potential(model=m3gnet)

trainer = PotentialTrainer(
    potential=potential, optimizer=tf.keras.optimizers.Adam(1e-3)
)

trainer.train(
    structures,
    energies,
    forces,
    stresses,
    validation_graphs_or_structures=val_structures,
    val_energies=val_energies,
    val_forces=val_forces,
    val_stresses=val_stresses,
    epochs=100,
    fit_per_element_offset=True,
    save_checkpoint=False,
)
```

# Datasets

The training data used to develop the universal M3GNet IAP is `MPF.2021.2.8` and is hosted on
[figshare](https://figshare.com/articles/dataset/MPF_2021_2_8/19470599) with DOI `10.6084/m9.figshare.19470599`.


# Reference

Please cite the following work:

```
Chi Chen, and Shyue Ping Ong. "A Universal Graph Deep Learning Interatomic Potential for the Periodic Table." 
arXiv preprint [arXiv:2202.02450](https://arxiv.org/abs/2202.02450) (2022).
```

# Acknowledgements

This work was primarily supported by the Materials Project, funded by the U.S. Department of Energy, Office of Science, 
Office of Basic Energy Sciences, Materials Sciences and Engineering Division under contract no. 
DE-AC02-05-CH11231: Materials Project program KC23MP. This work used the Expanse supercomputing cluster at the Extreme
Science and Engineering Discovery Environment (XSEDE), which is supported by National Science Foundation grant number
ACI-1548562.
