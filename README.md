# Gradio Molecule3D

<a href="https://pypi.org/project/gradio_molecule3d/" target="_blank"><img alt="PyPI - Version" src="https://img.shields.io/pypi/v/gradio_molecule3d"></a>
<img alt="Python" src="https://img.shields.io/badge/python-3.8+-blue.svg">
<img alt="License" src="https://img.shields.io/badge/license-MIT-green.svg">

**A Gradio custom component for 3D molecular structure visualization based on FAIR Chem's UMA component**

This component is **based on the [FAIR Chemistry UMA Molecule3D component](https://huggingface.co/spaces/facebook/fairchem_uma_demo)** and provides advanced 3D visualization of molecular structures with support for:
- ✨ **Multi-model PDB animation** for MD trajectories
- 🔬 **Periodic boundary conditions** (PBC) for crystals
- 🚀 **Automatic ASE format conversion** 
- 🎨 **Optimized visual defaults** (sphere + stick, Jmol colors)

## 🎯 Based on FAIR Chem UMA Component

This component incorporates the excellent work from Meta's FAIR Chemistry team, specifically their [UMA educational demo](https://huggingface.co/spaces/facebook/fairchem_uma_demo). The UMA (Universal Model for Atoms) component provides state-of-the-art molecular visualization capabilities optimized for:
- Molecular dynamics simulations
- Materials science applications
- Protein structure visualization
- Crystal structure analysis

**Attribution:** This component is derived from the FAIR Chemistry UMA Molecule3D component, which is MIT licensed by Meta Platforms, Inc. and its affiliates.

## ✨ Key Features

### 🎬 Multi-Model PDB Animation
The component handles **multi-model PDB files** as animation frames, perfect for MD trajectories:
```python
# Your ASE trajectory is automatically converted to animated PDB
viewer = Molecule3D(
    label="MD Simulation",
    inputs=[trajectory_file],
    value=lambda x: x,
)
# Users see smooth atomic motion! 🎥
```

### 🔬 Periodic Boundary Conditions
Automatically visualizes crystal structures with PBC:
- Unit cell repetition (15Å minimum)
- CRYST1 records for cell parameters
- Perfect for materials science

### 🚀 Automatic Format Conversion
Backend converts any ASE-compatible format to multi-model PDB:
- `.traj` → multi-model PDB with MODEL/ENDMDL tags
- `.cif`, `.xyz`, `.pdb` → optimized PDB
- Automatic PBC detection and handling

### 🎨 Optimized Visual Defaults
```python
DEFAULT_REPRESENTATIONS = [
    {"style": "sphere", "color": "Jmol", "scale": 0.3},
    {"style": "stick", "color": "Jmol", "scale": 0.2},
]
DEFAULT_CONFIG = {
    "backgroundColor": "white",
    "orthographic": False,
    "disableFog": False,
}
```

## Installation

### From GitHub

```bash
pip install git+https://github.com/FarisFLAIFIL/gradio-molecule3d.git
```

### From sync branch (latest UMA features)

```bash
pip install git+https://github.com/FarisFLAIFIL/gradio-molecule3d.git@sync-uma-component
```

### Local Development

```bash
git clone https://github.com/FarisFLAIFIL/gradio-molecule3d.git
cd gradio-molecule3d
pip install -e .
```

## Quick Start

### UMA-Style MD Visualization

```python
import gradio as gr
from gradio_molecule3d import Molecule3D

# Create demo with UMA defaults
with gr.Blocks() as demo:
    gr.Markdown("# 🎬 Molecular Dynamics Viewer")
    
    with gr.Row():
        with gr.Column():
            # File upload - supports ASE trajectories
            trajectory_file = gr.File(
                label="Upload MD Trajectory",
                file_types=[".traj", ".pdb", ".xyz", ".cif"]
            )
        
        with gr.Column(scale=2):
            # 3D viewer with animation
            viewer = Molecule3D(
                label="Animated Trajectory",
                reps=[
                    {"style": "sphere", "color": "Jmol", "scale": 0.3},
                    {"style": "stick", "color": "Jmol", "scale": 0.2},
                ],
                config={
                    "backgroundColor": "white",
                    "orthographic": False,
                    "disableFog": False,
                },
                height=500,
                inputs=[trajectory_file],
                value=lambda x: x,
            )

if __name__ == "__main__":
    demo.launch()
```

### MACE Chatbot Integration

```python
from gradio_molecule3d import Molecule3D
from ase.io import read, write

def run_md_simulation(structure_file, steps=100):
    """Run MD with MACE and visualize"""
    # Your MACE MD code here...
    # atoms_list = run_mace_md(structure_file, steps)
    
    # Save as multi-model PDB for animation
    output_pdb = "trajectory.pdb"
    write(output_pdb, atoms_list, format='proteindatabank', multimodel=True)
    
    return output_pdb

# In your Gradio app:
with gr.Blocks() as demo:
    structure_input = gr.File(label="Structure")
    md_steps = gr.Slider(1, 500, 100, label="MD Steps")
    trajectory_output = gr.File(label="Trajectory")
    
    viewer = Molecule3D(
        label="MD Simulation",
        height=500,
        inputs=[trajectory_output],
        value=lambda x: x,
    )
    
    run_button = gr.Button("Run MD")
    run_button.click(
        run_md_simulation,
        inputs=[structure_input, md_steps],
        outputs=[trajectory_output]
    )
```

### Custom Representations

```python
import gradio as gr
from gradio_molecule3d import Molecule3D

# Custom representation configuration
custom_reps = [
    {
        "model": 0,
        "style": "cartoon",
        "color": "hydrophobicity",
        "around": 0,
        "byres": False,
    },
    {
        "model": 0,
        "chain": "A",
        "resname": "HIS",
        "style": "stick",
        "color": "red"
    }
]

with gr.Blocks() as demo:
    molecule_viewer = Molecule3D(
        label="Custom Styled Molecule",
        reps=custom_reps
    )
    
demo.launch()
```

## 🔧 How It Works

### Multi-Model PDB Generation

The backend automatically converts trajectories to multi-model PDB:

```python
def write_proteindatabank(fileobj, images):
    """Write multi-model PDB with MODEL/ENDMDL tags"""
    for n, atoms in enumerate(images):
        if atoms.get_pbc().any():
            # Write CRYST1 record for unit cell
            fileobj.write('CRYST1...')
        fileobj.write(f'MODEL     {n + 1}\n')
        # Write atoms...
        fileobj.write('ENDMDL\n')
```

### PBC Handling

```python
def find_minimum_repeats(atoms, min_length=15.0):
    """Repeat unit cell to meet minimum size"""
    cell_lengths = atoms.get_cell().lengths()
    repeats = [max(1, int(np.ceil(min_length / length))) 
               for length in cell_lengths]
    return tuple(repeats)
```

### Format Conversion

```python
def convert_file_to_pdb(file_path, gradio_cache):
    """Convert any ASE format to multi-model PDB"""
    structures = ase.io.read(file_path, ':')  # Read all frames
    
    if all(structures[0].pbc):
        repeats = find_minimum_repeats(structures[0])
        structures = [s.repeat(repeats) for s in structures]
    
    # Write multi-model PDB
    temp_pdb = tempfile.NamedTemporaryFile(suffix=".pdb", delete=False)
    write_proteindatabank(temp_pdb, structures)
    return temp_pdb.name
```

## `Molecule3D`

### Initialization

<table>
<thead>
<tr>
<th align="left">name</th>
<th align="left" style="width: 25%;">type</th>
<th align="left">default</th>
<th align="left">description</th>
</tr>
</thead>
<tbody>
<tr>
<td align="left"><code>value</code></td>
<td align="left" style="width: 25%;">

```python
str | list[str] | Callable | None
```

</td>
<td align="left"><code>None</code></td>
<td align="left">Default file(s) to display, given as a str file path or URL, or a list of str file paths / URLs. If callable, the function will be called whenever the app loads to set the initial value of the component.</td>
</tr>

<tr>
<td align="left"><code>reps</code></td>
<td align="left" style="width: 25%;">

```python
Any | None
```

</td>
<td align="left"><code>[
    {
        "model": 0,
        "chain": "",
        "resname": "",
        "style": "sphere",
        "color": "Jmol",
        "around": 0,
        "byres": False,
        "scale": 0.3,
    },
    {
        "model": 0,
        "chain": "",
        "resname": "",
        "style": "stick",
        "color": "Jmol",
        "around": 0,
        "byres": False,
        "scale": 0.2,
    },
]</code></td>
<td align="left">Visual representation configuration</td>
</tr>

<tr>
<td align="left"><code>config</code></td>
<td align="left" style="width: 25%;">

```python
Any | None
```

</td>
<td align="left"><code>{
    "backgroundColor": "white",
    "orthographic": False,
    "disableFog": False,
}</code></td>
<td align="left">3Dmol.js viewer configuration</td>
</tr>

<tr>
<td align="left"><code>confidenceLabel</code></td>
<td align="left" style="width: 25%;">

```python
str | None
```

</td>
<td align="left"><code>"pLDDT"</code></td>
<td align="left">label for confidence values stored in the bfactor column of a pdb file</td>
</tr>

<tr>
<td align="left"><code>file_count</code></td>
<td align="left" style="width: 25%;">

```python
Literal["single", "multiple", "directory"]
```

</td>
<td align="left"><code>"single"</code></td>
<td align="left">if single, allows user to upload one file. If "multiple", user uploads multiple files. If "directory", user uploads all files in selected directory.</td>
</tr>

<tr>
<td align="left"><code>file_types</code></td>
<td align="left" style="width: 25%;">

```python
list[str] | None
```

</td>
<td align="left"><code>None</code></td>
<td align="left">List of file extensions (e.g. ['.pdb', '.cif', '.traj'])</td>
</tr>

<tr>
<td align="left"><code>height</code></td>
<td align="left" style="width: 25%;">

```python
int | float | None
```

</td>
<td align="left"><code>None</code></td>
<td align="left">Height of the viewer in pixels</td>
</tr>

<tr>
<td align="left"><code>inputs</code></td>
<td align="left" style="width: 25%;">

```python
Component | Sequence[Component] | None
```

</td>
<td align="left"><code>None</code></td>
<td align="left">Components to watch for updates (e.g., trajectory file)</td>
</tr>

<tr>
<td align="left"><code>interactive</code></td>
<td align="left" style="width: 25%;">

```python
bool | None
```

</td>
<td align="left"><code>None</code></td>
<td align="left">if True, allows file upload; if False, display only</td>
</tr>

</tbody></table>

### Events

| name | description |
|:-----|:------------|
| `change` | Triggered when the value changes |
| `select` | Triggered when user selects/deselects |
| `clear` | Triggered when user clears the component |
| `upload` | Triggered when user uploads a file |
| `delete` | Triggered when user deletes a file |

## 🎓 Use Cases

### 1. Molecular Dynamics Simulations
- Visualize MACE, UMA, or other ML potential MD trajectories
- Smooth animation of atomic motion
- Perfect for educational demonstrations

### 2. Materials Science
- Crystal structure visualization with PBC
- Unit cell repetition for better viewing
- Inorganic and organic molecular crystals

### 3. Protein Structure Analysis
- PDB file visualization
- Multi-model NMR ensembles
- Structural biology applications

### 4. Chemistry Education
- Interactive molecular visualization
- Real-time structure manipulation
- Automatic format handling

## 📚 Resources

- **FAIR Chemistry UMA Demo**: https://huggingface.co/spaces/facebook/fairchem_uma_demo
- **UMA Model**: https://huggingface.co/facebook/UMA
- **FAIR Chemistry GitHub**: https://github.com/facebookresearch/fairchem
- **3Dmol.js Documentation**: https://3dmol.csb.pitt.edu/
- **ASE Documentation**: https://wiki.fysik.dtu.dk/ase/

## 📄 License

This component is **MIT licensed**, matching the FAIR Chemistry UMA component.

**Original UMA Component**: Copyright (c) Meta Platforms, Inc. and its affiliates.  
**This Repository**: Copyright (c) 2025 Faris FLAIFIL

See [LICENSE](LICENSE) for full details.

## 🙏 Acknowledgments

This component is based on the excellent work by the FAIR Chemistry team at Meta:
- UMA component from the [FAIR Chem educational demo](https://huggingface.co/spaces/facebook/fairchem_uma_demo)
- Built on [gradio-molecule3d](https://huggingface.co/spaces/simonduerr/gradio_molecule3d) by Simon Duerr
- Uses [3Dmol.js](https://3dmol.csb.pitt.edu/) for visualization
- Integrates with [ASE](https://wiki.fysik.dtu.dk/ase/) for structure handling

Special thanks to the entire FAIR Chemistry team for making UMA and this component available as open source!

## 🔧 Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## 📧 Contact

For issues or questions:
- Open an issue on [GitHub](https://github.com/FarisFLAIFIL/gradio-molecule3d/issues)
- Check the [FAIR Chemistry discussions](https://github.com/facebookresearch/fairchem/discussions)
