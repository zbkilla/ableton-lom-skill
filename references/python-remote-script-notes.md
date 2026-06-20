# Python Remote Script Notes — host differences vs the Max/JS member view

This skill mirrors the Cycling '74 Live Object Model, whose member tables describe the API as seen from **Max for Live / JS** (members often typed `dict`, notes passed as plain dictionaries, etc.). From a **Python control surface / Remote Script** (`from _Framework.ControlSurface import ControlSurface`, working against `self._song` / `self.application()`), several of those members behave differently. These are cross-cutting gotchas verified live on **Live 12.4 (Lite)** — read them alongside the per-object reference files.

> Rule of thumb: the object/member *names* in this skill are reliable; the *representation* (dict vs object, what a method accepts/returns) is the Max/JS projection. When in doubt, introspect the live object (`dir(obj)`, `type(member)`) rather than trusting the type column.

## 1. "dict" members are usually objects in Python

Members the reference tables type as `dict` — routing types/channels, and similar structured values — are **objects** from Python, not literal dictionaries.

- Read fields by **attribute**, not subscript: `track.input_routing_type.display_name` / `.identifier` — *not* `track.input_routing_type['display_name']`.
- Assign the **object itself** (taken from the `available_*` collection), not a dict:
  ```python
  track.input_routing_type = track.available_input_routing_types[0]   # assign the object
  print(track.input_routing_type.display_name)
  ```
- The same applies to object-valued properties like `clip.groove` (a `Groove`) — assign a real `Groove` (e.g. from `song.groove_pool.grooves`), not a string or dict. Note `clip.groove` **cannot be cleared from Python** — `None`/`0` are rejected and there is no Python handle for the LOM null object. (In Max/JS you clear an object-valued property by assigning the null object `id 0`; the Python object API has no equivalent, so reassign a different groove or recreate the clip.)

See `track.md` (routing) and `grooves-tuning.md` (groove).

## 2. Some methods need C++ types you cannot build from Python literals

A few APIs accept a native (C++) specification object. A plain Python `dict`/`tuple` raises `No registered converter … from … type dict` (or `… type str`). You must construct the real type.

- **MIDI notes (modern API).** `clip.add_new_notes(...)` and `clip.apply_note_modifications(...)` need a sequence of **`Live.Clip.MidiNoteSpecification`** objects:
  ```python
  import Live
  spec = Live.Clip.MidiNoteSpecification(
      pitch=60, start_time=0.0, duration=0.5, velocity=100,
      mute=False, probability=1.0, velocity_deviation=0.0, release_velocity=64)
  ids = clip.add_new_notes((spec,))            # iterable of specs, NOT {"notes":[…]}
  ```
  The `{"notes": [...]}` wrapper and bare dicts are the Max/JS form; passing the wrapper makes Live iterate the dict's *keys* (a `str`). See `clip.md`.
- **Automation events.** `Envelope.create_event(time, value)` needs a C++ `TEnvelopeEvent` you can't build from Python — use **`insert_step(time, length, value)`** instead. See `clip.md` ("Writing automation").

## 3. Reads return objects, not list[dict] — and they can be path-less

Methods documented as returning `list[dict]` return **objects** from Python:

- `clip.get_notes_extended(...)` / `get_all_notes_extended(...)` return a **`MidiNoteVector`** of **`MidiNote`** objects. Read each note's fields by attribute (`note.pitch`, `note.start_time`, … `note.note_id`); to edit, mutate the fetched notes in place (their fields are settable) and pass the vector back to `clip.apply_note_modifications(...)`.
- `clip.create_automation_envelope(param)` / `automation_envelope(param)` return an **`Envelope`** object with **no canonical `live_set …` path** — you can only reach it through the returning method, so hold the reference and call its methods directly (`insert_step`, `value_at_time`, `delete_events_in_range`).
- `application().browser.<category>` returns a **`BrowserItem`**; navigate `.children` (also `BrowserItem`s) and check `.is_loadable`.

If you serialize such an object naïvely (e.g. only reading `name`/`value`), the real fields are lost — read the specific attributes you need.

## 4. Parameter values are frequently normalized 0.0–1.0

`DeviceParameter.value` is the internal number between `min`/`max`, which for many parameters is `0.0–1.0` and does **not** match the displayed unit (Auto Filter `Frequency` `0.9` ≈ "10.0 kHz"). Always read `min`/`max` and `str_for_value(value)` before setting; quantized params (`is_quantized`) take discrete indices from `value_items`. Mixer sends (`mixer_device.sends[i]`, one per return track) are likewise `0.0–1.0` `DeviceParameter`s. See `device.md`, `track.md`.

## 5. `browser.load_item` targets the *selected* track

`application().browser.load_item(item)` loads onto **`song.view.selected_track`** — there is no track argument. To load onto a specific track, select it first:
```python
song.view.selected_track = song.return_tracks[0]          # e.g. a return track
app.browser.load_item(app.browser.audio_effects.children[k])   # k = a loadable child
```
Selecting the track *is* the targeting — this is how you load a device onto a **return track**, which take no clip and are otherwise hard to address. See `browser.md`.

## 6. State changes run on the main thread

Mutating the live set (creating tracks/clips, setting values, calling state-changing methods) must happen on Live's main thread. From a socket/listener thread, marshal the call with `schedule_message`; read-only access can happen on any thread. See SKILL.md → "Threading."

## 7. Not every GUI feature is in the LOM

Some things you can do in Live's UI have **no LOM member at all** — they're simply not scriptable. Verify by introspection before assuming a feature is reachable; absence is a real answer.

- **Follow actions** — verified Live 12.4: no `follow_action_*` member on `Clip`, `ClipSlot`, `Scene`, or `Song`. The Launch box's Follow Action settings can't be read or set from **any** host (the LOM lacks them, so Max/JS is no better). They *are* persisted in the `.als` project XML, so the only programmatic route is editing the gzipped project file offline and reopening. (Clip *launch* settings — `launch_mode`, `launch_quantization`, `legato`, `velocity_amount` — *are* exposed.)
- **Some host editions cap creation, not the API** — e.g. Live Lite caps tracks and return tracks: `create_midi_track` / `create_audio_track` / `create_return_track` raise `"Couldn't create track"` at the limit, even though the methods exist. That's an edition limit, not a missing member.

When the GUI has a control the LOM doesn't, record it as a verified absence so the next author doesn't search for it.

---

*This page collects fork-verified, Python-host-specific behavior. The per-object reference files remain the source for member names, types, and access; this page explains where the Python surface diverges from the Max/JS projection those tables describe.*
