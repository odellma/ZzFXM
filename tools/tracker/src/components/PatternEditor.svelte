<script>
import { patterns, sequence, channelMeters, patternsMeta } from '../stores.js';
import { addPattern, deletePattern, addChannel, deleteChannel, addRow, deleteRow, createTrack, createPattern, clearRow, setPatternData } from '../services/PatternService.js';
import { playPattern, stopSong } from '../services/RendererService.js';
import { createEventDispatcher } from 'svelte';
import { clamp } from '../lib/utils.js';
import Channel from './Channel.svelte';
import Pane from './Pane.svelte';
import Toolbar from './Toolbar.svelte';
import Field from './Field.svelte';
import Button from './Button.svelte';
import TextProperty from './TextProperty.svelte';
import NumberProperty from './NumberProperty.svelte';
import PianoInput from './PianoInput.svelte';

export let selectedChannel = 0;
export let selectedRow = 0;
export let selectedPattern = 0;

let showPiano = window.matchMedia('(min-height: 48em)').matches;
let channelClipboard;
let patternClipboard;

$: patternCount = $patterns.length;
$: channelCount = $patterns[selectedPattern].length;
$: selectedPattern = clamp(selectedPattern, 0, $patterns.length - 1);
$: selectedChannel = clamp(selectedChannel, 0, $patterns[selectedPattern].length - 1);
$: selectedRow = clamp(selectedRow, 0, $patterns[selectedPattern][0].length - 3) | 0;


$: usage = $sequence.map((pattern, i) => {
  return pattern === selectedPattern && i
}).filter(x => x !== false);


const dispatch = createEventDispatcher();

const channelElems = [];


const handleAddPatternClick = () => {
  selectedPattern = addPattern(createPattern(3, 64));
}

const handleDeletePatternClick = () => {
  deletePattern(selectedPattern);
}

const handleAddRowClick = () => {
  selectedRow = addRow(selectedPattern);
}

const handleDeleteRowClick = () => {
  deleteRow(selectedPattern, selectedRow);
}

const handleClearRowClick = () => {
  clearRow(selectedPattern, selectedRow);
}

const moveChannelLeft = () => {
  swapChannels(selectedChannel, --selectedChannel);
}

const moveChannelRight = () => {
  swapChannels(selectedChannel, ++selectedChannel);
}

const handleAddChannelClick = () => {
  selectedChannel = addChannel(selectedPattern);
}

const deleteChannelClick = () => {
  deleteChannel(selectedPattern, selectedChannel);
}

const clearChannel = () => {
  $patterns[selectedPattern][selectedChannel].fill(0, 2);
  $patterns[selectedPattern] = $patterns[selectedPattern];
}

const swapChannels = (a, b) => {
  const channels = $patterns[selectedPattern].slice();
  [channels[a], channels[b]] = [channels[b], channels[a]];
  $patterns[selectedPattern] = channels;
}

const copyChannel = () => {
  channelClipboard = $patterns[selectedPattern][selectedChannel].slice();
}

const pasteChannel = () => {
  $patterns[selectedPattern][selectedChannel] = channelClipboard.slice();
  $patterns[selectedPattern] = $patterns[selectedPattern];
}

const handleCopyPatternClick = () => {
  patternClipboard = $patterns[selectedPattern].slice();
}

const handlePastePatternClick = () => {
  setPatternData(selectedPattern, patternClipboard.slice());
}

const handlePatternChange = () => {
  dispatch('patternselect');
  selectedRow = 0;
}

const handlePlayClick = () => {
  playPattern(selectedPattern);
}

const handleStopClick = () => {
  stopSong();
}

const handlePianoToggleClick = () => {
  showPiano = !showPiano;
}
</script>


<div class="splitView">
  <Pane>
    <div slot="head">
      <Toolbar>
        <Field label="Patterns">
          <Button label="Add" on:click={handleAddPatternClick} />
          <Button label="Delete" disabled={usage.length} on:click={handleDeletePatternClick} />
          <Button label="Copy" on:click={handleCopyPatternClick} />
          <Button label="Paste" disabled={!patternClipboard} on:click={handlePastePatternClick} />
        </Field>
      </Toolbar>
    </div>
    <select class="select" on:input={handlePatternChange} bind:value={selectedPattern} size="2">
      {#each $patterns as pattern, i}
        <option value={i}>{i}: {$patternsMeta[i]}</option>
      {/each}
    </select>
  </Pane>

  <Pane>
    <div slot="head">
      <Toolbar>
        <NumberProperty label="#" max={patternCount} bind:value={selectedPattern} on:input={handlePatternChange}></NumberProperty>
        <TextProperty label="Name" bind:value={$patternsMeta[selectedPattern]}></TextProperty>
        <Field label="Playback">
          <Button keyboard="ENTER" label="Play Pattern" on:click={handlePlayClick} />
          <Button keyboard="ENTER" label="Stop" on:click={handleStopClick} />
        </Field>
        <Field label="Track">
          <Button on:click={handleAddChannelClick} label="Add" />
          <Button on:click={clearChannel} label="Clear" />
          <Button on:click={copyChannel} label="Copy" />
          <Button disabled={!channelClipboard} on:click={pasteChannel} label="Paste" />
          <Button disabled={channelCount === 1} on:click={deleteChannelClick} label="Delete" />
          <Button disabled={channelCount === 1 || selectedChannel === 0} on:click={moveChannelLeft} label="Move Left" />
          <Button disabled={channelCount === 1 || selectedChannel === channelCount - 1} on:click={moveChannelRight} label="Move Right" />
        </Field>
        <Field label="Row">
          <Button on:click={handleAddRowClick} label="Add" />
          <Button disabled={channelCount === 1} on:click={handleDeleteRowClick} label="Delete" />
          <Button label="Clear" on:click={handleClearRowClick} />
        </Field>
        <Field label="Tools">
          <Button on:click={handlePianoToggleClick} label="Toggle Piano" />
        </Field>
        <div class="outset"></div>
      </Toolbar>
    </div>
    <div class="channels outset">
      {#each $patterns[selectedPattern] as channel, i}
        <div class="channel inset" bind:this={channelElems[i]} on:focusin={()=>selectedChannel = i} class:selected={i==selectedChannel}>
          <Channel on:rowselect title={`Track ${i}`} bind:selectedRow={selectedRow} data={channel}></Channel>
          <div class="level" style="transform:scaleY({$channelMeters[i] || 0})"></div>
        </div>
      {/each}
    </div>
    {#if showPiano}
      <PianoInput />
    {/if}
  </Pane>
</div>

<style>
  select {
    display: block;
    width: 100%;
    height: 100%;
    margin: 0;
  }
  .selected {
    background: var(--active-selection-color);
  }
  .channels {
    display: grid;
    grid-auto-flow: column;
    grid-auto-columns: minmax(120px, 1fr);
    overflow: auto;
    gap: 10px;
    padding: 10px
  }
  .channel {
    position:relative;
  }
  .level {
    position: absolute;
    width: 32px;
    background: linear-gradient(#fc0, #0f0);
    bottom: 50%;
    left: calc(50% - 16px);
    height: 150px;
    box-shadow: inset 0 0 1px #0004;
    transform-origin: 50% 100%;
  }
</style>