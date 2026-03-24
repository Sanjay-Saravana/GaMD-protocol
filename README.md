# GaMD-protocol

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