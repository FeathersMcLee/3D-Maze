# 3D-Maze
# 3D maze generator Python script for Blender objects.
import bpy
import bmesh
import random
from mathutils import Vector, Matrix

# -----------------------
# Parameters
cell_size = 1.0
padding = 0.0
seed = 42
num_walkers = 30
turn_chance = 0.25
feeler_range = 1  # "radius" of the feeler grid: 1 = 3x3, 2 = 5x5, etc.
# -----------------------

random.seed(seed)

# --- Setup / bounding box ---
obj = bpy.context.active_object
if obj is None or obj.type != 'MESH':
    raise RuntimeError("Select a mesh object (the bounding volume) before running.")

bbox_size = Vector(obj.dimensions)
nx = int((bbox_size.x - 2*padding) // cell_size)
ny = int((bbox_size.y - 2*padding) // cell_size)
nz = int((bbox_size.z - 2*padding) // cell_size)
if min(nx, ny, nz) < 3:
    raise RuntimeError("Bounding object too small for given cell_size/padding.")

world_center = obj.matrix_world @ Vector((0, 0, 0))
grid_total = Vector((nx * cell_size, ny * cell_size, nz * cell_size))
origin = world_center - (grid_total / 2.0)

dirs = [(1,0,0), (-1,0,0), (0,1,0), (0,-1,0), (0,0,1), (0,0,-1)]
occupied = set()
cells = []

def in_bounds(x, y, z):
    return 0 <= x < nx and 0 <= y < ny and 0 <= z < nz

def free_neighbors(x, y, z):
    return [(dx,dy,dz) for dx,dy,dz in dirs
            if in_bounds(x+dx, y+dy, z+dz)
            and (x+dx, y+dy, z+dz) not in occupied]

def is_area_clear(x, y, z, direction):
    """
    Check a cube area in front of walker.
    Ignore out-of-bounds cells — only occupied ones matter.
    """
    dx, dy, dz = direction
    fx, fy, fz = x + dx, y + dy, z + dz

    for ix in range(-feeler_range, feeler_range+1):
        for iy in range(-feeler_range, feeler_range+1):
            for iz in range(-feeler_range, feeler_range+1):
                # limit feelers to the forward-facing plane
                cx = fx + (ix if dx == 0 else 0)
                cy = fy + (iy if dy == 0 else 0)
                cz = fz + (iz if dz == 0 else 0)
                if not in_bounds(cx, cy, cz):
                    continue  # ignore out-of-bounds
                if (cx, cy, cz) in occupied:
                    return False
    return True

def near_boundary(x, y, z, margin=1):
    """Check if near any boundary."""
    return (
        x < margin or y < margin or z < margin or
        x > nx - 1 - margin or y > ny - 1 - margin or z > nz - 1 - margin
    )

# --- Walker simulation ---
for w in range(num_walkers):
    x = random.randrange(nx)
    y = random.randrange(ny)
    z = random.randrange(nz)
    direction = random.choice(dirs)
    dead = False

    while not dead:
        if (x, y, z) not in occupied:
            occupied.add((x, y, z))
            cells.append((x, y, z))

        options = free_neighbors(x, y, z)
        if not options:
            break

        clear_dirs = [d for d in options if is_area_clear(x, y, z, d)]
        if not clear_dirs:
            break

        forward = (x + direction[0], y + direction[1], z + direction[2])
        forward_free = (
            in_bounds(*forward)
            and forward not in occupied
            and is_area_clear(x, y, z, direction)
        )

        # Encourage turning when near edges
        edge_bias = 0.5 if near_boundary(x, y, z) else 0.0
        if not forward_free or random.random() < (turn_chance + edge_bias):
            choices = [d for d in clear_dirs if d != (-direction[0], -direction[1], -direction[2])]
            if not choices:
                break
            direction = random.choice(choices)

        x += direction[0]
        y += direction[1]
        z += direction[2]

# --- Geometry creation ---
bm = bmesh.new()

def cell_center(i, j, k):
    return origin + Vector(((i + 0.5) * cell_size,
                            (j + 0.5) * cell_size,
                            (k + 0.5) * cell_size))

for (i, j, k) in cells:
    M = Matrix.Translation(cell_center(i, j, k))
    bmesh.ops.create_cube(bm, size=cell_size, matrix=M)

bmesh.ops.remove_doubles(bm, verts=bm.verts, dist=1e-6)

mesh_data = bpy.data.meshes.new("FeelerSpaghettiMesh")
bm.to_mesh(mesh_data)
bm.free()

maze_obj = bpy.data.objects.new("FeelerSpaghetti", mesh_data)
bpy.context.collection.objects.link(maze_obj)
maze_obj.select_set(True)
bpy.context.view_layer.objects.active = maze_obj
mesh_data.update()

print(f"Generated FeelerSpaghetti: {len(cells)} cubes, {num_walkers} walkers, cell_size={cell_size}, feeler_range={feeler_range}")
