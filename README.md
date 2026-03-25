# GaMD-protocol

### **Dependencies:**
* wget
* Gromacs
* python3
* [gamd-openmm](https://github.com/MiaoLab20/gamd-openmm)

### **Step 1:** Get PDB structure
```bash
wget https://files.rcsb.org/download/1UAO.pdb
```

> [!NOTE]
> eg: 1UAO &rarr; PDB ID

### **Step 2:** Generate Topology
```bash
gmx pdb2gmx -f 1UAO.pdb -o processed.gro -ignh
```
**Choose:**
* Force field
* Water

**Output:**
* processed.gro
* topol.top

### **Step 3:** Define simulation box
```bash
gmx editconf -f processed.gro -o box.gro -c -d 1.0 -bt cubic
```

### **Step 4:** Solvate
```bash
gmx solvate -cp box.gro -cs tip3p.gro -o solv.gro -p topol.top
```
> [!NOTE]
> -cs &rarr; ex: tip3p.gro, you can choose any solvent as per your forcefield.

### **Step 5:** Add ions (neutralize system)
```bash
gmx grompp -f mdp/ions.mdp -c solv.gro -p topol.top -o ions.tpr
gmx genion -s ions.tpr -o solv_ions.gro -p topol.top -pname NA -nname CL -neutral
```
> [!NOTE]
> All mdp files used here are available inside the mdp directory of this repository

### **Step 6:** Run minimization
```bash
gmx grompp -f mdp/minim.mdp -c solv_ions.gro -p topol.top -o em.tpr
gmx mdrun -deffnm em
```
### **Step 7:** Run heating steps
```bash
gmx grompp -f mdp/nvt_300.mdp -c em.gro -p topol.top -o nvt_300.tpr
gmx mdrun -deffnm nvt_300

gmx grompp -f mdp/nvt_400.mdp -c nvt_300.gro -p topol.top -o nvt_400.tpr
gmx mdrun -deffnm nvt_400

gmx grompp -f mdp/nvt_500.mdp -c nvt_400.gro -p topol.top -o nvt_500.tpr
gmx mdrun -deffnm nvt_500

gmx grompp -f mdp/nvt_600.mdp -c nvt_500.gro -p topol.top -o nvt_600.tpr
gmx mdrun -deffnm nvt_600

gmx grompp -f mdp/nvt_700.mdp -c nvt_600.gro -p topol.top -o nvt_700.tpr
gmx mdrun -deffnm nvt_700
```

### **Step 8:** Production
```bash
gmx grompp -f mdp/prod_700.mdp -c nvt_700.gro -p topol.top -o prod_700.tpr
gmx mdrun -deffnm prod_700
```
> [!IMPORTANT]  
> Select highly extended structure based on Rg or end-end distance in case of chignolin for GaMD.

### **Step 9:** Run GaMD:
```bash
gamdRunner xml input.xml -p CUDA -d 0 -o gamd
```
> [!NOTE]
> example xml file provided in this repository.