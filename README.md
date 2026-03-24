# GaMD-protocol

### **Step 1:** get PDB structure:
```bash
wget https://files.rcsb.org/download/XXXX.pdb
```

> [!NOTE]
> XXXX &rarr; PDB ID

### **Step 2:** Generate Topology:
```bash
gmx pdb2gmx -f 1UAO.pdb -o processed.gro -ignh
```
**Choose:**
* Force field
* Water

**Output:**
* processed.gro
* topol.top