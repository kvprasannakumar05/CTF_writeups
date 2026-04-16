# Take a Slice Writeup

I started this challenge with almost no type information beyond the name:

> It's in the name!

The flag format was `UMASS{...}`, and the provided file was just named `cake`.

That was already suggestive:

- `cake`
- `take-a-slice`

So I expected something geometric, volumetric, or otherwise recoverable by literally taking slices.

## Initial Triage

I first checked what the challenge directory contained.

### Command

```bash
ls -la /home/prasanna/umass/take-a-slice
```

### Output

```text
total 1928
drwxr-xr-x  2 prasanna prasanna    4096 Apr 11 12:10 .
drwxr-xr-x 11 prasanna prasanna    4096 Apr 11 12:10 ..
-rw-r--r--  1 prasanna prasanna 1960584 Apr 11 12:10 cake
-rw-r--r--  1 prasanna prasanna      25 Apr 11 12:10 cake:Zone.Identifier
```

The artifact had no extension, so I checked its file type.

### Command

```bash
file /home/prasanna/umass/take-a-slice/*
```

### Output

```text
/home/prasanna/umass/take-a-slice/cake:                 data
/home/prasanna/umass/take-a-slice/cake:Zone.Identifier: ASCII text, with CRLF line terminators
```

So `file` did not help directly.

## Looking at the Raw Bytes

My next step was to inspect the beginning of the file manually.

### Command

```bash
xxd -l 128 /home/prasanna/umass/take-a-slice/cake
```

### Output

```text
00000000: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000010: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000020: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000030: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000040: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000050: 2a99 0000 59a9 053f 24a9 05bf 78a4 2cbf  *...Y..?$...x.,.
00000060: f669 d740 6242 6140 2fd6 8b40 2a71 d940  .i.@bBa@/..@*q.@
00000070: 5a24 6540 63e7 8b40 bbb3 d940 3d3c 6440  Z$e@c..@...@=<d@
```

The first 80 bytes were all zeroes. After that, the data started looking like binary floats.

That pattern immediately made me suspect **binary STL**:

- 80-byte header
- 4-byte triangle count
- repeated 50-byte triangle records

## Confirming It Was a Binary STL

I parsed the expected STL fields directly.

### Command

```bash
python3 - <<'PY'
import struct
from pathlib import Path
b=Path('/home/prasanna/umass/take-a-slice/cake').read_bytes()
count=struct.unpack('<I', b[80:84])[0]
print('triangles', count, 'expected size', 84+50*count, 'actual', len(b))
mins=[1e9,1e9,1e9]; maxs=[-1e9,-1e9,-1e9]
for i in range(count):
    off=84+i*50
    vals=struct.unpack('<12fH', b[off:off+50])
    verts=[vals[3:6], vals[6:9], vals[9:12]]
    for v in verts:
        for j in range(3):
            mins[j]=min(mins[j], v[j]); maxs[j]=max(maxs[j], v[j])
print('bbox min', mins)
print('bbox max', maxs)
PY
```

### Output

```text
triangles 39210 expected size 1960584 actual 1960584
bbox min [-1.3470383882522583, -2.5399999618530273, 0.0]
bbox max [59.05500030517578, 43.105228424072266, 25.399999618530273]
```

That confirmed it:

- the file was a valid **binary STL**
- it contained **39,210 triangles**
- the size matched exactly

At this point the challenge name made sense. I probably needed to literally **take slices** through the 3D mesh.

## Rendering Simple Projections

Before slicing, I looked at the raw point projections to understand the model shape.

### Command

```bash
python3 - <<'PY'
import struct
from pathlib import Path
import numpy as np
import matplotlib.pyplot as plt
b=Path('/home/prasanna/umass/take-a-slice/cake').read_bytes()
count=struct.unpack('<I', b[80:84])[0]
verts=np.empty((count*3,3),dtype=np.float32)
for i in range(count):
    off=84+i*50
    vals=struct.unpack('<12fH', b[off:off+50])
    verts[i*3:(i+1)*3,:]=np.array([vals[3:6], vals[6:9], vals[9:12]],dtype=np.float32)
for (a,bn,name) in [(0,1,'xy'),(0,2,'xz'),(1,2,'yz')]:
    plt.figure(figsize=(8,6))
    plt.scatter(verts[:,a], verts[:,bn], s=0.01)
    plt.gca().set_aspect('equal')
    plt.title(name)
    plt.savefig(f'/tmp/cake_{name}.png', dpi=300, bbox_inches='tight')
    plt.close()
    print(f'/tmp/cake_{name}.png')
PY
```

### Output

```text
/tmp/cake_xy.png
/tmp/cake_xz.png
/tmp/cake_yz.png
```

Those projections clearly showed a **cake-like triangular wedge** with circular features, which matched the file name perfectly.

Saved copies in the challenge folder:

- [cake_xy.png](/home/prasanna/umass/take-a-slice/cake_xy.png)
- [cake_xz.png](/home/prasanna/umass/take-a-slice/cake_xz.png)
- [cake_yz.png](/home/prasanna/umass/take-a-slice/cake_yz.png)

## Taking Horizontal Slices

Since the challenge name explicitly told me to take a slice, I computed intersections of the STL triangles with a set of horizontal planes `z = constant`.

I wrote a small slicer:

- for each triangle
- intersect each edge with a chosen `z` plane
- if exactly two intersections occur, that gives one line segment in the slice
- plot all those segments for each height

### Command

```bash
python3 - <<'PY'
import struct
from pathlib import Path
import numpy as np
import matplotlib.pyplot as plt
b=Path('/home/prasanna/umass/take-a-slice/cake').read_bytes()
count=struct.unpack('<I', b[80:84])[0]
tris=np.empty((count,3,3),dtype=np.float32)
for i in range(count):
    off=84+i*50
    vals=struct.unpack('<12fH', b[off:off+50])
    tris[i,:,:]=np.array([vals[3:6], vals[6:9], vals[9:12]],dtype=np.float32)

def section_segments_z(tris, z):
    segs=[]
    for tri in tris:
        pts=[]
        for a,b in [(0,1),(1,2),(2,0)]:
            p1,p2=tri[a],tri[b]
            z1,z2=p1[2],p2[2]
            if (z1<z and z2<z) or (z1>z and z2>z) or (z1==z2):
                continue
            t=(z-z1)/(z2-z1)
            if 0<=t<=1:
                p=p1 + t*(p2-p1)
                pts.append(p[:2])
        if len(pts)==2:
            segs.append(np.array(pts))
    return segs

zs=np.linspace(0.2,13.4,12)
fig,axs=plt.subplots(3,4,figsize=(16,12))
for ax,z in zip(axs.ravel(),zs):
    segs=section_segments_z(tris,z)
    for s in segs:
        ax.plot(s[:,0], s[:,1], 'k-', linewidth=0.3)
    ax.set_title(f'z={z:.2f} segs={len(segs)}')
    ax.set_aspect('equal')
    ax.invert_yaxis()
plt.tight_layout()
out='/tmp/cake_z_slices_grid.png'
plt.savefig(out,dpi=200)
print(out)
PY
```

### Output

```text
/tmp/cake_z_slices_grid.png
```

Saved copy:

- [cake_z_slices_grid.png](/home/prasanna/umass/take-a-slice/cake_z_slices_grid.png)

## Finding the Hidden Region

Looking at the slice grid, most layers just showed the cake outline, but several **mid-height slices** contained a tiny extra feature in the upper-left interior.

That was the hidden payload.

The interesting region appeared roughly between:

- `z ~= 3.8`
- `z ~= 7.4`

So I zoomed that area and rendered a denser set of slices.

### Command

```bash
python3 - <<'PY'
import struct
from pathlib import Path
import numpy as np
import matplotlib.pyplot as plt
b=Path('/home/prasanna/umass/take-a-slice/cake').read_bytes()
count=struct.unpack('<I', b[80:84])[0]
tris=np.empty((count,3,3),dtype=np.float32)
for i in range(count):
    off=84+i*50
    vals=struct.unpack('<12fH', b[off:off+50])
    tris[i,:,:]=np.array([vals[3:6], vals[6:9], vals[9:12]],dtype=np.float32)

def section_segments_z(tris, z):
    segs=[]
    for tri in tris:
        pts=[]
        for a,b in [(0,1),(1,2),(2,0)]:
            p1,p2=tri[a],tri[b]
            z1,z2=p1[2],p2[2]
            if (z1<z and z2<z) or (z1>z and z2>z) or (z1==z2):
                continue
            t=(z-z1)/(z2-z1)
            if 0<=t<=1:
                p=p1 + t*(p2-p1)
                pts.append(p[:2])
        if len(pts)==2:
            segs.append(np.array(pts))
    return segs

zs=np.linspace(3.5,7.8,12)
fig,axs=plt.subplots(3,4,figsize=(16,10))
for ax,z in zip(axs.ravel(),zs):
    segs=section_segments_z(tris,z)
    for s in segs:
        if np.max(s[:,0]) < 20 and np.max(s[:,1]) < 10:
            ax.plot(s[:,0], s[:,1], 'k-', linewidth=1.2)
        elif np.max(s[:,0]) < 20 and np.min(s[:,1]) < 10:
            ax.plot(s[:,0], s[:,1], 'k-', linewidth=1.2)
    ax.set_xlim(4,18)
    ax.set_ylim(9,2)
    ax.set_title(f'z={z:.2f}')
    ax.set_aspect('equal')
plt.tight_layout()
out='/tmp/cake_hidden_z_zoom_grid.png'
plt.savefig(out,dpi=250)
print(out)
PY
```

### Output

```text
/tmp/cake_hidden_z_zoom_grid.png
```

That grid made it obvious that the hidden feature was **text embedded inside the model**, but the slices were mirrored / skewed depending on the cut height.

## Making the Text More Readable

I rendered a few of the clearest slice levels and also generated flipped/rotated variants so I could read the embedded string more easily.

The most useful saved images were:

- [slice_5.06_b.png](/home/prasanna/umass/take-a-slice/slice_5.06_b.png)
- [slice_5.45_b.png](/home/prasanna/umass/take-a-slice/slice_5.45_b.png)
- [slice_6.24_b.png](/home/prasanna/umass/take-a-slice/slice_6.24_b.png)
- [slice_5.06_b_rot90.png](/home/prasanna/umass/take-a-slice/slice_5.06_b_rot90.png)

The clearest transformed slice showed the string strongly enough to recover the flag:

`UMASS{SL1C3_&_D1C3}`

## Final Flag

`UMASS{SL1C3_&_D1C3}`

## Solve Summary

The file `cake` looked like raw binary data at first, but the zeroed 80-byte header plus matching triangle record count identified it as a **binary STL**. I parsed the mesh, plotted simple projections, and then literally followed the challenge name by taking horizontal **slices** through the model. Mid-height cross-sections exposed a small hidden text feature embedded inside the cake mesh. After zooming and transforming the clearest slices, the flag read as:

`UMASS{SL1C3_&_D1C3}`
