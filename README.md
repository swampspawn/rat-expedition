# Rat Expedition
PLAYABLE BUILD ON GITHUB PAGES: https://swampspawn.github.io/rat-expedition-web-build/

PLAYABLE BUILD ON ITCH.IO: https://swampspawn.itch.io/rat-expedition

# Summary
A fun little game made completely from scratch in Blender, Godot and some other FOSS. First and foremost it's a portfolio game, so the focus at the early stage of development was firm techincal foundation. (Yet I can't leave it unfinished, so expect updates) It has an internal asset browser to check textures and wireframes, texture atlases, custom shaders and many more.

# Tech features:
* Chunk system
* Multimesh
* Texture atlas
* Animation tree
* Asset browser
* Custom shaders

# Chunk system

https://github.com/user-attachments/assets/15c69884-c1e8-44bf-8282-4d04443e4493


One of the key features of gameplay is an endless street where the rat collects food. Naturally, the chunk system is a good way to do that. It works as such:
1 Constantly checks if player changed chunk. If so, initiates manage_chunk_tasks().
2 In manage_chunk_tasks() checks for chunks that need to load and need to be deleted, and appends them to corresponding arrays.
3 Processes these tasks in execute_chunk_tasks(): loads and deletes if there are free task slots.
4 Remembers where loaded chunks are and how many resources were collected in each with richness dictionary.
5 It also emits a signal that player chunk has changed so each chunk can separately check if it has to draw collectibles multimesh.
Overall it seems robust, as the array operations are comparatively fast and each array syncs with others to avoid unaccounted chunks (had this problem in the previous project).

# Multimesh
As there can be many assets in chunk, a cool optimization technique is using multimesh - telling GPU to draw all instances at once instead of per-instance GPU->CPU communication. When loaded, each chunk checks the collectibles list in Global, then creates separate multimeshes and item groups with collisions for each chunk. Then, fetches random coordinates and item type and appends coordinates to a corresponding item array, while also creating collision items in scatter_objects():. Redraws with multimesh_redraw(), which places multimesh instance and collision item in the same spot. When the item is hit, it erases coordinates from mm array and redraws it. This systems allows drawing numerous collectibles cheaper, however, it is potentially subject to change in the future due to the fact that balancing the game might severely reduce the number of collectibles in a chunk. It might still be recycled for some kind of thrash-misc objects though.

#Texture atlas
This optimization technique somewhat shares the principle with multimesh, but in regard to textures. Instead of storing 12 textures (albedo and normals per collectible), 

# Asset browser
Not only does it allow you to browse assets in different viewmodes to check textures, it also can show wireframe! To achieve that, once you've selected wireframe viewmode it tears the mesh apart with MeshDataTool, assigns barycentric coordinates for each face and assembles a new, de-indexed mesh. (meaning the mesh where no faces share vertices). Barycentric coodrinates are needed to establish whether a given fragment (pixel) is part of an edge (with configurable width) or not. The logic behind them is that every point in a triangle can be described by distances to the verts forming that triangle. If a distance from any point is more than some value, that means that the point is on the opposite edge from said vertex.  Coordinates are assigned during de-indexing for each vertex-face combinaton (one vertex in an indexed mesh contributes to more than one face) as a vertex color attribute, and then each combination is saved as a separate vertex (basically unique vertices for each face). Here, the logic of barycentric coordinates is inversed, though the principle stays the same: as they are coded in color attribute values, each channel actually describes the "closeness" to the according vertex. Afterwards the wireframe shader can finally function properly - for a given fragment if one of the three "closeness" values is less than width value, it means that the fragment is part of an edge (check in the asset inspector, the "green" edge is actually red-blue, as the green channel is near zero for all edge fragments)

# Custom shaders
They are utilized not only for changing view modes in AB, but also for collectibles rotation, bobbing and emission. Skipping the more "standard" elements of reading textures using right UVs, for rotation and bobbing an actual matrice operation is needed, as the shader is completely oblivious to the existence of the whole object - it knows only one vertex at a time. Thus, if you want to rotate, you need to move, and matrice transform comes in handy. Skipping the cool math details, once you multiply a VERTEX vector by a matrice, you get a rotated variant of it. Once you add TIME (passed) as an argument, you get continious rotation. After adding said TIME once again, but simply to the VERTEX y element, you get a bobbing up-and-down effect. Use that same technique, but now multiply EMMISION taken from a negative normal map blue channel (which shows how much the region faces front) - and you get a glow only on sharper visual edges.

# Animation tree
It is in a primitive state right now, purely for prototyping/planning for the future updates. Still, it already uses player speed\direction to blend between different walking animations :)
