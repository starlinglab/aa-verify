<script>
  import { createEventDispatcher } from "svelte";

  const dispatch = createEventDispatcher();

  function onClick() {
    document.getElementById("file-input").click();
  }

  function onDragOver(evt) {
    evt.preventDefault();
  }

  function onDrop(evt) {
    evt.preventDefault();
    if (evt.dataTransfer.items.length === 1) {
      handleFile(evt.dataTransfer.items[0].getAsFile());
    }
  }

  function onChange(evt) {
    // File maybe uploaded through file picker
    if (evt.target.files.length === 1) {
      handleFile(evt.target.files[0]);
    }
  }

  function handleFile(file) {
    dispatch("fileUpload", { file: file });
  }
</script>

<!-- svelte-ignore a11y-click-events-have-key-events -->
<!-- svelte-ignore a11y-no-static-element-interactions -->
<div
  id="drop-zone"
  on:click={onClick}
  on:drop={onDrop}
  on:dragover={onDragOver}
>
  <svg
    xmlns="http://www.w3.org/2000/svg"
    viewBox="0 0 448 512"
    class="icon"
    aria-hidden="true"
    focusable="false"
    fill="currentColor"
    ><!--!Font Awesome Free 6.6.0 by @fontawesome - https://fontawesome.com License - https://fontawesome.com/license/free Copyright 2024 Fonticons, Inc.--><path
      d="M246.6 9.4c-12.5-12.5-32.8-12.5-45.3 0l-128 128c-12.5 12.5-12.5 32.8 0 45.3s32.8 12.5 45.3 0L192 109.3 192 320c0 17.7 14.3 32 32 32s32-14.3 32-32l0-210.7 73.4 73.4c12.5 12.5 32.8 12.5 45.3 0s12.5-32.8 0-45.3l-128-128zM64 352c0-17.7-14.3-32-32-32s-32 14.3-32 32l0 64c0 53 43 96 96 96l256 0c53 0 96-43 96-96l0-64c0-17.7-14.3-32-32-32s-32 14.3-32 32l0 64c0 17.7-14.3 32-32 32L96 448c-17.7 0-32-14.3-32-32l0-64z"
    /></svg
  >
</div>
<input id="file-input" type="file" on:change={onChange} />

<style>
  #drop-zone {
    border: 5px var(--pico-color) dashed;
    border-radius: 10px;
    padding: 20%;
    background-color: var(--pico-form-element-background-color);
    height: 100%;
    width: 100%;
  }
  #drop-zone:hover {
    cursor: pointer;
  }
  #drop-zone > svg {
    height: 100%;
    width: 100%;
  }
  #file-input {
    appearance: none;
    display: none;
  }
</style>
