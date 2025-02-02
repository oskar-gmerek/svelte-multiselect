<script lang="ts">
  import { createEventDispatcher } from 'svelte'
  import type { DispatchEvents, MultiSelectEvents, ObjectOption, Option } from './'
  import { get_label, get_value } from './'
  import CircleSpinner from './CircleSpinner.svelte'
  import { CrossIcon, DisabledIcon, ExpandIcon } from './icons'
  import Wiggle from './Wiggle.svelte'

  export let activeIndex: number | null = null
  export let activeOption: Option | null = null
  export let addOptionMsg: string = `Create this option...`
  export let allowUserOptions: boolean | 'append' = false
  export let autocomplete: string = `off`
  export let autoScroll: boolean = true
  export let breakpoint: number = 800 // any screen with more horizontal pixels is considered desktop, below is mobile
  export let defaultDisabledTitle: string = `This option is disabled`
  export let disabled: boolean = false
  export let disabledInputTitle: string = `This input is disabled`
  export let filterFunc = (op: Option, searchText: string): boolean => {
    if (!searchText) return true
    return `${get_label(op)}`.toLowerCase().includes(searchText.toLowerCase())
  }
  export let focusInputOnSelect: boolean | 'desktop' = `desktop`
  export let id: string | null = null
  export let input: HTMLInputElement | null = null
  export let inputClass: string = ``
  export let invalid: boolean = false
  export let liActiveOptionClass: string = ``
  export let liOptionClass: string = ``
  export let liSelectedClass: string = ``
  export let loading: boolean = false
  export let matchingOptions: Option[] = []
  export let maxSelect: number | null = null // null means any number of options are selectable
  export let maxSelectMsg: ((current: number, max: number) => string) | null = null
  export let name: string | null = null
  export let noOptionsMsg: string = `No matching options`
  export let open: boolean = false
  export let options: Option[]
  export let outerDiv: HTMLDivElement | null = null
  export let outerDivClass: string = ``
  export let parseLabelsAsHtml: boolean = false // should not be combined with allowUserOptions!
  export let placeholder: string | null = null
  export let removeAllTitle: string = `Remove all`
  export let removeBtnTitle: string = `Remove`
  export let required: boolean = false
  export let searchText: string = ``
  export let selected: Option[] =
    options?.filter((op) => (op as ObjectOption)?.preselected) ?? []
  export let selectedLabels: (string | number)[] = []
  export let selectedValues: unknown[] = []
  export let sortSelected: boolean | ((op1: Option, op2: Option) => number) = false
  export let ulOptionsClass: string = ``
  export let ulSelectedClass: string = ``

  type $$Events = MultiSelectEvents // for type-safe event listening on this component

  if (!(options?.length > 0)) {
    if (allowUserOptions) {
      options = [] // initializing as array avoids errors when component mounts
    } else {
      // only error for empty options if user is not allowed to create custom options
      console.error(`MultiSelect received no options`)
    }
  }
  if (parseLabelsAsHtml && allowUserOptions) {
    console.warn(
      `Don't combine parseLabelsAsHtml and allowUserOptions. It's susceptible to XSS attacks!`
    )
  }
  if (maxSelect !== null && maxSelect < 1) {
    console.error(`maxSelect must be null or positive integer, got ${maxSelect}`)
  }
  if (!Array.isArray(selected)) {
    console.error(`selected prop must be an array, got ${selected}`)
  }

  const dispatch = createEventDispatcher<DispatchEvents>()
  let add_option_msg_is_active: boolean = false // controls active state of <li>{addOptionMsg}</li>
  let window_width: number

  let wiggle = false // controls wiggle animation when user tries to exceed maxSelect
  $: selectedLabels = selected.map(get_label)
  $: selectedValues = selected.map(get_value)

  // formValue binds to input.form-control to prevent form submission if required
  // prop is true and no options are selected
  $: formValue = selectedValues.join(`,`)
  $: if (formValue) invalid = false // reset error status whenever component state changes

  // options matching the current search text
  $: matchingOptions = options.filter(
    (op) => filterFunc(op, searchText) && !selectedLabels.includes(get_label(op)) // remove already selected options from dropdown list
  )
  // raise if matchingOptions[activeIndex] does not yield a value
  if (activeIndex !== null && !matchingOptions[activeIndex]) {
    throw `Run time error, activeIndex=${activeIndex} is out of bounds, matchingOptions.length=${matchingOptions.length}`
  }
  // update activeOption when activeIndex changes
  $: activeOption = activeIndex !== null ? matchingOptions[activeIndex] : null

  // add an option to selected list
  function add(label: string | number) {
    if (maxSelect && maxSelect > 1 && selected.length >= maxSelect) wiggle = true
    // to prevent duplicate selection, we could add `&& !selectedLabels.includes(label)`
    if (maxSelect === null || maxSelect === 1 || selected.length < maxSelect) {
      // first check if we find option in the options list

      let option = options.find((op) => get_label(op) === label)
      if (
        !option && // this has the side-effect of not allowing to user to add the same
        // custom option twice in append mode
        [true, `append`].includes(allowUserOptions) &&
        searchText.length > 0
      ) {
        // user entered text but no options match, so if allowUserOptions=true | 'append', we create
        // a new option from the user-entered text
        if (typeof options[0] === `object`) {
          // if 1st option is an object, we create new option as object to keep type homogeneity
          option = { label: searchText, value: searchText }
        } else {
          if (
            [`number`, `undefined`].includes(typeof options[0]) &&
            !isNaN(Number(searchText))
          ) {
            // create new option as number if it parses to a number and 1st option is also number or missing
            option = Number(searchText)
          } else option = searchText // else create custom option as string
        }
        if (allowUserOptions === `append`) options = [...options, option]
      }
      searchText = `` // reset search string on selection
      if (!option) {
        console.error(`MultiSelect: option with label ${label} not found`)
        return
      }
      if (maxSelect === 1) {
        // for maxselect = 1 we always replace current option with new one
        selected = [option]
      } else {
        selected = [...selected, option]
        if (sortSelected === true) {
          selected = selected.sort((op1: Option, op2: Option) => {
            const [label1, label2] = [get_label(op1), get_label(op2)]
            // coerce to string if labels are numbers
            return `${label1}`.localeCompare(`${label2}`)
          })
        } else if (typeof sortSelected === `function`) {
          selected = selected.sort(sortSelected)
        }
      }
      if (selected.length === maxSelect) close_dropdown()
      else if (
        focusInputOnSelect === true ||
        (focusInputOnSelect === `desktop` && window_width > breakpoint)
      ) {
        input?.focus()
      }
      dispatch(`add`, { option })
      dispatch(`change`, { option, type: `add` })
    }
  }

  // remove an option from selected list
  function remove(label: string | number) {
    if (selected.length === 0) return

    selected.splice(selectedLabels.lastIndexOf(label), 1)
    selected = selected // Svelte rerender after in-place splice

    const option =
      options.find((option) => get_label(option) === label) ??
      // if option with label could not be found but allowUserOptions is truthy,
      // assume it was created by user and create correspondidng option object
      // on the fly for use as event payload
      (allowUserOptions && { label, value: label })

    if (!option) {
      return console.error(`MultiSelect: option with label ${label} not found`)
    }

    dispatch(`remove`, { option })
    dispatch(`change`, { option, type: `remove` })
  }

  function open_dropdown() {
    if (disabled) return
    open = true
    input?.focus()
    dispatch(`focus`)
  }

  function close_dropdown() {
    open = false
    input?.blur()
    activeOption = null
    dispatch(`blur`)
  }

  // handle all keyboard events this component receives
  async function handle_keydown(event: KeyboardEvent) {
    // on escape or tab out of input: dismiss options dropdown and reset search text
    if (event.key === `Escape` || event.key === `Tab`) {
      close_dropdown()
      searchText = ``
    }
    // on enter key: toggle active option and reset search text
    else if (event.key === `Enter`) {
      event.preventDefault() // prevent enter key from triggering form submission

      if (activeOption) {
        const label = get_label(activeOption)
        selectedLabels.includes(label) ? remove(label) : add(label)
        searchText = ``
      } else if (allowUserOptions && searchText.length > 0) {
        // user entered text but no options match, so if allowUserOptions is truthy, we create new option
        add(searchText)
      }
      // no active option and no search text means the options dropdown is closed
      // in which case enter means open it
      else open_dropdown()
    }
    // on up/down arrow keys: update active option
    else if ([`ArrowDown`, `ArrowUp`].includes(event.key)) {
      // if no option is active yet, but there are matching options, make first one active
      if (activeIndex === null && matchingOptions.length > 0) {
        activeIndex = 0
        return
      } else if (allowUserOptions && searchText.length > 0) {
        // if allowUserOptions is truthy and user entered text but no options match, we make
        // <li>{addUserMsg}</li> active on keydown (or toggle it if already active)
        add_option_msg_is_active = !add_option_msg_is_active
        return
      } else if (activeIndex === null) {
        // if no option is active and no options are matching, do nothing
        return
      }
      const increment = event.key === `ArrowUp` ? -1 : 1

      activeIndex = (activeIndex + increment) % matchingOptions.length
      // % in JS behaves like remainder operator, not real modulo, so negative numbers stay negative
      // need to do manual wrap around at 0
      if (activeIndex < 0) activeIndex = matchingOptions.length - 1

      if (autoScroll) {
        // TODO This ugly timeout hack is needed to properly scroll element into view when wrapping
        // around start/end of option list. Find a better solution than waiting 10 ms to.
        setTimeout(() => {
          const li = document.querySelector(`ul.options > li.active`)
          if (li) {
            li.parentNode?.scrollIntoView({ block: `center` })
            li.scrollIntoViewIfNeeded()
          }
        }, 10)
      }
    }
    // on backspace key: remove last selected option
    else if (event.key === `Backspace` && selectedLabels.length > 0 && !searchText) {
      remove(selectedLabels.at(-1) as string | number)
    }
  }

  function remove_all() {
    dispatch(`removeAll`, { options: selected })
    dispatch(`change`, { options: selected, type: `removeAll` })
    selected = []
    searchText = ``
  }

  $: is_selected = (label: string | number) => selectedLabels.includes(label)

  const if_enter_or_space = (handler: () => void) => (event: KeyboardEvent) => {
    if ([`Enter`, `Space`].includes(event.code)) {
      event.preventDefault()
      handler()
    }
  }

  function on_click_outside(event: MouseEvent | TouchEvent) {
    if (outerDiv && !outerDiv.contains(event.target as Node)) {
      close_dropdown()
    }
  }
</script>

<svelte:window
  on:click={on_click_outside}
  on:touchstart={on_click_outside}
  bind:innerWidth={window_width}
/>

<div
  bind:this={outerDiv}
  class:disabled
  class:single={maxSelect === 1}
  class:open
  aria-expanded={open}
  aria-multiselectable={maxSelect === null || maxSelect > 1}
  class:invalid
  class="multiselect {outerDivClass}"
  on:mouseup|stopPropagation={open_dropdown}
  title={disabled ? disabledInputTitle : null}
  aria-disabled={disabled ? `true` : null}
>
  <input
    {required}
    bind:value={formValue}
    tabindex="-1"
    aria-hidden="true"
    aria-label="ignore this, used only to prevent form submission if select is required but empty"
    class="form-control"
    on:invalid={() => (invalid = true)}
  />
  <ExpandIcon width="15px" style="min-width: 1em; padding: 0 1pt;" />
  <ul class="selected {ulSelectedClass}">
    {#each selected as option, idx}
      <li class={liSelectedClass} aria-selected="true">
        <slot name="selected" {option} {idx}>
          {#if parseLabelsAsHtml}
            {@html get_label(option)}
          {:else}
            {get_label(option)}
          {/if}
        </slot>
        {#if !disabled}
          <button
            on:mouseup|stopPropagation={() => remove(get_label(option))}
            on:keydown={if_enter_or_space(() => remove(get_label(option)))}
            type="button"
            title="{removeBtnTitle} {get_label(option)}"
          >
            <slot name="remove-icon"><CrossIcon width="15px" /></slot>
          </button>
        {/if}
      </li>
    {/each}
    <li style="display: contents;">
      <input
        class={inputClass}
        bind:this={input}
        {autocomplete}
        bind:value={searchText}
        on:mouseup|self|stopPropagation={open_dropdown}
        on:keydown={handle_keydown}
        on:focus={open_dropdown}
        {id}
        {name}
        {disabled}
        placeholder={selectedLabels.length ? `` : placeholder}
        aria-invalid={invalid ? `true` : null}
      />
    </li>
  </ul>
  {#if loading}
    <slot name="spinner">
      <CircleSpinner />
    </slot>
  {/if}
  {#if disabled}
    <slot name="disabled-icon">
      <DisabledIcon width="15px" />
    </slot>
  {:else if selected.length > 0}
    {#if maxSelect && (maxSelect > 1 || maxSelectMsg)}
      <Wiggle bind:wiggle angle={20}>
        <span style="padding: 0 3pt;">
          {maxSelectMsg?.(selected.length, maxSelect) ??
            (maxSelect > 1 ? `${selected.length}/${maxSelect}` : ``)}
        </span>
      </Wiggle>
    {/if}
    {#if maxSelect !== 1 && selected.length > 1}
      <button
        type="button"
        class="remove-all"
        title={removeAllTitle}
        on:mouseup|stopPropagation={remove_all}
        on:keydown={if_enter_or_space(remove_all)}
      >
        <slot name="remove-icon"><CrossIcon width="15px" /></slot>
      </button>
    {/if}
  {/if}

  <!-- only render options dropdown if options or searchText is not empty needed to avoid briefly flashing empty dropdown -->
  {#if searchText || options?.length > 0}
    <ul class:hidden={!open} class="options {ulOptionsClass}">
      {#each matchingOptions as option, idx}
        {@const {
          label,
          disabled = null,
          title = null,
          selectedTitle = null,
          disabledTitle = defaultDisabledTitle,
        } = option instanceof Object ? option : { label: option }}
        {@const active = activeIndex === idx}
        <li
          on:mousedown|stopPropagation
          on:mouseup|stopPropagation={() => {
            if (!disabled) is_selected(label) ? remove(label) : add(label)
          }}
          title={disabled
            ? disabledTitle
            : (is_selected(label) && selectedTitle) || title}
          class:selected={is_selected(label)}
          class:active
          class:disabled
          class="{liOptionClass} {active ? liActiveOptionClass : ``}"
          on:mouseover={() => {
            if (!disabled) activeIndex = idx
          }}
          on:focus={() => {
            if (!disabled) activeIndex = idx
          }}
          on:mouseout={() => (activeIndex = null)}
          on:blur={() => (activeIndex = null)}
          aria-selected="false"
        >
          <slot name="option" {option} {idx}>
            {#if parseLabelsAsHtml}
              {@html get_label(option)}
            {:else}
              {get_label(option)}
            {/if}
          </slot>
        </li>
      {:else}
        {#if allowUserOptions && searchText}
          <li
            on:mousedown|stopPropagation
            on:mouseup|stopPropagation={() => add(searchText)}
            title={addOptionMsg}
            class:active={add_option_msg_is_active}
            on:mouseover={() => (add_option_msg_is_active = true)}
            on:focus={() => (add_option_msg_is_active = true)}
            on:mouseout={() => (add_option_msg_is_active = false)}
            on:blur={() => (add_option_msg_is_active = false)}
            aria-selected="false"
          >
            {addOptionMsg}
          </li>
        {:else}
          <span>{noOptionsMsg}</span>
        {/if}
      {/each}
    </ul>
  {/if}
</div>

<style>
  :where(div.multiselect) {
    position: relative;
    align-items: center;
    display: flex;
    cursor: text;
    border: var(--sms-border, 1pt solid lightgray);
    border-radius: var(--sms-border-radius, 3pt);
    background: var(--sms-bg);
    max-width: var(--sms-max-width);
    padding: var(--sms-padding, 0 3pt);
    color: var(--sms-text-color);
    font-size: var(--sms-font-size, inherit);
    min-height: var(--sms-min-height, 19pt);
    margin: var(--sms-margin);
  }
  :where(div.multiselect.open) {
    /* increase z-index when open to ensure the dropdown of one <MultiSelect />
    displays above that of another slightly below it on the page */
    z-index: var(--sms-open-z-index, 4);
  }
  :where(div.multiselect:focus-within) {
    border: var(--sms-focus-border, 1pt solid var(--sms-active-color, cornflowerblue));
  }
  :where(div.multiselect.disabled) {
    background: var(--sms-disabled-bg, lightgray);
    cursor: not-allowed;
  }

  :where(div.multiselect > ul.selected) {
    display: flex;
    flex: 1;
    padding: 0;
    margin: 0;
    flex-wrap: wrap;
  }
  :where(div.multiselect > ul.selected > li) {
    align-items: center;
    border-radius: 3pt;
    display: flex;
    margin: 2pt;
    line-height: normal;
    transition: 0.3s;
    white-space: nowrap;
    background: var(--sms-selected-bg, rgba(0, 0, 0, 0.15));
    padding: var(--sms-selected-li-padding, 1pt 5pt);
    color: var(--sms-selected-text-color, var(--sms-text-color));
  }
  :where(div.multiselect button) {
    border-radius: 50%;
    display: flex;
    transition: 0.2s;
    color: inherit;
    background: transparent;
    border: none;
    cursor: pointer;
    outline: none;
    padding: 0;
    margin: 0 0 0 3pt; /* CSS reset */
  }
  :where(div.multiselect button.remove-all) {
    margin: 0 3pt;
  }
  :where(ul.selected > li button:hover, button.remove-all:hover, button:focus) {
    color: var(--sms-button-hover-color, lightskyblue);
  }

  :where(div.multiselect input) {
    margin: auto 0; /* CSS reset */
    padding: 0; /* CSS reset */
  }
  :where(div.multiselect > ul.selected > li > input) {
    border: none;
    outline: none;
    background: none;
    flex: 1; /* this + next line fix issue #12 https://git.io/JiDe3 */
    min-width: 2em;
    /* ensure input uses text color and not --sms-selected-text-color */
    color: var(--sms-text-color);
    font-size: inherit;
    cursor: inherit; /* needed for disabled state */
    border-radius: 0; /* reset ul.selected > li */
  }
  :where(div.multiselect > ul.selected > li > input)::placeholder {
    padding-left: 5pt;
    color: var(--sms-placeholder-color);
    opacity: var(--sms-placeholder-opacity);
  }
  :where(div.multiselect > input.form-control) {
    width: 2em;
    position: absolute;
    background: transparent;
    border: none;
    outline: none;
    z-index: -1;
    opacity: 0;
    pointer-events: none;
  }

  :where(div.multiselect > ul.options) {
    list-style: none;
    padding: 4pt 0;
    top: 100%;
    left: 0;
    width: 100%;
    position: absolute;
    border-radius: 1ex;
    overflow: auto;
    background: var(--sms-options-bg, white);
    max-height: var(--sms-options-max-height, 50vh);
    overscroll-behavior: var(--sms-options-overscroll, none);
    box-shadow: var(--sms-options-shadow, 0 0 14pt -8pt black);
    transition: all 0.2s;
  }
  :where(div.multiselect > ul.options.hidden) {
    visibility: hidden;
    opacity: 0;
    transform: translateY(50px);
  }
  :where(div.multiselect > ul.options > li) {
    padding: 3pt 2ex;
    cursor: pointer;
    scroll-margin: var(--sms-options-scroll-margin, 100px);
  }
  /* for noOptionsMsg */
  :where(div.multiselect > ul.options span) {
    padding: 3pt 2ex;
  }
  :where(div.multiselect > ul.options > li.selected) {
    background: var(--sms-li-selected-bg);
    color: var(--sms-li-selected-color);
  }
  :where(div.multiselect > ul.options > li.active) {
    background: var(--sms-li-active-bg, var(--sms-active-color, rgba(0, 0, 0, 0.15)));
  }
  :where(div.multiselect > ul.options > li.disabled) {
    cursor: not-allowed;
    background: var(--sms-li-disabled-bg, #f5f5f6);
    color: var(--sms-li-disabled-text, #b8b8b8);
  }
</style>
