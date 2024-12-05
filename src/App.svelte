<script>
  import { onMount } from "svelte";
  import { FFmpeg } from "@ffmpeg/ffmpeg";
  import { fetchFile, toBlobURL } from "@ffmpeg/util";
  import { fade } from "svelte/transition";

  /** @type {FFmpeg} */
  let ffmpeg;

  /** @type {string} */
  let message =
    "Drop a video file below to extract the audio. All processing is done in your web browser.";

  /** @type {string|null} */
  let downloadUrl = null;

  /** @type {HTMLInputElement|null} */
  let fileInput;

  /** @type {boolean} */
  let isLoaded = false;

  onMount(async () => {
    ffmpeg = new FFmpeg();

    const baseURL = "https://unpkg.com/@ffmpeg/core@0.12.6/dist/esm";
    ffmpeg.on("log", ({ message }) => {
      console.log(message);
    });

    try {
      isLoaded = await ffmpeg.load({
        coreURL: await toBlobURL(
          `${baseURL}/ffmpeg-core.js`,
          "text/javascript",
        ),
        wasmURL: await toBlobURL(
          `${baseURL}/ffmpeg-core.wasm`,
          "application/wasm",
        ),
      });
    } catch (e) {
      console.error("Error loading FFmpeg:", e);
      message = "Failed to load ffmpeg.";
    }
  });

  async function processFile(file) {
    if (!file) return;

    try {
      message = "Fetching file...";
      const f = await fetchFile(file);

      if (!ffmpeg.loaded) {
        throw new Error("FFmpeg is not loaded");
      }

      message = "Writing file...";
      await ffmpeg.writeFile("input.mp4", f);
      message = "Executing...";
      await ffmpeg.exec([
        "-i",
        "input.mp4",
        "-q:a",
        "0",
        "-map",
        "a",
        "output.mp3",
      ]);

      const data = await ffmpeg.readFile("output.mp3");

      const audioBlob = new Blob([data], { type: "audio/mp3" });
      downloadUrl = URL.createObjectURL(audioBlob);
      message =
        "Audio extracted successfully! Click the link below to download.";
    } catch (error) {
      console.error("Error during file processing:", error);
      message = "An error occurred while processing the video.";
      downloadUrl = null;
    }
  }

  async function handleDrop(event) {
    event.preventDefault();
    const file = event.dataTransfer.files[0];
    await processFile(file);
  }

  async function handleFileSelect(event) {
    const file = event.target.files?.[0];
    await processFile(file);
  }

  function handleDragOver(event) {
    event.preventDefault();
  }

  function handleClick() {
    fileInput?.click();
  }

  function handleKeyDown(event) {
    if (event.key === 'Enter' || event.key === ' ') {
      handleClick();
    }
  }
</script>

<main ondrop={handleDrop} ondragover={handleDragOver}>
  <h1>Audio Extractor</h1>
  <h3>{message}</h3>

  {#if isLoaded}
    <div transition:fade={{ duration: 200 }}>
      <div class="drop-area" onclick={handleClick} onkeydown={handleKeyDown} tabindex="0" role="button">
        <p>Drag and drop a video file here or click to browse.</p>
      </div>
      <input
        type="file"
        accept="video/*"
        bind:this={fileInput}
        onchange={handleFileSelect}
        style="display: none"
      />
      {#if downloadUrl}
        <div class="download-link">
          <a class="btn" href={downloadUrl} download="output.mp3"
            >Download Audio</a
          >
        </div>
      {/if}
    </div>
  {/if}
</main>

<style>
  .drop-area {
    border: 2px dashed #ccc;
    padding: 2em;
    text-align: center;
    margin: 2em auto;
    cursor: pointer;
    max-width: 360px;
    outline: none;
  }

  .download-link {
    margin-top: 1em;
    text-align: center;
  }
</style>
