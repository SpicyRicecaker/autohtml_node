<script lang="ts">
  import Handlebars from 'handlebars';
  import { local } from './stores';
  import { promises as fs } from 'fs';
  import path from 'path';
  import { onMount } from 'svelte';
  interface form {
    disc: { value: string; modified: boolean };
    topic: { value: string; modified: boolean };
    pdfURL: { value: string; modified: boolean };
    desmosURL: { value: string; modified: boolean };
    repoURL: { value: string; modified: boolean };
    outputDir: { value: string; modified: boolean };
  }

  let form: form = {
    disc: { value: '', modified: false },
    topic: { value: '', modified: false },
    pdfURL: { value: '', modified: false },
    desmosURL: { value: '', modified: false },
    repoURL: { value: '', modified: false },
    outputDir: { value: '', modified: false },
  };

  onMount(() => {
    (Object.keys(form) as (keyof form)[]).forEach((key) => {
      const t = local.getItem(key);
      if (t) {
        form[key].value = t;
      }
    });
  });

  $: formEntries = Object.keys(form) as (keyof form)[];

  // Take in key of form
  const isLocalChange = (key: keyof form): boolean =>
    local.getItem(key) === form[key].value ? false : true;

  const getColor = (bool: boolean): string => (bool ? '#9DBF9E' : '#987284');

  const make = async () => {
    const template = Handlebars.compile(
      await fs.readFile(
        path.join(path.resolve(), 'pages', 'src', 'index.hbs'),
        'utf-8'
      )
    );
    const filepath = path.join(
      form.outputDir.value,
      `disc-${form.disc.value}`,
      `disc-${form.disc.value}-${form.topic.value}`,
      `disc-${form.disc.value}-${form.topic.value}.html`
    );
    // Write the file
    await fs.writeFile(filepath, template(form));
    // Update local storage
    Object.entries(form).forEach(([key, value]) => {
      console.log(key, value.value);
      local.setItem(key, value.value);
    });
  };
</script>

<style lang="scss">
  :global(body) {
    margin: 0;
    padding: 0;
  }
  main {
    // Max out content
    width: 100%;
    height: 100%;
    max-width: 100%;
    max-height: 100%;
    display: grid;
    grid-template:
      'form action' /
      minmax(0, 1fr) minmax(0, 1fr);
    // Make it more native, turn it on later if needed
    user-select: none;
  }

  .form {
    grid-area: form;
    padding: 1rem;
    display: grid;
    & * {
      min-width: 0;
      word-break: break-all;
    }
  }

  .action {
    grid-area: action;
    padding: 1rem;
    display: grid;
    min-height: 0; /* NEW */
    min-width: 0;
  }
</style>

<main>
  <div class="form">
    {#each formEntries as key}
      <input
        type="text"
        placeholder={key}
        bind:value={form[key].value}
        on:input={() => (form[key].modified = isLocalChange(key))} />
      <div style="color: {getColor(form[key].modified)}">
        {#if form[key].modified}😍{:else}🤔{/if}
        {local.getItem(key)}
      </div>
    {/each}
    <!-- <button>Choose path</button> -->
  </div>
  <div class="action"><button on:click={make}>MAKE</button></div>
</main>
