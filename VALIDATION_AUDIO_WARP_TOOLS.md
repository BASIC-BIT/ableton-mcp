# Audio Clip and Warp Tool Validation

This branch is not PR-ready based on a shallow smoke test alone. The notes below record what was tested, what was observed, and what still needs to be learned or repeated before asking upstream to review the change.

## Environment

- Date: 2026-05-24
- OS: Windows
- Ableton process: Ableton Live 12 Suite 12.4
- Remote Script installed locally in Ableton User Library as `AbletonMCP` and `AbletonMCPDev` during testing.
- MCP server command used by OpenCode config: `uvx --from D:/bench/ableton-mcp ableton-mcp`
- Scratch audio file: `D:\bench\sandbox\mcp_audio_tool_test.wav`
- Scratch audio metadata: PCM 16-bit WAV, 48000 Hz, stereo, 4.0 seconds, 768078 bytes.

## Live Object Model References

- `ClipSlot.create_audio_clip(path)`: creates a Session View audio clip from an absolute path and errors if the slot is not on an audio track or the track is frozen.
- `Track.create_audio_clip(file_path, position)`: creates an Arrangement View audio clip on an audio track at a beat position.
- `Clip.available_warp_modes`, `Clip.warp_mode`, `Clip.warping`, `Clip.gain`, `Clip.file_path`, `Clip.sample_length`, and `Clip.sample_rate` are audio-clip-only surfaces.
- `Clip.warp_markers` returns marker dictionaries with `sample_time` in seconds and `beat_time` in beats. The final marker can be hidden from the Live UI and is used to calculate the last segment BPM.
- `Clip.add_warp_marker` accepts `beat_time` and optional `sample_time`; omitting `sample_time` asks Live to calculate a timing-preserving sample time.
- `Clip.warping` is deferred internally by Live, so sequencing dependent API calls immediately after setting it can produce unintuitive ordering.

Reference pages:

- https://docs.cycling74.com/apiref/lom/clip/
- https://docs.cycling74.com/apiref/lom/clipslot/
- https://docs.cycling74.com/apiref/lom/track/

## Baseline

- `get_session_info` succeeded before validation.
- Initial session state: tempo `120.0`, signature `4/4`, track count `6`, return track count `2`.
- Existing scratch tracks from the earlier smoke test were present:
  - `MCP TEST - Session Audio`, track index `4`, audio track with a Session audio clip in slot `0`.
  - `MCP TEST - Arrangement Audio`, track index `5`, audio track.

## Validation Cases Run

| Case | Steps | Expected | Observed | Status |
| --- | --- | --- | --- | --- |
| Server import | `uvx --from "D:/bench/ableton-mcp" python -c "import MCP_Server.server as s; print('local fork package import ok')"` | MCP server imports from local branch. | Printed `local fork package import ok`. | Pass |
| Python syntax | `py -m compileall MCP_Server AbletonMCP_Remote_Script` | Python files compile. | Compileall completed for both directories. | Pass |
| Diff hygiene | `git diff --check` | No whitespace errors. | No whitespace errors; Git warned that LF will become CRLF on checkout. | Pass |
| Audio track creation | `create_audio_track(index=-1)`, then `set_track_name(6, "MCP VALIDATION - Audio")` | Creates a new audio track at the end and allows naming it. | Created `7-Audio`, renamed track index `6` to `MCP VALIDATION - Audio`. | Pass |
| MIDI track creation for type checks | `create_midi_track(index=-1)`, then `set_track_name(7, "MCP VALIDATION - MIDI")` | Creates a new MIDI track at the end and allows naming it. | Created `8-MIDI`, renamed track index `7` to `MCP VALIDATION - MIDI`. | Pass |
| Session audio import | `create_audio_clip(track_index=6, clip_index=0, file_path=scratch_wav)` | Creates a Session View audio clip and returns JSON-safe clip info. | Returned audio clip info with `file_path`, `warping=true`, `warp_mode=4`, `sample_length=192000`, `sample_rate=48000.0`, gain `0.4000000059604645`, and warp markers. | Pass |
| Arrangement audio import | `create_arrangement_audio_clip(track_index=6, file_path=scratch_wav, position=16)` | Creates an Arrangement clip at beat position 16 and returns summary info. | Returned `created=true`, `position=16.0`, `arrangement_clip_count=1`, and `last_arrangement_clip` summary. | Pass |
| Detailed clip info | `get_clip_info(track_index=6, clip_index=0)` | Returns Session clip details without serialization failures. | Returned summary, markers, sample metadata, gain, pitch, loop markers, and warp state. | Pass |
| Gain mutation | `set_clip_gain(track_index=6, clip_index=0, gain=0.5)` | Sets normalized clip gain and returns updated info. | Returned `gain=0.5` and display string `4.00 dB`. | Pass |
| Warp mode mutation | `set_clip_warp_mode(track_index=6, clip_index=0, warp_mode=6)` | Sets Complex Pro mode and returns updated info. | Returned `warp_mode=6`. | Pass |
| Warping off | `set_clip_warping(track_index=6, clip_index=0, warping=false)` | Disables warping and returns updated info. | Returned `warping=false`; Live also changed `looping=false`, `loop_end=4.0`. | Pass with caution |
| Warping on | `set_clip_warping(track_index=6, clip_index=0, warping=true)` | Enables warping and returns updated info. | Returned `warping=true`; Live changed `end_marker=16.0`, `loop_end=8.0`, and reduced warp markers to start/hidden marker. | Pass with caution |
| Add explicit warp marker | `add_warp_marker(track_index=6, clip_index=0, beat_time=2, sample_time=1)` | Adds a marker at beat 2 mapped to sample time 1 second. | Returned marker `{beat_time: 2.0, sample_time: 1.0}` and hidden marker near `{beat_time: 2.03125, sample_time: 1.015625}`. | Pass |
| Move warp marker | `move_warp_marker(track_index=6, clip_index=0, beat_time=2, beat_time_distance=0.5)` | Moves marker to beat 2.5 while preserving sample time. | Returned marker `{beat_time: 2.5, sample_time: 1.0}` and hidden marker near `{beat_time: 2.53125, sample_time: 1.015625}`. | Pass |
| Remove warp marker | `remove_warp_marker(track_index=6, clip_index=0, beat_time=2.5)` | Removes the marker at beat 2.5. | Returned only start plus hidden marker. | Pass |
| Marker defaults via Live | `add_warp_marker(track_index=6, clip_index=0, beat_time=4, sample_time=null)`, then remove beat 4 | Omitting sample time lets Live calculate it without changing playback timing. | Added `{beat_time: 4.0, sample_time: 1.6}` based on current warped state, then removal succeeded. | Pass |
| Clip markers restore | `set_clip_markers(track_index=6, clip_index=0, start_marker=0, end_marker=8, loop_start=0, loop_end=8, looping=true)` | Sets start/end/loop fields explicitly. | Returned `start_marker=0.0`, `end_marker=8.0`, `loop_start=0.0`, `loop_end=8.0`, `looping=true`. | Pass |
| MIDI clip info | `create_clip(track_index=7, clip_index=0, length=4)`, then `get_clip_info(track_index=7, clip_index=0)` | MIDI clips return common clip details and omit audio-only fields. | Returned `is_audio_clip=false`, `is_midi_clip=true`, length and loop markers; no audio-only fields. | Pass |
| Audio import on MIDI track | `create_audio_clip(track_index=7, clip_index=1, file_path=scratch_wav)` | Fails clearly because the track is not audio. | Returned `Track is not an audio track`. | Pass |
| Audio gain on MIDI clip | `set_clip_gain(track_index=7, clip_index=0, gain=0.5)` | Fails clearly because the clip is not audio. | Returned `Clip is not an audio clip`. | Pass |
| Empty slot info | `get_clip_info(track_index=6, clip_index=7)` | Fails clearly because there is no clip. | Returned `No clip in slot`. | Pass |
| Missing local audio file | `create_audio_clip(track_index=6, clip_index=1, file_path="D:\\bench\\sandbox\\does-not-exist.wav")` | Server-side path validation fails before sending the command to Ableton. | Returned `Audio file not found: D:\bench\sandbox\does-not-exist.wav`. | Pass |
| Invalid warp mode | `set_clip_warp_mode(track_index=6, clip_index=0, warp_mode=7)` | Server-side validation rejects out-of-range mode. | Returned `warp_mode must be between 0 and 6`. | Pass |
| Invalid gain | `set_clip_gain(track_index=6, clip_index=0, gain=1.5)` | Server-side validation rejects out-of-range gain. | Returned `gain must be between 0.0 and 1.0`. | Pass |
| Occupied slot import | `create_audio_clip(track_index=6, clip_index=0, file_path=scratch_wav)` | Fails clearly because the target slot already has a clip. | Returned `Clip slot already has a clip`. | Pass |
| Out-of-range track | `get_track_info(track_index=99)` | Fails clearly because the track index does not exist. | Returned `Track index out of range`. | Pass |
| Arrangement import on MIDI track | `create_arrangement_audio_clip(track_index=7, file_path=scratch_wav, position=24)` | Fails clearly because the track is not audio. | Returned `Track is not an audio track`. | Pass |

Final session state after validation: tempo `120.0`, signature `4/4`, track count `8`, return track count `2`.

## Important Findings

- The main-thread routing change is necessary for reads and writes. Before routing read commands through Ableton's main thread, session/clip reads could time out or fail with Live Object Model signature mismatches.
- `Live.Clip.WarpMarker(beat_time=..., sample_time=...)` is required in the Remote Script. Passing a plain Python dict into `clip.add_warp_marker` failed in this Python Remote Script context.
- `sample_time` in `warp_markers` and `add_warp_marker` is seconds, not sample frames. `sample_length` is sample frames and `sample_rate` can be used for bounds reasoning.
- `available_warp_modes` and `warp_markers` are vector-like Live API return values in this context. They must be converted to JSON-safe primitives before sending responses.
- `Clip.warping` is not a pure isolated boolean toggle. In Live 12.4, toggling warping off and on during this test also changed loop state, end markers, and the warp-marker set. Any DJ-prep workflow should re-read the clip after toggling warping and then explicitly set desired markers/loop state.
- The last warp marker returned by the API may be hidden from the Live UI. This is expected per the LOM docs and should not be treated as a stray visible marker.

## Known Gaps Before PR

- This was tested only on Ableton Live 12 Suite 12.4. The project README says Live 10 or newer, but the new `get_clip_info` behavior relies on some audio clip properties that the official docs mark as Live 11+ for at least `warp_markers` reads and `arrangement_clips`.
- The validation used a short 4-second WAV. It does not prove performance or correctness for long DJ files, very large files, MP3 import, or files with complex auto-warp analysis.
- Arrangement clip import is validated only through the immediate return payload and `arrangement_clip_count`; there is no MCP tool yet to fetch detailed Arrangement clip info by index.
- The MCP tool return values are strings. Error cases are human-readable but not structured enough for automated downstream validation.
- The validation was manual and mutated the open Ableton set. There is no cleanup/delete-track tool in this branch, so scratch tracks remain in the test set unless removed manually in Ableton.
- This branch has not been tested after a fresh Ableton restart since the documentation update. Before PR, restart Ableton, ensure only the intended `AbletonMCP` Remote Script is selected, and repeat the successful-path and error-path checks.

## PR Gate

Do not open the upstream PR until the following are done:

1. Repeat validation after a full Ableton restart with a clean or intentionally scratch Live set.
2. Run at least one long-audio test representative of the DJ-prep use case.
3. Decide whether the branch should claim Live 10+ compatibility, degrade gracefully on older Live versions, or document a narrower Live version requirement for the new audio/warp tools.
4. Decide whether upstream should receive `AGENTS.md` and this validation note, or whether those should stay fork-local while the code changes go upstream separately.
5. Draft the PR body from observed evidence, not operator confidence.
