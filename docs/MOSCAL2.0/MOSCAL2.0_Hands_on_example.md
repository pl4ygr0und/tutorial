## Exploring the Spin-Boson Model with `MOSCAL2.0`

This guide aims to provide a step-by-step walkthrough of utilizing the `c++` binary program in `MOSCAL2.0` to apply the `DEOM` formalism in studying the spin-boson model. To provide an overview:

- **Prerequisites:** We'll first outline the prerequisites necessary for you to successfully follow this tutorial.
- **Understanding the Spin-Boson Model:** We'll delve into the fundamental concepts behind the spin-boson model to lay the groundwork for our exploration.
- **`MOSCAL2.0` Workflow:** Explaining how `MOSCAL2.0` harnesses its `python` part to generate input for the `c++` binaries used in the spin-boson model analysis.
- **Generating Spin-Boson Model Inputs:** We'll guide you through the process of generating inputs tailored for the spin-boson model using `MOSCAL2.0`.

This walkthrough will equip you with the necessary understanding and tools to explore and analyze the spin-boson model using `MOSCAL2.0` effectively.

### Prerequisites: `bose_linear_2.out`

Before diving into the hands-on experience with the spin-boson model in `MOSCAL2.0`, it's essential to ensure that you have successfully compiled the package following the installation guide. 

Additionally, make sure you have obtained the specific binary, `bose_linear_2.out`, which corresponds to the spin-boson model within the `MOSCAL2.0` package.

Finally, you shall install the official `python` module.

If you have any problems, please refer to the installation guide.

### Understanding the Spin-Boson Model
The spin-boson model serves as a fundamental framework in quantum physics, describing the interaction between a quantum system (often representing a 'spin') and a surrounding environment (the 'bath'). Such interactions can be represented by Hamiltonian
$$ H = H_\text{s} + H_\text{b} + H_\text{sb}. $$
In this model, the system Hamiltonian is a general two-state Hamiltonian, i.e., a 2-by-2 matrix. The bath are bosonic degrees of freedom, 
$$ H_\text{b} = \frac{1}{2} \sum_j \frac{\omega_j}{2}(p_j^2 + q_j^2), $$
where $q_j$ and $p_j$ are the position and momentum operators for $j$-th oscillator, whose frequency is $\omega_j$ (note that we have set $\hbar=1$).

In the vacuum, the isolated two-state system (the spin) and the bath oscillators are exactly solvable. Whereas, the interesting physics arise when these two system have interaction. 

We will consider the simplest interaction possible -- the linear interaction: 
$$ H_\text{sb} = Q_\text{s} x_\text{B}, \quad x_\text{B} = \sum_j c_j q_j, $$
where we see a system mode, $Q_\text{s}$, which is a 2-by-2 matrix, interacts with a collective phonon mode $x_\text{B}$. Phonon mode $x_\text{B}$ is a linear combination of all the bath operators.
Since we have a linear combination of ten's of thousands of oscillators (imagine a solid-state material), the collective mode $x_\text{B}$ corresponds to a spectral density defined by the following equation
$$ J(\omega) = \sum_j \frac{c_j}{2} \delta(\omega - \omega_j). $$
In practice, experimentalists can characterize $x_\text{B}$ by measuring the phonon spectra. But as theory people, we primarily use analytic spectral function to make our life easier. In particular, the Drude-Lorentz spectral function
$$ J(\omega) = \frac{2\lambda\gamma\omega}{\omega^2 + \gamma^2} $$
is the simplest one we can use. Here in the Drude spectral function:

- $\lambda$ characterizes the system-bath interaction strength
- $\gamma$ characterizes the "peak" position 

(Feel free to use python `matplotlib` package to visualize this spectarl function.)

In conclusion, the spin-boson model characterize the linear coupling between a 2-state system and a very large bath of harmnic oscillators.
The detail of the interaction between the discrete system and the continuum bath is fully characterized by the spectral function $J(\omega)$.

The goal of open quantum dynamics simultion is to know, if we have a initial density matrix $\rho_\text{s}(t=0)$ for the system, what will this reduced density matrix for $t>0$.

!!! info
    From this very simple description of the spin-boson model, can you see why it's required for you to compile `bose_linear_2.out`.

Despite the goal of this tutorial is to tell you how to numerically simulate the spin-boson model using the DEOM formalism. 
It is important to know that dynamics for such model is exactly solvable on paper. If you feel interested (, and have time of course), you can check Chapter 12 of Chemical Dynamics in Condensed Phases by Abraham Nitzan.

### The Workflow for the `MOSCAL2.0` package: 

The `MOSCAL2.0` package streamlines the process of generating inputs for simulations by following a structured workflow:

1. **Utilizing Official Python Scripts:** `MOSCAL2.0` provides official Python scripts designed to facilitate the generation of inputs required for simulations. These scripts are crafted to simplify the input creation process by allowing users to define parameters, configurations, and specifications in a structured manner. These scripts generate the necessary input data in `json` format, encapsulating the simulation details.

2. **Creating the `input.json` File:** Upon executing the official Python scripts, the result is the creation of an `input.json` file. This file contains the formatted input data, encapsulating the specifics necessary for the subsequent `MOSCAL2.0` simulations. The `json` format ensures readability, portability, and ease of use for the generated input information.

3. **Invoking `MOSCAL2.0` Binary Execution:** With the `input.json` file prepared, the next step involves invoking the relevant `MOSCAL2.0` binary files. These binaries, residing within the same directory as the `input.json` file, are executed to conduct simulations or analyses based on the provided input parameters. The binaries interpret the information within `input.json`, enabling the initiation of simulations tailored to the defined configurations.

### Concrete example

In this example, we will simulate dynamics the following system Hamiltonian
$$ H_\text{s} = \\begin{bmatrix} 1 & 1 \\\\ 1 & -1 \\ \\end{bmatrix}, $$
with system bath interaction
$$ H_\text{sb} = Q x_\text{B} = \\begin{bmatrix} 0 & 1 \\\\ 1 & 0 \\ \\end{bmatrix} x_\text{B}.$$
The spectral function is
$$ J(\omega) = \frac{2\lambda\gamma\omega}{\omega^2 + \gamma^2}, \lambda = \gamma = 1.0, $$
at bath temperature $k_\text{B} T = 1.0$ (i.e., $\beta = 1.0$).

First, you write your `python` script `spin-boson.py` to generate the input for the `bose_linear_2.out` binary.

```python
# spin-boson.py

import numpy as np
import sympy as sp

import json
from deom import convert, decompose_spe, complex_2_json, init_qmd


def decompose_drude_spectral_function(npsd: int, beta: float, gamma: float=1.0, lambd: float=1.0):
    w_sp, lambd_sp, gamma_sp, beta_sp = sp.symbols(r"\omega, \lambda, \gamma, \beta", real=True)
    spe_vib_sp = 2 * lambd_sp * gamma_sp / (gamma_sp - sp.I * w_sp)
    sp_para_dict = {lambd_sp: lambd, gamma_sp: gamma}
    condition_dict = {}
    para_dict = {'beta': beta}

    etal, etar, etaa, expn = decompose_spe(spe_vib_sp, w_sp, sp_para_dict, para_dict,
                                           condition_dict, npsd)

    return etal, etar, etaa, expn

if __name__ == "__main__":
    #------------------------------------------------------------------------------#
    # The simulation parameters
    #------------------------------------------------------------------------------#

    # These parameters controls the dynamics
    dt = 0.01
    tf = 25

    # These parameters controls the accuracy of DEOM
    npsd = 2       # number of exponential decays to decompose the correlation function
    lmax = 50      # maximum tiers for the dissipaton density operators
    ferr = 1e-12   # the filtering tolerence (ignore the very small numbers)
    nmax = 1000000 # maximum number of dissipaton density operators

    # These parameters controls the bath, bath mode, and spectral function
    beta  = 1.0   # The inverse temperature 1/ (k_B T)
    gamma = 1.0   # gamma in drude spectral
    lambd = 1.0   # lambda in drude spectral
    nmod  = 1     # number of collective modes x_B to consider

    #------------------------------------------------------------------------------#
    # Hamiltonian H, system mode Q, and initial density matrix rho
    #------------------------------------------------------------------------------#
    rho0 = np.zeros((2, 2), dtype=complex)
    rho0[0, 0] = 1
    rho0[1, 1] = 0 # rho0 is initilize as |0><0|

    hams = np.zeros((2, 2), dtype=complex)
    hams[0, 0] = 1
    hams[0, 1] = 1
    hams[1, 0] = 1
    hams[1, 1] = -1 # This is the two-state system Hamiltonian

    qmds = np.zeros((nmod, 2, 2), dtype=complex)
    qmds[0, 0, 1] = 1
    qmds[0, 1, 0] = 1 # This is the interacting system mode, here we consider only one mode; essentially the mode is the pauli matrix sigma_x

    nsys = 2

    #------------------------------------------------------------------------------#
    # Decompose the spectral function (more precisely the correlation function)
    #------------------------------------------------------------------------------#
    etal, etar, etaa, expn = decompose_drude_spectral_function(npsd, beta, gamma, lambd)
    mode = np.zeros_like(expn, dtype=int)

    json_init = {
         "nmax": nmax,
         "lmax": lmax,
         "ferr": ferr,
         "filter": True,
         "nind": len(expn),
         "nmod": nmod,
         "equilibrium": {
             "sc2": False,
             "dt-method": True,
             "ti": 0,
             "tf": 25,
             "dt": dt,
             "backup": True,
         },
         "expn": complex_2_json(expn),
         "ham1": complex_2_json(hams),
         "coef_abs": complex_2_json(etaa),
         "read_rho0": True,
         "rho0": complex_2_json(rho0),
     }
    
    init_qmd(json_init, qmds, qmds, mode, nsys, etaa, etal, etar)
    with open('input.json', 'w') as f:
        json.dump(json_init, f, indent=2, default=convert)
```

Run this script,

```bash
python spin-boson.py
```

then you shall see `input.json` in the current directory.

Finally, you can run your `MOSCAL` program by executing these commands in your command line:
```bash
MOSCAL_BIN="/path/to/bose_linear_2.out"

${MOSCAL_BIN} # make sure you have input.json in your working directory
```
(Alternatively, you can write a shell script `run_spin_boson.sh` to do this.)