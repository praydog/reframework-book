This is the REFramework wiki. It will mostly serve as documentation for the scripting and plugin system.

[VR Troubleshooting](troubleshooting/VR-Troubleshooting.md)

[Contributing to documentation](https://github.com/cursey/reframework-book)

[Nightly builds](https://github.com/praydog/REFramework-nightly/releases)

## Reporting a bug
Report it on the [Issues](https://github.com/praydog/REFramework/issues) page.

If you are crashing, or are having a technical problem then upload these files from your game folder:
* `re2_framework_log.txt` - The WHOLE LOG, not snippets of it.
* `reframework_crash.dmp` if you are crashing

## Help! My pirated copy does not work

Contrary to some belief, **this mod does not contain any anti-piracy checks of any kind**. Pirated copies just do not receive support. If it works then, it works. If not, additional support is not going to be added.

## Lua Scripting

REFramework comes with a scripting system using Lua. 

Leveraging the RE Engine's IL2CPP implementation, REFramework gives developers powerful control over the game engine.

If you are interested in native plugins: [read the plugin section](#plugins)

## Loading a script

### Manual Loading
Click on `ScriptRunner` from the main REFramework menu. From there, press `Run Script` and locate the corresponding `*.lua` file you wish to load.

### Automatic Loading
Create an `reframework/autorun` folder in your game directory. This is automatically created when REFramework loads. REFramework will automatically load whatever `*.lua` scripts are in here during initialization.

## Handling Lua errors

### During script startup
When a Lua error occurs here, a MessageBox will pop up explaining what the error is.

### During callback execution
When a Lua error occurs here, the reason will be written to a debug log. ~~[DebugView](https://docs.microsoft.com/en-us/sysinternals/downloads/debugview) is required to view these.~~ In newer nightly builds, the errors can be viewed directly within the ScriptRunner window.

We don't pop a MessageBox here so the user doesn't lock their game.

## Finding game functions to call, and fields to grab
Use the `ObjectExplorer`. It can be found under `DeveloperTools` in the REFramework menu.

Poke around the singletons until you find something you're interested in. 

Objects under `Singletons` can be obtained with `sdk.get_managed_singleton("name")`

Objects under `Native Singletons` can be obtained with `sdk.get_native_singleton("name")`

Do note that the `Singletons` (AKA Managed Singletons) are the usually the most exposed. They were originally written in C#.

`Native Singletons` have fields and methods exposed, but they are usually hand picked. These ones were written in C++, and have the least amount of data exposed about them.

Anything under `TDB Methods` or `TDB Fields` of something within the `ObjectExplorer` can be called or grabbed using the various call and field getter/setter methods found here in the wiki. 

You **CANNOT** use the `Reflection Methods` or `Reflection Properties` yet without direct memory reading/writing, only the TDB versions are fully supported.

Good APIs to start on: [sdk](https://github.com/praydog/REFramework/wiki/sdk) and [re](https://github.com/praydog/REFramework/wiki/re)

[Example Scripts](examples/Example-Scripts.md)

[Further Object Explorer Documentation](object_explorer/object_explorer.md)

---

## Plugins
REFramework has the ability to run native DLL plugins. This can also just be used as a loose DLL loader, with no awareness of REF.

The plugins can perform much of what Lua can, with much more freedom. They have access to much of the important SDK functionality of REFramework, as well as useful callbacks for rendering/input/game code.

### Loading a plugin
Drop the `.dll` file into the `reframework/plugins` directory.

### Using the plugin API
Include the `include` directory from the root REFramework project directory in your plugin. Include `API.hpp` if you are using C++ and want a more C++ approach to using the SDK.

From there, you have the option to export these functions:

```cpp
// OPTIONAL
// Enforces plugin versioning
// If REF's major version does not match the plugin's required version, the plugin will not load
// If REF's minor version is less than the plugin's required version, the plugin will not load
// If REF's patch version does not match, the plugin will load but a warning will be displayed in the plugin menu
extern "C" __declspec(dllexport) void reframework_plugin_required_version(REFrameworkPluginVersion* version) {
    version->major = REFRAMEWORK_PLUGIN_VERSION_MAJOR;
    version->minor = REFRAMEWORK_PLUGIN_VERSION_MINOR;
    version->patch = REFRAMEWORK_PLUGIN_VERSION_PATCH;

    // Optionally, specify a specific game name that this plugin is compatible with.
    //version->game_name = "MHRISE";
}
```

```cpp
using namespace reframework; // For API class

// OPTIONAL
// Used for initializing the REFramework SDK and additional functions
extern "C" __declspec(dllexport) bool reframework_plugin_initialize(const REFrameworkPluginInitializeParam* param) {
    API::initialize(param);

    // Example usage of param functions
    const auto functions = param->functions;
    functions->on_lua_state_created(on_lua_state_created);
    functions->on_lua_state_destroyed(on_lua_state_destroyed);
    functions->on_frame(on_frame);
    functions->on_pre_application_entry("BeginRendering", on_pre_begin_rendering); // Look at via.ModuleEntry or the wiki for valid names here
    functions->on_post_application_entry("EndRendering", on_post_end_rendering);
    functions->on_device_reset(on_device_reset);
    functions->on_message((REFOnMessageCb)on_message);
    functions->log_error("%s %s", "Hello", "error");
    functions->log_warn("%s %s", "Hello", "warning");
    functions->log_info("%s %s", "Hello", "info");
    API::get()->log_error("%s %s", "Hello", "error");
    API::get()->log_warn("%s %s", "Hello", "warning");
    API::get()->log_info("%s %s", "Hello", "info");

    return true;
}
```

Optionally, you can specify a DllMain if for example your plugin absolutely needs to load immediately, or do not want the additional functionality of REFramework's plugin API.

### Example of using native SDK functionality
```cpp
// Grabbing the game window size with a C++ call and invoke
auto& api = API::get();
const auto tdb = api->tdb();

auto vm_context = api->get_vm_context();

const auto scene_manager = api->get_native_singleton("via.SceneManager");
const auto scene_manager_type = tdb->find_type("via.SceneManager");

const auto scene_manager_full_name = scene_manager_type->get_full_name();

OutputDebugString((std::stringstream{} << scene_manager_full_name << " Size: " << scene_manager_full_name.size()).str().c_str());

const auto get_main_view = scene_manager_type->find_method("get_MainView");
const auto main_view = get_main_view->call<API::ManagedObject*>(vm_context, scene_manager);

if (main_view == nullptr) {
    return;
}

// Method 1: Call
float size[3]{};
main_view->call("get_Size", &size, vm_context, main_view);

// Method 2: Invoke
auto get_size_invoke_result = main_view->invoke("get_Size", {});
float* size_invoke = (float*)&get_size_invoke_result;
```

### Example plugins
[REFramework Example Plugin](https://github.com/praydog/REFramework/tree/master/examples)

[REFramework Direct2D Plugin](https://github.com/cursey/reframework-d2d)

### Plugin headers
[REFramework Plugin API Headers](https://github.com/praydog/REFramework/tree/master/include/reframework)

## Side notes
Everything is subject to change and maybe refactored over time.

Lua uses a shared state across all scripts. Use `local` variables so as to not cause conflicts with other scripts.

If there's something you find you can't do without native code, Lua can `require` native DLLs. Native plugins are also an option.

RE Engine's IL2CPP implementation is not the same as Unity's. While RE Engine and Unity have many similarities, they are not the same, and no existing tooling for Unity or IL2CPP will work on the RE Engine.

C# scripting maybe a possibility in the future for more natural interaction with the engine, but is not currently being looked at. REFramework is open source, so any developer wishing to do that can try.

While REFramework's scripting API can do all sorts of things, [RE_RSZ](https://github.com/alphazolam/RE_RSZ) is another powerful tool that may be more suitable in some scenarios. 
For example:
* Inserting/cloning more game objects into a specific scene
* Edits that don't require runtime awareness of the game's state
* Time-sensitive edits to files (like something REFramework can't capture during startup)
* Using it for base edits with an REFramework script on top for additional functionality
* Changes that are impossible with REFramework's scripting system

## Thanks
[cursey](https://github.com/cursey/) for helping build the scripting system.

[The Hitchhiker](https://github.com/youwereeatenbyalid/) for testing/writing scripts/finding bugs/helpful suggestions.

[alphaZomega](https://github.com/alphazolam) for testing/writing scripts/finding bugs/helpful suggestions.

## Discords
[Modding Haven](https://discord.gg/9Vr2SJ3) (General RE Engine modding)

[Infernal Warks](https://discord.com/invite/nX5EzVU) (DMC5 modding)

[Monster Hunter Modding Discord](https://discord.gg/gJwMdhK)

[Flatscreen to VR Modding Discord](http://flat2vr.com)


# Summary

[Introduction](README.md)

# Troubleshooting

- [VR Troubleshooting](troubleshooting/VR-Troubleshooting.md)

# Examples

- [Example Scripts](examples/Example-Scripts.md)
- [Example Snippets](examples/Example-Snippets.md)

# Tools
- [Object Explorer](object_explorer/object_explorer.md)
- [Chain Viewer](chain_viewer/chain_viewer.md)
- [Behavior Tree/FSM Editor](bhvt_editor/bhvt_editor.md)

# Lua Scripting Reference
- [General](api/general/README.md)
    - [Notes on Return Types](api/general/Notes-on-Return-Types.md)
    - [Notes on Method Arguments](api/general/Notes-on-Method-Arguments.md)
    - [Best Practices](api/general/best-practices.md)
- [APIs](api/README.md)
    - [draw](api/draw.md)
    - [fs](api/fs.md)
    - [imgui](api/imgui.md)
    - [imnodes](api/imnodes.md)
    - [imguizmo](api/imguizmo.md)
    - [json](api/json.md)
    - [log](api/log.md)
    - [re (Callbacks)](api/re.md)
    - [reframework](api/reframework.md)
    - [sdk](api/sdk.md)
    - [vrmod](api/vrmod.md)
    - [object_explorer](api/object_explorer.md)
    - [thread](api/thread.md)
- [Types](api/types/README.md)
    - [VectorXf](api/types/VectorXf.md)
    - [Matrix4x4f](api/types/Matrix4x4f.md)
    - [Quaternion](api/types/Quaternion.md)
    - [REComponent](api/types/REComponent.md)
    - [REField](api/types/REField.md)
    - [REManagedObject](api/types/REManagedObject.md)
    - [REMethodDefinition](api/types/REMethodDefinition.md)
    - [RETypeDefinition](api/types/RETypeDefinition.md)
    - [RETransform](api/types/RETransform.md)
    - [REResource](api/types/REResource.md)
    - [SystemArray](api/types/SystemArray.md)
    - [ValueType](api/types/ValueType.md)

# C# Scripting Reference
- [General](api_cs/general/README.md)

Methods to be used on `re.on_frame` or `re.on_draw_ui`.

If you need more rendering functionality, check out the REFramework plugin [reframework-d2d](https://github.com/cursey/reframework-d2d)

## Methods
### `draw.world_to_screen(world_pos)`
Returns an optional `Vector2f` corresponding to the 2D screen position. Returns `nil` if `world_pos` is not visible.
### `draw.world_text(text, 3d_pos, color)`
### `draw.text(text, x, y, color)`
### `draw.filled_rect(x, y, w, h, color)`
### `draw.outline_rect(x, y, w, h, color)`
### `draw.line(x1, y1, x2, y2, color)`
### `draw.outline_circle(x, y, radius, color, num_segments)`
### `draw.filled_circle(x, y, radius, color, num_segments)`
### `draw.outline_quad(x1, y1, x2, y2, x3, y3, x4, y4, color)`
### `draw.filled_quad(x1, y1, x2, y2, x3, y3, x4, y4, color)`
### `draw.sphere(world_pos, radius, color, outline)`
Draws a 3D sphere with a 2D approximation in world space.

### `draw.capsule(world_start_pos, world_end_pos, radius, color, outline)`
Draws a 3D capsule with a 2D approximation in world space.

### `draw.gizmo(unique_id, matrix, operation, mode)`
* `unique_id`, an int64 that must be unique for every gizmo. Usually an address of an object will work. The same ID will control multiple gizmos with the same ID.
* `matrix`, the Matrix4x4f the gizmo is modifying.
* `operation`, defaults to UNIVERSAL. Use `imgui.ImGuizmoOperation` enum.
* `mode`, defaults to WORLD. WORLD or LOCAL. Use `imgui.ImGuizmoMode` enum.

Returns a tuple of `changed`, `mat`. Mat is the modified `matrix` that was passed.

```
    imgui.new_enum("ImGuizmoOperation", 
                    "TRANSLATE", ImGuizmo::OPERATION::TRANSLATE, 
                    "ROTATE", ImGuizmo::OPERATION::ROTATE,
                    "SCALE", ImGuizmo::OPERATION::SCALE,
                    "SCALEU", ImGuizmo::OPERATION::SCALEU,
                    "UNIVERSAL", ImGuizmo::OPERATION::UNIVERSAL);
    imgui.new_enum("ImGuizmoMode", 
                    "WORLD", ImGuizmo::MODE::WORLD,
                    "LOCAL", ImGuizmo::MODE::LOCAL);
```

Example video

<video width="640" height="480" controls>
<source src="https://user-images.githubusercontent.com/2909949/176351319-c070b216-fe71-4eb9-84f2-46c665892b11.mp4" type="video/mp4">
</video>

### `draw.cube(matrix)`

### `draw.grid(matrix, size)`


This is the filesystem API. ~~REFramework purposefully restricts scripts from the usual Lua `io` API so that scripts do not have unrestricted access to a users system.~~ Instead, this API focuses specifically on the `reframework/data` subdirectory.

In newer builds of REFramework, the `io` API is fully supported, but it can only access the `reframework/data` directory and the stdout/in/err streams. `io.popen` is also removed.

The `io` APIs have access to the `natives` directory via the `$natives` token passed at the start of the filepath string for any of the functions.

## Methods

### `fs.glob(filter)`
Returns a table of file paths that match the `filter`. `filter` should be a regex string for the files you wish to match.

#### Example

```lua
-- Get my mods JSON files.
local json_files = fs.glob([[my-cool-mod\\.*json]])

-- Iterate over them.
for k, v in ipairs(json_files) do
    -- v will be something like `my-cool-mod\config-file-1.json` 
end
```

### `fs.read(filename)`
Reads `filename` and returns the data as a string.

### `fs.write(filename, data)`
Writes `data` to `filename`. `data` is a string.


Bindings for ImGui. Can be used in the `re.on_draw_ui` callback.

Some methods can be used in `re.on_frame` if `begin_window`/`end_window` is used.

Example:
```
local thing = 1
local things = { "hi", "hello", "howdy", "hola", "aloha" }

re.on_draw_ui(function()
    if imgui.button("Whats up") then 
        thing = 1
    end

    local changed, new_thing = imgui.combo("Greeting", thing, things) 
    if changed then thing = new_thing end
end)
```

## Methods
### `imgui.begin_window(name, open, flags)`
Creates a new window with the title of `name`.

`open` is a bool. Can be `nil`. If not `nil`, a close button will be shown in the top right of the window.

`flags` - ImGuiWindowFlags.

`begin_window` must have a corresponding `end_window` call.

This function may only be called in `on_frame`, not `on_draw_ui`.

Returns a bool. Returns `false` if the user wants to close the window.

### `imgui.end_window()`
Ends the last `begin_window` call. Required.

### `imgui.begin_child_window(size, border, flags)`
`size` - Vector2f

`border` - bool

`flags` - ImGuiWindowFlags

### `imgui.end_child_window()`
### `imgui.begin_group()`
### `imgui.end_group()`

### `imgui.begin_rect()`
### `imgui.end_rect(additional_size, rounding)`
These two methods draw a rectangle around the elements between `begin_rect` and `end_rect`

### `imgui.button(label)`
Draws a button with `label` text.

Returns `true` when the user presses the button.

### `imgui.small_button(label)`

### `imgui.invisible_button(id, size, flags)`
`size` is a Vector2f or a size 2 array.

### `imgui.arrow_button(id, dir)`
`dir` is an `ImguiDir`

### `imgui.text(text)`
Draws text.

### `imgui.text_colored(text, color)`
Draws text with color.

`color` is an integer color in the form ARGB.

### `imgui.checkbox(label, value)`
Returns a tuple of `changed`, `value`

### `imgui.drag_float(label, value, speed, min, max, display_format (optional))`
Returns a tuple of `changed`, `value`

### `imgui.drag_float2(label, value (Vector2f), speed, min, max, display_format (optional))`
Returns a tuple of `changed`, `value`

### `imgui.drag_float3(label, value (Vector3f), speed, min, max, display_format (optional))`
Returns a tuple of `changed`, `value`

### `imgui.drag_float4(label, value (Vector4f), speed, min, max, display_format (optional))`
Returns a tuple of `changed`, `value`

### `imgui.drag_int(label, value, speed, min, max, display_format (optional))`
Returns a tuple of `changed`, `value`

### `imgui.slider_float(label, value, min, max, display_format (optional))`
Returns a tuple of `changed`, `value`

### `imgui.slider_int(label, value, min, max, display_format (optional))`
Returns a tuple of `changed`, `value`

### `imgui.input_text(label, value, flags (optional))`
Returns a tuple of `changed`, `value`, `selection_start`, `selection_end`

### `imgui.input_text_multiline(label, value, size, flags (optional))`
Returns a tuple of `changed`, `value`, `selection_start`, `selection_end`

### `imgui.combo(label, selection, values)`
Returns a tuple of `changed, value`. 

`changed` = true when selection changes. 

`value` is the selection index within `values` (a table)

`values` can be a table with any type of keys, as long as the values are strings.

### `imgui.color_picker(label, color, flags)`

Returns a tuple of `changed`, `value`. `color` is an integer color in the form ABGR which `imgui` and `draw` APIs expect.

### `imgui.color_picker_argb(label, color, flags)`

Returns a tuple of `changed`, `value`. `color` is an integer color in the form ARGB.

### `imgui.color_picker3(label, color (Vector3f), flags)`

Returns a tuple of `changed`, `value`

### `imgui.color_picker4(label, color (Vector4f), flags)`

Returns a tuple of `changed`, `value`

### `imgui.color_edit(label, color, flags)`

Returns a tuple of `changed`, `value`. `color` is an integer color in the form ABGR which `imgui` and `draw` APIs expect.

### `imgui.color_edit_argb(label, color, flags)`

Returns a tuple of `changed`, `value`. `color` is an integer color in the form ARGB.

### `imgui.color_edit3(label, color (Vector3f), flags)`

Returns a tuple of `changed`, `value`

### `imgui.color_edit4(label, color (Vector4f), flags)`

Returns a tuple of `changed`, `value`

`flags` for `color_picker/edit` APIs: `ImGuiColorEditFlags`

### `imgui.tree_node(label)`
### `imgui.tree_node_ptr_id(id, label)`
### `imgui.tree_node_str_id(id, label)`
### `imgui.tree_pop()`
All of the above `tree` functions must have a corresponding `tree_pop`!

### `imgui.same_line()`
### `imgui.spacing()`
### `imgui.new_line()`
### `imgui.is_item_hovered(flags)`
### `imgui.is_item_active()`
### `imgui.is_item_focused()`

### `imgui.collapsing_header(name)`

### `imgui.load_font(filepath, size, [ranges])`
Loads a font file from the `reframework/fonts` subdirectory at the specified `size` with optional Unicode `ranges` (an array of start, end pairs ending with 0). Returns a handle for use with `imgui.push_font()`. If `filepath` is nil, it will load the default font at the specified size.

### `imgui.push_font(font)`
Sets the font to be used for the next set of ImGui widgets/draw commands until `imgui.pop_font` is called.

### `imgui.pop_font()`
Unsets the previously pushed font.

### `imgui.get_default_font_size()`
Returns size of the default font for REFramework's UI.

### `imgui.set_next_window_pos(pos (Vector2f or table), condition, pivot (Vector2f or table))`
`condition` is the `ImGuiCond` enum.

```cpp
enum ImGuiCond_
{
    ImGuiCond_None          = 0,        // No condition (always set the variable), same as _Always
    ImGuiCond_Always        = 1 << 0,   // No condition (always set the variable)
    ImGuiCond_Once          = 1 << 1,   // Set the variable once per runtime session (only the first call will succeed)
    ImGuiCond_FirstUseEver  = 1 << 2,   // Set the variable if the object/window has no persistently saved data (no entry in .ini file)
    ImGuiCond_Appearing     = 1 << 3    // Set the variable if the object/window is appearing after being hidden/inactive (or the first time)
};
```

### `imgui.set_next_window_size(size (Vector2f or table), condition)`
`condition` is the `ImGuiCond` enum.

### `imgui.push_id(id)`
`id` can be an `int`, `const char*`, or `void*`.

### `imgui.pop_id()`

### `imgui.get_id()`

### `imgui.get_mouse()`
Returns a `Vector2f` corresponding to the user's mouse position in window space.

### `imgui.progress_bar(progress, size, overlay)`
`progress` is a float between 0 and 1.

`size` is a `Vector2f` or a size 2 array.

`overlay` is a string on top of the progress bar.

```lua
local progress = 0.0

re.on_frame(function()
    progress = progress + 0.001
    if progress > 1.0 then 
        progress = 0.0
    end
end)

re.on_draw_ui(function()
    imgui.progress_bar(progress, Vector2f.new(200, 20), string.format("Progress: %.1f%%", progress * 100))
end)
```

### `imgui.item_size(pos, size)`

### `imgui.item_add(pos, size)`

Adds an item with the specified position and size to the current window.

### `imgui.draw_list_path_clear()`

Clears the current window's draw list path.

### `imgui.draw_list_path_line_to(pos)`

Adds a line to the current window's draw list path given the specified `pos`

- `pos` is a `Vector2f` or a size 2 array.

### `imgui.draw_list_path_stroke(color, closed, thickness)`

Strokes the current window's draw list path with the specified `color`, `closed` state, and `thickness`.

- `color` is an integer color in the form ARGB.
- `closed` is a bool.
- `thickness` is a float.

### `imgui.get_key_index(imgui_key)`

Returns the index of the specified `imgui_key`.
### `imgui.is_key_down(key)`

Returns true if the specified `key` is currently being held down.
### `imgui.is_key_pressed(key)`

Returns true if the specified `key` was pressed during the current frame.
### `imgui.is_key_released(key)`

Returns true if the specified `key` was released during the current frame.
### `imgui.is_mouse_down(button)`

Returns true if the specified mouse `button` is currently being held down.
### `imgui.is_mouse_clicked(button)`

Returns true if the specified mouse `button` was clicked during the current frame.
### `imgui.is_mouse_released(button)`

Returns true if the specified mouse `button` was released during the current frame.
### `imgui.is_mouse_double_clicked(button)`

Returns true if the specified mouse `button` was double-clicked during the current frame.
### `imgui.indent(indent_width)`

Indents the current line by `indent_width` pixels.
### `imgui.unindent(indent_width)`

Unindents the current line by `indent_width` pixels.
### `imgui.begin_tooltip()`

Starts a tooltip window that will be drawn at the current cursor position.
### `imgui.end_tooltip()`

Ends the current tooltip window.
### `imgui.set_tooltip(text)`

Sets the text for the current tooltip window.
### `imgui.open_popup(str_id, flags)`

Opens a popup with the specified str_id and flags.
### `imgui.begin_popup(str_id, flags)`

Begins a new popup with the specified str_id and flags.
### `imgui.begin_popup_context_item(str_id, flags)`

Begins a new popup with the specified str_id and flags, anchored to the last item.
### `imgui.end_popup()`

Ends the current popup window.
### `imgui.close_current_popup()`

Closes the current popup window.
### `imgui.is_popup_open(str_id)`

Returns true if the popup with the specified str_id is open.
### `imgui.calc_text_size(text)`

Calculates and returns the size of the specified text as a Vector2f.
### `imgui.get_window_size()`

Returns the size of the current window as a Vector2f.
### `imgui.get_window_pos()`

Returns the position of the current window as a Vector2f.
### `imgui.set_next_item_open(is_open, condition)`

Sets the open state of the next collapsing header or tree node to `is_open` based on the specified `condition`.
### `imgui.begin_list_box(label, size)`

Begins a new list box with the specified label and size.
### `imgui.end_list_box()`

Ends the current list box.
### `imgui.begin_menu_bar()`

Begins a new menu bar.
### `imgui.end_menu_bar()`

Ends the current menu bar.
### `imgui.begin_main_menu_bar()`

Begins the main menu bar.
### `imgui.end_main_menu_bar()`

Ends the main menu bar.
### `imgui.begin_menu(label, enabled)`

Begins a new menu with the specified label. The menu will be disabled if enabled is false.
### `imgui.end_menu()`

Ends the current menu.
### `imgui.menu_item(label, shortcut, selected, enabled)`

Adds a menu item with the specified label, shortcut, selected state, and enabled state.
### `imgui.get_display_size()`

Returns the size of the display as a `Vector2f.`
### `imgui.push_item_width(item_width)`

Pushes the width of the next item to `item_width` pixels.
### `imgui.pop_item_width()`

Pops the last item width off the stack.
### `imgui.set_next_item_width(item_width)`

Sets the width of the next item to `item_width` pixels.
### `imgui.calc_item_width()`

Calculates and returns the current item width.
### `imgui.push_style_color(style_color, color)`

Pushes a new style color onto the style stack.
### `imgui.pop_style_color(count)`

Pops count style colors off the style stack.
### `imgui.push_style_var(idx, value)`

Pushes a new style variable onto the style stack.
### `imgui.pop_style_var(count)`

Pops count style variables off the style stack.
### `imgui.get_cursor_pos()`

Returns the current cursor position as a `Vector2f`.
### `imgui.set_cursor_pos(pos)`

Sets the current cursor position to `pos`.
### `imgui.get_cursor_start_pos()`

Returns the initial cursor position as a `Vector2f`.
### `imgui.get_cursor_screen_pos()`

Returns the current cursor position in screen coordinates as a `Vector2f`.
### `imgui.set_cursor_screen_pos(pos)`

Sets the current cursor position in screen coordinates to `pos`.
### `imgui.set_item_default_focus()`

Sets the default focus to the next widget.

## Table API
### `imgui.begin_table(str_id, column, flags, outer_size, inner_width)`

Begins a new table with the specified `str_id`, `column` count, `flags`, `outer_size`, and `inner_width`.

- `str_id` is a string.
- `column` is an integer.
- `flags` is an optional `ImGuiTableFlags` enum.
- `outer_size` is a `Vector2f` or a size 2 array.
- `inner_width` is an optional float.

### `imgui.end_table()`

Ends the current table.
### `imgui.table_next_row(row_flags, min_row_height)`

Begins a new row in the current table with the specified `row_flags` and `min_row_height`.

- `row_flags` is an optional `ImGuiTableRowFlags` enum.
- `min_row_height` is an optional float.

### `imgui.table_next_column()`

Advances to the next column in the current table.
### `imgui.table_set_column_index(column_index)`

Sets the current column index to column_index.
### `imgui.table_setup_column(label, flags, init_width_or_weight, user_id)`

Sets up a column in the current table with the specified label, flags, init_width_or_weight, and user_id.
### `imgui.table_setup_scroll_freeze(cols, rows)`

Sets up a scrolling region in the current table with cols columns and rows rows frozen.
### `imgui.table_headers_row()`

Submits a header row in the current table.
### `imgui.table_header(label)`

Submits a header cell with the specified label in the current table.
### `imgui.table_get_sort_specs()`

Returns the sort specifications for the current table.
### `imgui.table_get_column_count()`

Returns the number of columns in the current table.
### `imgui.table_get_column_index()`

Returns the current column index.
### `imgui.table_get_row_index()`

Returns the current row index.
### `imgui.table_get_column_name(column)`

Returns the name of the specified `column` in the current table.
### `imgui.table_get_column_flags(column)`

Returns the flags of the specified `column` in the current table.
### `imgui.table_set_bg_color(target, color, column)`

Sets the background color of the specified `target` in the current table with the given `color` and `column` index.

## imguizmo
APIs for Imguizmo. For drawing widgets, see the `draw` API.

### `imguizmo.is_over()`
Returns true if any imguizmo element is being hovered by the user.

### `imguizmo.is_using()`
Returns true if any imguizmo element is being edited by the user.


Bindings for [ImNodes](https://github.com/Nelarius/imnodes). Can be used in the `re.on_frame` callback.

Real world examples can be found in the [BHVT Editor](https://github.com/praydog/RE-BHVT-Editor)

<video width="640" height="480" controls>
<source src="https://user-images.githubusercontent.com/2909949/178182705-7f4e31bb-9be4-4a9f-8a9e-951a9668da32.mp4" type="video/mp4">
</video>

---

### `imnodes.is_editor_hovered`

### `imnodes.is_node_hovered`

### `imnodes.is_link_hovered`

### `imnodes.is_pin_hovered`

### `imnodes.begin_node_editor`

### `imnodes.end_node_editor`

### `imnodes.begin_node`

### `imnodes.end_node`

### `imnodes.begin_node_titlebar`

### `imnodes.end_node_titlebar`

### `imnodes.begin_input_attribute`

### `imnodes.end_input_attribute`

### `imnodes.begin_output_attribute`

### `imnodes.end_output_attribute`

### `imnodes.begin_static_attribute`

### `imnodes.end_static_attribute`

### `imnodes.minimap`

### `imnodes.link`

### `imnodes.get_node_dimensions`

### `imnodes.push_color_style`

### `imnodes.pop_color_style`

### `imnodes.push_style_var`

### `imnodes.push_style_var_vec2`

### `imnodes.pop_style_var`

### `imnodes.set_node_screen_space_pos`

### `imnodes.set_node_editor_space_pos`

### `imnodes.set_node_grid_space_pos`

### `imnodes.get_node_screen_space_pos`

### `imnodes.get_node_editor_space_pos`

### `imnodes.get_node_grid_space_pos`

### `imnodes.is_editor_hovered`

### `imnodes.is_node_selected`

### `imnodes.is_link_selected`

### `imnodes.num_selected_nodes`

### `imnodes.num_selected_links`

### `imnodes.clear_node_selection`

### `imnodes.clear_link_selection`

### `imnodes.select_node`

### `imnodes.select_link`

### `imnodes.is_attribute_active`

### `imnodes.snap_node_to_grid`

### `imnodes.editor_move_to_node`

### `imnodes.editor_reset_panning`

### `imnodes.editor_get_panning`

### `imnodes.is_link_started`

### `imnodes.is_link_dropped`

### `imnodes.is_link_created`

### `imnodes.is_link_destroyed`

### `imnodes.get_selected_nodes`

### `imnodes.get_selected_links`


Functions for converting Lua values to/from JSON. Useful for saving configuration setting tables (ideally in a [config save callback](re.md#reon_config_savefunction)), importing data that users are intended to edit externally, and storing large data structures outside your Lua code.

The `indent` parameter is an integer specifying the number of spaces to indent with when dumping tables. 0 disables indentation, and -1 also disables line breaks.

Supports Lua's boolean, number, string, and table (see warning below) types. Other types will be converted to `nil`, stored as JSON `null`s. Due to JSON limitations, non-string table keys will be converted to strings (if numbers) or an empty string (if another type), unless the table is a sequence (consecutive integer keys, starting at 1; see the [lua manual](https://www.lua.org/manual/5.4/manual.html#3.4.7) for more details). Sequences are stored as JSON arrays.

**WARNING:** Care should be taken when storing non-sequence tables with numeric keys, as those keys will be converted to strings. Extra work must be done to convert those keys back into numbers.

The following are examples of tables that won't change when converted to and back from JSON:
```lua
{1, 3, 2, "this is a sequence"}
{foo="bar", baz=42}
{table1={1,2,3}, table2={foo=1,bar=2}}
```
The following are examples of tables that **will** change when converted to and back from JSON:
```lua
{9, 8, nil, 6, 5} -- Sequence with a "hole", becomes {["1"]=9,["2"]=8,["4"]=6,["5"]=5}
{[0]=0,[1]=1,[2]=2} -- Sequence doesn't start at 1, becomes {["0"]=0,["1"]=1,["2"]=2}
{"foo", "bar", baz=17} -- Becomes {["1"]="foo", ["2"]="bar", baz=17}
local function f() end
{[f]="function key", funcval=f} -- Becomes {[""]="function key"}
```

## Methods

### `json.load_string(json_str)`
Takes a JSON string and turns it into a Lua value (usually a table). Returns `nil` on error.

### `json.dump_string(value, [indent])`
Takes a Lua value (usually a table) and turns it into a JSON string. Returns an empty string on error. If unspecified, `indent` will default to -1.

### `json.load_file(filepath)`
Loads a JSON file identified by `filepath` relative to the `reframework/data` subdirectory and returns it as a Lua value (usually a table). Returns `nil` if the file does not exist.

### `json.dump_file(filepath, value, [indent])`
Takes a Lua value (usually a table), and turns it into a JSON file identified as `filepath` relative to the `reframework/data` subdirectory.  Returns `true` if the dump was successful, `false` otherwise. If unspecified, `indent` will default to 4


Tools for logging to REFramework's log.

## Methods
### `log.info(text)`
### `log.warn(text)`
### `log.debug(text)`
Requires DebugView or a debugger to see this. Can also be viewed in the debug console spawned with "Spawn Debug Console" under ScriptRunner.
### `log.error(text)`


This page refers to functions like:
* `sdk.call_native_func`
* `sdk.call_object_func`
* `sdk.get_native_field`
* `REManagedObject:call`
* `REManagedObject:get_field`

These functions have auto conversions for some types:
* `System.String`
    * Gets converted to a normal lua string
* `System.Int`, `System.UInt`, `System.Boolean`, `System.Single` types
    * Gets converted to native lua equivalents
* `via.vec2`, `via.vec3`, `via.vec4`
    * Gets converted to Vector2f, Vector3f, Vector4f
* `via.mat4`
    * Gets converted to Matrix4x4f


Gives access to some of the Object Explorer's UI display. Must be called within `re.on_draw_ui`.

## Methods
### `object_explorer:handle_address(addr)`
Same as typing in the address in the Object Explorer. 

`addr` must point to an REManagedObject for the display to work. 

Verification is not necessary, Object Explorer automatically handles it.

Contains some utility functions and callback creators.

## Methods
### `re.msg(text)`
Creates a MessageBox with `text`. Note that this will pause game execution until the user presses OK.

## Callback Creators
Most creators have a pre and post function. Pre meaning the callback gets triggered before the original method is called, and post being after it's called.
### `re.on_script_reset(function)`
Calls `function` when scripts are being reset. Useful for cleaning up stuff. Calls `on_config_save()`.

### `re.on_config_save(function)`
Called when REFramework wants to save its config.

### `re.on_draw_ui(function)`
Called every frame when the "Script Generated UI" in the menu is open. 

`imgui` functions can be called here, and will be placed in their own dropdown in the REFramework menu.

### `re.on_frame(function)`
Called every frame. `draw` functions can be used here. Don't use `imgui` functions unless using `begin_window` etc...

Try to minimize calling game methods when inside `on_frame` and `on_draw_ui`.

### `re.on_pre_application_entry(name, function)`
### `re.on_application_entry(name, function)`
Triggers `function` when the application/module entry associated with  `name` is being executed.

This is very powerful, and can be used to run code at many important points in the engine's logic loop.

### `re.on_pre_gui_draw_element(function)`
### `re.on_gui_draw_element(function)`
Function prototype: `function on_*_draw_element(element, context)`

Triggers `function` when a GUI element is being drawn.

Requires that a `bool` is returned in `on_pre_gui_draw_element`. When `false` is returned, the GUI element is not drawn.

`element` is an `REComponent*`.

Example:
```
re.on_pre_gui_draw_element(function(element, context)
    local game_object = element:call("get_GameObject")
    if game_object == nil then return true end

    local name = game_object:call("get_Name")

    log.info("drawing element: " .. name)

    -- Stops the crosshair from being drawn in most RE Engine games.
    if name == "GUIReticle" or name == "GUI_Reticle" then
        return false
    end

    return true
end)
```

## Valid names for re.on_application_entry
These can be found by viewing `via.ModuleEntry` in the Object Explorer.

```
Initialize
InitializeLog
InitializeGameCore
InitializeStorage
InitializeResourceManager
InitializeScene
InitializeRemoteHost
InitializeVM
InitializeSystemService
InitializeHardwareService
InitializePushNotificationService
InitializeDialog
InitializeShareService
InitializeUserService
InitializeUDS
InitializeModalDialogService
InitializeGlobalUserData
InitializeSteam
InitializeWeGame
InitializeXCloud
InitializeRebe
InitializeBcat
InitializeEffectMemorySettings
InitializeRenderer
InitializeVR
InitializeSpeedTree
InitializeHID
InitializeEffect
InitializeGeometry
InitializeLandscape
InitializeHoudini
InitializeSound
InitializeWwiselib
InitializeSimpleWwise
InitializeWwise
InitializeAudioRender
InitializeGUI
InitializeSpine
InitializeMotion
InitializeBehaviorTree
InitializeAutoPlay
InitializeScenario
InitializeOctree
InitializeAreaMap
InitializeFSM
InitializeNavigation
InitializePointGraph
InitializeFluidFlock
InitializeTimeline
InitializePhysics
InitializeDynamics
InitializeHavok
InitializeBake
InitializeNetwork
InitializePuppet
InitializeVoiceChat
InitializeVivoxlib
InitializeStore
InitializeBrowser
InitializeDevelopSystem
InitializeBehavior
InitializeMovie
InitializeMame
InitializeSkuService
InitializeTelemetry
InitializeHansoft
InitializeNNFC
InitializeMixer
InitializeThreadPool
Setup
SetupJobScheduler
SetupResourceManager
SetupStorage
SetupGlobalUserData
SetupScene
SetupDevelopSystem
SetupUserService
SetupSystemService
SetupHardwareService
SetupPushNotificationService
SetupShareService
SetupModalDialogService
SetupVM
SetupHID
SetupRenderer
SetupEffect
SetupGeometry
SetupLandscape
SetupHoudini
SetupSound
SetupWwiselib
SetupSimpleWwise
SetupWwise
SetupAudioRender
SetupMotion
SetupNavigation
SetupPointGraph
SetupPhysics
SetupDynamics
SetupHavok
SetupMovie
SetupMame
SetupNetwork
SetupPuppet
SetupStore
SetupBrowser
SetupVoiceChat
SetupVivoxlib
SetupSkuService
SetupTelemetry
SetupHansoft
StartApp
SetupOctree
SetupAreaMap
SetupBehaviorTree
SetupFSM
SetupGUI
SetupSpine
SetupSpeedTree
SetupNNFC
Start
StartStorage
StartResourceManager
StartGlobalUserData
StartPhysics
StartDynamics
StartGUI
StartTimeline
StartOctree
StartAreaMap
StartBehaviorTree
StartFSM
StartSound
StartWwise
StartAudioRender
StartScene
StartRebe
StartNetwork
Update
UpdateDialog
UpdateRemoteHost
UpdateStorage
UpdateScene
UpdateDevelopSystem
UpdateWidget
UpdateAutoPlay
UpdateScenario
UpdateCapture
BeginFrameRendering
UpdateVR
UpdateHID
UpdateMotionFrame
BeginDynamics
PreupdateGUI
BeginHavok
UpdateAIMap
CreatePreupdateGroupFSM
CreatePreupdateGroupBehaviorTree
UpdateGlobalUserData
UpdateUDS
UpdateUserService
UpdateSystemService
UpdateHardwareService
UpdatePushNotificationService
UpdateShareService
UpdateSteam
UpdateWeGame
UpdateBcat
UpdateXCloud
UpdateRebe
UpdateNNFC
BeginPhysics
BeginUpdatePrimitive
BeginUpdatePrimitiveGUI
BeginUpdateSpineDraw
UpdatePuppet
UpdateGUI
PreupdateBehavior
PreupdateBehaviorTree
PreupdateFSM
PreupdateTimeline
UpdateBehavior
CreateUpdateGroupBehaviorTree
CreateNavigationChain
CreateUpdateGroupFSM
UpdateTimeline
PreUpdateAreaMap
UpdateOctree
UpdateAreaMap
UpdateBehaviorTree
UpdateTimelineFsm2
UpdateNavigationPrev
UpdateFSM
UpdateMotion
UpdateSpine
EffectCollisionLimit
UpdatePhysicsAfterUpdatePhase
UpdateGeometry
UpdateLandscape
UpdateHoudini
UpdatePhysicsCharacterController
BeginUpdateHavok2
UpdateDynamics
UpdateNavigation
UpdatePointGraph
UpdateFluidFlock
UpdateConstraintsBegin
LateUpdateBehavior
EditUpdateBehavior
LateUpdateSpine
BeginUpdateHavok
BeginUpdateEffect
UpdateConstraintsEnd
UpdatePhysicsAfterLateUpdatePhase
PrerenderGUI
PrepareRendering
UpdateSound
UpdateWwiselib
UpdateSimpleWwise
UpdateWwise
UpdateAudioRender
CreateSelectorGroupFSM
UpdateNetwork
UpdateHavok
EndUpdateHavok
UpdateFSMSelector
UpdateBehaviorTreeSelector
BeforeLockSceneRendering
EndUpdateHavok2
UpdateJointExpression
UpdateBehaviorTreeSelectorLegacy
UpdateEffect
EndUpdateEffect
UpdateWidgetDynamics
LockScene
WaitRendering
EndDynamics
EndPhysics
BeginRendering
UpdateSpeedTree
RenderDynamics
RenderGUI
RenderGeometry
RenderLandscape
RenderHoudini
UpdatePrimitiveGUI
UpdatePrimitive
UpdateSpineDraw
EndUpdatePrimitive
EndUpdatePrimitiveGUI
EndUpdateSpineDraw
GUIPostPrimitiveRender
ShapeRenderer
UpdateMovie
UpdateMame
UpdateTelemetry
UpdateHansoft
DrawWidget
DevelopRenderer
EndRendering
UpdateStore
UpdateBrowser
UpdateVoiceChat
UpdateVivoxlib
UnlockScene
UpdateVM
StepVisualDebugger
WaitForVblank
Terminate
TerminateScene
TerminateRemoteHost
TerminateHansoft
TerminateTelemetry
TerminateMame
TerminateMovie
TerminateSound
TerminateSimpleWwise
TerminateWwise
TerminateWwiselib
TerminateAudioRender
TerminateVoiceChat
TerminateVivoxlib
TerminatePuppet
TerminateNetwork
TerminateStore
TerminateBrowser
TerminateSpine
TerminateGUI
TerminateAreaMap
TerminateOctree
TerminateFluidFlock
TerminateBehaviorTree
TerminateFSM
TerminateNavigation
TerminatePointGraph
TerminateEffect
TerminateGeometry
TerminateLandscape
TerminateHoudini
TerminateRenderer
TerminateHID
TerminateDynamics
TerminatePhysics
TerminateResourceManager
TerminateHavok
TerminateModalDialogService
TerminateShareService
TerminateGlobalUserData
TerminateStorage
TerminateVM
TerminateJobScheduler
Finalize
FinalizeThreadPool
FinalizeHansoft
FinalizeTelemetry
FinalizeMame
FinalizeMovie
FinalizeBehavior
FinalizeDevelopSystem
FinalizeTimeline
FinalizePuppet
FinalizeNetwork
FinalizeStore
FinalizeBrowser
finalizeAutoPlay
finalizeScenario
FinalizeBehaviorTree
FinalizeFSM
FinalizeNavigation
FinalizePointGraph
FinalizeAreaMap
FinalizeOctree
FinalizeFluidFlock
FinalizeMotion
FinalizeDynamics
FinalizePhysics
FinalizeHavok
FinalizeBake
FinalizeSpine
FinalizeGUI
FinalizeSound
FinalizeWwiselib
FinalizeSimpleWwise
FinalizeWwise
FinalizeAudioRender
FinalizeEffect
FinalizeGeometry
FinalizeSpeedTree
FinalizeLandscape
FinalizeHoudini
FinalizeRenderer
FinalizeHID
FinalizeVR
FinalizeBcat
FinalizeRebe
FinalizeXCloud
FinalizeSteam
FinalizeWeGame
FinalizeNNFC
FinalizeGlobalUserData
FinalizeModalDialogService
FinalizeSkuService
FinalizeUDS
FinalizeUserService
FinalizeShareService
FinalizeSystemService
FinalizeHardwareService
FinalizePushNotificationService
FinalizeScene
FinalizeVM
FinalizeResourceManager
FinalizeRemoteHost
FinalizeStorage
FinalizeDialog
FinalizeMixer
FinalizeGameCore
```


# APIs


## Methods
### `reframework:is_drawing_ui()`
Returns `true` if the REFramework menu is open.

### `reframework:get_game_name()`
Returns the name of the game this REFramework was compiled for.

e.g. "dmc5" or "re2"

### `reframework:is_key_down(key)`
`key` is a Windows virtual key code.

### `reframework:get_commit_count()`
Returns the total number of commits on the current branch of the REFramework build.

### `reframework:get_branch()`
Returns the branch name of the REFramework build.

ex: "master"

### `reframework:get_commit_hash()`
Returns the commit hash of the REFramework build.

### `reframework:get_tag()`
Returns the last tag of the REFramework build on its current branch.

ex: "v1.5.4"

### `reframework:get_tag_long()`

### `reframework:get_commits_past_tag()`
Returns the number of commits past the last tag.

### `reframework:get_build_date()`
Returns the date that REFramework was built (mm/dd/yyyy).

### `reframework:get_build_time()`
Returns the time that REFramework was built.

Main starting point for most things.

## Methods
### `sdk.get_tdb_version()`
Returns the version of the type database. A good approximation of the version of the RE Engine the game is running on.

### `sdk.game_namespace(name)`
Returns `game_namespace.name`.

DMC5: `name` would get converted to `app.name`

RE3: `name` would get converted to `offline.name`

### `sdk.get_thread_context()`

### `sdk.get_native_singleton(name)`
Returns a `void*`. Can be used with [sdk.call_native_func](#sdkcall_native_funcobject-type_definition-method_name-args)

Possible singletons can be found in the Native Singletons view in the Object Explorer.
### `sdk.get_managed_singleton(name)`
Returns an [REManagedObject*](types/REManagedObject.md).

Possible singletons can be found in the Singletons view in the Object Explorer.
### `sdk.find_type_definition(name)`
Returns an [RETypeDefinition*](types/RETypeDefinition.md).

### `sdk.typeof(name)`
Returns a `System.Type`. 

Equivalent to calling `sdk.find_type_definition(name):get_runtime_type()`.

Equivalent to `typeof` in C#.

### `sdk.create_instance(typename, simplify)`
Returns an [REManagedObject](types/REManagedObject.md).

Equivalent to calling `sdk.find_type_definition(typename):create_instance()`

`simplify` - defaults to `false`. Set this to `true` if this function is returning `nil`.

### `sdk.create_managed_string(str)`
Creates and returns a new `System.String` from `str`.

### `sdk.create_managed_array(type, length)`
Creates and returns a new [SystemArray](types/SystemArray.md) of the given `type`, with `length` elements.

`type` can be any of the following:

* A `System.Type` returned from [sdk.typeof](#sdktypeofname)
* An [RETypeDefinition](types/RETypeDefinition.md) returned from [sdk.find_type_definition](#sdkfind_type_definitionname)
* A Lua `string` representing the type name.

Any other type will throw a Lua error.

If `type` cannot resolve to a valid `System.Type`, a Lua error will be thrown.

### `sdk.create_sbyte(value)`
Returns a fully constructed [REManagedObject](types/REManagedObject.md) of type `System.SByte` given the `value`.

### `sdk.create_byte(value)`
Returns a fully constructed [REManagedObject](types/REManagedObject.md) of type `System.Byte` given the `value`.

### `sdk.create_int16(value)`
Returns a fully constructed [REManagedObject](types/REManagedObject.md) of type `System.Int16` given the `value`.

### `sdk.create_uint16(value)`
Returns a fully constructed [REManagedObject](types/REManagedObject.md) of type `System.UInt16` given the `value`.

### `sdk.create_int32(value)`
Returns a fully constructed [REManagedObject](types/REManagedObject.md) of type `System.Int32` given the `value`.

### `sdk.create_uint32(value)`
Returns a fully constructed [REManagedObject](types/REManagedObject.md) of type `System.UInt32` given the `value`.

### `sdk.create_int64(value)`
Returns a fully constructed [REManagedObject](types/REManagedObject.md) of type `System.Int64` given the `value`.

### `sdk.create_uint64(value)`
Returns a fully constructed [REManagedObject](types/REManagedObject.md) of type `System.UInt64` given the `value`.

### `sdk.create_single(value)`
Returns a fully constructed [REManagedObject](types/REManagedObject.md) of type `System.Single` given the `value`.

### `sdk.create_double(value)`
Returns a fully constructed [REManagedObject](types/REManagedObject.md) of type `System.Double` given the `value`.

### `sdk.create_resource(typename, resource_path)`
Returns an `REResource`.

If the typename does not correctly correspond to the resource file or is not a resource type, `nil` will be returned.

### `sdk.create_userdata(typename, userdata_path)`
Returns an [REManagedObject](types/REManagedObject.md) which is a `via.UserData`. `typename` can be `"via.UserData"` unless you know the full typename.

### `sdk.deserialize(data)`
Returns a list of [REManagedObject](types/REManagedObject.md) generated from `data`. 

`data` is the raw RSZ data contained for example in a `.scn` file, starting at the `RSZ` magic in the header.

`data` must in `table` format as an array of bytes.

Example usage:
```
local rsz_data = json.load_file("Foobar.json")
local objects = sdk.deserialize(rsz_data)

for i, v in ipairs(objects) do
    local obj_type = v:get_type_definition()
    log.info(obj_type:get_full_name())
end
```

### `sdk.call_native_func(object, type_definition, method_name, args...)`
Return value is dependent on what the method returns.

Full function prototype can be passed as `method_name` if there are multiple functions with the same name but different parameters.

Should only be used with native types, not [REManagedObject](types/REManagedObject.md) (though, it can be if wanted).

Example:
```lua
local scene_manager = sdk.get_native_singleton("via.SceneManager")
local scene_manager_type = sdk.find_type_definition("via.SceneManager")
local scene = sdk.call_native_func(scene_manager, scene_manager_type, "get_CurrentScene")

if scene ~= nil then
    -- We can use call like this because scene is a managed object, not a native one.
    scene:call("set_TimeScale", 5.0)
end
```
### `sdk.call_object_func(managed_object, method_name, args...)`
Return value is dependent on what the method returns.

Full function prototype can be passed as `method_name` if there are multiple functions with the same name but different parameters.

Alternative calling method:
`managed_object:call(method_name, args...)`

### `sdk.get_native_field(object, type_definition, field_name)`
### `sdk.set_native_field(object, type_definition, field_name, value)`

### `sdk.get_primary_camera()`
Returns a [REManagedObject*](types/REManagedObject.md). Returns the current camera being used by the engine.

### `sdk.hook(method_definition, pre_function, post_function, ignore_jmp)`
Creates a hook for [method_definition](types/REMethodDefinition.md), intercepting all incoming calls the game makes to it.

`ignore_jmp` - Skips trying to follow the first jmp in the function. Defaults to `false`.

Using `pre_function` and `post_function`, the behavior of these functions can be modified.

NOTE: Some native methods may not be able to be hooked with this, e.g. if they are just a  wrapper over the native function. Some additional work will need to be done from our end to make those work.

pre_function and post_function looks like so:
```lua
local function pre_function(args)
    -- args are modifiable
    -- args[1] = thread_context
    -- args[2] = "this"/object pointer
    -- rest of args are the actual parameters
    -- actual parameters start at args[2] in a static function
    -- Some native functions will have the object start at args[1] and rest at args[2]
    -- All args are void* and not auto-converted to their respective types.
    -- You will need to do things like sdk.to_managed_object(args[2])
    -- or sdk.to_int64(args[3]) to get arguments to better interact with or read.

    -- OPTIONAL: Specify an sdk.PreHookResult
    -- e.g.
    -- return sdk.PreHookResult.SKIP_ORIGINAL -- prevents the original function from being called
    -- return sdk.PreHookResult.CALL_ORIGINAL -- calls the original function, same as not returning anything
end

local function post_function(retval)
    -- return something else if you don't want the original return value
    -- NOTE: the post_function will still be called if SKIP_ORIGINAL is returned from the pre_function
    -- So, if your function expects something valid in return, keep that in mind, as retval will not be valid.
    -- Make sure to convert custom retvals to sdk.to_ptr(retval)
    return retval
end
```

Example hook:
```lua
local function on_pre_get_timescale(args)
end

local function on_post_get_timescale(retval)
    -- Make the game run 5 times as fast instead
    -- TODO: Make it so casting return values like this is not necessary
    return sdk.float_to_ptr(5.0)
end

sdk.hook(sdk.find_type_definition("via.Scene"):get_method("get_TimeScale"), on_pre_get_timescale, on_post_get_timescale)
```

### `sdk.hook_vtable(obj, method, pre, post)`
Similar to `sdk.hook` but hooks on a **per-object** basis instead, instead of hooking the function globally for all objects.

Only works if the target method is a **virtual method**.

### `sdk.is_managed_object(value)`
Returns true if `value` is a valid [REManagedObject](types/REManagedObject.md).

Use only if necessary. Does a bunch of checks and calls [IsBadReadPtr](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-isbadreadptr) a lot.
### `sdk.to_managed_object(value)`
Attempts to convert `value` to an [REManagedObject*](types/REManagedObject.md).

`value` can be any of the following types:

* An [REManagedObject*](types/REManagedObject.md), in which case it is returned as-is
* A lua number convertible to `uintptr_t`, representing the object's address
* A `void*`

Any other type will return `nil`.

A `value` that is not a valid [REManagedObject*](types/REManagedObject.md) will return `nil`, equivalent to calling [sdk.is_managed_object](#sdkis_managed_objectvalue) on it.

### `sdk.to_double(value)`
Attempts to convert `value` to a `double`.

`value` can be any of the following types:

* A `void*`

### `sdk.to_float(value)`
Attempts to convert `value` to a `float`.

`value` can be any of the following types:
* A `void*`

### `sdk.to_int64(value)`
Attempts to convert `value` to a `int64`.

`value` can be any of the following types:
* A `void*`

If you need a smaller datatype, you can do:
* `(sdk.to_int64(value) & 1) == 1` for a boolean
* `(sdk.to_int64(value) & 0xFF)` for an unsigned byte
* `(sdk.to_int64(value) & 0xFFFF)` for an unsigned short (2 bytes)
* `(sdk.to_int64(value) & 0xFFFFFFFF)` for an unsigned int (4 bytes)

### `sdk.to_ptr(value)`
Attempts to convert `value` to a `void*`.

`value` can be any of the following types:

* An [REManagedObject*](types/REManagedObject.md)
* A lua number convertible to `int64_t`
* A lua number convertible to `double`
* A lua boolean
* A `void*`, in which case it is returned as-is

Any other type will return `nil`.

### `sdk.float_to_ptr(number)`
Converts `number` to a `void*`.

## Enums
### `sdk.PreHookResult`
* `sdk.PreHookResult.CALL_ORIGINAL`
* `sdk.PreHookResult.SKIP_ORIGINAL`


# thread

The `thread` API is for storing thread-specific data and querying information about the current thread.

Added in `8e9375ce5433c5b4ce38e8398c168c3ab036415c`.

## `thread.get_id()`

Returns the ID of the current thread.

## `thread.get_hash()`

Returns the hash of the ID of the current thread.

## `thread.get_hook_storage()`

Returns the ephemeral hook storage meant to be used within `sdk.hook`.

This is preferred over storing variables you need in a global variable in the `pre` hook when you need the data in the `post` hook.

The hook storage is popped/destroyed at the end of the `post` hook. Safe to be used within a recursive context.

This API is preferred because there are no longer any guarantees that the entire hook will be locked during pre/post hooks, due to deadlocking issues seen.

### Example

```lua
local pawn_t = sdk.find_type_definition("app.Pawn")

sdk.hook(
    pawn_t:get_method("updateMove"),
    function(args)
        local storage = thread.get_hook_storage()
        storage["this"] = sdk.to_managed_object(args[2])
    end,
    function(retval)
        local this = thread.get_hook_storage()["this"]
        print("this: " .. tostring(this:get_type_definition():get_full_name()))
        return retval
    end
)
```

Bindings to access VR from lua.

## Methods
### `vrmod:get_controllers()`
Returns a list of device indices for the active controllers.

### `vrmod:get_position(index)`
Returns the position for a given device index.

### `vrmod:get_rotation(index)`
Returns the rotation for a given device index.

### `vrmod:get_transform(index)`
Returns the full transformation matrix for a given device index.

### `vrmod:get_velocity(index)`
Returns the velocity for a given device index.

### `vrmod:get_angular_velocity(index)`
Returns the angular velocity for a given device index.

### `vrmod:get_left_stick_axis()`
Returns a `Vector2f`.

### `vrmod:get_right_stick_axis()`
Returns a `Vector2f`.

### `vrmod:get_current_eye_transform(flip)`
### `vrmod:get_current_projection_matrix(flip)`
### `vrmod:get_standing_origin()`
### `vrmod:set_standing_origin(pos)`
`pos` is a Vector4f.

### `vrmod:get_rotation_offset()`
### `vrmod:set_rotation_offset(quat)`
### `vrmod:recenter_view()`
### `vrmod:recenter_gui(from)`
`from` is a `Quaternion`.

### `vrmod:get_action_set()`
### `vrmod:get_active_action_set()`
### `vrmod:get_action_trigger()`
### `vrmod:get_action_grip()`
### `vrmod:get_action_joystick()`
### `vrmod:get_action_joystick_click()`
### `vrmod:get_action_a_button()`
### `vrmod:get_action_b_button()`
### `vrmod:get_action_weapon_dial()`
### `vrmod:get_action_minimap()`
### `vrmod:get_action_block()`
### `vrmod:get_action_dpad_up`
### `vrmod:get_action_dpad_down`
### `vrmod:get_action_dpad_left`
### `vrmod:get_action_dpad_right`
### `vrmod:get_action_heal`
### `vrmod:get_left_joystick()`
Returns a `vr::VRInputValueHandle_t`. To be used in `vrmod:is_action_active` as the `source`.

### `vrmod:get_right_joystick()`
Returns a `vr::VRInputValueHandle_t`. To be used in `vrmod:is_action_active` as the `source`.

### `vrmod:get_right_joystick()`
### `vrmod:is_using_controllers()`
Returns `true` if the user has issued any inputs to the controllers within the last 10 seconds.

### `vrmod:is_openvr_loaded()`
Returns `true` if OpenVR is loaded.

### `vrmod:is_openxr_loaded()`
Returns `true` if OpenXR is loaded.

### `vrmod:is_hmd_active()`
Returns `true` if the user currently has their VR headset on.
### `vrmod:is_action_active(action, source)`
Returns `true` if the `action` belonging to `source` is active. 

Active meaning that the user is e.g. holding the A button down if the A button was the `action`.
### `vrmod:is_using_hmd_oriented_audio()`
### `vrmod:toggle_hmd_oriented_audio()`
### `vrmod:apply_hmd_transform(rotation, position)`
Applies the headset's transform to the given rotation and position. Both parameters are in and out parameters.

### `vrmod:apply_haptic_vibration(seconds_from_now, duration, frequency, amplitude, source)`
### `vrmod:get_last_render_matrix()`


## Shared State
Lua uses a shared state across all scripts. Use `local` variables *and* `local` functions (not necessary for local tables) so as to not cause conflicts with other scripts.

### Example
```lua
-- explicit local variables
local foo = {}

-- implicitly local due to local foo
function foo.baz()
    return "baz"
end

-- explicitly local because not part of a table
local function bar()
    return "bar"
end
```

## Hooking gotchas
As of commit [7145dbda6cb7cb5b8dd12d9dee14f51850a76ec6](https://github.com/praydog/REFramework/commit/7145dbda6cb7cb5b8dd12d9dee14f51850a76ec6), these duplicate functions are displayed next to the method.

![duplicate](https://user-images.githubusercontent.com/2909949/185234701-b58f3ae1-1bf1-400f-8f21-de9c358492bc.png)


If you are hooking a function that looks like a `get_` or `set_` function, double check the disassembly in the Object Explorer. More often than not, these functions will be **extremely** simple and prone to compiler optimizations causing multiple unrelated function calls to go through the hook. This means garbage data you don't want will flow through the function, and you can potentially crash the game if you are modifying the control flow of the wrong functions.

If you **really** want to hook these functions, you will need to verify that the object type being passed through the arguments is the one you want. Leave the control flow state and arguments pristine until you are 100% sure what is being passed through is what you want.

![dghsdfgsd](https://user-images.githubusercontent.com/2909949/178127784-2f5103df-6959-4150-8fc4-b1d7713b875a.png)
![dont do it](https://user-images.githubusercontent.com/2909949/178127840-af14513c-30f5-4f88-8268-2ec963b4b24e.png)

## Hooking performance considerations
Using `sdk.hook` is very useful. However, this comes with a few things to take note of.

When you are hooking something like an `update` function that runs every frame, for every entity in the game, this can cause performance problems if you are calling a lot of functions within the hook.

A way to work around this, if it's possible in your script, is to stagger the function calls across multiple ticks. Also cache the methods and fields. [A real world example can be found here](https://github.com/GreenComfyTea/MHR-Overlay/pull/19).

You can also create a plugin that will run the heavy code natively, and then call it from Lua.

## Method calls & field access performance considerations
Another very useful pair of functions is `object:call` and `object:get/set_field`. Whenever these functions are called, they perform a hashmap lookup. These can be slightly more efficent if you cache off the method and field definitions.

Example implementation
```lua
local method1 = sdk.find_type_definition("Foo"):get_method("Bar")
local field1 = sdk.find_type_definition("Foo"):get_field("Baz")

re.on_frame(function()
    local some_object = sdk.get_managed_singleton("Qux")
    local bar_result = method1:call(some_object, 1, 2, 3)
    local baz_result = field1:get_data(some_object)
end)
```

## Plugins
If there's something you find you can't do without native code, Lua can `require` native DLLs. Native plugins are also an option.

Plugins can also be used to run heavy code that would otherwise be very slow in Lua. Parts of your script can still run in Lua, but you can expose APIs from your plugin that would run the heavy parts natively.

## Modules
Lua can require modules. Modules are a way to organize your code.

### Module1.lua
This goes in a subfolder inside the autorun folder.

```lua
local test = {}

function test.foo()
    print("foo")
end

return test
```

### Main.lua
This goes in the autorun folder.

```lua
local module1 = require("subfolder/Module1")

-- Now you can call module1.foo()
module1.foo()
```


## `sdk.hook` method arguments
`ByRef` parameters are not correctly supported by REFramework. `ByRef` parameters are essentially `T**` instead of `T*` type pointers. They are usually used for out parameters.

These will need to be manually dereferenced using a trick by instantiating a `System.UInt64` and reading the `mValue` field to dereference it.

```lua
local function deref_ptr(ptr)
    local fake_int64 = sdk.to_valuetype(ptr, "System.UInt64")
    local deref = fake_int64:get_field("mValue")

    return deref
end

sdk.hook(sdk.find_type_definition("foo"):get_method("bar"),
    function(args)
        local deref = deref_ptr(args[6])
        local arg = sdk.to_managed_object(deref):add_ref()
    end,
    function(retval)
        return retval
    end
)
```


This page refers to functions like:
* `sdk.call_native_func`
* `sdk.call_object_func`
* `sdk.get_native_field`
* `REManagedObject:call`
* `REManagedObject:get_field`

These functions have auto conversions for some types:
* `System.String`
    * Gets converted to a normal lua string
* `System.Int`, `System.UInt`, `System.Boolean`, `System.Single` types
    * Gets converted to native lua equivalents
* `via.vec2`, `via.vec3`, `via.vec4`
    * Gets converted to Vector2f, Vector3f, Vector4f
* `via.mat4`
    * Gets converted to Matrix4x4f


# General


## Constructors
### `Matrix4x4f.new()`

### `Matrix4x4f.new(x1, y1, z1, w1, x2, y2, z2, w2 x3, y3, z3, w3, x4, y4, z4, w4)`

### Static methods
### `Matrix4x4f.identity()`
Returns the identity matrix.

## Methods

### `self:to_quat()` 
Returns a `Quaternion` built from `self`.

### `self:inverse()`
Returns a `Matrix4x4f` that is the inverse of `self`.

### `self:invert()`
Inverts `self`. Returns nothing.

### `self:interpolate(other, t)`
Returns the linear interpolation between `self` and `other` with the given `t`.

### `self:matrix_rotation()`
Extracts the rotation matrix from `self`.

## Meta-methods

### `Matrix4x4f * Matrix4x4f`
`Matrix4x4f` multiplication.

### `Matrix4x4f * Vector4f`
`Matrix4x4f` `Vector4f` multiplication

### `Matrix4x4f[]`
`Matrix4x4f` element indexing. Valid range is `[0, 3)`.

Returns a `Vector4f`.

## Constructor
### `Quaternion.new(w, x, y, z)`

## Static Methods
### `Quaternion.identity()`
Returns the identity quaternion.

## Fields
### `x: number`
The X component of the `Quaternion`.

### `y: number`
The Y component of the `Quaternion`.

### `z: number`
The Z component of the `Quaternion`.

### `w: number`
The W component of the `Quaternion`.

## Methods

### `self:to_mat4()`
Returns a `Matrix4x4f` built from `self`.

### `self:to_euler()`
Returns a `Vector3f` representing the Euler angles for this Quaternion.

### `self:inverse()`
Returns a `Quaternion` that is the inverse of `self`.

### `self:invert()`
Inverts `self`. Returns nothing.

### `self:normalize()`
Normalizes `self`. Returns nothing.

### `self:normalized()`
Returns a `Quaternion` that is the normalization of `self`.

### `self:slerp(other, t)`
Returns a `Quaternion` that is the spherical linear interpolation between `self` and `other` with the given `t`.

### `self:dot(other)`
Returns the dot product between `self` and `other`.

### `self:length()`
Returns the length of `self`.

### `self:conjugate()`
Returns a `Quaternion` that is the conjugate of `self`.

## Meta-methods

### `Quaternion * Quaternion`
`Quaternion` multiplication.

### `Quaternion * Vector3f`
`Quaternion` `Vector3f` multiplication.

### `Quaternion * Vector4f`
`Quaternion` `Vector4f` multiplication.

### `Quaternion[]`
`Quaternion` element indexing. Valid range is `[0, 4)`.


# Types


`REComponents` are fundamental building blocks for all `GameObjects`. They can be removed and added at runtime to `GameObjects`.

Inherits from `REManagedObject`.

## Methods
None at the moment.

## Methods
### `self:get_name()`
### `self:get_type()`
Returns an `RETypeDefinition*`.
### `self:get_offset_from_base()`
### `self:get_offset_from_fieldptr()`
### `self:get_declaring_type()`
### `self:get_flags()`
### `self:is_static()`
### `self:is_literal()`
### `self:get_data(obj)`
Returns the data contained in the field for `obj`. 

`obj` can be any of the following type:
  - `nil`, if the field is static
  - `REManagedObject*`
  - `void*` pointing to a `REManagedObject` or `ValueType`


REManagedObjects are the basic building blocks of most types in the engine (unless they're native types).

They are returned from methods like:
* `sdk.call_native_func`
* `sdk.call_object_func`
* `sdk.get_managed_singleton`
* `REManagedObject:call`

Example usage:
```lua
local scene_manager = sdk.get_native_singleton("via.SceneManager")
local scene_manager_type = sdk.find_type_definition("via.SceneManager")
local scene = sdk.call_native_func(scene_manager, scene_manager_type, "get_CurrentScene")

-- Scene is an REManagedObject
if scene ~= nil then
    local current_timescale = scene:call("get_TimeScale")
    log.info("Current timescale: " .. tostring(current_timescale))

    scene:call("set_TimeScale", 5.0)
end
```

## Custom indexers
### `self.foo`
If `foo` is a field or method of the object, returns either the field or `REMethodDefinition` if it exists.

### `self:foo(bar, baz)`
If `foo` is a method of the object, calls `foo` with the supplied arguments.

If the method is an overloaded function, you must instead use `self:call(name, args...)` with the correct function prototype, as this does not deduce the correct function based on the passed arguments.

### `self.foo = bar`
If `foo` is a field of the object, assigns the value `bar` to the field.

This automatically handles the reference counting for the old and new field. Do not use `:force_release()` and `:add_ref_permanent()` in this case to handle the references.

### `self[i]`
Checks if the object has a `get_Item` method and calls it with `i`.

### `self[i] = foo`
Checks if the object has a `set_Item` method and calls it with `i` and `foo` as the respective parameters.

## Methods
### `self:call(method_name, args...)`
Return value is dependent on the method's return type. Wrapper over `sdk.call_object_func`.

Full function prototype can be passed as `method_name` if there are multiple functions with the same name but different parameters.

e.g. `self:call("foo(System.String, System.Single, System.UInt32, System.Object)", a, b, c, d)`

Valid method names can be found in the Object Explorer. Find the type you're looking for, and valid methods will be found under `TDB Methods`.
### `self:get_type_definition()`
Returns an `RETypeDefinition*`.
### `self:get_field(name)`
Return type is dependent on the field type.
### `self:set_field(name, value)`
### `self:get_address()`
### `self:get_reference_count()`

### `self:deserialize_native(data, objects)`
Experimental API to deserialize `data` into `self`.

`data` is RSZ data, in `table` format as an array of bytes.

Will only work on native `via` types.

## Dangerous Methods
Only use these if necessary!

### `self:add_ref()`
Increments the object's internal reference count.

### `self:add_ref_permanent()`
Increments the object's internal reference count without REFramework managing it. Any objects created with REFramework and also using this method will not be deleted after the Lua state is destroyed.

### `self:release()`
Decrements the object's internal reference count. Destroys the object if it reaches 0. Can only be used on objects managed by Lua.

### `self:force_release()`
Decrements the object's internal reference count. Destroys the object if it reaches 0. Can be used on any REManagedObject. Can crash the game or cause undefined behavior.

When a new Lua reference is created to an `REManagedObject`, REFramework automatically increments its reference count internally with `self:add_ref()`. This will keep the object alive until you are no longer referencing the object in Lua. `self:release()` is automatically called when Lua is no longer referencing the object anywhere.

The only time you will need to manually call `self:add_ref()` and `self:release()` is when a newly created object is returned by the engine, e.g. an array, or something from `sdk.create_instance()`.

A more in-depth explanation can be found in the "FrameGC Algorithm" section of this GDC presentation by Capcom:

https://github.com/kasicass/blog/blob/master/3d-reengine/2021_03_10_achieve_rapid_iteration_re_engine_design.md#framegc-algorithm-17

### `self:read_byte(offset)`
### `self:read_short(offset)`
### `self:read_dword(offset)`
### `self:read_qword(offset)`
### `self:read_float(offset)`
### `self:read_double(offset)`
### `self:write_byte(offset, value)`
### `self:write_short(offset, value)`
### `self:write_dword(offset, value)`
### `self:write_qword(offset, value)`
### `self:write_float(offset, value)`
### `self:write_double(offset, value)`


Method descriptor.

## Methods
### `self:get_name()`

### `self:get_return_type()`
Returns an `RETypeDefinition*`.

### `self:get_function()`
Returns a `void*`. Pointer to the actual function in memory.

### `self:get_declaring_type()`
Returns an `RETypeDefinition*` corresponding to the class/type that declared this method.

### `self:get_num_params()`
Returns the number of parameters required to call the function.

### `self:get_param_types()`
Returns a list of `RETypeDefinition`

### `self:get_param_names()`
Returns a list of strings for the parameter names

### `self:is_static()`
Returns whether this method is static or not.

### `self:call(obj, args...)`
Equivalent to calling `obj:call(args...)`

Can also use `self(obj, args...)`


## Methods
### `self:add_ref()`
Adds a reference to `self`. `REResource` types are not automatically reference counted like `REManagedObject`.

### `self:release()`
Releases a reference to `self`. `REResource` types are not automatically reference counted like `REManagedObject`.

### `self:get_address()`
Returns the address of `self`.

### `self:create_holder(typename)`
Returns a `via.ResourceHolder` variant which holds `self`. Automatically adds a reference to `self`.

```lua
local res = sdk.create_resource("via.motion.MotionFsm2Resource", "_Chainsaw/AppSystem/Character/ch0Common/Motion/Fsm/ch0Common.motfsm2"):add_ref()
local holder = res:create_holder("via.motion.MotionFsm2ResourceHolder"):add_ref()
```


`RETransform` is the basic building block of all GameObjects, they always contain one. 

Inherits from `REComponent`.

## Methods
### `self:calculate_base_transform(joint)`
Returns a `Matrix4x4f`. Returns the reference pose (T-pose) for a specific joint relative to the transform's origin (in local transform space).

### `self:set_position(position, no_dirty)`
Sets the world position (`Vector4f`) of the transform.

When `no_dirty` is `true`, the transform and its parents will not be marked as dirty. This seems to be necessary when the scene is locked, because parent transforms will end up getting stuck.

### `self:set_rotation(rotation)`
Sets the world rotation (`Quaternion`) of the transform.

### `self:get_position()`
Gets the world position (`Vector4f`) of the transform.

### `self:get_rotation()`
Gets the world rotation (`Quaternion`) of the transform.

Type descriptor for objects in the RE Engine. 

Returned from things like `REManagedObject:get_type_definition()` or `sdk.find_type_definition(name)`

## Methods
### `self:get_full_name()`
Returns the full name of the class.

Equivalent to concatenating `self:get_namespace()` and `self:get_name()`.

### `self:get_name()`
Returns the type name. Does not contain namespace.

### `self:get_namespace()`
Returns the namespace this type is contained in.

### `self:get_method(name)`
Returns an `REMethodDefinition`. To be used in things like `sdk.hook`.

The full function prototype can be supplied to get an overloaded function.

Example: `foo:get_method("Bar(System.Int32, System.Single)")`

### `self:get_methods()`
Returns a list of `REMethodDefinition`

Filters out methods that are potentially just stubs or null.

### `self:get_field(name)`
Returns an `REField`.

### `self:get_fields()`
Returns a list of `REField`

### `self:get_parent_type()`
Returns the `RETypeDefinition` this type inherits from.

### `self:get_runtime_type()`
Returns a `System.Type`. Useful for methods that require this. Equivalent to `typeof` in C#.

### `self:get_size()`
Returns the full size of the object. e.g. 0x14 for `System.Int32`.

### `self:get_valuetype_size()`
Returns the value type size. e.g. 4 for `System.Int32`. 

### `self:get_generic_argument_types()`

### `self:get_generic_type_definition()`

### `self:is_a(typename or RETypeDefinition)`
Returns whether `self` or its parents are a `typename` or the `RETypeDefinition` passed.

### `self:is_value_type()`
Returns whether the type is a [ValueType](https://docs.microsoft.com/en-us/dotnet/api/system.valuetype?view=net-5.0).

Does not necessarily need to inherit from `System.ValueType` for this to be true. An example would be `via.vec3`.

### `self:is_by_ref()`

### `self:is_pointer()`

### `self:is_primitive()`

### `self:is_generic_type()`

### `self:is_generic_type_definition()`

### `self:create_instance()`
Returns an `REManagedObject`.


Easy-to-use wrapper over `System.Array`. Functions calls that return arrays or objects will automatically get converted to `SystemArray` types if eligible.

Inherits from `REManagedObject`.

## Notes
Do not use `ipairs` on `SystemArray` types. Use `pairs` instead, unless you return the elements in a lua array via `get_elements()`. Using `ipairs` will skip the first element and go past the end of the array.

## Methods
### `self:get_elements()`
Returns the array's elements as a lua table. 

Keep in mind these objects will all be full `REManagedObject` types, not the ValueTypes they represent, if any, like `System.Int32`
### `self:get_element(index)`
Returns the object at `index` in the array.
### `self:get_size()`
Returns the size of the array.

## Meta-methods
### `SystemArray[]`
Wrapper for `self:get_element(index)`


Container for unknown ValueTypes.

## Methods
### `ValueType.new(typename)`

### `self:call(name, args...)`
### `self:get_field(name)`
### `self:set_field(name, value)`
Note that this does not change anything in-game. `ValueType` is just a local copy.

You'll need to pass the `ValueType` somewhere that would make use of the changed data.

### `self:address()`
### `self:get_type_definition()`

### `self.type`
### `self.data`
`std::vector<uint8_t>`

## Dangerous Methods
Only use these if necessary!
### `self:read_byte(offset)`
### `self:read_short(offset)`
### `self:read_dword(offset)`
### `self:read_qword(offset)`
### `self:read_float(offset)`
### `self:read_double(offset)`
### `self:write_byte(offset, value)`
### `self:write_short(offset, value)`
### `self:write_dword(offset, value)`
### `self:write_qword(offset, value)`
### `self:write_float(offset, value)`
### `self:write_double(offset, value)`

There are 3 `VectorXf` types:
* `Vector2f`
* `Vector3f`
* `Vector4f`

## Creation
### `Vector2f.new(x, y)`

### `Vector3f.new(x, y, z)`

### `Vector4f.new(x, y, z, w)`

## Fields

### `x: number`
The X component of the `VectorXf`

### `y: number`
The Y component of the `VectorXf`

### `z: number`
The Z component of the `VectorXf`. Only `Vector3f` and `Vector4f` have this field.

### `w: number`
The W component of the `VectorXf`. Only `Vector4f` has this field.

## Methods

### `self:dot(other)`
Returns the dot product between `self` and `other`.

### `self:cross(other)`
Returns the cross product between `self` and `other`.

### `self:length()`
Returns the length of `self`.

### `self:normalize()`
Normalizes `self`. Nothing is returned.

### `self:normalized()`
Returns the normalization of `self`.

### `self:reflect(normal)`
Returns the reflection of `self` over `normal`.

### `self:refract(normal, eta)`
Returns the refraction of `self` over `normal` with the given `eta`.

### `self:lerp(other, t)`
Returns the linear interpolation between `self` and `other` with the given `t`.

### `self:to_vec2()`
Converts `self` to a `Vector2f`. Not available if `self` is already a `Vector2f`.

### `self:to_vec3()`
Converts `self` to a `Vector3f`. Not available if `self` is already a `Vector3f`.

### `self:to_vec4()`
Converts `self` to a `Vector4f`. Not available if `self` is already a `Vector4f`.

### `self:to_mat()`
Converts `self` to a `Matrix4x4f`. Treats `self` as the forward vector.

### `self:to_quat()`
Converts `self` to a `Quaternion`. Treats `self` as the forward vector.

Equivalent to `self:to_mat():to_quat()`.

## Meta-methods

### `VectorXf + VectorXf`
`VectorXf` addition.

### `VectorXf - VectorXf`
`VectorXf` subtraction.

### `VectorXf * scalar`
`VectorXf` `scalar` multiplication.


# General

TODO: Write a general overview of the C# scripting API.

# Behavior Tree Editor
The Behavior Tree Editor is a Lua script that isn't part of REFramework. It may become native and built-in at some point.

[https://github.com/praydog/RE-BHVT-Editor](https://github.com/praydog/RE-BHVT-Editor)

## Examples

### The UI

<video width="640" height="480" controls>
<source src="https://user-images.githubusercontent.com/2909949/178182705-7f4e31bb-9be4-4a9f-8a9e-951a9668da32.mp4" type="video/mp4">
</video>

### Using Lua driven condition evaluators (custom actions are preferred in newer versions) to run on-hit effects for all child nodes

<video width="640" height="480" controls>
<source src="https://user-images.githubusercontent.com/2909949/178722895-0c521cc6-004f-4ef9-9133-39112cfdf7f6.mp4" type="video/mp4">
</video>

### Using Lua driven condition evaluators (custom actions are preferred in newer versions) to dynamically adjust the game speed on hit and play on hit effects

<video width="640" height="480" controls>
<source src="https://user-images.githubusercontent.com/2909949/178723228-73cfd435-16b7-4f57-92f2-67a4f35a46e3.mp4" type="video/mp4">
</video>

### Adding custom actions/effects to specific nodes

<video width="640" height="480" controls>
<source src="https://user-images.githubusercontent.com/2909949/178724426-5feb9624-c071-42b6-919a-f9efc037b04c.mp4" type="video/mp4">
</video>

### All of the above combined, with modification of transition states to create a custom combo that loops back around

<video width="640" height="480" controls>
<source src="https://user-images.githubusercontent.com/2909949/178810141-1195f33e-1b60-4c34-9900-9d7e21960329.mp4" type="video/mp4">
</video>




# Chain Viewer

The chain viewer is used to view any active `via.motion.Chain` objects, and in particular, visualize their collisions. 

This can be used to make better decisions when modding chain files.

<video width="640" height="480" controls>
<source src="https://user-images.githubusercontent.com/2909949/176352416-8beb0d3a-27b6-4895-b05e-9536b14400b2.mp4" type="video/mp4">
</video>

<video width="640" height="480" controls>
<source src="https://user-images.githubusercontent.com/2909949/176352654-099fffe0-b7b6-41ec-9919-0fb588b97897.mp4" type="video/mp4">
</video>


[http://praydog.com/projects/reframework-scripts/](http://praydog.com/projects/reframework-scripts/)

[https://github.com/praydog/REFramework/tree/master/scripts](https://github.com/praydog/REFramework/tree/master/scripts)

[https://infernalwarks.boards.net/thread/572/enhanced-model-viewer-dmc5?page=1&scrollTo=1005](https://infernalwarks.boards.net/thread/572/enhanced-model-viewer-dmc5?page=1&scrollTo=1005)

[https://www.nexusmods.com/residentevilvillage/mods/162](https://www.nexusmods.com/residentevilvillage/mods/162)

[https://github.com/Sarayalth/mhr_scripts](https://github.com/Sarayalth/mhr_scripts)

[https://www.nexusmods.com/monsterhunterrise/mods/39](https://www.nexusmods.com/monsterhunterrise/mods/39)

[https://github.com/cursey/mhrise-scripts](https://github.com/cursey/mhrise-scripts)

[https://github.com/alphazolam/REFramework-Scripts](https://github.com/alphazolam/REFramework-Scripts)

[https://github.com/originalnicodr/RELit](https://github.com/originalnicodr/RELit)


## RE Engine
### Grabbing components from a game object
```lua
-- Find a component contained in a game object by its type name
local function get_component(game_object, type_name)
    local t = sdk.typeof(type_name)

    if t == nil then 
        return nil
    end

    return game_object:call("getComponent(System.Type)", t)
end

-- Get all components of a game object as a table
local function get_components(game_object)
    local transform = game_object:call("get_Transform")

    if not transform then
        return {}
    end

    return game_object:call("get_Components"):get_elements()
end
```

### Getting the current elapsed time in seconds
In newer builds, `os.clock` is available.

```lua
local app_type = sdk.find_type_definition("via.Application")
local get_elapsed_second = app_type:get_method("get_UpTimeSecond")

local function get_time()
    return get_elapsed_second:call(nil)
end
```

### Generating enums/static fields
```lua
local function generate_enum(typename)
    local t = sdk.find_type_definition(typename)
    if not t then return {} end

    local fields = t:get_fields()
    local enum = {}

    for i, field in ipairs(fields) do
        if field:is_static() then
            local name = field:get_name()
            local raw_value = field:get_data(nil)

            log.info(name .. " = " .. tostring(raw_value))

            enum[name] = raw_value
        end
    end

    return enum
end

via.hid.GamePadButton = generate_enum("via.hid.GamePadButton")
app.HIDInputMode = generate_enum("app.HIDInputMode")
```

### GUI Debugger
```lua
known_elements = {}

re.on_pre_gui_draw_element(function(element, context)
    known_elements[element:call("get_GameObject")] = os.clock()
end)

local draw_control = nil
local draw_children = nil
local draw_next = nil

draw_control = function(control, prefix, seen)
    prefix = prefix or ""
    if control == nil then return end

    seen = seen or {}
    if seen[control] then return end
    seen[control] = true

    local name = control:call("get_Name")
    if imgui.tree_node(prefix .. name) then
        draw_children(control, prefix, seen)
        object_explorer:handle_address(control)
        imgui.tree_pop()
    end

    draw_next(control, prefix, seen)
end

draw_next = function(control, prefix, seen)
    prefix = prefix or ""
    if control == nil then return end

    local ok, next = pcall(control.call, control, "get_Next")

    if ok then
        draw_control(next, prefix, seen)
    end

    --draw_next(control, prefix)
    --draw_children(control, prefix .. " ")
end

draw_children = function(control, prefix)
    prefix = prefix or ""
    if control == nil then return end

    local child = control:call("get_Child")
    draw_control(child, prefix .. " ", seen)
end

re.on_draw_ui(function()
    local sorted_elements = {}

    for k, v in pairs(known_elements) do
        local succeed, result = pcall(k.call, k, "get_Name")

        if not succeed or result == nil or k:get_reference_count() == 1 or (os.clock() - v > 1) then
            known_elements[k] = nil
        else
            table.insert(sorted_elements, k)
        end
    end

    table.sort(sorted_elements, function(a, b)
        return a:call("get_Name") < b:call("get_Name")
    end)

    for i, element in ipairs(sorted_elements) do
        imgui.push_id(tostring(element:get_address()))

        if imgui.tree_node(element:call("get_Name") .. " " .. string.format("%x", element:get_address())) then
            local gui = element:call("getComponent(System.Type)", sdk.typeof("via.gui.GUI"))
            local transform = element:call("get_Transform")
            local joints = transform:call("get_Joints")

            if joints then
                imgui.text("Joints: " .. tostring(joints:get_size()))
            end

            if gui ~= nil then
                local ok, world_pos_attach = pcall(element, call, "getComponent(System.Type)", sdk.typeof("app.UIWorldPosAttach"))

                if ok and world_pos_attach ~= nil then
                    local now_target_pos = world_pos_attach:get_field("_NowTargetPos")
                    local screen_pos = draw.world_to_screen(now_target_pos)

                    if screen_pos then
                        local name = element:call("get_Name")

                        draw.text(name, screen_pos.x, screen_pos.y, 0xFFFFFFFF)
                    end
                end

                local view = gui:call("get_View")

                if view ~= nil then
                    draw_control(view)
                end
            end

            object_explorer:handle_address(element)
            imgui.tree_pop()
        end

        imgui.pop_id()
    end
end)
```



<video width="1280" height="720" controls>
<source src="https://user-images.githubusercontent.com/2909949/176351319-c070b216-fe71-4eb9-84f2-46c665892b11.mp4" type="video/mp4">
</video>



### 3D Gizmo test script
```
local gn = reframework:get_game_name()

local function get_localplayer()
    if gn == "re2" or gn == "re3" then
        local player_manager = sdk.get_managed_singleton(sdk.game_namespace("PlayerManager"))
        if player_manager == nil then return nil end
    
        return player_manager:call("get_CurrentPlayer")
    elseif gn == "dmc5" then
        local player_manager = sdk.get_managed_singleton(sdk.game_namespace("PlayerManager"))
        if player_manager == nil then return nil end
    
        local player_comp = player_manager:call("get_manualPlayer")
        if player_comp == nil then return nil end

        return player_comp:call("get_GameObject")
    elseif gn == "mhrise" then
        local player_manager = sdk.get_managed_singleton(sdk.game_namespace("player.PlayerManager"))
        if player_manager == nil then return nil end
    
        local player_comp = player_manager:call("findMasterPlayer")
        if player_comp == nil then return nil end

        return player_comp:call("get_GameObject")
    end

    return nil
end

local joint_work = {}

re.on_pre_application_entry("LockScene", function()
    for k, v in pairs(joint_work) do
        v.func(v.mat)
    end

    joint_work = {}
end)

re.on_frame(function()
    local player = get_localplayer()
    if player == nil then return end

    local transform = player:call("get_Transform")
    if transform == nil then return end

    local mat = transform:call("get_WorldMatrix")
    local changed = false

    changed,mat = draw.gizmo(transform:get_address(), mat)

    if changed then
        transform:set_rotation(mat:to_quat())
        transform:set_position(mat[3])
    end

    local joints = transform:call("get_Joints")
    local mouse = imgui.get_mouse()

    for i, joint in ipairs(joints:get_elements()) do
        mat = joint:call("get_WorldMatrix")

        local mat_screen = draw.world_to_screen(mat[3])
        local mat_screen_top = draw.world_to_screen(mat[3] + Vector3f.new(0, 0.1, 0))

        if mat_screen and mat_screen_top then
            local delta = (mat_screen - mat_screen_top):length()
            local mouse_delta = (mat_screen - mouse):length()
            if mouse_delta <= delta then

                changed, mat = draw.gizmo(joint:get_address(), mat)

                if changed then
                    table.insert(joint_work, { ["mat"] = mat, ["func"] = function(mat)
                        joint:call("set_Rotation", mat:to_quat())
                        joint:call("set_Position", mat[3])
                    end
                })
                end
            end
        end
    end
end)
```

### RE2/RE3 material toggler with keybinding system
```lua
local game_name = reframework:get_game_name()
if game_name ~= "re2" and name ~= "re3" then
    re.msg("This script is only for RE2 or RE3")
    return
end

local display_children = nil
local display_siblings = nil

local waiting_for_input_map = {}
local key_bindings = {}
local prev_key_states = {}

local function was_key_down(i)
    local down = reframework:is_key_down(i)
    local prev = prev_key_states[i]
    prev_key_states[i] = down

    return down and not prev
end

local function display_mesh(transform)
    local gameobj = transform:get_GameObject()
    if gameobj == nil then return end

    imgui.set_next_item_open(true, 2)
    imgui.push_id(gameobj:get_address())

    -- Look for via.render.Mesh components within the game object.
    -- It will have the materials we can toggle.
    if imgui.tree_node(gameobj:get_Name()) then
        -- Object explorer display for debugging.
        if imgui.tree_node("Object explorer") then
            object_explorer:handle_address(gameobj:get_address())
            imgui.tree_pop()
        end

        local mesh = gameobj:call("getComponent(System.Type)", sdk.typeof("via.render.Mesh"))

        -- Now display the materials in the mesh.
        if mesh ~= nil then
            imgui.text("Materials: " .. tostring(mesh:get_MaterialNum()))
            for i=0, mesh:get_MaterialNum()-1 do
                imgui.push_id(i)

                local name = mesh:getMaterialName(i)
                local enabled = mesh:getMaterialsEnable(i)

                local bound_key = key_bindings[name]
                local is_key_down = bound_key ~= nil and was_key_down(bound_key)

                if imgui.checkbox(name, enabled) or is_key_down then
                    mesh:setMaterialsEnable(i, not enabled)
                end

                imgui.same_line()
                if not waiting_for_input_map[name] then
                    if imgui.button("bind key") then
                        waiting_for_input_map[name] = true
                    end

                    if key_bindings[name] ~= nil then
                        imgui.same_line()
                        if imgui.button("clear") then
                            key_bindings[name] = nil
                        end

                        imgui.same_line()
                        imgui.text_colored("key: " .. tostring(key_bindings[name]), 0xFF00FF00)
                    end
                else
                    imgui.text_colored("Press a key to bind", 0xFF00FFFF)

                    local key = reframework:get_first_key_down()
                    if key ~= nil then
                        key_bindings[name] = key
                        waiting_for_input_map[name] = false
                    end
                end

                imgui.pop_id()
            end
        else
            imgui.text("No via.render.Mesh component found")
        end

        imgui.tree_pop()
    end

    imgui.pop_id()
end

display_children = function(transform)
    local child = transform:get_Child()

    if child ~= nil then
        display_mesh(child)
        display_children(child)
        display_siblings(child)
    end
end

display_siblings = function(transform)
    local next = transform:get_Next()

    if next ~= nil then
        display_mesh(next)
        display_children(next)
        display_siblings(next)
    end
end

re.on_draw_ui(function()
    -- Obtain the FigureManager singleton.
    local figure_manager = sdk.get_managed_singleton(sdk.game_namespace("FigureManager"))

    if figure_manager == nil then
        imgui.text("FigureManager not found")
        return
    end

    if imgui.tree_node("Material toggler") then
        -- Get the current figure/model being displayed.
        local figure = figure_manager:get_CurrentFigureObj()

        if figure ~= nil then
            local figure_name = figure:get_Name()
            imgui.text("Current figure: " .. figure_name)

            local transform = figure:get_Transform()

            -- Go through all of the children transforms and look for mesh components.
            -- The mesh components will have the materials we can toggle.
            display_children(transform)
        else
            imgui.text("No figure found")
        end

        imgui.tree_pop()
    end
end)
```

### Dumping fields of an REManagedObject or type (very verbose)
Use `object:get_type_definition():get_fields()` for an easier way to do this. The below snippet should rarely be used.

```lua
-- type is the "typeof" variant, not the type definition
local function dump_fields_by_type(type)
    log.info("Dumping fields...")

    local binding_flags = 32 | 16 | 4 | 8
    local fields = type:call("GetFields(System.Reflection.BindingFlags)", binding_flags)

    if fields then
        fields = fields:get_elements()

        for i, field in ipairs(fields) do
            log.info("Field: " .. field:call("ToString"))
        end
    end
end

local function dump_fields(object)
    local object_type = object:call("GetType")

    dump_fields_by_type(object_type)
end
```

## Monster Hunter Rise
### Getting the local player
```lua
local function get_localplayer()
    local playman = sdk.get_managed_singleton("snow.player.PlayerManager")

    if not playman then 
         return 
    end

    return playman:call("findMasterPlayer")
end
```

## Devil May Cry 5
### Getting the local player
```lua
local function get_localplayer()
    local playman = sdk.get_managed_singleton(sdk.game_namespace("PlayerManager"))

    if not playman then
        return nil
    end

    return playman:call("get_manualPlayer")
end
```

## Resident Evil 2/3
### Getting the local player
```lua
local function get_localplayer()
    local playman = sdk.get_managed_singleton(sdk.game_namespace("PlayerManager"))

    if not playman then
        return nil
    end

    return playman:call("get_CurrentPlayer")
end
```

## Resident Evil 8
### Getting the local player
```lua
local function get_localplayer()
    if not propsman then
        propsman = sdk.get_managed_singleton(sdk.game_namespace("PropsManager"))
    end

    return propsman:call("get_Player")
end
```

## General
### Spinner and progress bar in ImGui
```lua
local progress = 0.0

re.on_frame(function()
    progress = progress + 0.001
    if progress > 1.0 then 
        progress = 0.0
    end
end)

local function lerp(x0, x1, t)
    return (1.0 - t) * x0 + t * x1
end

local function interval(t0, t1, tween_func)
    return function(t)
        --return t < t0 and 0.0 or t > t1 and 1.0 or tween_func((t - t0) / (t1 - t0))
        if t < t0 then
            return 0.0
        elseif t > t1 then
            return 1.0
        end
        
        return tween_func((t - t0) / (t1 - t0))
    end
end

local function sawtooth(x, t)
    return math.fmod(x * t, 1.0)
end

local function cubic_bezier(t, p0, p1, p2, p3)
    local u = 1.0 - t
    return p0 * u*u*u + p1 * 3.0 * u*u*t + p2 * 3.0 * u*t*t + p3 * t*t*t
end

local function stroke_head_tween(d, t)
    t = sawtooth(d, t)
    return interval(0.0, 0.5, function(x) return cubic_bezier(x, 0.2, 0.0, 0.4, 1.0) end)(t)
end

local function stroke_tail_tween(d, t)
    t = sawtooth(d, t)
    return interval(0.5, 1.0, function(x) return cubic_bezier(x, 0.2, 0.0, 0.4, 1.0) end)(t)
end

local function step_tween(x, t)
    return math.floor(lerp(0.0, x, t))
end

-- https://github.com/ocornut/imgui/issues/1901
local function draw_spinner(center, radius, color, thickness)
    local rect = {
        imgui.get_cursor_pos(),
        imgui.get_cursor_pos() + Vector2f.new(radius * 2, radius * 2) -- todo: frame padding
    }

    imgui.item_size(rect[1], rect[2])
    if not imgui.item_add(rect[1], rect[2], "circle") then
        --print("oh no")
        --return
    end

    local period = 5.0
    local t = math.fmod(os.clock(), period) / period

    imgui.draw_list_path_clear()

    local num_segments = 24

    local num_detents = 5
    local skip_detents = 3

    local head_value = stroke_head_tween(num_detents, t);
    local tail_value = stroke_tail_tween(num_detents, t);
    local step_value = step_tween(num_detents, t);
    local rotation_value = sawtooth(num_detents, t);

    local min_arc =  30.0 / 360.0 * 2.0 * math.pi
    local max_arc = 270.0 / 360.0 * 2.0 * math.pi
    local step_offset = skip_detents * 2.0 * math.pi / num_detents
    local rotation_compensation = math.fmod(4.0*math.pi - step_offset - max_arc, 2.0 * math.pi);

    local start_angle = -math.pi * 2.0
    local a_min = start_angle + tail_value * max_arc + rotation_value * rotation_compensation - step_value * step_offset;
    local a_max = a_min + (head_value - tail_value) * max_arc + min_arc;

    for i = 0, num_segments - 1 do
        local a = a_min + (i / num_segments) * (a_max - a_min)
        local x = center.x + math.cos(a) * radius
        local y = center.y + math.sin(a) * radius
        imgui.draw_list_path_line_to(Vector2f.new(x, y))
    end

    imgui.draw_list_path_stroke(color, false, thickness)
end

re.on_draw_ui(function()
    local center = imgui.get_cursor_pos() + imgui.get_window_pos()
    local radius = 10
    local color = 0x5050BFFF
    local thickness = 2

    draw_spinner(Vector2f.new(center.x + radius, center.y + radius), radius, color, thickness)

    imgui.same_line()
    imgui.progress_bar(progress, Vector2f.new(200, 20), string.format("Progress: %.1f%%", progress * 100))
end)
```

# Object Explorer

The object explorer will be your go-to reference when actively working on a script or a plugin. It will provide you with tools for examining and modifying objects.

Found under `DeveloperTools` in the REFramework menu.


<video width="640" height="480" controls>
<source src="https://user-images.githubusercontent.com/2909949/176354040-b118473d-2def-4439-bdb9-c8899497aae4.mp4" type="video/mp4">
</video>


## Definitions
### TDB
**T**ype **D**ata**b**ase. Contains all of the metadata for classes, fields, methods, events, etc...

Comparable to IL2CPP metadata in Unity.

## Finding game functions to call, and fields to grab
Poke around the singletons until you find something you're interested in. 

Objects under `Singletons` can be obtained with `sdk.get_managed_singleton("name")`

Objects under `Native Singletons` can be obtained with `sdk.get_native_singleton("name")`

Do note that the `Singletons` (AKA Managed Singletons) are the usually the most exposed. They were originally written in C#.

`Native Singletons` have fields and methods exposed, but they are usually hand picked. These ones were written in C++, and have the least amount of data exposed about them.

Anything under `TDB Methods` or `TDB Fields` of something within the `ObjectExplorer` can be called or grabbed using the various call and field getter/setter methods found here in the wiki. 

You **CANNOT** use the `Reflection Methods` or `Reflection Properties` yet without direct memory reading/writing, only the TDB versions are fully supported.

## Dump SDK
This button will create a few things.

1. A `il2cpp_dump.json` in your game folder
2. An `sdk` folder with C++ headers and sources generated from the TDB data

The `il2cpp_dump.json` is usually the most relevant. It can be used as an offline reference for looking up fields and methods. It can be parsed to your liking with Python or your go-to programming or scripting language.

This dump can take a few minutes to run, so expect your game to freeze. The dump will be quite large, and seem to get larger with each new game that comes to the RE Engine (MHRise's is almost 1GB). Keep this in mind when choosing a text editor to view the file.

Python scripts that make use of the il2cpp dump can be found [Here](https://github.com/praydog/REFramework/tree/master/reversing/scripts) and [Here](https://github.com/praydog/REFramework/tree/master/reversing/rsz)

<details>
<summary>Example piece of il2cpp_dump.json output in RE8</summary>
<pre><code lang=json>
"app.PropsManager": {
    "address": "14814d4f0",
    "crc": "c3e89da7",
    "deserializer_chain": [
        {
            "address": "0x14602b540",
            "name": "via.Object"
        },
        {
            "address": "0x14602a530",
            "name": "System.Object"
        },
        {
            "address": "0x14602a850",
            "name": "via.Component"
        },
        {
            "address": "0x14602a9d0",
            "name": "via.Behavior"
        }
    ],
    "fields": {
        "&lt;Camera>k__BackingField": {
            "flags": "Private",
            "id": 110417,
            "init_data_index": 0,
            "offset_from_base": "0x60",
            "offset_from_fieldptr": "0x10",
            "type": "via.Camera"
        },
        "&lt;Player>k__BackingField": {
            "flags": "Private",
            "id": 110416,
            "init_data_index": 0,
            "offset_from_base": "0x58",
            "offset_from_fieldptr": "0x8",
            "type": "via.GameObject"
        },
        "FlotageProcess": {
            "flags": "FamANDAssem | Family",
            "id": 110418,
            "init_data_index": 0,
            "offset_from_base": "0x68",
            "offset_from_fieldptr": "0x18",
            "type": "app.FlotageProcess"
        },
        "SwingRopeProcess": {
            "flags": "FamANDAssem | Family",
            "id": 110419,
            "init_data_index": 0,
            "offset_from_base": "0x70",
            "offset_from_fieldptr": "0x20",
            "type": "app.SwingRopeProcess"
        }
    },
    "flags": "Public | BeforeFieldInit | NativeCtor | ManagedVTable",
    "fqn": "cdbfb0f2",
    "id": 75313,
    "methods": {
        ".ctor550755": {
            "flags": "FamANDAssem | Family | HideBySig | SpecialName | RTSpecialName",
            "function": "1400522b0",
            "id": 550755,
            "impl_flags": "EmptyCtor | HasThis",
            "invoke_id": 3,
            "returns": {
                "name": "",
                "type": "System.Void"
            }
        },
        "doAwake550751": {
            "flags": "Family | Virtual | HideBySig",
            "function": "1417678a0",
            "id": 550751,
            "impl_flags": "HasThis",
            "invoke_id": 3,
            "returns": {
                "name": "",
                "type": "System.Void"
            },
            "vtable_index": 16
        },
        "doLateUpdate550754": {
            "flags": "Family | Virtual | HideBySig",
            "function": "1400b52d0",
            "id": 550754,
            "impl_flags": "HasThis",
            "invoke_id": 3,
            "returns": {
                "name": "",
                "type": "System.Void"
            },
            "vtable_index": 19
        },
        "doOnDestroy550750": {
            "flags": "Family | Virtual | HideBySig",
            "function": "1400b1410",
            "id": 550750,
            "impl_flags": "HasThis",
            "invoke_id": 3,
            "returns": {
                "name": "",
                "type": "System.Void"
            },
            "vtable_index": 20
        },
        "doStart550752": {
            "flags": "Family | Virtual | HideBySig",
            "function": "1400b3780",
            "id": 550752,
            "impl_flags": "HasThis",
            "invoke_id": 3,
            "returns": {
                "name": "",
                "type": "System.Void"
            },
            "vtable_index": 17
        },
        "doUpdate550753": {
            "flags": "Family | Virtual | HideBySig",
            "function": "14176e430",
            "id": 550753,
            "impl_flags": "HasThis",
            "invoke_id": 3,
            "returns": {
                "name": "",
                "type": "System.Void"
            },
            "vtable_index": 18
        },
        "get_Camera550748": {
            "flags": "FamANDAssem | Family | HideBySig | SpecialName",
            "function": "140061200",
            "id": 550748,
            "impl_flags": "HasRetVal | HasThis",
            "invoke_id": 4,
            "returns": {
                "name": "",
                "type": "via.Camera"
            }
        },
        "get_Player550746": {
            "flags": "FamANDAssem | Family | HideBySig | SpecialName",
            "function": "14005a350",
            "id": 550746,
            "impl_flags": "HasRetVal | HasThis",
            "invoke_id": 4,
            "returns": {
                "name": "",
                "type": "via.GameObject"
            }
        },
        "set_Camera550749": {
            "flags": "FamANDAssem | Family | HideBySig | SpecialName",
            "function": "140062dc0",
            "id": 550749,
            "impl_flags": "HasThis",
            "invoke_id": 17,
            "params": [
                {
                    "name": "value",
                    "type": "via.Camera"
                }
            ],
            "returns": {
                "name": "",
                "type": "System.Void"
            }
        },
        "set_Player550747": {
            "flags": "FamANDAssem | Family | HideBySig | SpecialName",
            "function": "14005b6b0",
            "id": 550747,
            "impl_flags": "HasThis",
            "invoke_id": 17,
            "params": [
                {
                    "name": "value",
                    "type": "via.GameObject"
                }
            ],
            "returns": {
                "name": "",
                "type": "System.Void"
            }
        }
    },
    "parent": "app.SingletonBehavior`1<app.PropsManager>",
    "properties": {
        "Camera": {
            "getter": "get_Camera",
            "id": 126015,
            "setter": "set_Camera"
        },
        "Player": {
            "getter": "get_Player",
            "id": 126014,
            "setter": "set_Player"
        }
    },
    "size": "78"
}
</code></pre>
</details>

## Singletons
Are generally global managers dedicated to certain parts of the game, e.g. `app.EnemyManager` for enemies, `app.InteractManager` for interactions, etc...

## Native Singletons
Are also global managers, but they were created in C++ instead of C#. This means they may not have as much data exposed about them, if any at all.

These singletons are usually much more related to engine behavior than the usual `Singletons`.
    
## TDB Fields
Lists all of the fields for a given type visible within the TDB.

## TDB Methods
Lists all of the methods for a given type visible within the TDB. Can right click on any method to open a context menu.

### Context Menu
#### Copy Address
#### Copy Name
#### Hook
Hooks the method and opens a separate window, adds onto it if it already exists. The window contains each method you've hooked from the Object Explorer. 

Each method contains
* Skip function call
* Call count
    
Useful for debugging if you need to know if a method gets called or not. You can also choose to skip calling the original method.


Here you'll find various solutions and FAQ to various problems you may encounter with the VR mod.

[Getting Started Guide](https://beastsaber.notion.site/beastsaber/Praydog-s-Resident-Evil-2-3-VR-mod-3db8bd110ebf4a38870e1a5114b16998)

Newer builds can be found [here](https://github.com/praydog/REFramework-nightly/releases/) (master branch only)

The old pre-RT beta builds of RE2/RE3/RE7 may be more stable with the mod on some computers. You can switch to the beta in Steam under the game's properties. Once this is done, the old version of the mod will need to be downloaded, these are the zip files in the release with "TDB" in them.

## Reporting a bug
Report it on the [Issues](https://github.com/praydog/REFramework/issues) page.

If you are crashing, or are having a technical problem then upload these files from your game folder:
* `re2_framework_log.txt`
* `reframework_crash.dmp` if you are crashing

If you do not have an `reframework_crash.dmp` and are crashing, download a newer build, links at the top of the page.

## Trying newer/beta builds (pd-upscaler)
GitHub account required: [https://github.com/praydog/REFramework/actions/](https://github.com/praydog/REFramework/actions/)

## Opening the in-game menu with motion controllers (OpenVR only right now)
Aim at the palm of your left hand with your head and your right hand. Do not press anything, and an overlay menu should show up.

If that doesn't work, you can use the desktop version of the menu using the Insert key. This method won't work in the headset.

If that still doesn't work, options can be changed in the `re2_fw_config.txt` in your game directory.


## For those with motion sickness
Enable "Decoupled Camera Pitch" under "VR" in the REFramework menu. This will stop the camera from moving vertically in any way. Do note that while this may not necessarily break anything, it may make it less clear of what to do in certain parts of the game when the camera is supposed to shift vertically, or what the camera is intending to look at in a cutscene.

## Common fixes
* Restarting SteamVR
* Disabling overlay software
* Disabling SteamVR theater
* Disabling "Hardware-accelerated GPU scheduling"
   * This **MUST** be disabled if you are getting extremely low frames
* Swapping between DX11 and DX12
* Taking the headset off and putting it back on
* If your game appears "rainbow" colored, or you are stuck in the SteamVR void
    * You have an HDR monitor and HDR must be disabled in some way
    * Unplugging the monitor temporarily has been a reported fix
    * Also setting the game to windowed mode can fix this, HDR sometimes gets forced on in fullscreen
* If your screen looks squished with black bars turn off Borderless window mode
* Make sure no graphical settings are being forced globally (e.g. from Nvidia Control Panel)
    * The exception to this is disabling HDR which is required or else the game will not display within the headset

### In RE2
There is a known issue of a softlock sometimes occurring in the Birkin fight if it goes on too long. It can be fixed simply by disabling FirstPerson until he spawns again.

## Switching to OpenXR
By default, REFramework uses OpenVR for the VR functionality. In some cases, switching to OpenXR can increase performance anywhere from slightly, to a significant amount. The most significant gains have been observed to come when running the games in DX12, but your mileage may vary.

To switch to OpenXR, simply delete the `openvr_api.dll` that came with the zip file. Make sure the `openxr_loader.dll` that came with the mod is present in the game folder.

Not all headsets may have an OpenXR runtime. Headsets like the Index which run natively through SteamVR may not see a performance increase.

If you are using an Oculus headset or a headset that has its own dedicated OpenXR runtime, it is recommended to switch to the runtime provided by your headset manufacturer, e.g. the Oculus OpenXR runtime for Oculus headsets. Using SteamVR as the runtime is only recommended if your headset does not have a dedicated runtime, or are using something like Virtual Desktop.

## OpenXR Pitfalls
* There is no wrist overlay for modifying VR settings yet
* Modifying controller bindings is not as expressive as OpenVR
* Personally only tested on Oculus Quest 2 and CV1, reports that it works on Reverb G2

## What about the others like DMC5 and MHRise?
They are both fully 6DOF but with the least support.

~~They have the same issue of audio positioning being incorrect~~ **only in MHRise now**.

~~DMC5 has some issues with some incorrect UI elements.~~ Fixed in a recent update.

## Gameplay
### All games
#### Switching Weapons
On supported controllers, bound to left trigger + joystick by default. Otherwise, "weapondial" will need to be bound to something, or the d-pad bindings will need to be bound.

### RE2 and RE3
### General
* Motion controller support
* Head-based movement
* Smooth locomotion
* Smooth turning
* Mostly right-handed

Playing with a gamepad is supported. IK gets disabled when using one.

### Gestures
#### Opening the map
Can be done by pressing the inventory button while holding your controller behind your head/over your shoulder.

### Additional options
#### Disabling the crosshair
The option to disable it is under "Script Generated UI" in the REFramework menu. The corresponding script can also be removed from the `reframework/autorun` folder.

---

### RE7 and RE8
#### General
* Motion controller support
* Head-based movement
* Smooth locomotion
* Smooth turning
* Mostly right-handed

Playing with a gamepad is supported.

### Controls not working?
1. Unplug or disconnect your gamepad. The gamepad conflicts with the VR controls.
2. Not all controllers may have proper default bindings, and will need to be manually bound

### Body is annoying or getting in the way?
Body parts can be selectively disabled under "RE8VR" in the REFramework menu

### Want to play without facegun or motion controls, or any additional features, only VR?
Just delete `re8_vr.lua` from the `reframework` directory.

### Broken graphical settings
(RE7) Ambient occlusion must be set to SSAO or Off. The max setting is broken/buggy.

### Gestures
#### Opening the map
Can be done by pressing the inventory button while holding your controller behind your head/over your shoulder.

#### Blocking
Hold your hands in front of your face.

#### Healing
Reach behind your head with your right hand, hold down the grip, and a medicine bottle will appear in your hand. Press right trigger to initiate a heal.

A softlock can occur in the first fight with Mia if you pull out the bottle.

### Additional options
#### Disabling the crosshair
The option to disable it is under "Script Generated UI" in the REFramework menu.

## Bindings
Bindings can be changed in SteamVR's controller bindings section.

Known working default bindings:
* Oculus Touch
* Valve Index Knuckles

Needs additional testing:
* Vive Wands

If a set of controllers don't work as expected, they can be set up in the SteamVR controller bindings.

In OpenXR, the bindings can be changed under "VR".

## Performance
One of the most taxing parts of these mods is the **resolution** you have set. The in-game resolution has no effect, it must be changed in SteamVR.

To modify the resolution in your headset:
* Open the SteamVR overlay
* Click on the cogwheel on the bottom right
* Click on "Video"
* Change "Render Resolution" to "Custom" and then lower the resolution until it is playable

You can also use [openvr_fsr](https://github.com/fholger/openvr_fsr) with this mod.

There are other demanding in-game quality settings (ranked by approximate performance impact):
* Ray Tracing (RE8 only at the moment, will need a very powerful GPU to run this at a good framerate)
* Image Quality (set this to 100% if you're not sure, lower it if it improves performance)
* Shadow Quality
* Screen Space Reflections
* Ambient Occlusion
* Subsurface Scattering

Some other ones:
* Mesh Quality

You may want to start on all low or use the "Performance Priority" preset and work your way up to acceptable settings.

The Lua scripts can have a minor impact on performance. If you don't mind playing without physical knifing and physical grenade throwing, you can remove their respective scripts from the `autorun` folder in your game directory.

Enabling AFR/AER can be done under the "VR" section of the menu.

## What graphical settings are broken?
Volumetric Lighting, Lens Flares, TAA, and Motion Blur. Will need the help of someone more experienced with shaders to fix these.

TAA has a partial fix in the latest nightly builds.

## What graphical settings are forced?
These are forced, but the forcing can be toggled off in REFramework's menu, under VR.

* FPS, gets forced to "Variable" (uncapped)
* Antialiasing, gets forced to "None" if using any TAA variant
* Motion Blur, gets forced to "Off"
* VSync, gets forced to "Off"
* Lens Distortion, gets forced to "Off"
* Lens Flares, gets forced to "Off"
* Volumetric Lighting, gets forced to "Off"

These forced changes are not visual in the options menu, but will take effect.


