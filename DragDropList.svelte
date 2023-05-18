<script>

  // 참고자료 : https://github.com/jwlarocque/svelte-dragdroplist/blob/master/DragDropList.svelte

  import { flip } from "svelte/animate";
  import { createEventDispatcher } from "svelte";
  const dispatcher = createEventDispatcher();

  export let data = [];
  export let removesItems = false;
  export let upDownButtons = false;
  export let dataKey = "id";

  let ghost;
  let grabbed;

  let lastTarget;

  let mouseY = 0; // pointer y coordinate within client
  let offsetY = 0; // y distance from top of grabbed element to pointer
  let layerY = 0; // distance from top of list to top of client

  function grab(clientY, element) {
    // modify grabbed element
    grabbed = element;
    grabbed.dataset.grabY = clientY;

    // modify ghost element (which is actually dragged)
    ghost.innerHTML = grabbed.innerHTML;

    // record offset from cursor to top of element
    // (used for positioning ghost)
    offsetY = grabbed.getBoundingClientRect().y - clientY;
    drag(clientY);
  }

  // drag handler updates cursor position
  function drag(clientY) {
    if (grabbed) {
      mouseY = clientY;
      layerY = ghost.parentNode.getBoundingClientRect().y;
    }
  }

  // touchEnter handler emulates the mouseenter event for touch input
  // (more or less)
  function touchEnter(ev) {
    drag(ev.clientY);
    // trigger dragEnter the first time the cursor moves over a list item
    let target = document.elementFromPoint(ev.clientX, ev.clientY).closest(".item");
    if (target && target !== lastTarget) {
      lastTarget = target;
      dragEnter(ev, target);
    }
  }

  function dragEnter(ev, target) {
    // swap items in data
    if (grabbed && target !== grabbed && target.classList.contains("item")) {
      moveDatum(parseInt(grabbed.dataset.index), parseInt(target.dataset.index));
    }
  }

  // does the actual moving of items in data
  function moveDatum(from, to) {
    let temp = data[from];
    data = [...data.slice(0, from), ...data.slice(from + 1)];
    data = [...data.slice(0, to), temp, ...data.slice(to)];
  }

  function release() {
    grabbed = null;
  }

  function removeDatum(index) {
    data = [...data.slice(0, index), ...data.slice(index + 1)];
    dispatcher("remove", index);
  }
</script>

<style>
  section {
    position: relative;
  }

  .list {
    z-index: 5;
    display: flex;
    flex-direction: column;
  }

  .item {
    cursor: grab;
    box-sizing: border-box;
    display: inline-flex;
    width: 100%;
    min-height: 3em;
    margin-bottom: 0.5em;
    user-select: none;
    align-items: center;
  }

  .item:last-child {
    margin-bottom: 0;
  }

  /*noinspection CssUnusedSymbol*/
  .item:not(#grabbed):not(#ghost) {
    z-index: 10;
  }

  .buttons {
    width: 32px;
    min-width: 32px;
    margin: auto 0;
    display: flex;
    flex-direction: column;
  }

  .buttons button {
    cursor: pointer;
    padding: 4px;
    border: 1px solid rgba(0, 0, 0, 0);
    background-color: inherit;
  }

  .buttons button:hover {
    background-color: #9a9a9a33;
    border-radius: 2pt;
  }

  .rear-buttons {
    position: absolute;
    right: 0;
  }

  .delete {
    width: 32px;
    margin-right: 8px;
  }

  /*noinspection CssUnusedSymbol*/
  #grabbed {
    opacity: 0.0;
  }

  #ghost {
    pointer-events: none;
    z-index: -5;
    position: absolute;
    top: 0;
    left: 0;
    opacity: 0.0;
  }

  #ghost * {
    pointer-events: none;
  }

  /*noinspection ALL*/
  #ghost.haunting {
    z-index: 20;
    opacity: 1.0;
  }

  .drag-shadow {
    box-shadow: 0 0 #0000, 0 0 #0000,
      0 10px 15px -3px rgb(0 0 0 / 0.1), 0 4px 6px -4px rgb(0 0 0 / 0.1);
    transition-property: box-shadow;
    transition-timing-function: cubic-bezier(0.4, 0, 0.2, 1);
    transition-duration: 150ms;
  }

</style>

<!-- All the documentation has to go up here, sorry.
     (otherwise it conflicts with the HTML or svelte/animate)
     The .list has handlers for pointer movement and pointer up/release/end.
     Each .item has a handler for pointer down/click/start, which assigns that
     element as the item currently being "grabbed".  They also have a handler
     for pointer enter (the touchmove handler has extra logic to behave like the
     no longer extant 'touchenter'), which swaps the entered element with the
     grabbed element when triggered.
     You'll also find reactive styling below, which keeps it from being directly
     part of the imperative javascript handlers. -->
<section class="drag-drop-list">
  <div
    bind:this={ghost}
    id="ghost"
    class="item drag-shadow {grabbed ? 'haunting' : ''}"
    style={"top: " + (mouseY + offsetY - layerY) + "px"}>
    <p></p>
  </div>

  <div
    class="list"
    on:mousemove={function(ev) {ev.stopPropagation(); drag(ev.clientY);}}
    on:touchmove={function(ev) {ev.stopPropagation(); drag(ev.touches[0].clientY);}}
    on:mouseup={function(ev) {ev.stopPropagation(); release(ev);}}
    on:touchend={function(ev) {ev.stopPropagation(); release(ev.touches[0]);}}>

    {#each data as item, index (item[dataKey] ? item[dataKey].toString() : JSON.stringify(item))}
      <div
        id={(grabbed && (item[dataKey] ? item[dataKey].toString() : JSON.stringify(item)) === grabbed.dataset.id) ? "grabbed" : ""}
        class="item"
        data-index={index}
        data-id={(item[dataKey] ? item[dataKey] : JSON.stringify(item))}
        data-grabY="0"
        on:mousedown={function(ev) {grab(ev.clientY, this);}}
        on:touchstart={function(ev) {grab(ev.touches[0].clientY, this);}}
        on:mouseenter={function(ev) {ev.stopPropagation(); dragEnter(ev, ev.target);}}
        on:touchmove={function(ev) {ev.stopPropagation(); ev.preventDefault(); touchEnter(ev.touches[0]);}}
        animate:flip|local={{duration: 200}}>

        {#if upDownButtons}
          <div class="buttons">
            <button
              class="up"
              style={"visibility: " + (index > 0 ? "" : "hidden") + ";"}
              on:click|preventDefault={function() {moveDatum(index, index - 1)}}>
              <svg width="15" height="15" viewBox="0 0 15 15" fill="none" xmlns="http://www.w3.org/2000/svg">
                <path
                  d="M3.13523 8.84197C3.3241 9.04343 3.64052 9.05363 3.84197 8.86477L7.5 5.43536L11.158 8.86477C11.3595 9.05363 11.6759 9.04343 11.8648 8.84197C12.0536 8.64051 12.0434 8.32409 11.842 8.13523L7.84197 4.38523C7.64964 4.20492 7.35036 4.20492 7.15803 4.38523L3.15803 8.13523C2.95657 8.32409 2.94637 8.64051 3.13523 8.84197Z"
                  fill="currentColor" fill-rule="evenodd" clip-rule="evenodd"></path>
              </svg>
            </button>
            <button
              class="down"
              style={"visibility: " + (index < data.length - 1 ? "" : "hidden") + ";"}
              on:click|preventDefault={function() {moveDatum(index, index + 1)}}>
              <svg width="15" height="15" viewBox="0 0 15 15" fill="none" xmlns="http://www.w3.org/2000/svg">
                <path
                  d="M3.13523 6.15803C3.3241 5.95657 3.64052 5.94637 3.84197 6.13523L7.5 9.56464L11.158 6.13523C11.3595 5.94637 11.6759 5.95657 11.8648 6.15803C12.0536 6.35949 12.0434 6.67591 11.842 6.86477L7.84197 10.6148C7.64964 10.7951 7.35036 10.7951 7.15803 10.6148L3.15803 6.86477C2.95657 6.67591 2.94637 6.35949 3.13523 6.15803Z"
                  fill="currentColor" fill-rule="evenodd" clip-rule="evenodd"></path>
              </svg>
            </button>
          </div>
        {/if}

        <div class="w-full">
          <slot item={item}></slot>
        </div>

        <div class="buttons rear-buttons delete">
          {#if removesItems}
            <button on:click|preventDefault|stopImmediatePropagation={ () => removeDatum(index) }>
              <svg width="15" height="15" viewBox="0 0 15 15" fill="none" xmlns="http://www.w3.org/2000/svg">
                <path
                  d="M11.7816 4.03157C12.0062 3.80702 12.0062 3.44295 11.7816 3.2184C11.5571 2.99385 11.193 2.99385 10.9685 3.2184L7.50005 6.68682L4.03164 3.2184C3.80708 2.99385 3.44301 2.99385 3.21846 3.2184C2.99391 3.44295 2.99391 3.80702 3.21846 4.03157L6.68688 7.49999L3.21846 10.9684C2.99391 11.193 2.99391 11.557 3.21846 11.7816C3.44301 12.0061 3.80708 12.0061 4.03164 11.7816L7.50005 8.31316L10.9685 11.7816C11.193 12.0061 11.5571 12.0061 11.7816 11.7816C12.0062 11.557 12.0062 11.193 11.7816 10.9684L8.31322 7.49999L11.7816 4.03157Z"
                  fill="currentColor" fill-rule="evenodd" clip-rule="evenodd"></path>
              </svg>
            </button>
          {/if}
        </div>
      </div>
    {/each}
  </div>
</section>