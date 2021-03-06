<script>
  import { onMount, onDestroy } from "svelte";
  import { fade } from "svelte/transition";
  import { scenes, chapters, cards, state, settings, ui } from "../stores";
  import { push } from "svelte-spa-router";
  import { _ } from "svelte-i18n";
  import OmniaEditor from "omnia-editor";
  import tippy from "sveltejs-tippy";
  import Overview from "./Write/Overview.svelte";
  import Placeholder from "../shared/Placeholder.svelte";
  import Modal from "../shared/Modal.svelte";

  export let params = {};
  let currentScene;
  let editor;
  let editorStatus = 0;
  let currentHistory = 0;
  let lengthHistory = 0;
  let showCards = false;
  let filteredCards = [];
  const editorLanguage = {
    placeholder: $_("write.editor.placeholder"),
    switch: $_("write.editor.switch"),
    delete: $_("write.editor.delete"),
    confirmDelete: $_("write.editor.confirmDelete"),
    blocks: {
      paragraph: $_("write.editor.blocks.paragraph"),
      heading: $_("write.editor.blocks.heading"),
      quote: $_("write.editor.blocks.quote"),
      code: $_("write.editor.blocks.code"),
    },
  };

  $: currentScene = $scenes.find(scene => scene.id == params.sceneId);
  $: {
    state.setCurrentTitle(
      params.sceneId ? currentScene.title : "No scene selected!"
    );
  }

  const titleInput = () => {
    scenes.setSceneTitle(currentScene.id, currentScene.title);
  };

  onMount(() => {
    window.addEventListener("hashchange", routeChange, false);
  });

  onDestroy(() => {
    window.removeEventListener("hashchange", routeChange, false);
  });

  const routeChange = () => {
    if (editor) {
      editor.update();
      document.querySelector(".content").scrollTo(0, 0);
    }
  };

  const change = e => {
    scenes.setSceneContent(params.sceneId, e.detail);
    editorStatus = 2;
    filteredCards = $cards
      .filter(e => e.showTooltip && e.project == $state.currentProject)
      .filter(c =>
        e.detail.blocks.some(
          block => block.data.text && block.data.text.includes(c.title)
        )
      );
  };

  const switchScene = e => {
    push("/write/" + e.target.value);
    e.target.selectedIndex = 0;
  };

  const toggleFocus = () => {
    $ui.focus = !$ui.focus;
  };

  const undo = () => {
    editor.history.undo();
  };

  const redo = () => {
    editor.history.redo();
  };

  const init = () => {
    editor.history.subscribe(n => {
      currentHistory = n.current;
      lengthHistory = n.data.length;
    });
  };
</script>

<div in:fade={{ duration: 100 }}>
  {#if $state.currentProject}
    {#if params.sceneId !== null}
      <Modal bind:show={showCards}>
        <h2 slot="header">{$_('write.toolbar.cards')}</h2>
        {#each filteredCards as card}
          <div>
            <h2>{card.title}</h2>
            <p>{card.content}</p>
          </div>
        {:else}
          <p>
            <Placeholder>{$_('write.toolbar.cardsEmpty')}</Placeholder>
          </p>
        {/each}
      </Modal>
      <div class="toolbar">
        <div class="inner">
          <div>
            <span
              class="lnr lnr-bold"
              use:tippy={{ content: $_('write.toolbar.bold'), placement: 'bottom' }}
              on:click={() => editor.toggleFormat('bold')} />
            <span
              class="lnr lnr-italic"
              use:tippy={{ content: $_('write.toolbar.italic'), placement: 'bottom' }}
              on:click={() => editor.toggleFormat('italic')} />
            <span
              class="lnr lnr-underline"
              use:tippy={{ content: $_('write.toolbar.underline'), placement: 'bottom' }}
              on:click={() => editor.toggleFormat('underline')} />
            {#if currentHistory < lengthHistory - 1}
              <span
                transition:fade={{ duration: 100 }}
                class="lnr lnr-undo"
                use:tippy={{ content: $_('write.toolbar.undo'), placement: 'bottom' }}
                on:click={undo} />
            {/if}
            {#if currentHistory !== 0}
              <span
                transition:fade={{ duration: 100 }}
                class="lnr lnr-redo"
                use:tippy={{ content: $_('write.toolbar.redo'), placement: 'bottom' }}
                on:click={redo} />
            {/if}
          </div>
          <div>
            {#if filteredCards.length > 0}
              <span
                transition:fade={{ duration: 100 }}
                class="lnr lnr-bookmark"
                use:tippy={{ content: $_('write.toolbar.cards'), placement: 'bottom' }}
                on:click={() => (showCards = true)} />
            {/if}
            {#if $ui.focus}
              <!-- svelte-ignore a11y-no-onchange -->
              <select
                id="focusSceneSelect"
                transition:fade={{ duration: 100 }}
                on:change={switchScene}
                class="lnr"
                use:tippy={{ content: $_('write.toolbar.switchScene'), placement: 'bottom' }}>
                <option value="" selected="selected">&#xe871;</option>
                {#each $chapters.filter(chapter => chapter.project == $state.currentProject) as chapter}
                  <optgroup label={chapter.title}>
                    {#each $scenes.filter(scene => scene.chapter == chapter.id) as scene}
                      <option value={scene.id}>{scene.title}</option>
                    {/each}
                  </optgroup>
                {/each}
              </select>
            {/if}
          </div>
          <div>
            <span
              class="lnr"
              use:tippy={{ content: $_('write.toolbar.focus'), placement: 'bottom' }}
              on:click={toggleFocus}
              class:lnr-eye={!$ui.focus}
              class:lnr-exit={$ui.focus} />
            <span
              class="lnr"
              use:tippy={{ content: $_('write.toolbar.savestate'), placement: 'bottom' }}
              class:lnr-sync={editorStatus === 1}
              class:spinner={editorStatus === 1}
              class:lnr-checkmark-circle={editorStatus === 2} />
          </div>
        </div>
      </div>
      <div class="editpane" style="--quotation-marks:{$_('write.quote.marks')}">
        <h1
          contenteditable
          bind:textContent={currentScene.title}
          on:input={titleInput} />
        <OmniaEditor
          bind:this={editor}
          bind:data={currentScene.content}
          spellCheck={$settings.spellCheck}
          translation={editorLanguage}
          on:init={init}
          on:input={() => (editorStatus = 1)}
          on:change={change} />
      </div>
    {:else}
      <Overview />
    {/if}
  {:else}
    <Placeholder />
  {/if}
</div>

<style lang="scss">
  * {
    -webkit-tap-highlight-color: rgba(255, 255, 255, 0) !important;
    -webkit-focus-ring-color: rgba(255, 255, 255, 0) !important;
    outline: 0 !important;
  }

  .editpane {
    margin: 0 auto;
    max-width: 800px;
    margin-bottom: 50vh;

    @media (min-width: 1200px) {
      padding: 10px;
      box-shadow: 0 5px 26px 2px rgba(0, 0, 0, 0.2);
    }

    > h1 {
      font-family: "Libre Baskerville";
      text-align: center;
    }
  }

  select#focusSceneSelect {
    background-color: var(--select-background);
    font-weight: bold;
    width: 1.5rem;
    color: var(--text-color);
    text-align: center;
    -webkit-appearance: none;
    border: 0;
    cursor: pointer;

    &:focus,
    &:active {
      border: 0;
      outline: 0;
    }
  }

  .toolbar {
    position: sticky;
    position: -webkit-sticky;
    top: 0;
    background-color: var(--background-color);
    padding: 1rem;
    font-size: 1.5rem;
    margin-bottom: 2rem;
    z-index: 8;

    > .inner {
      display: flex;
      justify-content: space-between;
      align-items: center;
      align-content: center;
      margin: auto;
      max-width: 800px;

      > div {
        * {
          margin: 0 0.5rem;
          opacity: 0.65;
          cursor: pointer;
        }
        *:hover {
          opacity: 1;
        }
        span {
          font-size: 1rem;
        }
        .lnr {
          font-size: unset;

          &.spinner {
            display: inline-block;
            -webkit-animation: spin 2s infinite linear;
            animation: spin 2s infinite linear;
          }
          @keyframes spin {
            0% {
              -webkit-transform: rotate(0deg);
              transform: rotate(0deg);
            }

            100% {
              -webkit-transform: rotate(359deg);
              transform: rotate(359deg);
            }
          }
        }
      }
    }
  }
</style>
