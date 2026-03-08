

# Rat Expedition
PLAYABLE WEB BUILD ON GITHUB PAGES: https://swampspawn.github.io/rat-expedition-web-build/

PLAYABLE WEB BUILD ON ITCH.IO: https://swampspawn.itch.io/rat-expedition

# Summary
A fun little game made completely from scratch in Blender, Godot and some other FOSS. First and foremost it's a portfolio game, so the focus at the early stage of development was firm techincal foundation. (Yet I can't leave it unfinished, so expect updates) It has an internal asset browser to check textures and wireframes, texture atlases, custom shaders and many more.

* Tech features:
  * [Chunk system](#chunk-system)
  * [Multimesh](#multimesh)
  * [Texture atlas](#texture-atlas)
  * [Asset browser](#asset-browser)
  * [Custom shaders](#custom-shaders)
  * [Animation tree](#animation-tree)

# Chunk system

https://github.com/user-attachments/assets/15c69884-c1e8-44bf-8282-4d04443e4493


One of the key features of gameplay is an endless street where the rat collects food. Naturally, the chunk system is a good way to do that. It works as such:
1. Constantly checks if player changed chunk. If so, initiates manage_chunk_tasks().

```gdscript
func _physics_process(delta: float) -> void:
	if player:
		player_chunk = get_chunk_coord(player.global_position) + 1
	if player_last_chunk != player_chunk:
		player_last_chunk = player_chunk
		signal_chunk_transition()
		chunk_changed.emit
		manage_chunk_tasks()
```
2. In manage_chunk_tasks() checks for chunks that need to load and need to be deleted, and appends them to corresponding arrays.
```gdscript
func manage_chunk_tasks() -> void:
	for x in range(0, LOAD_DISTANCE):
		for v in [-1,1]: #for negative variants of chunk coords
			var chunk_coord = x*v + player_chunk
			
			if not chunk_coord in loaded_chunks and not chunk_coord in chunk_load_queue:
				chunk_load_queue.append(chunk_coord)
		
	for chunk_coord:int in loaded_chunks:
		if abs(chunk_coord - player_chunk) > UNLOAD_DISTANCE and not chunk_coord in chunks_to_remove:
			if loaded_chunks[chunk_coord].richness != Global.Chunk_default_richness:
				Chunk_Richness.set(chunk_coord, loaded_chunks[chunk_coord].richness)
			chunks_to_remove.append(chunk_coord)
```
3. Processes these tasks in execute_chunk_tasks(): loads and deletes if there are free task slots.
```gdscript
func execute_chunk_tasks() -> void:
	for chunk_coord in chunks_to_remove:
		loaded_chunks[chunk_coord].queue_free()
		loaded_chunks.erase(chunk_coord)
		chunks_to_remove.erase(chunk_coord)
	if current_tasks < max_tasks and not chunk_load_queue.is_empty():
		current_tasks += 1
		var chunk_to_load:int = chunk_load_queue.pop_front()
		load_chunk(chunk_to_load)
```
4. Remembers where loaded chunks are and how many resources were collected in each with richness dictionary.
```gdscript
func load_chunk(chunk_coord: int):
	var new_chunk: World_Chunk = Regular_chunk_scene.instantiate()
	add_child(new_chunk, true)
	if Chunk_Richness.has(chunk_coord):
		new_chunk.richness = Chunk_Richness[chunk_coord]
	new_chunk.position.x = chunk_coord*Global.Chunk_size
	new_chunk.chunk_floored_coordinate = chunk_coord
	#new_chunk.check_particles()
	loaded_chunks[chunk_coord] = new_chunk
	new_chunk.get_node("Number").text = str(chunk_coord)
	current_tasks -= 1
	signal_chunk_transition()
```
5. It also emits a signal that player chunk has changed so each chunk can separately check if it has to draw collectibles multimesh.
```gdscript
func _physics_process(delta: float) -> void:
	if player:
		player_chunk = get_chunk_coord(player.global_position) + 1
	if player_last_chunk != player_chunk:
		player_last_chunk = player_chunk
		signal_chunk_transition()
```
Overall it seems robust, as the array operations are comparatively fast and each array syncs with others to avoid unaccounted chunks (had this problem in the previous project).

# Multimesh
As there can be many assets in chunk, a cool optimization technique is using multimesh - telling GPU to draw all instances at once instead of per-instance GPU->CPU communication. When loaded, each chunk checks the collectibles list in Global, then creates separate multimeshes and item groups with collisions for each chunk.
```gdscript
func _ready() -> void:
	for item in Global.Collectibles:
		var item_row = []
		Multimesh_tasks.append(item_row)
		var m_mesh = MultiMeshInstance3D.new()
		$Multimeshes.add_child(m_mesh)
		m_mesh.name = item.name
		m_mesh.multimesh = MultiMesh.new()
		m_mesh.multimesh.transform_format = MultiMesh.TRANSFORM_3D
```
Then, fetches random coordinates and item type and appends coordinates to a corresponding item array, while also creating collision items in scatter_objects()
```gdscript
func scatter_objects():
	while (richness - spawned_richness) > 101:
		var collectible_instance = Collectible_spawn.new()
		collectible_instance.coords = vec2_random_pos(food_area)
		collectible_instance.item_id = Global.item_pick()
		Multimesh_tasks[collectible_instance.item_id].append(collectible_instance)
		var item_collision = item_scene.instantiate()
		$Item_areas.add_child(item_collision)
		item_collision.item_id = collectible_instance.item_id
		item_collision.position = collectible_instance.coords
		item_collision.item_picked.connect(_on_item_item_picked)
		spawned_richness += Global.Collectibles[collectible_instance.item_id].food_points
```
Redraws with multimesh_redraw(), which places multimesh instance and collision item in the same spot.
```gdscript
func multimesh_redraw():
	var i:int = 0
	for col_type in Multimesh_tasks:
		var mm:MultiMeshInstance3D = get_node(str("Multimeshes/"+Global.Collectibles[i].name))
		mm.multimesh.instance_count = col_type.size()
		mm.multimesh.mesh = Global.Collectibles[i].mesh
		var j:int = 0
		for col in col_type:
			mm.multimesh.set_instance_transform(j, Transform3D(Basis(), col.coords))
			j += 1
		i += 1
```
When the item is hit, it erases coordinates from mm array and redraws it.
```gdscript
func _on_item_item_picked(id: int, coords: Vector3) -> void:
	for mminstance in Multimesh_tasks[id]:
		if coords == mminstance.coords:
			Multimesh_tasks[id].erase(mminstance)
			Global.add_food_points(Global.Collectibles[id].food_points)
			richness -= Global.Collectibles[id].food_points
	multimesh_redraw()
```
This systems allows drawing numerous collectibles cheaper, however, it is potentially subject to change in the future due to the fact that balancing the game might severely reduce the number of collectibles in a chunk. It might still be recycled for some kind of thrash-misc objects though.

# Texture atlas
This optimization technique somewhat shares the principle with multimesh, but in regard to textures. Instead of storing 12 textures (albedo and normals per collectible), all collectibles use the same 2 textures - for normal and albedo. The atlas was baked in Blender with all collectibles unwrapped simultaneously. This allows faster iterration (as the only thing you need to change to add new collectible is retopology UVs), utilizing atlas more effectively (as UVs can overlap and theres no need to allocate convex regions for collectible UVs), and storing less textures in memory (a general atlas bonus).


<img width="1024" height="1024" alt="Collectibles_albedo" src="https://github.com/user-attachments/assets/8095a4de-662d-426b-8137-0df218ddae9c" />
<img width="1024" height="1024" alt="Collectibles_normal" src="https://github.com/user-attachments/assets/70615f0c-dab7-431c-8efb-42f7e03d92d9" />



# Asset browser
Not only does it allow you to browse assets in different viewmodes to check textures, it also can show wireframe! To achieve that, once you've selected wireframe viewmode it tears the mesh apart with MeshDataTool, assigns barycentric coordinates for each face and assembles a new, de-indexed mesh. (meaning the mesh where no faces share vertices):
```gdscript
func wireframe_mesh(original_mesh: Mesh) -> ArrayMesh:
	var face_count: int = 0
	var vertex_count: int = 0
	
	var data_tool = MeshDataTool.new()
	var surface_count = original_mesh.get_surface_count()
	var new_mesh = ArrayMesh.new()
	
	
	for s in range(surface_count):
		data_tool.create_from_surface(original_mesh, s)

		var verts = PackedVector3Array()
		var colors = PackedColorArray()
		var normals = PackedVector3Array()
		
		face_count += data_tool.get_face_count()
		vertex_count += data_tool.get_vertex_count()
		
		for face in range(data_tool.get_face_count()):
			for i in range(3):
				var vertex_id = data_tool.get_face_vertex(face, i)
				verts.append(data_tool.get_vertex(vertex_id))
				normals.append(data_tool.get_vertex_normal(vertex_id))
				
				colors.append(Color(1,0,0) if i == 0 else (Color(0,1,0) if i == 1 else Color(0,0,1)))
		
		var arrays = []
		arrays.resize(Mesh.ARRAY_MAX)
		arrays[Mesh.ARRAY_VERTEX] = verts
		arrays[Mesh.ARRAY_COLOR] = colors
		arrays[Mesh.ARRAY_NORMAL] = normals
		
		new_mesh.add_surface_from_arrays(Mesh.PRIMITIVE_TRIANGLES, arrays)
```
Barycentric coodrinates are needed to establish whether a given fragment (pixel) is part of an edge (with configurable width) or not. The logic behind them is that every point in a triangle can be described by distances to the verts forming that triangle. If a distance from any point is more than some value, that means that the point is on the opposite edge from said vertex.  Coordinates are assigned during de-indexing for each vertex-face combinaton (one vertex in an indexed mesh contributes to more than one face) as a vertex color attribute, and then each combination is saved as a separate vertex (basically unique vertices for each face). Here, the logic of barycentric coordinates is inversed, though the principle stays the same: as they are coded in color attribute values, each channel actually describes the "closeness" to the according vertex. Afterwards the wireframe shader can finally function properly - for a given fragment if one of the three "closeness" values is less than width value, it means that the fragment is part of an edge (check in the asset inspector, the "green" edge is actually red-blue, as the green channel is near zero for all edge fragments)

```gdshader
void fragment() {
	if (view_mode == 0) {
		ALBEDO = texture(asset_albedo, UV).rgb;
		NORMAL_MAP = texture(asset_normal, UV).rgb;
	} else if (view_mode == 1) {
		ALBEDO = texture(asset_albedo, UV).rgb;
	} else if (view_mode == 2) {
		ALBEDO = texture(asset_normal, UV).rgb;
	} else if (view_mode == 3) {
    float is_edge = min(min(COLOR.r, COLOR.g), COLOR.b);
    if (is_edge > width) discard;
    ALBEDO = vec3(COLOR.rgb);
	EMISSION = vec3(COLOR.rgb);
}}
```

# Custom shaders
They are utilized not only for changing view modes in AB, but also for collectibles rotation, bobbing and emission. Skipping the more "standard" elements of reading textures using right UVs, for rotation and bobbing an actual matrice operation is needed, as the shader is completely oblivious to the existence of the whole object - it knows only one vertex at a time. Thus, if you want to rotate, you need to move, and matrice transform comes in handy. Skipping the cool math details, once you multiply a VERTEX vector by a matrice, you get a rotated variant of it. Once you add TIME (passed) as an argument, you get continious rotation. After adding said TIME once again, but simply to the VERTEX y element, you get a bobbing up-and-down effect. Use that same technique, but now multiply EMMISION taken from a negative normal map blue channel (which shows how much the region faces front) - and you get a glow only on sharper visual edges.

```gdshader
shader_type spatial;

global uniform sampler2D collectibles_albedo : source_color;
global uniform sampler2D collectibles_normal : hint_normal;
global uniform float collectibles_rotate_speed;
global uniform vec2 collectibles_bob;

mat4 rotate_y(float theta) {
    float c = cos(collectibles_rotate_speed * TIME);
    float s = sin(collectibles_rotate_speed * TIME);
    return mat4(
        vec4(c, 0, -s, 0),
        vec4(0, 1, 0, 0),
        vec4(s, 0, c, 0),
        vec4(0, 0, 0, 1)
    );
}

void vertex() {
	VERTEX = (vec4(VERTEX, 1.0) * rotate_y(1)).xyz;
	VERTEX.y += cos(collectibles_bob.x*TIME)*collectibles_bob.y;
}

void fragment() {
	ALBEDO = texture(collectibles_albedo, UV).rgb;
	NORMAL_MAP = texture(collectibles_normal, UV).rgb;
	EMISSION = texture(collectibles_albedo, UV).rgb * vec3(abs(sin(TIME*2.)), abs(sin(TIME*2.)), abs(sin(TIME*3.))) * (1. - texture(collectibles_normal, UV).b) * 8.;
}
```

# Animation tree
It is in a primitive state right now, purely for prototyping/planning for the future updates. Still, it already uses player speed\direction to blend between different walking animations :)
