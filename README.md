# GaMD Protocol (GROMACS → OpenMM)

A reproducible workflow to prepare a protein system using **GROMACS** and run **Gaussian Accelerated Molecular Dynamics (GaMD)** using **OpenMM (gamd-openmm)**.

---

## 📦 Dependencies

Ensure the following are installed and available in your `$PATH`:

- `wget`
- `gmx` (GROMACS 2020+ recommended)
- `python3`
- [gamd-openmm](https://github.com/MiaoLab20/gamd-openmm)

Optional but recommended:
- CUDA-enabled GPU (for faster GaMD simulations)

---

## 📁 Input

- Protein structure in **PDB format** (e.g., `1UAO.pdb`)

---

## ⚙️ Workflow Overview

1. Download structure
2. Generate topology
3. Define box & solvate
4. Add ions
5. Energy minimization
6. Equilibration (multi-temperature NVT)
7. Production MD
8. Select extended structure
9. Run GaMD (OpenMM)

---

## 🚀 Step-by-Step Protocol

### **1. Download PDB structure**
```bash
wget https://files.rcsb.org/download/1UAO.pdb
```

> Replace `1UAO` with your desired PDB ID.

---

### **2. Generate topology**
```bash
gmx pdb2gmx -f 1UAO.pdb -o processed.gro -ignh
```

**Select interactively:**
- Force field (e.g., AMBER, CHARMM)
- Water model (must match later steps)

**Outputs:**
- `processed.gro`
- `topol.top`

---

### **3. Define simulation box**
```bash
gmx editconf -f processed.gro -o box.gro -c -d 1.0 -bt cubic
```

- `-d 1.0` → 1 nm padding from protein to box edge

---

### **4. Solvate system**
```bash
gmx solvate -cp box.gro -cs tip3p.gro -o solv.gro -p topol.top
```

> Ensure the solvent model (`tip3p.gro`) is consistent with the chosen force field.

---

### **5. Add ions (neutralization)**
```bash
gmx grompp -f mdp/ions.mdp -c solv.gro -p topol.top -o ions.tpr
gmx genion -s ions.tpr -o solv_ions.gro -p topol.top -pname NA -nname CL -neutral
```

- Replaces solvent molecules with ions
- Neutralizes total system charge

---

### **6. Energy minimization**
```bash
gmx grompp -f mdp/minim.mdp -c solv_ions.gro -p topol.top -o em.tpr
gmx mdrun -deffnm em
```

- Removes steric clashes
- Ensures stable starting structure

---

### **7. Stepwise heating (NVT equilibration)**

Gradual heating improves stability and avoids structural distortion.

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

---

### **8. Production MD (high temperature)**
```bash
gmx grompp -f mdp/prod_700.mdp -c nvt_700.gro -p topol.top -o prod_700.tpr
gmx mdrun -deffnm prod_700
```

- High-temperature simulation helps explore unfolded conformations
- Useful for generating initial structures for GaMD

---

### **9. Select extended structure**

From the trajectory (`prod_700.xtc`), select a **highly extended conformation**:

Typical metrics:
- Radius of gyration (Rg)
- End-to-end distance

> For systems like **chignolin**, choosing a maximally extended structure improves GaMD sampling efficiency.

---

### **10. Run GaMD (OpenMM)**
```bash
gamdRunner xml input.xml -p CUDA -d 0 -o gamd
```

**Arguments:**
- `xml` → GaMD configuration file
- `-p CUDA` → GPU execution
- `-d 0` → device ID

> An example `input.xml` is provided in this repository.

---

## 📌 Notes & Best Practices

- Ensure **consistency between force field and solvent model**
- Validate energy minimization convergence before proceeding
- Check temperature stability during NVT steps
- For GaMD:
  - Proper boost parameter tuning is critical
  - Monitor boost potential statistics (σ₀, ΔV)

---

## 📂 Repository Structure (Expected)

```
.
├── mdp/
│   ├── ions.mdp
│   ├── minim.mdp
│   ├── nvt_*.mdp
│   └── prod_700.mdp
├── input.xml
└── README.md
```

---

## 🧪 GaMD Parameter Tuning Guidelines (σ₀P, σ₀D, Dual Boost)

Proper tuning of GaMD boost parameters is critical for **accurate reweighting** and **stable enhanced sampling**.

### 🔑 Key Quantities

- **ΔV (boost potential)**: Added to smooth the potential energy surface
- **σ(ΔV)**: Standard deviation of boost potential
- **σ₀**: Upper bound constraint for σ(ΔV)

Two boost components in **dual-boost GaMD**:
- **Dihedral boost (D)** → enhances local conformational transitions
- **Total potential boost (P)** → enhances global motions

---

### ⚙️ σ₀ Selection (Critical)

Typical recommended ranges:

- **σ₀P (total potential)**: `1–6 kcal/mol`
- **σ₀D (dihedral)**: `0.1–6 kcal/mol`

Empirical guidance:
- Start with **σ₀P = 1**, **σ₀D = 0.1**
- Increase till k reaches 1 individually run potential and dihedral for tuning the sigma, not dual.

---

---

### 🔄 Boost Scheme Selection

| Scheme        | When to Use |
|--------------|------------|
| **Dihedral only** | Small proteins, local folding transitions |
| **Dual boost**    | Recommended default; balances local + global sampling |
| **Total only**    | Rare; can distort thermodynamics if not tuned carefully |

---

### ⚠️ Practical Tips (Based on Common Issues)

- If **σ₀D always gives σ(ΔV)=σ₀ (e.g., 1.0)**:
  - Likely dihedral energy distribution is narrow
  - Try lowering σ₀D or check system flexibility

- If **reweighting fails (noisy PMF)**:
  - Reduce σ₀ values
  - Ensure sufficient sampling length

- If **Rg/RMSD space not explored**:
  - Increase σ₀P slightly
  - Ensure starting structure is sufficiently extended

- Always verify:
  - Boost potential distribution ~ near-Gaussian
  - No extreme ΔV outliers

---

### 📈 Validation Checklist

Before production analysis:

- [ ] σ(ΔV) within target bounds
- [ ] Smooth boost potential distribution
- [ ] Adequate sampling across CV space (RMSD, Rg)
- [ ] Reweighting produces physically meaningful PMF

---

## 🧠 Summary

This protocol:
- Uses **high-temperature unfolding** to generate diverse conformations
- Selects an **extended starting state**
- Applies **GaMD (dual boost recommended)** for enhanced sampling
- Requires **careful σ₀ tuning** for reliable thermodynamics

---
