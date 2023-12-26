<script lang="ts">
  import { fade } from 'svelte/transition';

  interface StoredMusicConfig {
    filename: string;
    artist: string;
    name: string;
    category: string;
    points: number;
    questionTimings: { start: number; end: number };
    answerTimings: { start: number; end: number };
  }

  interface MusicConfig extends StoredMusicConfig {
    audio: HTMLAudioElement;
    stopTimeout: ReturnType<typeof setTimeout> | null;
    timerInterval: ReturnType<typeof setInterval> | null;
    currentTime: number;
  }

  let musicConfigs: MusicConfig[] | null = null;
  let configsFileHandle: FileSystemFileHandle | null = null;
  let categories: string[] = [];
  let categoriesFileHandle: FileSystemFileHandle | null = null
  let newCategoryName = '';
  const points = [100, 200, 300, 400, 500];


  async function startAudio(config: MusicConfig, timings: 'question' | 'answer') {
    console.log('Start playing');
    const timingsKey = `${timings}Timings` as const;
    config.audio.currentTime = config[timingsKey].start;
    config.currentTime = config[timingsKey].start;
    await config.audio.play();

    config.stopTimeout = setTimeout(
      () => stopAudio(config),
      (config[timingsKey].end - config[timingsKey].start) * 1000,
    );

    config.timerInterval = setInterval(() => {
      config.currentTime += 0.1;
      musicConfigs = musicConfigs;
    }, 100);

    musicConfigs = musicConfigs;
  }

  function stopAudio(config: MusicConfig) {
    if (config.stopTimeout) clearTimeout(config.stopTimeout);
    if (config.timerInterval) clearInterval(config.timerInterval);
    config.audio.pause();
    config.stopTimeout = null;
    config.timerInterval = null;
    musicConfigs = musicConfigs;
  }

  function sortConfigs() {
    if (!musicConfigs) return;

    musicConfigs = musicConfigs.sort((a, b) => {
      return categories.indexOf(a.category) - categories.indexOf(b.category) || a.points - b.points;
    });
  }

  async function saveConfigs() {
    if (!configsFileHandle || !musicConfigs || !categoriesFileHandle) return;
    const songsConfigStream = await configsFileHandle.createWritable({ keepExistingData: false });
    const categoriesConfigStream = await categoriesFileHandle.createWritable({ keepExistingData: false });

    await songsConfigStream.write(
      JSON.stringify(
        musicConfigs.map(
          (c): StoredMusicConfig => ({
            filename: c.filename,
            artist: c.artist,
            name: c.name,
            category: c.category,
            points: c.points,
            questionTimings: c.questionTimings,
            answerTimings: c.answerTimings,
          }),
        ),
      ),
    );

    await categoriesConfigStream.write(JSON.stringify(categories));
    await categoriesConfigStream.close()
    await songsConfigStream.close();
  }

  async function pickFolder() {
    const dir = await window.showDirectoryPicker();

    if ((await dir.requestPermission({ mode: 'readwrite' })) === 'denied') {
      alert('А всё, этот сайт больше не работает');
    }

    configsFileHandle = await dir.getFileHandle('config.json', { create: true });
    categoriesFileHandle = await dir.getFileHandle('categories.json', { create: true });

    categories = await categoriesFileHandle
      .getFile()
      .then(file => file.text())
      .then((str) => JSON.parse(str));

    const musicConfigsFromFile: StoredMusicConfig[] = await configsFileHandle
      .getFile()
      .then((configFile) => configFile.text())
      .then((str) => JSON.parse(str))
      .catch(() => []);

    const musicFilesHandles: FileSystemFileHandle[] = [];

    for await (const handle of dir.values()) {
      const exts = ['.mp3', '.ogg', '.wav'];
      if (handle.isDirectory || !exts.some(ext => handle.name.endsWith(ext))) continue;
      musicFilesHandles.push(handle);
    }

    musicConfigs = await Promise.all(
      musicFilesHandles.map(async (handle): Promise<MusicConfig> => {
        const file = await handle.getFile();
        const audio = new Audio(URL.createObjectURL(file));

        await new Promise<void>((res) => {
          const listener = () => {
            audio.removeEventListener('loadedmetadata', listener);
            res();
          };
          audio.addEventListener('loadedmetadata', listener);
        });

        return {
          filename: file.name,
          artist: '',
          name: file.name,
          audio,
          category: '',
          points: 0,
          questionTimings: { start: 0, end: audio.duration },
          answerTimings: { start: 0, end: audio.duration },
          stopTimeout: null,
          timerInterval: null,
          currentTime: 0,
          ...musicConfigsFromFile.find((c) => c.filename === handle.name),
        };
      }),
    );

    sortConfigs();
    await saveConfigs();
  }
</script>

<main>
  {#if musicConfigs?.length}
    <button on:click={saveConfigs}>Сохранить конфигурацию</button>
  {:else}
    <button on:click={pickFolder}>Выбрать папку с музыкой</button>
  {/if}

  {#if musicConfigs?.length}
    <div class="grid">
      <h3>Категория:</h3>
      <h3>Очки:</h3>
      <h3>Исполнитель:</h3>
      <h3>Название:</h3>
      <h3>Вопрос:</h3>
      <h3>Ответ:</h3>
      {#each musicConfigs as config}
        <select bind:value={config.category} on:change={sortConfigs}>
          {#each categories as category}
            <option value={category}>{category}</option>
          {/each}
        </select>
        <select bind:value={config.points} on:change={sortConfigs}>
          {#each points as point}
            <option value={point}>{point}</option>
          {/each}
        </select>
        <input type="text" bind:value={config.artist} />
        <input type="text" bind:value={config.name} />
        <div class="flex">
          от: <input
            class="full"
            type="number"
            step={1}
            bind:value={config.questionTimings.start}
          />
          до: <input class="full" type="number" step={1} bind:value={config.questionTimings.end} />
          {#if config.stopTimeout}
            <button on:click={() => stopAudio(config)}>⏹️</button>
          {:else}
            <button on:click={() => startAudio(config, 'question')}>▶️</button>
          {/if}
          {#if config.stopTimeout}
            <div out:fade={{ delay: 500, duration: 100 }} class="timer">
              {config.currentTime.toFixed(1)}
            </div>
          {/if}
        </div>
        <div class="flex">
          от: <input class="full" type="number" step={1} bind:value={config.answerTimings.start} />
          до: <input class="full" type="number" step={1} bind:value={config.answerTimings.end} />
          {#if config.stopTimeout}
            <button on:click={() => stopAudio(config)}>⏹️</button>
          {:else}
            <button on:click={() => startAudio(config, 'answer')}>▶️</button>
          {/if}
          {#if config.stopTimeout}
            <div out:fade={{ delay: 500, duration: 100 }} class="timer">
              {config.currentTime.toFixed(1)}
            </div>
          {/if}
        </div>
      {/each}
    </div>

    <h3>Категории:</h3>
    <div style:display="flex" style:flex-direction="column" style:gap="12px" style:align-items="flex-start">
      {#each categories as category}
        <div>
          <button on:click={() => {
            categories = categories.filter(c => c !== category);
          }}>-</button>
          {category}
        </div>
      {/each}
      <div>
        <input type="text" placeholder="Новая категория" bind:value={newCategoryName}>
        <button on:click={() => {
          categories = [...categories, newCategoryName];
          newCategoryName = '';
        }}>+</button>
      </div>
    </div>
  {/if}
</main>

<style>
  main {
    margin: 2rem;
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  }
  input,
  option,
  select,
  button {
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    font-size: 1rem;
  }
  .grid {
    display: grid;
    grid-template-columns: 2fr 1fr 2fr 3fr 5fr 5fr;
    row-gap: 1rem;
    column-gap: 2rem;
    margin-top: 2rem;
  }
  .flex {
    display: flex;
    gap: 0.5rem;
    justify-content: end;
  }
  .full {
    width: 100%;
  }
  .timer {
    position: absolute;
    padding: 0.25rem;
    transform: translate(0%, 100%);
    background: white;
    border-radius: 2px;
    border: 1px grey solid;
    box-shadow: rgba(0, 0, 0, 0.5) 0px 5px 5px;
  }
</style>
