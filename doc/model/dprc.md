# Deep Potential - Range Correction (DPRc)

Deep Potential - Range Correction (DPRc) is designed to combine with QM/MM method, and corrects energies from a low-level QM/MM method to a high-level QM/MM method:

$$ E=E_\text{QM}(\mathbf R; \mathbf P)  + E_\text{QM/MM}(\mathbf R; \mathbf P) + E_\text{MM}(\mathbf R) + E_\text{DPRc}(\mathbf R) $$

See the [JCTC paper](https://doi.org/10.1021/acs.jctc.1c00201) for details.

## Training data

Instead the normal _ab initio_ data, one needs to provide the correction from a low-level QM/MM method to a high-level QM/MM method:

$$ E = E_\text{high-level QM/MM} - E_\text{low-level QM/MM} $$

Two levels of data use the same MM method, so $E_\text{MM}$ is eliminated.

## Training the DPRc model

In a DPRc model, QM atoms and MM atoms have different atom types. Assuming we have 4 QM atom types (C, H, O, P) and 2 MM atom types (HW, OW):

```json
"type_map": ["C", "H", "HW", "O", "OW", "P"]
```

As described in the paper, the DPRc model only corrects $E_\text{QM}$ and $E_\text{QM/MM}$ within the cutoff, so we use a hybrid descriptor to describe them separatedly:

```json
"descriptor" :{
    "type":             "hybrid",
    "list" : [
        {
            "type":     "se_e2_a",
            "sel":              [6, 11, 0, 6, 0, 1],
            "rcut_smth":        1.00,
            "rcut":             9.00,
            "neuron":           [12, 25, 50],
            "exclude_types":    [[2, 2], [2, 4], [4, 4], [0, 2], [0, 4], [1, 2], [1, 4], [3, 2], [3, 4], [5, 2], [5, 4]],
            "axis_neuron":      12,
            "set_davg_zero":    true,
            "_comment": " QM/QM interaction"
        },
        {
            "type":     "se_e2_a",
            "sel":              [6, 11, 100, 6, 50, 1],
            "rcut_smth":        0.50,
            "rcut":             6.00,
            "neuron":           [12, 25, 50],
            "exclude_types":    [[0, 0], [0, 1], [0, 3], [0, 5], [1, 1], [1, 3], [1, 5], [3, 3], [3, 5], [5, 5], [2, 2], [2, 4], [4, 4]],
            "axis_neuron":      12,
            "set_davg_zero":    true,
            "_comment": " QM/MM interaction"
        }
    ]
}
```

{ref}`exclude_types <model/descriptor[se_e2_a]/exclude_types>` can be generated by the following Python script:
```py
from itertools import combinations_with_replacement, product

qm = (0, 1, 3, 5)
mm = (2, 4)
print(
    "QM/QM:",
    list(map(list, list(combinations_with_replacement(mm, 2)) + list(product(qm, mm)))),
)
print(
    "QM/MM:",
    list(
        map(
            list,
            list(combinations_with_replacement(qm, 2))
            + list(combinations_with_replacement(mm, 2)),
        )
    ),
)
```

Also, DPRc assumes MM atom energies ({ref}`atom_ener <model/fitting_net[ener]/atom_ener>`) are zero:

```json
"fitting_net": {
   "neuron": [240, 240, 240],
   "resnet_dt": true,
   "atom_ener": [null, null, 0.0, null, 0.0, null]
}
```

Note that {ref}`atom_ener <model/fitting_net[ener]/atom_ener>` only works when {ref}`descriptor/set_davg_zero <model/descriptor[se_e2_a]/set_davg_zero>` is `true`.

## Run MD simulations

The DPRc model has the best practices with the [AMBER](../third-party/out-of-deepmd-kit.md#amber-interface-to-deepmd-kit) QM/MM module. An example is given by [GitLab RutgersLBSR/AmberDPRc](https://gitlab.com/RutgersLBSR/AmberDPRc/). In theory, DPRc is able to be used with any QM/MM package, as long as the DeePMD-kit package accepts QM atoms and MM atoms within the cutoff range and returns energies and forces.

## Pairwise DPRc

If one wants to correct from a low-level method into a full DFT level, and the system is too large to do full DFT calculation, one may try the experimental pairwise DPRc model.
In a pairwise DPRc model, the total energy is divided into QM internal energy and the sum of QM/MM energy for each MM residue $l$:

$$ E = E_\text{QM} + \sum_{l} E_{\text{QM/MM},l} $$

In this way, the interaction between the QM region and each MM fragmentation can be computed and trained separately.
Thus, the pairwise DPRc model is divided into two sub-[DPRc models](./dprc.md).
`qm_model` is for the QM internal interaction and `qmmm_model` is for the QM/MM interaction.
The configuration for these two models is similar to the non-pairwise DPRc model.
It is noted that the [`se_atten` descriptor](./train-se-atten.md) should be used, as it is the only descriptor to support the mixed type.

```json
{
  "model": {
    "type": "pairwise_dprc",
    "type_map": [
      "C",
      "P",
      "O",
      "H",
      "OW",
      "HW"
    ],
    "type_embedding": {
      "neuron": [
        8
      ],
      "precision": "float32"
    },
    "qm_model": {
      "descriptor": {
        "type": "se_atten_v2",
        "sel": 24,
        "rcut_smth": 0.50,
        "rcut": 9.00,
        "attn_layer": 0,
        "neuron": [
          25,
          50,
          100
        ],
        "resnet_dt": false,
        "axis_neuron": 12,
        "precision": "float32",
        "seed": 1
      },
      "fitting_net": {
        "type": "ener",
        "neuron": [
          240,
          240,
          240
        ],
        "resnet_dt": true,
        "precision": "float32",
        "atom_ener": [
          null,
          null,
          null,
          null,
          0.0,
          0.0
        ],
        "seed": 1
      }
    },
    "qmmm_model": {
      "descriptor": {
        "type": "se_atten_v2",
        "sel": 27,
        "rcut_smth": 0.50,
        "rcut": 6.00,
        "attn_layer": 0,
        "neuron": [
          25,
          50,
          100
        ],
        "resnet_dt": false,
        "axis_neuron": 12,
        "set_davg_zero": true,
        "exclude_types": [
          [
            0,
            0
          ],
          [
            0,
            1
          ],
          [
            0,
            2
          ],
          [
            0,
            3
          ],
          [
            1,
            1
          ],
          [
            1,
            2
          ],
          [
            1,
            3
          ],
          [
            2,
            2
          ],
          [
            2,
            3
          ],
          [
            3,
            3
          ],
          [
            4,
            4
          ],
          [
            4,
            5
          ],
          [
            5,
            5
          ]
        ],
        "precision": "float32",
        "seed": 1
      },
      "fitting_net": {
        "type": "ener",
        "neuron": [
          240,
          240,
          240
        ],
        "resnet_dt": true,
        "seed": 1,
        "precision": "float32",
        "atom_ener": [
          0.0,
          0.0,
          0.0,
          0.0,
          0.0,
          0.0
        ]
      }
    }
  }
}
```

The pairwise model needs information for MM residues.
The model uses [`aparam`](../data/system.md) with the shape of `nframes x natoms` to get the residue index.
The QM residue should always use `0` as the index.
For example, `0 0 0 1 1 1 2 2 2` means these 9 atoms are grouped into one QM residue and two MM residues.
